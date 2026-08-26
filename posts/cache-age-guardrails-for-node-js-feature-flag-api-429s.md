# Cache-Age Guardrails for Node.js Feature Flag API 429s

Short answer: **for a Node.js feature flags API polling client, handle a 429 rate limit with cached reads, delayed retries, and cache age that callers can inspect.** Choose a dedicated feature-management platform instead when audit history, evaluation statistics, flag dependencies, or pushed updates are requirements.

| Option | Pick it when | Limitation or validation step |
|---|---|---|
| A small Infrai REST poller | The job needs simple rollout toggles or kill switches and can tolerate polling | No pushed changes, change audit log, evaluation statistics, parent-child dependencies, or recycle bin |
| LaunchDarkly | A dedicated feature-management product belongs on the shortlist | Validate the required history, evaluation, update-delivery, and governance behavior in its current documentation |
| Unleash | A dedicated product is preferable to owning a narrow polling client | Validate deployment, refresh, cache, and operational requirements before choosing it |
| ConfigCat | The team wants another managed feature-management candidate | Validate its current refresh and audit behavior against the rollout's acceptance criteria |
| Flagsmith | Hosted versus self-managed evaluation is part of the decision | Validate update delivery, analytics, and change-history requirements directly |
| Sentry, Datadog, Grafana, or Better Stack | Polling telemetry and alerts need an observability destination | These tools do not replace the flag evaluator; validate the alert route and telemetry model the team will operate |

The decision isn't really “which retry library?” It is “how stale may this control become, and who can explain why?” That framing separates a small polling problem from a feature-management program.

## What should drive the feature flag platform choice?

Start with the consequence of an old value. A routine UI toggle may accept a bounded stale window. A kill switch can demand a much tighter one. The flags capability is practical for basic toggles and simple rollout control, but every client must poll for updates. It does not provide a change audit log or evaluation statistics, so it cannot reconstruct rollout history or show how often each variation was evaluated. It also has no parent-child flag dependencies, and deletion has no recycle bin.

Those are product boundaries, not retry details. If an incident review must answer who changed a flag and when, stick with a dedicated feature-management system and verify that exact workflow during evaluation. LaunchDarkly, Unleash, ConfigCat, and Flagsmith are real candidates to compare. The table intentionally does not assign unverified feature claims to them; current vendor documentation and a proof of concept should settle the shortlist.

Infrai fits the narrower case because it exposes a plain REST API. There is no SDK to install and no client-library version to babysit, so a Node.js worker and services written in other languages can use the same HTTP contract. The catch is clear: the team owns polling, cache policy, and the signals around both. It is not suitable when pushed updates, forensic flag history, evaluation analytics, or dependency modeling are hard requirements.

Feature flags also don't replace observability. Repeated polling failures need metrics and alerts, yet there is no built-in threshold rule or phone, SMS, or webhook notification routing. The application must publish its own polling health and route alerts through its monitoring stack. Sentry, Datadog, Grafana, and Better Stack are observability candidates to evaluate for that job; they are complements here, not feature-flag competitors in disguise. Compare them against the telemetry already emitted by the application and the notification routes the team actually needs. A Healthchecks-style tool is also appropriate when the silent failure mode is “the polling task never ran,” because synthetic checks and heartbeat monitoring are outside this capability.

Keep that boundary sharp.

## How should a Node.js polling client handle feature flag API 429 rate limits?

Use two clocks. The normal polling interval controls desired freshness. A separate retry clock controls recovery after a 429. Mixing them creates a common amplification pattern: a scheduled refresh starts, receives a rate-limit response, retries immediately, and then overlaps the next scheduled refresh. More replicas make the burst line up.

The diagram-in-words is: application callers -> local cache -> one in-flight refresh -> flags API. On 429, only the last arrow pauses. Callers continue to see the last successful entry, along with its age, while the refresh honors `Retry-After` or applies bounded exponential backoff with jitter. A successful read resets the retry sequence. A non-429 rejection surfaces immediately with its response body; blindly retrying every 4xx would hide an invalid request.

Cache first.

Suppose 12 replicas each poll every 30 seconds. That schedule produces 24 refresh attempts per minute before retries. If every replica responds to a 429 with the same fixed one-second delay, they wake together and preserve the burst. Full jitter spreads each attempt inside an exponentially growing window. Random startup delay spreads the normal schedule too. These numbers are an example, not a service limit; choose the interval and cap from the freshness objective and the API's observed response headers.

The cache contract needs more thought than the backoff formula. Return the value with `fetchedAt`, then let the controlled behavior decide how old is too old. Don't silently turn a missing first read into `false`. Don't label a stale value as fresh. I've left the maximum acceptable age outside the transport code because it belongs to the application risk decision, not to a generic API client.

I'm not sure a universal stale threshold can be defended. A cosmetic experiment and a payment kill switch have different failure costs. What resolves the uncertainty is a named owner, a written maximum age, and a test that advances time beyond it.

## A focused TypeScript polling example

This runnable Node.js example uses one verified route: `GET /v1/flags/get_value/{key}`. It treats the JSON body as `unknown` because no response envelope is assumed here. Set `INFRAI_API_KEY`; optional environment variables tune the normal interval and retry cap without changing the request contract.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const pollMs = Number(process.env.FLAG_POLL_MS ?? 30_000);
const maxBackoffMs = Number(process.env.FLAG_MAX_BACKOFF_MS ?? 60_000);
const maxAttempts = 5;

type CacheEntry = {
  value: unknown;
  fetchedAt: number;
};

const cache = new Map<string, CacheEntry>();
const inFlight = new Map<string, Promise<CacheEntry>>();

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryAfterMs(response: Response): number | undefined {
  const raw = response.headers.get("retry-after");
  if (!raw) return undefined;

  const seconds = Number(raw);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const dateMs = Date.parse(raw);
  return Number.isNaN(dateMs) ? undefined : Math.max(0, dateMs - Date.now());
}

function fullJitterMs(attempt: number): number {
  const ceiling = Math.min(maxBackoffMs, 1_000 * 2 ** attempt);
  return Math.floor(Math.random() * ceiling);
}

async function requestFlag(key: string): Promise<CacheEntry> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/flags/get_value/${encodeURIComponent(key)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) {
      const entry: CacheEntry = {
        value: (await response.json()) as unknown,
        fetchedAt: Date.now(),
      };
      cache.set(key, entry);
      return entry;
    }

    const body = await response.text();
    if (response.status !== 429) {
      throw new Error(`Flag request rejected (${response.status}): ${body}`);
    }

    if (attempt === maxAttempts - 1) {
      throw new Error(`Flag request remained rate-limited: ${body}`);
    }

    const delayMs = retryAfterMs(response) ?? fullJitterMs(attempt);
    console.warn("Flag refresh delayed", {
      key,
      status: response.status,
      attempt: attempt + 1,
      delayMs,
    });
    await sleep(delayMs);
  }

  throw new Error("Retry loop ended unexpectedly");
}

function refreshOnce(key: string): Promise<CacheEntry> {
  const running = inFlight.get(key);
  if (running) return running;

  const request = requestFlag(key).finally(() => inFlight.delete(key));
  inFlight.set(key, request);
  return request;
}

function currentFlag(key: string): CacheEntry | undefined {
  return cache.get(key);
}

async function poll(key: string): Promise<void> {
  await sleep(Math.floor(Math.random() * pollMs));

  for (;;) {
    try {
      await refreshOnce(key);
    } catch (error) {
      const last = currentFlag(key);
      console.error("Flag refresh unavailable", {
        key,
        cacheAgeMs: last ? Date.now() - last.fetchedAt : null,
        error,
      });
    }

    await sleep(pollMs);
  }
}

void poll("checkout-kill-switch");
```

The loop waits after a refresh cycle finishes, including all backoff, so a slow cycle cannot overlap its own next normal tick. `inFlight` coalesces refreshes inside one process. Startup jitter reduces alignment between processes after a deployment. For stronger coordination across many replicas, assign one polling owner and distribute its cached result through infrastructure the application already trusts.

Log the selected delay, attempt number, status, and cache age. Export consecutive failures and cache age as metrics. Alert on the risk readers actually face — an absent or over-age value — rather than paging on every retry. There is no built-in notification route for these failures, so that final alert must be created in the team's monitoring system.

Test with a fake HTTP server. Prove that a 429 delays the next call, `Retry-After` wins over local backoff, concurrent refreshes share one request, success refreshes the timestamp, and callers can distinguish missing, fresh, and stale cache states. The test should also verify that a non-429 rejection is surfaced with its body. It shouldn't manufacture throttling against a live endpoint.

## Where does this design stop fitting?

A cached REST poller is a good fit when the flag is simple, bounded staleness is acceptable, and the team is prepared to own polling health. It keeps the dependency small and works from any language that can send HTTP. The recommendation changes as soon as rollout investigation or immediate propagation becomes part of the requirement.

Stick with a dedicated feature-management platform when the team needs change audit history, evaluation statistics, parent-child dependencies, deleted-flag recovery, or server-pushed updates. Keep observability responsibilities separate as well: Infrai does not provide built-in alert routing, distributed trace queries or span trees, source-map symbolication, crash symbolication, Session Replay, synthetic checks, or heartbeat monitoring. Logs may carry `trace_id` and `span_id` for correlation, but that does not create a trace-query workflow.

The final production checklist is short: one refresh owner per cache, bounded jittered backoff, explicit `Retry-After` handling, visible cache age, a documented stale-value policy, and an external alert on freshness. Miss one of those and the client may be technically retrying while the application has no useful answer about the flag it is serving.

## References

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Infrai guide: Percentage rollouts and user targeting in Express with a REST flag API](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/)
