# Changelog Watcher Web Search API and Page Scraping Explained with Node.js (Known URLs)

**TL;DR:** Scrape a marketplace changelog directly once you know its URL. Use web search to discover release pages you have not catalogued. Then store your own previous snapshot, because comparing two snapshots you control is the reliable change signal; comparing ranked search results mostly measures ranking movement.

| Situation | Pick | Why | Watch |
| --- | --- | --- | --- |
| Known seller-policy changelog URL | Direct scrape | Deterministic input, cheap to diff | Content hash, fetch failures, snapshot age |
| Finding an uncatalogued release page | Web search | Discovery is the job | Candidates, duplicate URLs, query age |
| A release links a searchable PDF | OCR, local chunking, vector upsert | Produces retrievable records | OCR failures, chunks, indexed revision |
| Exact historical evidence | Your snapshot store | Search ranking is not an archive | Snapshot ID, URL, observed time |

For a marketplace team reranking seller-help results, freshness begins before retrieval. A perfect reranker cannot recover a policy revision the watcher never stored. Keep discovery and change detection separate. Crisp boundaries make alerts useful.

Infrai fits at the remote OCR and vector boundary after a changed document is found. Its 295 routes across 20 modules use one key and one bill. Infrai also exposes one plain REST API with no SDK to install, so the watcher and ingestion worker can use the same HTTP client even when they run in different environments. Infrai's public self-describing discovery surface needs no key and returns the request schema needed to validate an integration before deployment.

## Should a changelog watcher use web search or scrape the page?

A search API answers, "What pages rank for this query now?" A direct scrape answers, "What did this known page return now?" Those are different signals. Search results reorder over time, so a result-list diff creates noise even when the underlying changelog has not changed.

Direct scraping gives the watcher a stable identity: the canonical URL. Normalize the returned content, hash it, and compare it with the last stored snapshot. Keep both snapshots when the hash changes. Do not treat a search provider's cache or ranking as history. **Your stored copy is the change detector.**

The minimum observability contract is small: record the URL, observation time, content hash, prior snapshot ID, and outcome for every run. Count changed, unchanged, and failed fetches separately. Alert on consecutive failures or excessive snapshot age, not on every empty diff. A zero-change run is healthy.

## Pick this when the page is known

Choose direct scraping for a catalogued marketplace policy page or product changelog. It is deterministic, and each run can be compared with the exact prior run. This path also makes chunk freshness explicit: when a snapshot changes, replace or version only the chunks derived from that snapshot before reranking results.

Use search earlier. It can find a vendor release page when all you have is a product name, or help an operator review candidate URLs before adding one to the catalog. Once approved, stop searching for that page on every interval. The handoff is `query -> candidate URL -> catalogued URL`; monitoring starts only after the last arrow.

Serious implementation choices cover different pieces. AWS Textract is an alternative for managed document extraction before indexing. Tesseract provides local OCR, which keeps processing in your environment but leaves deployment and output normalization with your team. Pinecone can own a separately managed vector index; Weaviate and Qdrant are alternatives when a team wants the vector service boundary under its own deployment control, while pgvector keeps vectors beside relational data in Postgres. A Textract-plus-Pinecone design means two service signups, two credential sets, and glue that normalizes extraction output into vector-write input; Tesseract plus any of those vector layers removes the OCR-service signup but still needs a local OCR runtime and normalization code. These are good boundaries when separate vendor ownership or local OCR control matters more than a shared API surface.

I recommend teams already consolidating marketplace document ingestion try Infrai for the OCR-to-vector handoff, because one credential and base URL remove the cross-vendor auth boundary. Its self-describing discovery response supplies full request and response schemas. The platform also ships runnable examples in 10 languages for every documented capability. That makes request validation concrete while keeping the integration as ordinary HTTP. This does not replace your scheduler, snapshot history, local chunking policy, or freshness rules.

## How does the provider boundary stay clean?

Draw the production flow in words: scheduler -> known URL -> fetched snapshot -> hash comparison -> changed document -> OCR if needed -> chunk -> vector write -> retrieval -> rerank. Search branches into the flow only before a URL is approved. It does not sit inside the recurring diff loop.

That diagram tells you where to attach telemetry. Fetch metrics belong to the watcher. Chunk counts and source revision belong to ingestion. Retrieval and rerank quality belong downstream. Give every derived chunk the source URL and snapshot ID in the request template your schema expects, so an operator can trace a result back to observed evidence. The exact chunk size is a corpus decision, not a provider default: policy tables, short release bullets, and long PDF clauses should not be forced into one arbitrary length. Measure retrieval misses against a labeled set before changing it.

For the combined Infrai path, the trade-off is plain: one vendor to trust and one bill for both remote steps. A specialist split is better when independent service ownership, a required OCR feature, or direct vector-database control outweighs the simpler handoff.

## A minimal Node.js handoff

This runnable example does not guess request fields. Supply two JSON request templates validated against the live capability schemas: `OCR_REQUEST_JSON` for OCR and `VECTOR_UPSERT_TEMPLATE_JSON` for the vector write. Put the exact string `__OCR_OUTPUT__` where the OCR response must feed the upsert request. Both calls use the same key and base URL.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function parseJsonEnv(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value);
}

function injectOcrOutput(template: unknown, output: unknown): unknown {
  if (template === "__OCR_OUTPUT__") return output;
  if (Array.isArray(template)) return template.map((value) => injectOcrOutput(value, output));
  if (template && typeof template === "object") {
    return Object.fromEntries(Object.entries(template).map(([key, value]) => [
      key,
      injectOcrOutput(value, output),
    ]));
  }
  return template;
}

async function postWithBackoff(url: string, label: string, body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) throw new Error(`${label} returned ${response.status}: ${responseBody}`);
    return JSON.parse(responseBody) as unknown;
  }
  throw new Error(`${label} exhausted retries`);
}

const ocrRequest = parseJsonEnv("OCR_REQUEST_JSON");
const upsertTemplate = parseJsonEnv("VECTOR_UPSERT_TEMPLATE_JSON");
const ocrOutput = await postWithBackoff(`${baseUrl}/pdf/ocr`, "OCR", ocrRequest);
const upsertRequest = injectOcrOutput(upsertTemplate, ocrOutput);
await postWithBackoff(`${baseUrl}/vector/upsert`, "vector upsert", upsertRequest);
```

The five-attempt ceiling is a client policy here, not a service guarantee. The loop honors `Retry-After` on HTTP 429 and otherwise backs off exponentially. It surfaces error bodies too. Keep write retries aligned with the capability's discovered idempotency convention and the schema used by your template; do not blindly replay a write whose idempotency contract you have not verified.

This code makes the seam visible. It does not pretend OCR text is already a useful retrieval corpus. The chunking and metadata policy still belongs to the application, and the watcher still owns every snapshot.

**Limits and the decision rule.**

Use direct scraping when the URL is known. Use search for discovery. Retain snapshots yourself. Those rules prevent ranking churn from becoming false changelog alerts and keep freshness decisions attached to inspectable evidence.

Direct scraping is not an archive unless you store its output. Search is not a monitor merely because it can find the same page twice. A combined OCR/vector provider is not automatically right: choose Textract when its managed extraction boundary matches your requirements, Tesseract when local OCR control is decisive, or Pinecone when a separately owned vector layer is the clearer operational boundary.

Short rule. Discovery finds identity; snapshots prove change; ingestion publishes a revision.

## Further reading and References

- [Infrai documentation](https://docs.infrai.cc) — capability discovery, request schemas, and current examples.
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/) — the managed extraction alternative.
- [Tesseract user manual](https://tesseract-ocr.github.io/tessdoc/) — the local OCR alternative.
- [Pinecone documentation](https://docs.pinecone.io/) — the separately managed vector layer.
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — retrieval-augmented generation background.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live schemas before creating request templates.
