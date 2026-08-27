# 3 Fine-Tuning Alternatives for Support Ticket Tagging via Embeddings and CRM Actions

**The answer is:** start support ticket tagging and sales-call-to-CRM action extraction with a zero-shot LLM behind a strict output validator; move stable, repetitive tags to an embeddings classifier, and add reranking only when label descriptions are too similar for nearest-neighbor distance alone.

| Path | First useful implementation | Structured-output control | Main catch |
|---|---|---|---|
| Zero-shot or few-shot chat | Define labels, examples, and a JSON contract | Strong when every response is parsed and validated | Each item invokes generation; prompt changes can alter behavior |
| Embeddings plus lightweight logic | Embed labeled examples and compare vectors | The classifier emits only code-owned label IDs | Needs a stable label space and representative examples |
| Reranking candidate descriptions | Score the ticket against a short candidate set | Application code maps the top candidate to a label ID | Adds another retrieval stage and more moving parts |

For a small B2B SaaS team, the recommendation is boring on purpose: use chat classification for the pilot, record every validation result, and promote only proven, repetitive categories to the embeddings path. Don't fine-tune before the team has a versioned taxonomy and an evaluation set. Otherwise the training job hardens disagreements about labels instead of resolving them.

This applies to more than incoming support tickets. A sales-call summary can produce CRM actions such as `schedule_demo`, `send_security_review`, or `no_action`, using the same contract and test harness. The hard part isn't obtaining plausible prose. It is returning the right action ID, with the required evidence, every time.

## Should support ticket tagging use an embeddings classifier, a zero-shot LLM, or rerank?

Use a zero-shot LLM when labels still change weekly, the input carries nuanced context, and shipping a clear prompt is cheaper than maintaining training infrastructure. A few labeled examples in the prompt can clarify boundaries without creating a fine-tuning pipeline. This is usually the quickest route to a real baseline because the application can define labels and request JSON in one call.

Use embeddings when the categories are stable and repetitive. The application stores vectors for reviewed examples, retrieves nearby examples for a new ticket or transcript, and applies lightweight logic such as a similarity threshold plus an abstain branch. pgvector is one standards-friendly way to keep vector similarity search beside ordinary Postgres data, though the architecture doesn't depend on that extension. The attraction is control: label IDs come from application code, not generated text, and the recurring classification path can become less expensive at scale.

Reranking fits a narrower gap. Represent each possible tag as a candidate description, retrieve a small candidate set, then score the input against those descriptions. It can help when `billing_question`, `refund_request`, and `invoice_correction` sit close in embedding space but their written policies distinguish them. It also adds a stage. More stages mean more traces, more failure states, and more config — exactly the tax a tiny developer-tools team should question.

No path removes the need for abstention. If a call asks for both a security review and revised pricing, forcing one label corrupts the CRM record. Allow multiple actions where the business process permits them; otherwise return `needs_review` with evidence. Quietly guessing is worse than a visible queue.

## Structured output correctness is the first decision criterion

Accuracy needs a more useful definition than “the label looked right.” For this workflow, a result is correct only if it parses, matches the current schema, uses allowed label IDs, cites evidence present in the source text, and follows cardinality rules. A fluent answer that invents `urgent_enterprise_followup` is invalid even if a salesperson likes the wording.

Version the contract — for example, `crm-action/1.0` — separately from the prompt. Reject unknown fields. Reject labels outside the allowlist. Check that every quoted evidence span occurs in the normalized input. Those checks are deterministic, cheap to run, and identical across chat, embeddings, and reranking. They also keep the benchmark honest: malformed output cannot be waved through as “basically correct.” The evaluation set should preserve the ugly cases rather than average them away. Include empty transcripts, two intents in one utterance, negation (“don't schedule a demo”), quoted customer text, and messages that try to override the classifier instructions. OWASP's LLM application guidance treats prompt injection and insecure output handling as application risks, so the source text belongs in a data field and the returned object must still pass normal authorization and validation. A tagger must never turn model text directly into a privileged CRM mutation. Track separate counters for schema validity, exact label match, abstention, and evidence validity, then slice them by label and input source; a single aggregate score can hide a classifier that gets common `no_action` calls right while missing every security-review request. I'm not sure one universal acceptance threshold makes sense. Your mileage may vary with the cost of a false action: a reversible follow-up task can tolerate more automation than deleting or reassigning a live opportunity. Set the release rule before looking at results. One practical policy is to require zero schema failures in the acceptance set and route any label below its agreed precision target to review. The percentage is a business decision, not a model fact. Keep it in config with an owner and a review date (not buried in a prompt).

Measure it.

This is where most “cheap” comparisons go wrong. They count inference calls while ignoring review time, invalid payload retries, evaluation maintenance, vector refreshes, and the engineer who has to explain why the CRM changed. Price can matter, but total operator work and correction risk usually decide the design.

## A simple TypeScript implementation keeps the model replaceable

The useful abstraction is a classifier that returns a discriminated union. It shouldn't expose raw provider output to the rest of the application. The zero-shot adapter below calls the verified chat-completions route, while the validator owns the actual contract.

```ts
type ActionId =
  | "schedule_demo"
  | "send_security_review"
  | "no_action";

type Classification =
  | { status: "accepted"; actions: ActionId[]; evidence: string[] }
  | { status: "review"; reason: string };

const allowedActions = new Set<ActionId>([
  "schedule_demo",
  "send_security_review",
  "no_action",
]);

function validateClassification(
  value: unknown,
  source: string,
): Classification {
  if (!value || typeof value !== "object") {
    return { status: "review", reason: "SCHEMA_MISMATCH" };
  }

  const candidate = value as Record<string, unknown>;
  if (!Array.isArray(candidate.actions) || !Array.isArray(candidate.evidence)) {
    return { status: "review", reason: "SCHEMA_MISMATCH" };
  }

  const actions = candidate.actions.filter(
    (item): item is ActionId =>
      typeof item === "string" && allowedActions.has(item as ActionId),
  );
  const evidence = candidate.evidence.filter(
    (item): item is string => typeof item === "string" && source.includes(item),
  );

  if (actions.length !== candidate.actions.length || evidence.length === 0) {
    return { status: "review", reason: "UNVERIFIED_OUTPUT" };
  }

  return { status: "accepted", actions, evidence };
}

async function classifyCall(source: string): Promise<Classification> {
  const endpoint = new URL(
    "/v1/chat/completions",
    process.env.AI_BASE_URL,
  );
  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      "authorization": `Bearer ${process.env.AI_API_KEY}`,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      model: process.env.AI_MODEL,
      messages: [
        {
          role: "system",
          content: "Classify CRM actions. Return only JSON with actions and evidence.",
        },
        { role: "user", content: JSON.stringify({ source }) },
      ],
    }),
  });

  if (!response.ok) {
    return { status: "review", reason: `UPSTREAM_${response.status}` };
  }

  const envelope = await response.json() as {
    choices?: Array<{ message?: { content?: string } }>;
  };
  const content = envelope.choices?.[0]?.message?.content;
  if (!content) return { status: "review", reason: "EMPTY_OUTPUT" };

  try {
    return validateClassification(JSON.parse(content), source);
  } catch {
    return { status: "review", reason: "INVALID_JSON" };
  }
}
```

Keep the embeddings and rerank adapters behind the same return type. They can select an `ActionId` without generation, then attach evidence from the matched example or candidate description only when that evidence can be tied back to the input. The CRM writer receives `Classification`, never a vector score or chat envelope. That's less glue at the dangerous boundary.

The deployment loop can stay small: shadow the new adapter on recorded, consented inputs; compare it with reviewed labels; enable it for one low-risk action; watch review and abstention rates; then expand. Retry read-only classification requests after `429` responses with bounded exponential backoff, but give the later CRM mutation its own idempotency key so a retry cannot duplicate an action. Store the taxonomy version, adapter version, and decision trace with each result. Without those three values, a later regression is archaeology.

Keep it dull.

## When are embeddings or reranking the better runner-up?

Stick with zero-shot chat when the taxonomy is moving, the volume is modest, and tickets need contextual judgment that cannot be represented by a few stable examples. It is not suitable when every item must have tightly predictable latency and generation cost, or when policy forbids sending source text to the chosen runtime. Self-hosting may change the policy calculation, but it also shifts operations onto the team.

Choose embeddings plus lightweight logic when labels have settled and repeated examples dominate. The catch is threshold calibration. A nearest neighbor always exists, even for an unrelated input, so low similarity must lead to review rather than the “closest” tag. Embeddings are also a poor fit when a label depends on a precise rule that semantic similarity blurs, such as distinguishing a request for pricing from consent to issue a quote.

Choose reranking when candidate descriptions contain the discriminating detail and first-stage retrieval can reliably keep the correct label in a small set. Don't add it to a six-label taxonomy just because the benchmark has a rerank column. The extra component earns its place only if it fixes a documented confusion class without making time-to-first-call and debugging materially worse.

Fine-tuning becomes reasonable later, after reviewed data is plentiful, labels are stable, and the measured errors point to behavior that prompting, retrieval, and deterministic rules cannot fix. Before that point, it creates a data and deployment lifecycle with no guarantee that structured-output correctness improves. The simplest implementation is the one whose mistakes the team can see, reproduce, and route safely.

## Sources

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://github.com/pgvector/pgvector
