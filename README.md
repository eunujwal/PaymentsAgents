# Payments Agent Ecosystem

Eleven specialized AI agents that monitor, optimize, and protect a multi-PSP payments platform, one synthesis agent that turns their raw signals into a briefing a human can actually act on, and a shared context layer that keeps every agent's output correlatable.

The design goal is the opposite of most monitoring setups: fewer, better findings. Each agent produces structured data. The Synthesis Agent decides what's worth a human's attention, correlates signals across agents, and attaches a dollar figure to everything it surfaces.

## Prerequisites

Before importing any agent, confirm the following. Missing items don't necessarily block the ecosystem — each agent lists its own data dependencies and degrades gracefully — but they set the ceiling on how much of the value you'll actually get.

### 1. Runtime

- **Claude workspace** with skills support enabled (skills are imported from `packages/*.skill`)
- **A scheduler** that can trigger agents on their declared cadence — cron, GitHub Actions, Temporal, or the runtime's own scheduler. `payments-synthesis` must run *after* the other agents each cycle
- **A place for outputs to land.** The reference wiring is Slack (six channels below); any structured sink (Linear, Jira, Notion, email) works if you rewrite the `slack_notification` block

### 2. Payments platform assumptions

The ecosystem is designed for:

- **Multi-PSP setups** (Stripe + Adyen + Braintree, or similar). Single-PSP orgs still get value from checkout, fraud, and reconciliation agents, but cost-optimizer and partner-health assume routing choice exists
- **Meaningful transaction volume.** Anomaly detection uses >2σ from a 28-day baseline with a >500-session floor per segment. Below ~10K transactions/day per segment, findings will be sparse and low-confidence
- **A defined market list.** `oracle.markets` needs each active market's timezone, currency, preferred payment methods, and SCA regime. This can be a static JSON file to start

### 3. Data sources by agent

| Agent | Requires | Degrades gracefully without |
|---|---|---|
| `payments-context` | Static config: markets, PSP registry. GMV feed (data warehouse or PSP report). Change log source (Git/CI + `payments-storefront-audit`) | GMV oracle missing → dollar impacts marked `manual_estimate` / `confidence: low` |
| `payments-kpi-monitor` | Read access to PSP transaction data (API or warehouse), APM for latency | Without APM: client-side latency signals unavailable, flag in output |
| `payments-fraud-analyst` | Fraud model score outputs, chargeback data, user DB (for 7-day rebuy join) | Without user DB: falls back to PostHog rebuy proxy at `confidence: medium` |
| `payments-cost-optimizer` | PSP fee schedules, transaction data with routing decisions | — |
| `payments-checkout-optimizer` | Product analytics with the canonical event contract (PostHog reference — see `skills/payments-checkout-optimizer/posthog-queries.md`) | No analytics → PSP-only submit-onward view, agent flags the blindness |
| `payments-incident-agent` | PSP status page feeds, internal alerting integration | — |
| `payments-reconciliation` | Bank statement access, PSP settlement reports, internal ledger | — |
| `payments-partner-health` | PSP contracts (SLAs, renewal dates), historical uptime data | — |
| `payments-regulatory-monitor` | Regulatory feed (subscription or scraped) covering active markets | — |
| `payments-security-audit` | Infra read access, webhook endpoint list | — |
| `payments-storefront-audit` | **Shopify merchant** with CLI ≥3.x authenticated (`shopify auth login`); admin API scopes: `read_payment_terms`, `read_shopify_payments_accounts`, `read_apps`, `read_themes`, `read_checkouts` | Not a Shopify merchant → skip this agent; Synthesis's `checkout_config_changed` correlation is dead but nothing else breaks |
| `payments-synthesis` | Read access to the structured outputs of the other agents | Runs against however many agents are wired — output quality scales with input coverage |

### 4. Credentials & access

You'll need, minimum:

- **PSP API keys** (restricted, read-only) for each active PSP
- **Product analytics access** — PostHog project + personal API key for HogQL queries, or equivalent Amplitude/Mixpanel key
- **Shopify CLI auth** and the admin scopes above (only if using `payments-storefront-audit`)
- **Slack** — bot token with `chat:write` on the six briefing channels, or incoming webhook URLs
- **APM** — Datadog/Grafana read token if you want latency signals from beyond the PSP boundary

Store these in your runtime's secret manager. No agent reads secrets from disk; they're passed at invocation time.

### 5. First-run checklist

Before your first scheduled cycle:

1. Populate `oracle.markets` for every active market (timezone, currency, preferred methods)
2. Populate `oracle.psps` with your live PSP registry, SLA days, renewal dates
3. Point `oracle.gmv` at your revenue source (warehouse view, PSP report aggregation, or manual CSV to start)
4. Wire `oracle.changes` to at least one change source (Git deploy log, feature-flag audit log, or `payments-storefront-audit` output). Without this, the "success rate drop + recent change" correlation cannot fire
5. Create the six Slack channels listed below
6. Run `payments-context` self-check to confirm every downstream agent's Finding output validates against the schema

### 6. Not the right fit if…

- You're on a single PSP with no plan to route (much of the ecosystem is redundant)
- You have <1K transactions/day (thresholds won't have enough signal to fire cleanly)
- Your compliance context prohibits sending briefings to Slack/external chat, and you don't have a replacement sink
- You want the agents to *take* actions (they don't — they surface findings for humans to act on)

## Quick start

> **Note on runtime:** These agents are packaged as Claude skills. Each lives as a `SKILL.md` (readable source in `skills/`) and a bundled `.skill` file (`packages/`) you can import directly. Confirm the two lines below match your setup before publishing.

1. **Import an agent.** Grab any `.skill` file from `packages/` and import it into your Claude workspace as a skill.
2. **Wire the schedule.** The cadence column in the Agents table (hourly, daily, weekly) is the intended trigger. Point your scheduler at each agent accordingly. The `payments-synthesis` agent should run *after* the others so it has fresh outputs to read.
3. **Connect the outputs.** Agents write findings that the Synthesis Agent reads. Route the final briefings to the Slack channels in the table below.

Want to understand the system before running it? Read `payments_agents_handbook.pdf` for the full reference, or browse `skills/` for any individual agent's logic in plain markdown.

## Architecture

The agents are organized in four layers, and signal flows upward. Detection agents watch continuously, optimization and compliance agents run on cadence, and everything funnels into a single synthesis step.

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
│  Real-Time     │   │  Optimization    │   │  Compliance &    │
│  Detection     │   │  & Analysis      │   │  Relationships   │
│                │   │                  │   │                  │
│  Incident      │   │  Cost Optimizer  │   │  Reconciliation  │
│  KPI Monitor   │   │  Checkout Opt.   │   │  Partner Health  │
│  Fraud Analyst │   │  Security Audit  │   │  Regulatory Mon. │
│                │   │  Storefront Audit│   │                  │
└────────────────┘   └──────────────────┘   └─────────┬────────┘
        ▲                      ▲                      ▲
        └──────────────────────┴──────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │  payments-context   │   foundation layer
                    │  schema + oracles   │   shared vocabulary
                    └─────────────────────┘
```

| Layer | Agents | Purpose |
| --- | --- | --- |
| **Foundation** | `payments-context` | Shared Finding schema, signal taxonomy, dollar-impact contract, market/PSP/GMV/change oracles |
| **Real-Time Detection** | Incident Agent, KPI Monitor, Fraud Analyst | Continuous monitoring, outage detection, fraud calibration |
| **Optimization & Analysis** | Cost Optimizer, Checkout Optimizer, Security Audit, Storefront Audit | Routing efficiency, pre-submit funnel, security posture, merchant-config drift |
| **Compliance & Relationships** | Reconciliation, Partner Health, Regulatory Monitor | Settlement verification, vendor SLAs, regulatory compliance |
| **Synthesis** | Synthesis Agent | Correlates signals across agents, produces daily/weekly briefings |

## Agents

| Agent | Schedule | What it does |
| --- | --- | --- |
| `payments-context` | Referenced by every run | Foundation — Finding schema, canonical signal names, shared oracles; not an agent you schedule |
| `payments-incident-agent` | Always on | Detects outages, triages before alerting, suggests routing shifts |
| `payments-kpi-monitor` | Hourly | Tracks success rate, auth rate, latency, uptime across all PSPs |
| `payments-fraud-analyst` | Hourly + triggered | Balances fraud rate vs false positives, detects model drift |
| `payments-cost-optimizer` | Daily | Finds routing inefficiencies, token gaps, method mix savings |
| `payments-checkout-optimizer` | Daily | Owns the pre-submit funnel PSP data can't see (PostHog-native — see `skills/payments-checkout-optimizer/posthog-queries.md`) |
| `payments-storefront-audit` | Weekly + pre-market-launch | Audits Shopify checkout config, apps, extensibility functions, webhooks, and theme diffs via Shopify CLI; emits change signals for Synthesis |
| `payments-security-audit` | Weekly + on integration | PCI DSS scope, OWASP checks, webhook validation |
| `payments-reconciliation` | Daily/weekly/monthly | Verifies every settled transaction matches ledger and bank |
| `payments-partner-health` | Weekly + before renewals | SLA tracking, contract milestones, negotiation readiness |
| `payments-regulatory-monitor` | Weekly scans | SCA/PSD2, AML, card scheme mandates, market licensing |
| `payments-synthesis` | Daily 08:00 + weekly Monday | Reads all agent outputs, produces briefings (max 5 findings/day) |

## Key design principles

- **Agents produce structured data, they don't make decisions for other agents.** The Incident Agent recommends a routing shift; the Cost Optimizer validates capacity.
- **Every finding needs a dollar figure.** Qualitative observations without numbers don't get surfaced.
- **The Synthesis Agent is editorial, not a forwarder.** An auth rate drop + fraud model change + 3DS spike is one story, not three alerts.
- **Always report paired metrics.** Fraud rate without false positive rate is meaningless. Success rate without segmentation is not actionable.

## Cross-agent correlation patterns

The Synthesis Agent checks for these before treating any output independently. This is where the ecosystem earns its keep, a single degraded number from one agent is noise; the same number next to a second agent's signal is a story.

| Signal pattern | What it means |
| --- | --- |
| Auth rate down + fraud rate down + FP up | Fraud model tightened too aggressively |
| Auth rate drop on PSP A only + PSP A status degraded | PSP incident, route around it |
| Success rate drop + recent deploy | Engineering incident, not payments-specific |
| Success rate drop + checkout_config_changed | Merchant-side config drift, not engineering |
| Success rate drop + checkout_extension_deployed | Payment customization function is the likely cause |
| Checkout abandonment up + 3DS challenge rate up | 3DS overtriggering is killing conversion |
| Method coverage gap (analytics) + checkout_config_changed (CLI) | Confirmed method disable — promote finding to high confidence |
| Fraud rate up + payment_app_installed (fraud category) | New fraud tool recalibrating, not model drift |
| Recon variance up + webhook_endpoint_changed | Payout webhook regression, not partner issue |
| Cost per txn up + method mix shifting to cards | Mix shift problem, not fee negotiation |
| Recon variance up + PSP settlement delays | PSP payout issue, escalate to partner health |

## Example: a synthesized briefing

This is what the system produces at 08:00. Notice that six raw agent signals collapse into **two** findings, each with paired metrics and a dollar impact. Everything else got filtered as noise.

---

**Payments Daily Briefing — Tue, 14 Jul**
*Synthesis Agent · 2 findings · signals reviewed: 47 · surfaced: 2*

**1. Fraud model overcorrected overnight — costing ~$28K/day in declined good orders**
*Sources: Fraud Analyst, KPI Monitor, Checkout Optimizer*

The fraud model retrained at 02:00 and tightened harder than intended. Three signals line up:

- Auth rate down 1.4pts (94.1% → 92.7%)
- Fraud rate down 0.09pts (good) **but** false positive rate up 2.1pts
- 3DS challenge rate up 6pts, checkout abandonment on challenged sessions up 4pts

This is the classic *tightened-too-aggressively* pattern. Blocked-good-order volume maps to roughly **$28K/day** in lost authorized revenue, against ~$3K/day of incremental fraud prevented. Net negative.

**Recommendation:** Roll back to the pre-02:00 threshold, or relax the challenge trigger on the two segments driving the FP spike (returning customers, sub-$150 orders). Fraud Analyst has the segment breakdown ready.

**2. PSP-B settlement lag is creating a $410K reconciliation gap**
*Sources: Reconciliation, Partner Health*

Recon variance jumped to $410K over the last 48 hours, isolated entirely to PSP-B. Cross-referenced against Partner Health: PSP-B payout timing slipped from T+1 to T+3 starting Sunday. The transactions are authorized and captured, the money just hasn't landed.

This isn't a ledger error, it's a partner SLA breach. PSP-B's contract specifies T+1 settlement, and they're up for renewal in November.

**Recommendation:** No engineering action needed. Partner Health is escalating to the PSP-B account team today and logging the breach for the renewal negotiation. Reconciliation will confirm the gap closes once payouts catch up.

*Filtered as noise: minor latency blip on PSP-C (self-resolved), expected weekly security scan (clean), routine regulatory digest (no new mandates).*

---

## Repo structure

```
skills/                                    # Extracted SKILL.md files (readable markdown)
  payments-context/                        # Foundation — schema, signal taxonomy, oracles
  payments-incident-agent/
  payments-kpi-monitor/
  payments-fraud-analyst/
  payments-cost-optimizer/
  payments-checkout-optimizer/
    SKILL.md
    posthog-queries.md                     # Ready-to-run HogQL for funnel, coverage, 3DS, mix-shift
  payments-storefront-audit/               # Shopify CLI-driven merchant-config audit
  payments-security-audit/
  payments-reconciliation/
  payments-partner-health/
  payments-regulatory-monitor/
  payments-synthesis/
packages/                                  # Original .skill zip files (importable)
payments_agents_handbook.pdf               # Comprehensive PDF reference
```

## Analytics and storefront backends

The ecosystem is analytics-backend-agnostic, but two integrations are documented end-to-end:

- **PostHog** is the reference product-analytics backend for `payments-checkout-optimizer` and the false-positive proxy for `payments-fraud-analyst`. HogQL queries and PostHog Alert wiring live in `skills/payments-checkout-optimizer/posthog-queries.md`. `payments-synthesis` posts each briefing back to PostHog as an annotation so the conversion chart carries a timeline of "what we said" next to "what happened."
- **Shopify CLI** powers `payments-storefront-audit`. It converts silent merchant-side config changes (method toggles, extensibility function deploys, app installs, webhook edits) into `checkout_config_changed` / `checkout_extension_deployed` / `payment_app_installed` / `webhook_endpoint_changed` signals — the storefront equivalent of an engineering deploy log, without which Synthesis's "success rate drop + recent change" correlation is one-eyed.

## Slack channels

Example routing. Findings from the Synthesis Agent land here based on severity and type.

| Channel | Purpose |
| --- | --- |
| `#payments-incidents` | P0 critical outages with @here |
| `#payments-alerts` | P1-P2 alerts, security findings, fraud spikes |
| `#payments-briefing` | Daily synthesized briefing at 08:00 |
| `#payments-leadership` | Weekly Monday executive summary |
| `#payments-optimisation` | Cost and checkout opportunities |
| `#payments-compliance` | Regulatory alerts |

## License

MIT. See [LICENSE](LICENSE).
