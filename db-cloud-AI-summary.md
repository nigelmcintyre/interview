## DATABASES

### 1. SQL vs NoSQL

**What to say:** *"SQL/relational — Postgres, SQL Server — gives you fixed schema, joins, ACID transactions, strong consistency. Best when integrity matters, which for a fund-services or trade-ops platform, it usually does. NoSQL — document, key-value, column, graph stores — trades that for flexible schema and easier horizontal scaling, often with eventual consistency. The nuance that scores points: Postgres blurs the line — JSONB gives you schemaless document storage inside a relational, ACID database, so you often don't need a separate NoSQL store at all."*

**Real example:** *"My day job is literally both worlds — relational Postgres via Django, and a versioned document datastore for legislative assets, accessed through an internal ORM-like layer. A 300-page bill with full revision history isn't a row, it's a document — right store for the shape of the data. And JSONB specifically: SaunaGuide stores flexible per-listing attributes in a JSONB column."* (Careful: don't call SaunaGuide's model "config-driven by JSONB" — the filter config is a separate Python module; JSONB is just the flexible attribute storage.)

**Likely pushback:** *"If Postgres JSONB can do flexible/schemaless storage, why would you ever reach for a dedicated NoSQL store like MongoDB?"*
**Answer:** *"Scale and access pattern. If you need massive horizontal write throughput, or your whole data model is genuinely document-shaped with no relational structure anywhere, a dedicated document store is built for that from the ground up. Postgres JSONB is great when most of your data is relational and only a slice of it is flexible — you get schemaless storage without giving up ACID and joins for everything else."*

---

### 2. EXPLAIN ANALYZE ⭐ MUST HAVE

**What to say:** *"`EXPLAIN` shows the planner's predicted execution plan. `EXPLAIN ANALYZE` actually runs the query and shows real timings and row counts. I'm looking for a Seq Scan on a large table where I'm filtering — that means a missing index. An Index Scan is what I want. I'd also check estimated vs actual row counts — if they're wildly different, statistics are stale and I'd run `ANALYZE`. Adding `BUFFERS` shows cache hits vs disk reads too."*

**Real example:** *"In DocIntel, I ran EXPLAIN ANALYZE on a full-text search query and saw a sequential scan over the documents table. I added a GIN index on the tsvector column, re-ran it, and it flipped to an index scan — seconds to milliseconds."* (Make this true before the interview — actually run it.)

**Likely pushback:** *"You said the estimated and actual row counts were different — why does that matter if the query still ran correctly?"*
**Answer:** *"The planner uses those estimates to choose the execution strategy — which join type, whether to use an index at all. If the estimates are badly wrong, it can pick a bad plan even though the query is logically correct — like choosing a nested loop join when a hash join would be far faster for the actual data volume. Stale statistics after a big data change is the usual cause; ANALYZE refreshes them."*

**Likely pushback:** *"What's the difference between a B-tree index helping and not helping here?"*
**Answer:** *"A B-tree supports equality and range lookups efficiently by walking down a sorted tree structure. It doesn't help with something like `ILIKE '%term%'`, because there's no left-anchored prefix to search from — every row still has to be checked, so it seq-scans regardless of the index. That's why full-text search needs a GIN index, not a B-tree."*

---

### 3. Indices and their types

**What to say:** *"B-tree is the default — equality and range, and it maintains order. GIN is for values with many items inside them — full-text tsvector, JSONB, arrays. GiST is for geometric/spatial data. BRIN is for huge, naturally-ordered tables like append-only timestamps — tiny index footprint. Partial indexes only cover rows matching a WHERE clause, so they're smaller and faster for a common filtered query. Composite indexes follow the leftmost-prefix rule — an index on (a, b) helps queries filtering on a, or a and b together, but not b alone."*

**Real example:** *"SaunaGuide has a real composite index — `Index(fields=['-is_featured', 'name'])` — featured listings first, then alphabetical, plus a separate single-column index on county since that's the main filter path. GIN indexes on tsvector and JSONB columns are worth studying as patterns even if they're not in my own projects yet."*

**Likely pushback:** *"Why not just index every column that gets filtered on?"*
**Answer:** *"Every index slows down writes — inserts and updates have to maintain it — and takes disk space. If a table is write-heavy and a column is rarely filtered on, indexing it is pure cost with no benefit. I'd index based on actual query patterns, not defensively index everything."*

**Likely pushback:** *"Give me a concrete case where the leftmost-prefix rule actually bites someone."*
**Answer:** *"If I have a composite index on `(county, is_featured)` and a query filters only on `is_featured`, that index can't be used at all — the leftmost column has to be part of the filter for the index to help. So if I know both filter patterns matter separately, I'd either reorder the index or add a second one just for `is_featured`."*

---

### 4. Transactions, ACID & locking

**What to say:** *"ACID — atomic, all-or-nothing; consistent, constraints always hold; isolated, concurrent transactions don't see each other's partial work; durable, a commit survives a crash. In Django, `transaction.atomic()` wraps a block so it all commits or all rolls back together. For locking: pessimistic — `select_for_update()` — locks rows now, other writers block until commit, right for anything touching money or where conflicts are likely. Optimistic — a version column, checked in the WHERE clause on update — for when conflicts are rare, no blocking, just a conflict response if someone beat you to it."*

**Real example:** *"My day job runs real pessimistic locking — documents are checked out and checked in through the document service, held for the whole editing session, because two drafters silently merging edits to a bill is unacceptable. For fund administration specifically though, I'd lean optimistic for most record updates — conflicts on a given fund record are rare, so there's no reason to make users queue when they'll almost never collide."*

**Likely pushback:** *"What's a deadlock, and how do you avoid it?"*
**Answer:** *"Two transactions each hold a lock the other one wants — transaction A holds row 1 and wants row 2, transaction B holds row 2 and wants row 1, neither can proceed. Postgres detects this and kills one of them. The mitigation is acquiring locks in a consistent order everywhere in the codebase, and keeping transactions short so the window for this is small."*

**Likely pushback:** *"With optimistic locking, what happens to the second user's data when they get the conflict?"*
**Answer:** *"It shouldn't just be discarded — the front end should show a conflict UI: here's what changed, here's your edit, reload and re-apply, or auto-merge if the fields don't overlap. The whole point of optimistic locking is catching the conflict, not silently losing work either way."*

---

### 5. Read replicas, connection pooling, partitioning

**What to say:** *"Read replicas are copies following the primary via replication — route heavy reads and reports there, keep writes on the primary. The caveat to always name: replication lag — fine for a report, wrong for read-your-own-write flows. Connection pooling, like PgBouncer, multiplexes many short-lived app connections over a few real database connections, since each Postgres connection is an expensive process. Partitioning splits a huge table, usually by date, so queries scan less and old partitions can be dropped cheaply."*

**Real example:** conceptual unless Propylon runs a replica setup — this one's fine to keep at the concept level, it's already baked into your rehearsed big-report answer.

**Likely pushback:** *"How would a user actually experience replication lag as a bug?"*
**Answer:** *"They save something, get redirected to a page that reads from the replica, and don't see their own change yet because it hasn't replicated over. That's why writes and any 'read my own write' flow should always hit the primary — replicas are for things like dashboards and reports where a few seconds of staleness is fine."*

---

## CLOUD — EC2, ECS, RDS, load balancers, deploys, CI/CD

Be honest here — hands-on cloud is limited (Azure AD SSO at Irish Life; Docker/Ansible on Red Hat at Infobip). Know the concepts, bridge honestly.

### Core concepts

**What to say:** *"EC2 is a virtual machine — you manage the OS and runtime yourself, maximum control, maximum ops burden. ECS is container orchestration for Docker — with Fargate you don't manage servers at all. RDS is a managed relational database — AWS handles backups, patching, failover, replicas; Aurora is the higher-performance variant. Load balancers: ALB works at layer 7, HTTP, routing by path or host; NLB is layer 4, raw TCP, very high throughput. Deploys: rolling replaces instances gradually, blue-green stands up a whole new environment and switches traffic with easy rollback, canary sends a small percentage of traffic first."*

**Real example — the on-prem vs cloud bridge, reasoned live today:** *"At Propylon, we deploy RPMs to servers the Montana team already manages and pre-configures — Python, dependencies, system libraries are already installed, so we just deploy the app code. In AWS, you can't assume anything's pre-installed on the target server, so you containerize with Docker — bundle the app, the runtime, every dependency into one image — and any server can run it without pre-setup. Same CI/CD discipline either way: lint, test, build, deploy — I've done that with GitLab CI at Propylon, Jenkins at Infobip, TeamCity at Irish Life. The tools change, the discipline doesn't."*

**Likely pushback:** *"If containers are the modern standard, why does Propylon still deploy RPMs to pre-configured servers instead of containerizing?"*
**Answer:** *"A few likely reasons — it predates containerization being mainstream, and migrating a stable deployment pipeline has real risk for uncertain benefit. Containers add operational overhead — image registries, possibly orchestration — that's not worth it for a small, stable on-prem footprint. There's also debugging simplicity: SSH into a known server directly versus debugging inside a container layer. Containers aren't universally better — it's a tradeoff between portability/scalability and simplicity/control, and the right answer depends on your constraints, not on which is 'more modern.'"*

**Likely pushback:** *"What's the actual difference between ALB and NLB in practice — when would you pick one over the other?"*
**Answer:** *"ALB understands HTTP — it can route based on URL path or hostname, which is what you want for a typical web app with multiple services behind one load balancer. NLB just moves raw TCP packets at very high throughput and low latency, with no awareness of the content — you'd use it for something like a high-volume, low-latency protocol that isn't HTTP, or when you need to preserve the client's real IP without proxying."*

**Likely pushback:** *"Blue-green vs canary — what's the actual tradeoff?"*
**Answer:** *"Blue-green gives you an instant, clean rollback — flip traffic back to the old environment immediately if something's wrong — but you're running two full environments at once, which costs more and the switch is all-or-nothing. Canary catches problems with less blast radius, since only a small percentage of users see the new version first, but it's slower to fully roll out and you need good monitoring to actually detect a canary regression before it reaches everyone."*

**Likely pushback:** *"You mentioned RDS handles failover — what does that actually mean happens?"*
**Answer:** *"RDS runs a standby replica in a different availability zone. If the primary fails, RDS automatically promotes the standby to primary and redirects connections to it — typically within a minute or two. The application doesn't need custom failover logic; RDS handles the detection and switch."*

---

## AI — RAG pipeline, tokenisation

### RAG (Retrieval-Augmented Generation)

**What to say:** *"LLMs hallucinate and don't know your private documents — they're frozen at training time. RAG solves that by grounding the answer in your actual data: embed the user's question into a vector, run a similarity search over your document chunks to find the most relevant ones, feed those chunks into the LLM's prompt as context, and the model generates an answer from that context — ideally with citations back to the source documents. It's not fine-tuning the model, it's giving it the right information at query time."*

**Real example pattern:** *"If building a RAG system, the core decision is where to store embeddings — a dedicated vector database like Pinecone or Chroma, or inside your existing Postgres with pgvector. pgvector keeps retrieval inside the same database as your document metadata, no second system to sync and operate, same principle as reaching for Postgres full-text search before Elasticsearch. I understand the mechanics of this tradeoff even if I haven't built the full end-to-end pipeline yet."*

**Likely pushback:** *"Walk me through what happens end-to-end when a user asks a question."*
**Answer:** *"The question gets embedded into a vector using the same embedding model that indexed the documents. That vector is compared against the stored document-chunk vectors using a similarity metric — cosine similarity, typically — to find the closest matches. The top N chunks get pulled and inserted into the prompt alongside the user's question, with instructions to answer only from that context. The LLM generates the answer, and I'd return the source chunks alongside it so the user can verify — that's the citation piece, and it's what actually makes RAG trustworthy rather than just another way to hallucinate confidently."*

**Likely pushback:** *"What if the similarity search returns irrelevant chunks — how do you handle that?"*
**Answer:** *"A few levers: tune the number of chunks retrieved and a similarity-score threshold so weak matches get dropped rather than forced into the prompt. You can also ask the LLM to say 'I don't have enough information' rather than force an answer from bad context — that's part of the responsible-AI piece the spec asks about. And chunk size matters a lot here too small and you lose surrounding context, too large and irrelevant text dilutes the actually-relevant part."*

**Likely pushback:** *"Why not just put the entire document corpus in the prompt instead of doing retrieval?"*
**Answer:** *"Context window and cost — even large context windows have limits, and every token costs money and adds latency. Retrieval means you're only paying for and processing the handful of chunks actually relevant to this specific question, not the whole corpus every time. It also tends to produce better answers, because irrelevant context can actively distract the model from the right answer, not just sit there unused."*

---

### Tokenisation

**What to say:** *"LLMs don't process words or characters directly — they process tokens, which are subword units, roughly four characters each in English on average, produced by byte-pair encoding. That matters practically in three ways: the context window is measured in tokens, not words or characters, so 'how much can I fit in the prompt' is a token count; cost is billed per token, both input and output; and it drives your chunking strategy — how big each retrieved chunk is has to account for how many tokens it actually costs, not just how many words look reasonable."*

**Real example:** *"When designing document chunking for a RAG system, chunk size has to account for token costs, not just word count — you'd measure in tokens that the embedding model and LLM will actually consume. I understand this constraint even though I haven't built the full retrieval pipeline myself."*

**Likely pushback:** *"Why subword tokens instead of just whole words?"*
**Answer:** *"Whole-word tokenization would need an enormous vocabulary to cover every word, including rare ones, typos, and words in other languages — and it can't handle a word it's never seen. Subword tokens let the model represent any word by combining known pieces — so an unfamiliar word just becomes more tokens, rather than an unknown/broken token. It's a tradeoff between vocabulary size and sequence length."*

---

### The honest bridge (your prepared line — use it as-is)

*"I've studied RAG patterns and understand the retrieve-then-generate architecture conceptually. I built the API and database foundations for a document system using FastAPI and Postgres, so I'm comfortable with the infrastructure side — schema design, indexing strategy, query optimization. The RAG retrieval and synthesis piece specifically is still a gap I want to close hands-on, which is why I'm preparing for this role — it's exactly the kind of work I want to own."*

**Why this line matters:** the AI section in your prep doc is intentionally kept brief compared to Python/DB — this is a smaller, "nice to have" surface area on the Citco spec relative to backend fundamentals. Being honest about what you've built and what's next is better than overclaiming — it shows judgment and self-awareness. Stick to what you can defend, and frame the gap as genuine interest, not a weakness.
