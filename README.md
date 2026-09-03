# Payments Agent Ecosystem — Shopify + Stripe

Six specialized agents that monitor, tune, and reconcile payments on a Shopify storefront running Stripe (via Shopify Payments or as a direct gateway), plus one synthesis agent that turns their raw signals into a briefing a human can actually act on.

The design goal is the opposite of most monitoring setups: fewer, better findings. Each agent produces structured Findings that share a schema. The Synthesis Agent decides what's worth a human's attention, correlates signals across agents, and attaches a dollar figure to everything it surfaces.

This is a deliberately narrow build. If you're on multi-PSP or non-Shopify infrastructure, see [Not the right fit if…](#not-the-right-fit-if-) below.

## Architecture

```
                    ┌─────────────────────┐
                    │   Synthesis Agent   │   editorial layer
                    │  (one story, not    │   correlates + quantifies
                    │   three alerts)     │
                    └──────────▲──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────┴────────┐   ┌─────────┴────────┐   ┌─────────┴────────┐
│  Detection     │   │  Optimization    │   │  Verification    │
│                │   │                  │   │                  │
│  Health        │   │  Checkout        │   │  Reconciliation  │
│  Radar Tuner   │   │  Storefront Audit│   │                  │
└────────────────┘   └──────────────────┘   └──────────────────┘
```

| Layer | Agents | Purpose |
| --- | --- | --- |
| **Detection** | Health, Radar Tuner | Anomaly detection, incident triage, Stripe Radar calibration |
| **Optimization** | Checkout, Storefront Audit | Pre-submit funnel (PostHog), merchant-side config drift (Shopify CLI) |
| **Verification** | Reconciliation | Stripe payout ↔ Shopify order ↔ bank ↔ ledger |
| **Synthesis** | Synthesis | Correlates the five specialist outputs, produces daily/weekly briefings |

## Agents

| Agent | Schedule | What it does |
| --- | --- | --- |
| `payments-health` | Hourly + real-time | Anomaly detection and incident triage. Merges classic KPI monitoring and incident response into one agent — no routing decisions to make on a single-processor stack |
| `payments-radar-tuner` | Hourly + triggered | Tunes Stripe Radar. Balances fraud rate against false-positive rate (uses Shopify customer history for the cheap FP proxy). Recommends exact Radar rule changes |
| `payments-checkout` | Daily | Owns the pre-submit funnel Stripe can't see. PostHog-native — queries in `skills/payments-checkout/posthog-queries.md`. Shop Pay tracked as a first-class method |
| `payments-storefront-audit` | Weekly + pre-market-launch | Shopify CLI audit of checkout config, extensibility functions, installed apps, webhook subscriptions, and theme diffs. Emits change signals for Synthesis |
| `payments-reconciliation` | Daily/weekly/monthly | Three-hop join: Stripe Payout → BalanceTransaction → Charge → Shopify Order → internal ledger |
| `payments-synthesis` | Daily 08:00 + weekly Monday | Reads all five specialist outputs, produces briefings (max 5 findings/day) |

## Prerequisites

Before importing any agent, confirm the following.

### 1. Runtime
- **Claude workspace** with skills support
- **A scheduler** — cron, GitHub Actions, Temporal, or the runtime's own scheduler. `payments-synthesis` must run *after* the others each cycle
- **Slack** (or another structured sink) for the six channels below

### 2. Stack assumptions
- **Shopify storefront** — classic theme or Hydrogen. Some agents (storefront-audit, checkout) rely on the Shopify Admin API and CLI
- **Stripe as the processor** — via Shopify Payments (Stripe under the hood) or as a direct third-party gateway. This ecosystem is designed around one processor; multi-PSP is out of scope
- **Meaningful volume** — anomaly thresholds assume >500 sessions per segment. Below ~10K transactions/day, findings will be sparse

### 3. Data sources & credentials

| Need | For which agents | Notes |
| --- | --- | --- |
| **Stripe API key** (restricted, read-only) | health, radar-tuner, reconciliation | Scopes: `charges`, `payment_intents`, `payouts`, `balance_transactions`, `disputes`, `radar.rules`, `radar.reviews`, `radar.value_lists` |
| **Stripe Sigma or Data Pipeline** | health (dollar impact), reconciliation (fee audit) | Warehouse view acceptable; needed for GMV baselines |
| **Shopify Admin API token** | radar-tuner (customer join), reconciliation (order match), storefront-audit | Scopes: `read_orders`, `read_customers`, `read_payment_terms`, `read_shopify_payments_accounts`, `read_apps`, `read_themes`, `read_checkouts` |
| **Shopify CLI ≥3.x** authenticated | storefront-audit | `shopify auth login`; store connected via `shopify app config link` |
| **PostHog project + personal API key** | checkout (HogQL queries), synthesis (posts annotations) | Canonical event contract in `skills/payments-checkout/SKILL.md` |
| **Bank statement access** (CSV export or API) | reconciliation | Required for the bank-hop verification |
| **Stripe rate card** (static config is fine) | reconciliation | For fee variance detection |
| **Slack bot token** or webhook URLs | synthesis, all agents that notify | `chat:write` on the six channels |

Store all secrets in your runtime's secret manager. No agent reads secrets from disk.

### 4. First-run checklist
1. Populate a static `markets.json` for every active market: `{ code, timezone, currency, preferred_methods, sca_regime, launch_date }`
2. Wire Stripe Sigma (or warehouse mirror) so `payments-health` can compute dollar impact
3. Confirm `charge.metadata.shopify_order_id` is populated on every Stripe charge — this is the primary join key `payments-reconciliation` depends on. If it's missing, fix that first; every downstream reconciliation finding will be `medium` confidence otherwise. If Claude commerce agents run upstream, also wire `charge.metadata.agent_session_id` and `intent_id` — see [Join keys and metadata contract](#join-keys-and-metadata-contract)
4. Ensure PostHog is emitting the canonical event contract (see `skills/payments-checkout/SKILL.md`); map non-canonical event names via a CTE if needed
5. Create the six Slack channels below
6. Run each specialist agent once manually to confirm it produces valid Findings before enabling the schedule
7. Run `payments-synthesis` last — verify it can read all five inputs

### 5. Not the right fit if…
- You're on multi-PSP routing (this build has no routing logic — see git history for the multi-PSP version)
- You're not on Shopify (storefront-audit and much of checkout will not apply)
- You have <1K transactions/day (thresholds won't have signal to fire cleanly)
- You want the agents to *take* actions (they don't — they surface findings for humans to act on; Radar rule application, checkout config toggles, etc. are human-approved)

## Key design principles

- **Agents produce structured Findings, they don't make decisions for other agents.** Health flags an anomaly; Radar Tuner decides whether to loosen a rule; a human applies it.
- **Every finding needs a dollar figure.** Qualitative observations without numbers don't get surfaced. Dollar impact comes from Stripe Sigma, not private estimates.
- **The Synthesis Agent is editorial, not a forwarder.** An auth-rate drop + Radar tightening + 3DS spike is one story, not three alerts.
- **Always report paired metrics.** Fraud rate without false-positive rate is meaningless. Success rate without segmentation is not actionable. Every Finding carries both a `primary` and a `counter` metric.

## The Finding schema

Every specialist agent emits an array of Findings conforming to this shape. The Synthesis Agent discards anything non-conforming.

```json
{
  "agent": "<agent name, e.g. payments-health>",
  "finding_id": "<stable hash of {agent, signals[], segment, window.start} — same underlying issue produces the same id across runs>",
  "emitted_at": "<ISO 8601>",
  "window": {
    "start": "<ISO 8601>",
    "end":   "<ISO 8601>",
    "market_timezone": "<IANA tz>"
  },
  "signals": ["<canonical signal names from the taxonomy below>"],
  "segment": {
    "market":        "<ISO 3166-1 alpha-2 or 'global'>",
    "method":        "<card | shop_pay | apple_pay | google_pay | klarna | ... | 'all'>",
    "device":        "mobile | desktop | all",
    "card_brand":    "<visa | mc | amex | ... | 'all'>",
    "sales_channel": "<online_store | pos | shop_app | 'all'>"
  },
  "metric_paired": {
    "primary": { "name": "<metric>", "value": <n>, "unit": "<unit>", "baseline": <n>, "delta_pct": <signed n> },
    "counter": { "name": "<counter>", "value": <n>, "unit": "<unit>", "baseline": <n>, "delta_pct": <signed n> }
  },
  "dollar_impact": {
    "amount_usd": <number>,
    "basis":      "per_hour | per_day | per_month | one_time",
    "method":     "stripe_sigma | raw_variance | manual_estimate",
    "confidence": "high | medium | low"
  },
  "severity":   "P0 | P1 | P2 | P3 | opportunity | info",
  "confidence": <0.0 - 1.0>,
  "hypothesis":         "<one sentence — what likely caused this>",
  "recommended_action": "<one specific next step>",
  "action_owner":       "payments_eng | fraud_team | head_of_payments | merchant_admin | product",
  "evidence_refs":      ["<dashboard URL, query id, event id, or log line>"],
  "carry_over_of":      "<finding_id of a previous finding if this is the same issue, else null>",
  "commerce_agent_context": {
    "session_id": "<upstream shopping/merchant agent session id, or null>",
    "intent_id":  "<stable id issued at the shopping agent's checkout handoff and expected on Stripe PaymentIntent.metadata.intent_id, or null>",
    "surface":    "shopping | merchant | none",
    "vertical":   "retail | travel | telecom | entertainment | custom | none"
  }
}
```

`commerce_agent_context` is optional. Set every field to `null` (or omit the object) if no upstream Claude commerce agent is running — every correlation and suppression rule works unchanged. See [Join keys and metadata contract](#join-keys-and-metadata-contract) for wiring.

### Signal taxonomy

Use only these canonical names in the `signals[]` array — the correlation table below is keyed on them.

**Metric signals** — `success_rate_drop`, `success_rate_recovery`, `auth_rate_drop`, `auth_rate_recovery`, `latency_p99_spike`, `latency_p95_spike`, `uptime_breach`, `fraud_rate_up`, `fraud_rate_down`, `false_positive_rate_up`, `false_positive_rate_down`, `chargeback_rate_up`, `dispute_win_rate_down`, `3ds_challenge_rate_up`, `3ds_abandonment_up`, `checkout_abandonment_up`, `method_coverage_gap`, `settlement_lag`, `recon_variance_up`

**Change signals** — `stripe_status_degraded`, `shopify_status_degraded`, `recent_deploy`, `checkout_config_changed`, `checkout_extension_deployed`, `payment_app_installed`, `payment_app_removed`, `webhook_endpoint_changed`, `checkout_theme_changed`

Do not emit unnamed signals. Add new ones via PR to this README.

### Suppression rules

Applied by Synthesis based on `finding_id` history:

| Age | Ack'd? | Action |
|---|---|---|
| Day 1 | — | Surface at declared severity |
| Day 2 | No | Surface with "still open" tag |
| Day 3 | No | Downgrade one severity level |
| Day 5 | No | Auto-mute, move to weekly summary only |
| Any | Yes | Respect ack — suppress until unsnoozed or a >2σ shift breaks it |

## Join keys and metadata contract

Every Finding anchors to the underlying commerce record through one of three keys, in order of preference. All three live in Stripe charge / PaymentIntent metadata — no separate store, no separate API call.

| Key | Set by | Which agents use it | If missing |
|---|---|---|---|
| `charge.metadata.shopify_order_id` | Shopify Payments (native) | Reconciliation, Health (dollar attribution) | Order lookup falls back to amount+timestamp, downgrading Reconciliation findings to `medium` confidence |
| `charge.metadata.agent_session_id` | Host, when a Claude commerce agent handed off checkout | Health, Radar Tuner, Checkout, Synthesis (agent-vs-non-agent slicing) | `commerce_agent_context.session_id` is `null`; every Finding is still emitted, just cannot be sliced "agent-driven vs. rest" |
| `charge.metadata.intent_id` | Host, from the shopping agent's `checkout` handoff payload | Reconciliation (attribution audit), Checkout (funnel replay) | `commerce_agent_context.intent_id` is `null` |

**If you run Claude commerce agents upstream** ([anthropics/commerce-agents](https://github.com/anthropics/commerce-agents)): the shopping agent's `checkout` tool hands off a cart to the host. The host is expected to forward `agent_session_id` (its session id) and `intent_id` (a stable id it mints for the handoff) into the resulting Stripe `PaymentIntent.metadata`. Once wired, PaymentsAgents' Findings carry `commerce_agent_context` populated and Synthesis can distinguish "auth-rate drop on agent-driven sessions" from "auth-rate drop overall" — a very different story for a Head of Payments.

**If you don't:** leave `commerce_agent_context` out or set every field to `null`. Nothing else changes.

## Cross-agent correlation patterns

The Synthesis Agent checks for these before treating any output independently.

| Signal pattern | What it means |
| --- | --- |
| `auth_rate_drop` + `fraud_rate_down` + `false_positive_rate_up` | Radar tightened too aggressively — one story |
| `success_rate_drop` + `checkout_extension_deployed` (recent) | Payment customization function is the likely cause — roll back |
| `success_rate_drop` + `checkout_config_changed` (recent) | Merchant-side config drift, not an engineering incident |
| `method_coverage_gap` (checkout) + `checkout_config_changed` (audit) same market/method | Confirmed gap — promote to high confidence |
| `checkout_abandonment_up` + `3ds_challenge_rate_up` | 3DS is overtriggering; loosen Radar challenge rules |
| `false_positive_rate_up` + `payment_app_installed` (fraud category) | New fraud app recalibrating, not model drift — give it a week |
| `recon_variance_up` + `webhook_endpoint_changed` | Payout webhook regression, not a Stripe payout issue |
| `settlement_lag` + `recon_variance_up` | Stripe payout issue — one finding, not two |
| Shop Pay submit rate falls below card submit rate | Red flag — usually a Shop Pay UI regression from a Checkout Extensibility change |

## Example: a synthesized briefing

This is what the system produces at 08:00. Notice how five raw signals collapse into **two** findings, each with paired metrics and a dollar impact.

---

**Payments Daily Briefing — Tue, 14 Jul**
*Synthesis Agent · 2 findings · signals reviewed: 23 · surfaced: 2*

**1. Radar tightened overnight — costing ~$28K/day in declined good orders**
*Sources: Radar Tuner, Health, Checkout*

A Radar rule change deployed at 02:00 tightened NL card transactions harder than intended. Three signals line up:

- Auth rate down 1.4pts in NL (94.1% → 92.7%)
- Fraud rate down 0.09pts (good) **but** false positive rate up 2.1pts
- 3DS challenge rate up 6pts, 3DS abandonment up 4pts on challenged sessions

Classic tightened-too-aggressively pattern. Blocked-good-order volume ≈ **$28K/day** against ~$3K/day of incremental fraud prevented. Net negative.

**Recommendation:** Fraud team to relax rule_XYZ from `:risk_level: = 'elevated'` to `:risk_level: = 'highest'` for NL. Estimated 340 blocks/day flip to allow. Radar Tuner has the exact rule string ready.

**2. Payout missing — $41K unreconciled from 2026-07-13**
*Sources: Reconciliation*

Stripe reports payout `po_1PabcXYZ` (=$41,200) as paid on 2026-07-13, but nothing landed in the bank account by end-of-day. Not a fee variance — the entire payout is missing. Most likely SWIFT/ACH failure at the receiving bank.

**Recommendation:** Head of payments to contact bank operations for ACH receipt confirmation and open a Stripe support ticket referencing `po_1PabcXYZ`. Hold finance close for this period.

*Filtered as noise: minor latency blip on Shopify checkout (self-resolved), routine storefront audit (clean), weekly Shop Pay share up 2pts (positive drift, no action).*

---

## Repo structure

```
skills/
  payments-health/                          # KPI + incident, single-processor context
    SKILL.md
  payments-radar-tuner/                     # Stripe Radar tuning + Shopify FP proxy
    SKILL.md
  payments-checkout/                        # Pre-submit funnel, PostHog-backed
    SKILL.md
    posthog-queries.md                      # Ready-to-run HogQL
  payments-storefront-audit/                # Shopify CLI merchant-config drift
    SKILL.md
  payments-reconciliation/                  # Stripe ↔ Shopify ↔ bank ↔ ledger
    SKILL.md
  payments-synthesis/                       # Daily/weekly briefing
    SKILL.md
```

Packages (`.skill` bundles) are not shipped — rebuild locally with the Anthropic skill packager once you've customized the SKILL.md files for your workspace.

## Slack channels

Example routing. Findings from the Synthesis Agent land here based on severity and type.

| Channel | Purpose |
| --- | --- |
| `#payments-incidents` | P0 critical outages with @here |
| `#payments-alerts` | P1-P2 alerts, radar findings, reconciliation issues |
| `#payments-briefing` | Daily synthesized briefing at 08:00 |
| `#payments-leadership` | Weekly Monday executive summary |
| `#payments-optimisation` | Checkout conversion and Shop Pay opportunities |

## License

MIT. See [LICENSE](LICENSE).
