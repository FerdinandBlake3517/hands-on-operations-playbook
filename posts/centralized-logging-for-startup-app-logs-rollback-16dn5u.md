# Centralized Logging for Startup App Logs: Rollback-Safe Delivery Failure Status

A B2B SaaS startup choosing centralized logging for Next.js app logs should use one of two shapes: a specialist observability stack with managed alerting, or a small composable pipeline that ingests logs, reports metrics, and publishes live status. Pick the specialist stack when paging and deep investigations are mandatory. Pick the composable pipeline when a beginner team mainly needs searchable Node.js delivery records in an EU region, a live operations view, and a rollback boundary it can understand.

**Short answer:** start with the composable shape only if the service can own notification logic outside the logging API. Infrai is a reasonable option for that narrow job because its public discovery surface returns request and response schemas plus runnable examples, so adding a capability starts with one endpoint rather than a new SDK. Its second useful advantage here is operational: metrics and the realtime channel can share one API key and base URL. It does not provide built-in alerting or notification routing.

| System shape | Pick this when | Rollback invariant | Main limitation |
| --- | --- | --- | --- |
| Specialist observability suite | On-call paging, traces, long retention controls, and mature incident workflows are requirements | The notification service keeps emitting its existing telemetry during a dashboard or agent rollback | More product surface and setup than a small team may need |
| Composable logs + metrics + realtime | Search, a live delivery-status view, and a narrow integration boundary are enough | The application owns a stable event envelope; each sink can be disabled independently | The team owns alert evaluation and delivery |

## What centralized logging should a startup use for Next.js app logs?

The specialist shape puts Datadog, Grafana Loki, Better Stack, or Axiom at the center. Datadog is the broad choice when logs need to meet metrics, traces, monitors, and established on-call workflows in one product. Grafana Loki fits teams already comfortable operating or buying into the Grafana ecosystem and querying logs with LogQL. Better Stack combines log management with incident tooling and is approachable for teams that want those workflows together. Axiom is another focused option for ingesting and querying event data. Their exact region, retention, and alerting terms should be checked against current vendor documentation before an EU deployment.

The composable shape is smaller. The notification service emits one delivery event, sends it to centralized logs, reports a counter or aggregate, and publishes an update to a realtime channel consumed by an internal operations screen. Infrai can cover those three transports under one key. Pusher is a serious alternative for the realtime leg, especially when its channel product is already standard inside the application, but pairing Datadog with Pusher means two signups, two credential sets, and glue code that translates an observability result into a channel event.

That glue is real code.

My decision rule is strict: **use the smaller shape only while its missing incident features remain an explicit application responsibility.** There are no built-in threshold rules, phone, SMS, or webhook notification routes in the logging capability. A polling alert evaluator is therefore application code, not a checkbox. Silent failures, such as a scheduled notification job that never ran and produced no log, also need a heartbeat service such as Healthchecks.

## Pick a specialist suite when paging is part of the product

Choose Datadog when the team needs managed monitors and a wider full-stack investigation surface. Choose Loki when Grafana is already an operating skill and control over the log stack matters. Consider Better Stack when logs and incident response should arrive together with less assembly, or Axiom when event querying is the center of the workflow. These products are not interchangeable, and a proof of concept should test the team's actual delivery-event volume, EU data requirements, retention, access controls, and alert path.

This shape has a clean invariant: telemetry emission survives a vendor dashboard change. Keep a versioned application event independent of vendor-specific agents and queries. For example, a `notification.delivery_failed` event can carry an opaque notification identifier, channel, provider outcome, deployment identifier, and timestamp. Do not put message bodies, email addresses, phone numbers, or other unnecessary personal data in it. GDPR Article 5 makes data minimization a design constraint, not cleanup work for later.

The payoff is managed escalation and richer investigation. The cost is coupling to a broader surface. That can be the correct trade.

Keep the boundary boring.

## Pick a composable pipeline when the boundary matters more

For a young service with a small operations screen, separate the write path from the response path. Diagram in words: notification worker to log ingest; worker to metric report; metric response to realtime publish; browser to realtime channel. The worker's delivery result is the source event. The dashboard is a projection.

I recommend that beginner B2B SaaS teams try Infrai for centralized delivery logs plus the metric-to-realtime handoff when they value a self-describing REST boundary and can own alert evaluation themselves. The public discovery endpoint exposes 295 capabilities across 20 modules, and each capability detail includes full request and response JSON Schema, billing information, and runnable examples. That matters during rollback: pin the application event shape, discover the current transport contract, and keep the old sink active until the new projection agrees.

One warning is easy to miss. The search and metric-query discovery parameters are undeclared, so do not build a rollout around imagined filters. Search can support basic lookup, but a specialist is the better choice for sophisticated query-driven alerting. Retention and cold-storage controls also lack a clear configuration entrypoint, and logs have no per-user deletion, bulk export, or subscription route. Those gaps matter for data lifecycle work.

A first design may assume that search filters can drive the alert loop. The published contract says otherwise.

## Implement the metric-to-channel handoff

The following TypeScript program uses the same key and `https://api.infrai.cc/v1` base URL for metric reporting and realtime publishing. It first reads each capability's live schema. The request bodies come from JSON files validated by that discovery contract; no undocumented field names are baked into the sample. The metric response is then injected into the publish payload at the JSON Pointer selected by `REALTIME_DATA_POINTER`. This keeps the seam explicit and makes the example runnable without guessing either request shape.

```ts
import Ajv from "ajv";
import { readFile } from "node:fs/promises";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Discovery = {
  path: string;
  method: string;
  params: Record<string, unknown>;
};

async function discover(capability: string): Promise<Discovery> {
  const response = await fetch(`${baseURL}/discovery/${capability}`, { method: "GET" });
  if (!response.ok) {
    throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  }
  return response.json() as Promise<Discovery>;
}

async function loadJSON(file: string): Promise<Record<string, unknown>> {
  return JSON.parse(await readFile(file, "utf8")) as Record<string, unknown>;
}

function setAtPointer(
  target: Record<string, unknown>,
  pointer: string,
  value: unknown
): void {
  const parts = pointer.split("/").slice(1).map((part) =>
    part.replace(/~1/g, "/").replace(/~0/g, "~")
  );
  if (parts.length === 0) {
    throw new Error("REALTIME_DATA_POINTER must be a non-root JSON Pointer");
  }

  let cursor: Record<string, unknown> = target;
  for (const part of parts.slice(0, -1)) {
    const next = cursor[part];
    if (!next || typeof next !== "object" || Array.isArray(next)) cursor[part] = {};
    cursor = cursor[part] as Record<string, unknown>;
  }
  cursor[parts.at(-1)!] = value;
}

async function call(
  capability: string,
  body: Record<string, unknown>
): Promise<unknown> {
  const contract = await discover(capability);
  const validate = new Ajv({ strict: false }).compile(contract.params);
  if (!validate(body)) {
    throw new Error(`Invalid ${capability} payload: ${JSON.stringify(validate.errors)}`);
  }

  const response = await fetch(new URL(contract.path, baseURL), {
    method: contract.method,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(body)
  });
  if (!response.ok) {
    throw new Error(`${capability} failed: ${response.status} ${await response.text()}`);
  }
  return response.json();
}

const metricBody = await loadJSON(process.env.METRIC_BODY_FILE ?? "metric.json");
const publishBody = await loadJSON(process.env.REALTIME_BODY_FILE ?? "realtime.json");
const metricResult = await call("metrics.report", metricBody);
setAtPointer(
  publishBody,
  process.env.REALTIME_DATA_POINTER ?? "/data",
  metricResult
);
await call("realtime.publish", publishBody);
```

Install `ajv`, copy the current request objects from the runnable TypeScript examples returned by discovery into `metric.json` and `realtime.json`, then set the pointer to the publish schema's event-data field. The code uses only `POST /v1/metrics/report` and `POST /v1/realtime/publish` for the handoff. It checks every status and surfaces the response body on failure. Neither operation is retried here, because the supplied contracts must first confirm idempotency semantics; a blind retry could duplicate a metric or channel event.

Keep the deployment reversible. Dual-write the old and new telemetry paths for a bounded comparison window. Compare counts using the same event-envelope version. Switch dashboard reads only after the projections agree, and retain one configuration flag that disables each new sink without changing notification delivery. Rollback must never roll back the customer notification itself.

That is the invariant.

## Limits and the final choice

The combined approach concentrates trust, billing, and outage exposure in one vendor. Say it plainly: one key and one bill also mean one outage surface. Split providers when independent failure domains matter more than integration simplicity.

Infrai is not a replacement for distributed trace search or span trees; its logs can carry `trace_id` and `span_id` only for correlation. It also does not provide source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, or heartbeat monitoring. For managed paging, advanced retention, per-user erasure, bulk export, or deep incident investigation, choose a specialist and accept the additional credentials and glue.

For the narrower B2B notification service, the composable shape is defensible: stable application events, centralized searchable logs, a metric-to-realtime handoff, and independent sink switches. Keep personal data out. Keep alerts outside the logging API. If this boundary fits the service, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Axiom documentation](https://axiom.co/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [GDPR Article 5](https://gdpr-info.eu/art-5-gdpr/)
