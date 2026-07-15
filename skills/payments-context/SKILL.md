---
name: payments-context
description: "Foundation skill that defines the shared Finding schema, canonical signal taxonomy, dollar-impact contract, and shared oracles (market → timezone, PSP registry, deploy timeline, GMV/AOV lookup) used by every other payments agent. This is not an agent you run — it is a contract every other agent reads before emitting output, so that the Synthesis Agent can dedupe, correlate, and prioritize deterministically. Use this skill whenever you need to: emit a Finding from a specialist agent, look up the canonical name for a signal, ask 'what timezone is this market in', 'what is our current GMV baseline', 'was there a deploy in this window', 'is PSP X in our stack'. Triggers on: 'finding schema', 'signal taxonomy', 'payments context lookup', 'canonical signal name', 'market timezone', 'PSP registry', 'deploy timeline', 'GMV baseline'."
---

# Payments Context

You are the shared context and schema layer for the payments agent ecosystem. You do not detect anything yourself. You define the shape of what other agents emit and provide the shared oracles they call.

Every specialist agent (KPI monitor, fraud analyst, cost optimizer, checkout optimizer, incident agent, reconciliation, partner health, regulatory monitor, storefront audit) must conform to the contracts in this file when producing output. The Synthesis Agent depends on that conformance to correlate, deduplicate, and prioritize.

You do not make findings. You make findings comparable.

## Why this exists

Before this skill, each specialist agent invented its own JSON shape and its own vocabulary for the same idea. Synthesis had to translate. That is where correlation quietly broke — "auth_rate_drop" from one agent and "authorization_declined" from another looked like two problems instead of one.

The contracts below fix that. Every agent uses the same field names, the same severity ladder, the same signal names, and pulls dollar impact from the same GMV oracle.

## The Finding schema

Every specialist agent emits an array of Findings. Every Finding must conform to this shape. Missing required fields = the Synthesis Agent discards it.

```json
{
  "agent": "<agent name, e.g. payments-kpi-monitor>",
  "finding_id": "<stable hash of {agent, signals[], segment, window.start} — same underlying issue produces the same id across runs>",
  "emitted_at": "<ISO 8601>",
  "window": {
    "start": "<ISO 8601>",
    "end": "<ISO 8601>",
    "market_timezone": "<IANA tz, e.g. Europe/Amsterdam — resolved via oracle>"
  },
  "signals": ["<canonical signal names — see taxonomy below>"],
  "segment": {
    "psp": "<psp id from registry, or 'all'>",
    "market": "<ISO 3166-1 alpha-2, or 'global'>",
    "method": "<card | wallet | bnpl | bank_transfer | local | 'all'>",
    "device": "mobile | desktop | all",
    "card_brand": "<visa | mc | amex | ... | 'all'>"
  },
  "metric_paired": {
    "primary":   { "name": "<metric>", "value": <n>, "unit": "<unit>", "baseline": <n>, "delta_pct": <signed n> },
    "counter":   { "name": "<counter metric>", "value": <n>, "unit": "<unit>", "baseline": <n>, "delta_pct": <signed n> }
  },
  "dollar_impact": {
    "amount_usd": <number>,
    "basis": "per_hour | per_day | per_month | one_time",
    "method": "gmv_oracle | psp_report | manual_estimate",
    "confidence": "high | medium | low"
  },
  "severity": "P0 | P1 | P2 | P3 | opportunity | info",
  "confidence": <0.0 - 1.0>,
  "hypothesis": "<one sentence — what likely caused this>",
  "recommended_action": "<one specific next step>",
  "action_owner": "payments_eng | fraud_team | head_of_payments | psp_partner | product | merchant_admin",
  "evidence_refs": ["<pointer to raw data — dashboard URL, query id, event id, log line>"],
  "carry_over_of": "<finding_id of yesterday's finding if this is the same issue continuing, else null>",
  "suppression": {
    "days_active": <int>,
    "acknowledged": true | false,
    "downgrade_after_days": <int, default 3>
  }
}
```

### Required vs optional fields

Required for every Finding: `agent`, `finding_id`, `emitted_at`, `window`, `signals`, `severity`, `confidence`, `hypothesis`, `recommended_action`, `action_owner`.

Required unless the design principle forbids it: `metric_paired` (fraud/KPI/checkout findings must have both primary and counter — never one without the other), `dollar_impact` (every finding needs a dollar figure).

Optional: `carry_over_of` (only if the same issue appeared previously), `suppression` (Synthesis populates this).

## Signal taxonomy

Use these canonical names in the `signals` array. Do not invent new ones without adding them here first. The Synthesis Agent correlation table is keyed on these strings.

### Metric signals
- `success_rate_drop` — payment success rate below baseline
- `success_rate_recovery` — success rate returning to baseline after a drop
- `auth_rate_drop` — issuer authorization rate below baseline
- `auth_rate_recovery` — auth rate returning to baseline
- `latency_p99_spike` — p99 latency above threshold
- `latency_p95_spike` — p95 latency above threshold
- `uptime_breach` — unplanned downtime detected
- `fraud_rate_up` / `fraud_rate_down`
- `false_positive_rate_up` / `false_positive_rate_down`
- `chargeback_rate_up`
- `dispute_win_rate_down`
- `3ds_challenge_rate_up` — 3DS overtriggering
- `3ds_abandonment_up`
- `checkout_abandonment_up` — pre-submit funnel drop-off
- `method_coverage_gap` — preferred method missing in a market
- `cost_per_txn_up`
- `method_mix_shift` — payment method mix moved materially
- `settlement_lag` — PSP payout delay
- `recon_variance_up` — ledger vs bank mismatch

### Change signals (emitted by change-source agents)
- `psp_status_degraded` — PSP status page reports degradation
- `psp_status_recovered`
- `recent_deploy` — code deploy in the incident window
- `feature_flag_flip` — flag change in the incident window
- `fraud_model_retrain` — fraud model retrained in the window
- `checkout_config_changed` — merchant checkout config changed (storefront audit)
- `checkout_extension_deployed` — Shopify checkout extensibility function deployed
- `payment_app_installed` / `payment_app_removed`
- `webhook_endpoint_changed`

### Compliance signals
- `regulatory_deadline_near` — <30 days to a compliance deadline
- `regulatory_deadline_missed`
- `sca_challenge_required_change` — SCA exemption boundary shifted
- `pci_scope_change`

Adding a new signal: append it here, mark it as `added:YYYY-MM-DD`, and open a PR. Do not use unnamed signals in production.

## The dollar impact contract

Every finding needs a dollar figure. Do not compute it independently — call the GMV oracle so numbers are comparable across agents.

```
dollar_impact.amount_usd = f(affected_volume, avg_order_value, effect_size)
```

Formulas by finding type:

| Finding type | Formula |
|---|---|
| Auth/success rate drop | `hourly_gmv × market_share × delta_pct × recoverable_fraction` (default recoverable = 0.6) |
| Fraud model tightening | `blocked_good_volume × aov − prevented_fraud_volume × aov` (net) |
| Method coverage gap | `market_volume × preference_share × aov × (1 − fallback_rate)` |
| Latency-driven abandonment | `affected_sessions × (latency_delta_s × 0.015) × aov` (1.5% conversion loss per +1s) |
| Settlement lag | `delayed_amount × wacc_daily × delay_days` (working capital cost, not lost revenue) |
| Recon variance | `raw_variance` for headline, split into `timing` vs `true_loss` in narrative |

If a required input is missing from the GMV oracle, emit the finding with `dollar_impact.method: "manual_estimate"` and `confidence: "low"`. Do not fabricate.

## Shared oracles

Each specialist agent should call these oracles rather than hard-coding values. Concrete implementations vary by workspace — the contract is what matters.

### `oracle.markets(market_code)`
Returns:
```json
{ "code": "NL", "name": "Netherlands", "timezone": "Europe/Amsterdam",
  "currency": "EUR", "preferred_methods": ["ideal", "card", "paypal"],
  "sca_regime": "psd2", "launch_date": "2019-06-01" }
```
Used by KPI monitor (baselines by local time), checkout optimizer (coverage gaps), regulatory monitor.

### `oracle.psps()`
Returns the current PSP registry:
```json
[
  { "id": "stripe", "regions": ["global"], "settlement_sla_days": 2, "renewal_date": "2026-11-01" },
  { "id": "adyen",  "regions": ["EU","APAC"], "settlement_sla_days": 1, "renewal_date": "2026-03-15" }
]
```
Used by cost optimizer, partner health, incident agent, reconciliation.

### `oracle.gmv(window, segment)`
Returns GMV and AOV for the requested window and segment (market/method/psp). Every dollar-impact calculation must call this — do not maintain a private copy.

### `oracle.changes(window)`
Returns change events in the window from all sources:
```json
[
  { "type": "recent_deploy", "at": "2026-07-14T02:14:00Z", "service": "checkout-api", "sha": "abc123", "ref": "https://..." },
  { "type": "feature_flag_flip", "at": "...", "flag": "wallets_first_ordering", "value": true },
  { "type": "fraud_model_retrain", "at": "...", "model": "cnp_v3", "ref": "..." },
  { "type": "checkout_config_changed", "at": "...", "field": "payment_methods.NL", "diff": "..." }
]
```
This oracle is what powers "success rate drop + recent deploy" correlation. If your workspace has no change source wired, that pattern is dead — flag it clearly.

### `oracle.suppression(finding_id)`
Returns state of the same finding_id across previous briefings — days active, acknowledged, snoozed, resolved. Used by Synthesis to downgrade or auto-mute repeats.

## Suppression rules

Findings that recur without action rot the briefing. Apply these rules in Synthesis:

| Age | Ack'd? | Action |
|---|---|---|
| Day 1 | — | Surface at declared severity |
| Day 2 | No | Surface, add "still open" tag |
| Day 3 | No | Downgrade one severity level (P1 → P2) |
| Day 5 | No | Auto-mute, move to weekly summary only |
| Any | Yes | Respect ack — suppress until owner unsnoozes or state changes materially |

Material state change (>2σ shift in the primary metric) breaks any suppression and re-surfaces at full severity.

## Deploy-correlation policy

Synthesis must NOT emit the correlated finding "success rate drop + recent deploy" unless the change oracle returned an actual deploy event within `[incident_start - 30min, incident_start + 5min]`. Absence of a deploy record is not proof of no deploy — but do not manufacture the correlation from timing alone.

## Timezone policy

All `window.start` / `window.end` values are UTC ISO 8601. `window.market_timezone` carries the local timezone for that segment. KPI monitor's "Saturday night ≠ Monday morning" rule uses the local timezone, not UTC — apply `market_timezone` before selecting the baseline slot.

## Emitting a Finding — worked example

Bad (pre-context, ambiguous, missing dollar figure, missing counter metric):
```json
{ "agent": "kpi", "problem": "auth rate low on Stripe", "severity": "high" }
```

Good (post-context, dedupe-friendly, correlation-ready):
```json
{
  "agent": "payments-kpi-monitor",
  "finding_id": "kpi:auth_rate_drop:stripe:NL:2026-07-14T14:00Z",
  "emitted_at": "2026-07-14T14:22:00Z",
  "window": { "start": "2026-07-14T14:00:00Z", "end": "2026-07-14T14:20:00Z", "market_timezone": "Europe/Amsterdam" },
  "signals": ["auth_rate_drop", "psp_status_degraded"],
  "segment": { "psp": "stripe", "market": "NL", "method": "card", "device": "all", "card_brand": "all" },
  "metric_paired": {
    "primary": { "name": "auth_rate", "value": 90.8, "unit": "pct", "baseline": 94.1, "delta_pct": -3.5 },
    "counter": { "name": "false_positive_rate", "value": 2.1, "unit": "pct", "baseline": 2.0, "delta_pct": 5.0 }
  },
  "dollar_impact": { "amount_usd": 4200, "basis": "per_hour", "method": "gmv_oracle", "confidence": "high" },
  "severity": "P1",
  "confidence": 0.85,
  "hypothesis": "Stripe issuer connectivity degradation in EU — matches Stripe status page event at 14:03Z",
  "recommended_action": "Shift NL card volume to Adyen for the next 60 minutes; revert when Stripe status recovers",
  "action_owner": "payments_eng",
  "evidence_refs": [
    "https://status.stripe.com/incidents/abc123",
    "grafana://payments/auth-rate?psp=stripe&market=NL&t=2026-07-14T14:00Z"
  ],
  "carry_over_of": null
}
```

That output is directly correlatable, priced, and actionable. That is the standard.

## Failure modes to avoid

- **Never invent a new signal name inline** — add it to the taxonomy first
- **Never emit a Finding without paired metrics** where the design principle requires them (fraud, KPI, checkout)
- **Never compute dollar impact from a private GMV copy** — call the oracle, or mark the method as `manual_estimate` and confidence `low`
- **Never omit `finding_id`** — Synthesis needs stable ids to detect carry-overs and apply suppression
- **Never bypass the timezone oracle** — comparing 20:00 UTC in NL to 20:00 UTC in JP is meaningless
- **Never fabricate deploy correlations** from timing alone — the change oracle must confirm
