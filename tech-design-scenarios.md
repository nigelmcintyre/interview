## TECH DESIGN SCENARIOS

**Universal structure for all of these:** diagnose → options → tradeoff → what I'd pick. Name the Python/DB tool as you reach for it — "this is I/O-bound, so..." / "I'd EXPLAIN ANALYZE first..."

---

### 1. "We have a slow API request — what do you do?"

**What to say:** *"I measure first — profile, log, or APM — I don't guess. Then I match the fix to the actual cause: N+1 queries get `select_related`/`prefetch_related`, a missing index gets EXPLAIN ANALYZE and an index, over-fetching gets pagination or `.values()`, expensive computation moves to a background job, a slow external call gets cached, timed out, or made async."*

**Real example:** *"This is literally my Irish Life OLS story — I measured method execution times, found the service returning more fund-price data than needed, and switched to a leaner endpoint. And on SaunaGuide, two separate fixes: row-level, I added pagination because the homepage loaded every listing on every request; column-level, I switched to `.values()` for the map view because it was pulling a heavy base64 image blob per listing when I only needed lat/lng."*

**Likely pushback:** *"What if you fix the N+1 and it's still slow?"*
**Answer:** *"Then I re-measure — that's the whole point of measuring first instead of guessing. If it's still slow after the ORM fix, I go to the next likely cause: missing index, or the query itself doing more work than needed. I don't stack fixes without confirming each one actually moved the number."*

**Likely pushback:** *"How do you decide between caching and background jobs for a slow external call?"*
**Answer:** *"Depends on whether the result is reusable. If the same request repeats — same params, data doesn't change often — cache it. If it's a one-off action that just takes time, move it off the request path into a background job so the user isn't blocked."*

---

### 2. "We want a feature that generates a big report"

**What to say:** *"Never generate it inside the request — it'll block a worker and time out. Offload to Celery: the view enqueues and returns `202 Accepted` with a job ID immediately. Inside the worker, I stream with `.iterator(chunk_size=...)` so memory stays flat regardless of row count. The finished file goes to S3, keyed by the report params — that doubles as a cache, so identical requests return the existing file instead of regenerating. The client polls for status and gets a presigned URL when it's ready."*

**Real example:** *"The chunking pattern is the same one I'd use in DocIntel for any large export, and the pagination/column-selection instincts are straight from SaunaGuide."*

**Likely pushback:** *"Why not use WebSockets to notify the client the report is ready, instead of polling?"*
**Answer:** *"Polling every couple of seconds is honestly fine for job status — simple, works everywhere, no persistent connection to manage. I'd only reach for WebSockets if I needed true bidirectional communication. For a one-way 'is it done yet' check, polling or SSE is the simpler tool."*

**Likely pushback:** *"What stops the same user submitting the same report twice by accident?"*
**Answer:** *"I make the submit idempotent server-side — store the job's params in a DB row, and if a request comes in with matching params while a job's still in flight, return the existing job ID instead of enqueuing a duplicate. Disabling the button helps UX, but I never trust the client to enforce that alone."*

---

### 3. "We want real-time notifications / a live progress bar"

**What to say:** *"Match the tool to the need, lightest thing that works. Polling — simplest, works everywhere, more load, fine for a progress bar. Server-Sent Events — one-way server push over HTTP, great for notifications and progress. WebSockets, via Django Channels — full bidirectional persistent connection, only when I genuinely need two-way real-time, like chat or live collaboration."*

**Real example:** conceptual unless Propylon/Montana does live collaboration on documents — worth checking if the VSTO add-in or the datastore has any real-time sync mechanism you could point to.

**Likely pushback:** *"Why does WebSockets need async under the hood?"*
**Answer:** *"A WebSocket is a persistent open connection per client — you can't hold that open with a thread per connection at scale, that's expensive. An async event loop handles thousands of idle, waiting connections far more cheaply, because each one isn't consuming a dedicated OS thread while it's just sitting there waiting for the next message."*

**Likely pushback:** *"Would you ever use both polling and WebSockets in the same feature?"*
**Answer:** *"Sure — WebSocket as the primary channel, with a polling fallback if the connection drops or the client's on a network that blocks persistent connections. Graceful degradation rather than an all-or-nothing choice."*

---

### 4. "We'll process millions of rows a day in the DB"

**What to say:** *"Index the actual query paths, not everything — you're write-heavy here, and every index slows writes. Partition big tables by date, and use BRIN indexes for append-only, naturally time-ordered data. Bulk operations — `COPY` or `bulk_create` — never row-by-row inserts. Connection pooling so short-lived connections don't exhaust the DB. Read replicas for the analytics/reporting side. And I'd question the premise — does all of this actually need to live in the transactional database, or does analytics belong in a warehouse?"*

**Real example — front end half, your strongest angle here:** *"You never render millions of DOM nodes — you virtualize, render only what's visible, and paginate server-side so the client only ever fetches what's on screen. AG Grid's server-side row model does exactly this, and I've built with it in DocIntel."*

**Likely pushback:** *"Why BRIN over a regular B-tree index for time-series data?"*
**Answer:** *"BRIN is tiny compared to a B-tree because it stores value ranges per block instead of every row. It works well specifically because the data is naturally ordered by insertion time — append-only logs, timestamps. A B-tree would be far larger for the same query benefit on that shape of data."*

**Likely pushback:** *"Asking the premise question — 'should this even be in the transactional DB' — isn't that dodging the question?"*
**Answer:** *"No — it's often the actual right answer at that volume. A transactional database is optimized for consistency and small fast writes, not for scanning millions of rows for analytics. Separating operational data from analytical data, into a warehouse, is standard practice at that scale — naming that shows I'm not just reaching for more indexes as the only lever."*

---

### 5. "Two users edit the same record at the same time"

**What to say:** *"First question I'd ask: how likely is a conflict, and how bad is silently losing an edit? That decides the strategy. Optimistic locking — a version column, update checks `WHERE id=? AND version=?`, zero rows updated means someone else won, return a conflict — for when conflicts are rare. Pessimistic locking — `select_for_update()` inside a transaction, second writer blocks — for when conflicts are likely or the update must be serialized, like anything touching money."*

**Real example:** *"My day job runs real pessimistic locking — documents are checked out and checked in through the document service, held for the whole editing session, because two drafters silently merging edits to a bill is unacceptable. For fund administration specifically, I'd actually lean optimistic — conflicts on a given fund record are rare, so there's no reason to make users queue when they'll almost never collide."*

**Likely pushback:** *"What happens to the user's unsaved work when they hit a 409 conflict under optimistic locking?"*
**Answer:** *"The front end shouldn't just discard it — I'd show a conflict UI: here's what changed, here's your edit, either reload and re-apply, or in simple cases auto-merge non-overlapping fields. What you never do is silently overwrite — that's the whole risk optimistic locking is designed to catch, not create."*

**Likely pushback:** *"Could you combine both — pessimistic for some fields, optimistic for others?"*
**Answer:** *"In principle yes, if a record has both high-conflict fields (like a shared balance) and low-conflict fields (like a description) — but in practice that's added complexity most systems don't need. I'd only reach for that split if profiling showed real contention on a specific field, not as a default design."*

---

### 6. "We depend on a slow / flaky third-party API"

**What to say:** *"Timeouts, always, non-negotiable — an unbounded call eventually hangs every worker you have. Retries with exponential backoff and jitter, but only for idempotent calls — retrying a payment is how you double-charge someone. Circuit breaker — after N consecutive failures, stop calling for a cooldown and fail fast instead of queueing doomed calls. Move it off the request path into a background job where possible. And degrade gracefully — serve the last good cached response with a notice, rather than erroring the whole page."*

**Real example:** conceptual unless you've dealt with a flaky integration at Propylon (the Word add-in ↔ Django backend interactions could be one, if any of those calls are unreliable) — confirm before using.

**Likely pushback:** *"How do you decide the timeout value?"*
**Answer:** *"Based on the API's typical latency plus a margin — measure the p95/p99 response time under normal conditions, set the timeout a bit above that. Too tight and you fail healthy requests; too loose and you're back to hanging workers."*

**Likely pushback:** *"What's the risk with retries even for idempotent calls?"*
**Answer:** *"If everyone retries at the same interval, you get a retry storm — thundering herd — that makes the outage worse. That's what the jitter is for: randomizing the backoff slightly so retries spread out instead of syncing up."*

---

### 7. "Users need to upload large files"

**What to say:** *"Don't proxy the bytes through your app server — issue a presigned S3 URL and let the client upload directly to S3. My app only ever handles a small metadata request; the heavy bytes never transit it. Validate cheap things — file type, declared size, permissions — before issuing the URL. Process the heavy stuff — virus scan, parsing, thumbnailing — after, in a background task once the upload lands. For very large files, S3 multipart upload — parallel chunks, resumable if the connection drops."*

**Real example:** conceptual, mirror-image of the report-download presigned URL pattern in the big-report scenario — good to explicitly connect the two in the interview to show you see the pattern, not just the individual answer.

**Likely pushback:** *"If validation happens after upload, what if the file is actually malicious or the wrong type?"*
**Answer:** *"I still do cheap checks before issuing the URL — declared content-type and size limits on the presigned URL itself, which S3 will enforce. The expensive checks — virus scanning, deep content validation — happen after, and if they fail, I mark the upload as rejected and clean up the file. I never trust the client's declared type alone for anything security-sensitive, but I also don't want to block the upload on a full scan synchronously."*

**Likely pushback:** *"Why not just increase the app server's request size limit and handle the upload directly?"*
**Answer:** *"That ties up an app worker for the whole upload duration — for a large file over a slow connection, that's minutes of a worker doing nothing but babysitting bytes. Presigned URLs let S3 handle that entirely, and my worker capacity stays free for actual application logic."*

---

### 8. "An endpoint is getting hammered — how do you rate limit it?"

**What to say:** *"Two layers, ideally both: edge — a WAF or load balancer — for abuse and bots, and the app layer — DRF throttle classes — for per-user fairness. The mechanism is a counter in Redis keyed by user or IP plus a time window; it has to be Redis, not process memory, because with multiple app servers a local counter undercounts by a factor of however many servers you have. Return 429 with a Retry-After header so well-behaved clients back off properly. And I'd question the premise too — if it's legitimate traffic, the answer might be caching the hot response, not throttling real users."*

**Real example:** conceptual unless Propylon has hit this with the datastore API under load from multiple mt-* apps — worth checking.

**Likely pushback:** *"Fixed window vs token bucket — why would you pick one over the other?"*
**Answer:** *"Fixed window is simpler to implement and reason about, but it has a boundary problem — a user can burst right at the edge of two windows and effectively get double the limit in a short span. Token bucket smooths that out, allowing controlled bursts without the boundary exploit, at the cost of slightly more implementation complexity."*

**Likely pushback:** *"What's the front-end responsibility here — isn't rate limiting purely a backend concern?"*
**Answer:** *"No — the front end should honour the Retry-After header and back off, rather than immediately retrying and making it worse. And debouncing inputs that fire requests, like search-as-you-type, reduces how often you're hitting the limit in the first place — prevention alongside the server-side enforcement."*

---

### The pattern across all eight

Every single one of these rewards the same shape of answer: **name the naive/wrong-but-tempting approach first (implicitly, by contrast), diagnose the real constraint, offer 2-3 options with an explicit tradeoff, then commit to a pick with a reason.** The strongest answers you gave in our session (fund-admin locking, the image storage question) both had this shape — you reasoned to it live rather than reciting it, which is exactly what "plain language, real reasoning" means for this interviewer.