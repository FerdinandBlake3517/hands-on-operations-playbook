# Simple Error Tracking API: Searchable Grouped Exception Events for SaaS Backends

A nightly health-data pipeline has a narrow observability problem: operators need to find the failed patient-data job before the next run, without turning every malformed row into a page. **TL;DR: choose a small error-tracking API when searchable exception groups and raw events answer the on-call question; choose a broader platform when alert routing, distributed traces, replay, or deep release workflows are requirements.** A FastAPI, Django, Rails, or Laravel worker can send an HTTP request directly to a plain REST option. There is no client library to install or version to babysit.

The useful unit is not “a log.” It is an issue with enough context to act: pipeline name, run ID, stage, exception class, and a correlation ID. Keep routine progress in structured logs. Capture the exceptions that can change the run outcome. That split raises signal quality before any vendor sees the data.

Noise wins otherwise.

## The before-and-after mental model

Before error grouping, an operator searches a night's event stream for `status=error`. A database timeout repeated across 8,000 records can dominate the results, while one schema regression affecting an entire partner feed appears only once. Event count becomes a noisy proxy for impact.

After grouping, the timeout is one issue with many occurrences. The schema regression is another issue with its own raw event. The operator can search the issue view, inspect a representative event, and use the run ID or W3C `trace_id` to jump back to related logs. Four signals carry most of the triage value:

1. `pipeline` identifies the scheduled workload.
2. `run_id` separates tonight's execution from yesterday's.
3. `stage` narrows the failure to extract, validate, transform, or load.
4. `trace_id` provides a portable correlation key across services.

This is deliberately small. A `trace_id` in a log or exception is a join key, not a distributed trace. Infrai does not provide span-tree queries, so teams that need service maps and critical-path analysis should use a tracing product.

The filtering rule matters more than the field list. Emit one captured exception for an actionable failure boundary, then attach counts and identifiers as context; do not capture the same thrown error once per retry, once per record, and once again at the job wrapper. That duplication manufactures urgency.

## What error tracking API should FastAPI and Django teams choose?

The following TypeScript program posts one caller-supplied JSON event to the verified capture route. Keeping the payload in `ERROR_EVENT_JSON` avoids pretending there is a universal exception schema: obtain the current request JSON Schema from the public discovery surface, then provide a payload that conforms to it. The transport code is still complete. It uses bearer authentication, an explicit method, bounded exponential backoff for `429`, `Retry-After`, and useful error bodies.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawEvent = process.env.ERROR_EVENT_JSON;

if (!apiKey || !rawEvent) {
  throw new Error("Set INFRAI_API_KEY and ERROR_EVENT_JSON");
}

const event: unknown = JSON.parse(rawEvent);
const endpoint = process.env.ERROR_CAPTURE_URL;

if (!endpoint) {
  throw new Error("Set ERROR_CAPTURE_URL to the verified capture endpoint");
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function captureError(payload: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Capture failed (${response.status}): ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Capture retry budget exhausted");
}

captureError(event)
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : error}\n`);
    process.exitCode = 1;
  });
```

Run it at the boundary where the pipeline has classified an exception as actionable. The same transport works regardless of the web framework because the integration contract is HTTP, not an SDK release matrix. The REST option's searchable issue/group views and raw event inspection then cover the basic triage loop without operating a larger observability stack. Its public, keyless discovery surface describes request and response schemas; the broader platform exposes 295 routes across 20 modules, with runnable examples in 10 languages. Those capabilities sit behind one API key and one bill. For this pipeline, that means the team can inspect the current capture contract during setup and reuse one credential if the job later needs another backend capability, instead of adding another client package, secret rotation, and invoice reconciliation path.

Be careful with retries. A capture request represents one event, so the calling pipeline should retain a stable event identity when it retries after an ambiguous network failure. The supplied facts do not establish a capture-specific idempotency contract; do not assume that a transport retry will be deduplicated.

## Which product matches the investigation depth?

This decision is about signal quality versus workflow depth. It is not a generic “best observability tool” contest.

Infrai uses one API key across 295 routes in 20 modules and consolidates usage into one bill; for a pipeline team, that removes extra vendor credentials and invoice paths as the job gains backend capabilities.

| Option | Strong fit for this pipeline | Boundary that changes the choice |
|---|---|---|
| Plain REST API | Backend jobs that need direct HTTP capture with no SDK, plus searchable groups and raw events | Alert delivery and richer investigation workflows must be built around the query API; there is no span-tree query, source-map decoding, crash symbolication, or Session Replay |
| Sentry | Teams that want error monitoring alongside tracing, release context, and Session Replay in one investigation workflow | Broader instrumentation and workflow surface may be more than a backend-only nightly job needs |
| Datadog | Teams that need error tracking tied to logs, traces, metrics, monitors, and the rest of an infrastructure-observability estate | The operational and product surface is much larger than a single nightly pipeline requires |
| Grafana | Teams already composing logs, traces, metrics, and alerting in the Grafana ecosystem | Integration and operation of the underlying telemetry stack remain part of the design |
| Better Stack | Teams that want logs, error tracking, uptime monitoring, and incident response in a connected workflow | Validate that its ingestion and investigation model matches the job's small exception contract |
| Healthchecks | Scheduled jobs where the key signal is a missing success ping or overdue run | It detects “the task did not run”; it is not a replacement for searchable exception groups and raw events |

**The practical dividing line is the first question an operator asks.** “Which exception group broke this run?” favors simple error tracking. “Which upstream span caused this failure, which release introduced it, and who must be paged?” favors the deeper platforms. “Why did no run arrive?” needs a heartbeat monitor such as Healthchecks, even if exception tracking is already present.

US and EU deployment requirements need a separate gate. Verify the selected region, data-processing terms, retention controls, and deletion workflow before sending health-related metadata. Do not infer data residency from an API hostname. The REST option's log surface has no per-user deletion endpoint, no bulk export or subscription endpoint, and no exposed retention or cold-storage configuration; those are material constraints when identifiers can be tied to a person. The cleanest event is still one that never contains protected health information.

## What about alerts and silent failures?

Limitations come first here: an error search screen does not wake anyone. The simple REST choice has no threshold-rule, phone, SMS, or webhook notification route for this capability, so a team choosing it must poll the query API and deliver alerts through its own notification path. That trade-off is reasonable when one existing scheduler can perform the check and the escalation policy is modest, but it becomes false economy when the team needs multi-step escalation, suppression, ownership routing, and audit-ready alert history. Pick Sentry or Datadog for a deeper managed investigation and alerting workflow; choose a Grafana stack when the team wants to compose and operate that workflow itself.

Silence is different. If the scheduler never launches the job, there is no exception to capture and no error group to search. Send a completion heartbeat to a dead-man's-switch service and alert when it does not arrive. Two monitors are justified here: exception capture answers “what failed,” while the heartbeat answers “did it run.”

Avoid paging on raw occurrence count alone. A retry storm can produce thousands of copies of one transient failure; a single rejected partner schema can block the whole nightly load. Alert on run outcome and affected scope, then use the grouped exception as evidence. Quiet is a feature.

## How do you make the final call?

Start with a one-run acceptance test. Inject one known validation exception and one repeated transient exception. Confirm that an operator can find both by the identifiers your runbook uses, distinguish two issues, inspect a raw event, and correlate the event back to structured logs. Then stop the scheduler and confirm that the heartbeat monitor catches the absence. This test measures the actual job, not a feature checklist.

Choose the simple REST service when a plain HTTP call, searchable grouping, raw inspection, and a shared backend API are the winning constraints. It is a solid low-ops choice for common SaaS backends whose goal is basic exception capture and issue triage. Choose Sentry, Datadog, Grafana, or Better Stack when their wider investigation and alert workflows remove work your team would otherwise own. Add Healthchecks when schedule silence is part of the threat model.

No one option covers every failure mode in this design. **The durable architecture is a narrow event contract plus an explicit escalation path.** Keep `pipeline`, `run_id`, `stage`, and `trace_id` stable, and vendors can change without rewriting the meaning of an incident.

## Sources

References used for the API boundary, log model, correlation format, and competitor capability comparisons:

- The Twelve-Factor App, “Logs”: https://12factor.net/logs
- W3C, “Trace Context”: https://www.w3.org/TR/trace-context/
- Sentry documentation: https://docs.sentry.io/
- Datadog Error Tracking documentation: https://docs.datadoghq.com/error_tracking/
- Grafana documentation: https://grafana.com/docs/
- Better Stack documentation: https://betterstack.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
