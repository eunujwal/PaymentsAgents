# Payments Agent Ecosystem

10 specialized AI agents that work together to monitor, optimize, and protect a multi-PSP payments platform.

## Architecture

The agents are organized in four layers:

| Layer | Agents | Purpose |
|-------|--------|---------|
| **Real-Time Detection** | Incident Agent, KPI Monitor, Fraud Analyst | Continuous monitoring, outage detection, fraud calibration |
| **Optimization & Analysis** | Cost Optimizer, Checkout Optimizer, Security Audit | Routing efficiency, pre-submit funnel, security posture |
| **Compliance & Relationships** | Reconciliation, Partner Health, Regulatory Monitor | Settlement verification, vendor SLAs, regulatory compliance |
| **Synthesis** | Synthesis Agent | Correlates signals across agents, produces daily/weekly briefings |

## Agents

| Agent | Schedule | What it does |
|-------|----------|-------------|
| `payments-incident-agent` | Always on | Detects outages, triages before alerting, suggests routing shifts |
| `payments-kpi-monitor` | Hourly | Tracks success rate, auth rate, latency, uptime across all PSPs |
| `payments-fraud-analyst` | Hourly + triggered | Balances fraud rate vs false positives, detects model drift |
| `payments-cost-optimizer` | Daily | Finds routing inefficiencies, token gaps, method mix savings |
| `payments-checkout-optimizer` | Daily | Owns the pre-submit funnel PSP data can't see |
| `payments-security-audit` | Weekly + on integration | PCI DSS scope, OWASP checks, webhook validation |
| `payments-reconciliation` | Daily/weekly/monthly | Verifies every settled transaction matches ledger and bank |
| `payments-partner-health` | Weekly + before renewals | SLA tracking, contract milestones, negotiation readiness |
| `payments-regulatory-monitor` | Weekly scans | SCA/PSD2, AML, card scheme mandates, market licensing |
| `payments-synthesis` | Daily 08:00 + weekly Monday | Reads all agent outputs, produces briefings (max 5 findings/day) |

## Repo structure

```
skills/                          # Extracted SKILL.md files (readable markdown)
  payments-incident-agent/
  payments-kpi-monitor/
  payments-fraud-analyst/
  payments-cost-optimizer/
  payments-checkout-optimizer/
  payments-security-audit/
  payments-reconciliation/
  payments-partner-health/
  payments-regulatory-monitor/
  payments-synthesis/
packages/                        # Original .skill zip files (importable)
payments_agents_handbook.pdf     # Comprehensive PDF reference
```

## Key design principles

- **Agents produce structured data, they don't make decisions for other agents.** The Incident Agent recommends a routing shift; the Cost Optimizer validates capacity.
- **Every finding needs a dollar figure.** Qualitative observations without numbers don't get surfaced.
- **The Synthesis Agent is editorial, not a forwarder.** An auth rate drop + fraud model change + 3DS spike = one story, not three alerts.
- **Always report paired metrics.** Fraud rate without false positive rate is meaningless. Success rate without segmentation is not actionable.

## Cross-agent correlation patterns

The Synthesis Agent looks for these before treating outputs independently:

| Signal pattern | What it means |
|---------------|---------------|
| Auth rate down + fraud rate down + FP up | Fraud model tightened too aggressively |
| Auth rate drop on PSP A only + PSP A status degraded | PSP incident — route around it |
| Success rate drop + recent deploy | Engineering incident, not payments-specific |
| Checkout abandonment up + 3DS challenge rate up | 3DS overtriggering is killing conversion |
| Cost per txn up + method mix shifting to cards | Mix shift problem, not fee negotiation |
| Recon variance up + PSP settlement delays | PSP payout issue — escalate to partner health |

## Slack channels

| Channel | Purpose |
|---------|---------|
| `#payments-incidents` | P0 critical outages with @here |
| `#payments-alerts` | P1-P2 alerts, security findings, fraud spikes |
| `#payments-briefing` | Daily synthesized briefing at 08:00 |
| `#payments-leadership` | Weekly Monday executive summary |
| `#payments-optimisation` | Cost and checkout opportunities |
| `#payments-compliance` | Regulatory alerts |
