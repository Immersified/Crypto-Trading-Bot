# Security & Disclosure Policy

This is a **public reference / portfolio** repository for a private trading system. It exists
to demonstrate architecture and engineering — **not** to expose the live system.

## What this repository deliberately excludes

To protect both the trading strategy and any funds under management, this repo contains
**none** of the following:

- Source code of the strategy, execution engine, or connectors
- API keys, secrets, or private/wallet keys
- Wallet or account addresses
- Auth tokens, session cookies, or credentials of any kind
- Database connection strings, hostnames, or internal endpoints
- Notification topics/webhooks or other operational endpoints
- Real account balances, P&L figures, or position data
- Server paths, IPs, or deployment specifics that could aid an attacker

## What it *does* contain

- Architecture and data-flow diagrams (Mermaid)
- Feature- and design-level descriptions
- Technology choices and engineering rationale

## How this repository was assembled

The content was written **by hand from architectural knowledge**, not copied or generated
from the private codebase. No files were imported from the private repository. Before
publishing, the repository was scanned for common secret patterns (keys, addresses, tokens,
connection strings).

## Reporting

If you believe anything sensitive has slipped into this repository, please open an issue
(without including the sensitive value itself) or contact the author directly, and it will be
removed promptly.
