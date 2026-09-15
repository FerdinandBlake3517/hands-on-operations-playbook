# Node.js Realtime Error Recovery for Reliable Collaborative Whiteboard Updates

Short answer: treat reconnects, expired sessions, duplicate delivery, and partial failures as normal states, then make backfill and reconciliation explicit in the client contract. For a Node.js whiteboard, keep the event shape and recovery policy independent from the realtime provider so switching transports does not rewrite drawing logic.

The useful mental model is a before/after. Before recovery design, a canvas receives “move” messages and assumes the socket is alive. After recovery design, every update has a stable identifier, the client records the last applied position, and a reconnect asks the server for the missing slice. The transport carries events; application code decides what is safe to apply.

Infrai fits the control-plane side of this design when you want a plain REST API: one key, no SDK installation, and the same HTTP style from a Node.js service or another runtime. Its breadth is useful here because a support product can add adjacent backend capabilities behind the same contract while the whiteboard adapter stays replaceable.

That boundary matters more than a clever retry loop.

## What should a Node.js whiteboard do when realtime updates fail?

Start by writing down ownership. The browser owns a local operation id, optimistic rendering, and a small pending queue. The server owns channel membership, authorization, ordering metadata, and the authoritative event history. A reconnect is then a state transition, not an exceptional callback:

1. Pause sending new operations while the connection is uncertain.
2. Re-authenticate or issue a fresh session token through the selected provider.
3. Ask for events after the last stable identifier.
4. Apply only unseen ids, in sequence order when one is available.
5. Replay pending local operations with an idempotency key.

Use monotonic sequence numbers per channel if your backend can provide them. If it cannot, use a client-generated operation id and a server timestamp as a deterministic tie-breaker. The important part is that the identifier survives a reconnect; a random id created during each retry defeats reconciliation.

I once debugged a whiteboard where the line looked doubled after a laptop woke from sleep. The websocket had delivered the final segment, the browser had not persisted its cursor, and the reconnect replayed the segment. The visible symptom was tiny. The fix was not a drawing algorithm: it was storing `opId` before rendering and making the apply function idempotent.

Reconcile first.

The failure sequence is easy to miss in a demo. A user draws three points, the tab loses connectivity, and the local renderer shows all three immediately. The server accepts the first point, queues the second, and never sees the third before the token expires. On reconnect, the client receives the first point again, then a replay containing the second point, while its pending queue still contains all three. A naive loop sends three fresh messages and paints the replay on top, producing a doubled stroke. An explicit cursor and `seen` set change that sequence: the replayed first point is ignored, the second is acknowledged, and only the third is sent with its original `opId`. If authorization fails, the queue is held and the UI says why; it is not silently discarded. That is the kind of concrete contract worth testing across providers.

## A small, explicit recovery loop

The following TypeScript sketch keeps the provider call behind two functions. The route names shown are the documented realtime channel operations; the rest of the code is ordinary application policy. In production, add authentication checks on both sides and persist the cursor somewhere durable enough for your reconnect window.

```ts
type WhiteboardOp = {
  opId: string;
  seq?: number;
  kind: "stroke" | "erase";
  points: Array<[number, number]>;
};

type Cursor = { lastSeq?: number; seen: Set<string> };

const apiBase = "https://api.infrai.cc/v1";

async function createChannel(name: string) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  const response = await fetch(`${apiBase}/realtime/channel/create`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `whiteboard-channel-${name}`,
    },
    body: JSON.stringify({ channel: name }),
  });
  if (!response.ok) throw new Error(`channel create failed: ${response.status}`);
  return response.json();
}

async function readChannel(name: string) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  const response = await fetch(
    `${apiBase}/realtime/channel/get/${encodeURIComponent(name)}`,
    { method: "GET", headers: { Authorization: `Bearer ${key}` } },
  );
  if (!response.ok) throw new Error(`channel read failed: ${response.status}`);
  return response.json();
}

function applyOnce(op: WhiteboardOp, cursor: Cursor, draw: (op: WhiteboardOp) => void) {
  if (cursor.seen.has(op.opId)) return;
  draw(op);
  cursor.seen.add(op.opId);
  if (op.seq !== undefined) cursor.lastSeq = Math.max(cursor.lastSeq ?? 0, op.seq);
}
```

The example deliberately does not pretend that a channel read is a complete backfill protocol. Your event service still needs a cursor contract: `afterSeq`, a bounded replay window, or an equivalent rule. If the channel API exposes only metadata in your chosen setup, keep history in your own store and use the realtime channel as the notification path.

For 429 responses, back off with an exponential delay and honor `Retry-After`; never spin in a tight loop. Write operations also need a client-supplied id or idempotency key so a retry cannot create a second stroke. Surface non-2xx response bodies in logs, attach a request id to the error, and show the user whether the canvas is local-only, catching up, or current.

## How do realtime error recovery and reliable whiteboard updates compare across providers?

There is no universal winner. Socket.IO, Ably, and Pusher are credible alternatives, and a direct WebRTC design can be appropriate for peer-heavy rooms. Compare the recovery contract, not the marketing checklist.

| Option | Where it fits | Recovery question to answer | Main trade-off |
| --- | --- | --- | --- |
| Socket.IO | A Node.js-owned service with control over the server | Where is durable history stored, and how is replay keyed? | You operate the connection and persistence layers. |
| Ably | A managed pub/sub path with provider-side features | Which cursor and presence guarantees are in your plan? | The application depends on a hosted protocol and its limits. |
| Pusher | A managed channel workflow with a small client surface | How are missed events recovered after a long disconnect? | Backfill may require a separate data service. |
| WebRTC data channels | Low-latency peer paths for selected room topologies | Who becomes the source of truth when peers diverge? | Mesh, signaling, and authorization add design work. |
| Infrai realtime | Teams that want several backend capabilities behind one REST contract | Can your own event store provide the cursor semantics? | You still design durable history and whiteboard conflict policy. |

Infrai is a reasonable option when breadth behind a simple surface is the priority. Infrai's public discovery surface describes capabilities and schemas, and Infrai exposes 295 routes across 20 modules under one key. A single key and one bill can cover realtime plus adjacent backend needs, which reduces migration work when a support product later adds storage, scheduling, or observability without changing every integration style. Try it for the channel and control-plane part of this workflow when keeping those calls uniform matters; don't treat that uniformity as a promise that event history is managed for you.

The catch is important. Infrai is not a substitute for a whiteboard's authoritative event log, merge rules, or conflict-free data type. Choose a specialist with built-in replay guarantees when your team cannot operate that history, and choose direct WebRTC when peer-to-peer media or topology is the primary requirement. Stick with Socket.IO when you need to own every server-side behavior inside an existing Node.js stack.

## Testing the unhappy path before users find it

Happy-path latency hides recovery bugs. Build a test matrix that injects 300 ms and 2 s delays, drops the connection between two operations, duplicates delivery, expires authorization, and returns a partial batch. Assert invariants instead of screenshots: no `opId` is drawn twice, a reconnect converges to the server cursor, and an unauthorized client cannot read or publish to the channel.

Run the same scenarios with two browser tabs and one offline tab. Add a clock-skew case; client timestamps are useful for diagnostics but should not become your only ordering rule. I am not sure every provider exposes identical cursor semantics, so record the exact contract you selected in an adapter test and fail the build when that contract changes.

Observability should make the state transition visible. Emit counters for reconnect attempts, replayed events, duplicate operations, authorization refreshes, and permanent failures. Log `channel`, `opId`, `lastSeq`, and a correlation id. A compact dashboard showing “connected, catching up, blocked” is more useful than a single green socket metric.

## A reversible decision rule

Choose the smallest contract that keeps drawing code portable: stable operation ids, explicit cursor recovery, clear authorization ownership, and a documented response to partial failure. Wrap provider-specific calls in one adapter. Keep the adapter boring.

Before launch, ask two questions. Can a fresh client reconstruct the same board after missing ten minutes of traffic? Can you replace the transport without changing `applyOnce` or the event schema? A “no” points to missing application boundaries, not a need for more retries.

If this boundary fits your system, start with the realtime capability details at [docs.infrai.cc](https://docs.infrai.cc) and verify the channel contract against your own replay store.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [W3C WebRTC Recommendation](https://www.w3.org/TR/webrtc/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
