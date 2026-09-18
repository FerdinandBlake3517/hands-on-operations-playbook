# Marketplace Verification Guide: Choose a Custom-Domain Email API with Event Polling

Short answer: choose the email API that lets the signup service own the verification token while the mail boundary owns domain authentication, suppression checks, idempotent submission, and pull-based delivery evidence. For a US/EU marketplace that cannot receive webhooks, the deciding factor is integration effort across that whole loop, not the elegance of one send call.

| Pick this integration shape | Pick it when | Main engineering cost | Evidence available without webhooks |
|---|---|---|---|
| Direct API plus event polling | The provider exposes stable message IDs and a documented event-list or message-status read path | Poller, cursor state, rate-limit handling, and event normalization | Submission and later provider states tied to one internal attempt |
| Direct API plus mailbox-derived signals | Polling is absent, but test and support mailboxes can expose received messages | Inbound parsing and weaker production coverage | Receipt evidence for controlled addresses, not every recipient |
| Internal mail gateway | More than one application or region needs the same policy and evidence contract | A service to operate, plus adapters behind it | One internal ledger even when provider schemas differ |
| Queue plus asynchronous sender | Signup latency must be isolated from provider latency | Worker operations, retry ownership, and delayed-send UX | Durable application intent and attempt history; provider outcome still needs a read path |

The table exposes the real choice. A short initial integration can create a long operational tail. A few extra lines around a generic boundary often remove provider-specific fields from signup code and make the evidence queryable.

## How should you choose an email API for a custom welcome flow?

Start with a diagram in words: browser requests signup -> application creates a single-use verification token -> outbox records mail intent -> sender checks suppression policy -> provider accepts or rejects a submission -> poller collects later state -> normalized events update an evidence ledger. The verification endpoint consumes the token. Email transport never decides whether the account is verified.

That separation matters. An HTTP success from a send endpoint is evidence of API acceptance, not evidence that a mailbox received the message or that a person opened it. DKIM is narrower too: RFC 6376 defines a domain-level signing mechanism that lets a verifier check a signature against a public key in DNS. It does not promise inbox placement. DMARC builds policy and reporting around aligned identifiers; it still does not turn delivery into a certainty.

For this workflow, retain four distinct timestamps when the interfaces expose them: intent created, submission attempted, provider accepted, and latest observed transport event. Do not collapse those into one `sent_at`. The distinction is small in a schema and huge during support: “we intended to send” and “the remote API accepted it” answer different questions.

Keep recipient data controlled. Logs can use an internal signup ID and message ID instead of printing the address or verification URL. The URL contains a credential. Treating it as an ordinary log field expands who and what can read it.

That URL is a secret.

## Pick direct polling when the read contract is complete

Polling is the cleanest fit when outbound callbacks are prohibited and the API exposes enough state to reconcile a submission. Before choosing it, prove the read side in a sandbox or test domain. You need a stable message identifier from submission, a way to fetch status or list events, deterministic pagination or cursors, documented rate limits, and a retention window long enough for your polling and incident-response schedules. If any one of those is missing, the integration may send mail yet fail the observability requirement.

This is the first concrete trap: teams often prototype only the happy-path POST. The harder work starts after acceptance. A poller must survive duplicate events, late events, pagination, throttling, and a crash between fetching a page and saving its cursor. Exactly-once delivery is not a useful assumption here. Make event application idempotent.

Use a mailbox-derived signal only for bounded checks, such as deployment smoke tests or synthetic signup journeys. It can prove that a controlled mailbox received a particular message. It cannot provide complete production-recipient evidence, and a missing test message does not identify which transport stage failed. This option is a diagnostic probe, not a substitute for a provider read contract. Its limitation is coverage: choose it alone only when controlled-address receipt is the actual requirement, not when support needs evidence for every signup.

## Pick an internal gateway when policy repeats

A gateway earns its keep when several marketplace services need the same custom-domain rules, suppression behavior, regional routing, or event vocabulary. It centralizes the awkward pieces: token-free message input, sender authorization, recipient-policy checks, provider idempotency keys, and normalized observations. Signup remains focused on account state. The trade-off is operational ownership. You now deploy, monitor, and secure another service, so a gateway is a poor fit for one flow and one team; an in-process adapter can preserve the same boundary with less integration effort. Promote it only when policy or integrations actually repeat. The custom-domain review should then be explicit before production traffic. Confirm which exact domain or subdomain appears in the visible From address, which identifier the DKIM signature uses, how DNS verification is reported, and how DMARC alignment will be evaluated. Use a dedicated transactional subdomain if organizational policy calls for separation, but do not imply that the subdomain alone creates reputation or delivery guarantees. Authentication configuration and delivery outcomes are related, not interchangeable. Suppression also belongs in the contract: define which events make an address ineligible, how that decision is checked before a new attempt, who may reverse it, and how the change is audited. A provider-managed list can enforce policy close to submission, while an application-side mirror can expose the decision across providers. Mirroring adds synchronization work. Choose one authority and document its lag rather than letting two lists quietly disagree.

Do not add the gateway early.

## A minimal pull-based evidence loop

The example below keeps provider details behind a TypeScript interface. It records intent before the external call, uses the internal attempt ID as the idempotency key, and applies polled events by their stable event IDs. The shapes are illustrative internal contracts; map them only to fields the selected API documents.

```ts
type MailIntent = {
  attemptId: string;
  signupId: string;
  recipient: string;
  verificationUrl: string;
};

type TransportEvent = {
  eventId: string;
  messageId: string;
  kind: "accepted" | "delivered" | "deferred" | "bounced";
  occurredAt: string;
};

interface MailTransport {
  isSuppressed(recipient: string): Promise<boolean>;
  submit(input: {
    from: string;
    to: string;
    subject: string;
    html: string;
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
  listEvents(cursor?: string): Promise<{
    events: TransportEvent[];
    nextCursor?: string;
  }>;
}

interface EvidenceStore {
  createIntent(intent: MailIntent): Promise<void>;
  markSuppressed(attemptId: string): Promise<void>;
  markAccepted(attemptId: string, messageId: string): Promise<void>;
  applyOnce(event: TransportEvent): Promise<void>;
  loadCursor(): Promise<string | undefined>;
  saveCursor(cursor: string): Promise<void>;
}

async function sendVerification(
  transport: MailTransport,
  evidence: EvidenceStore,
  intent: MailIntent,
): Promise<void> {
  await evidence.createIntent(intent);

  if (await transport.isSuppressed(intent.recipient)) {
    await evidence.markSuppressed(intent.attemptId);
    return;
  }

  const result = await transport.submit({
    from: "accounts@notify.marketplace.example",
    to: intent.recipient,
    subject: "Verify your marketplace account",
    html: `<p><a href="${intent.verificationUrl}">Verify account</a></p>`,
    idempotencyKey: intent.attemptId,
  });

  await evidence.markAccepted(intent.attemptId, result.messageId);
}

async function pollEvidence(
  transport: MailTransport,
  evidence: EvidenceStore,
): Promise<void> {
  const page = await transport.listEvents(await evidence.loadCursor());

  for (const event of page.events) {
    await evidence.applyOnce(event);
  }

  if (page.nextCursor) {
    await evidence.saveCursor(page.nextCursor);
  }
}
```

There is one deliberate simplification: rendering inserts a URL directly into HTML. Production rendering must HTML-escape dynamic values and construct the verification URL from trusted configuration. The token should be single-use, expire according to the account-security policy, and be invalidated on successful verification. Those controls belong to the application even if the message body is rendered elsewhere.

Deployment needs two probes. First, a configuration check should fail before traffic if the sending identity is unverified or required credentials are absent. Second, a synthetic signup should periodically exercise intent creation, submission, polling, and receipt at a controlled mailbox. Alert on stages, not a vague “email failed” counter: growing unsubmitted intents indicate worker trouble; accepted messages with no later observations indicate a polling, retention, or transport question.

Track queue age, submission error rate, suppression decisions, poll lag, cursor age, and terminal-event counts. Slice by sending domain and region only when those labels have bounded cardinality. Never put recipient addresses, tokens, or message IDs into metric labels. IDs belong in access-controlled logs or traces.

## Limits and the final decision rule

Pull-based evidence is delayed by design. Its freshness depends on poll frequency, pagination depth, API limits, and event retention. It also cannot create an event the upstream system does not expose. This is a hard limitation, not a tuning problem. If the business requires immediate reaction to a transport event, polling is unsuitable and the no-webhook constraint conflicts with the requirement; record the mismatch instead of disguising it with a tighter schedule. The opposite boundary matters too: callbacks are a poor fit when the environment cannot expose and authenticate an inbound endpoint. In that case, accept measured staleness and choose polling only after verifying that its retention window and rate limits cover the recovery interval.

Choose the smallest option that preserves the evidence your operators need. For one marketplace signup flow, that is usually a direct adapter plus an outbox and idempotent poller when the read contract is complete. Add a gateway when policy repeats across teams or regions. Reject any candidate whose custom-domain authentication, suppression authority, or pull-based event semantics cannot be verified before launch.

## References

- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc8058
- https://resend.com/docs/introduction
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
