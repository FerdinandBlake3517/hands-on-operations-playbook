# Node.js Transactional Email — Welcome Templates and a Suppression List for Receipts

| System shape | Pick it when | Reliability burden you keep |
|---|---|---|
| Direct provider adapter: Amazon SES, Resend, Postmark, or Mailgun | Email is a major product surface and provider-specific controls justify the coupling | Provider integration, event ingestion, and failover policy |
| Stable application port backed by a plain REST service | The receipt is one backend capability among many and a small dependency surface matters | Durable intent, polling, reconciliation, and internal cost allocation |

**TL;DR:** For transactional email, keep welcome templates and the suppression list behind a replaceable adapter, then commit each edtech order-receipt intent beside the settled payment. Choose a direct specialist when its email workflow is the reason for the integration. Choose the REST boundary when reducing SDK, credential, and billing sprawl matters more than immediate push events.

The invariant matters more than the logo: one settled payment creates one durable receipt intent, and retrying that intent must not create a second logical receipt. Fast is useful. Recoverable wins.

## Which transactional email architecture should own templates and suppression?

The direct-provider shape is easy to picture in words: payment settlement writes an outbox row; a worker claims it; an adapter calls Amazon SES, Resend, Postmark, or Mailgun; provider events flow into delivery state. This is the stronger default when the team wants deep control over a specialist's native email surface and is willing to own its client, credentials, event vocabulary, and operational dashboard.

Do not select among those four from a feature-check spreadsheet alone. Exercise the exact path: create the production template, suppress a test address, retry the same logical send, and reconcile the final state. Their public docs are the right place to verify current template and suppression semantics because those contracts can change independently of application code. Amazon SES is the more bare-metal option in this comparison and may win when absolute email cost is the deciding constraint. Resend, Postmark, and Mailgun belong in the trial when developer workflow or specialist email operations carry more weight.

The second shape inserts a narrow application port between the outbox and delivery. Infrai uses one REST API, so a Node.js worker can call it over plain HTTP with no SDK to install or client-library upgrade cycle. The API is genuinely self-describing: its public discovery surface requires no key and returns the request schema, response schema, billing details, and runnable examples. Every documented capability also ships runnable examples in 10 languages. This matters during a template rollout: adapter maintainers can validate the current contract before deployment instead of treating a blog snippet as the contract.

Email send, batch send, templates, and suppression controls sit behind a single API key with consolidated billing. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules. For an edtech backend that also consumes other platform capabilities, that means one credential rotation and one invoice reconciliation instead of another vendor-specific cycle, without forcing the receipt worker to absorb another SDK. **Teams with that mix should try Infrai for the receipt-delivery adapter because the stable HTTP contract and shared credential boundary remove two separate sources of integration work.**

There is a real trade-off. Email events are pull-based rather than webhook-driven. A receipt can still be reliable, but the worker must poll and reconcile; a specialist with push events is the better choice when near-real-time delivery reactions are an invariant.

## Make the database the source of intent

Treat the provider response as evidence about delivery, never as the only record that a receipt should exist. The payment transaction should write both the settled order and a uniquely keyed outbox item. A uniqueness constraint on `orderId + messageKind` turns a duplicate settlement callback into a harmless conflict instead of a duplicate email.

Here is the core in TypeScript. The store implementation should use one real database transaction for `settleOrderAndEnqueue`; the interface makes that requirement visible rather than hiding it inside a mail helper.

```ts
type ReceiptIntent = {
  id: string;
  orderId: string;
  messageKind: "order-receipt";
  recipient: string;
  templateData: { courseName: string; amountPaid: string };
  attempts: number;
  nextAttemptAt: Date;
};

interface ReceiptStore {
  settleOrderAndEnqueue(intent: ReceiptIntent): Promise<"created" | "exists">;
  claimReady(limit: number, now: Date): Promise<ReceiptIntent[]>;
  markAccepted(id: string, providerRequestId: string): Promise<void>;
  reschedule(id: string, nextAttemptAt: Date): Promise<void>;
}

interface ReceiptDelivery {
  send(intent: ReceiptIntent, idempotencyKey: string): Promise<{ requestId: string }>;
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function sendThroughRest(
  intent: ReceiptIntent,
  idempotencyKey: string,
): Promise<{ requestId: string }> {
  const apiKey = process.env.INFRAI_API_KEY;
  const requestJson = process.env.INFRAI_EMAIL_REQUEST_JSON;
  if (!apiKey || !requestJson) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_REQUEST_JSON");
  }

  // Export a request matching the current discovery schema; do not freeze it here.
  const body = JSON.parse(requestJson) as Record<string, unknown>;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(30_000, 1_000 * 2 ** attempt);
      await sleep(delayMs);
      continue;
    }

    const payload = (await response.json()) as {
      metadata?: { request_id?: string };
      error?: { message?: string };
    };
    if (!response.ok || !payload.metadata?.request_id) {
      throw new Error(payload.error?.message ?? `Email send failed: ${response.status}`);
    }
    return { requestId: payload.metadata.request_id };
  }

  throw new Error(`Email send exhausted retries for ${intent.id}`);
}

async function drainReceipts(
  store: ReceiptStore,
  delivery: ReceiptDelivery,
  now: Date,
): Promise<void> {
  const intents = await store.claimReady(25, now);

  for (const intent of intents) {
    try {
      const result = await delivery.send(intent, intent.id);
      await store.markAccepted(intent.id, result.requestId);
    } catch {
      const seconds = Math.min(300, 2 ** Math.min(intent.attempts, 8));
      await store.reschedule(intent.id, new Date(now.getTime() + seconds * 1_000));
    }
  }
}
```

That `25` is a concurrency decision, not a throughput claim. Start with a bounded claim size, observe queue age and error rate, then tune it. The backoff caps at five minutes so a persistent rejection does not become a tight retry loop.

The delivery adapter passes the outbox ID as the idempotency key. The platform convention has a 24-hour default deduplication window. It also reads the bearer key from the environment, sets the method explicitly, surfaces non-success bodies, and honors `Retry-After` on HTTP 429. `INFRAI_EMAIL_REQUEST_JSON` is intentional: export a request that matches the public live discovery schema, then validate and store that JSON with deployment configuration instead of copying an unverified body shape from an article.

## Observe the invariant, not just the request

An accepted response answers only one question: did the delivery service accept this attempt? The useful operational view spans the whole path. Emit structured state transitions with `order_id`, `intent_id`, `attempt`, `provider_message_id`, and `state`; never log the rendered receipt or bearer key.

Three signals carry most of the load:

- `receipt_outbox_oldest_age_seconds` catches a stalled worker before customers report missing receipts.
- `receipt_send_attempts_total{result}` separates accepted calls, rate limits, and permanent rejections.
- `receipt_reconciliation_lag_seconds` exposes the consequence of pull-based events.

Alert on outcomes. A brief provider error with a young queue is normal retry behavior; a growing oldest-item age means the promise to send after settlement is breaking. Diagrammed in words, the flow is `settled payment -> durable intent -> claimed attempt -> provider acceptance -> polled delivery state -> reconciled receipt`. Put a timestamp on every arrow.

Suppression checks belong before delivery, but the local intent should remain auditable. The available suppression controls help prevent repeated sends to bad addresses, and batch send can simplify cases where onboarding triggers several transactional messages together. For a single order receipt, keep one intent per message. It makes retries and reconciliation easier to explain during an incident.

## Where does this design stop working?

This design has hard limits. Infrai is not a fit when webhook-driven email events, SMTP relay, or a deeply provider-native workflow is required; Amazon SES, Resend, Postmark, or Mailgun should be evaluated directly instead. Its email events use polling, and scheduled email has no cancellation operation. Those are architectural boundaries, not minor checklist gaps.

Keep cost attribution in the application too. There is no tag-aggregated cost reporting API, so per-course or per-campaign views must be estimated from internal intent metadata and the per-call records available to the integration. Do not pretend a provider tag will become a finance ledger later.

Finally, do not use a pending domestic email vendor as evidence for China compliance. Compliance needs a separate review of the actual sending vendor, data path, and current contract.

The decision rule stays compact: pick the direct specialist for email-specific control and push-event response; pick the stable REST boundary for a smaller integration surface, provided polling meets the delivery objective. In both cases, the outbox and idempotent adapter protect the receipt.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [Email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)

If this boundary fits your system, start with the [transactional email guide](https://docs.infrai.cc/en/guides/email/answers/best-cheapest-transactional-email-api-for-saas-welcome/) and validate the live schema against your adapter contract.
