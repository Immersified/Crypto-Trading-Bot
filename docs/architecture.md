# Architecture & Data Flow

This document explains how CryptoBot's layers fit together and how a trading decision flows
from raw market data all the way to a live order and back into the dashboard.

[← Back to README](../README.md)

---

## Design principles

The system was built around a few deliberate principles:

- **Separate *deciding* from *doing*.** The Python research layer decides *what* to trade;
  the TypeScript engine decides *how* to execute it reliably. Neither leaks into the other.
- **One source of truth.** MongoDB holds market data, accounts, live positions and trade
  history. Every component reads/writes the same store, so there is no hidden state.
- **Exchange-agnostic execution.** All exchange-specific behaviour hides behind a single
  connector interface, so adding or swapping an exchange touches one module.
- **Fail loud, recover safely.** Operations that can desync (missed fills, overdue orders)
  have explicit reconciliation paths, and a dry-run mode lets every change be tested first.

---

## End-to-end trade lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant MKT as Exchanges
    participant SCR as Scrapers (Py)
    participant DB as MongoDB
    participant ALG as Algorithm (Py)
    participant ENG as Engine (TS)
    participant EXC as Exchange API
    participant DSH as Dashboard

    MKT->>SCR: candles · funding · orderbook
    SCR->>DB: store historical market data
    ALG->>DB: read market data
    ALG->>ALG: feature engineering · model · backtest
    ALG->>DB: write Trade Sheet (signals + TP/SL + size)
    ENG->>DB: load Trade Sheet + account settings
    ENG->>EXC: place entry + TP/SL orders
    EXC-->>ENG: fills / order updates (websocket)
    ENG->>DB: update positions & trade history
    ENG->>EXC: adjust SL / close overdue / reconcile
    DSH->>DB: read positions, P&L, equity
    Note over ENG,DSH: errors raise push notifications
```

---

## Layered view

```mermaid
flowchart LR
    subgraph L1["1 · Data"]
        direction TB
        S1["Scrapers"]
        S2[("Market data<br/>collections")]
        S1 --> S2
    end
    subgraph L2["2 · Research"]
        direction TB
        A1["Feature engineering"]
        A2["Model + backtest"]
        A3["Trade Sheet (CSV)"]
        A1 --> A2 --> A3
    end
    subgraph L3["3 · Execution"]
        direction TB
        E1["CSV executer"]
        E2["Connector (per exchange)"]
        E3["Order lifecycle + websockets"]
        E1 --> E2 --> E3
    end
    subgraph L4["4 · State"]
        direction TB
        D1[("Accounts")]
        D2[("Positions")]
        D3[("Trade history")]
    end
    subgraph L5["5 · Monitoring"]
        direction TB
        M1["REST API"]
        M2["React dashboard"]
        M3["Alerting"]
        M1 --> M2
    end

    L1 --> L2 --> L3 --> L4 --> L5
    L3 -. notifications .-> M3
```

---

## The connector abstraction

Every exchange is wrapped behind one **`AbstractConnector`** interface, so the engine never
talks to an exchange SDK directly. This is the key to supporting Hyperliquid, Bybit and
Bitvavo from a single execution codebase.

```mermaid
classDiagram
    class AbstractConnector {
        <<interface>>
        +setLeverage(coin, margin)
        +makeTpSlOrder(params)
        +modifyTpSlOrder(params)
        +getTickerPrices()
        +getSingleTickerPrice(coin)
        +getWalletBalanceAndEquity()
        +getOrderDetails(orderId)
        +closePosition(orderId, coin)
        +subscribeToOrderChanges(cb)
        +perpsSpotTransfer(params)
    }
    class HyperliquidConnector
    class BybitConnector
    class BitvavoConnector
    AbstractConnector <|.. HyperliquidConnector
    AbstractConnector <|.. BybitConnector
    AbstractConnector <|.. BitvavoConnector
```

**Why it matters:** swapping exchanges, adding a new venue, or running the same strategy on
multiple exchanges is an *additive* change — implement the interface once and the entire
order-lifecycle engine works unchanged.

---

## Data model (conceptual)

The database is organized around **accounts** that own **positions**, with a separate body
of **market data** feeding research. (Field-level schema is intentionally omitted.)

```mermaid
erDiagram
    ACCOUNT ||--o{ POSITION : owns
    ACCOUNT ||--|| SETTINGS : configured-by
    POSITION ||--o{ TRADE_EVENT : records
    MARKET_PAIR ||--o{ CANDLE : has
    MARKET_PAIR ||--o{ FUNDING_RATE : has

    ACCOUNT {
        string profileName
        bool active
        number equity
    }
    POSITION {
        string coin
        string direction
        string status
    }
    SETTINGS {
        string buyCsvFile
        string settingsFile
    }
    MARKET_PAIR {
        string symbol
    }
```

---

## Multi-account model

A single engine invocation can drive **many independent bot accounts** in one run. Each
account has its own keys, trade sheet and settings, and accounts are processed
independently so one failing account never blocks the others.

```mermaid
flowchart TB
    RUN["Engine run"] --> PA{"For each active account"}
    PA --> A1["Account A<br/>own keys · sheet · settings"]
    PA --> A2["Account B"]
    PA --> A3["Account C"]
    A1 --> R1["execute → reconcile"]
    A2 --> R2["execute → reconcile"]
    A3 --> R3["execute → reconcile"]
    R1 & R2 & R3 --> AGG["Aggregate results<br/>(failures isolated per account)"]
```

[← Back to README](../README.md)
