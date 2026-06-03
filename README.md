# CryptoBot — Algorithmic Crypto Trading System

> **Reference overview of a production algorithmic trading platform.**
> This repository is a *showcase*. It documents the architecture, design decisions and
> features of a privately-developed crypto trading system through diagrams and write-ups.
> **It intentionally contains no source code, credentials, or proprietary strategy logic.**

---

## 📌 At a glance

CryptoBot is a **fully automated, multi-exchange algorithmic trading platform** for crypto
perpetual futures. It runs unattended on a server, turns model-generated signals into live
orders across exchanges, tracks every position in a database, and exposes a real-time
dashboard for monitoring performance.

| | |
|---|---|
| **Domain** | Algorithmic trading / quantitative finance |
| **Style** | Systematic, model-driven perpetual-futures trading |
| **Exchanges** | Hyperliquid (primary), Bybit, Bitvavo |
| **Markets** | BTC, ETH, SOL, AVAX, ADA, LINK, DOT, LTC … (USDC-quoted perps) |
| **Run mode** | 24/7 unattended on a Linux VPS, managed by PM2 |
| **Scale** | Multi-account / multi-profile — many bots from one engine |

**Built end-to-end by a single developer:** research & strategy modelling, data
engineering, trade-execution engine, exchange integrations, infrastructure, and a
full monitoring dashboard.

---

## 🧠 What problem does it solve?

Manual crypto trading doesn't scale: it can't watch the market 24/7, it's emotional, and it
can't consistently apply a tested strategy. CryptoBot closes that gap with a clean
separation between **deciding** and **doing**:

1. **Research & signal generation (Python)** — historical market data is collected,
   features are engineered, and a model produces a forward-looking *trade sheet*: which
   asset to trade, in which direction, at what size, with which take-profit and stop-loss.
2. **Execution (TypeScript)** — a robust engine reads those trade sheets and manages the
   full order lifecycle on the exchange: placing orders, attaching TP/SL, reacting to
   fills over websockets, and reconciling state.
3. **Monitoring (React + API)** — a dashboard shows live positions, realized/unrealized
   P&L, and per-account performance at a glance, with alerting when something needs
   attention.

---

## 🏗️ System architecture

```mermaid
flowchart TB
    subgraph EXCH["🌐 Exchanges & Market Data"]
        BIN["Binance<br/>(candles + funding)"]
        BYB["Bybit"]
        HL["Hyperliquid"]
    end

    subgraph DATA["📥 Data Pipeline (Python)"]
        SCRAPE["Market-data scrapers<br/>candles · funding · orderbook"]
    end

    subgraph ALGO["🧠 Algorithm Layer (Python)"]
        MODEL["Signal model + backtesting<br/>→ generates Trade Sheets (CSV)"]
    end

    DB[("🗄️ MongoDB<br/>market data · accounts ·<br/>positions · trade history")]

    subgraph ENGINE["⚙️ Trading Engine (TypeScript)"]
        EXEC["Order lifecycle engine<br/>connectors · TP/SL · websockets"]
    end

    subgraph MON["📊 Monitoring"]
        API["Dashboard API<br/>(Express)"]
        UI["Dashboard UI<br/>(React)"]
        ALERT["Alerting<br/>(push notifications)"]
    end

    BIN & BYB & HL --> SCRAPE --> DB
    DB --> MODEL --> DB
    MODEL -->|Trade Sheets| ENGINE
    ENGINE <-->|orders · fills · balances| EXCH
    ENGINE <--> DB
    DB --> API --> UI
    ENGINE --> ALERT
    API --> ALERT

    classDef py fill:#3776AB,stroke:#fff,color:#fff;
    classDef ts fill:#3178C6,stroke:#fff,color:#fff;
    classDef store fill:#13AA52,stroke:#fff,color:#fff;
    classDef ext fill:#555,stroke:#fff,color:#fff;
    class SCRAPE,MODEL py;
    class EXEC,API,UI ts;
    class DB store;
    class BIN,BYB,HL ext;
```

> **Clean separation of concerns:** Python owns *research & signals*, TypeScript owns
> *execution & reliability*, MongoDB is the single source of truth, and the dashboard is a
> read-mostly view on top. Each layer can evolve independently.

---

## ✨ Key features

### Trading & execution
- **Multi-exchange via a common connector interface** — Hyperliquid, Bybit and Bitvavo all
  sit behind one abstraction (`setLeverage`, `makeTpSlOrder`, `closePosition`,
  `getWalletBalanceAndEquity`, websocket order subscriptions …), so the engine is
  exchange-agnostic.
- **Full order lifecycle management** — entries, take-profit & stop-loss orders, overdue-order
  cleanup, and reconciliation of missed/partial fills.
- **Live reaction over websockets** — order-change subscriptions keep database state in sync
  with the exchange in real time, and dynamically adjust stop-losses.
- **Multi-account / multi-profile** — one engine drives many independent bot accounts, each
  with its own keys, trade sheet and settings.
- **Dry-run mode** — every action can be simulated without touching the live exchange.

### Data & research
- **Automated market-data scrapers** — historical 1-minute candles and funding rates
  (Binance), plus Bybit and Hyperliquid history/orderbook collection into MongoDB.
- **Backtesting & calibration** — strategies are validated over years of historical data
  across many assets before going live (see results below).

### Operations & monitoring
- **Real-time dashboard** — live positions, realized & unrealized P&L, per-account equity
  curves, and configurable settings.
- **Push-notification alerting** — operational errors raise urgent push notifications so
  issues are caught immediately.
- **Process management & deployment** — runs as managed PM2 processes on a Linux VPS behind
  nginx.

---

## 🧩 Component deep-dives

| Component | What it does | Details |
|---|---|---|
| 🧠 **Algorithm layer** | Research, signal modelling & backtesting → Trade Sheets | [`docs/algorithm.md`](docs/algorithm.md) *(author-maintained)* |
| 📥 **Data pipeline & infra** | Market-data scrapers, storage, deployment | [`docs/data-pipeline.md`](docs/data-pipeline.md) |
| 📊 **Dashboard (React + API)** | Real-time monitoring & control surface | [`docs/dashboard.md`](docs/dashboard.md) |
| 🏛️ **Architecture & data model** | How the layers fit together | [`docs/architecture.md`](docs/architecture.md) |

---

## 📈 Results & validation

The strategy is validated through extensive backtesting and calibration runs across multiple
assets (ETH, SOL, BTC, AVAX, ADA, LINK, DOT, LTC) and market regimes (bull / bear /
ranging). Each candidate configuration is scored on risk-adjusted metrics — **Sharpe ratio,
rate-of-return, and drawdown** — and only validated configurations are promoted to live
trading.

> 📷 *Backtest equity curves and performance tables can be added here as screenshots.*

---

## 🛠️ Tech stack

| Layer | Technologies |
|---|---|
| **Execution engine** | TypeScript · Node.js · WebSockets · ethers.js (Hyperliquid signing) · Joi |
| **Algorithm & data** | Python · pandas · NumPy · scikit-learn / joblib · MongoDB driver |
| **Data store** | MongoDB (market data, accounts, positions, trade history) |
| **Dashboard** | React · TypeScript · Express · REST API |
| **Infrastructure** | Linux VPS · PM2 · nginx · push-notification alerting |
| **Tooling** | Jest · ts-jest · ESLint / Prettier |

---

## 🔒 A note on this repository

This is a **public reference / portfolio** repository. The actual trading system is
private. To protect both the strategy and any funds, this repo deliberately:

- contains **no source code** of the strategy or execution engine,
- contains **no API keys, private keys, wallet addresses, tokens, cookies or endpoints**,
- shows only **architecture, design and feature-level** information.

See [`SECURITY.md`](SECURITY.md) for the policy followed when assembling this repository.

---

<sub>Architecture diagrams are written in <a href="https://mermaid.js.org/">Mermaid</a> and
render automatically on GitHub.</sub>
