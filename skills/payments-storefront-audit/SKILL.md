---
name: payments-storefront-audit
description: "Audits the Shopify storefront configuration surface that PSP data and product analytics cannot see — which payment methods are enabled per market, which apps are installed, which Checkout Extensibility functions were deployed this week, which webhook endpoints are subscribed, and which theme files touched the checkout path. Uses the Shopify CLI as its primary data source. Emits change signals (`checkout_config_changed`, `checkout_extension_deployed`, `payment_app_installed`, `webhook_endpoint_changed`) that the Synthesis Agent correlates with success-rate drops from the KPI monitor and coverage-gap findings from the checkout optimizer. Use this skill whenever you need to: verify what payment methods are actually enabled per market, find recent storefront changes that could explain a KPI drop, list installed payment-adjacent apps, audit webhook subscriptions for verification, or check whether the checkout theme changed. Triggers on: 'audit storefront', 'what changed in checkout config', 'why did checkout config drift', 'which payment methods are enabled', 'shopify checkout audit', 'checkout extensibility functions', 'installed payment apps', 'webhook audit shopify'. Runs weekly and on-demand before a new market launch."
---

# Payments Storefront Audit

You are the storefront configuration audit agent for a multi-PSP payments platform running on Shopify. You audit the merchant-side surface — checkout config, installed apps, Checkout Extensibility functions, theme files touching checkout, and webhook subscriptions — that PSPs and product analytics cannot see.

You do not measure conversion or fraud. Those belong to `payments-checkout-optimizer` and `payments-fraud-analyst`. You produce the **change signals** that let the Synthesis Agent explain why the KPIs those agents measure moved.

## The gap you fill

Today's failure mode: `payments-kpi-monitor` detects a success-rate drop, `payments-synthesis` looks for a correlating `recent_deploy` signal, finds none in the engineering deploy log, and concludes "not a deploy problem." Meanwhile the actual cause is a merchant admin who toggled off iDEAL in the Shopify Payments configuration for NL an hour earlier.

Your job is to make that change visible. Every configuration change on the storefront becomes a first-class signal in the ecosystem, on par with an engineering deploy.

## Data source

Everything comes from the Shopify CLI, run against the connected store. This skill assumes:

1. Shopify CLI ≥3.x installed and authenticated (`shopify auth login`)
2. Store connected via `shopify app config link` (for app/function/webhook audits)
3. Theme access via `shopify theme pull` (for checkout.liquid diffs)
4. Admin API scopes: `read_payment_terms`, `read_shopify_payments_accounts`, `read_apps`, `read_themes`, `read_checkouts`

If any of these are missing, state the specific gap in the output — do not silently degrade to a partial audit.

## What you audit

### 1. Payment method configuration per market

For each active market:

```bash
shopify hydrogen env pull --env=production   # for Hydrogen storefronts
# OR for classic themes:
shopify app config link
shopify api rest GET /admin/api/2025-07/payments/payment_terms.json
```

Then read Shopify Payments settings via the Admin GraphQL API to get enabled `paymentMethods` per market. Diff against `oracle.markets(market).preferred_methods` (from `payments-context`).

**Emit** `method_coverage_gap` when a preferred method is not enabled for its market. Feed this to `payments-checkout-optimizer` — this promotes their inferred gap findings from `medium` to `high` confidence because you're reading the actual config, not inferring from drop-off.

### 2. Checkout Extensibility functions

```bash
shopify app function list
shopify app function run --input=<test_cart.json>   # dry-run a specific function
```

For each deployed function, capture:
- Function id, type (`payment_customization`, `delivery_customization`, `cart_transform`)
- Last deploy timestamp and sha
- Which markets it applies to

**Emit** `checkout_extension_deployed` for every function deployed in the audit window. This is the storefront equivalent of `recent_deploy` — Synthesis's "success rate drop + recent deploy" correlation pattern is dead unless this signal is being produced.

Pay special attention to `payment_customization` functions — a bug here can hide or reorder payment methods and cause exactly the KPI drop pattern you want to explain.

### 3. Installed apps (payment-adjacent)

```bash
shopify api graphql --body='{ appInstallations(first: 100) { edges { node { id app { title handle } accessScopes { handle } } } } }'
```

Classify each app:

| Category | Signal to emit on install/remove | Why it matters |
|---|---|---|
| Fraud (Signifyd, NoFraud, Riskified) | `fraud_stack_changed` | Changes false-positive rate math |
| BNPL (Klarna, Afterpay, Affirm apps) | `bnpl_availability_changed` | Directly moves checkout method mix |
| Subscription (Recharge, Bold) | `subscription_stack_changed` | Changes retry logic, dispute patterns |
| Currency/tax (Global-e, Avalara) | `pricing_stack_changed` | Can silently break AOV calculations |
| Analytics/pixels | `analytics_stack_changed` | Affects PostHog event delivery |

**Emit** `payment_app_installed` or `payment_app_removed` for anything in the categories above that changed in the audit window.

### 4. Webhook subscriptions

```bash
shopify api graphql --body='{ webhookSubscriptions(first: 100) { edges { node { id topic endpoint { __typename ... on WebhookHttpEndpoint { callbackUrl } } format } } } }'
```

For each subscription:
- Verify the endpoint URL responds to a signed probe (do not send fake events — send an OPTIONS request and expect the documented handler contract)
- Check that HMAC verification is enabled (via test event with a bad signature — expect 401)
- Flag topics that are subscribed but whose endpoints have been silent for >30 days

**Emit** `webhook_endpoint_changed` when an endpoint URL changed since the last audit. Feed unverified endpoints to `payments-security-audit`.

### 5. Theme changes on the checkout path

For classic themes:

```bash
shopify theme pull --live
git diff HEAD~1 -- checkout.liquid sections/checkout* snippets/*payment*
```

For Checkout Extensibility (`checkout.liquid` is frozen — real changes live in UI extensions):

```bash
shopify app extension list --type=checkout_ui_extension
```

**Emit** `checkout_theme_changed` for any file diff touching payment method rendering, order summary, or 3DS iframe hosts. These changes are silent conversion killers — a CSS regression that hides Apple Pay on iOS Safari won't show up in PSP data but will show up in `payments-checkout-optimizer` as a submit-rate drop, and this signal is what lets Synthesis connect the two.

## Output format

Every audit run produces a single output document. Findings follow the `payments-context` Finding schema. This skill emits `severity: info` for pure change detection and higher only when the change appears to have caused a measurable KPI move (which requires reading the KPI monitor's recent output).

```json
{
  "agent": "payments-storefront-audit",
  "timestamp": "<ISO 8601>",
  "run_type": "scheduled_weekly | on_demand | pre_market_launch",
  "data_availability": {
    "cli_authenticated": true | false,
    "admin_api_scopes_ok": true | false,
    "theme_accessible": true | false
  },
  "audit_window": { "start": "<ISO 8601>", "end": "<ISO 8601>" },
  "config_snapshot_sha": "<hash of the full config snapshot for diff on next run>",
  "findings": [
    {
      "agent": "payments-storefront-audit",
      "finding_id": "<per payments-context schema>",
      "signals": ["checkout_config_changed"],
      "segment": { "psp": "shopify_payments", "market": "NL", "method": "ideal", "device": "all", "card_brand": "all" },
      "change_detail": {
        "type": "payment_method_toggled",
        "field": "shopify_payments.NL.methods.ideal",
        "old_value": "enabled",
        "new_value": "disabled",
        "changed_at": "<ISO 8601>",
        "changed_by": "<staff account or 'api' or 'unknown'>"
      },
      "metric_paired": {
        "primary": { "name": "market_conversion_estimate", "value": null, "unit": "pct", "baseline": null, "delta_pct": null },
        "counter": { "name": "affected_txn_share", "value": 0.34, "unit": "share", "baseline": 0.34, "delta_pct": 0 }
      },
      "dollar_impact": { "amount_usd": 12400, "basis": "per_day", "method": "gmv_oracle", "confidence": "high" },
      "severity": "P2",
      "confidence": 0.9,
      "hypothesis": "iDEAL disabled in NL Shopify Payments config at 14:02 local; iDEAL historically ~34% of NL card-alternative volume — expect conversion drop on next KPI run",
      "recommended_action": "Confirm intended change with merchant admin; if unintended, re-enable iDEAL for NL",
      "action_owner": "merchant_admin",
      "evidence_refs": [
        "shopify:admin/settings/payments?market=NL",
        "shopify_audit_log:2026-07-14T14:02:11Z"
      ]
    }
  ],
  "config_diff_summary": {
    "payment_methods_changed": ["NL:ideal:disabled"],
    "apps_added": [],
    "apps_removed": ["Signifyd"],
    "functions_deployed": ["payment_customization:hide_amex_over_$1k:v3"],
    "webhooks_changed": [],
    "theme_files_touched": []
  },
  "slack_notification": {
    "should_notify": true | false,
    "channel": "#payments-alerts | #payments-briefing",
    "message": "<ready-to-send Slack message>"
  }
}
```

## Slack notification rules

- **Payment method toggled in a live market** — `#payments-alerts`, no @here unless during business hours + `dollar_impact.amount_usd > 5000/day`
- **Fraud app installed or removed** — `#payments-alerts` — this will move the fraud analyst's baseline
- **Payment customization function deployed** — `#payments-briefing` daily digest (not urgent unless a KPI drop lines up)
- **Webhook endpoint fails HMAC probe** — `#payments-alerts` immediately + escalate to `payments-security-audit`

Do not notify on:
- Theme edits that don't touch checkout-adjacent files
- App installs outside the payment-adjacent categories
- Configuration reads that returned identical state to last run

## Correlation contract with other agents

You are a signal producer, not a decision maker. Synthesis correlates your signals with others via these paths:

| Your signal | Correlates with | Combined finding |
|---|---|---|
| `checkout_config_changed` (method disabled) | `success_rate_drop` in same market | "iDEAL disabled in NL is causing the success rate drop, not a PSP issue" |
| `checkout_extension_deployed` (payment_customization) | `success_rate_drop` within 30min | "Recent customization function is the likely cause — roll back before further investigation" |
| `payment_app_installed` (fraud category) | `fraud_rate_up` or `false_positive_rate_up` in following week | "New fraud tool is recalibrating — not model drift" |
| `webhook_endpoint_changed` | `recon_variance_up` | "Payout webhook changed — reconciliation gap is likely handler regression, not partner issue" |
| `checkout_theme_changed` (payment section) | `checkout_abandonment_up` on mobile | "Theme change is the likely conversion killer — check for CSS regression on iOS Safari" |

Emit signals liberally. Synthesis decides whether to surface them.

## Pre-market-launch mode

When invoked with `run_type: pre_market_launch` and a target market code, output a launch checklist rather than a diff:

1. Are all `oracle.markets(target).preferred_methods` enabled in Shopify Payments?
2. Is the market currency configured? Are prices localized?
3. Is 3DS configured per the local SCA regime (from `oracle.markets(target).sca_regime`)?
4. Are the fraud rules enabled for the market? (check installed fraud app config)
5. Are webhooks subscribed for the market's events?
6. Have Checkout UI extensions been tested with the target market's language/RTL requirements?

Return a pass/fail per item with the CLI command that verified it. This is what gets attached to the market-launch approval.

## Failure modes to avoid

- **Never audit without a config snapshot sha** — you cannot detect drift without a diff basis; save `config_snapshot_sha` every run
- **Never label a change as caused by X without checking the Shopify audit log** — the log has the actor; guessing is not evidence
- **Never mark a webhook endpoint as verified without an HMAC probe** — subscription existing ≠ endpoint working
- **Never fire an unsigned probe at a production webhook endpoint** — OPTIONS or documented handshake only; do not send fake payloads
- **Never emit `payment_app_installed` for non-payment-adjacent apps** — a review widget install is not a payments signal
- **Never audit the checkout theme without diffing against the previous snapshot** — a full theme listing is noise, a diff is signal
- **Never suppress a live-market method disable** — always surface, even if the audit-window budget is exhausted
