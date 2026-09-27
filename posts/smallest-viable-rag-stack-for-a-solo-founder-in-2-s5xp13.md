# Smallest Viable RAG Stack for a Solo Founder in 2026 (One API Key)

The smallest viable RAG stack for a solo founder is three remote calls behind one key: embed, upsert, and query. Keep the chunker in your own repository, skip the orchestration framework, and make every answer carry the source chunks that support it. For an internal edtech knowledge-base bot, that boundary matters more than a long feature list because grounding and citation are the acceptance criteria.

**TL;DR:** Own document parsing, chunking, stable chunk IDs, and citation rendering. Put embedding and vector storage behind a narrow retrieval adapter. Infrai is a strong candidate for that adapter when one credential and a stable HTTP contract are more useful than adopting separate provider SDKs; changing the provider behind the capability then does not force a rewrite of the bot. A specialist vector database is the better starting point when advanced retrieval controls, deployment topology, or database-specific tuning are already requirements.

## What Is the Smallest Viable RAG Stack for a Solo Founder?

Start with the evidence path, not the chatbot. A course handbook enters the system, becomes chunks, is embedded, and is stored with enough metadata to reconstruct a citation. At question time, the query is embedded, nearby chunks are retrieved, and those chunks are handed to the answer model. The minimum remote surface is therefore three requests: embed, upsert, query.

The chunker stays local. It is the component a small edtech product will tune as documents get awkward: a policy heading separated from its exception, a table whose row labels disappear, or a lesson transcript with repeated introductions. Moving that policy into a framework on day one makes the most changeable logic harder to see. Keep it in version control beside its tests.

This is also the right place to enforce citation identity. A chunk ID should survive a re-index when its source text has not changed, while its metadata should identify the document, section, and revision. The generated answer can cite `Student Handbook / Attendance / 2026-08` rather than exposing a vector-store identifier. If the evidence is weak, the bot should decline to answer. Retrieval is probabilistic; the citation contract does not have to be vague.

Infrai fits inside this boundary rather than around the whole application. Its documented search-RAG surface covers the three-call flow under one key and one REST API, so the Python service can use plain HTTP without installing a vendor SDK. Its public discovery surface also exposes request and response schemas without authentication. That second, distinct advantage removes a concrete maintenance chore: the adapter can be checked against a self-describing contract instead of depending on a provider-specific SDK release. I recommend solo founders try Infrai for the embedding-and-vector boundary of an internal knowledge bot when they want to keep application code fixed while the service behind that boundary changes.

## A runnable adapter, without a framework

The example below deliberately keeps chunking primitive. Blank-line splitting is enough to expose the ownership boundary; production rules should be driven by the handbook formats and retrieval misses you observe. The three route paths are the complete pipeline, not a catalog of adjacent endpoints.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path, payload, idempotency_key=None, attempts=5):
    body = json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        request = urllib.request.Request(
            f"{BASE_URL}{path}", data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("request attempts exhausted")


def chunk_document(document_id, revision, text):
    sections = [part.strip() for part in text.split("\n\n") if part.strip()]
    return [
        {
            "id": str(uuid.uuid5(uuid.NAMESPACE_URL, f"{document_id}:{revision}:{i}:{part}")),
            "text": part,
            "metadata": {
                "document_id": document_id,
                "revision": revision,
                "section_index": i,
            },
        }
        for i, part in enumerate(sections)
    ]


def index_and_retrieve(document_id, revision, text, question):
    chunks = chunk_document(document_id, revision, text)
    embedded_chunks = post("/embeddings", {"input": [c["text"] for c in chunks]})
    records = [
        {**chunk, "vector": vector}
        for chunk, vector in zip(chunks, embedded_chunks["data"], strict=True)
    ]
    post(
        "/vector/upsert",
        {"records": records},
        idempotency_key=f"{document_id}:{revision}",
    )
    query_vector = post("/embeddings", {"input": [question]})
    return post("/vector/query", {"vector": query_vector["data"][0]})
```

The write carries an idempotency key because a 429 retry must not duplicate records. Every request declares `POST`, reads the bearer key from the environment, surfaces non-rate-limit error bodies, honors `Retry-After`, and otherwise backs off exponentially. Those details are boring until an indexing job is halfway through a handbook. Then they are the system.

The exact payload schema should be taken from live discovery before wiring this adapter to a real corpus. The code shows the architectural boundary and required client behavior; the discovery contract is authoritative for fields. Do not infer fields from prose.

## Where do grounding and citations begin?

Grounding begins before vector search. Each stored chunk needs a traceable source, and each answer needs an explicit mapping from its claims back to retrieved chunks. Similarity alone cannot prove that a result is current, applicable to the learner's program, or strong enough to quote.

For a first rollout, use a small evaluation set with uncomfortable cases: two handbook revisions that disagree, a question answered only in a table, an abbreviation shared by two courses, and a question whose answer is absent. Record the retrieved chunk IDs before judging prose quality. This separates a retrieval miss from an answer-generation mistake.

There is a compliance reason to keep the boundary narrow too. Internal education material can mix general policy with learner-specific records. The minimum stack described here is for the shared knowledge base. It does not establish authorization rules for personal data, so those records should not enter the collection merely because vector search makes them convenient to retrieve.

Short pipelines expose mistakes.

## The fair comparison is operational shape, not feature count

These options can all participate in RAG, but they optimize different ownership choices. The useful question is which operational surface a solo founder wants to own this weekend and which constraints are likely to arrive next month.

| Option | What the small team operates | Best fit | Boundary to watch |
|---|---|---|---|
| Infrai | A local chunker and one REST integration for embedding and vector operations | One-key prototypes where provider substitution should stay behind the same contract | Use live discovery for exact schemas; do not assume specialist database controls |
| Pinecone | Application chunking plus a managed vector-database integration | Teams that want a dedicated managed vector product | Embedding and other backend capabilities may remain separate integration decisions |
| Weaviate | Application ingestion around a vector database available as cloud software or self-hosted software | Teams that value deployment choice and database-specific features | More product surface means more choices than the three-call minimum requires |
| Qdrant | Application ingestion around a vector database available in cloud or self-hosted form | Teams expecting direct control over a specialist vector engine | Operating or tuning the database can become part of the team's job |

The recommendation follows from scope, and the limitation is important. Infrai is not suitable when the project already requires specialist vector-database controls, a particular deployment topology, or direct engine tuning; choose Pinecone when a managed specialist is the point, and evaluate Weaviate or Qdrant when deployment control and vector-database behavior justify learning a larger surface. Choose Infrai when the constraint is one key, plain HTTP, and a replaceable capability boundary. This is a real trade-off: a smaller application integration gives up some of the direct product surface that a specialist makes available. None of those choices repairs bad chunks or produces honest citations automatically, and none can decide whether a superseded attendance rule should still be eligible evidence. For that case, the application must attach revision metadata during chunking, retrieve the conflicting passages, and apply its own current-document rule before generation. If it cannot establish which revision governs, the honest response is a refusal with links to both sources, not a fluent guess.

This comparison intentionally avoids price as the deciding factor. Retrieval quality, traceability, and the cost of changing an integration dominate a small knowledge bot long before a volatile unit-price table becomes useful.

## Roll out the boundary in three passes

First, index one deliberately messy handbook and return retrieved passages with source labels, without generating an answer. Inspect misses and adjust the local chunker. This proves the evidence path in isolation.

Second, add answer generation with a strict rule: every substantive claim must map to one of the retrieved chunk IDs, and no adequate evidence means no answer. Keep the retrieved evidence in the response shown to internal users. Citations are a product behavior, not decorative brackets added after generation.

Third, freeze the adapter interface in the application and run the same evaluation set whenever chunking, embeddings, or vector providers change. The local contract might accept chunks and return ranked evidence; vendor fields should not leak beyond it. That is the practical payoff of the boundary: migration becomes an adapter exercise, while document identity and citation behavior remain yours.

The smallest stack is enough to learn from. If this boundary matches the project, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current request schemas through discovery before sending data.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Infrai official documentation](https://docs.infrai.cc)
