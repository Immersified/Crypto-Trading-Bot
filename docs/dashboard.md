# Monitoring Dashboard (React + API)

A real-time control surface for watching every bot account in one place.

[Back to README](../README.md)

---

## Purpose

An automated trading system that you can't *see* is a liability. The dashboard turns the
database into an at-a-glance operational view: where is my money, what's open, how is each
account performing, and is anything broken?

```mermaid
flowchart LR
    DB[("MongoDB")] --> API
    EXCH["Exchange APIs<br/>(live balances/prices)"] --> API

    subgraph API["Dashboard API · Express + TypeScript"]
        ACC["/account"]
        DBR["/database<br/>(accounts, positions)"]
        ALG["/algorithm"]
        LOG["/logs"]
    end

    subgraph UI["Dashboard UI · React + TypeScript"]
        LOGIN["Login (auth-gated)"]
        OVER["Account overview<br/>+ equity tiles"]
        POS["Positions table<br/>(filter / detail)"]
        GRAPH["P&L / profit graph"]
        SET["Settings + add-profile"]
    end

    API --> UI
    API -. cron .-> NTFY["Push notifications"]
```

---

## Backend API

A small **Express + TypeScript** service exposes read-mostly endpoints over the trading
database plus a few live exchange queries:

| Route | Responsibility |
|---|---|
| `/account` | Account profiles, equity & summary data |
| `/database` | Accounts and positions (e.g. positions grouped by account) |
| `/algorithm` | Algorithm/trade-sheet related data |
| `/logs` | Operational logs surfaced to the UI |

It uses standard hardening middleware (CORS, `helmet`) and runs a **cron job** that scans
for new issues and fires **push notifications** when problems appear.

---

## Frontend

A **React + TypeScript** single-page app, organized by feature:

```mermaid
flowchart TB
    APP["App"] --> CTX["Context providers<br/>(login · global data · positions)"]
    CTX --> HOOKS["Hooks<br/>(useAPI · useRefetchLoop · useIsMobile)"]
    HOOKS --> FEAT

    subgraph FEAT["Features"]
        F1["login"]
        F2["dashboard"]
        F3["settings"]
        F4["add-profile"]
    end

    F2 --> COMP["Components<br/>account overview · positions filter ·<br/>position details · profit graph · setting chips"]
```

### Highlights
- **Live refetch loop** — a polling hook keeps positions and balances fresh without manual
  refresh.
- **Multi-account overview** — every bot account is shown as a tile with its equity and
  status, so the whole fleet is visible at once.
- **Drill-down positions** — an expandable table filters and expands into per-position
  detail (entry, direction, TP/SL, P&L).
- **Profit / equity graphs** — visualize performance over time per account.
- **Auth-gated** — a login context guards the dashboard behind authentication.
- **Mobile-aware** — responsive layout via a dedicated `useIsMobile` hook.

> *Live dashboard screenshots can be dropped in here.*

[Back to README](../README.md)
