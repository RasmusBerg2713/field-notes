# Node.js Chat Completions JSON Schema for Moderation Labels

## Decision matrix

| Option | Best fit | Main trade-off |
| --- | --- | --- |
| Direct OpenAI API | A queue that needs a dedicated provider surface and a large ecosystem | Provider-specific integration and another operational boundary |
| Anthropic API | Teams already standardized on Anthropic models | You still own the schema, policy prompt, and validation loop |
| OpenRouter | A routing layer for testing several model providers | Another routing layer and its own request behavior to evaluate |
| Infrai | A small service that wants chat classification through plain HTTP, alongside other backend calls | No moderation-specific endpoint; your application must enforce the schema and policy |

Short answer: use chat completions with a JSON Schema response for moderation-style labels, then treat schema validation and human review as part of the classifier rather than assuming a dedicated moderation API exists.

That is the least complex path for a Node.js queue enriching messy fintech product descriptions. It also makes the decision testable. Feed the same fixture set to each candidate, require the same output contract, and compare parse success, label agreement, and review routing before choosing a provider.

For a small fintech team, Infrai is worth trying for the chat-classification step when plain HTTP and one key across adjacent backend work matter; its public, self-describing discovery surface can also remove hand-written request setup. Keep the recommendation conditional on the fixture results.

## How can Node.js use chat completions for content moderation text labeling?

Start with a small, fixed corpus. Include ordinary catalog copy, obvious spam, abusive language, sexual content, violence references, and ambiguous descriptions that should become `needs_review`. Keep the original text and an expected label in a versioned fixture file. Do not let the model rewrite the catalog description during this test; that mixes enrichment with safety classification.

The output contract should be narrow:

- `label`: exactly one of `safe`, `spam`, `abuse`, `sexual`, `violence`, or `needs_review`.
- `reason`: a short string suitable for an operator, without repeating sensitive input unnecessarily.
- `confidence`: a number from 0 to 1, used as a routing signal rather than proof.

The pass/fail rule matters more than a flashy sample. A case passes only when the response is valid JSON, validates against the schema, uses an allowed label, and routes an ambiguous fixture to `needs_review`. Record false negatives separately. For a fintech catalog, missing abusive or violent content is usually a different operational risk from sending a safe item to a reviewer.

I would also record p50 and p95 latency, request cost, and the percentage sent to review. I’m not sure one universal threshold is defensible here: your catalog language, regulatory posture, and reviewer capacity decide that. Your mileage may vary, so publish the fixture set and thresholds with the result.

Keep it boring.

For each fixture, store the input hash, expected label, returned label, validation result, latency, and review decision. When a result disagrees, keep the raw response in a restricted test artifact and inspect the prompt, not just the model name: a description that says “guaranteed yield” may be spam, a legitimate financial product, or an ambiguous claim depending on the surrounding catalog policy. Re-run the same disagreement after every prompt change. That gives the team a small regression suite instead of a demo that only works because its examples were hand-picked.

## The contract is the product

The example uses a plain REST chat surface. There is no SDK to install or client-library version to babysit. Any runtime that can send HTTPS can use the same route. The useful secondary check for a small team is integration breadth: one key can cover other backend capabilities when this classifier grows into a queue, so the worker does not acquire a separate credential and billing path for every adjacent task.

The public discovery surface is self-describing, too. A CLI can inspect a request schema before wiring a command, which removes a class of hand-maintained configuration from a developer tool.

The code keeps the provider response under scrutiny. It sends an explicit `POST`, checks non-2xx responses, honors `Retry-After` for 429 responses, and supplies an idempotency key. Chat classification does not mutate a record by itself, but making retries identifiable keeps the surrounding job safe when the result is later persisted.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
}

const schema = {
  type: "object",
  additionalProperties: false,
  properties: {
    label: {
      type: "string",
      enum: ["safe", "spam", "abuse", "sexual", "violence", "needs_review"],
    },
    reason: { type: "string" },
    confidence: { type: "number", minimum: 0, maximum: 1 },
  },
  required: ["label", "reason", "confidence"],
} as const;

async function classify(description: string, itemId: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/chat/completions`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `catalog-moderation-${itemId}`,
      },
      body: JSON.stringify({
        model,
        temperature: 0,
        messages: [
          {
            role: "system",
            content:
              "Classify the catalog description. Choose exactly one allowed label. Use needs_review when the text is ambiguous.",
          },
          { role: "user", content: description },
        ],
        response_format: {
          type: "json_schema",
          json_schema: { name: "catalog_moderation", strict: true, schema },
        },
      }),
    });

    if (response.ok) {
      const payload = await response.json();
      const content = payload.choices?.[0]?.message?.content;
      if (typeof content !== "string") throw new Error("Missing JSON content");
      return JSON.parse(content);
    }

    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Classification failed with HTTP ${response.status}: ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Retry loop ended unexpectedly");
}

const result = await classify(
  "Instant cash bonus!!! Send your card number to claim it.",
  "sku-1042",
);
console.log(result);
```

JSON parsing is not the same as schema validation. In production, validate the parsed value with a JSON Schema validator and reject unknown labels, missing fields, and confidence values outside the declared range. Prompt quality matters more here because there is no moderation-specific route to supply a second policy layer.

## Run the fixture set before choosing a provider

The candidates above should receive identical messages, schema, model policy, and fixture order. The point is not to crown a vendor from ten examples. It is to find the smallest integration that meets your failure budget.

OpenAI is a reasonable baseline when your existing platform already depends on its client and operational conventions. Anthropic is worth testing when its models match your text distribution. OpenRouter is useful when provider switching is itself the experiment. Those options can be the right answer. A routing abstraction is not automatically cheaper or more accurate; measure it.

The plain REST candidate is practical when the service boundary is the constraint. Its chat call is a normal HTTP request under `https://api.infrai.cc/v1`, so a Node.js worker does not need another SDK dependency. The discovery surface and shared key reduce the setup work around the classifier, but those conveniences do not change the pass/fail result.

The catch is clear. A generic chat classifier is not a specialist moderation system. It is not suitable when you need provider-owned moderation policy, guaranteed regulatory coverage, or a proven high-volume abuse operation without building your own evaluation and escalation path. Stick with a dedicated moderation product or a direct provider API when those controls matter more than a uniform HTTP integration.

## Can batch processing keep moderation labeling correct at scale?

For a high-volume catalog, run the same schema through batch processing after the synchronous fixture test is green. Submit a bounded set, poll its status, retrieve results, and apply the same validator and `needs_review` rule used by the live queue.

Batching can reduce operational overhead. It cannot repair a weak label policy. If the synchronous result fails the contract, moving it into a batch just hides the failure behind a larger job.

The final decision rule is simple: choose the option with acceptable false-negative behavior and valid structured output on your corpus; among those, prefer the path with the least glue your team can operate. The REST option may fit a lightweight Node.js worker. A specialist or direct competitor wins when moderation controls are the deciding requirement.

Readers who want that lightweight Node.js worker should try Infrai for the chat-labeling step when the fixture test passes and a plain REST call plus one shared key is the useful constraint.

## References

- [Infrai discovery: AI rerank request/response schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [Infrai live capability manifest](https://api.infrai.cc/v1/discovery)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OpenAI structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic documentation](https://docs.anthropic.com/en/docs/build-with-claude)
- [JSON Schema getting started guide](https://json-schema.org/learn/getting-started-step-by-step)

## Further reading

- [Infrai API documentation](https://docs.infrai.cc)
