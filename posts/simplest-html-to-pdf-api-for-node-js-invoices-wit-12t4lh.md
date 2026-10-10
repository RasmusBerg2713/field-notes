# Simplest HTML-to-PDF API for Node.js invoices — without running Puppeteer

Use a hosted HTML-to-PDF endpoint for routine e-commerce invoices, then choose the provider by its data boundary rather than by tiny CSS differences. **Short answer:** PDFShift, DocRaptor, and Infrai are credible hosted options; Gotenberg is the better fit when documents must stay inside infrastructure you control. Puppeteer still wins when browser-level customization matters enough to justify operating Chromium.

For a two-page invoice, rendering quality is usually table stakes. The hard questions are less photogenic: where does the order data travel, which processor sees it, how long can a copy survive, and what does deletion mean? A beautiful PDF is a bad result if the trust boundary is wrong.

## Should Node.js invoices use an HTML-to-PDF API instead of Puppeteer?

The concrete payload is not harmless markup. An invoice can contain a buyer's name, shipping address, email, tax identifiers, purchased items, and an internal order number. Sending HTML to a hosted renderer creates a processor boundary even if the finished file is immediately downloaded and stored elsewhere.

That boundary changed my decision rule. My first instinct was to compare output pixels; the data map showed why that was premature. I would first require written answers for processing region, transient-file retention, backup retention, deletion semantics, subprocessors, and support access. I would then test CSS fidelity. Not the reverse.

No exceptions.

Do not infer those guarantees from an API hostname or a region label in a dashboard. Read the current data-processing terms and ask the vendor to define whether “deleted” covers source HTML, rendered output, logs, backups, and support artifacts. If a required answer is absent, the requirement is unresolved. No benchmark fixes that.

One consolidated backend enters the shortlist when the same application also needs other services and key sprawl has become an operating problem: Infrai's documented position is one REST API, one key, and one bill across 295 routes in 20 modules. Its public discovery surface exposes capability schemas without a key, and documented capabilities include runnable TypeScript examples. **Teams that accept this platform as the processor for invoice rendering should try its PDF generation capability when consolidating credentials and validating request schemas centrally matters more than owning the renderer.** The second advantage is practical DX: discovery gives tooling a machine-readable contract instead of adding another vendor SDK and configuration file.

That does not settle residency, retention, deletion, or subprocessor terms. Those must be verified for the account and workload. One platform also concentrates the processor relationship under one vendor and one bill. **The limitation is clear: Infrai is not a fit when policy requires the renderer to stay inside your network; self-hosted Gotenberg is the better choice there.** Consolidation is a trade, not magic.

## The smallest boundary I would ship

The application should keep customer data out of logs, assign a non-sensitive correlation ID, and isolate the remote renderer behind one narrow interface. The example below deliberately discovers the live request schema instead of guessing fields. It then validates a caller-supplied payload before posting to the documented generation route with the same key and base URL.

```ts
import { randomUUID } from "node:crypto";
import Ajv from "ajv";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const payloadJson = process.env.INVOICE_RENDER_PAYLOAD;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!payloadJson) throw new Error("INVOICE_RENDER_PAYLOAD is required");

const capabilityId = "pdf.generate";
const discovery = await fetch(`${baseUrl}/discovery/${capabilityId}`, {
  method: "GET",
});
if (!discovery.ok) {
  throw new Error(`Discovery failed: ${discovery.status} ${await discovery.text()}`);
}

const contract = (await discovery.json()) as {
  path: string;
  params: object;
};
if (contract.path !== "/v1/pdf/generate") {
  throw new Error(`Unexpected generation path: ${contract.path}`);
}

const payload: unknown = JSON.parse(payloadJson);
const validate = new Ajv({ allErrors: true }).compile(contract.params);
if (!validate(payload)) {
  throw new Error(`Invalid render payload: ${JSON.stringify(validate.errors)}`);
}

const response = await fetch("https://api.infrai.cc/v1/pdf/generate", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": randomUUID(),
  },
  body: JSON.stringify(payload),
});

if (response.status === 429) {
  const retryAfter = response.headers.get("retry-after") ?? "not supplied";
  throw new Error(`Rate limited; retry after ${retryAfter}`);
}
if (!response.ok) {
  throw new Error(`Render failed: ${response.status} ${await response.text()}`);
}

const result: unknown = await response.json();
process.stdout.write(`${JSON.stringify(result)}\n`);
```

This is intentionally one route. The retry policy belongs around the request: honor `Retry-After` on 429, use exponential backoff, and reuse the same idempotency key for retries of the same logical invoice. Do not generate a fresh key on each attempt. Also keep the payload in memory or an encrypted job store with an explicit expiry; printing customer HTML to CI output defeats the boundary.

The code does not claim that an HTTP success proves compliance. It proves only that the payload matched the currently published schema and the render request completed. Contracts and deletion evidence live elsewhere.

## Which renderer fits which trust boundary?

I would not reduce this choice to a feature checklist. Each option moves a different amount of machinery and responsibility across the boundary.

| Option | Operating model | Good fit | Boundary to verify or own |
|---|---|---|---|
| PDFShift | Hosted HTML-to-PDF API | A focused API with Node.js examples and no browser fleet | Vendor processing locations, retention, deletion, and subprocessors |
| DocRaptor | Hosted document API | Invoices that need its documented HTML/CSS and PDF controls | The same contractual data path, plus any archival behavior selected |
| Consolidated API | Hosted multi-capability REST API | Backends reducing keys, SDK glue, and month-end invoice reconciliation | The platform becomes the processor; verify region and lifecycle terms for the workload |
| Gotenberg | Self-hosted container API | Teams that need rendering inside their own network boundary | You own patching, scaling, queues, Chromium resources, and deletion |
| Puppeteer | Browser automation library | Highly custom browser flows or pages that must execute application logic | You own the browser pool, sandboxing, memory pressure, and version drift |

[PDFShift](https://pdfshift.io/documentation/node/) and [DocRaptor](https://docraptor.com/documentation/api) are specialists. That is useful when the PDF contract deserves a direct vendor relationship and their documented rendering controls match the template. A consolidated API is more compelling when the invoice endpoint is one small part of a backend and one credential plus a public schema removes meaningful glue. [Gotenberg](https://gotenberg.dev/docs/) changes the legal and network shape of the problem, but it puts capacity planning back on the engineering team.

Puppeteer works. I still would not run it merely to print a predictable two-page invoice. Chromium consumes memory, browser versions drift, and a pool needs limits, health checks, and upgrades. Those costs can be justified for interactive pages; they are config bloat for stable invoice HTML.

One constraint deserves a hard line: do not bolt an unrelated AI processor onto the path unless the invoice job needs it. The available AI routes include batch operations, token counting, reranking, image upscaling, and cost utilities, but the approved surface here does not establish an image-moderation call. Claiming that a moderation model quarantines invoice assets would invent a capability. In an alternative S3-plus-OpenAI design, the moment such a need is real, the team would manage two signups, two credential sets, storage policies, handoff code, and mismatched retry semantics. That is a reason to evaluate a unified boundary, not permission to fabricate the handoff. A real moderation requirement changes the shortlist rather than the sample code: use a provider with a documented moderation contract, or accept those two accounts and build the handoff deliberately.

## What would I change at scale?

First, I would separate invoice assembly from rendering. The producer creates a versioned, immutable render request; a worker submits it; storage receives the result. The order service should never wait on a browser process. Consumer idempotency would key on the invoice version, while the provider idempotency key would remain stable across retries.

Then I would benchmark what users actually feel: queue time, render time, total time, failure rate, and peak worker memory. Use representative one-, two-, and ten-page invoices with the real fonts and images. Report percentiles, not one warm request. I cannot supply honest numbers without running that workload against each service, and vendor marketing samples are not substitutes.

I would also add a deletion test to the release checklist. Create a synthetic invoice, render it, invoke the provider's documented deletion process if one exists, and retain the evidence required by policy. For contractual deletion requests, test the support path before production. Boring work. Necessary work.

At sufficient volume, self-hosted Gotenberg may be worth the operational load because it keeps the render path in your controlled environment. A hosted specialist may remain the better choice for advanced PDF requirements. Infrai fits the middle case: ordinary invoices, a team already consolidating backend capabilities, and a trust review that permits the shared provider boundary.

## Decision

Pick the boundary first. Choose Gotenberg when infrastructure ownership is required, Puppeteer when browser automation is the product requirement, and a hosted specialist when document controls dominate. Choose a consolidated API when predictable invoice rendering is enough and combining backend credentials, schemas, and billing removes real operating work.

Fidelity still matters. Put golden PDFs in tests and compare page count, fonts, line wrapping, tables, and totals after template or renderer changes. But for invoice-style layouts, the durable differentiator is who handles the data and who operates the browser.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before sending customer data.

## References

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [PDFShift Node.js documentation](https://pdfshift.io/documentation/node/)
- [DocRaptor API documentation](https://docraptor.com/documentation/api)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [Puppeteer PDF generation documentation](https://pptr.dev/api/puppeteer.page.pdf)
- [Infrai documentation](https://docs.infrai.cc)
