# Attributing Log Cost per Job: Structured Search Across Node.js, Docker, and Cron

Pick the fields that will explain your bill before you pick where the logs live. For a small SaaS running a notification service — a Node.js API, Docker workers doing the fan-out, cron jobs retrying yesterday's failures — cheap centralized logging is mostly a data-shape problem. Every store charges you for volume, and most charge again for whatever you make searchable. So the deciding constraint is attribution: if each line carries the job that produced it, one structured search answers both "why did this push fail?" and "which job is eating the ingest budget?"

Shape first. Destination second.

## What a logging bill is actually charging you for

Three levers show up in almost every pricing model, hosted or self-run: how many bytes arrive, how much of that becomes searchable, and how long you keep it. Managed vendors make the split explicit — Datadog's pricing page, for instance, prices log ingestion separately from indexed events, which is why a team can be surprised by a bill that has nothing to do with query volume. Label-indexed systems like Loki flip the shape: content stays in cheap object storage while labels carry the index, and a high-cardinality label such as `message_id` quietly rebuilds the expensive thing you were avoiding. Self-hosted Elasticsearch moves the cost into disks, cluster upgrades, and somebody's Thursday afternoon.

| Cost lever | What it charges for | Attribution field that explains it |
| --- | --- | --- |
| Ingest | Bytes accepted, before any search | `service`, `job_name` |
| Index | Fields or events made searchable | `error_code`, `channel` |
| Retention | How long searchable data lives | `level`, `tenant_id` |

None of that is knowable from a flat monthly number. A notification service is a fan-out machine, so one bad provider afternoon can multiply log volume by ten while the product does nothing new. Without a producer field on every line, the finance question and the debugging question are two different investigations.

## One envelope, three producers

Before centralization, the diagram looks like three dead ends. API writes prose to stdout; the Docker worker writes its own format with different key names; the cron container logs to a file that nobody has read since it was set up. One bill, three vocabularies, zero attribution.

After, the picture is a funnel: API + worker + cron emit one JSON envelope per event, the container runtime collects stdout, a shipper forwards it, and the store indexes a small, deliberate set of fields. Same envelope everywhere. A run of the retry job and a live API request describe themselves in the same words, which is what makes a single query work across all three.

The envelope I'd argue for on a delivery pipeline carries `service`, `job_name`, `run_id`, `channel`, `tenant_id`, `error_code`, and `attempt`. Seven fields, all low cardinality except `tenant_id` and `run_id` — keep those out of your index labels if the backend indexes labels rather than content. Everything else that feels interesting during development, including the provider's full response body, belongs in the message field or nowhere at all. OWASP's logging guidance is blunt about the "nowhere at all" case: recipient addresses, tokens, and raw payloads are a liability that also happens to be the most expensive thing you can ship, since PII usually drags stricter retention behind it.

## Sample the examples, count everything

Here's the failure mode that generates the scary invoice. A campaign fans out 50,000 pushes, an upstream provider starts rejecting them, and the worker faithfully logs every rejection with the full error object attached. That's not observability. That's paying per byte to store the same sentence 12,000 times.

The pattern that keeps the signal and drops the bill: sample the examples, count everything. Emit the first few failures per error code — enough to read a real one — then let exact counters carry the rest and flush them as one summary event when the run ends. Counts stay accurate for alerting and cost attribution. Volume stops scaling with the size of the outage.

```ts
type Channel = "email" | "push" | "sms";

type Attribution = {
  service: "api" | "worker" | "cron";
  job_name: string;
  run_id: string;
  channel: Channel;
  tenant_id: string;
};

type LogEvent = Attribution & {
  level: "info" | "warn" | "error";
  event: string;
  error_code?: string;
  attempt?: number;
  count?: number;
  duration_ms?: number;
};

function emit(event: LogEvent): void {
  process.stdout.write(`${JSON.stringify({ ts: new Date().toISOString(), ...event })}\n`);
}

const SAMPLES_PER_CODE = 5;

class DeliveryRun {
  private readonly failures = new Map<string, number>();
  private readonly startedAt = Date.now();

  constructor(private readonly at: Attribution) {
    emit({ ...at, level: "info", event: "run_started" });
  }

  failed(errorCode: string, attempt: number): void {
    const seen = (this.failures.get(errorCode) ?? 0) + 1;
    this.failures.set(errorCode, seen);
    if (seen <= SAMPLES_PER_CODE) {
      emit({ ...this.at, level: "error", event: "delivery_failed", error_code: errorCode, attempt });
    }
  }

  finish(): void {
    for (const [error_code, count] of this.failures) {
      emit({ ...this.at, level: "warn", event: "delivery_failed_total", error_code, count });
    }
    emit({ ...this.at, level: "info", event: "run_finished", duration_ms: Date.now() - this.startedAt });
  }
}
```

Twelve thousand rejections become five sampled lines plus one counter line per code. The API and the cron retry job import the same class and change one field. That's the whole trick.

Note what the code refuses to accept: there's no free-form `metadata` bag. New fields have to survive code review, which is the cheapest privacy control and the cheapest cost control you will ever ship.

## How do you search structured logs from Node.js, Docker, and cron jobs in one place?

Get all three producers onto stdout as one JSON object per line, then let the runtime do the collection. Docker's json-file driver already captures stdout per container, and any shipper can tail it; a cron job that runs inside the same image inherits the same path without special handling. Don't build a second delivery mechanism inside the app — an HTTP logger in-process will drop events exactly when the process is dying, which is the moment you care about.

Then make the search self-serve, because that's what decides whether anyone uses it. An engineer should be able to filter `job_name = "retry_failed_deliveries"` and `error_code = "PROVIDER_TIMEOUT"` over the last day without asking a platform team for a saved view. Roll it out one producer at a time and verify each with a deliberately harmless staging event that you then find in the search UI. That proves the whole path, not just that `stdout.write` executed.

One boundary worth naming: a run that never starts emits nothing, so absence is not searchable. Pair the scheduled jobs with a heartbeat monitor that knows when a ping was due. Log search explains what ran. Heartbeats reveal what didn't.

## Where this stops being a good fit

"Sampling will hide the one failure I need." It can, and that's a real trade-off. The mitigation is to sample per error code rather than globally, keep counters exact, and add an escape hatch: a debug flag on a tenant that disables sampling for that tenant's next run. If your incident review keeps reaching for a line that sampling dropped, raise the per-code sample before you abandon the pattern.

"Self-hosting is always cheaper." Sometimes. The storage math often favors it, and OpenTelemetry's log data model means the envelope above ports either way, which is the point of standardizing the shape first. The catch is that a search cluster is a system with its own on-call, upgrades, and disk-pressure incidents, and a three-person team paying for that in weekends hasn't saved money — it has moved the line item somewhere nobody measures. Stick with a managed store while nobody owns the cluster; revisit when ingest is large enough that an engineer-week per month is genuinely the smaller number.

I'm not sure where that crossover sits for any particular team, and honestly the answer moves with headcount more than with data volume. What holds regardless: an event contract with attribution fields survives a change of destination, and a flat pile of prose logs doesn't.

## References

- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- Datadog pricing (log ingestion and indexing are billed separately): https://www.datadoghq.com/pricing/
- Grafana Loki labels and cardinality guidance: https://grafana.com/docs/loki/latest/get-started/labels/
- Docker json-file logging driver: https://docs.docker.com/engine/logging/drivers/json-file/
- OpenTelemetry logs data model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
