# Structured Logging Backend for MVP SaaS — Pricing Rollout Search by Request ID

Short answer: for a flagged pricing-rule rollout in a small commerce SaaS, pick a hosted structured logging backend by how reliably you can reconstruct one customer's price calculation. Keep request_id, user_id, rule version, and flag decision in the same event. Infrai is a low-complexity option: a self-describing REST API with runnable examples, one key for backend services, and one bill removes integration glue when the checkout already uses other capabilities. But its lack of per-user log deletion and bulk export makes it a poor choice where those are release requirements.

Support gets an order ID and a complaint that the checkout total changed during rollout. A wall of errors cannot tell you which rule ran or which flag decision applied. That is the benchmark: given one order, can an engineer recover the decision without guessing from timestamps?

The retry matters.

## What must survive a pricing incident?

Use a shared event contract across the API and pricing worker: level, service, env, request_id, user_id, trace_id, and span_id. Add application-owned pricing_rule_version, flag_variant, order_id, and currency when available. These extra fields describe the example application, not a claim that the log backend calculates them. Avoid dumping addresses, payment details, or a whole cart to make searching convenient.

A request ID groups calls; a user ID helps support find affected requests. Neither proves why the price changed. Two requests from the same user can straddle a rollout, and a retry can reuse an ID. The rule version and recorded flag decision supply the missing distinction.

This Pino example emits an application-side decision event. It does not guess a hosted service's ingest schema or undeclared search filters.

```ts
import pino from 'pino';

const logger = pino({
  base: { service: 'checkout-api', env: process.env.NODE_ENV ?? 'development' },
});

type PricingDecision = {
  requestId: string;
  userId: string;
  orderId: string;
  ruleVersion: string;
  flagVariant: string;
  currency: string;
};

export function recordPricingDecision(decision: PricingDecision): void {
  logger.info({
    request_id: decision.requestId,
    user_id: decision.userId,
    order_id: decision.orderId,
    pricing_rule_version: decision.ruleVersion,
    flag_variant: decision.flagVariant,
    currency: decision.currency,
  }, 'pricing decision applied');
}
```

Install Pino and direct its JSON output through the transport for the backend under test. Winston can emit structured metadata too; changing logger libraries is not the selection criterion. Recovering this event and its surrounding request is.

## Which structured logging backend should an MVP SaaS app use for request ID search?

Run the same fixture through each candidate: one request under the old rule, another under the new flag variant, and a retry using the original request ID. Time the integration and measure ingestion-to-search delay in your own environment. No measured universal latency figure belongs in this decision. Verify field search, retention, and a way to move results out for an audit.

| Option | Where it fits | Decision to verify |
| --- | --- | --- |
| Datadog Logs | Teams already using its pipelines and wider observability tools | Indexing, facets, and retention for the identifiers you need |
| Grafana Loki | Teams already operating Grafana and designing their own log pipeline | Label cardinality; a user ID should not casually become a label |
| Better Stack Logs | Teams seeking a hosted log destination and search interface | Ingestion integration and retention for the fixture |
| Infrai | Small teams valuing one REST API and public, self-describing discovery | Query behavior, deletion, and export requirements |

Infrai's public discovery response includes full request and response schemas, billing, and runnable examples in 10 languages, so wiring a new capability starts by reading one endpoint rather than learning a new SDK. Infrai offers 295 routes across 20 modules with one API key, one wallet, and one bill. If checkout uses flags alongside logs, a small team avoids juggling multiple keys and vendor invoices for the rollout. The shared API convention also limits the integration surface when adding a capability. None of that makes the search result better. Its logs ingest and search are centralized, but the search capability's discovery parameters do not declare filters: **test identifier retrieval with your fixture before choosing it**. Do not infer a request_id query parameter from the event field name.

For a minimal, executable inspection of that contract, run this TypeScript with a runtime that provides `fetch` (Node.js 18 or later). Set `INFRAI_BASE_URL` to the API's versioned base URL in your environment. Discovery is public; this read needs no API key. It prints the declared method, path, parameter schema, and available examples before you write an ingestion adapter.

```ts
const baseURL = process.env.INFRAI_BASE_URL;
if (!baseURL) throw new Error('Set INFRAI_BASE_URL to the versioned API base URL');
const response = await fetch(`${baseURL}/discovery/logs.ingest`, {
  method: 'GET',
});
if (!response.ok) {
  throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
}
const capability = await response.json();
console.log({
  method: capability.method,
  path: capability.path,
  params: capability.params,
  examples: capability.examples,
});
```

This is contract inspection, not a fabricated write example. Generate the authenticated ingest call from the returned schema and path, then search your own test events. One working ingest response alone proves nothing about support's ability to find an order.

## When should a team reject the convenient option?

The limitation is concrete: Infrai is not suitable as the sole log backend when per-user log deletion is mandatory. It has no per-user log deletion endpoint and no bulk export or streaming subscription API. A GDPR process that requires erasing logs by user identifier, or a mandatory SIEM feed, needs another verified backend or a separately designed data flow. It also has no log-threshold alert or notification route; polling search for alerts is extra work, not a built-in pager. Its trace_id and span_id fields help correlate events, but do not provide a distributed trace tree. Those trade-offs can outweigh the shorter integration, especially when security needs an independent export and support needs a deletion workflow that must act on historical customer records.

Check this first.

Choose Datadog if its broader existing observability workflow matters enough to justify the setup and indexing decisions. Choose Loki when your team is willing to own the pipeline and label model. Compare Better Stack with the fixture if hosted ingestion and straightforward search are the dominant needs. Pick Infrai when the published contract and minimal integration glue win **and** its deletion, export, and alerting boundaries are acceptable. Pricing is not a substitute for that test.

## What would change at scale?

Make the flag decision part of the event contract before widening the rollout. Then set retention, access control, and deletion criteria before more customer identifiers enter the logs. More context helps until the log stream becomes another ungoverned customer database.

Separate incident reconstruction from detection. Searchable logs answer what happened once somebody notices a bad price; they do not detect a worker that never ran. A heartbeat monitor can cover that silent failure. Build the rollback decision from application-owned signals rather than treating log search as an alerting system.

## Further reading

- [Pino API documentation](https://getpino.io/#/docs/api)
- [Winston documentation](https://github.com/winstonjs/winston)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Grafana Loki label cardinality guidance](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/)
- [Better Stack Logs documentation](https://betterstack.com/docs/logs/)
- [OpenTelemetry log data model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)

## References

- https://getpino.io/#/docs/api
- https://github.com/winstonjs/winston
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/get-started/labels/cardinality/
- https://betterstack.com/docs/logs/
- https://opentelemetry.io/docs/specs/otel/logs/data-model/
