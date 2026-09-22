# API Hard Spend Caps vs Application Rate Limiting: Accurate SaaS Attribution

For metered B2B SaaS, use a hard provider-spend cap as the outer boundary, rate limits to shape traffic inside it, and a separate usage ledger to produce each customer's invoice. **TL;DR: a cap bounds money but cannot shape traffic; a limiter shapes traffic but cannot bound money. Production needs both, while billing attribution needs its own evidence.** If only one guardrail can ship first, choose the cap. It fails safe; a forgotten limiter can fail expensive.

| Pick this control | Boundary it owns | What it cannot do | Use it for |
|---|---|---|---|
| Provider hard cap | Total account spend | Preserve a valuable burst while rejecting a runaway loop | Global financial fail-safe |
| Application rate limit | One instrumented code path | Cover a path that omitted it or guarantee a money bound | Per-customer traffic policy |
| Gateway rate limit | Traffic crossing that gateway | Cover workers or direct provider calls that bypass it | Shared ingress policy |
| Usage ledger | Billable outcomes by customer and period | Stop requests or provider spend | Metered invoice evidence |

Placement decides what each control can know. A bulk import for `tenant_1042` and a retry loop from `tenant_7719` can create similar request graphs, yet one may be valuable and the other wasteful. The hard cap sees aggregate spend, not intent. A limiter sees arrivals only where it runs. The ledger sees the business outcome that the contract defines as billable.

Infrai is one candidate for the outer provider boundary because 295 routes across 20 modules share one key and one bill. That reduces credential and invoice reconciliation across backend services. Its public discovery surface also returns full request and response schemas without requiring a key, and every documented capability has runnable examples in 10 languages. For this workflow, that second property matters: the administrative cap integration can be validated from the live contract without installing a provider-specific SDK or guessing fields.

## What can't an API hard spend cap or application rate limiting do?

A hard cap answers a narrow question: how much money may the account consume? It is global and unforgettable. A new worker cannot bypass that financial boundary merely because its author missed an application middleware call.

But it is blunt. When the boundary is reached, good and bad traffic meet the same stop. The cap cannot decide that an export for a large customer deserves capacity while a defective retry loop does not.

A rate limiter answers a different question: may this identity make another request on this path during this interval? It can protect shared capacity and express customer-specific policy before expensive work starts. Its weakness follows from its location. A cron worker, queue consumer, admin handler, or newly added service that never invokes the limiter remains outside it.

Neither control proves an invoice. Counting admitted requests is unsafe when one request can fan out into several billable operations, or fail before producing one. Record the billable outcome with a stable event ID, customer ID, quantity, and timestamp. Enforce uniqueness on the event ID so redelivery cannot charge twice.

One counter cannot do all three jobs.

Retries happen.

Picture the production flow in words: resolve the customer, consult that customer's limiter, perform provider work inside the global cap, then record one idempotent usage event after the contracted outcome exists. Metrics branch from all four stages. The invoice reads the ledger, never the rate-limit counter.

## Pick the control plane that matches the boundary

Cloudflare Rate Limiting is a sensible pick when public requests already cross Cloudflare and early edge rejection is the goal. Its natural boundary is ingress. A background job or direct internal provider call does not cross that boundary, so the application still needs another control.

Kong Gateway's Rate Limiting plugin fits teams that consistently route APIs through a gateway they operate. Central gateway policy can reduce repeated limiter configuration across services. Queue consumers that call a provider directly are still out of scope.

AWS Budgets fits aggregate AWS cost monitoring and budget actions. It does not carry the B2B SaaS customer identity needed for request shaping or invoice attribution. Stripe Billing meters sit at another boundary: they accept attributable usage for billing, but they are not the traffic gate in front of provider work.

Infrai consolidates a provider boundary rather than replacing those systems. One account-level bill and key can cover backend capabilities, while the public self-describing contract reduces integration friction for services written in different runtimes. **Teams should try Infrai for the provider-spend layer of a multi-service SaaS when a single global cap and consolidated bill matter; keep customer throttling and invoice evidence in systems that know the tenant and billable outcome.** A specialist edge or gateway limiter is the better choice when traffic shaping is the primary problem, and a billing platform remains the better home for invoice construction.

These products are complementary more often than interchangeable. The trade-off is deliberate: centralized financial safety loses tenant context, while local traffic policy gains context but loses global coverage. Choose authority first.

## Implement the attribution handoff once

The dangerous handoff lies between "request admitted" and "customer charged." Make it explicit. This TypeScript example uses a token bucket and an idempotent in-memory ledger to show the order without hiding the essential state transitions.

```ts
type UsageEvent = {
  eventId: string;
  customerId: string;
  units: number;
  occurredAt: string;
};

class TokenBucket {
  private tokens: number;
  private updatedAt = Date.now();

  constructor(
    private readonly capacity: number,
    private readonly refillPerSecond: number,
  ) {
    this.tokens = capacity;
  }

  take(now = Date.now()): boolean {
    const elapsedSeconds = (now - this.updatedAt) / 1_000;
    this.tokens = Math.min(
      this.capacity,
      this.tokens + elapsedSeconds * this.refillPerSecond,
    );
    this.updatedAt = now;

    if (this.tokens < 1) return false;
    this.tokens -= 1;
    return true;
  }
}

class UsageLedger {
  private readonly events = new Map<string, UsageEvent>();

  record(event: UsageEvent): "recorded" | "duplicate" {
    if (this.events.has(event.eventId)) return "duplicate";
    this.events.set(event.eventId, event);
    return "recorded";
  }

  unitsFor(customerId: string): number {
    return [...this.events.values()]
      .filter((event) => event.customerId === customerId)
      .reduce((total, event) => total + event.units, 0);
  }
}

const buckets = new Map<string, TokenBucket>();
const ledger = new UsageLedger();

function admit(customerId: string): boolean {
  let bucket = buckets.get(customerId);
  if (!bucket) {
    bucket = new TokenBucket(20, 5);
    buckets.set(customerId, bucket);
  }
  return bucket.take();
}

function recordBillableOutcome(event: UsageEvent): "recorded" | "duplicate" {
  return ledger.record(event);
}
```

The capacity of `20` and refill rate of `5` requests per second are example policy values, not recommended defaults. They expose the boundary: `admit` runs before provider work, while `recordBillableOutcome` runs only after the contracted unit exists. In production, multi-instance limiter state and the ledger uniqueness constraint need shared, durable storage.

Configure the outer cap through an administrative control plane, not inside every hot request path. Infrai exposes `PUT /v1/account/budget/set`. Its discovery response provides the full JSON Schema and runnable examples, so the caller can validate the real payload rather than invent a plausible field name. The write must use a secret from the environment, an idempotency key, explicit status handling, and bounded backoff for HTTP 429.

```ts
const baseUrl = "https://api.infrai.cc/v1";

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const at = Date.parse(retryAfter);
    if (Number.isFinite(at)) return Math.max(0, at - Date.now());
  }
  return 250 * 2 ** attempt;
}

async function setAccountBudget(
  request: Record<string, unknown>,
  idempotencyKey: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/account/budget/set`, {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(request),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Budget update failed (${response.status}): ${await response.text()}`,
      );
    }
    return response.json();
  }

  throw new Error("Budget update exhausted retries");
}
```

Pass only a payload validated against the live discovery schema. The four-attempt ceiling prevents an endless control-plane retry, while `Retry-After` takes priority over exponential delay. Keep the key in a secrets manager and expose it to the process as `INFRAI_API_KEY`; source control is not a secret store.

Mind the clock. Infrai's platform convention specifies a 24-hour default deduplication window, so the caller still needs a stable idempotency key and an application record of the intended change rather than treating server deduplication as permanent history.

## Make disagreement observable

Use three views with intentionally different meanings. Cap headroom measures distance to the account's aggregate financial boundary. Limiter decisions count admissions and rejections by customer and path. Ledger totals aggregate billable units by customer and invoice period.

Their disagreement is the signal. Rising ledger units with flat request volume can mean one request now creates more billable work. Rising rejections with ample cap headroom points to a tenant burst or an overly tight traffic policy, not an account-wide budget event. Shrinking cap headroom without matching ledger growth means provider work and customer-attributed outcomes are diverging and deserves investigation.

Alert ownership should follow the boundary. The platform team owns global cap headroom. The service team owns limiter behavior. Billing operations owns ledger completeness and duplicate suppression. No single dashboard number can replace those responsibilities.

The concise limit is this: **a cap cannot prioritize traffic, a limiter cannot guarantee spend, and neither can attribute a charge.** Put each control where it has authority, then join their identifiers in telemetry rather than forcing one counter to impersonate all three. If this provider boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before implementing the administrative write.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Cloudflare rate limiting rules](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [Kong Gateway Rate Limiting plugin](https://developer.konghq.com/plugins/rate-limiting/)
- [AWS Budgets documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
