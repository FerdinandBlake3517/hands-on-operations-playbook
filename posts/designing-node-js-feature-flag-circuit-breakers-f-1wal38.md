# Designing Node.js Feature-Flag Circuit Breakers from Error-Rate Metrics

**Use an error-rate guardrail to disable a feature only when the metric isolates that feature, the sample is large enough, and multiple completed windows agree.** Otherwise, pause the release and ask a human. A fast toggle driven by a vague service-wide alarm can turn one incident into two.

Start with the decision, not the query language.

| Control | Pick this when | Do not pick it when |
|---|---|---|
| Automatic feature disable | The new path is separately measurable, reversible, and tested in the off state | Writes are irreversible, telemetry is delayed, or both cohorts fail |
| Rollout pause | Exposure is still increasing and causality is uncertain | The dangerous path remains active for already exposed traffic |
| Human-confirmed disable | Volume is low, signals disagree, or the action has broad impact | The response budget is shorter than a person can reliably meet |

Picture the system as a loop. Completed requests feed counters. Counters become closed time windows. A tiny evaluator classifies those windows. An actuator requests the off state. A read-back confirms that state. Finally, an alert carries the counts, decision, and release identity to an operator. Each arrow is observable. Each box has one job.

## How should Node.js error-rate metrics trigger a feature flag toggle?

The numerator is failed requests that entered the flagged path. The denominator is all completed requests that entered the same path during the same window. Keep both counts next to the ratio. An error rate without its denominator hides whether the decision came from 10 requests or 10,000, and those situations should not have the same operational weight.

No traffic, no verdict.

Define “failed” before deployment. It might mean an unhandled exception, an HTTP response covered by the service's error policy, or an explicit unsuccessful domain outcome. The exact definition varies by service; the important part is that the application counter, dashboard, alert, and rollback evaluator use the same one. Don't mix process crashes, dependency timeouts, user cancellations, and domain rejections into one number unless the service contract genuinely treats them alike.

Then segment the signal. At minimum, compare flagged and unflagged cohorts over aligned windows. If both error rates rise together, the release flag is weak evidence: a shared database, network, or upstream service may be responsible. If only the flagged cohort moves, disabling that path is easier to justify. Labels must stay bounded, too. A flag variant and release identifier can be useful dimensions; a raw user ID or request ID can create an unbounded series set and belongs in logs or traces instead. One window is rarely enough. Evaluate closed windows so late samples don't rewrite a decision in progress, require a policy-defined minimum request count, and demand consecutive bad windows before acting. The values are workload decisions, not universal constants. A high-volume API can gather evidence quickly. A weekly batch job cannot. I'm not sure a generic threshold can ever be defensible without the traffic distribution, telemetry delay, and error budget; those inputs are what resolve the uncertainty.

Error rate is also only one of the four golden signals. Traffic protects the denominator, while latency and saturation provide context around the failure. Keep the rule legible rather than blending every signal into an unexplained score. The operator should be able to answer, in one minute, “Which windows failed, how many eligible requests were in them, and what changed?”

## Pick automatic disable for an isolated, reversible path

Automatic disable fits a narrow contract: the flag fully gates the risky behavior, the off path is known to work, and stopping new entries limits the harm. Test that contract before granting write access. Run the evaluator in report-only mode against canary traffic, compare its proposed decisions with the dashboard, and exercise the disabled path under realistic load. Then enable actuation with the exact same evaluator policy.

Idempotence matters. Replaying the same evaluation must request the same desired state, not flip the current state. After the write, read the flag state back. Command acceptance and observed state are separate facts — alerting “rollback complete” before verification makes the control plane look healthier than the application evidence supports.

Keep a durable decision record containing the closed window boundaries, eligible and failed counts, policy version, release identifier, prior flag state, requested state, and observed state. This is the before/after: before, a specific set of metric windows crossed a named policy; after, the feature was observed off. Crisp. Debuggable.

## Pick a pause or human confirmation when evidence is weak

A rollout pause is the honest choice when exposure is increasing but the feature is not the only change. It caps the sample while preserving evidence for comparison. It does not pretend the previous cohort is healthy. Pair the pause with an alert that shows both cohorts and the four golden signals, then let the service owner choose between disabling the feature, reverting the artifact, or fixing a shared dependency.

Human confirmation is better for sparse traffic, irreversible side effects, overlapping releases, or telemetry that arrives near the end of the response window. Five failures out of ten requests and 5,000 failures out of 10,000 requests have the same ratio; they don't carry the same confidence or blast radius. A person needs the raw counts, window times, flag state, and release identity, not just a red percentage.

Stop there.

The alert should distinguish a release decision from control-loop health. Missing metric windows, an unreadable flag state, or failed alert delivery say that the automation lacks evidence; they do not say the release is healthy or unhealthy. Fail closed on automation: stop changing state, preserve the current evidence, and notify the guardrail owner. Recovery should be a separate action. Turning the feature back on because errors disappeared after it was disabled is circular reasoning.

## Implement the evaluator as a small TypeScript state machine

Keep vendor query syntax and flag SDK details behind adapters. The core below receives completed windows, applies a supplied policy, requests `false`, verifies the result, and emits a structured decision. Thresholds are configuration because the article cannot know a service's traffic shape or error budget.

```ts
type MetricWindow = {
  startMs: number;
  endMs: number;
  eligibleRequests: number;
  failedRequests: number;
};

type GuardrailPolicy = {
  minimumRequests: number;
  maximumErrorRate: number;
  consecutiveBadWindows: number;
};

type FlagStore = {
  read(key: string): Promise<boolean>;
  write(key: string, enabled: boolean, reason: string): Promise<void>;
};

type DecisionSink = {
  publish(decision: {
    flagKey: string;
    action: "disabled" | "already-disabled";
    reason: string;
    evidence: MetricWindow[];
  }): Promise<void>;
};

function errorRate(window: MetricWindow): number | undefined {
  if (window.eligibleRequests === 0) return undefined;
  return window.failedRequests / window.eligibleRequests;
}

function isBad(window: MetricWindow, policy: GuardrailPolicy): boolean {
  const rate = errorRate(window);
  return (
    window.eligibleRequests >= policy.minimumRequests &&
    rate !== undefined &&
    rate > policy.maximumErrorRate
  );
}

async function evaluateAndDisable(
  flagKey: string,
  completedWindows: MetricWindow[],
  policy: GuardrailPolicy,
  flags: FlagStore,
  decisions: DecisionSink,
): Promise<void> {
  const evidence = completedWindows
    .filter((window) => window.endMs <= Date.now())
    .sort((left, right) => left.endMs - right.endMs)
    .slice(-policy.consecutiveBadWindows);

  const shouldDisable =
    evidence.length === policy.consecutiveBadWindows &&
    evidence.every((window) => isBad(window, policy));

  if (!shouldDisable) return;

  const reason = `${evidence.length} completed windows exceeded the error-rate policy`;
  const wasEnabled = await flags.read(flagKey);

  if (wasEnabled) {
    await flags.write(flagKey, false, reason);
  }

  const isEnabled = await flags.read(flagKey);
  if (isEnabled) {
    throw new Error(`flag state was not verified as disabled: ${flagKey}`);
  }

  await decisions.publish({
    flagKey,
    action: wasEnabled ? "disabled" : "already-disabled",
    reason,
    evidence,
  });
}
```

Test the pure boundary first: zero traffic, just below the request minimum, exactly at the error-rate threshold, just above it, too few windows, and windows arriving out of order. Notice that the comparison is strictly greater than the configured maximum. The dashboard and runbook must use that same boundary.

Next, contract-test both adapters. The flag adapter must make repeated writes safe and return the observed state. The decision sink must preserve evidence without dropping alerts that share a flag key. Finally, replay synthetic windows in a non-production environment and confirm three visible stages: report-only decision, requested off state, observed off state. That's the whole loop.

## Limits

The catch is simple: a feature flag can stop future entries into a code path, but it cannot undo database mutations, retract queued work, reverse an incompatible schema change, or repair a dependency shared by both cohorts. Stick with a rollout pause and human review when those conditions apply. Also account for telemetry cost and delay; commercial log plans may separate ingestion from indexing, so a guardrail should prefer bounded counters for decisions and retain detailed events according to an explicit investigation policy.

## References

- Google SRE Book, “Monitoring Distributed Systems” (latency, traffic, errors, and saturation): https://sre.google/sre-book/monitoring-distributed-systems/
- Datadog pricing (log ingestion and indexing pricing model): https://www.datadoghq.com/pricing/
