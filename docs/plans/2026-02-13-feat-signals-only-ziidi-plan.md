---
title: "feat: Tailor Mahoraga into a signals-only system for manual Ziidi trading"
type: "feat"
date: "2026-02-13"
status: "planned"
owner: "codex"
---

# feat: Tailor Mahoraga into a signals-only system for manual Ziidi trading

## Overview

Refactor Mahoraga from autonomous Alpaca execution to a **signals-only advisory engine** for manual order placement in Ziidi.

The plan preserves Mahoraga’s strongest architecture pieces (continuous Durable Object loop, structured LLM research, policy checks, persistent storage) while hard-disabling order submission and replacing execution with alert + audit-trail workflows.

## Why this is the right boundary

- Ziidi appears designed for in-app human authorization and manual order flow.
- Mahoraga already has clean separation between gather/research/policy/execute, so execution can be replaced without rewriting the whole system.
- A hard safety boundary (execution mode flag + MCP submit choke-point) prevents accidental broker execution.

## Current codebase anchors (what we will change)

- Durable Object orchestration: `src/durable-objects/mahoraga-harness.ts`
- Strategy contract and broker hooks: `src/strategy/types.ts`
- MCP trading tools (`orders-preview`, `orders-submit`, `positions-*`, `technicals-*`): `src/mcp/agent.ts`
- Policy-wrapped broker path: `src/core/policy-broker.ts`
- Agent config schema and tests: `src/schemas/agent-config.ts`, `src/schemas/agent-config.test.ts`
- Persistent storage + D1 migrations for approvals/logging: `src/storage/d1/**`, `migrations/**`

## Scope

### In scope (MVP)

1. Add execution-mode control (`signals_only`, `paper`, `live`) with `signals_only` as default.
2. Hard-disable MCP submit tools in signals-only mode.
3. Replace strategy-level buy/sell actions with recommendation emission + alert sinks.
4. Add signal ledger tables and APIs for reproducible audit trail.
5. Add NSE-compatible market-data adapter skeleton + technical indicators pipeline.
6. Add alert routing (Telegram first, plus pluggable sinks for WhatsApp/Sheets).
7. Add portfolio-mirror input (manual holdings) to support HOLD/SELL recommendations.

### Out of scope (MVP)

- Direct Ziidi automation or PIN/OTP interaction.
- Licensed NSE low-latency feed protocols (MITCH/FIX implementation).
- Fully productionized backtesting engine integration.

## Proposed architecture (signals-only)

1. Gather market + sentiment data.
2. Build candidates using deterministic filters (liquidity/regime/indicators).
3. Run LLM research with strict structured outputs.
4. Apply hard risk and sizing gates (LLM cannot override).
5. Emit recommendation ticket + route alerts.
6. Persist full decision ledger for replay/evaluation.

## Implementation plan (work packages)

### WP1 — Execution safety boundary (week 1)

**Goal:** impossible to place orders when in `signals_only` mode.

Tasks:

- Extend config schema with:
  - `execution_mode: "signals_only" | "paper" | "live"`
  - default: `signals_only`
- Enforce in MCP tools:
  - `orders-submit` and `options-order-submit` return deterministic `EXECUTION_DISABLED` error in `signals_only`.
  - keep `orders-preview` allowed for risk validation and recommendation shaping.
- Enforce in policy broker adapter:
  - `ctx.broker.buy/sell` become no-op false returns (or throw typed disabled error) in `signals_only`.
- Update tests for all paths.

Acceptance criteria:

- Any submit call in `signals_only` cannot reach broker provider.
- Preview remains functional with policy evaluation.
- Unit tests prove submit path is blocked.

### WP2 — Harness/strategy refactor to recommendations (week 1-2)

**Goal:** replace buy/sell execution behavior with recommendation generation.

Tasks:

- Introduce recommendation model:
  - `verdict`, `confidence_raw`, `confidence_capped`, `entry_ref`, `risk`, `sizing`, `validity`, `evidence`, `unknowns`, `guardrail_flags`.
- Refactor strategy runtime so selection output can be persisted/emitted without placing trades.
- Replace direct execution calls in harness cycle with:
  1) `createRecommendation(...)`
  2) `persistRecommendation(...)`
  3) `dispatchAlerts(...)`

Acceptance criteria:

- Alarm cycles produce recommendation artifacts even with zero broker credentials.
- Existing research and policy phases still run.
- No callsite remains that can submit orders in signals-only.

### WP3 — Alerting channels + delivery guarantees (week 2)

**Goal:** human-readable, actionable tickets.

Tasks:

- Add alert abstraction (`src/notifications/`):
  - Telegram sink first (`sendMessage`).
  - stubs/interfaces for WhatsApp and Google Sheets append sink.
- Build ticket templates:
  - side, symbol, reference price/time, stop/target, max_cash/max_shares, validity window, confidence cap, invalidation conditions.
- Add retry/backoff + idempotency key for duplicate suppression.

Acceptance criteria:

- Recommendation appears in Telegram within one cycle.
- Duplicate alerts for same recommendation ID are suppressed.

### WP4 — Signal ledger and manual portfolio mirror (week 2-3)

**Goal:** auditable decisions + optional HOLD/SELL logic from manual holdings.

Tasks:

- Add D1 migration tables:
  - `recommendations`
  - `recommendation_alerts`
  - `manual_portfolio_snapshots`
- Add query modules in `src/storage/d1/queries/`.
- Add API/MCP surface for portfolio mirror ingest (CSV/JSON payload).
- Add dashboard endpoint readiness (can be UI-followup).

Acceptance criteria:

- Every recommendation can be traced to input snapshot + model output.
- Manual holdings can be uploaded and consumed in next cycle.

### WP5 — NSE data adapter + indicators baseline (week 3-4)

**Goal:** reliable candidate generation for NSE symbols.

Tasks:

- Introduce `src/providers/nse/` provider interfaces:
  - symbol master
  - EOD snapshots
  - optional intraday abstraction
- Build canonical symbol mapping and validation.
- Implement indicators in technicals flow (SMA/EMA/RSI/MACD/ATR baseline).
- Tag every datapoint with source + latency/provenance.

Acceptance criteria:

- System can generate recommendations from NSE source data only.
- Indicators are available to prompts and ledger records.

## Risk and guardrails

- Hard cap recommendation confidence unless multi-source confirmation passes.
- Liquidity gate blocks low-turnover symbols.
- Fail-safe mode: stale/missing critical data downgrades verdict to WAIT.
- Explicit non-automation policy for Ziidi execution stored in docs/config.

## Testing strategy

1. **Unit tests**
   - config parsing for execution mode
   - MCP submit blocked behavior
   - policy broker short-circuiting
   - recommendation schema validation
2. **Integration tests**
   - alarm cycle in `signals_only` produces ledger row + alert dispatch
   - no broker submit calls emitted
3. **Replay tests (phase 2 prep)**
   - deterministic run from fixed historical fixture

## Delivery phases and milestones

### Milestone M1 (end week 1)
- Execution-mode guardrails merged.
- Submit tools blocked in signals-only.
- Regression tests green.

### Milestone M2 (end week 2)
- Recommendation pipeline + Telegram alerts live.
- Ledger persistence available.

### Milestone M3 (end week 4)
- NSE adapter baseline + indicator-driven candidate flow.
- Manual portfolio mirror integrated.

## Open questions to resolve before implementation kickoff

1. Which NSE feed tier is available for MVP (public snapshot vs paid EOD)?
2. What exact manual sizing constraints should default to (e.g., max cash per idea)?
3. Preferred primary alert channel in production: Telegram or WhatsApp?
4. Should dashboard work be bundled into MVP or deferred to phase 2?

## Definition of done (MVP)

- System runs continuously and emits signals-only recommendations.
- No code path can place broker orders when `execution_mode=signals_only`.
- Alerts are actionable for manual Ziidi execution.
- Full recommendation audit trail is queryable from D1.
- Core flows covered by unit/integration tests.
