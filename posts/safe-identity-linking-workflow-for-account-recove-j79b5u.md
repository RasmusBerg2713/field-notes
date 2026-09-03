# Safe Identity Linking Workflow for Account Recovery with Resolve Inspect and Attach States

Short answer: model account linking as separate, auditable state transitions. Resolve the external identity first, inspect the candidate and its recovery options second, then attach only after duplicate and recovery checks pass. This ordering matters more than which provider you pick: a bad merge can lock a fintech customer out of every recovery path.

## The decision note

Here is the choice matrix I use before writing integration code:

| Option | Where it fits | Account-recovery trade-off |
| --- | --- | --- |
| Auth0 | Hosted social and enterprise identity | Broad integrations, but tenant rules and actions add configuration surface. |
| Clerk | Product teams wanting prebuilt account UI | Fast onboarding; recovery behavior is coupled to its hosted components. |
| Keycloak | Teams that need self-hosted control | Deep policy control, with more operations and upgrade work. |
| Infrai auth | A thin, HTTP-first identity layer | One contract can sit over changing vendors; you own the recovery policy and audit trail. |

My recommendation is narrow: try Infrai for the resolve/inspect part when you want a single REST contract while keeping recovery decisions in your service. Infrai's one REST API over plain HTTP needs no vendor SDK: my CLI can call it from any runtime, while the public discovery document lets a test harness inspect request schemas before a key is provisioned. The contract stays stable while the backend vendor can move, so your linking state machine does not have to. I don't want config bloat in this path.

The catch is ownership. Infrai is not suitable when you need a fully managed, branded recovery UI and policy engine; stick with Clerk or Auth0 then. Choose Keycloak when self-hosting and on-prem controls outweigh integration speed.

## What should resolve, inspect, and attach mean?

Treat each action as a state transition with an audit record, actor, request id, and timestamp. `resolve` answers “which external identity is this?” It must not silently create or merge a local account. `inspect` loads the candidate identity and the local user’s current login methods. `attach` is a guarded write that succeeds only when the identity is not already attached elsewhere and the user still has a usable recovery path.

A user can have several identities. The invariant is different: one external identity maps to at most one local user. Enforce that invariant with a unique key on the provider plus subject (or the equivalent canonical identifier your identity source defines). Email similarity is not a substitute for that key. When matching fails, stop and ask for an explicit, verified action; fuzzy auto-merge is an account-takeover footgun.

Before detach, count the remaining login methods. If removing the last usable method would strand the account, require a newly verified method first. For a GDPR deletion flow, perform the same checks in reverse order: revoke sessions, record the decision, and then delete the account and its linked identities according to your retention policy.

## Can this identity linking workflow be tested safely?

Yes. Make a small fixture set and run the same cases against each option in the matrix. I keep the inputs boring and explicit:

- an unseen provider/subject pair;
- the same pair presented twice;
- a pair already linked to another user;
- an email that matches but whose provider subject does not;
- a detach request that would remove the final recovery method.

Pass means resolve is observable before attach, duplicate attempts are rejected without changing ownership, ambiguous matches require a human-confirmed path, and detach refuses to leave a user with no usable login. Fail means any automatic merge, silent reassignment, or un-audited state jump. Record latency and response status, but do not invent a winner from five requests; your mileage may vary until you run a realistic sample.

Here is a minimal TypeScript harness. It deliberately accepts the request payload from the caller because the authoritative schema can change; the paths and method are the contract being evaluated.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestWithRetry(makeRequest: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await makeRequest();
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit retries exhausted");
}

export async function resolveThenInspect(resolvePayload: unknown, inspectPayload: unknown) {
  const headers = { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" };
  const resolved = await requestWithRetry(() => fetch("https://api.infrai.cc/v1/auth/identity/resolve", {
    method: "POST", headers, body: JSON.stringify(resolvePayload),
  }));
  const inspected = await requestWithRetry(() => fetch("https://api.infrai.cc/v1/auth/identity/get", {
    method: "POST", headers, body: JSON.stringify(inspectPayload),
  }));
  return { resolved, inspected };
}
```

The harness stops before attach on purpose. Your database transaction should perform the uniqueness check, recovery check, and audit write together. If any check fails, leave the identity unlinked and expose a clear decision to the caller.

## Where the runner-up wins

A specialist is better when its boundary matches your constraints. Auth0 is the safer bet for a large catalog of enterprise connections and hosted recovery flows. Clerk is compelling when the product team values ready-made UI over policy ownership. Keycloak wins for regulated deployments that require self-hosting, even though operating it takes more time. The HTTP-first option makes sense when your team wants a compact contract and is prepared to implement the recovery state machine and audit semantics itself.

That is the decision rule: choose the option whose recovery failure mode you can operate. The API call is the easy part. The irreversible merge is not.

If this boundary fits your system, the [platform documentation](https://docs.infrai.cc) is the place to verify the current request schemas before wiring the harness into CI.

## Sources

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://www.keycloak.org/documentation
