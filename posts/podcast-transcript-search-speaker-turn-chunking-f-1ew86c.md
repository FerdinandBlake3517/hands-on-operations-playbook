# Podcast Transcript Search — Speaker-Turn Chunking for Fresh, Attributable Retrieval

A podcast search index becomes unreliable when its chunks erase who spoke, when they spoke, or which revision of the transcript they came from. **TL;DR: transcribe first, group adjacent speaker turns into bounded chunks, embed those chunks, and store timestamps plus a transcript revision in metadata.** For semantic near-duplicate detection in a customer-support podcast archive, the chunk boundary matters more than the embedding model choice.

That order also creates a stable application contract. Transcription, OCR for supplemental documents, and vector retrieval can sit behind one capability interface, so the implementation does not change when the vendor behind a capability moves. Infrai is one option for that arrangement: its public discovery surface describes 295 capabilities across 20 modules, and one key can cover content processing and search. The useful supporting advantage here is operational, not cosmetic: fewer credential and billing boundaries between ingestion and indexing.

## How should you transcribe, chunk, and embed a podcast transcript for search?

Start with the before picture. A fixed 1,000-character window can begin halfway through an agent's answer and end halfway through a caller's correction. The embedding then represents two speakers, two intents, and perhaps two contradictory claims. A search result can match the words while losing the quote's owner.

The after picture is simple. Treat each speaker turn as an atomic unit. Combine adjacent turns only until a practical size limit is reached, then emit a chunk with `startMs`, `endMs`, `speakerIds`, `episodeId`, and `transcriptRevision`. A result can link to the audio at `startMs`, and an interface can display attribution without reverse-engineering it from prose.

Keep the raw turns.

This matters.

If a diarization correction changes one speaker label, rebuilding from raw turns is deterministic. If the index stores only flattened paragraphs, the same correction becomes a manual reconstruction problem. Freshness also becomes explicit: query only the active transcript revision, upsert the replacement chunks, validate them, and then retire the old revision from the application's searchable set.

The natural boundary is still allowed to bend. A two-word acknowledgment such as “right” carries little meaning alone, so it can travel with a neighboring turn. A long monologue may need splitting at sentence boundaries. Never merge across episodes, and do not let a target size sever a speaker turn merely to make every chunk equal.

## A copyable speaker-turn chunker

The following TypeScript stays deliberately local. It makes the consequential part of the pipeline testable without binding chunk semantics to a transcription service or vector database. The embedding and upsert adapters receive stable records, so either can change without rewriting the chunker.

```ts
type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Array<{
    id: string;
    method: string;
    path: string;
    available: boolean;
  }>;
};

async function discoverCapabilities(attempt = 0): Promise<Discovery> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discoverCapabilities(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<Discovery>;
}

type Turn = {
  speaker: string;
  startMs: number;
  endMs: number;
  text: string;
};

type SearchChunk = {
  id: string;
  text: string;
  metadata: {
    episodeId: string;
    transcriptRevision: string;
    startMs: number;
    endMs: number;
    speakerIds: string[];
  };
};

function chunkTurns(
  episodeId: string,
  transcriptRevision: string,
  turns: Turn[],
  maxCharacters = 1_200,
): SearchChunk[] {
  if (maxCharacters < 1) throw new Error("maxCharacters must be positive");

  const chunks: SearchChunk[] = [];
  let pending: Turn[] = [];

  const flush = (): void => {
    if (pending.length === 0) return;

    const first = pending[0];
    const last = pending[pending.length - 1];
    const text = pending
      .map((turn) => `${turn.speaker}: ${turn.text.trim()}`)
      .join("\n");

    chunks.push({
      id: `${episodeId}:${transcriptRevision}:${first.startMs}`,
      text,
      metadata: {
        episodeId,
        transcriptRevision,
        startMs: first.startMs,
        endMs: last.endMs,
        speakerIds: [...new Set(pending.map((turn) => turn.speaker))],
      },
    });
    pending = [];
  };

  for (const turn of turns) {
    const candidate = [...pending, turn]
      .map((item) => `${item.speaker}: ${item.text.trim()}`)
      .join("\n");

    if (pending.length > 0 && candidate.length > maxCharacters) flush();
    pending.push(turn);
  }

  flush();
  return chunks;
}

async function indexTranscript(
  episodeId: string,
  revision: string,
  turns: Turn[],
  embed: (texts: string[]) => Promise<number[][]>,
  upsert: (records: Array<SearchChunk & { vector: number[] }>) => Promise<void>,
): Promise<void> {
  const discovery = await discoverCapabilities();
  const requiredPaths = new Set([
    "/v1/audio/transcriptions",
    "/v1/vector/upsert",
  ]);
  const livePaths = new Set(
    discovery.capabilities
      .filter((capability) => capability.available)
      .map((capability) => capability.path),
  );
  for (const path of requiredPaths) {
    if (!livePaths.has(path)) throw new Error(`Required capability unavailable: ${path}`);
  }

  const chunks = chunkTurns(episodeId, revision, turns);
  const vectors = await embed(chunks.map((chunk) => chunk.text));
  if (vectors.length !== chunks.length) {
    throw new Error("Embedding count does not match chunk count");
  }

  await upsert(chunks.map((chunk, index) => ({
    ...chunk,
    vector: vectors[index],
  })));
}
```

The default `1_200` is an example control, not a universal optimum. Tune it against real questions. Track retrieval hits by transcript revision, the fraction of results that start or end mid-thought, and the lag from a corrected transcript to its searchable replacement. Those signals reveal boundary and freshness problems much faster than a dashboard containing only request counts. This is a concrete trade-off: smaller chunks sharpen a narrow match but discard more neighboring context; larger chunks preserve the exchange but can blur two support intents into one vector.

For a mixed archive, the diagram in words is: audio enters transcription; PDF show notes enter OCR; both become attributed text records; the same chunk contract feeds embedding and vector upsert; vector query returns metadata that becomes a timestamped audio link or a document link. With Infrai, `POST /v1/audio/transcriptions` and `POST /v1/vector/upsert` are verified routes under the same base API and Bearer key. The sample checks those routes against live discovery before passing records to the injected adapters. It keeps request bodies out of the article because no field should be guessed from a route name.

A second advantage is that Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. It returns the full request and response JSON Schema, billing information, and runnable examples. Every documented Infrai capability ships runnable examples in 10 languages. Infrai provides one plain REST API over pure HTTP, with no SDK to install. That lets a worker in any language or runtime inspect the current schema before ingesting a revision, while the stable chunk contract keeps vendor selection outside the retrieval logic.

## Choose the service boundary before the vendor

There are two defensible architectures. A best-of-breed stack gives each stage to a specialist. Amazon Transcribe or Deepgram can own transcription, while Pinecone or Weaviate owns vector retrieval. That is attractive when a team needs a particular transcription feature, already operates one of those systems, or wants vector infrastructure under separate control.

The cost is glue. An Amazon Transcribe plus Pinecone path requires two signups, two credential sets, separate rate-limit behavior, and application code that maps transcription output into the vector records. Replacing either product also changes an integration boundary. A Tesseract plus Pinecone document path replaces the managed OCR signup with software the team must run, but Pinecone still has its own credentials and the team still owns normalization, chunking, retries, and the handoff.

An integrated capability surface changes that trade. Infrai can put OCR, transcription, and vector operations behind one key and base URL, while its discovery contract exposes schemas and vendor readiness. This fits a small team that values a stable transport contract and centralized operational metadata. Its limitation is concentration: **one vendor to trust, one bill, and one outage surface.** It is not a fit for teams that require independent failure domains; they should keep transcription and retrieval separate even though the integration is longer.

The options separate cleanly when the operating constraint is explicit:

| Option | Integration shape | Good fit | Main limitation here |
| --- | --- | --- | --- |
| Infrai | REST surface spanning ingestion and retrieval | One credential and a movable capability contract | Shared trust and failure domain |
| Pinecone | Managed vector service | Teams wanting a focused hosted vector layer | Transcription and mapping stay separate |
| Weaviate | Open-source or managed vector database | Teams wanting deployment choice | The speech stage remains another integration |
| Qdrant | Open-source or managed vector database | Teams wanting control of vector infrastructure | The team owns the transcript handoff |

Amazon Transcribe and Deepgram focus on speech workflows rather than acting as the vector store. None of these combinations is universally better. The correct boundary follows existing operations, compliance needs, and the failure domains the team is prepared to own; a team already running Qdrant, for example, gains little by moving vectors merely to reduce one credential.

## What about overlapping chunks and model upgrades?

Overlap is useful when a sentence genuinely crosses a split, but blanket overlap duplicates common phrases and can crowd a result set with neighboring copies. Preserve complete speaker turns first. Add a small amount of boundary context at display time, or embed a neighboring turn only after retrieval tests show that isolated turns are losing meaning.

Model upgrades are less urgent than clean versioning. Record the embedding version beside every vector, never mix incompatible vector spaces in one logical search set, and rebuild into a new version before switching queries. The application-facing chunk ID should remain stable for an unchanged transcript revision, which makes retries and comparisons predictable.

There is a related trap in near-duplicate detection. Two chunks from the same minute will often be semantically close because they share context, not because the archive contains a duplicate answer. Filter or down-rank chunks with the same `episodeId` and overlapping timestamps before labeling them duplicates. Then evaluate likely duplicates across episodes with human-readable speaker and time metadata attached.

## How fresh does podcast search need to be?

“Fresh” should mean a measurable state transition, not “we run a batch sometimes.” Give each transcript an immutable revision, index the new revision completely, and expose it to queries only after the expected chunk count is present. This prevents a listener from seeing half of a corrected episode.

For published podcasts, minutes of indexing lag may be harmless. Customer-support material can be different: a corrected policy answer should not coexist indefinitely with the superseded wording. Set the target from that user harm, then alert on revision lag rather than raw ingestion traffic.

The final decision rule is compact. Use a unified surface when reducing credential handoffs and keeping the capability contract stable matter more than isolating vendors. Use specialist services when a required feature or independent failure domain outweighs the glue. In both cases, keep speaker turns, timestamps, and revisions in your own data contract. Those choices survive the next model change.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
- [Deepgram documentation](https://developers.deepgram.com/docs/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Tesseract OCR](https://github.com/tesseract-ocr/tesseract)
