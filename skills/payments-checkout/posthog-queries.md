# PostHog HogQL queries for `payments-checkout`

Reference queries the agent should call. Assume the canonical event contract from `SKILL.md`. If your events are named differently, map them once in a preamble CTE rather than editing each query.

Run these against the HogQL query endpoint:
```
POST /api/projects/:project_id/query
{ "query": { "kind": "HogQLQuery", "query": "<sql>" } }
```

The returned JSON goes into the Finding's `evidence_refs` as the query id.

Timezone note: all timestamps are UTC. Convert to the market's local timezone before selecting a baseline slot — Saturday night ≠ Monday morning, per market.

---

## 1. End-to-end funnel — last 24h vs 28-day baseline

Produces the drop-off percentage at each stage, per market and device, compared to the same weekday-and-hour slot averaged over the prior 28 days.

```sql
WITH
  today AS (
    SELECT
      properties.market AS market,
      properties.device AS device,
      countIf(event = 'checkout_viewed')      AS s_viewed,
      countIf(event = 'method_displayed')     AS s_method_shown,
      countIf(event = 'method_selected')      AS s_method_picked,
      countIf(event = 'details_entered')      AS s_details,
      countIf(event = 'submit_clicked')       AS s_submit,
      countIf(event = 'payment_succeeded')    AS s_success
    FROM events
    WHERE timestamp >= now() - INTERVAL 24 HOUR
    GROUP BY market, device
  ),
  baseline AS (
    SELECT
      properties.market AS market,
      properties.device AS device,
      avg(daily_viewed)   AS b_viewed,
      avg(daily_success)  AS b_success,
      avg(daily_submit)   AS b_submit
    FROM (
      SELECT
        properties.market AS market,
        properties.device AS device,
        toStartOfDay(timestamp) AS d,
        countIf(event = 'checkout_viewed')   AS daily_viewed,
        countIf(event = 'submit_clicked')    AS daily_submit,
        countIf(event = 'payment_succeeded') AS daily_success
      FROM events
      WHERE timestamp >= now() - INTERVAL 29 DAY
        AND timestamp <  now() - INTERVAL 1  DAY
        AND toDayOfWeek(timestamp) = toDayOfWeek(now())
        AND toHour(timestamp)      = toHour(now())
      GROUP BY market, device, d
    )
    GROUP BY market, device
  )
SELECT
  t.market, t.device,
  t.s_viewed, t.s_method_shown, t.s_method_picked, t.s_details, t.s_submit, t.s_success,
  round(1 - t.s_method_shown / nullif(t.s_viewed,       0), 4) AS drop_view_to_display,
  round(1 - t.s_method_picked / nullif(t.s_method_shown, 0), 4) AS drop_display_to_pick,
  round(1 - t.s_details       / nullif(t.s_method_picked,0), 4) AS drop_pick_to_details,
  round(1 - t.s_submit        / nullif(t.s_details,      0), 4) AS drop_details_to_submit,
  round(t.s_success / nullif(t.s_viewed, 0), 4)                 AS effective_conversion,
  round(b.b_success / nullif(b.b_viewed, 0), 4)                 AS baseline_conversion,
  round(t.s_success / nullif(t.s_viewed, 0) - b.b_success / nullif(b.b_viewed, 0), 4) AS delta
FROM today t
LEFT JOIN baseline b USING (market, device)
ORDER BY delta ASC
```

**Emit a Finding when:** `abs(delta) > 0.01` AND `s_viewed > 500` in a segment.

---

## 2. Method coverage gap — preferred method missing

Compares which methods were `method_displayed` in each market against the market's preferred methods (from `oracle.markets`). Anything preferred that never appears is a coverage gap.

```sql
SELECT
  properties.market AS market,
  arraySort(groupUniqArrayArray(properties.methods)) AS methods_ever_shown,
  count() AS sessions,
  sum(toInt32(properties.preferred_method_available = false)) AS sessions_missing_preferred,
  round(sum(toInt32(properties.preferred_method_available = false)) / count(), 4) AS pct_missing_preferred
FROM events
WHERE event = 'method_displayed'
  AND timestamp >= now() - INTERVAL 7 DAY
GROUP BY market
HAVING pct_missing_preferred > 0.02
ORDER BY sessions_missing_preferred DESC
```

Cross-check the resulting `market → methods_ever_shown` list against your market config (`preferred_methods` per market) to name which method is missing. Price with the `method_coverage_gap` formula: `market_volume × preference_share × aov × (1 − fallback_rate)`.

Then ask `payments-storefront-audit` for the actual Shopify Payments config for the market. If the audit confirms the method is disabled, upgrade `confidence` to `high` and cite the audit's finding id in `evidence_refs`.

---

## 3. Latency vs abandonment — the +1s = -1.5% rule, quantified for your traffic

Buckets sessions by observed `submit_clicked` latency and reports the resulting `submit → success` rate. Slope tells you your true latency sensitivity.

```sql
SELECT
  properties.device AS device,
  floor(properties.latency_ms / 500) * 500 AS latency_bucket_ms,
  count() AS sessions,
  countIf(event = 'payment_succeeded') / nullif(count(), 0) AS success_rate
FROM events
WHERE event IN ('submit_clicked', 'payment_succeeded')
  AND timestamp >= now() - INTERVAL 14 DAY
  AND properties.latency_ms > 0
GROUP BY device, latency_bucket_ms
HAVING sessions > 200
ORDER BY device, latency_bucket_ms
```

Fit a linear regression client-side. The slope × your GMV = your actual latency cost per +100ms. Do not use the default 1.5% industry assumption if this query returns >200 sessions per bucket — use the measured slope.

---

## 4. 3DS abandonment rate — is the challenge killing conversion?

```sql
SELECT
  properties.market AS market,
  properties.method AS method,
  countIf(event = 'threeds_challenged')                                  AS challenged,
  countIf(event = 'threeds_completed' AND properties.outcome = 'passed') AS passed,
  countIf(event = 'threeds_completed' AND properties.outcome = 'abandoned') AS abandoned,
  round(countIf(event = 'threeds_completed' AND properties.outcome = 'abandoned')
        / nullif(countIf(event = 'threeds_challenged'), 0), 4) AS abandonment_rate
FROM events
WHERE timestamp >= now() - INTERVAL 7 DAY
GROUP BY market, method
HAVING challenged > 100
ORDER BY abandonment_rate DESC
```

**Emit a Finding with signals `["3ds_abandonment_up", "checkout_abandonment_up"]`** when `abandonment_rate > 0.15` in a segment with >100 challenges. This is the input for the Synthesis correlation pattern "checkout abandonment up + 3DS challenge rate up".

---

## 5. False-positive rebuy proxy — cheap fallback for fraud analyst

On this Shopify + Stripe stack, `payments-radar-tuner` should use the Shopify customer-history join instead — it has the user database with `customer.orders_count` and `customer.created_at`, which is cheaper and more accurate. This PostHog proxy is only useful if that Shopify join is unavailable — treat as a fallback (`confidence: medium`).

A blocked session is a probable false positive if the same `distinct_id` completed a payment within 7 days on any method.

```sql
WITH blocked AS (
  SELECT distinct_id, min(timestamp) AS blocked_at
  FROM events
  WHERE event = 'payment_failed'
    AND properties.decline_type = 'fraud'
    AND timestamp >= now() - INTERVAL 14 DAY
  GROUP BY distinct_id
),
rebuys AS (
  SELECT b.distinct_id
  FROM blocked b
  JOIN events e
    ON e.distinct_id = b.distinct_id
   AND e.event = 'payment_succeeded'
   AND e.timestamp BETWEEN b.blocked_at AND b.blocked_at + INTERVAL 7 DAY
  GROUP BY b.distinct_id
)
SELECT
  count() AS blocked_sessions,
  countIf(distinct_id IN (SELECT distinct_id FROM rebuys)) AS probable_false_positives,
  round(countIf(distinct_id IN (SELECT distinct_id FROM rebuys)) / count(), 4) AS false_positive_proxy_rate
FROM blocked
```

Hand the resulting rate to `payments-radar-tuner` as the counter metric when the Shopify join is unavailable — enforces the "never report fraud rate without false positive rate" rule.

---

## 6. Method-mix shift — for cost optimizer and synthesis correlation

Weekly method-share, week-over-week. Feeds the "cost per txn up + method mix shifting to cards" pattern.

```sql
SELECT
  toStartOfWeek(timestamp) AS week,
  properties.method AS method,
  count() AS txns,
  round(count() / sum(count()) OVER (PARTITION BY week), 4) AS share
FROM events
WHERE event = 'payment_succeeded'
  AND timestamp >= now() - INTERVAL 60 DAY
GROUP BY week, method
ORDER BY week DESC, share DESC
```

Emit `method_mix_shift` when a method's share changes by >3pts week-over-week.

---

## Wiring PostHog Alerts

The scheduled runs of this agent don't need to poll — configure PostHog Alerts on:

- **Effective conversion rate** (query #1, `effective_conversion` column) — alert on >1% drop
- **3DS abandonment rate** (query #4) — alert on >15% in any market with >100 challenges
- **Method-mix shift** (query #6) — alert on >3pt weekly change

The alert webhook posts to the agent's input queue as an `on_demand` run with `run_reason: posthog_alert` and the triggering query id. That id must appear in the emitted Finding's `evidence_refs`.

## Wiring PostHog Annotations

At the end of every `payments-synthesis` daily briefing, POST an annotation to PostHog's conversion chart:

```
POST /api/projects/:project_id/annotations
{
  "content": "<headline_sentence from briefing>",
  "date_marker": "<generated_at>",
  "scope": "organization"
}
```

Result: the analytics chart shows a timeline of "what did we say" next to "what happened." Enables the feedback loop the ecosystem is currently missing.
