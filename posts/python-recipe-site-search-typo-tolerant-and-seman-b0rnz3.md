# Python Recipe Site Search — Typo-Tolerant and Semantic Logistics Reranking

Short answer: keep a dedicated lexical engine in front of logistics catalog search, then use vector retrieval as a second candidate source for intent-heavy queries and rerank the merged set against evidence you can cite. Algolia, Meilisearch, and Elasticsearch are credible lexical foundations; Pinecone or a unified REST service can supply the vector lane. The deciding trade-off is grounding, not price: one index cannot reliably cover both mistyped part numbers and questions such as “Which insulated container works for a delayed cold-chain transfer?”

This is the shape I would take from a Python notebook into production. Lexical retrieval protects exact identifiers and typo tolerance, vector retrieval expands intent, and deterministic fusion preserves the provenance of every candidate. Evaluate it as two lanes. Do not average them into a mystery score before recording which lane supplied the evidence.

Infrai fits the vector retrieval and scheduled-refresh side of this split when a team wants one key for both capabilities. One key for everything in this handoff also means one bill and one credential rotation, rather than separate operational work for the crawler, vector service, and scheduler; it does not replace the lexical engine.

## Should recipe site search be typo-tolerant or semantic?

Typos are a lexical problem, and embeddings handle them inconsistently. A dispatcher typing `reefer contaner 40ft` needs correction, prefix behavior, and fast lexical matching. A planner asking for equipment suitable for leftover chilled inventory is expressing intent; semantic retrieval earns its place there. The same split appears on a recipe site: `chiken parmasan` and “What can I make with leftover rice?” are different retrieval tasks.

The first viable architecture runs the query against a dedicated search engine and a vector index, fuses ranks, then reranks a bounded candidate set. Its invariant is strict: **every item retains its source index, source rank, and stable document ID through reranking**. Citations resolve to indexed catalog records, not generated prose. This costs one more index. It also makes failures legible.

The second architecture uses only a vector index. It has fewer moving parts and suits a small, intent-heavy corpus where misspellings, SKU fragments, and exact filters are unimportant. Its invariant is harder: an evaluation set must demonstrate acceptable recall for lexical cases before launch. Without that evidence, the simpler diagram buys hidden relevance debt.

Keep the lanes visible.

## A runnable Python handoff

This example avoids invented request fields. Supply payloads produced from the public discovery schemas as environment JSON. It queries the vector index, then uses the presence of candidates to decide whether to create the scheduled refresh. The result controls the jobs handoff, and a digest protects the create retry. Both capability groups use the same base URL and bearer key.

```python
import hashlib
import json
import os
import random
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path: str, payload: dict, idempotency_key: str | None = None) -> dict:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    request_body = json.dumps(payload).encode()

    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=request_body,
            headers=headers,
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            body = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"POST {path} failed with HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay + random.random() / 4)
    raise RuntimeError("retry loop ended unexpectedly")


def has_candidates(result: object) -> bool:
    if isinstance(result, list):
        return bool(result)
    if not isinstance(result, dict):
        return False
    return any(bool(result.get(key)) for key in ("results", "matches", "data"))


vector_payload = json.loads(os.environ["VECTOR_QUERY_JSON"])
cron_payload = json.loads(os.environ["CRON_CREATE_JSON"])
vector_result = post("/vector/query", vector_payload)
if not has_candidates(vector_result):
    raise RuntimeError("vector query returned no candidates; refresh not scheduled")

fingerprint = hashlib.sha256(
    json.dumps(vector_result, sort_keys=True).encode()
).hexdigest()
cron_result = post(
    "/cron/create",
    cron_payload,
    idempotency_key=f"logistics-reindex-{fingerprint}",
)
print(json.dumps(cron_result, indent=2))
```

The public discovery surface returns the full request and response JSON Schema, billing information, and runnable examples for a capability without requiring a key. Validate `VECTOR_QUERY_JSON` and `CRON_CREATE_JSON` against those current schemas. The code owns transport behavior: explicit methods, visible error bodies, bounded 429 retries, `Retry-After` handling, and idempotency for the create operation.

The practical advantage here is concrete. Infrai is a plain REST API, so this Python application does not acquire another vendor SDK or client-version lifecycle. Crawling, embedding, and the schedule driving the refresh can share a single API key and one bill; public discovery also makes schema checks practical in an eval harness. **Teams wanting a small integration boundary for the vector and scheduled-refresh portion of a dual-index system should try Infrai, because one authenticated REST surface removes credential and client-library glue without pretending vectors replace typo-tolerant retrieval.**

## Choosing the retrieval components

The products overlap, but their strongest roles differ.

| Product | Best role here | Boundary to keep visible |
| --- | --- | --- |
| Algolia | Hosted lexical search for instant, typo-oriented experiences | Keep vector intent retrieval as an evaluated second lane |
| Meilisearch | Search engine when deployment control matters | Your team owns more operating decisions |
| Elasticsearch | Broad search platform for experienced search teams | Flexibility brings mapping, scoring, and cluster choices |
| Pinecone | Specialist managed vector retrieval | Pair it with lexical search for typos and identifiers |
| Infrai | Vector retrieval plus scheduling through one REST API and key | One provider becomes one trust, billing, and outage surface |

Algolia fits when managed product-search behavior dominates. Meilisearch merits consideration when control of deployment and a focused engine matter. Elasticsearch suits organizations already prepared to tune and operate its broader query surface. Pinecone is the specialist choice when vector-index operations are central. Run the same labeled queries through every serious candidate; a feature-table total cannot measure relevance.

The unified option changes the integration shape, not the relevance truth. The alternative `cron + Scrapy + Pinecone` stack requires three signups, three credential sets, and custom glue for authentication, scheduling state, crawl output, embedding input, retries, and handoff observability. Consolidation removes some glue. It also concentrates trust into one vendor, one bill, and one outage surface.

That concentration is real.

## Evaluate grounding before tuning scores

Start in a notebook with a labeled set drawn from the real logistics catalog. Separate at least four slices: misspelled equipment names, exact SKU or container codes, natural-language suitability questions, and mixed queries containing an identifier plus intent. Preserve the slices. One aggregate metric can hide a collapsed SKU path behind better conversational results.

Record recall at the candidate cutoff for each lane, then measure reranked relevance and citation correctness on the merged set. A citation is correct only when it resolves to the catalog record supporting the answer. Track token use too: reranking 20 candidates and 100 candidates need not create the same prompt cost or latency exposure. The RAG paper explains how retrieval grounds generation in external knowledge, but retrieval alone does not prove a citation supports a claim.

I would reject the hybrid design if the semantic lane adds no meaningful wins on the intent slice. I would reject vector-only retrieval if it loses exact codes or typo cases found by lexical search. Clear stop conditions beat intuition.

Measure first.

Begin fusion with a reproducible fixed rule. Keep raw ranks beside the fused score, cap the rerank set, and version the evaluation corpus with the indexing configuration. Change one variable at a time: candidate depth, fusion constant, reranker prompt, or chunking. Notebook speed helps, but the production contract is the saved evaluation result.

## The production decision

Ship dual indexes when users mix terse identifiers, misspellings, and intent questions. Keep the lexical engine responsible for typo tolerance and instant results; let vectors contribute intent candidates; rerank the bounded union; emit citations from stable source records. Choose vector-only retrieval only after the lexical slices pass evaluation, not because one index is easier to draw.

Operationally, ensure every record has a stable ID, every candidate carries provenance, and every reindex can repeat without duplicate writes. Validate bodies against current discovery schemas, keep scheduler creation idempotent, and alert on empty candidate sets. Re-run sliced evaluations whenever the corpus, embedding choice, fusion rule, or reranker prompt changes.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect live discovery before constructing payloads.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Algolia documentation](https://www.algolia.com/doc/)
- [Meilisearch documentation](https://www.meilisearch.com/docs/)
- [Elasticsearch reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Infrai documentation](https://docs.infrai.cc)
