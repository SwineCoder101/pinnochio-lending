# MVP Sequence Diagrams: Lending Protocol Data & Money Flow

## Overview

This document provides sequence diagrams for the data flow and money flow in a lending protocol, focusing on the interactions between borrowers, lenders, and the protocol infrastructure. The diagrams capture the usage of obligation reserves and lending markets, showing what happens when liquidity enters the market and when borrowers create obligations.

## Architecture Components

- **Lending Market**: The primary protocol contract managing reserves and obligations
- **Reserve**: Individual asset pools with liquidity, borrow positions, and risk parameters
- **Obligation**: A borrower's position containing collateral and debt across multiple reserves
- **Lender**: Supplies liquidity to reserves in exchange for interest
- **Borrower**: Deposits collateral and borrows against it
- **Oracle**: Provides price feeds for collateral and borrowed assets
- **Liquidator**: Monitors and liquidates unhealthy positions

---

## Diagram 1: Lender Money Flow - Entering the Market

This diagram shows the complete flow when a lender supplies liquidity to a reserve, including interest accrual and potential yield farming.

```mermaid
sequenceDiagram
    participant L as Lender
    participant LM as Lending Market
    participant R as Reserve
    participant F as Farm/Strategy
    participant T as Treasury
    participant O as Oracle

    Note over L,O: Lender enters the market

    L->>O: Request asset price
    O-->>L: Current price data

    L->>LM: supply_liquidity(amount, asset)
    LM->>R: check_reserve_status()
    R-->>LM: Reserve active, within caps

    LM->>R: calculate_supply_amount(amount)
    R-->>LM: Supply amount & fees

    LM->>R: deposit_liquidity(amount)
    R->>R: Update available_liquidity += amount
    R->>R: Update total_supply += amount
    R->>R: Mint collateral_tokens to lender

    Note over R: Interest accrual begins
    R->>R: accrue_interest()
    R->>R: Update supply_index
    R->>R: Update borrow_index

    alt Reserve has active farm
        R->>F: rebalance_to_farm(excess_liquidity)
        F-->>R: Farm tokens received
        R->>R: Update farm_balance
    end

    R->>T: Collect protocol_fee
    T->>T: Update treasury_balance

    R-->>LM: Supply successful
    LM-->>L: Collateral tokens minted
    L->>L: Track position & interest accrual

    Note over L,T: Ongoing: Interest accrues continuously
    loop Every block
        R->>R: accrue_interest()
        R->>R: compound_interest()
        R->>F: harvest_yield() if applicable
    end
```

---

## Diagram 2: Lender Data Flow - Market Entry

This diagram shows the data structures and state changes when a lender enters the market.

```mermaid
sequenceDiagram
    participant L as Lender
    participant LM as Lending Market
    participant R as Reserve
    participant DB as Database/State
    participant C as Config

    Note over L,C: Data flow for lender market entry

    L->>LM: supply_liquidity(amount, asset)
    
    LM->>DB: load_reserve_state(asset)
    DB-->>LM: Reserve data
    
    LM->>C: get_reserve_config(asset)
    C-->>LM: Config parameters
    
    LM->>LM: validate_supply_caps()
    LM->>LM: calculate_supply_apy()
    LM->>LM: compute_collateral_amount()
    
    LM->>DB: update_reserve_state()
    Note over DB: available_liquidity += amount<br/>total_supply += amount<br/>supply_index updated
    
    LM->>DB: create_supply_position()
    Note over DB: lender_address<br/>collateral_amount<br/>supply_index_snapshot<br/>timestamp
    
    LM->>DB: update_global_metrics()
    Note over DB: total_tvl += amount<br/>reserve_utilization updated<br/>interest_rates updated
    
    DB-->>LM: State updated
    LM-->>L: Position created successfully
    
    Note over L,C: Position tracking begins
    loop Continuous monitoring
        LM->>DB: accrue_interest()
        DB->>DB: Update supply_index
        DB->>DB: Update position_balance
    end
```

---

## Diagram 3: Borrower Money Flow - Creating Obligation

This diagram shows the complete flow when a borrower creates an obligation, deposits collateral, and borrows against it.

```mermaid
sequenceDiagram
    participant B as Borrower
    participant LM as Lending Market
    participant R1 as Collateral Reserve
    participant R2 as Borrow Reserve
    participant O as Obligation
    participant Or as Oracle
    participant L as Liquidator

    Note over B,L: Borrower creates obligation

    B->>Or: Request collateral price
    Or-->>B: Collateral price data

    B->>LM: init_obligation()
    LM->>O: create_obligation_account()
    O-->>LM: Obligation created

    B->>LM: deposit_collateral(collateral_amount, asset)
    LM->>R1: check_collateral_reserve()
    R1-->>LM: Reserve active, collateral enabled

    LM->>R1: calculate_collateral_value()
    R1-->>LM: Collateral value & LTV

    LM->>R1: deposit_collateral(amount)
    R1->>R1: Update collateral_supply += amount
    R1->>R1: Mint collateral_tokens to borrower

    LM->>O: update_collateral_balance()
    O->>O: collateral_balance += amount
    O->>O: collateral_value += value

    Note over B,L: Borrower requests loan
    B->>Or: Request borrow asset price
    Or-->>B: Borrow asset price

    B->>LM: borrow_liquidity(borrow_amount, asset)
    LM->>R2: check_borrow_reserve()
    R2-->>LM: Reserve active, within borrow_caps

    LM->>LM: calculate_health_factor()
    LM->>LM: validate_borrow_limits()

    LM->>R2: execute_borrow(amount)
    R2->>R2: Update available_liquidity -= amount
    R2->>R2: Update total_borrows += amount
    R2->>R2: Mint debt_tokens to borrower

    LM->>O: update_debt_balance()
    O->>O: debt_balance += amount
    O->>O: debt_value += value

    LM->>O: calculate_health_factor()
    O-->>LM: Health factor > 1.0 (healthy)

    LM-->>B: Loan successful
    B->>B: Track obligation & debt

    Note over B,L: Ongoing monitoring
    loop Continuous monitoring
        LM->>O: monitor_health_factor()
        alt Health factor < liquidation_threshold
            LM->>L: trigger_liquidation()
            L->>L: Execute liquidation
        end
    end
```

---

## Diagram 4: Borrower Data Flow - Obligation Management

This diagram shows the data structures and state changes when a borrower manages their obligation.

```mermaid
sequenceDiagram
    participant B as Borrower
    participant LM as Lending Market
    participant O as Obligation
    participant R1 as Collateral Reserve
    participant R2 as Borrow Reserve
    participant DB as Database/State
    participant Or as Oracle

    Note over B,Or: Data flow for borrower obligation management

    B->>LM: init_obligation()
    LM->>DB: create_obligation_record()
    Note over DB: obligation_id<br/>borrower_address<br/>created_timestamp<br/>status: active

    B->>LM: deposit_collateral(amount, asset)
    LM->>Or: get_price(asset)
    Or-->>LM: Asset price
    
    LM->>DB: update_obligation_collateral()
    Note over DB: collateral_assets[] += asset<br/>collateral_amounts[] += amount<br/>collateral_values[] += value

    LM->>R1: update_collateral_reserve()
    R1->>DB: update_reserve_state()
    Note over DB: collateral_supply += amount<br/>total_collateral_value += value

    B->>LM: borrow_liquidity(amount, asset)
    LM->>Or: get_price(asset)
    Or-->>LM: Asset price

    LM->>LM: calculate_health_factor()
    Note over LM: health_factor = total_collateral_value / total_debt_value

    LM->>DB: update_obligation_debt()
    Note over DB: debt_assets[] += asset<br/>debt_amounts[] += amount<br/>debt_values[] += value

    LM->>R2: update_borrow_reserve()
    R2->>DB: update_reserve_state()
    Note over DB: total_borrows += amount<br/>available_liquidity -= amount

    LM->>DB: update_obligation_health()
    Note over DB: health_factor = calculated<br/>liquidation_threshold = config<br/>status = healthy/unhealthy

    DB-->>LM: Obligation updated
    LM-->>B: Transaction successful

    Note over B,Or: Ongoing obligation monitoring
    loop Every block
        LM->>DB: accrue_interest()
        DB->>DB: Update debt_balances
        DB->>DB: Update collateral_values
        DB->>DB: Recalculate health_factor
        
        alt Health factor < liquidation_threshold
            DB->>DB: status = liquidatable
            LM->>LM: trigger_liquidation_available()
        end
    end
```

---

## Key Data Structures

### Reserve State
```rust
struct Reserve {
    // Liquidity management
    available_liquidity: u64,
    total_supply: u64,
    total_borrows: u64,
    
    // Interest accrual
    supply_index: u128,
    borrow_index: u128,
    last_update_timestamp: i64,
    
    // Risk parameters
    ltv: u16,
    liquidation_threshold: u16,
    liquidation_bonus: u16,
    
    // Caps and limits
    borrow_cap: u64,
    deposit_cap: u64,
    
    // Farm integration
    farm_address: Option<Pubkey>,
    farm_balance: u64,
}
```

### Obligation State
```rust
struct Obligation {
    // Borrower info
    borrower: Pubkey,
    created_at: i64,
    status: ObligationStatus,
    
    // Collateral tracking
    collateral_assets: Vec<Pubkey>,
    collateral_amounts: Vec<u64>,
    collateral_values: Vec<u64>,
    
    // Debt tracking
    debt_assets: Vec<Pubkey>,
    debt_amounts: Vec<u64>,
    debt_values: Vec<u64>,
    
    // Health metrics
    health_factor: u64,
    liquidation_threshold: u64,
}
```

### Supply Position
```rust
struct SupplyPosition {
    lender: Pubkey,
    reserve: Pubkey,
    collateral_amount: u64,
    supply_index_snapshot: u128,
    last_update_timestamp: i64,
}
```

---

## Money Flow Summary

### Lender Flow
1. **Entry**: Lender supplies liquidity → Reserve receives tokens → Collateral tokens minted
2. **Interest**: Continuous accrual → Supply index updates → Position value grows
3. **Yield**: Optional farm integration → Excess liquidity deployed → Additional yield earned
4. **Exit**: Collateral tokens burned → Reserve liquidity returned → Interest paid

### Borrower Flow
1. **Setup**: Obligation created → Collateral deposited → Collateral value tracked
2. **Borrow**: Loan requested → Health factor calculated → Liquidity borrowed
3. **Monitoring**: Continuous health checks → Interest accrual → Position monitoring
4. **Management**: Repayments → Additional collateral → Debt reduction

### Protocol Flow
1. **Fees**: Protocol fees collected → Treasury balance increased
2. **Risk**: Health monitoring → Liquidation triggers → Bad debt prevention
3. **Liquidity**: Reserve management → Farm integration → Yield optimization
4. **Governance**: Parameter updates → Risk management → Protocol evolution

---

## Integration Points

- **Oracle Integration**: Price feeds for all assets
- **Farm Integration**: Yield optimization for excess liquidity
- **Liquidation System**: Automated position monitoring and liquidation
- **Governance**: Parameter updates and risk management
- **Analytics**: Position tracking and protocol metrics

This MVP provides a comprehensive view of the lending protocol's data and money flows, showing how lenders and borrowers interact with the system and how the protocol manages risk and liquidity.
