# Concepts: Vector Databases

Once you have vectors, you need somewhere to store them and search them fast. That is a vector database. This maps to the schema and queries you write in Project 7.

---

## 1. What a vector database does

A vector database stores your chunk vectors and answers one core question quickly: **"given this query vector, which stored vectors are closest to it?"** (nearest-neighbour search). For a handful of chunks you could scan them all and compute every distance. For millions of chunks, scanning everything per query is too slow, so vector databases add **indexes** that make the search approximate but fast.

---

## 2. PostgreSQL + pgvector (your choice)

**pgvector** is a PostgreSQL extension that adds:

- A `vector(N)` column type - stores an N-dimensional vector in a normal table column.
- Distance operators - `<=>` (cosine), `<->` (L2), `<#>` (inner product).
- Vector indexes - HNSW and IVFFlat (below).

Why this is the pragmatic enterprise choice:

- **One database.** Your documents, their metadata, your access rules, and their vectors all live in Postgres. Retrieval, metadata filtering, and access control are all just SQL. No second system to run, secure, and back up.
- **You already know it.** Transactions, `WHERE`, `JOIN`, backups, roles - all the Postgres you know still applies.
- **Good enough at scale.** pgvector comfortably handles millions of vectors, which covers the vast majority of enterprise knowledge bases.

A chunk row looks like this:

```
id | doc_id | chunk_text | embedding vector(384) | source | page | access_level | lang | embed_model
```

The `embedding` column is the vector; every other column is metadata you filter and cite on.

---

## 3. Managed and dedicated alternatives (know them, for the client conversation)

You should be able to name the alternatives and say when they matter:

- **Pinecone, Weaviate, Qdrant, Milvus** - dedicated vector databases. Faster at extreme scale (hundreds of millions to billions of vectors), more vector-specific features, but a separate system to operate and pay for.
- **Managed Postgres with pgvector** (AWS RDS/Aurora, Supabase, Neon) - pgvector without running the server yourself.

The consultant's line: *"Start on Postgres + pgvector. Move to a dedicated vector DB only when scale or latency proves you need it."* Do not add operational complexity you have not earned.

---

## 4. Indexing: HNSW vs IVFFlat

Without an index, pgvector does an **exact** search - it computes the distance to every row. Correct, but O(rows) per query. With an index, it does an **approximate** search - much faster, occasionally missing a true nearest neighbour.

- **HNSW** (Hierarchical Navigable Small World) - a graph index. Best recall/speed trade-off, higher memory, slower to build. The default choice for most workloads.
- **IVFFlat** - clusters vectors into lists and only searches the nearest lists. Faster to build, less memory, needs the data present before you build it (it learns clusters from the data).

For Project 7's small dataset you can skip the index (exact search is instant on a few dozen chunks) and add HNSW when the data grows. The concept to carry: **an index trades a little accuracy for a lot of speed, and you tune how much.**

---

## 5. Metadata filtering

The power of Postgres for RAG: you can combine vector search with an ordinary `WHERE` clause.

```sql
SELECT chunk_text, source, page
FROM chunks
WHERE lang = 'en' AND access_level <= :user_level
ORDER BY embedding <=> :query_vector
LIMIT 5;
```

That single query does three jobs at once: semantic search (`ORDER BY <=>`), language filtering (`lang = 'en'`), and access control (`access_level <= :user_level`). In a dedicated vector DB you would wire metadata filters through a separate API; in Postgres it is just SQL you already know.

---

## 6. Access-control filtering - the enterprise non-negotiable

This is the most important idea in the tier. In an enterprise, **not every user may see every document.** An intern must not retrieve a chunk from a board-level compensation memo. If your retrieval query does not filter by the user's permission, the LLM will happily quote a secret document in its answer - a data breach.

The rule: **access control is part of the retrieval query, enforced at the database, not filtered out afterward in application code.** If you fetch everything and then try to hide forbidden results in Python, one missed code path leaks data. If the `WHERE access_level <= :user_level` (or a per-document permission join) is in the SQL, the forbidden rows never leave the database.

Two common models you will implement:

- **Level-based** - each chunk has an `access_level` integer; each user has a clearance; retrieve only `access_level <= user_level`.
- **Document-level ACL** - a join table of `(doc_id, allowed_role)`; retrieve only chunks whose document the user's role is granted.

A subtle, dangerous bug: filtering the *documents* but not the *chunks*, or applying the filter to the display but not the retrieval. Your SURVIVE `access-control-bypass` scenario injects exactly this bug; your job is to catch it and fix it so the forbidden chunk can never be retrieved.

---

## 7. Hybrid search: dense + sparse

- **Dense** retrieval = vector/embedding search. Great at meaning, weak at exact tokens (a specific error code `ORA-00942`, a part number).
- **Sparse** retrieval = keyword search (e.g. Postgres full-text search / BM25-style ranking). Great at exact tokens, weak at meaning.

**Hybrid search** runs both and combines the scores, so you get semantic recall *and* exact-term precision. You build this in USE exercise 2 and measure that it improves retrieval over dense alone. In Postgres, dense is pgvector and sparse is `tsvector` / `ts_rank`, combined with a scoring formula - all in one query.

Next: [04-document-processing.md](04-document-processing.md).
