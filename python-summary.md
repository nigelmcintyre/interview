**Python** section — what to say, real examples, and likely pushback for each topic.

---

## PYTHON

### 1. Concurrency — multithreading, multiprocessing, async ⭐ MUST HAVE

**What to say:** *"It all comes down to the GIL — in CPython, only one thread executes Python bytecode at a time. So for I/O-bound work — waiting on a network call, a DB, a disk read — threads help, because the GIL releases while a thread is blocked. For CPU-bound work — heavy computation, parsing — threads don't help at all, because the GIL means only one thread runs Python code regardless. That needs multiprocessing, separate processes with their own interpreter and GIL, giving true parallelism. Async is a third option for I/O-bound work at much higher concurrency — a single thread, one event loop, cooperative multitasking — but it requires async-aware libraries all the way down."*

**Real example:** *"At Propylon, if there's CPU-heavy XML parsing of legacy `.doc` files, that's a multiprocessing story. If it's many API calls between the VSTO add-in and the Django backend, that's an I/O and threads story."* (Confirm which is actually true before using it.)

**Likely pushback:** *"If threads can't run Python code in parallel, why do they help at all?"*
**Answer:** *"Because the bottleneck isn't the Python code — it's waiting. While one thread is blocked on a network response, the GIL is released and another thread runs. The GIL only serializes actual bytecode execution, not I/O waiting."*

**Likely pushback:** *"When would you choose threads over async for I/O-bound work?"*
**Answer:** *"Threads are a bolt-on — you wrap existing blocking calls in a `ThreadPoolExecutor` with no rewrite. Async needs the whole stack to be async-aware — your HTTP client, DB driver, everything. If it's a handful of blocking calls, threads are simpler. If I need thousands of concurrent connections, async is the only thing that scales cheaply enough."*

**Worked pattern to have ready** (you didn't know this cold today — practice it):
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=3) as executor:
    future_1 = executor.submit(call_api_1)
    future_2 = executor.submit(call_api_2)
    future_3 = executor.submit(call_api_3)
    results = [f.result() for f in [future_1, future_2, future_3]]
```

---

### 2. Types

**What to say:** *"Python is dynamically typed — types are checked at runtime, not declared — but strongly typed, meaning it won't silently coerce a string into an int. Type hints are optional and not enforced at runtime by the language itself; they're for readability and tools like mypy. The exception is Pydantic and FastAPI, which use hints at runtime to actually validate incoming data."*

**Real example:** *"Coming from C#, I lean on type hints heavily to keep dynamic code readable, and Pydantic gives me the C#-like guarantee that the type is actually enforced, not just documented."*

**Likely pushback:** *"If type hints aren't enforced, what's the point of writing them?"*
**Answer:** *"Two things: tooling — my editor and mypy catch mismatches before I even run the code — and documentation that can't drift, because it's checked by tools even if not by the runtime. Pydantic is the exception that actually enforces at runtime, which is why FastAPI leans on it for request validation."*

---

### 3. Mutable / immutable

**What to say:** *"Immutable types — int, float, str, tuple, frozenset — can't be changed after creation. Mutable types — list, dict, set, most custom objects — can. The classic gotcha is a mutable default argument: `def add(item, items=[])` creates that list once, at function definition time, and every call shares it. Fix it with `items=None`, then `items = items or []` inside the function."*

**Real example:** *"I hit exactly this bug pattern once — a shared list persisting state across calls that should have been independent."* (Use a real one if you have it; otherwise the general pattern is fine to describe.)

**Likely pushback:** *"Walk me through exactly what happens when that buggy function is called twice."*
**Answer:** *"First call: `add('Alice')` returns `['Alice']`. Second call: `add('Bob')` — but the default list wasn't recreated, it's the same object from the first call — so it returns `['Alice', 'Bob']`, which the caller wasn't expecting since they didn't pass Alice in this time."*

**Likely pushback:** *"What's the difference between `copy.copy()` and `copy.deepcopy()`?"*
**Answer:** *"`copy.copy()` copies the outer object but shares nested mutable objects — a shallow copy. `copy.deepcopy()` recursively copies everything, so nested objects are fully independent too. Slicing a list or `dict.copy()` are both shallow."*

---

### 4. Scope — LEGB

**What to say:** *"Name lookup follows LEGB — Local, Enclosing, Global, Built-in. The gotcha: assigning to a name anywhere in a function makes Python treat it as local for the *entire* function, even before the assignment line — so reading it first raises `UnboundLocalError`. `nonlocal` lets you reassign a name from an enclosing scope instead of creating a new local one."*

**Real example:** conceptual — this is a language-mechanics question, not usually tied to a real story. Keep it crisp rather than forcing an example.

**Likely pushback:** *"Why does Python decide a variable is local for the whole function, even before it's assigned?"*
**Answer:** *"Python determines scope at compile time by scanning the function body — if it sees an assignment to a name anywhere in the function, it marks that name as local for the whole function, regardless of execution order. That's a static decision, not a runtime one, which is exactly why it trips people up — it doesn't matter that the read happens before the write in execution order."*

---

### 5. Generators / iterators

**What to say:** *"An iterator is anything you can call `next()` on. A generator — a function using `yield` — is the easy way to write one: it pauses at each yield and resumes where it left off, so values are produced lazily, one at a time, with only one item in memory at once. That's different from a list comprehension, which builds the whole thing upfront."*

**Real example:** *"This is the memory story behind streaming a big report — Django's `.iterator(chunk_size=...)` streams a queryset instead of loading a million rows into memory at once. Underneath, it's lazy — one chunk at a time."*

**Likely pushback:** *"What's the tradeoff of using a generator versus just building a list?"*
**Answer:** *"A generator is one-shot — once exhausted, you can't iterate it again, and you can't index into it or check its length upfront. A list holds everything in memory but you can access any element repeatedly and randomly. Use a generator when the dataset is large and you're processing sequentially; use a list when you need to reuse or randomly access the data."*

---

### 6. Garbage collector

**What to say:** *"Two mechanisms. Reference counting is primary — every object tracks how many references point to it, and it's freed immediately when that hits zero. That can't catch reference cycles, though — A pointing to B pointing back to A — so there's a backup generational cyclic garbage collector that periodically scans for and collects cycles."*

**Real example:** conceptual — this is pure language mechanics, no real-world example needed. Keep the C# contrast ready instead.

**Likely pushback:** *"How is this different from C#'s garbage collector?"*
**Answer:** *"C# uses a tracing generational GC with no reference counting at all — objects are only collected when the tracing collector runs. Python frees most things instantly via refcounts the moment they're no longer referenced, and only falls back to a tracing collector for the cycle case. So Python's memory reclamation is generally more immediate and predictable, except for cycles."*

---

### 7. Decorators

**What to say:** *"A decorator is a function that takes a function and returns a wrapped version — `@decorator` is sugar for `f = decorator(f)`. They're for cross-cutting concerns you don't want to repeat in every function body: auth checks, logging, timing, caching, registering a FastAPI route."*

**Real example:** *"`@login_required` on a Django view — it checks the user's authenticated before the view runs, so I don't have to write that check in every single view function. In FastAPI, `@app.get('/documents')` registers the function as a web route — FastAPI handles parsing the request, validating it, calling the function, and serializing the response, all invisible to me."*

**Likely pushback:** *"Why not just call a check function manually at the top of every view instead of using a decorator?"*
**Answer:** *"You could, but then that boilerplate is duplicated in every view, and it's easy to forget on a new one. The decorator extracts it once, applies it declaratively with one line, and it's impossible to forget — the annotation is right there next to the function definition."*

**Likely pushback:** *"What does `functools.wraps` do and why does it matter?"*
**Answer:** *"Without it, the wrapper function replaces the original's `__name__` and docstring with its own generic ones, which breaks introspection and makes debugging confusing — stack traces and `help()` show the wrapper, not the real function. `functools.wraps` copies that metadata over so the wrapped function still looks like itself."*

---

### 8. Context managers

**What to say:** *"`with` guarantees setup and teardown run, even if the body raises an exception — `with open(f) as fh:` always closes the file. Under the hood, `__enter__` runs on entry, `__exit__` runs on exit regardless of whether an exception occurred. You can write one with a class, or more simply with `@contextlib.contextmanager` on a generator function, where the `yield` marks the boundary between setup and teardown."*

**Real example:** *"Django's `transaction.atomic()` is a context manager — everything inside commits together or rolls back together if anything raises. That's the actual mechanism behind the concurrent-edits and duplicate-request answers."*

**Likely pushback:** *"How is a context manager different from a decorator, since both 'wrap' something?"*
**Answer:** *"A decorator wraps a function — it changes behaviour every time that function is called. A context manager wraps a block of code inside `with` — it's about guaranteeing setup/teardown around a specific scope, not about the function's identity. You could use either for logging, say, but a context manager is the natural fit when you need guaranteed cleanup even on an exception."*

---

### 9. Middlewares / signals (Django)

**What to say:** *"Middleware are components in the request/response pipeline — every request passes through them on the way in, and the response passes back through in reverse order on the way out. Auth, sessions, CSRF, CORS, logging all live here, and order matters. Signals are decoupled event notifications — a sender fires a signal like `post_save`, and any number of receivers can react, without the sender knowing who's listening."*

**Real example:** *"Does the Ascended Datastore use signals for an audit trail or cache invalidation, or to keep the Word front end and backend consistent? That maintaining-data-consistency work on my CV is a natural fit for this — need to confirm the actual mechanism before claiming it."*

**Likely pushback:** *"What's the downside of signals, since decoupling sounds like a good thing?"*
**Answer:** *"Decoupling makes control flow hard to trace — if you're reading the model's save method, you won't see that a signal fires and something else reacts elsewhere in the codebase. An explicit function call is often clearer. I'd reach for signals when the sender genuinely shouldn't know about the receiver — like multiple independent apps all reacting to the same event — otherwise I'd just call the function directly."*

---

### 10. Django ORM — N+1, `select_related` / `prefetch_related`

**What to say:** *"N+1 is: one query fetches N rows, then accessing a related field lazily on each row fires one more query per row — N+1 queries total instead of one or two. `select_related` fixes it for a foreign key by doing a SQL JOIN in one query. `prefetch_related` fixes it for many-to-many or reverse foreign keys with two queries, stitched together in Python, because a JOIN would multiply the rows out."*

**Real example:**
```python
bills = Bill.objects.filter(session="2025")
for bill in bills:
    bill.primary_sponsor.full_name       # +1 query per bill — N+1
    [s.full_name for s in bill.co_sponsors]  # +1 more per bill

# Fixed:
bills = (
    Bill.objects.filter(session="2025")
    .select_related("primary_sponsor")   # one JOIN
    .prefetch_related("co_sponsors")     # two queries, stitched by key
)
```
*"And I've seen this exact pattern beyond Django — our internal ORM-like layer over the legislative document store has the same two methods, solving the same N+1 problem: `select_related` returns the related asset inline in the same HTTP response instead of a JOIN, and `prefetch_related` batch-fetches every relation the queryset needs and stitches it back in by key. Same algorithm, worse penalty, because each lazy access there is an HTTP round trip, not just a SQL query."*

**Likely pushback:** *"How would you actually detect N+1 in production, versus just guessing?"*
**Answer:** *"The symptom is a lot of identical, individually fast queries — not one slow query — so a single EXPLAIN ANALYZE on one query won't show it. I'd use Django's debug toolbar locally, or `django.db.connection.queries` to count actual queries fired, or an APM tool in production that shows query count per request."*

**Likely pushback:** *"Why does `prefetch_related` need two queries instead of one JOIN like `select_related`?"*
**Answer:** *"Because it's a many-to-many or reverse FK — if you tried to JOIN, each row would multiply out by however many related rows it has, so a bill with 5 co-sponsors would appear 5 times in the result set. Fetching separately and stitching by key in Python avoids that row multiplication."*

---

### 11. Cache — in-memory vs Redis

**What to say:** *"In-memory caching is fastest, but it's per-process and doesn't survive a restart. Redis is a network hop slower but shared across all app servers and persists — which matters the moment you're running more than one app instance. The hard part isn't caching, it's invalidation — knowing when to expire or clear a stale entry. TTLs and event-based invalidation, like on `post_save`, are the usual tools."*

**Real example:** *"At Irish Life, the OLS fix I made was about not over-fetching fund prices — caching would be the natural next lever if that data doesn't change often within a session."*

**Likely pushback:** *"If Redis is slower than in-memory, why would you ever choose it?"*
**Answer:** *"The moment you have more than one app server, an in-memory cache is useless for consistency — each server has its own separate cache, so a user could get different answers depending on which server handles their request. Redis being shared is worth the network hop in any horizontally-scaled setup."*

---

### 12. Background tasks — Celery / SQS

**What to say:** *"Never do slow work inside the request cycle — report generation, file processing, emails — because it blocks a worker and risks timing out. Celery needs a broker, usually Redis or RabbitMQ; workers pull tasks off it and run them. The pattern is: the request enqueues a job and returns immediately with a job ID, then the client polls or gets notified when the work is done."*

**Real example:** *"If Propylon uses Celery for any document processing, that's worth confirming and using — otherwise this stays conceptual, tied into my rehearsed big-report answer where it's the central mechanism."*

**Likely pushback:** *"What happens if a Celery worker crashes mid-task?"*
**Answer:** *"Depends on the broker's acknowledgment settings — with 'late ack,' the task is only marked done after it completes, so if the worker dies mid-task, the message goes back on the queue and another worker picks it up. That's also exactly why background tasks need to be idempotent — at-least-once delivery means the same task can genuinely run twice."*

--