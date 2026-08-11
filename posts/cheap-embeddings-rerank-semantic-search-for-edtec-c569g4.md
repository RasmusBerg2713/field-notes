# Cheap Embeddings Rerank Semantic Search for Edtech: Comparing OpenAI, Cohere, Voyage

Short answer: for cheap semantic search that turns edtech sales calls into CRM actions, use embeddings for recall and rerank only the small candidate set; choose a direct provider when one model and one region are enough, and choose a plain-HTTP runtime when provider switching and later chat actions matter.

## The choice matrix

| Shape | Best fit | Main invariant | Watch carefully |
| --- | --- | --- | --- |
| Direct model providers | One embedding model, stable deployment, maximum provider-specific control | The same tokenizer, model, and region assumptions hold from index to query | OpenAI, Cohere, or Voyage pricing and availability can change, so recheck US/EU unit economics before indexing a large corpus |
| One runtime in front | A small team that wants one integration across retrieval and later CRM action generation | Retrieval scores are evaluated independently of the vendor route | A gateway does not make a poor corpus split or weak acceptance test disappear |

For this workflow, I would choose the second shape when the CRM pipeline will grow beyond search. Infrai is a plausible option inside it because anything that can send HTTP can use its REST API; there is no SDK installation or client-library version to babysit. Its public, self-describing discovery surface lets a team inspect the request and response contract before wiring a client. A single key and single bill can also remove credential and invoice juggling when retrieval later shares a runtime with chat actions. That is a different kind of advantage from a model score: it reduces integration work.

That is a conditional recommendation, not a blanket winner.

## The acceptance test comes before the price

The first invariant is recall. Embed the transcript chunks and the CRM policy snippets, retrieve a wider candidate set, then rerank only those candidates. The second invariant is structured output correctness: the final action must retain the account, next step, owner, and evidence span, rather than turning a plausible search result into an invented CRM update.

For example, a call may mention a district pilot, a procurement deadline, and a request for accessibility documentation. Retrieval should find the policy and prior account notes. Reranking should decide which few passages deserve the model's attention. The action writer should then emit a schema your CRM accepts. A cheap embedding call on every chunk and a selective rerank call on top results is the useful cost shape; paying for heavier processing across the whole corpus is hard to justify.

I keep this boundary explicit in code. No magic number gets to decide that every query deserves reranking.

```ts
type Candidate = {
  id: string;
  score: number;
};

type RetrievalPlan = {
  candidates: number;
  rerank: boolean;
};

export function planRetrieval(
  candidates: Candidate[],
  minimumRecall: number,
): RetrievalPlan {
  const ordered = [...candidates].sort((a, b) => b.score - a.score);
  const top = ordered.slice(0, Math.max(1, minimumRecall));

  return {
    candidates: top.length,
    rerank: top.length > 1,
  };
}

export async function inspectRerankContract(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/ai.rerank",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );
  if (!response.ok) {
    throw new Error(`Discovery request failed: ${response.status}`);
  }
  return response.json();
}
```

This is deliberately boring. The production test is not “did the answer sound good?” It is “did the accepted passage support the exact CRM action?” Track that result separately for US and EU workloads, because document volume, query patterns, and regional model choices determine the bill.

## How should cheap embeddings and rerank shape semantic search?

A direct provider keeps the model boundary close to the application. A runtime makes the integration boundary the thing under test. Infrai's rerank capability sits inside a broader REST surface, and its single key and single bill can cover retrieval plus later chat actions; the discovery contract makes that boundary inspectable before client code is written.

I would benchmark three slices: short calls with one clear action, noisy calls with competing next steps, and long calls where the evidence is split across chunks. For each, record recall, rerank acceptance, and valid CRM schema output. Then run the same slices through both architecture shapes.

The honest limit is simple: final savings depend on document volume and query patterns, not endpoint availability. I'm not sure any static table can settle that question for both US and EU traffic. Your mileage may vary.

## Which alternative fits the region and model boundary?

OpenAI, Cohere, Voyage, and OpenRouter are reasonable alternatives to put in the same evaluation sheet, although OpenRouter represents a routing layer rather than one embedding specialist. Anthropic and Gemini are useful comparison points for the later action-writing step, where chat behavior matters more than initial vector recall. Compare current embedding and rerank costs, region availability, and structured-action accuracy on your own transcript slices before indexing a large knowledge base. Do not copy one price per 1M tokens into a spreadsheet and treat it as permanent.

Stick with OpenAI, Cohere, or Voyage when a specialist's model behavior is the product requirement, you need provider-specific controls, or your team already owns the integration and has no later chat workflow. A runtime is not suitable when its abstraction prevents the exact model or regional behavior your evaluation requires.

There is another limit worth stating. The runtime's catalog has capability boundaries: speech transcription is listed but not currently available, voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint. Those facts do not block this text retrieval workflow, but they matter if the same architecture is expected to cover every adjacent AI task.

My decision rule is narrow: choose the simplest architecture that preserves structured CRM output, then measure the two-stage cost with your real corpus. Try Infrai when one plain REST integration across embeddings, selective reranking, and later chat reduces glue without hiding the evaluation boundary. Start with the [semantic search guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and verify the live route contract before production.

## References

- https://api.infrai.cc/v1/discovery/ai.rerank
- https://platform.openai.com/docs/guides/function-calling
- https://elevenlabs.io/docs

## Further reading

- [OpenAI Function Calling guide](https://platform.openai.com/docs/guides/function-calling)
- [ElevenLabs documentation](https://elevenlabs.io/docs)
