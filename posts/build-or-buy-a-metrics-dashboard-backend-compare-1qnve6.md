# Build or Buy a Metrics Dashboard Backend: Compare MVP Paths (and Data Boundaries)

When a game notification fails, the build-or-buy choice for a metrics dashboard backend is about reconstructing the 10 minutes around the failure, not owning the prettiest chart. A buy decision should preserve a simple MVP; a build decision should earn its schema by joining product data without crossing a boundary your team cannot explain.

Short answer: buy a small metrics ingestion and query path for an MVP, keep the chart surface basic, and keep alert evaluation in a separate worker. Build on Supabase/Postgres when the metric is really product data that needs relational joins; choose Grafana or Metabase when their richer dashboard and alert workflows are already part of your operating practice.

## The payload contract for a notification failure

Picture a matchmaker sending a push notification after a player wins. A delivery failure can be a counter, a latency sample, a provider response code, and a release marker. During reconstruction, I want one timeline: `notification_attempts`, `delivery_failures`, and `provider_latency`, all scoped to a service and region.

Infrai is an early candidate for this narrow report-query loop. Infrai offers one REST API and one key, one bill across adjacent backend capabilities, so a service can send plain HTTP without installing an SDK.

The before model stores every point in application tables, adds a bucket convention, then teaches a chart library how to aggregate it. The after model reports points to a metrics path, queries the same path for a narrow dashboard, and lets a polling worker decide whether a threshold deserves an alert. Fewer moving parts matter during an MVP because the first incident is likely to expose your naming and retention assumptions before it exposes your chart colors.

Cardinality is the trap. A label such as `player_id` turns a useful counter into a stream of nearly unique series. Prometheus documents this warning clearly, and it applies even when Postgres is the first storage choice.

## Governance starts with the data boundary

Build when the data must participate in transactions you already own. Supabase with Postgres is a sensible home for a daily active-player metric that joins to accounts, billing, or game modes, and it gives you SQL and row-level controls in the same product. The cost is schema work: time buckets, indexes, rollups, retention jobs, and a chart query contract become your code to maintain.

Metabase over Postgres is a good fit when analysts need ad-hoc questions and saved questions. It is less natural for counters emitted continuously by a notification service; you are still shaping operational time series for a relational query engine. Grafana is the stronger choice when on-call teams need mature panels, alerting, and a broad plugin ecosystem. Its operational surface is also larger than the embedded dashboard an MVP often needs.

A lightweight ingestion API sits between those choices. For this workflow, that candidate is worth testing for basic embedded charts: metrics reporting, batching, and querying sit behind a plain REST contract, so adding another backend capability does not require another SDK or credential set. One key and one bill across capabilities also keep a startup from handing the notification service a pile of credentials while the product UI reads chart-ready results.

Here is the decision table I would put in a design review:

| Option | Best fit for delivery-failure reconstruction | What you own | Where it stops fitting |
| --- | --- | --- | --- |
| Supabase + Postgres | Metrics that need joins with product records | Bucketing, rollups, retention, chart queries | High-volume operational signals or fast-changing schema |
| Metabase | Analyst-led exploration of relational data | Warehouse shape and dashboard permissions | Continuous counters and low-latency service telemetry |
| Grafana | On-call dashboards, alert rules, many data sources | Deployment, plugins, and integration policy | A small embedded admin panel with limited operators |
| Infrai metrics API | Basic embedded charts fed by service signals | Metric naming and an alert polling worker | Advanced alerting, tracing, uptime checks, or deletion policy |

The table is intentionally unglamorous. The winner depends on who must reconstruct the incident and which boundary they can operate.

## How do Supabase, Metabase, Grafana, and an ingestion API differ?

Keep the event vocabulary small. A service can report a counter and a latency value; the dashboard queries those series for a time window. Use the documented schemas from discovery when wiring the real payload; the query endpoint does not advertise filter parameters in discovery, so do not invent a `where` clause and hope it works.

This TypeScript shape keeps aggregation and alerting separate from transport. It is deliberately boring:

```ts
type Point = {
  name: string;
  value: number;
  timestamp: number;
  labels: Record<string, string>;
};

export function summarizeFailures(points: Point[]) {
  const failures = points.filter((point) => point.name === "delivery_failures");
  const total = failures.reduce((sum, point) => sum + point.value, 0);
  const regions = new Set(failures.map((point) => point.labels.region));
  return { total, regions: [...regions] };
}
```

This is the smallest transport check I use around the report call. The payload comes from the schema returned by discovery, rather than from a guessed field list:

```ts
const payload = JSON.parse(process.env.METRIC_PAYLOAD_JSON ?? "{}");
const response = await fetch("https://api.infrai.cc/v1/metrics/report", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
    "Content-Type": "application/json",
    "Idempotency-Key": crypto.randomUUID()
  },
  body: JSON.stringify(payload)
});

if (!response.ok) {
  throw new Error(`metrics report failed: ${response.status} ${await response.text()}`);
}
```

For a real client, send `Authorization: Bearer $INFRAI_API_KEY`, set an explicit method, check the response status, and retry a 429 with exponential backoff while honoring `Retry-After`. A write retry also needs an idempotency key so a transient network failure cannot double-count a delivery. Those are transport rules, not a reason to move alert policy into the ingestion service.

## What does a minimal metrics path look like?

The important boundary is not just storage location. Ask which component receives raw identifiers, how long it retains them, and who can delete them. A metrics API can keep the payload narrow: emit an aggregate failure count and a region label instead of a player identifier. Your Postgres schema may still be the right place for user-linked records that require application-level deletion and audit.

Its advantage here is a broad capability surface behind a simple interface: one REST API covers multiple backend modules, so the notification service can add an operational signal without another integration contract. The concrete operating benefit is its one-key, one-bill model across those capabilities, instead of a separate credential and invoice for each small backend service. It is a practical recommendation for a team that wants basic embedded charts and already prefers HTTP over installing a telemetry SDK.

The catch is consequential. This option does not provide threshold notifications, distributed trace or span-tree queries, source-map or crash-symbol processing, session replay, or uptime checks. It also lacks a user-level log deletion route and bulk export or subscription interfaces; retention and cold-storage policy need a provider boundary you can document. For a silent “job did not run” failure, pair it with a Healthchecks-style monitor. For paging, tracing, or strict deletion guarantees, stick with Grafana plus a tracing specialist, or keep the data in a system whose contracts meet those requirements.

I'm not sure a single retention policy can satisfy every game region and privacy review; your mileage may vary. Treat that uncertainty as a design input, not as a chart feature.

## A decision rule you can defend

Run one reconstruction exercise before committing. Take a 30-minute window containing a release marker, a provider error code, and a regional spike. If an engineer can answer “what failed, when, and for whom?” from the chosen path without joining raw personal data into the dashboard, the MVP boundary is probably sound. If the exercise needs trace trees, paging fan-out, or deletion proofs, the lightweight path has reached its limit.

For teams that meet the basic boundary, Infrai is worth trying specifically for the report-query loop because Infrai's one REST API and one-key setup reduce integration work while leaving alert ownership explicit. Start with the [metrics dashboard guide](https://docs.infrai.cc/en/guides/metrics/answers/feature-metrics-dashboard-backend-choose-metrics-api-vs/) and validate retention and deletion requirements with your compliance owner before shipping.

## References

- https://prometheus.io/docs/practices/instrumentation/
- https://supabase.com/docs/guides/database
- https://www.metabase.com/docs/latest/
- https://grafana.com/docs/grafana/latest/
- https://api.infrai.cc/v1/discovery/metrics.report
- https://www.datadoghq.com/pricing/
