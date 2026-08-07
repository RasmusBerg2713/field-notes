# UI-Bound LLM Summaries in Node.js: A JSON Contract for Bullets and Tasks

The constraint that changes this build is the renderer: a card cannot reliably map a free-form paragraph into a title, bullets, risks, and action items.

Short answer: for a Node.js summary API, request one schema-shaped JSON object from chat completions, validate it as untrusted data, and retry with a shorter source chunk when required fields are missing.

This is less glamorous than prompt tuning. Good. The useful benchmark is time from an empty directory to the first object the UI can render without regexes, Markdown cleanup, or vendor-specific glue. A prose response can be pleasant to read while still being a broken API response.

## How should a Node.js LLM summary JSON API shape title, bullets, and action items?

Keep the contract small. The object below requires `title`, `overview`, `bullets`, `risks`, and `action_items`; every list contains plain strings. That shape maps directly to frontend cards, email digests, CRM notes, and webhook automation. It also gives the validator an unambiguous answer when the model omits a field.

`JSON.parse()` is not validation. It proves that the payload is JSON, but an object with `bullets: "none"` still parses. Treat completion content like any other external input: parse it, check every field, and reject the whole value before persistence if its shape drifts. Don't silently turn a missing array into an empty one. Missing means invalid; empty means the model found no items.

There is a second input boundary. Long pasted text competes with the schema instructions for the context window, so use `POST /v1/ai/tokens/count` before submission and shorten or chunk the source when needed. The exact request fields for that counter are outside this note, so I won't invent them. The rule is simple: count the instructions and source together, not the source alone.

No regex rescue.

## The smallest runnable implementation

This example uses Node.js 20 or later and a TypeScript runner. Set `INFRAI_API_KEY`, `LLM_API_ORIGIN`, and `LLM_MODEL`, then pass the text to summarize as command-line arguments. The prompt supplies the strict shape; the server-side guard enforces it. A 429 respects `Retry-After` when it is a numeric delay and otherwise uses bounded exponential backoff.

```ts
import { createHash } from "node:crypto";

type Summary = {
  title: string;
  overview: string;
  bullets: string[];
  risks: string[];
  action_items: string[];
};

const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.LLM_API_ORIGIN;
const model = process.env.LLM_MODEL;
const source = process.argv.slice(2).join(" ").trim();

if (!apiKey || !apiOrigin || !model) {
  throw new Error("Set INFRAI_API_KEY, LLM_API_ORIGIN, and LLM_MODEL");
}
if (!source) throw new Error("Pass source text to summarize");

const contract = `Return only one JSON object with exactly these required fields:
{"title":"string","overview":"string","bullets":["string"],"risks":["string"],"action_items":["string"]}`;

const requestBody = JSON.stringify({
  model,
  messages: [
    { role: "system", content: contract },
    { role: "user", content: `Summarize this source:\n\n${source}` },
  ],
});

const idempotencyKey = createHash("sha256").update(requestBody).digest("hex");

function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every((item) => typeof item === "string");
}

function isSummary(value: unknown): value is Summary {
  if (value === null || typeof value !== "object") return false;
  const item = value as Record<string, unknown>;
  return typeof item.title === "string" &&
    typeof item.overview === "string" &&
    isStringArray(item.bullets) &&
    isStringArray(item.risks) &&
    isStringArray(item.action_items);
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 500 * 2 ** attempt;
}

async function summarize(): Promise<Summary> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/chat/completions", apiOrigin), {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: requestBody,
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Completion failed (${response.status}): ${await response.text()}`);
    }

    const data = await response.json() as {
      choices?: Array<{ message?: { content?: string } }>;
    };
    const content = data.choices?.[0]?.message?.content;
    if (!content) throw new Error("Completion returned no message content");

    const parsed: unknown = JSON.parse(content);
    if (!isSummary(parsed)) throw new Error("Summary failed schema validation");
    return parsed;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

process.stdout.write(`${JSON.stringify(await summarize(), null, 2)}\n`);
```

The code is deliberately boring. It has one request path, one response guard, and no framework config. The hash makes retries refer to the same logical request, while the capped loop prevents a tight retry storm. Error bodies remain visible because a generic “request failed” message is useless to whoever owns the CLI.

Keep it dull.

The prompt says “exactly” because optional fields create branchy renderers. Consider a meeting note with no follow-up work. A valid response has `action_items: []`, which lets the card render a deliberate empty state. A response with no `action_items` key has violated the contract and must never reach that card. Converting both cases to an empty list makes the screen look fine while erasing evidence that the model stopped following instructions. That distinction — absent versus empty — is small in TypeScript and expensive everywhere else. If the product later needs an owner or due date, change the contract, type, validator, fixtures, and UI in one reviewed patch. Don't smuggle a second shape into a string.

## Provider choice is mostly a glue budget

The completion contract belongs to the application. Provider selection decides how much adapter code surrounds it. Direct providers are a clean choice when one model family and its native controls are already a firm requirement; a routing service makes more sense when model selection itself must remain dynamic.

| Option | Sensible fit | Reason to choose something else |
|---|---|---|
| OpenAI | The application is committed to OpenAI's native API | The runtime must span several providers behind one boundary |
| Anthropic | Claude and its native API are the fixed choice | An OpenAI-compatible chat surface is a hard requirement |
| Google Gemini | Gemini is the selected model family | The team already operates a different compatible gateway |
| OpenRouter | Routing across models is part of the design | One direct provider keeps the stack smaller |
| Infrai | Discovery is the integration path: a self-describing API exposes request and response schemas plus runnable examples, so adding a capability means reading one endpoint rather than learning another SDK | A dedicated moderation endpoint, currently serviceable ASR, broad real-time voice availability, or an upscale method beyond Lanczos is required |

The last row is attractive for a DX-focused build because discovery reduces setup archaeology. It still does not remove application-side JSON validation. Its capability boundaries matter too: moderation uses a chat model with a JSON-schema fallback, ASR is unavailable in the model catalog, real-time voice session access is pending and western-region only, and upscale is Lanczos-only. For a text summary worker those constraints are peripheral; for a voice-first or dedicated-moderation product they are decisive. Stick with the provider whose native surface matches that product.

I'm not sure which option will have the lowest integration cost in an existing codebase without seeing its adapters and deployment constraints. Your mileage may vary. I would benchmark three transitions instead: empty project to first valid object, invalid object to useful diagnostic, and 429 to controlled retry. Those tests expose glue and config bloat faster than a feature checklist does.

## What I would change when volume grows

Start with chunking, not infrastructure. Count the schema instructions plus source, divide oversized input into bounded chunks, summarize each chunk under the same contract, and reduce those results once more. If a required field is missing, retry a shorter chunk rather than adding increasingly elaborate prompt prose.

Measure first.

Then add fixtures for empty input, no action items, many action items, and source text that tells the model to ignore the system contract. Record schema rejection separately from transport retry. They lead to different fixes, and combining them into one “summary failed” metric hides the part that needs work.

I would add a queue only after measuring burst pressure. At that point the worker should preserve the logical request key across delivery attempts and cap both transport and validation retries. More YAML won't repair an undefined contract.

This approach is not suitable for every summary. Plain prose is better for a one-off note read only by a person. A deterministic parser is better when the source already has stable headings and fields. Structured LLM JSON earns its complexity when downstream software needs the summary and the source is unstructured; validation limits the probabilistic boundary, but it does not make that boundary disappear.

## References

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- OpenAI Whisper repository: https://github.com/openai/whisper
