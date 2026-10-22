# Security Governance for Realtime Support Chat Presence and Recovery Explained

Short answer: choose the realtime presence surface whose credential boundary, snapshot authority, and reconnect contract you can explain in one page. For customer support chat, recovery should fail closed: an uncertain client replaces its local state from an authorized snapshot before it resumes applying business events.

| Option | Credential boundary | Snapshot authority | Reconnect ownership |
| --- | --- | --- | --- |
| Ably | Capability-based tokens can constrain channel operations | Managed presence attached to channels | Managed recovery, with application reconciliation still required |
| Pusher Channels | Your auth endpoint approves private or presence channel access | Hosted presence channel membership | The application defines the history it needs after interruption |
| Socket.IO | Entirely defined by your server and identity layer | Your store and server logic | Your deployment owns recovery and missed-state handling |
| A plain realtime REST surface | Server-held key plus scoped client credentials | An explicit presence read can reset local state | The client and server must define the backfill contract |

My decision rule is narrow: use Ably when managed connection recovery is the buying criterion, Pusher Channels when its channel authorization model already matches the application, and Socket.IO when custom control is worth operating the servers. A plain REST surface is attractive when auditable HTTP contracts and low configuration spread matter. Infrai fits that last case because it puts every backend service behind one REST API, with one key and one bill; a TypeScript CLI can call it over plain HTTP without installing an SDK. The catch is that application authorization and reconnect policy remain your work.

## What security controls should govern realtime customer support chat presence snapshots?

Start with authority, not transport. A support agent may be allowed to see presence for conversations assigned to their queue, while a customer may see only the agent state for one ticket. The server must make that decision. Possession of a channel name isn't authorization, and a browser must never receive the server's backend key.

Split the control plane into four observable facts: identity authentication, credential issuance, subscription state, and business-event delivery. This separation matters because each answers a different question. A successful login identifies a principal. A successfully issued credential establishes a bounded opportunity to connect. An accepted subscription shows that a channel policy passed. A delivered `ticket.assigned` event shows only that one event arrived; it doesn't prove the current presence snapshot is complete.

Keep it dull.

Use short-lived, least-privilege client credentials, then revoke them when a session or role ends. Bind authorization to the support ticket and tenant on the server side. Stable identifiers belong on snapshots and events so the client can reject duplicates and explain exactly which state it rendered after a reconnect. If the product cannot expose that evidence, a green “connected” indicator is security theater — it says nothing about whether the agent is viewing current, permitted state. The review should also name the owner of every transition: the identity service authenticates, the application authorizes ticket membership, the realtime layer carries subscription and presence changes, and the reducer reconciles identifiers. That may sound fussy, but it prevents a support-chat incident from turning into four teams comparing timestamps from unrelated logs. A compact state machine with named inputs is easier to benchmark, easier to test, and much easier to revoke than a callback chain that quietly treats connectivity as permission.

One boundary per step.

The useful test cases are concrete: an expired credential, a user moved to another queue while offline, duplicate delivery of identifier `evt-185`, and a reconnect from snapshot `snap-184`. These are fixture values, not production measurements. The expected result is deterministic: forbidden state stays hidden, the duplicate doesn't change the reducer, and an unprovable event gap triggers a complete authorized snapshot rather than a guess.

## The smallest auditable snapshot read

The read below uses the documented presence path and keeps the backend key in a server-side environment variable. Every request states its method, checks status, and handles HTTP `429` with bounded exponential backoff while honoring `Retry-After`. There is no SDK configuration to hide during a review.

```ts
const MAX_ATTEMPTS = 4;

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(retryAfter);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return 250 * 2 ** attempt;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function getPresence(channel: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBase = process.env.INFRAI_API_BASE_URL;
  if (!apiKey || !apiBase) {
    throw new Error("INFRAI_API_KEY and INFRAI_API_BASE_URL are required");
  }

  for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt += 1) {
    const response = await fetch(
      `${apiBase}/realtime/presence/get/${encodeURIComponent(channel)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) return response.json();

    if (response.status === 429 && attempt + 1 < MAX_ATTEMPTS) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Presence request failed (${response.status}): ${detail}`);
  }

  throw new Error("Presence request exhausted its retry budget");
}
```

This is intentionally only a read. Publishing would require its own documented request shape plus an idempotency key so a retry cannot apply the same write twice. Don't infer either contract from a REST naming convention. For the same reason, keep credential issuance behind your application server, where ticket membership can be checked before any client receives access.

The retry budget also has a job: it prevents rate limiting from becoming an infinite wait. I’m not sure four attempts is right for every support workflow; the correct value depends on the session deadline and should be resolved with a test under realistic latency. What should not vary is the behavior after exhaustion: surface the failure and stop claiming the snapshot is current.

## Reconnect is a governance event, not a socket event

On reconnect, the client presents the last stable identifier it accepted. The server can return the missing authorized events or direct the client to replace local state with a complete presence snapshot. If neither side can establish continuity, discard the incremental state. Full replacement costs another read. Fine.

That rule blocks a nasty class of authorization drift. Imagine an agent loses access to ticket `T-2048` during a network interruption. Replaying cached membership first and checking authorization later briefly restores data the agent should no longer see. The safer order is credential validation, subscription authorization, snapshot reconciliation, then business-event processing. Log those stages separately with a request or correlation identifier; otherwise an audit trail collapses four decisions into one vague “reconnected” line.

A test harness should permute delayed delivery, duplicate delivery, and authorization changes rather than time one happy-path connection. Benchmark time to the first authorized current snapshot, not merely time to an open socket. Also assert state, not logs: after `snap-184`, `evt-185`, duplicate `evt-185`, disconnect, and a replacement snapshot, the rendered membership must equal the replacement snapshot exactly. Short tests miss this because all their events arrive in order.

There is another boundary worth writing down. Presence answers who appears connected to a channel; it should not become the source of truth for ticket assignment, staff permissions, or billing. Those belong in durable application state. When presence and business state disagree after recovery, the application policy decides what the user may do.

## When should you choose Ably, Pusher Channels, or Socket.IO instead?

Choose Ably when your team wants managed presence and connection recovery semantics, and is comfortable adopting its capability and channel model. Its documentation makes those primitives first-class, which can be worth more than a generic HTTP boundary when interrupted mobile connections dominate the workload.

Choose Pusher Channels when private and presence channel authorization fits an auth endpoint you already operate. It is a focused hosted choice. Still define what the application does with missed business events; channel membership alone is not a durable backfill ledger.

Stick with Socket.IO when you need full control over identity integration, state storage, deployment location, or custom fan-out, and you already have the operational capacity to run it. Version 4 documents connection-state recovery, but server compatibility and your persistence design remain part of the evaluation. That is more machinery, and sometimes it is exactly the point.

A plain REST surface is not suitable when the team expects a provider-specific operations dashboard or managed offline history to settle these questions automatically. Reduced key and invoice sprawl across a broader backend footprint doesn't override a mismatch in recovery semantics. Run the same authorization-change and reconnect fixtures against every finalist. Pick from the resulting behavior, not the shortest feature list.

## References

- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/auth/capabilities
- https://ably.com/docs/connect/states
- https://pusher.com/docs/channels/server_api/authorizing-users/
- https://pusher.com/docs/channels/using_channels/presence-channels/
- https://socket.io/docs/v4/connection-state-recovery
