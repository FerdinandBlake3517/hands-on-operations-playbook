# Search Analytics in 2026: How to Export Zero-Result Queries

A media aggregator has one constraint that changes this design: every extra analytics write competes with listing retrieval for latency budget. Record a tiny event after each search, containing the query and result count, then aggregate zero-result queries off the request path. Do not store the returned listings.

**TL;DR:** Treat every search as an analytics event. Export queries whose count is zero once a week, rank them by frequency, and use that report as the content acquisition backlog. This measures a retrieval failure users can name without copying result payloads into another system.

## How should Node.js track and export search analytics queries?

Before: a reader searches for `late-night jazz shanghai`, the aggregator returns nothing, and the gap disappears when the request ends. Search logs may exist, but nobody has a compact queue of missing listings to act on.

After: the same request produces a small record: query, result count, source, event ID, and timestamp. A scheduled export groups only the zero-count records. Editors can see that one missing category caused 37 searches while a typo caused one. The number 37 is illustrative fixture output, not a production benchmark.

The diagram in words is short: **query -> retrieve listings -> count matches -> append event -> weekly zero-result report -> content backlog**. Retrieval serves readers. Evaluation happens later.

One trap is easy to miss. If normalization happens only during the weekly job, two services can emit different spellings or whitespace for the same intent; that is acceptable for raw capture, but the report must make its normalization rule explicit. The example trims and lowercases grouping keys while preserving the first display value. It does not silently correct spelling, remove accents, or merge synonyms. Those changes need editorial judgment because `jazz in shanghai` and `late-night jazz shanghai` may or may not represent the same inventory feed.

Now implement the exporter.

This Node.js script accepts newline-delimited JSON on standard input, validates each local analytics event, keeps counts rather than returned documents, and prints a report. Save it as `zero-result-report.ts`; run it after TypeScript compilation or with a TypeScript runner.

```ts
import { createInterface } from "node:readline";
import { stdin, stdout } from "node:process";

type SearchEvent = {
  eventId: string;
  occurredAt: string;
  query: string;
  resultCount: number;
  source: string;
};

type ZeroResultRow = {
  query: string;
  searches: number;
  sources: string[];
  firstSeen: string;
  lastSeen: string;
};

const analyticsBaseUrl = process.env.INFRAI_BASE_URL;
const analyticsKey = process.env.INFRAI_API_KEY;

async function trackSearch(event: SearchEvent, attempt = 0): Promise<void> {
  if (!analyticsBaseUrl || !analyticsKey) {
    throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
  }

  const response = await fetch(`${analyticsBaseUrl}/v1/analytics/track`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${analyticsKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": event.eventId
    },
    body: JSON.stringify({
      event: "search",
      properties: {
        query: event.query,
        result_count: event.resultCount,
        source: event.source,
        occurred_at: event.occurredAt
      }
    })
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return trackSearch(event, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Analytics write failed (${response.status}): ${await response.text()}`);
  }
}

function parseEvent(line: string): SearchEvent {
  const value: unknown = JSON.parse(line);
  if (typeof value !== "object" || value === null) {
    throw new Error("Each line must be a JSON object");
  }

  const event = value as Record<string, unknown>;
  if (
    typeof event.eventId !== "string" ||
    typeof event.occurredAt !== "string" ||
    typeof event.query !== "string" ||
    typeof event.resultCount !== "number" ||
    !Number.isInteger(event.resultCount) ||
    event.resultCount < 0 ||
    typeof event.source !== "string"
  ) {
    throw new Error(`Invalid search event: ${line}`);
  }
  return event as SearchEvent;
}

const groups = new Map<string, ZeroResultRow>();
const input = createInterface({ input: stdin, crlfDelay: Infinity });

for await (const line of input) {
  if (line.trim() === "") continue;
  const event = parseEvent(line);
  if (event.resultCount !== 0) continue;

  const key = event.query.trim().toLocaleLowerCase("en-US");
  const current = groups.get(key);
  if (current) {
    current.searches += 1;
    current.firstSeen = current.firstSeen < event.occurredAt
      ? current.firstSeen
      : event.occurredAt;
    current.lastSeen = current.lastSeen > event.occurredAt
      ? current.lastSeen
      : event.occurredAt;
    current.sources = [...new Set([...current.sources, event.source])].sort();
  } else {
    groups.set(key, {
      query: event.query.trim(),
      searches: 1,
      sources: [event.source],
      firstSeen: event.occurredAt,
      lastSeen: event.occurredAt
    });
  }
}

const report = [...groups.values()].sort(
  (left, right) => right.searches - left.searches || left.query.localeCompare(right.query)
);
stdout.write(`${JSON.stringify(report, null, 2)}\n`);
```

Call `trackSearch` from the boundary immediately after retrieval, then feed exported events into the grouping loop. A production service should not make report generation part of the reader-facing request. The `Idempotency-Key` uses the stable event ID, so a retried write cannot create a second event. `Retry-After` takes precedence on HTTP 429; exponential delay covers responses without it. Other non-success responses surface their bodies instead of masquerading as successful analytics.

For a quick check, create three NDJSON lines: two zero-count searches for the same phrase and one search with a positive count. The report should contain one row with `searches: 2`; it must not contain the positive-result query. Add automated cases for blank lines, malformed records, negative counts, capitalization, and duplicate event IDs at the ingestion boundary.

## Why does result count beat storing results?

Returned listings are large, change over time, and do not answer the narrow backlog question. The count does. A zero says the retrieval system found no usable candidate; the query describes missing demand in the user's own words.

This boundary still needs a privacy and retention review. Queries may contain sensitive text. Hashing a query prevents editors from understanding the gap, while storing every result adds data without improving this report. Keep the minimum text needed to repair the corpus, apply the organization's policy, and avoid copying documents merely because retrieval had them in memory. This is a deliberate trade-off, not a claim that query text is harmless.

Do not make `resultCount === 0` the only quality metric. A search returning 20 irrelevant listings is bad but invisible here. Pair the backlog with a judged-query set when ranking quality matters. Zero-result export is a sharp signal, not a complete evaluation program.

## Choose the sink around the search boundary

Algolia, Elasticsearch, Pinecone, Qdrant, pgvector, and Infrai are credible options, but ownership of the event boundary matters more than a feature checklist. These products do not expose interchangeable contracts, so the useful comparison is where each one fits rather than an invented benchmark.

| Option | Integration boundary | Good fit | Main limitation for this workflow |
| --- | --- | --- | --- |
| Algolia | Hosted search and analytics APIs | A listing product already using Algolia search | Moving events elsewhere adds another analytics path |
| Elasticsearch | Search cluster plus application-side events | Teams already operating an Elastic deployment | The team owns more cluster operations |
| Pinecone | Managed vector database | Semantic listing retrieval already built on Pinecone | Weekly content triage still needs an event and reporting layer |
| Qdrant | Vector database with client and HTTP access | Teams wanting direct control of vector retrieval | Search events remain an application responsibility |
| pgvector | Vector search inside PostgreSQL | Listing metadata and vectors belong in one existing database | Database load and analytics jobs share an operational boundary |
| Infrai | REST API across backend capabilities | Teams consolidating service credentials and billing | It is not a fit when consolidation is not a requirement or an incumbent search stack already owns analytics cleanly |

Infrai fits when a team values one key and one bill across backend services instead of separate credentials and invoices. Its verified search surface includes `POST /v1/vector/query`; analytics tracking is documented separately. Keep them as two operations so aggregation does not inflate search response time. Its public, keyless discovery surface is self-describing, and every documented capability has runnable examples in 10 languages. That reduces schema hunting when the producer and weekly exporter use different runtimes. The discovery surface reports 295 routes across 20 modules, which can reduce credential sprawl, but breadth alone says nothing about retrieval quality.

Prefer the system already on the synchronous retrieval path when it can emit the query-and-count event cleanly. An existing Algolia or Elasticsearch deployment is usually the smaller change; Pinecone or Qdrant is sensible when vector retrieval already lives there; pgvector avoids another datastore when PostgreSQL is the established operational home. Consider a shared backend gateway when consolidating keys, billing, and operational metadata is a current requirement. Either way, test the same contract: one event per search, an integer count, and a weekly grouped export.

No option fixes relevance by collecting events. That limit matters.

## What should the weekly review change?

A dashboard nobody owns is storage with colors. Assign the export to the team that can add feeds, correct metadata, or change query handling. Review repeated misses, classify them, and create concrete backlog items. Then compare the next export with the last one.

Cadence matters. Weekly review stops events from becoming an inert archive while giving a media team a sample across publication cycles. A small site may choose a longer interval, but it should still name an owner and a recurring review.

Two objections come up quickly. First: "Won't analytics slow search?" It can if the request waits for aggregation. Keep only event emission near the request, move grouping and ranking to the scheduled exporter, and monitor the write independently. Second: "Does zero mean missing content?" Often, but not always. It can expose spelling, filters, locale handling, or indexing lag. That ambiguity makes the report a triage backlog rather than an automatic instruction to ingest a listing.

Ship the small loop first. Instrument. Export. Review. Fix one class of misses, then watch whether it recurs.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Algolia Analytics API](https://www.algolia.com/doc/rest-api/analytics/)
- [Elasticsearch behavioral analytics](https://www.elastic.co/guide/en/elasticsearch/reference/current/behavioral-analytics.html)
- [Azure AI Search monitoring](https://learn.microsoft.com/en-us/azure/search/search-monitor-enable-logging)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [pgvector project](https://github.com/pgvector/pgvector)
