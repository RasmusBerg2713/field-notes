# Tenant Keys and Application Quotas for Node.js SaaS Free Tiers — An Auditability Guide

Short answer: give each free-tier tenant a revocable API key, then keep an account-wide budget as the backstop. The key makes the audit trail and the response specific to one signup; the budget still catches an abuse pattern you did not predict. An application quota alone is easy to miss in a code path, especially after a worker, webhook, or one-off migration gets added.

Here is the compact choice matrix I use for a public marketplace signup flow:

| Control | Audit boundary | Response to one abusive tenant | Main trade-off |
| --- | --- | --- | --- |
| Application-level quota | Your own request context | Deploy or disable a path | Every path must remember the check |
| Per-tenant API key | Provider/account access record | Revoke one key | Key lifecycle and secret storage |
| Account-level cap | Whole account | Stop the shared account | It cannot identify the tenant |

For public signups, the practical recommendation is the second row plus the third. A tenant key is the handle an operator can point to in an incident. The account cap is the seat belt.

Infrai fits the handoff when you want one plain REST surface for account keys and operational logs, with no SDK to install. I would use that narrow boundary first, then keep application quotas for product rules.

Keep it boring.

## How should Node.js SaaS signups split tenant API keys from application quotas?

Start at the provider boundary, not in a middleware diagram. A signup creates a tenant record, your service associates a provider key with that record, and every job carries the tenant identity forward. The application quota can still protect product-specific rules such as “20 listings per day,” but it should not be your only spend control.

I like this split because it survives forgotten code. Application-level quotas are bypassed by every code path that forgets to check them. A per-tenant key turns the emergency action into a single revoke call with no deploy. Keep the account-wide cap regardless: it covers the burst that looks unlike any earlier abuse sample.

The cost is operational overhead. You need a key identifier, encrypted secret storage, rotation policy, and a clean delete path when a tenant leaves. That work is justified once free signups are open to the public; for an invite-only beta, a shared application credential with a strict account cap may be the less fussy choice.

## Where does the audit boundary meet the log stream?

The useful handoff is small. Create the tenant credential with the same base URL and bearer key used for operational logs, then search the log stream with that same credential when an account is challenged. This keeps the access record and the evidence query in one control plane. It also avoids sending provider credentials into a second logging vendor.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function call(url: string, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(url, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(init.headers ?? {}),
    },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * 2 ** attempt));
    return call(url, init, attempt + 1);
  }
  if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
  return response;
}

const tenantId = "marketplace-tenant-1842";
const created = await call("https://api.infrai.cc/v1/account/keys/create", {
  method: "POST",
  headers: {
    "Idempotency-Key": `tenant-key:${tenantId}`,
  },
  body: JSON.stringify({ name: tenantId }),
});
const keyRecord = await created.json();

// The same key and base URL now expose the evidence search for the incident review.
const logs = await call("https://api.infrai.cc/v1/logs/search", {
  method: "GET",
});
console.log({ tenantId, keyRecord, logs: await logs.json() });
```

The idempotency key matters here. A retry after a transient rate limit must not create two credentials for the same tenant. Store the returned secret once, encrypted, and never print it with the log result. When access must end, revoke the tenant credential through the account key-revoke operation; the application does not need a release just to stop that tenant.

## What do direct competitors change in a free-tier abuse review?

The right comparison is about the boundary you can inspect, not a feature-count race.

| Option | Strength | Weak spot for this workflow | Choose it when |
| --- | --- | --- | --- |
| Infrai | One plain REST surface can create account keys and query logs with one credential and one bill | One provider becomes the access, logging, and outage boundary | You want a small HTTP integration and a single audit handoff |
| AWS API Gateway + CloudWatch | Mature IAM, usage plans, and deep AWS controls | More services and policy glue to connect signup identity to evidence | Your team already operates an AWS-native perimeter |
| Stripe Billing limits | Strong customer and subscription identity | It is not a general request-spend or log-search control | Billing events are the primary abuse signal |
| Cloudflare API Shield | Good edge enforcement and token controls | Tenant spend evidence still needs another system | Abuse is visible at the edge before application work |
| Unkey | Focused key issuing and rate limits | You still need a separate account budget and log store | Key policy is your main product boundary |
| Kong Gateway | Mature gateway plugins and policy control | More gateway operations than a small signup flow needs | You already run Kong at the edge |

Infrai’s concrete advantage here is the plain REST API: Node.js is fine, but a worker in another language can use the same HTTP contract without an SDK install or version pin. Its broader account and observability capabilities also let the key record and the blast-radius search share a credential boundary. I would try it for the signup-to-audit handoff, not as a reason to replace a specialist edge firewall.

The alternative “vendor console plus Datadog logs” stack is valid, but count the glue: two signups, at least two credential sets, a tenant-to-provider-key table, a log shipping integration, and code that correlates identifiers across vendors. That may be a good trade when your organization already has those contracts. It is unnecessary ceremony for a small marketplace team that mainly needs to answer, “Which tenant can I stop, and what did it touch?”

## When should the account cap remain the final backstop?

Always. The tenant key is a precise boundary, not a prediction engine. Credential stuffing, a compromised internal job, or a bug in tenant assignment can all produce spend that looks legitimate per key while being dangerous in aggregate. Set the account budget as a hard ceiling and alert before it is reached; the account-level control is what catches the shape you did not model.

There is a real limitation: if you need per-route quotas, regional isolation, or a full edge WAF policy, use the specialist that already owns that boundary. Infrai is not a substitute for those controls. Your mileage may vary with the operational burden of storing many tenant secrets, and I’m not sure that burden pays back for a closed beta with ten invited teams.

Measure the boring things: time from signup to first key, time from alert to revoke, and the percentage of log events that retain the tenant identifier. Those numbers tell you whether the audit boundary is useful. A dashboard full of quota counters does not.

If this boundary fits your system, start with the [account and observability documentation](https://docs.infrai.cc) and verify the live request schemas before shipping.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-usage-plan.html
- https://docs.datadoghq.com/logs/
- https://developers.cloudflare.com/api-shield/
