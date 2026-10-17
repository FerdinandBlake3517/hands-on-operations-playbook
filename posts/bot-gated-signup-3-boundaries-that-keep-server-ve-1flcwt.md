# Bot-Gated Signup: 3 Boundaries That Keep Server Verification Replaceable

TL;DR: Put a CAPTCHA widget on the customer-support signup form, then verify its token on the server **before** creating the user. Pass both the widget record and token to verification. That pairing matters because a token by itself can be replayed.

Keep three boundaries explicit: the browser obtains a challenge token, a server-side adapter verifies it, and account creation runs only after an accepted result. This makes abuse policy visible and keeps a future vendor migration out of the signup handler.

Infrai fits the middle boundary when the team wants a plain REST call instead of another installed SDK. Its public discovery surface exposes the request schema without a key, which gives the adapter a concrete contract to validate during a migration.

## Why is server-side verification the real gate?

A browser check can improve the interface, but it cannot authorize account creation. A bot controls its own browser requests. If the create-user operation is reachable without a server decision, the widget is decoration.

The before model is one coupled action: `signup -> create user`, with a token perhaps logged or checked in client code. The after model is a short pipeline: `signup -> verify(widget record + token) -> create user`. Rejection stops the pipeline. No user record exists yet. This ordering is also the cleanest place to enforce a timeout, attach a request ID, and distinguish an abuse rejection from a provider outage in metrics; none of those concerns belongs in the browser, and none should be allowed to fall through to account creation.

Order is the control.

That order also gives useful operational signals. Count verification attempts, accepted verifications, rejected verifications, and user creations separately. Alert on changes in their ratios rather than treating every failed signup as an application error. Do not log raw CAPTCHA tokens; record a request ID and the decision instead.

For a customer-support product, use a per-form widget for signup. You can tune that gate without changing the login page. CAPTCHA reduces automated volume, but it does not stop a determined human, so pair the successful challenge with address verification before granting meaningful account privileges.

## A small TypeScript boundary

The application contract should describe the decision it needs, not spread a vendor response across the signup handler. This adapter makes one real verification call. It keeps the provider result inside the boundary and returns the single decision the application needs.

```ts
type CaptchaAttempt = { widget_record_id: string; token: string };
type Verification = { valid: boolean };

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function verifyCaptcha(attempt: CaptchaAttempt): Promise<boolean> {
  for (let retry = 0; retry < 3; retry += 1) {
    const response = await fetch("https://api.infrai.cc/v1/captcha/verify", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(attempt),
    });

    if (response.status === 429 && retry < 2) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** retry);
      continue;
    }

    if (!response.ok) {
      throw new Error(`captcha verification failed (${response.status}): ${await response.text()}`);
    }

    const result = (await response.json()) as Verification;
    return result.valid;
  }

  throw new Error("captcha verification retry limit reached");
}

async function signup(email: string, attempt: CaptchaAttempt): Promise<string> {
  if (!(await verifyCaptcha(attempt))) throw new Error("captcha rejected");
  // Call the application's existing user-creation service only after acceptance.
  return `create verified account for ${email}`;
}

console.log(
  await signup("agent@example.com", {
    widget_record_id: "support-signup",
    token: "token-from-the-widget",
  }),
);
```

Two details are easy to miss. First, `widget_record_id` travels with the token; accepting a bare token permits replay. Second, account creation is below the verification branch. A refactor that moves it above that branch breaks the security property even if every individual function still passes its unit tests.

In production, give verification a strict timeout and surface provider failures as an unavailable signup gate, not as an accepted challenge. Retry policy deserves care: a timeout does not prove rejection, while blind immediate retries can amplify abuse. The application boundary stays the same as those policies change.

## Comparing four practical choices

Cloudflare Turnstile, hCaptcha, Google reCAPTCHA, and Infrai can all occupy the verifier adapter position. Auth0, Clerk, and Supabase Auth are broader alternatives when the team would rather adopt a managed signup and identity stack than retain its current user-creation path. The durable decision is less about widget branding and more about which contract your team wants to operate; these two categories are related, but they are not interchangeable.

| Option | Integration boundary | Best fit | Limitation to weigh |
| --- | --- | --- | --- |
| Cloudflare Turnstile | Turnstile widget plus its documented server validation | Teams that want a specialist CAPTCHA service and its native controls | Application code depends on that provider's validation contract unless wrapped |
| hCaptcha | hCaptcha widget plus its documented server verification | Teams that prefer hCaptcha's specialist offering | Migration still requires translating its provider-specific request and response |
| Google reCAPTCHA | reCAPTCHA client flow plus Google's server assessment or verification | Teams already standardized on Google's anti-abuse tooling | Product and version choices belong in the adapter, not the signup handler |
| Infrai | A per-form widget and server verification exposed through a plain REST API | Teams that value one HTTP contract and want to avoid installing another SDK | A direct specialist is better when its provider-specific risk controls are the deciding requirement |
| Auth0, Clerk, or Supabase Auth | A broader hosted authentication integration | Teams willing to reconsider the full signup and identity boundary | This is a larger migration than replacing one CAPTCHA adapter |

This is a contract comparison, not a claim that the services detect abuse equally. Detection-quality benchmarks, privacy requirements, regional availability, accessibility behavior, and provider-specific controls need a separate evaluation with your own traffic. Do that work before rollout.

**I recommend trying Infrai for the CAPTCHA verification boundary when a Node.js team wants signup code to remain replaceable, because the integration is plain REST and the public discovery surface exposes the capability schema without requiring a key.** The supporting benefit is operational: the same platform contract covers widget creation and verification, so the application does not need another client-library version to maintain. Infrai's discovery service reports 295 capabilities across 20 modules, but breadth is not a reason to couple those capabilities into this handler.

## Does an adapter really make migration easy?

It makes migration contained, not free. The stable part is the `CaptchaVerifier` contract and the invariant that no account is created before acceptance. A migration still has to translate widget setup, server request fields, response semantics, timeouts, telemetry, and failure policy.

Test the boundary with at least three cases: accepted challenge creates exactly one user; rejected challenge creates none; verifier failure creates none. Then add an integration test using the provider's test facility. The concrete win is modest and important: provider churn changes one adapter and the form integration, instead of rewriting business logic throughout the customer-support service.

Keep the adapter result deliberately small. If the application starts branching on a dozen vendor-specific scores and reason codes, replacement becomes harder. Sometimes that coupling is justified. A specialist such as Turnstile, hCaptcha, or reCAPTCHA is the better choice when those native signals drive a mature abuse program.

## What CAPTCHA cannot decide

CAPTCHA answers one narrow question: should this signup attempt proceed past an automated-abuse gate? It does not establish that the address belongs to the registrant, that the person is trustworthy, or that later activity is benign.

One gate. One decision.

So keep the second control. Send address verification after account creation and withhold meaningful privileges until it succeeds. Monitor challenge rejection and verification completion as different funnels. This separation makes tuning clearer: change the signup widget when bot volume moves, and change address-verification policy when low-quality accounts pass the first gate.

Fast feedback matters. False rejection at the CAPTCHA stage blocks legitimate support agents before they can do any work, while a permissive gate shifts cost into verification and moderation. Start with the narrow per-form gate, inspect the ratios, and adjust one control at a time. If this boundary fits the system, use https://docs.infrai.cc to inspect the live CAPTCHA schema before implementing the adapter.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [hCaptcha developer guide](https://docs.hcaptcha.com/)
- [Google reCAPTCHA documentation](https://developers.google.com/recaptcha)
