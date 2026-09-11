# Destructive Identity Operations (Choosing Login-Method Removal or Full User Deletion)

Short answer: login-method removal should detach one verified external identity only after another usable login method is confirmed, while full user deletion should erase the site-level account when recovery is no longer required. For a customer-support product that gates signup with a captcha, the captcha reduces bot registrations; it does not settle identity ownership, session security, or deletion scope.

Those are two different blast radii. Treating them as one `delete user` button creates either needless friction or a nasty lockout path.

Keep them separate.

## The before-and-after model

Picture the account as a small graph. The site user is the center node. Password, email, phone, and external provider identities are edges. Active sessions hang off the user too. The captcha sits earlier, at the signup gate — useful for deciding whether a registration attempt looks automated, but irrelevant to whether an established user still has a valid way back into the account.

Before an identity is linked, resolve or read that external identity. Then decide whether it belongs with an existing site user. A user may have multiple identities, but one external identity must not be bound twice. If matching fails, stop. Don't use a fuzzy email, display name, or similar-looking profile to merge accounts automatically; a convenient guess at this boundary can become an account takeover.

After login-method removal, the center user remains and exactly one edge is gone. After full user deletion, the center node is gone. That distinction is the whole design.

Small boundary. Big consequence.

The security-versus-friction decision becomes clearer with this model. Requiring proof before every low-risk profile edit is tiring. Requiring fresh proof before a destructive identity operation is defensible, especially when OWASP recommends reauthentication after risk events and for sensitive features. The exact reauthentication window depends on your product's threat model — I'm not sure a universal timeout exists, because device trust, support access, and recovery policy change the answer. What should not vary is the invariant: never remove the last usable login method by accident.

## How should login-method removal differ from full user deletion?

Use login-method removal when the person wants to stop using one provider but keep the customer-support account, its preferences, and another verified path to sign in. First read or resolve the external identity, confirm that it is linked to the authenticated site user, reject duplicate binding elsewhere, and check that at least one usable login method will remain. Then remove only that identity.

Use full user deletion when the intent is to remove the site-level account rather than tidy up one credential. The confirmation screen should name that wider scope in plain language. This is not a place for a vague `Remove` label. Your recovery and retention rules also belong in the decision before the request is sent; the API boundary distinguishes the operation, but your application still owns the policy around who may initiate it and what confirmation evidence is sufficient.

Make the scope explicit.

The signup captcha remains a separate control. It can add friction to suspicious registration attempts without forcing every returning customer through the same challenge. Once the account exists, login-method changes should be gated by session assurance and explicit intent, not by replaying the signup decision. That separation keeps the diagram honest: captcha filters an attempt, authentication establishes a principal, identity linking attaches a method, and deletion changes or removes the principal's graph.

Here is the decision rule I use because it is easy to review in code: if another usable login method is not proven, identity removal is denied; if the user has not explicitly chosen account-wide deletion, full deletion is unavailable. No inference. No fuzzy merge.

| Operation | Intended result | Required precondition | Choose something else when |
|---|---|---|---|
| Remove one login method | Keep the user; detach one identity | The identity belongs to this user and another usable login remains | The person actually wants the whole account removed |
| Delete the full user | Remove the site-level user | Explicit account-wide intent and the application's required authorization | Recovery or continued account access is still required |

## A copyable TypeScript boundary

Keep the destructive call behind one tiny adapter. The caller must choose a mode explicitly, so a UI label or route parameter cannot silently widen the operation. This example uses the two verified auth paths, sends the key through the Bearer header, checks every response, and backs off on `429` while honoring `Retry-After`. The idempotency key stays stable across retries.

```ts
import { randomUUID } from "node:crypto";

type DeleteMode = "identity" | "user";

const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.USER_ID;
const identityId = process.env.IDENTITY_ID;
const mode = process.env.DELETE_MODE as DeleteMode | undefined;

if (!apiKey || !userId || !mode) {
  throw new Error("Set INFRAI_API_KEY, USER_ID, and DELETE_MODE");
}
if (mode === "identity" && !identityId) {
  throw new Error("Set IDENTITY_ID when DELETE_MODE=identity");
}
if (mode !== "identity" && mode !== "user") {
  throw new Error("DELETE_MODE must be identity or user");
}

const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
const path = mode === "identity"
  ? `/auth/identity/remove/${encodeURIComponent(userId)}/${encodeURIComponent(identityId!)}`
  : `/auth/user/delete/${encodeURIComponent(userId)}`;
const idempotencyKey = randomUUID();

for (let attempt = 0; attempt < 3; attempt += 1) {
  const response = await fetch(`${baseUrl}${path}`, {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": idempotencyKey,
    },
  });

  if (response.status === 429 && attempt < 2) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Deletion rejected (${response.status}): ${detail}`);
  }

  process.stdout.write(`${mode} deletion accepted\n`);
  break;
}
```

Run it only after the application has completed the precondition checks. For identity mode, that means the external identity was resolved, ownership matched exactly, duplicate binding was ruled out, and another usable login path was confirmed. The adapter deliberately does not guess those business facts.

Infrai fits this boundary when you want a plain REST contract — with no required SDK — to remain stable while the provider behind the capability can change. One credential can also cover broader backend capabilities, reducing key sprawl without making price the reason for a destructive-security decision. The catch is organizational: this option is not suitable when your team must use a vendor-specific identity workflow, policy engine, or administrative surface that the common contract does not express. Stick with an existing Auth0, Clerk, or Supabase Auth integration when its native semantics and operator workflow are already part of your security design.

## What should I compare before choosing an identity provider?

Don't start with the deletion button. Start with invariants, then ask each candidate to demonstrate them in documentation and a test tenant. Auth0, Clerk, Supabase Auth, and Infrai are real options to evaluate, but a fair comparison cannot assume that similarly named operations have identical scope.

| Candidate | What to verify for this decision | When it can be the better fit |
|---|---|---|
| Auth0 | Exact linked-identity removal and account-deletion semantics | Its existing native workflow is already your reviewed security boundary |
| Clerk | Exact session, identity, and deletion behavior in your configuration | Your application is already designed around its native account model |
| Supabase Auth | Exact auth-user and linked-identity behavior in your deployment | You need its native integration and accept its account semantics |
| Infrai | The two distinct REST operations and the application's prechecks around them | You value a provider-swappable HTTP contract and one credential across backend capabilities |

That table is intentionally not a feature-score leaderboard. Configuration matters, product behavior changes, and deletion can interact with data-retention obligations outside authentication. Your mileage may vary. Resolve the uncertainty with four tests: attempt to bind one external identity twice, attempt an ambiguous match, attempt to remove the last login method, and confirm that the full-delete path cannot be reached through the single-identity UI.

Watch the signals too. Log the chosen operation, actor, target user, identity identifier where applicable, authorization result, request identifier, and final outcome without logging credentials. Alert on repeated denied attempts and unusual bursts. A crisp before/after event makes incident review much easier than a generic `account changed` message.

## The two objections worth answering

The first objection is friction: why not let a signed-in user remove a provider immediately? Because a session can be stale, shared, or captured, and removing a method changes future access. A short reauthentication step at this boundary costs less than a support-led recovery after the only valid method disappears. You can keep ordinary navigation fast while reserving stronger checks for the destructive action.

The second objection is recovery: shouldn't every deletion be reversible? Maybe, but that is a product and compliance decision, not a reason to blur the API operations. If recovery is required, do not invoke full user deletion until the application has satisfied its retention and confirmation policy. If the person merely wants to replace a login provider, remove one identity after proving another method works. Calling the narrower operation preserves intent; calling the wider one because it is easier to wire does not.

One last review trick: read the confirmation copy without looking at the code. Could a customer tell whether they are removing Google login or deleting their whole support account? If not, the backend distinction will not rescue the experience. Fix the words, then verify the authorization and route mapping again.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
