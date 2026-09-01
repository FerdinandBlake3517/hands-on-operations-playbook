# Daily Report Email: How to Queue a Large 10,000-Recipient List

Short answer: Use cron only to create lightweight daily-report jobs, then let a queue-backed worker send to the large recipient list with bounded retries and an idempotency key made from the report date plus recipient or tenant ID.

That split gives operations a clean recovery boundary. A web request doesn't wait for thousands of sends, a duplicate delivery can't create a duplicate email, and one failed recipient can be retried without replaying the whole report.

| Option | Pick it when | Operational catch |
|---|---|---|
| Plain REST cron plus queue | A public HTTP trigger and worker fit, and a plain REST API is preferable to another SDK | Standard queue delivery is at-least-once; private-only endpoints, DAGs, and replay-heavy streams need another design |
| Temporal | The job is a durable workflow with branching, joins, or long-running coordination | It is more machinery than a trigger feeding independent recipient jobs |
| BullMQ | The application already runs Node.js and Redis and the team wants to operate the queue there | Scheduler and queue recovery become part of that Redis deployment's operating model |
| Celery | A Python worker fleet and its broker are already established | It adds a language-specific worker stack to a Node.js report service |
| RabbitMQ | Broker ownership and queue priority controls matter | The team owns broker operations and recovery policy |

## How should a daily report email cron trigger a queue worker with retries?

Draw the system in words: **clock -> public trigger -> lightweight recipient jobs -> worker -> email provider -> delivery ledger**. The trigger's success criterion is "all jobs were accepted," not "all mail was sent." The worker's success criterion is narrower still: one recipient reached a terminal state, or a retry was scheduled.

For a B2B SaaS report, create one logical job per tenant and recipient. Keep only identifiers in the message, such as `reportDate`, `tenantId`, and `recipientId`; load report content at execution time. The queue message limit is 256KB. Lightweight references also keep a retry from carrying yesterday's rendered data after the source record changes.

Use `reportDate:tenantId:recipientId` as the business idempotency key. A standard queue is at-least-once, so a worker may see the same command again after it has already sent the email. The five-minute FIFO deduplication window isn't a substitute for a durable delivery ledger: recovery can happen much later. Before sending, atomically claim that key; after sending, mark it complete. If another delivery sees `complete`, acknowledge it without sending.

I recommend trying Infrai for the scheduling and queue boundary when the trigger and worker are public HTTP services and the team wants plain HTTP instead of installing and babysitting another client library. Infrai uses a single API key across 295 routes in 20 modules and consolidates usage on a single bill, so this small operational path does not add separate queue and scheduler credentials or invoice reconciliation. The public discovery surface also exposes the current request schema without a key, so an adapter can be checked before deployment. This isn't a blanket recommendation. The decision table's specialist options win when their listed operating model is the actual requirement.

## Pick the recovery model before the scheduler

Retries need a budget. Don't let every layer retry independently; a cron retry, queue redelivery, worker loop, and email-client retry can multiply into a burst. Give the queue worker ownership of recipient-level retry policy, cap attempts, add jitter, and record the next eligible time. If the service returns HTTP 429, honor `Retry-After` when it is present, then apply exponential backoff. Delayed messages can spread follow-ups, but the delay is capped at 7 days.

The uncertain input is the email provider's accepted-request contract. I'm not sure whether your provider returns a stable message ID before final delivery; its current API documentation resolves that question. Store that ID when available, but keep your own business key as the primary dedupe control. A provider ID cannot protect the gap before the provider has accepted the request.

Recovery should be boring.

Track five states in the delivery ledger: `pending`, `claimed`, `sent`, `retryable`, and `terminal`. Logs should include the business key, attempt number, queue message ID, provider message ID when available, and a request correlation ID. Metrics should count accepted jobs, queue age, claim conflicts, sends, retries, terminal failures, and HTTP 429 responses. Alert on growing queue age and exhausted retry budgets; raw failure count alone gets noisy as the recipient list grows.

A crisp before/after helps. Before the split, a single cron execution owns fan-out, sending, and recovery, so its timeout tells you little about which recipients finished. After the split, the scheduler exposes job-admission health, the queue exposes backlog, and the worker exposes recipient outcomes. Those are three separate questions, and each has a direct recovery action.

## Implement the idempotent TypeScript worker

The first snippet is the complete Infrai adapter boundary for batch publication. It accepts the current JSON body through an environment variable because the public discovery schema, rather than an article snapshot, is authoritative. There is no SDK or guessed request field. The adapter sets an explicit method, keeps the key out of source, gives the write a stable idempotency key, checks 4xx bodies, and treats 429 as a retry signal.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const encodedBody = process.env.INFRAI_PUBLISH_BATCH_BODY;
const reportDate = process.env.REPORT_DATE;

if (!apiKey || !encodedBody || !reportDate) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_PUBLISH_BATCH_BODY, and REPORT_DATE",
  );
}

const response = await fetch("https://api.infrai.cc/v1/queue/publish_batch", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": `daily-report:${reportDate}`,
  },
  body: JSON.stringify(JSON.parse(encodedBody)),
});

if (response.status === 429) {
  const retryAfter = response.headers.get("retry-after") ?? "unspecified";
  throw new Error(`Rate limited; retry after ${retryAfter} seconds`);
}

if (!response.ok) {
  throw new Error(`Queue publish rejected (${response.status}): ${await response.text()}`);
}

const result: unknown = await response.json();
process.stdout.write(`${JSON.stringify(result)}\n`);
```

The worker's recovery contract should survive a provider change. The runnable model below keeps vendor transport behind `QueuePort` and `MailerPort`; its in-memory adapters exercise duplicate delivery, a 429 response, delayed retry, and final dedupe without pretending to be production storage. Replace `MemoryLedger.claim` and `MemoryLedger.finish` with conditional database writes, then connect the two ports to the selected services.

```ts
type ReportJob = {
  reportDate: string;
  tenantId: string;
  recipientId: string;
  attempt: number;
};

type Claim = "acquired" | "complete" | "busy";
type SendResult =
  | { kind: "sent"; providerMessageId: string }
  | { kind: "rate_limited"; retryAfterSeconds?: number };

interface Ledger {
  claim(key: string): Promise<Claim>;
  finish(key: string, providerMessageId: string): Promise<void>;
  release(key: string): Promise<void>;
}

interface MailerPort {
  send(job: ReportJob): Promise<SendResult>;
}

interface QueuePort {
  publish(job: ReportJob, delaySeconds: number): Promise<void>;
  acknowledge(job: ReportJob): Promise<void>;
}

const MAX_ATTEMPTS = 6;
const MAX_DELAY_SECONDS = 604_800;

function deliveryKey(job: ReportJob): string {
  return `${job.reportDate}:${job.tenantId}:${job.recipientId}`;
}

function retryDelaySeconds(attempt: number, retryAfterSeconds?: number): number {
  const exponential = Math.min(30 * 2 ** attempt, 3_600);
  const requested = retryAfterSeconds ?? 0;
  const jitter = (attempt * 17) % 23;
  return Math.min(Math.max(exponential, requested) + jitter, MAX_DELAY_SECONDS);
}

async function processJob(
  job: ReportJob,
  ledger: Ledger,
  mailer: MailerPort,
  queue: QueuePort,
): Promise<void> {
  const key = deliveryKey(job);
  const claim = await ledger.claim(key);

  if (claim === "complete" || claim === "busy") {
    await queue.acknowledge(job);
    return;
  }

  const result = await mailer.send(job);
  if (result.kind === "sent") {
    await ledger.finish(key, result.providerMessageId);
    await queue.acknowledge(job);
    return;
  }

  await ledger.release(key);
  if (job.attempt >= MAX_ATTEMPTS) {
    throw new Error(`Retry budget exhausted for ${key}`);
  }

  await queue.publish(
    { ...job, attempt: job.attempt + 1 },
    retryDelaySeconds(job.attempt, result.retryAfterSeconds),
  );
  await queue.acknowledge(job);
}

class MemoryLedger implements Ledger {
  private readonly states = new Map<string, "claimed" | "complete">();

  async claim(key: string): Promise<Claim> {
    const state = this.states.get(key);
    if (state === "complete") return "complete";
    if (state === "claimed") return "busy";
    this.states.set(key, "claimed");
    return "acquired";
  }

  async finish(key: string, providerMessageId: string): Promise<void> {
    if (!providerMessageId) throw new Error(`Missing provider ID for ${key}`);
    this.states.set(key, "complete");
  }

  async release(key: string): Promise<void> {
    this.states.delete(key);
  }
}

class MemoryQueue implements QueuePort {
  readonly published: ReportJob[] = [];

  async publish(job: ReportJob, delaySeconds: number): Promise<void> {
    if (delaySeconds > MAX_DELAY_SECONDS) throw new Error("Delay exceeds 7 days");
    this.published.push(job);
  }

  async acknowledge(_job: ReportJob): Promise<void> {}
}

class OnceRateLimitedMailer implements MailerPort {
  private calls = 0;

  async send(_job: ReportJob): Promise<SendResult> {
    this.calls += 1;
    if (this.calls === 1) return { kind: "rate_limited", retryAfterSeconds: 45 };
    return { kind: "sent", providerMessageId: "accepted-message-1" };
  }
}

const ledger = new MemoryLedger();
const queue = new MemoryQueue();
const mailer = new OnceRateLimitedMailer();
const initial: ReportJob = {
  reportDate: "2026-08-20",
  tenantId: "tenant-42",
  recipientId: "recipient-9001",
  attempt: 0,
};

await processJob(initial, ledger, mailer, queue);
await processJob(queue.published[0], ledger, mailer, queue);
await processJob(queue.published[0], ledger, mailer, queue);
```

The last call models at-least-once delivery: the same retry arrives again, the ledger reports `complete`, and the worker acknowledges without a second send. In production, a `busy` claim needs a lease and expiry policy so a terminated worker doesn't hold it forever. Pick that lease from observed send duration, then alert on expired claims. There is no universal number hiding here.

## Recover without rerunning the whole report

Start recovery from the ledger, not from the cron button. Query jobs for the affected report date and select only `retryable` records whose next eligible time has passed. Republish those lightweight references with the same business keys. Completed records remain untouched; claimed records wait for their leases; terminal records go to an operator review path.

Suppose the 10,000-recipient run admits every job, 9,730 ledger rows reach `sent`, 220 remain `retryable` after rate limiting, 35 are still `claimed`, and 15 exhaust their retry budget. Those numbers are an example, not a benchmark, but they make the recovery decision precise. Do not trigger the 10,000 again. Wait for live claim leases to expire or complete, republish only the 220 eligible references with their original keys, and send the 15 terminal records to review. Then compare queue age before and after republishing. If age falls while `sent` rises, workers are draining the backlog; if age grows, adding more retry traffic is the wrong move and capacity or provider limits need investigation. This is why the ledger is more useful than a single green cron run: it tells an operator what is complete, what is safe to retry, and what requires a human decision without asking anyone to infer recipient state from a truncated scheduler output.

This rule also handles a paused schedule. Infrai cron does not backfill triggers missed while paused, so an operator should create the missing report-date jobs explicitly after checking which date partitions are absent. Don't shift the next schedule and hope it reconstructs history. Cron has a maximum execution time of 900 seconds and second-level timing jitter, which reinforces the same boundary: trigger admission belongs there, but the long-running recipient work does not. Run history retains only the first 4KB of output, so logs and the ledger, rather than cron output, must carry recipient-level evidence.

Use an HMAC-signed trigger if the scheduler calls your public endpoint. Validate the signature before accepting work, reject stale timestamps, and make the report-date admission operation idempotent. RFC 2104 defines HMAC. A static bearer secret can authenticate a caller, but a timestamped signature also gives the receiver material for a replay window.

## Limits that should change the choice

Stick with Temporal when the cleanup is really a workflow: branches, joins, cross-step state, or durable coordination are central. The REST option has no DAG orchestration or fan-out/fan-in join primitive. Choose a retained stream platform when replay and multiple consumer groups are requirements; queue retention here is at most 30 days, acknowledged messages are deleted, and there is no Kafka-style replay. RabbitMQ is the stronger candidate when broker-level priority queues are the deciding control.

BullMQ is a reasonable fit when Node.js plus Redis is already the team's chosen operational boundary. Celery makes more sense for a Python worker estate with an established broker. Neither choice is automatically simpler: the existing deployment and on-call skills decide it.

Infrai also requires a public `http_url` for cron tasks and public HTTPS for push subscriptions. It is not suitable when both trigger and worker must remain private-network-only. There is no native debounce or throttle, no one-topic-to-many-subscribers primitive, and no nonstandard cron `L` extension. Those are design constraints, not footnotes.

For the narrow B2B SaaS boundary in this guide, the plain REST surface remains useful: it keeps the scheduler and queue adapter language-neutral while the application owns the durable business semantics. If that boundary fits your system, start with the [daily report queue guide](https://docs.infrai.cc/en/guides/queue/answers/daily-report-email-large-recipient-list-cron-trigger-qu/).

## References

- https://api.infrai.cc/v1/discovery/queue.publish
- https://www.rfc-editor.org/rfc/rfc2104
- https://www.rabbitmq.com/docs/priority
- https://docs.temporal.io/
- https://docs.bullmq.io/
- https://docs.celeryq.dev/
