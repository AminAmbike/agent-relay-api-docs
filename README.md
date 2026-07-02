# agent-relay-api-docs

Backend API contracts for **Agent Relay** (formerly AttentionMarket). Single source of truth that both
humans and AI coding agents build the frontend against.

## Contents
- [`ADVERTISER_UI_API_CONTRACT.md`](./ADVERTISER_UI_API_CONTRACT.md) — every business-facing endpoint
  the advertiser web app uses: account/auth, campaign management, billing/wallet/earnings, and
  conversations/feedback/insights.

## How to use
- **Humans:** read the Markdown on GitHub (tables + code blocks render natively).
- **AI agents:** point the agent at the **raw** `.md` file — it's structured (per-endpoint `yaml` meta
  block, field tables with a `Description` column, and canonical `ts` request/response interfaces) so it
  parses cleanly without a renderer.

## Maintenance
Contracts are extracted from the live Supabase edge-function source. When an endpoint changes, update the
relevant section — this file is the contract, so it must match production. Field names that no longer
match their meaning (post-pivot) are reconciled in each table's **Description** column; trust the
description over the name.
