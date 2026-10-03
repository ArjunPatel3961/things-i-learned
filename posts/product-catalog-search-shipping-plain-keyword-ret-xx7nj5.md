# Product Catalog Search: Shipping Plain Keyword Retrieval Before RAG

TL;DR: Ship fielded keyword retrieval first for a B2B SaaS catalog that aggregates listings from multiple sources. It is the smaller operational system, and it preserves exact identifiers, filters, and newly changed records without adding generation. Add semantic retrieval only after measured queries expose a vocabulary gap; add generation only when users need a synthesized answer rather than a list of current records. The decisive constraint is not language choice. It is whether chunking can retain listing boundaries while updates reach every retrieval representation inside a stated freshness budget.

A catalog is closer to an OTP directory than an essay collection: a near match can be wrong, and stale state can be more damaging than an empty result. Exact plan codes, regions, availability states, and compliance labels deserve field-aware treatment. Free text still matters, but it should not erase those boundaries.

## Should product catalog search use plain keyword retrieval or RAG?

Start with the response contract. A query such as `SOC 2 invoicing integration EU` usually asks for matching records that can be filtered, sorted, opened, and audited. Keyword retrieval fits that contract directly. An inverted index associates terms with documents; structured fields can participate in filters, and the application returns source records rather than prose about them. BM25 is a common ranking family for this job, but the important design property is simpler: retrieval stops at retrieval.

RAG has a broader contract. The original RAG paper describes combining parametric generation with retrieved non-parametric memory for knowledge-intensive language tasks. That architecture is useful when the required output is an answer assembled from evidence. It also creates more states to validate: the selected passages, the model input, the generated claim, and the citation-to-claim relationship. A product list does not gain value merely because a model can narrate it.

Use the narrowest contract that answers the query. For an initial release, make the result a ranked array of canonical listing IDs plus scores and matched fields. Keep filtering authoritative in the catalog store or search index. If a later feature needs a comparison summary, treat that as a separate presentation path fed by retrieved, access-checked records.

Return records first.

This separation has a compliance benefit. Entitlement filtering cannot be repaired after restricted text has already entered a generation prompt. Apply tenant, region, lifecycle, and visibility constraints before any candidate content crosses that boundary.

## Listing boundaries are the chunking strategy

A source page may contain twenty offers, repeated navigation, a vendor biography, and one legal disclaimer. Splitting that page every fixed number of tokens can join the end of one offer to the beginning of the next. Retrieval may then return a fluent fragment with the wrong price tier, region, or owner attached. The failure is subtle because every word existed in the source.

Consider one source that labels an integration “available in the EU” in a listing card, while a neighboring card says “US only.” A page-sized chunk can contain both phrases. A semantic match for an EU query may retrieve the chunk, yet the application no longer has a defensible way to attach the region phrase to the right canonical ID. Re-ranking cannot reconstruct a boundary discarded during ingestion. The repair belongs upstream: parse each card into a record, retain source coordinates for audit, reject records missing a stable source ID, and quarantine ambiguous updates rather than merging them into whichever title happens to be nearest. This is less glamorous than prompt tuning. It also prevents a class of confident, hard-to-detect catalog errors.

Make one canonical listing the default retrieval unit. Store identifiers and enumerated attributes as fields; keep descriptive prose in separate text fields. Long descriptions can be divided at semantic boundaries, but every child chunk must carry the canonical listing ID, source revision, visibility attributes, and an explicit field name. Do not copy a mutable status into anonymous text and hope ranking will reconcile it later.

A compact transformation can enforce those invariants before documents reach any index:

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Iterable


@dataclass(frozen=True)
class Listing:
    source: str
    source_id: str
    revision: str
    title: str
    summary: str
    regions: tuple[str, ...]
    status: str
    tenant_ids: tuple[str, ...]


def index_documents(item: Listing) -> Iterable[dict]:
    canonical_id = sha256(
        f"{item.source}:{item.source_id}".encode("utf-8")
    ).hexdigest()

    fields = {"title": item.title.strip(), "summary": item.summary.strip()}
    for field_name, text in fields.items():
        if not text:
            continue
        yield {
            "document_id": f"{canonical_id}:{field_name}",
            "canonical_id": canonical_id,
            "source_revision": item.revision,
            "field": field_name,
            "text": text,
            "regions": list(item.regions),
            "status": item.status,
            "tenant_ids": list(item.tenant_ids),
        }
```

The function deliberately does not split a summary by an arbitrary token count. If descriptions later exceed the retrieval system's practical document size, split within the `summary` field and append a stable chunk ordinal to `document_id`. Preserve the other metadata on every child. This is repetitive storage, but it makes deletion, authorization, and trace inspection tractable. The trade-off is worth stating plainly.

Do not embed the concatenation of every field as the only representation. Exact identifiers can be diluted by surrounding prose, and one changed status can force replacement of an otherwise unchanged vector. Fielded documents let the team choose which content participates in lexical matching, semantic matching, or neither.

## Freshness is an end-to-end budget

“Near real time” is not a testable requirement. Define a budget per change class instead. One reasonable project policy might require removals and visibility changes to disappear from candidate sets within 60 seconds, inventory-like status changes within 5 minutes, and descriptive edits within 30 minutes. Those are example service objectives, not universal thresholds. Select them from business harm and source capabilities.

Measure from the source revision becoming observable to the revision being queryable. That interval includes polling or webhook delay, normalization, deduplication, indexing, replica visibility, and cache invalidation. Reporting only indexing latency hides most of the path.

The clock starts upstream.

Freshness also needs a negative assertion. For each sampled update, query for the new revision and confirm that the superseded revision can no longer be returned. Deletions deserve their own queue and alert because a failed delete can survive indefinitely while ordinary upserts continue to look healthy. Keep a reconciliation scan as a backstop: compare source revision IDs with indexed revision IDs and repair drift idempotently.

This is where semantic retrieval costs operational attention even before generation appears. If lexical and vector indexes update through different pipelines, a query can see two versions of one listing. Use a shared revision marker, publish both representations from the same normalized event, and expose a record only under a defined consistency rule. For strict visibility changes, remove or suppress the record from all candidate paths before waiting for slower descriptive enrichment.

Caches need the same discipline. Cache keys should include tenant and filter context, while invalidation should follow canonical listing IDs rather than raw source URLs. A short time-to-live limits exposure but does not prove a freshness objective; direct invalidation plus a bounded fallback is easier to reason about.

## Choose the smallest retrieval stage that closes a measured gap

The comparison becomes useful after the constraints are explicit:

| Stage | Adds value when | Main new failure surface | Ship evidence |
| --- | --- | --- | --- |
| Fielded keyword retrieval | Queries contain names, codes, attributes, and filters | Analyzer mismatch, synonyms, stale fields | Relevance judgments and freshness checks meet the stated budgets |
| Hybrid lexical and semantic retrieval | Relevant listings use different vocabulary from queries | Duplicate candidates, vector lag, score fusion | A held-out query set shows material recall gains without unacceptable precision loss |
| Retrieval plus generation | Users need a sourced comparison or explanation | Unsupported claims, citation mismatch, prompt data exposure | Claim-level review and authorization tests pass for the answer path |

This is not a maturity ladder. Stop at the first row if it satisfies the product contract. A synonym map, spelling normalization, or curated category alias often closes a vocabulary gap with fewer moving parts than another retrieval representation. Conversely, paraphrased capability queries may justify semantic candidates even when no generated answer is wanted. RAG and vector search are related choices, not synonyms.

Sometimes one row is enough.

Build a judgment set from real query shapes without inventing relevance from click counts alone. Include exact IDs, acronyms, category phrases, cross-source duplicates, newly added listings, revoked listings, misspellings, and queries that should return nothing. For each query, record acceptable canonical IDs and required filters. Track retrieval quality separately for head and tail queries; one aggregate score can conceal a severe miss on low-volume compliance terms.

Operational checks belong in the same release gate. Log a trace ID, normalized query, applied filters, candidate canonical IDs, source revisions, retrieval stages, and final ranking. Avoid logging raw restricted descriptions when identifiers and revision metadata are enough. Rate-limit expensive semantic or generation paths independently, and define their timeout behavior. A keyword fallback is valid only if its different recall and ordering are visible in telemetry rather than silently presented as equivalent.

A useful decision record fits on one page: response contract, access boundary, listing and chunk schema, freshness objectives by change class, evaluation set version, rollback condition, and owner. Cost belongs there as a capacity constraint, but volatile unit prices should not drive the architecture. Candidate count, index growth, update amplification, cache hit rate, and generation volume are more stable inputs for load tests.

## Roll out without coupling ingestion to one ranking method

First, normalize every source into canonical listings and make upsert plus deletion idempotent. Release fielded keyword retrieval behind a stable search interface, then shadow any semantic candidate path against the same queries without changing user-visible results. Compare judged relevance, revision consistency, latency, and authorization outcomes.

Promote the extra stage only for query classes where it clears a predeclared threshold. Keep canonical IDs and revision markers in every response so rollback changes ranking behavior, not data identity. Generation, if the response contract eventually requires it, should consume the same authorized records through a separately monitored endpoint.

The simple choice is conditional but clear: begin with keyword search for listing discovery, preserve field and record boundaries, and spend the early engineering budget on freshness evidence. Add semantic retrieval for demonstrated language mismatch. Add generation for demonstrated answer synthesis. Each step must earn its own failure modes.

## Sources

- https://arxiv.org/abs/2005.11401
- https://lucene.apache.org/core/9_12_1/core/org/apache/lucene/search/similarities/BM25Similarity.html
- https://www.rfc-editor.org/rfc/rfc9110
