# What are Reserves

A reserve (sometimes called a market, pool, or asset reserve) is the contract-level structure that tracks the supply, borrow, and risk state of a particular underlying token.


It holds the liquidity (supplied tokens), outstanding borrow positions, interest accrual state, and configuration parameters (e.g. LTV, liquidation thresholds, fees) for that token.


It acts as the accounting and control point for user interactions (deposit, withdraw, borrow, repay, liquidation) on that asset.


Key invariants / responsibilities include:
   1. Maintain solvency: ensure total liabilities (borrow + accrued interest) ≤ total assets.
   2. Interest accrual / rate model: update internal indices so that supplier and borrower balances evolve appropriately over time.
   3. Risk management: enforce caps, limits, collateralization rules, and trigger liquidations when positions become unsafe.
   4. Reserve buffer / protocol reserve: optionally hold a fraction of interest or fees to protect against shortfalls or to act as insurance.


In many lending designs (e.g. Aave, Compound), each reserve is independent (with its own interest curve, parameters, and risk settings). See Aave’s definition of a reserve: “an instance of a token within an Aave liquidity pool … governed by a set of parameters that manage risk and optimise liquidity.” aave.com


On blockchains, reserves also interact with oracle price feeds, liquidity vaults, and external integrations (e.g. collateral swapping, yield strategies).


In a Solana context, the reserve will typically be a PDA / on-chain account structure with sub-accounts (e.g. liquidity vault, fee vault, collateral vault) and state fields (interest index, last update timestamp, etc.).


### Reserve Config
Loan-to-Value (LTV): max borrowable % of collateral value

Liquidation Threshold (LT): point at which position becomes liquidatable

Liquidation Bonus: incentive for liquidators (extra collateral seized)

Borrow Caps / Deposit Caps: max liquidity that can be borrowed or supplied

Interest Rate Curve Params: min borrow rate, optimal utilization, slopes, max borrow rate

Protocol Take Rate: % of interest diverted to protocol treasury

Flags: whether collateral is enabled, flash loans allowed, strategy linked

### Farm
A farm (or strategy) refers to an external yield-generating contract or protocol where excess reserve liquidity (i.e. idle tokens not currently lent out) can be deployed to earn additional yield (e.g. via staking, liquidity provision, or yield farming).


### Reserve Lifecycle Diagram (Admin & Safety States)

This diagram shows the lifecycle of a reserve account from creation to closure. A reserve begins uninitialized, is created via init_reserve, becomes active, and may later be linked to external strategies (farms). Governance or guardians can freeze or pause reserves if there are risks (oracle failure, governance intervention), while decommissioning and closure handle the safe wind-down of markets. This captures all the admin and safety transitions a reserve can go through.

``` mermaid
stateDiagram-v2
  [*] --> Uninitialized
  Uninitialized --> Initialized: init_reserve
  Initialized --> Active: enable_reserve
  Active --> StrategyLinked: init_farms_for_reserve
  Active --> Frozen: freeze_reserve
  StrategyLinked --> Frozen: freeze_reserve
  Frozen --> Active: unfreeze_reserve
  Active --> Paused: pause_reserve/program
  Paused --> Active: unpause_reserve/program
  Active --> Decommissioning: begin_decommission
  StrategyLinked --> Decommissioning: begin_decommission
  Decommissioning --> Closed: close_reserve
```

## Reserve Operational Diagram
This diagram focuses on how a live reserve behaves during normal operations. In the active state, it continuously accrues interest, processes user actions like borrows, repays, deposits, withdraws, and handles liquidations when accounts become unhealthy. For Kamino-style reserves, it also shows optional states for Rebalancing and Harvesting, where liquidity is moved into or out of farms (strategies), and Unwinding, when liquidity must be pulled back from strategies to fulfill withdrawals or liquidations. This captures the user-facing flows and keeper actions in the reserve’s day-to-day operation.
``` mermaid
stateDiagram-v2
  [*] --> Accruing
  Accruing --> Utilizing: borrow/repay
  Accruing --> Rebalancing: rebalance(threshold)
  Accruing --> Harvesting: harvest
  Accruing --> Unwinding: withdraw & low vault liquidity
  Unwinding --> Accruing: unwind complete
  Harvesting --> Accruing
  Rebalancing --> Accruing
  Accruing --> LiquidationProcessing: liquidate
  LiquidationProcessing --> Accruing
```
### Supply state diagram (deposits, collateral, fee redemptions)

Description
The supply state machine covers how liquidity enters and exits the reserve. Deposits mint collateral tokens, withdrawals burn them, and protocol fees accumulate in vaults. Interest accrual continuously updates balances, ensuring the exchange rate between collateral and liquidity reflects time and usage.

Idle: reserve initialized, no activity yet
Depositing: liquidity added → collateral minted
Withdrawing: collateral burned → liquidity redeemed
FeesRedeem: protocol treasury collects accumulated fees
Accruing: background step updating supply/borrow indices

``` mermaid
stateDiagram-v2
  [*] --> Idle

  Idle --> Accruing: init() / last_update.new()
  Accruing --> Ready

  state Ready {
    [*] --> Liquid
    Liquid --> Depositing: compute_depositable_amount_and_minted_collateral\n-> deposit_liquidity()
    Depositing --> Liquid: liquidity.available_amount += x\ncollateral.mint(y)

    Liquid --> Withdrawing: redeem_collateral()
    Withdrawing --> Liquid: collateral.burn(y)\nliquidity.withdraw(x)

    Liquid --> FeesRedeem: calculate_redeem_fees() > 0\nredeem_fees()
    FeesRedeem --> Liquid: fee_vault -> treasury\navailable_amount -= fee_out
  }

  %% periodic
  Accruing --> Accruing: accrue_interest()\n-> compound_interest()
  Ready --> Accruing: accrue_interest() on any mutating op
```


### Borrow state diagram (borrow, repay, indices, fees)
Description
The borrow machine represents user debt lifecycle. Borrowers request liquidity, pay fees, and are bound by reserve limits. Debt grows over time via interest accrual. Repayments reduce outstanding debt and restore available liquidity. Protocol and referrer fees are skimmed along the way.

Legend

Quoting: check borrow size, apply borrow factor & caps

Borrowing: reserve executes loan, deducts available liquidity

Debited: outstanding debt tracked via indices

Repaying: user specifies repayment, system caps to debt size

Credited: reserve updates balances, increasing available liquidity

Accruing: interest compounds across all outstanding borrows
``` mermaid
stateDiagram-v2
  [*] --> Accruing

  Accruing --> Quoting: calculate_borrow()
  Quoting --> Borrowing: passes caps & health checks
  Borrowing --> Debited: liquidity.borrow()
  Debited --> Accruing

  Accruing --> Repaying: calculate_repay()
  Repaying --> Credited: liquidity.repay()
  Credited --> Accruing

  Accruing: accrue_interest()

```

### Risk state diagram (caps, limits, elevation, status, autodeleverage)
The risk state machine shows how governance parameters and utilization caps control reserve behavior. Reserves can be active, hidden, or obsolete, with limits on deposits and borrows. Elevated groups offer preferential terms for correlated assets. Deleveraging and liquidation parameters define how bad debt is handled.

Legend

Active: reserve open for normal activity
Hidden / Obsolete: governance flags, restrict usage
CapsBreached: deposit or borrow exceeds configured cap
BlockBorrow: high utilization blocks new borrows
ElevatedMode: special terms for correlated collateral groups
ParamChange: governance updates to reserve config

``` mermaid
stateDiagram-v2
  [*] --> Active: config.status == Active
  Active --> Hidden: config.status == Hidden
  Active --> Obsolete: config.status == Obsolete
  Hidden --> Active
  Obsolete --> Active

  Active --> CapsBreached: deposit_limit_crossed() || borrow_limit_crossed()
  CapsBreached --> Active: update_*_limit_crossed_timestamp()

  Active --> BlockBorrow: utilization_limit_block_borrowing_above_pct reached
  BlockBorrow --> Active: utilization falls below limit

  Active --> ElevatedMode: is_in_elevation_group == true
  ElevatedMode --> Active

  %% governance/admin (not shown in code here but implied)
  Active --> ParamChange: update_reserve_config(...) 
  ParamChange --> Active
  ```