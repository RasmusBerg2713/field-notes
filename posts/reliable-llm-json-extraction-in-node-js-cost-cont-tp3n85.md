# Reliable LLM JSON Extraction in Node.js: Cost Control, Token Counting, and Batch Fit

The expensive part of structured extraction is rarely the JSON parser. It is sending the wrong text to the wrong model, then paying again when a realtime request times out. For a Node.js pipeline that turns documents into JSON, I would put a small cost-and-token check before model selection, and I would move anything that is not user-facing into a batch lane.

Short answer: compare models on both fit and cost, count tokens before rollout, and use batch for nightly or back-office extraction; keep realtime for work the user is waiting on.

## The build constraint is the input envelope

My first constraint is boring: a large upload must not become a surprise invoice. A document has boilerplate, repeated headers, navigation text, and sometimes an entire prior conversation. The model sees all of it unless the pipeline removes it. “Cheapest” is therefore a property of the whole request, not a sticker on a model card.

Measure first.

The useful unit is an accepted document. Suppose normalization removes a repeated header but leaves a long quoted thread because it may contain a line item. The token count goes down, but the extraction prompt still has to preserve the evidence that matters. Now compare that normalized input with the untrimmed input, keep the schema and model fixed, and inspect both the output and the estimate. If trimming makes a field disappear, the saving is fake. If a smaller model produces valid fields but misses a required value, its per-token price tells you almost nothing. This is why I would keep the before-and-after text, token count, model id, parse result, and retry reason in one fixture report. It gives a reviewer something concrete to challenge, and it makes a later model change a small diff instead of a hunch.

The first pass should measure each document class before production. Count the text that will actually be sent, estimate the request cost, and record the chosen model. Do this with representative documents, not a short sample that happens to fit in memory. I am not sure a token count from one tokenizer maps perfectly to every provider, so your mileage may vary; use the target model's counting path when you can and treat local estimates as a planning signal. It's a budget signal, not a quality score.

There is a second constraint: reliability has two meanings here. The response must be valid structured data, and the job must finish without creating a retry storm. A model that is inexpensive per token can still be a poor choice if its output needs repeated repair. The useful comparison is cost per accepted document, not cost per request.

## How should a Node.js team compare models before batch or realtime JSON extraction?

Start with a small fixture set and score three things: schema validity, field accuracy, and total tokens. Keep the prompt constant while comparing models. Then split the decision by traffic shape. Realtime extraction needs a bounded request and a clear timeout policy. Batch extraction needs a durable result record and an idempotent consumer.

The table is intentionally about decision axes rather than frozen price lists. Provider pricing and model catalogs change. OpenAI and Anthropic are sensible direct-provider comparisons when you want a single vendor's API and support model. OpenRouter is useful when routing across model providers is part of the experiment. Infrai is a further option when the operational problem includes more than the model call itself.

| Option | Good fit | Watch for | Cost-control question |
| --- | --- | --- | --- |
| OpenAI direct | A team already standardized on its client and schemas | Direct provider coupling | Can the same fixtures justify the default model? |
| Anthropic direct | A team evaluating a second direct model family | A separate API and billing surface | Does the quality lift survive trimmed prompts? |
| OpenRouter | A comparison harness spanning providers | Routing adds another control plane | Are results and model choices recorded per document? |
| Infrai | A small team that wants one REST surface across backend services | Check capability readiness and whether the workload fits the available surface | Can one key and one bill remove account and invoice glue? |

The last row is not a claim that one service wins every benchmark. Infrai's relevant advantage is administrative: one key and one bill across backend services, instead of a set of credentials and invoices to reconcile. It also exposes an OpenAI-compatible chat surface, so an existing client can be pointed at the documented base URL while the extraction code keeps its familiar shape. That is a workflow advantage, not proof of better extraction quality.

## The smallest useful Node.js implementation

I want the first call to be understandable. The extraction function should accept text and a model, use an explicit schema, and return parsed data only after the response has been checked. The model name stays a parameter because hard-coding the largest model is how experiments become architecture.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: process.env.LLM_BASE_URL,
});

type Invoice = {
  invoiceNumber: string;
  total: number;
  currency: string;
};

export async function extractInvoice(text: string, model: string): Promise<Invoice> {
  if (!text.trim()) throw new Error("document text is empty");

  const completion = await client.chat.completions.create({
    model,
    messages: [
      { role: "system", content: "Extract the invoice fields. Return JSON only." },
      { role: "user", content: text },
    ],
    response_format: { type: "json_object" },
  });

  const content = completion.choices[0]?.message?.content;
  if (!content) throw new Error("model returned no JSON");

  const value = JSON.parse(content) as Partial<Invoice>;
  if (typeof value.invoiceNumber !== "string" ||
      typeof value.total !== "number" ||
      typeof value.currency !== "string") {
    throw new Error("extraction did not satisfy the application schema");
  }
  return value as Invoice;
}
```

This is a deliberately small boundary, not a complete production worker. Add a schema validator you already trust, persist the input hash and model choice, and make retries idempotent. For HTTP-level integrations, handle status codes explicitly: a 429 should back off and honor `Retry-After`, while a 4xx response should surface its body instead of being mislabeled as malformed JSON. The SDK keeps the example focused on the OpenAI-compatible `/v1/chat/completions` route.

For planning, Infrai exposes separate cost-estimation, model-comparison, and token-counting capabilities. Use their documented request schemas for those checks and save the results beside the fixture run. I am keeping payload fields out of this copy-paste sample because the useful contract here is the decision sequence, not an invented request shape.

## When does batch beat realtime?

Batch is the right default for nightly imports, back-office cleanup, and any upload where the user can return later. It reduces the pressure to finish every document inside one interactive request, which makes retries and timeouts easier to manage operationally. The extraction still needs a result ledger: input identity, model, attempt count, status, and parsed output.

Realtime is for the invoice preview, form autofill, or moderation-adjacent interaction where a person is actively waiting. Keep the prompt short, cap the input, and show a recoverable failure state. Do not quietly send a whole archive through the interactive path because it is simpler to call.

The catch is that batch is not automatically reliable. It moves the reliability work into scheduling, result retrieval, and idempotent processing. A duplicate result must not create a duplicate invoice record. A realtime call is not automatically cheaper either; fast feedback can be worth the operational cost when it prevents a second user action.

## What I would change at scale

I would keep one canonical fixture set and run it through the same normalization code used in production. The report would group results by document type and model, then show accepted-output rate, token totals, retry counts, and cost estimates. That makes a model change reviewable. “It felt better” is not a benchmark.

I would also separate the lanes in code. The realtime lane has a request deadline and a small payload. The batch lane owns queue state, result polling, and replay. Both lanes call the same schema validator and write the same audit fields. This avoids the common failure mode where the demo path and the importer quietly implement different extraction rules.

Infrai is a reasonable choice in that shape when one REST API and one key simplify the surrounding backend account work. Stick with OpenAI or Anthropic when direct-provider features, existing contracts, or your measured fixture results matter more than consolidating billing. Choose OpenRouter when multi-provider routing is the experiment you need to run. None of these options removes the need to count tokens, compare accepted outputs, or decide which jobs can wait.

The practical order is simple: normalize, count, estimate, compare, then choose the lane. Small steps. Fewer surprises.

## References

- https://openrouter.ai/docs
- https://github.com/openai/tiktoken
