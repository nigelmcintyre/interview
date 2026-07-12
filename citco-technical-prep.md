# Citco — Technical Interview Prep

> **The one rule (from your referral):** the interviewer prefers *real problems and real solutions in plain language* over big terminology. For each topic below there's a concept explanation so you understand it, then a **🔧 Real example** hook — that's where you drop in something true from your Propylon Django/Postgres work (or DocIntel / SaunaGuide / Irish Life). Learn the concept; *lead with the story*.
>
> Where I've suggested a possible Propylon angle, it's a **candidate you must confirm is true** before using it. Only claim what you can defend under one follow-up question.

---

## PYTHON

### Concurrency — multiprocessing / multithreading / async  ⭐ MUST HAVE

The whole answer hinges on one fact: **the GIL (Global Interpreter Lock)**. In CPython only one thread runs Python bytecode at a time. So:

- **Multithreading** (`threading`) — threads share memory, but the GIL means they *don't* run Python code in true parallel. They still help for **I/O-bound** work, because the GIL is released while a thread waits on the network, disk, or a DB. Many DB/API calls in flight → threads help.
- **Multiprocessing** (`multiprocessing`) — separate processes, each with its own interpreter and GIL, so you get **true parallelism** for **CPU-bound** work (heavy computation, parsing). Cost: process startup and inter-process communication overhead; no shared memory by default.
- **Async** (`asyncio`, `async`/`await`) — single thread, one event loop, cooperative multitasking. One task yields while waiting on I/O so another runs. Scales to thousands of concurrent I/O operations far more cheaply than threads. Catch: needs async-aware libraries all the way down, and one blocking call stalls the whole loop.

**The decision rule (say this):** *"CPU-bound → multiprocessing. I/O-bound → threads or async. If it's lots of concurrent I/O, async; if I just want to overlap a handful of blocking calls, threads."*

**If asked "what is the GIL?" cold**, have a standalone one-liner ready (it's often asked separately from the concurrency question): *"A mutex in CPython that lets only one thread execute Python bytecode at a time — it keeps memory management thread-safe, but it's why threads don't give you CPU parallelism."*

**From C#:** you're used to real parallel threads and `Task`/`async`/`await` with no GIL — so the mental shift is just "in Python, threads ≠ CPU parallelism."

**Django note:** classic Django is synchronous (WSGI). Django 3.1+ supports async views (ASGI). Heavy work is usually pushed to **Celery** rather than done in the request. Worth saying if asked how Django handles it.

**🔧 Real example — multithreading (verified in the codebase):**

Fan-out/fan-in in a DRF view — [mt-cm-common-plugin/src/cm/cm_core/views/committee.py:140-171](../mt-cm-common-plugin/src/cm/cm_core/views/committee.py#L140-L171)

```python
threads = [
    WorkerThread(target=self.write_cmt_witnesses, args=(witnesses, committee)),
    WorkerThread(target=self.set_committee_members, args=(committee_members, committee)),
    WorkerThread(target=self.populate_meeting_committees, args=(meetings, committee)),
]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

Committee creation needs three independent writes to the datastore — witnesses, members, meetings. The parent `committee` is created and saved *first*; all three threads write against that already-existing committee, not against each other, so there's no ordering dependency between them. Each write is an HTTP call to the central document service — textbook I/O-bound: the GIL releases while one thread waits on the network, so the next thread's request fires instead of the CPU sitting idle.

Depth details, ready but not volunteered unprompted:
- Standard `threading.Thread` doesn't propagate exceptions to the joining thread by default — a custom `WorkerThread` subclass wraps `run`/`join` to capture and re-raise.
- A comment right above the spawn: *"Resolve backend in the main request thread before spawning workers — WorkerThreads have no thread-local request, so lazy resolution would fail."* A real thread-local-state gotcha: Django/DRF lazily resolves the request-scoped backend, and that lazy resolution breaks inside a spawned thread because no request is bound to that thread's local storage.
- `WorkerThreadPool` (a `ThreadPoolExecutor` subclass with the same fix) is defined alongside it but **no call site instantiates it** — the codebase consistently uses manual `start()`/`join()`. Say that precisely if asked; don't imply the pool is in active use.

⚠️ **Not cold yet** — the plain `ThreadPoolExecutor` API (as opposed to this codebase's manual `start()`/`join()`) isn't locked in. Have I actually written and timed a `ThreadPoolExecutor` example, not just described the concept?

**🔧 Worked example — `ThreadPoolExecutor`:**
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=3) as executor:
    future_1 = executor.submit(call_api_1)
    future_2 = executor.submit(call_api_2)
    result_1 = future_1.result()  # blocks until call_api_1 finishes
```
`submit()` returns a `Future` immediately and runs the call on a pool thread; `.result()` blocks the calling thread until that specific future finishes. The `with` block waits for all submitted work to complete before exiting — same fan-out/fan-in shape as the manual `start()`/`join()` pattern above, just with pooled threads and a cleaner call site.

**The line to say:** *"Committee creation needs three independent writes to our document store — witnesses, members, meetings — each an HTTP call. We fan them out on threads: the GIL releases while each thread waits on the network, so the three requests overlap instead of running one after another. Textbook I/O-bound case, so threads are the right tool, not multiprocessing."*

**🔧 Real example — multiprocessing (Celery):** Celery's default worker pool (**prefork**) is separate OS processes, each with its own interpreter and GIL — genuine multiprocessing, not threading, even though Celery gets introduced as "just a task queue." **The line to say:** *"Celery's prefork worker pool is already multiprocessing under the hood — separate OS processes, so if a task is CPU-heavy, I'd scale worker processes for real parallelism with no GIL contention between them."*

---

### Types

**Two separate axes — don't merge them (easy interview trap):**
- **Static vs dynamic** = *when* the type is checked. Static (C#, Java): declared up front, checked at compile time. Dynamic (Python): no declaration, checked at runtime — a name can point at an `int` today and a `str` tomorrow.
- **Strong vs weak** = *whether the language silently coerces between types*. Strong (Python): refuses — `"2" + 2` raises `TypeError`, nothing happens. Weak (JavaScript): coerces — `"2" + 2` silently becomes `"22"`.

Python is **dynamic + strong**. C# is **static + strong**. JS is **dynamic + weak**. Everything in Python is an object.

**Type hints** (`def f(x: int) -> str:`) are optional and *not enforced at runtime by the interpreter* — they're for readability and for static tools like `mypy`/`pyright` (which check them before you run, without executing the code — the closest thing to "compile-time" Python has, and it's opt-in tooling, not the language). The exception: **Pydantic and FastAPI use hints at runtime** to actually validate data.

**The line to say (Pydantic/FastAPI):** *"Pydantic and FastAPI take type hints — which the interpreter itself never enforces — and validate them at runtime. Before your function body executes, incoming data gets checked against the hint; if it doesn't match, it never reaches your code — the request is rejected with a structured error. So it's not static typing, but it gives you a static-typing-like guarantee at the boundary, enforced dynamically instead of at compile time."*

**Contrast with plain Django:** a plain view doesn't use hints for enforcement at all — nothing catches a mismatch automatically. If a view expects an int and gets a string from `request.POST`, nothing complains until your own code tries to do something int-like with it, and *that* raises the error, possibly several lines later, deep in your logic. DRF serializers or Pydantic validate the shape up front and reject cleanly with a 400/422 — fail-fast-and-clear vs fail-late-and-wherever-it-breaks.

**From C#:** hints look like C# types but behave more like documentation — the compiler isn't stopping you. Pydantic is the thing that gives you C#-like "the type is real" enforcement.

**🔧 Real example:** if the Django app uses type hints or Pydantic-style validation anywhere, mention it. If not, honest framing: "coming from C#, I lean on type hints heavily to keep dynamic code readable."

---

### Mutable / immutable

- **Immutable:** `int`, `float`, `str`, `tuple`, `frozenset`, `bytes`.
- **Mutable:** `list`, `dict`, `set`, most custom objects.

**Why an interviewer asks** — the gotchas:

1. **Mutable default argument:** `def add(item, items=[])` — the default `[]` is created **once, at function *definition* time**, not on every call. Any call that *doesn't* pass its own `items` reuses that same shared list object — `add("Mary")` then `add("John")` (both relying on the default) leaves you with `["Mary", "John"]`, not two independent single-item lists.
   Fix — create a genuinely new list every call, don't reuse or clear the old one:
   ```python
   def add(item, items=None):
       items = items if items is not None else []   # or: items = items or []
       items.append(item)
       return items
   ```
   `None` is immutable, so it's safe as the shared default; the fresh `[]` only gets built inside the function body, which runs anew each call.

2. **Passing mutables into functions — "pass by object reference," not "pass by reference":** the parameter name inside the function is bound to the *same object* the caller's variable points to, so a **mutation** through it (`.append()`, item assignment) is visible to the caller. But a **reassignment** of the parameter name (`items = []`) only rebinds that local name to a new object — it does not touch what the caller's variable points to. The test that proves it: a function that does `items = []` instead of mutating never empties the caller's list. Names are labels pointing at objects; each scope has its own labels even when they start out aimed at the same object.

3. **Dict/set keys must be hashable** → must be immutable. That's why you can key on a tuple but not a list.

**Two cheap add-ons that get asked in the same breath:**
- **`copy` vs `deepcopy`:** `copy.copy()` makes a new *outer* container but doesn't copy the elements inside it — nested mutables are shared. Concretely: `shallow = copy.copy(original)` where `original = [[1,2],[3,4]]` — `shallow[0]` and `original[0]` are the *same* inner-list object, so `shallow[0].append(99)` also changes `original`. `copy.deepcopy()` recursively copies every nested object too, so there's no shared reference at any depth. Slicing (`lst[:]`) and `dict.copy()` are shallow.
- **`is` vs `==`:** `is` checks **identity** — same object in memory (`id(a) == id(b)`), not type and not value. `==` compares value. Reserve `is` almost exclusively for `x is None` (identity is the actual meaning there). Gotcha: CPython caches small ints (-5 to 256) and interns some string literals, so `a is b` can *look* true for separately-written literals — that's a CPython implementation detail/optimization, not a language guarantee, and it silently breaks for runtime-built strings or larger ints. Never rely on it; use `==` for value comparison.

**🔧 Real example:** a bug you hit where a shared/mutable value caused unexpected behaviour is perfect here — even a small one.

---

### Data structures — quick reference

Not a named topic, but the vocabulary underneath everything above — worth having crisp.

| Structure | Ordered? | Mutable? | Duplicates? | Notes |
|---|---|---|---|---|
| `list` | ✅ | ✅ | ✅ | general-purpose sequence |
| `tuple` | ✅ | ❌ | ✅ | immutable → hashable if contents are; usable as a dict key (a list can't be) |
| `dict` | ✅ (insertion order, guaranteed since 3.7) | ✅ | keys unique, values can dup | hash table under the hood, O(1) avg lookup |
| `set` | ❌ | ✅ | ❌ (unique only) | hash table, O(1) avg membership test |
| `frozenset` | ❌ | ❌ | ❌ | immutable set → hashable, usable as a dict key |
| `str` | ✅ | ❌ | ✅ | immutable sequence of characters |

Tuple isn't "a kind of list" in a type-hierarchy sense — they're separate types. The similarity is that both are ordered sequences allowing duplicates/mixed types; the real difference is mutability, and that's exactly why a tuple can be a dict key or set member and a list can't (ties to the hashability rule above).

Worth knowing exist in `collections` if it comes up: `defaultdict` (auto-default on missing key), `Counter` (frequency counting), `deque` (O(1) append/pop from *both* ends, unlike list's O(n) from the front).

**`*args` / `**kwargs`:** `*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. Same symbols also *unpack* a tuple/dict back out into a call (`func(*args, **kwargs)`). The reason decorators lean on this so heavily: a generic decorator doesn't know the signature of whatever function it wraps, so `def wrapper(*args, **kwargs): return func(*args, **kwargs)` accepts anything the caller passes and forwards it transparently — without this, you'd need a differently-shaped decorator for every function signature.

---

### Scope — LEGB

Name lookup follows **LEGB**: **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in. `global` and `nonlocal` let you reassign names in outer scopes.

**The gotcha, precisely:** Python's compiler scans the **whole function body ahead of time** — if it sees an assignment to a name *anywhere* in that function, the name is classified as local **for the entire function**, from the first line, before any of it has run. So local wins the LEGB lookup immediately; Python never even checks the enclosing/global scope for that name. If you *read* the name before the line that assigns it, you get `UnboundLocalError` — not "used the outer value, then shadowed it."

```python
x = "global"

def outer():
    x = "outer"

    def inner():
        print(x)     # UnboundLocalError — x is already local to inner() (see x = "inner" below),
        x = "inner"   # so LEGB never looks at outer()'s x here.

    inner()
```

**The fix, `nonlocal`:** `nonlocal x` at the top of `inner()` redirects *assignment* targets — instead of creating a local shadow, `x = "inner"` now rebinds the existing `x` that lives in `outer()`. With it: `print(x)` prints `"outer"` (no error — `x` isn't local anymore), then `outer()`'s `x` genuinely becomes `"inner"` after `inner()` returns. `global` is the same idea one level up, for module-level names.

**Related trap — don't say "pass `x` in and reassign it" as a fix.** Reassigning a parameter inside a function only rebinds that local name; it never writes back to the caller's variable or object, mutable or immutable. Same rule as the mutable-default-argument gotcha above — reassignment never propagates outward, only in-place mutation of a shared mutable object does.

Keep this one short in the room — a crisp LEGB answer plus the `UnboundLocalError`/`nonlocal` gotcha is plenty. Closures capture the enclosing *variable*, not its value at definition time — worth a one-liner if asked about closures directly.

---

### Generators / iterators

An **iterator** is anything you can call `next()` on — it produces values one at a time and raises `StopIteration` when exhausted. That's the protocol (`__iter__` + `__next__`); a plain list is *iterable* but isn't itself an iterator — `iter(my_list)` produces the iterator object that `next()` actually operates on. A **generator** is the easy way to write one: a function with `yield` pauses at each yield, hands back the yielded value, and **freezes all local state** (variables, where execution stopped) so the next `next()` call resumes from exactly that point.

- `yield` vs `return`: `yield` pauses and preserves local state between calls; `return` ends the function and discards its state.
- **Generator expression** `(x*2 for x in rows)` vs list comprehension `[x*2 for x in rows]` — same syntax, but the genexp is lazy and constant-memory.
- One-shot: once exhausted, a generator is done — you can't iterate it twice; call the generator function again for a fresh one.

**Why it matters — put a number on it:** 10 million rows at ~1KB each. `list(queryset)` (or any full materialization) builds *every* row into memory before your loop even starts — ~10GB on one worker, before processing row #1. A generator (`.iterator(chunk_size=2000)`) only ever holds the current row/chunk — memory stays flat at a few KB–MB **regardless of table size**, because nothing downstream holds a reference to rows once they're processed.

**🧭 Tech design link:** this is the memory story inside **"big report"** and **"millions of rows"** — Django's `.iterator(chunk_size=...)` streams a queryset instead of loading it. *"That stays flat in memory because it's a generator underneath — lazy, one chunk at a time, so the report doesn't OOM the worker regardless of table size."*

**🔧 Real example:** any place you stream/chunk instead of load-all — the report chunking pattern *is* this.

---

### Garbage collector

Two mechanisms:
1. **Reference counting (primary):** every object tracks how many references point to it; when the count hits zero it's freed *immediately*.
2. **Cyclic GC (backup):** reference counting can't free **reference cycles** (A → B → A), so a **generational** garbage collector periodically finds and collects cycles. Three generations; newer objects are checked more often.

**From C#:** C# uses a tracing generational GC with *no* reference counting — so "Python frees most things instantly via refcounts, and only runs a tracing collector for cycles" is the contrast to draw.

---

### Decorators

A decorator is **a function that takes a function and returns a wrapped version** — `@decorator` is just syntactic sugar for `f = decorator(f)`. Used for cross-cutting concerns without touching the function body: timing, logging, caching (`@lru_cache`), auth (`@login_required`), FastAPI routes (`@app.get(...)`), `@property`.

⚠️ **Not cold yet** — you have the definition solid but not the *why*, and needed `@login_required` / `@app.get(...)` walked through from scratch in a prep session rather than producing them unprompted. The why, in one line: without the decorator you'd have to call the auth check (or route-registration code) manually at the top of *every* view — the decorator lets you write that boilerplate once and stick it on any function with `@`, instead of repeating it. Lock in a real example before the interview.

**`functools.wraps` — why it's needed:** `f = decorator(f)` means the decorated name now points at your inner `wrapper` function, not the original — so without intervention, introspection (`__name__`, `__doc__`, stack traces, logging) reports the *wrapper's* identity, not the real one:
```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def say_hello():
    """Prints hello"""

say_hello.__name__   # "wrapper" — wrong
```
**The concrete failure case:** every decorated function in a log line or stack trace shows up as `"wrapper"` — exactly the identifying detail you need to go find the actual bug disappears. `functools.wraps(func)` applied to the inner `wrapper` copies `__name__`, `__doc__`, `__module__` from the original onto it, so `say_hello.__name__` correctly reports `"say_hello"`.

**From C#:** conceptually close to attributes + middleware/aspects, but decorators actually *wrap and run* code rather than just annotate.

**🔧 Real example:** any custom decorator in the Django code (a permission check, a timing/logging wrapper)? Or just "`@login_required` and DRF's `@api_view` in the Django app."

> **Rehearsed answer:** "A decorator like `@app.get("/documents")` wraps a function to add cross-cutting behaviour — in this case, registering it as a web endpoint. FastAPI handles parsing the request, validating data, calling the function, and serializing the response. Without the decorator, it's just a regular function — the decorator is what makes it a web route."

---

### Context managers

`with` guarantees setup/teardown runs **even if the body raises** — `with open(f) as fh:` always closes the file. Under the hood: `__enter__` on entry, `__exit__` on exit (exception or not). Write your own with a class, or more easily with `@contextlib.contextmanager` on a generator — the `yield` marks the boundary between setup and teardown.

**Pairs with decorators in interviews:** both wrap behaviour around code — a decorator wraps a *function*, a context manager wraps a *block*.

**🧭 Tech design link:** Django's `transaction.atomic()` *is* a context manager — `__enter__` begins a transaction (or a savepoint, if nested), `__exit__` commits if the block completes clean or rolls back if an exception propagated out. Same pattern as `with open(...)`: guaranteed cleanup either way.

**Don't conflate atomicity with locking — a common trap.** `transaction.atomic()` on its own only guarantees **atomicity**: everything inside commits together or rolls back together, no partial writes. It does **not** lock anything or prevent two concurrent transactions from both reading and writing the same row. Plain `with transaction.atomic(): obj.save()` with no explicit locking can still lose an update if two requests race. Locking is a separate, additional thing you add *inside* the atomic block: **pessimistic** (`select_for_update()` — locks the row, others block until commit) or **optimistic** (a `version` column + conditional `UPDATE ... WHERE id=? AND version=?`). This feeds the **concurrent edits** and **duplicate requests** scenarios below.

---

### Middlewares / signals (Django)

- **Middleware:** components in the request/response pipeline. Every request passes through the stack on the way in and the response on the way out — auth, sessions, CSRF, CORS, logging all live here. Order matters.
- **Signals:** decoupled event notifications. A sender fires a signal (`post_save`, `pre_delete`, `request_started`), and any number of receivers react. Good for side effects like "after a document is saved, invalidate its cache."

**The tradeoff to name (interviewers love this):** signals decouple code but make control flow *hard to trace* — an explicit function call is often clearer. Reach for signals when the sender shouldn't know about the receiver; otherwise call the code directly.

**🔧 Real example — verified in the codebase, but ⚠️ not authored by you (`git log` shows Brant Watson on both files) — frame as "our codebase does this," never "I wrote this":**

- **Middleware:** [lrms-core/src/ascended/webapp/middleware.py:17-33](../lrms-core/src/ascended/webapp/middleware.py#L17-L33) — a function-based `version_middleware` and a class-based `AscendedAnalyticsMiddleware`, registered right after CORS and before CSRF/session/auth in settings. Real, demonstrable order-matters example.
- **Signals:** [lrms-core/src/ascended/authentication/signals.py](../lrms-core/src/ascended/authentication/signals.py) and [lrms-core/src/ascended/virtualview/signals.py:10-20](../lrms-core/src/ascended/virtualview/signals.py#L10-L20):
  ```python
  @receiver(post_save, sender=User)
  @receiver(post_delete, sender=User)
  @receiver(m2m_changed, sender=User)
  def invalidate_user_cache_user(sender, instance, **kwargs):
      invalidate_user_cache_usernames((instance.username,))
  ```
  ```python
  @receiver(post_save, sender=VVMapping)
  @receiver(post_delete, sender=VVMapping)
  def increment_virtualview_version(sender, instance, **kwargs):
      """Increment the virtualview version counter by 1"""
  ```
  `post_save`/`post_delete`/`m2m_changed` receivers invalidate or version a cache when `User`/`Group`/`Permission` or a `VVMapping` changes — genuinely the sender-shouldn't-know-about-the-receiver case: `User` has no idea a cache exists. **The tradeoff cuts both ways here too:** decoupled, but you have to go looking in `signals.py` to discover the side effect exists at all.

**The line to say (honest framing):** *"In our Django datastore service, cache invalidation on the auth models is wired through signals — `post_save`/`post_delete` receivers that invalidate a user's cache entry whenever the User or their permissions change. I haven't written that particular file, but it's exactly the pattern I'd reach for: the User model shouldn't need to know a cache exists, so the signal decouples that side effect from the save itself."*

Note: this pattern lives specifically in the standard-Django `ascended` app inside `lrms-core`. The apps you've actually committed to (`mt-cm-common-plugin`, `mt-chamber-interface`, etc.) sit on the custom Joplin/ORM-like datastore layer, not Django's ORM, so they don't emit `post_save`/`pre_save` signals the same way — don't imply you've used signals in your own commits.

---

### Django ORM performance — N+1, select_related / prefetch_related

They listed middlewares/signals, so this is a Django shop — ORM performance questions are near-certain.

- **N+1:** fetch 100 rows in one query, then accessing `row.author` lazily fires **one more query per row** — 101 queries instead of 2. The symptom is many identical *fast* queries, not one slow one — which is why an APM/log view catches it and `EXPLAIN ANALYZE` on a single query doesn't.
- **`select_related`** — foreign key / one-to-one: one query via a SQL **JOIN**.
- **`prefetch_related`** — many-to-many / reverse FK: **two queries** stitched in Python (a JOIN would multiply rows out).
- Column-level over-fetching: `.only()` / `.defer()` / `.values()`. To *see* the query count: debug toolbar or `django.db.connection.queries`.

**The worked example — legislative version of Post/author/tags** (real fields from `mt_models/sponsor.py`, simplified — in the real code they live on a Sponsor asset attached to the bill):

```python
# Bill ~ Post · primary_sponsor (FK, one legislator) ~ author · co_sponsors (M2M) ~ tags

bills = Bill.objects.filter(session="2025")          # 1 query
for bill in bills:
    bill.primary_sponsor.full_name                   # +1 per bill      ← N+1
    [leg.full_name for leg in bill.co_sponsors]      # +1 more per bill
# 100 bills → 201 round trips. In Joplin each is an HTTP call to the datastore, not a SQL query.

bills = (
    Bill.objects.filter(session="2025")
    .select_related("primary_sponsor")   # FK: related asset returned inline with each bill
    .prefetch_related("co_sponsors")     # M2M: one batched fetch, stitched back in by key
)
# 100 bills → 2 round trips. Spanning works too: .select_related("sponsors__party")
```

**How Joplin implements the same two methods (the depth detail):** there's no SQL, so no JOIN — `select_related` instead tells the datastore to *return the related asset's fields inline in the same HTTP response* (and a `RELATIONAL_DEFAULT_EAGER_LOADING_DEPTH` setting auto-eager-loads relations to a default depth). `prefetch_related` calls `populate_related()`, which walks the queryset, **collates and dedupes every APN the unloaded lazy loaders point at, batch-fetches each relation in one call** (parallelised with gevent greenlets), **and stitches the instances back in by key** — line for line the same algorithm as Django's `prefetch_related`.

**🧭 Tech design link:** the first diagnostic branch of **"slow API request"** — your rehearsed answer already walks it.

**🔧 Real example:** the Propylon codebase uses `select_related`/`prefetch_related` widely — and the stronger story is **Joplin**, Propylon's in-house Django-ORM-style model layer over the legislative document datastore. It hit *the same N+1 problem the Django ORM has* — accessing a related field lazily fired one datastore round trip per object — and solved it the same way: the `EagerLoadingDepth`/`query_cache.py` machinery batch-populates related fields up front, its equivalent of `prefetch_related`. **The line:** *"Our internal ORM-like layer over the document store had the same N+1 problem as the Django ORM, and solved it the same way — batched eager loading. N+1 isn't a Django quirk; it's what lazy loading does anywhere."* (Say "our internal ORM-like layer", not "Joplin" — don't make the interviewer decode codenames.)

---

### Cache — in-memory vs Redis (pros/cons)

**Why cache:** stop repeating expensive work — DB queries, computation, external calls.

| | In-memory (local process) | Redis / Memcached |
|---|---|---|
| Speed | Fastest (no network) | Fast, but a network hop |
| Shared across servers? | ❌ per-process | ✅ shared by all app instances |
| Survives restart? | ❌ | ✅ |
| Extra infra | None | A Redis server to run/manage |
| Extras | — | TTL, pub/sub, data structures, eviction (LRU) |

**Say this:** local memory cache is fine for a single process/small data; the moment you run multiple app servers or want persistence, you need a shared store like Redis. Django's cache framework supports both (per-view, template-fragment, and low-level `cache.get/set`).

**The hard part to acknowledge:** cache *invalidation* — serving stale data is the risk. TTLs and event-based invalidation (e.g. on `post_save`) are the tools.

**🔧 Real example:** Irish Life OLS — you fixed a slow load by not over-fetching fund prices; caching would be the *next* lever. If Propylon caches anything, use that; otherwise this is a conceptual answer, which is fine.

---

### Background tasks (Celery / SQS — briefly)

**Why:** never do slow work (report generation, file processing, emails) inside the request cycle — it blocks the worker and can time out.

- **Celery:** distributed task queue for Python. Needs a **broker** (Redis or RabbitMQ); workers pull and run tasks; Celery Beat schedules recurring jobs.
- **Amazon SQS:** managed message queue — producers push messages, consumers pull. Decouples services and smooths load spikes.

Pattern: request enqueues a job and returns immediately (e.g. `202 Accepted` + a job ID); a worker does the work; the client polls or gets notified when it's done.

Keep this brief unless pushed. **🔧** If Propylon uses Celery for document processing, that's a strong real example — confirm it.

---

## DATABASES

### SQL vs NoSQL

- **SQL / relational** (Postgres, SQL Server): fixed schema, tables + joins, **ACID** transactions, strong consistency. Best when data is structured and integrity matters — *which for a fund-services/trade-ops platform, it usually does.*
- **NoSQL:** document (MongoDB), key-value (Redis, DynamoDB), column (Cassandra), graph (Neo4j). Flexible schema, easier horizontal scaling, often eventual consistency. Best for unstructured/semi-structured data or very high write throughput.

**The nuance that scores points:** Postgres blurs the line — **JSONB** gives you schemaless document storage *inside* a relational, ACID database. So you often don't need a separate NoSQL store.

**From C#/Irish Life:** you're strong on relational (SQL Server, EF Core) — transfer that straight to Postgres.

**🔧 Real example (SQL *and* not-SQL, from the day job):** Montana is genuinely both worlds — relational Postgres via Django, *plus* a revisioned **document datastore** for legislative assets, accessed through Joplin, an in-house Django-ORM-style model layer (documents addressed by asset path + revision number, not primary key, with pluggable backends). **The line:** *"My day job is literally both: relational Postgres where integrity matters, and a versioned document store for the legislative documents themselves — because a 300-page bill with revision history isn't a row, it's a document. Right store for the shape of the data."* Also: JSONB is a genuine strength — SaunaGuide stores flexible per-listing data (`attributes`, `structured_data`) in **JSONB columns**, and DocIntel uses JSONB metadata. If the Propylon Postgres schema stores flexible legislative metadata in JSONB, even better. Use whichever is true. *⚠️ Don't say SaunaGuide's listing model is "config-driven by JSONB" — the filter config is a Python module (`niche_config`), the JSONB is just the flexible attribute storage. Honest line: "flexible listing attributes in JSONB; the filter config is a Python module that drives the UI."*

---

### EXPLAIN ANALYZE  ⭐ MUST HAVE

`EXPLAIN` shows the planner's **predicted** execution plan. `EXPLAIN ANALYZE` actually **runs** the query and shows *real* timings and row counts.

**What you look for (walk through this out loud):**
- **Seq Scan on a big table** where you filter/join → usually a **missing index**. An **Index Scan** is what you want on selective queries.
- **Estimated rows vs actual rows** wildly different → **stale statistics**; run `ANALYZE` (planner is guessing wrong).
- **Join type** — nested loop (fine for small sets), hash join, merge join.
- **Sort nodes**, especially "external merge Disk" → sorting spilled to disk; may need more `work_mem` or an index that provides order.
- **Total execution time** and **planning time**. Add `BUFFERS` (`EXPLAIN (ANALYZE, BUFFERS)`) to see cache hits vs disk reads.

**The plain-language answer they want:** *"Query was slow, I ran `EXPLAIN ANALYZE`, saw a sequential scan over a big table on a column I was filtering on, added an index, it flipped to an index scan and dropped from seconds to milliseconds."*

**🔧 Real example — this is the highest-value one to make real.** Have you ever profiled a slow query against the Propylon Postgres DB? If yes, that exact narrative (found the scan → added the index → measured the improvement) is your best technical story. If you haven't yet, it's worth *actually running* `EXPLAIN ANALYZE` on a real query in DocIntel before the interview so you can speak from having done it, not from theory.

---

### Indices and their types

Speed reads by avoiding full-table scans; cost is slower writes (every insert/update maintains the index) and disk. **Don't over-index.**

- **B-tree** (default): equality *and* range (`=`, `<`, `>`, `BETWEEN`), and ordering. The workhorse.
- **Hash:** equality only; rarely worth choosing over B-tree.
- **GIN** (Generalized Inverted Index): for values containing many items — **full-text search (`tsvector`)**, **JSONB**, arrays.
- **GiST:** geometric/spatial data, nearest-neighbour, some full-text.
- **BRIN** (Block Range Index): huge tables with naturally ordered data (e.g. append-only timestamps) — tiny index, great for time-series.
- **Partial index:** index only rows matching a `WHERE` clause (e.g. only `status = 'active'`) — smaller and faster.
- **Composite (multi-column):** order matters — the **leftmost-prefix** rule means an index on `(a, b)` helps queries on `a` or `a, b` but not `b` alone.

**🔧 Real example:** DocIntel genuinely uses **GIN indexes on `tsvector` and JSONB** — that's a concrete, specific answer few candidates give. Lead with it. **Composite-index example:** SaunaGuide's `Listing` has `Index(fields=["-is_featured", "name"])` — a real multi-column index where order matters (featured rows first, then alphabetical), plus a single `Index(fields=["county"])` because county is the main filter path. That's your leftmost-prefix / composite story from lived code. If the Propylon schema has indexed search columns, mention that too.

---

### Transactions, ACID & locking

The biggest omission from the original list — and Citco is fund administration, so transactional correctness is their bread and butter. Expect it.

- **ACID:** **A**tomic (all or nothing), **C**onsistent (constraints hold), **I**solated (concurrent transactions don't see each other's partial work), **D**urable (committed survives a crash).
- **In Django:** `transaction.atomic()` (context manager or decorator) — everything inside commits together or rolls back together. Classic use: create the order *and* decrement stock in one block.
- **Isolation levels** (know the two you'll meet): **Read Committed** — Postgres default; each statement sees only committed data, but two reads inside one transaction can see different data. **Repeatable Read** — one snapshot for the whole transaction. Higher isolation = fewer anomalies but more contention/retries.
- **Locking:**
  - **Pessimistic** — `select_for_update()`: lock the rows now; other writers block until you commit. Use when conflicts are likely or the operation must be serialized (anything touching money).
  - **Optimistic** — no lock; a `version` column and `UPDATE ... WHERE id = ? AND version = ?`. Zero rows updated → someone else got there first → retry or surface a conflict. Use when conflicts are rare.
- **Deadlock:** two transactions each hold a lock the other wants; Postgres detects it and kills one. Mitigate: acquire locks in a consistent order, keep transactions short.

**🧭 Tech design link:** the entire answer to **"two users edit the same record"** and the mechanism behind **"duplicate requests / idempotency"** below.

---

### Read replicas, connection pooling, partitioning (one sentence each)

- **Read replica:** a copy following the primary via replication — send heavy reads/reports there, writes to the primary. Caveat to name: **replication lag** (fine for a report, wrong for read-your-own-write).
- **Connection pooling (PgBouncer):** each Postgres connection is an expensive process; a pooler multiplexes many short app connections over a few real ones so bursts don't exhaust the DB.
- **Partitioning:** split a huge table (usually by date) so queries scan one partition and old data can be dropped cheaply.

**🧭 Tech design link:** all three live in **"millions of rows"**; the replica (with the lag caveat) is already in your rehearsed **"big report"** answer.

---

## CLOUD — EC2, ECS, RDS, load balancers, deploys, CI/CD

Be honest here: your hands-on cloud is limited (Azure AD SSO at Irish Life; Docker/Ansible on Red Hat at Infobip). Know the concepts, and bridge honestly rather than overclaim.

- **EC2:** virtual machines — you manage the OS and runtime. Maximum control, maximum ops burden.
- **ECS:** container orchestration for Docker. With **Fargate** you don't manage servers at all (serverless containers).
- **RDS:** managed relational DB (Postgres, etc.) — AWS handles backups, patching, failover, replicas. **Aurora** is AWS's higher-performance variant.
- **Load balancer (ALB/NLB):** spreads traffic across instances/containers, does health checks and SSL termination. **ALB** = layer 7 (HTTP, routing by path/host); **NLB** = layer 4 (raw TCP, very high throughput).
- **Deploys:** rolling (replace instances gradually), blue/green (stand up new environment, switch traffic, easy rollback), canary (send a small % first).
- **CI/CD:** automated build → test → deploy. **You have real experience here** — Jenkins at Infobip, TeamCity at Irish Life, GitLab CI. Say that, then map it to GitHub Actions / AWS CodePipeline.

**Honest bridge:** *"I've run containers with Docker and built CI/CD pipelines in Jenkins and TeamCity; I understand the AWS building blocks — EC2/ECS/RDS behind a load balancer — and I'd be picking up the AWS-managed versions of patterns I've already used."*

**🔧 Real example:** how is the Propylon Django app deployed? Even "it runs in containers / has a CI pipeline in X" is a real, usable answer.

**Stronger version — on-prem vs AWS deploy model, reasoned from the actual constraint:** at Propylon, the servers are pre-configured — Python, dependencies, system libraries already installed — so a deploy is just shipping the RPM/app code onto a known environment. In AWS you can't assume the target server has *anything* pre-installed, so you containerize instead: Docker bundles app + runtime + dependencies into one image, so any server (or Fargate, with no server at all) can run it with zero pre-setup. **Say this:** *"On-prem at Propylon, the servers are already configured with Python and the right dependencies, so we just ship the RPM. In AWS you can't assume that — so you'd containerize: bundle the app, runtime, and dependencies into one Docker image so it runs identically anywhere, no pre-setup needed."*

**Strong follow-up, if asked "why doesn't Propylon just containerize too?"** — don't answer "modern is better," answer with the actual constraints: legacy/path dependency (it works, and it's a small, stable on-prem footprint, not a fleet), the operational overhead of running an image registry and managing image builds/versions for that small footprint isn't worth it, direct-SSH-onto-the-box debugging is simpler for a small ops team than working through a container layer, and on-prem often carries compliance/data-residency constraints that shaped the deploy model in the first place. **Frame it as a tradeoff, not a maturity gap:** *"Containers aren't universally better — they buy you portability and scalability at the cost of extra operational machinery (registries, image lifecycle). For a small, stable on-prem footprint with direct-access debugging and possible compliance constraints, the RPM-onto-a-known-server model is genuinely the simpler, right-sized choice. The right infra follows the constraints, not a 'newer is better' assumption."*

---

## AI — RAG pipeline, tokenisation (briefly)

- **RAG (Retrieval-Augmented Generation):** LLMs hallucinate and don't know your private documents. RAG **embeds the user's question → vector-similarity search over your document chunks → feeds the top matches into the LLM prompt → returns a grounded answer with citations** back to the source. You built this in DocIntel (pgvector for retrieval, LLM for synthesis) — speak from that.
- **Tokenisation:** LLMs don't process words or characters — they process **tokens** (subword units; roughly ~4 characters each in English, via byte-pair encoding). Why it matters practically: the **context window** is measured in tokens, **cost is per token**, and it drives your **chunking strategy** (how big each retrieved chunk is).

Keep both brief; you've got the honest "I built RAG against the Anthropic API; Bedrock is the same invocation pattern" bridge from the behavioural doc.

---

## FRONT-END — React 18 / TypeScript / MUI / AG Grid

> Their list had no React questions, but React 18 + TypeScript is a **named requirement** and this is a **fullstack** role — expect it to be probed. Your day-job front end on Montana is **Aurelia**, which has faded from mainstream use, so *don't lean on it as a credential.* Extract the transferable concepts and bridge to React, which you've actually built in DocIntel.
>
> **Honest frame:** *"Day-to-day on Montana I work in Aurelia — component-based, two-way binding, DI — so a lot of the concepts transfer. I've been building the React 18 side hands-on in DocIntel with TypeScript, MUI and AG Grid, deliberately, because it's the stack I want to be in."* (Consistent with credibility-first: real day-job front end + deliberate React upskilling, no overclaiming.)

### React fundamentals (most likely to be asked)
- **Props vs state**, and when you'd *lift state up*.
- **Virtual DOM / reconciliation** — what it does, and why **keys** matter in lists.
- **Controlled vs uncontrolled** components (form inputs). **DocIntel search box, reasoned out:** controlled — because you want to react to *every* keystroke (character count, styling, showing "typing…" state) while still deferring the *expensive* part (the actual fetch) to a debounce timer or a submit/Enter handler. Controlled input, cheap per-keystroke reactions, expensive action gated separately — that's the general shape of the answer, not just "controlled because forms are usually controlled."
- **Hooks** — what a hook is; walk `useState`, `useEffect`, `useContext`, `useRef`.
- **`useEffect` in depth** — the dependency array, the cleanup function, and the classic pitfalls (infinite loops, stale closures, missing deps). *This is the #1 "do they really know React" probe — be solid here.* ⚠️ **Not cold yet** — you got to the mechanism in a prep session (below) but only with prompting. Drill this until you can say it unprompted.

**🔧 Worked example — debounced search with `useEffect` cleanup:**
```jsx
useEffect(() => {
  const timer = setTimeout(() => fetchResults(query), 300);
  return () => clearTimeout(timer); // cleanup: fires before the *next* effect run
}, [query]);
```
Every keystroke changes `query`, which re-runs the effect — but first React calls the cleanup from the *previous* run, clearing that pending `setTimeout`. So a burst of keystrokes only ever schedules one live timer at a time; only a pause in typing (300ms with no new keystroke) lets a `setTimeout` survive long enough to fire and trigger the fetch. That's the mechanism: cleanup isn't "on unmount only," it's "before every re-run of the effect."
- **React 18 specifics** — automatic batching, `StrictMode`'s double-render in dev, and *awareness* of concurrent features (`useTransition`, Suspense). You don't need to have shipped these; knowing what they're for is enough.

### Hooks, state & data
- Local state vs **Context** vs an external store (Redux/Zustand) vs a data layer (**React Query/TanStack**) — when each is warranted. "Prop drilling" is the problem Context solves.
- Data fetching: loading / error / empty states, and **cancelling stale requests**.
- `useMemo` / `useCallback` — what they do and, importantly, **when *not* to reach for them** (premature optimisation is a red flag they may bait). *Your "don't use without measuring" instinct is already right — the gap was just the labels. Straighten those out:*

  | Hook | Memoizes | Useful for |
  |---|---|---|
  | `useMemo` | a **value** (result of a computation) | skipping an expensive recalculation between renders |
  | `useCallback` | a **function reference** | keeping a stable prop identity — only pays off paired with a `React.memo` child, so it doesn't re-render on every parent render |
  | `useEffect` | *(nothing — different tool)* | **side effects** after render (fetch, subscribe, timers) — not a memoization hook at all, don't lump it in with the other two |

  Say it as: "`useMemo` caches a value, `useCallback` caches a function, `useEffect` isn't memoization at all — it's for side effects. And I wouldn't reach for the first two without measuring; they have their own overhead."
- Writing a **custom hook** and why you'd extract one.

### TypeScript (also a named requirement)
- Typing props and `useState<T>()`; typing event handlers.
- `interface` vs `type`; when generics earn their place.
- **Why TS on the front end** — catching shape mismatches against the API contract. *Ties straight to your FastAPI/Pydantic backend typing story: typed end-to-end.*

### MUI & AG Grid (spec nice-to-haves — DocIntel is your evidence)
- **MUI:** theming, the `sx` prop, overriding/customising components.
- **AG Grid:** column definitions, cell renderers, sorting/filtering, and the big one — **client-side vs server-side row model and row virtualization for large datasets.** Few candidates have actually used AG Grid, so speak from what you built — it's a differentiator.

### Framework-agnostic front end (your real strength zone — lean in here)
Years of real front-end work show regardless of framework:
- JS **event loop**, promises, `async`/`await`; **debounce vs throttle**.
- CSS: flexbox/grid, box model, specificity, responsive layout.
- Browser rendering: **reflow vs repaint**, roughly the critical rendering path.
- What a **bundler** does; **code splitting / lazy loading** for performance.
- Front-end security: **XSS and CSRF** (connects to your JWT/auth knowledge).
- **Accessibility** basics (semantic HTML, ARIA) — increasingly asked.

### The Aurelia → React bridge (a genuine depth signal)
Being able to *compare* frameworks reads as senior. What transfers:
- Custom elements → React components; templating → JSX; routing → React Router; lifecycle (`attached`/`bind`) → `useEffect`.
- **The difference to state out loud:** Aurelia defaults to **two-way binding** and has **built-in DI**; React deliberately favours **one-way data flow** with no built-in DI (props/context/hooks). *"I found one-way flow makes state easier to reason about"* is a strong, specific line.
- **Lifecycle/cleanup, verbatim:** *"In Aurelia, I'd use lifecycle hooks like `detached()` to clean up when the component unmounts — unsubscribe from observables, cancel pending requests. In React, that's what the cleanup function in `useEffect` does. Same discipline, different syntax."*
- **Shared state/DI, verbatim:** *"In Aurelia, I'd use a shared service with dependency injection — inject it wherever I need it, no prop drilling. React doesn't have built-in DI, so Context fills that role."*

**🔧 Real example:** Montana Aurelia gives you transferable material even though the framework's dated — a component you built, a tricky bit of state/binding, a data-grid or form. Confirm what's true and frame it as a *concept* ("component with local state driving a filtered list"), then say you've done the equivalent in React in DocIntel.

---

## AUTH — JWT

> Not on their original list, but it connects directly to the **front-end security (XSS/CSRF)** line above and to the **JWT auth** hop in your own architecture diagram below — likely to surface as a follow-up either way.

### How JWT works

A JWT is three base64url segments joined by dots: `header.payload.signature`.

- **Header** — algorithm + token type (`HS256`, `RS256`, …).
- **Payload** — claims: `exp` (expiry), a subject/user id, plus whatever custom claims you add. This is *encoded*, not encrypted — anyone can decode and read it, so never put secrets in the payload.
- **Signature** — the part that actually protects it:
  - **HS256** — HMAC with one shared secret; whoever signs and whoever verifies need the same secret.
  - **RS256** — RSA keypair; the issuer signs with the **private** key, anyone holding the **public** key can verify but not forge.

Verifying is just: recompute/check the signature, check `exp` hasn't passed — no DB lookup, no session store required. That's what "stateless" buys you: no shared session state, scales horizontally.

**The tradeoff to name:** stateless means revocation is hard — you can't invalidate one token early without a server-side blocklist, because nothing is tracked at issue time. Short expiries + refresh tokens are the standard mitigation.

**🔧 Real example — Montana runs two JWT layers, nested:**

1. **Service-to-service:** the internal ORM-like client layer that talks HTTP to the central document service signs its requests with an **RS256** JWT — asymmetric, so only the auth side holds the private signing key and the document service just needs the public key to verify. Attached as `Authorization: Bearer <token>` on every call the client layer makes. Access tokens are long-lived (7 days), refresh tokens 30 days.
2. **User-facing browser session:** each front-end app (e.g. the chamber-facing app, or a committee-management app) does its own login and issues its own JWT to the browser — **HS256**, signed with Django's `SECRET_KEY`, stored in an httponly cookie, ~12 hour expiry.

**The nesting — this is the depth signal:** the browser-session JWT embeds the service-layer token *inside its own payload* as a claim. So when a logged-in user's browser hits a front-end app, middleware extracts that embedded claim and uses it to authenticate the ORM-like client's calls to the document service on the user's behalf — one JWT carrying another JWT, so the user's session transparently becomes a service credential without a second login.

> **The line to say:** *"We actually run two JWT layers — a long-lived RS256 token between our internal ORM-like client and the central document service, and a shorter HS256 session token per front-end app for the browser. The session token embeds the service token as a claim, so a logged-in user's session carries the credential needed to authenticate into the document service without a second login."*

**If pushed on hardening (don't volunteer, but be ready):** signing keys are static and committed in-repo rather than pulled from a secrets manager, and the service-to-service access token is long-lived (7 days) for what's normally a short-lived credential. Naming that unprompted as "what I'd change" reads as senior.

---

## YOUR CURRENT PROJECT — the architecture answer

> "Tell me about your current project" is near-certain, probably the first technical question. Deploy the 60-second version below, then **let their follow-ups pull the depth out of you** — depth that's extracted reads as real; depth that's dumped reads as rehearsed. **Never say the codenames** (Ascended, Joplin, APN) — use the plain-language names.

**The 60-second version:**

> *"I work on a legislative drafting and workflow platform — about twenty Django applications covering committee management, bill drafting, and chamber processes. The interesting architectural piece is that the apps don't share a database: everything goes through a central **versioned document service**, because in legislative work 'what did this bill say at the moment the committee voted' is a legal requirement, not a feature. The service stores documents as append-only revisions — you never update in place, you write revision N+1 — with check-out/check-in locking for editing and point-in-time reads across the whole object graph. Apps talk to it over HTTP through an **internal ORM-like client layer** that mimics the Django ORM API, and Postgres sits *inside* the service as its storage engine. The cost we pay is that every lazy field access is an HTTP round trip, not a SQL query — so batched eager loading matters even more than it does in Django."*

**The layering (for your own clarity, not to recite):**

```
mt-* apps (committee mgmt, drafting, chamber …)   ← no models.Model, no direct SQL
   └─ internal ORM-like client layer (Django-style Model/Manager/Q API)
        └─ HTTP (JWT auth, retries, connection pooling)
             └─ versioned document service (append-only revisions, locking, metadata, events)
                  └─ Django ORM → Postgres (its internal storage engine)
```

**Why not one shared database?** That's the *integration-database* antipattern: with ~20 apps on one schema, the schema becomes a public API you can never change, every migration is lockstep, and nothing stops one app corrupting another's invariants. Behind a service, the HTTP API is the contract and the internal schema evolves freely — and the invariants the domain can't afford to lose (immutable history, locking, audit) are enforced at the only door.

**Question hooks — where this answer gets deployed:**

| Likely question | The hook |
|---|---|
| "Tell me about your current project" | The 60-second version, verbatim |
| Monolith vs microservices / how do services share data | "We live this — twenty apps, no shared DB, one data service. The shared-schema alternative becomes an API you can never change." Then **name the cost** (HTTP hops, eager-loading machinery) — naming your own architecture's cost is the strongest senior signal in this interview |
| Slow API / N+1 | Already in your rehearsed answer — same algorithm as Django, worse penalty because each lazy access is an HTTP round trip |
| Two users edit the same record | The check-out/check-in locking example (now in that scenario below) |
| Audit / history requirements | Append-only revisions, never update in place, point-in-time reads — *"legislation and fund administration have the same requirement shape: regulated domains where you must prove what the data said at a moment in time"* — your best bridge to Citco's world |
| SQL vs NoSQL | "My day job is literally both" — relational Postgres and a versioned document store (already in that section) |

---

## TECH DESIGN SCENARIOS

These are where "plain language, real problem" matters most. The interviewer isn't after buzzwords — they want your **reasoning**: *measure first, find the real bottleneck, pick the simplest fix that fits.* Structure each answer as **diagnose → options → tradeoff → what I'd pick.**

> **How to use the Python topics here (your friend's point):** the topics above aren't a trivia checklist — they're the *toolbox* you reach into to answer these scenarios. Each scenario below now starts with a **🧰 Python topics in play** line listing exactly which earlier sections it pulls from. Learn to *name the tool as you use it*: "this is I/O-bound, so..." / "I'd `EXPLAIN ANALYZE` first..." / "I'd move it to a Celery task...". That's how you turn topic knowledge into a design answer.

### "We have a slow API request — what do you do?"

> **🧰 Python topics in play:** **EXPLAIN ANALYZE** + **Indices** (the missing-index case) · Django ORM / **middleware** knowledge (N+1, `select_related`) · **Cache — in-memory vs Redis** (expensive computation / external calls) · **Background tasks (Celery)** (move slow work off the request) · **Concurrency** (a slow external call → make it async, or overlap several with threads).

**Core to lead with (4):** measure first (Irish Life story) → N+1 → `select_related`/`prefetch_related` → over-fetching → paginate + select only needed columns (SaunaGuide story) → re-measure after. *(Cache/background-job/external-call branches: one-liners only, don't over-rehearse.)*

1. **Measure first** — don't guess. Profile / log / APM to find where the time actually goes. (This is literally your Irish Life OLS story: you *measured method execution times* to find the bottleneck — reuse it.)
2. Then match fix to cause:
   - **N+1 queries** → `select_related` / `prefetch_related` in Django (eager-load instead of one query per row). *(Day-job tie-in now woven into the rehearsed answer below; full story in the Django ORM section.)*
   - **Missing index** → `EXPLAIN ANALYZE`, add the index.
   - **Over-fetching** → paginate; select only the columns you need. *(Two real examples: (1) your OLS fix — a service returning more data than the page needed. (2) SaunaGuide — the homepage loaded every listing on each request; I added `Paginator` (10/page) + HTMX infinite scroll (`hx-trigger="revealed"` sentinel) that returns only the next card partial. **And column-level:** for the map markers I switched to `.values(...)` specifically to avoid loading a heavy base64 `photo_data` image blob on every row — that's over-fetching **columns**, not just rows, which is the more sophisticated version of this answer.)*
     - **A step further — is the blob's home the design smell, not just the query?** Storing large binary data (images, files) alongside relational metadata is a separate problem from column-level over-fetching, and worth naming if the conversation goes there. Options, in order of increasing complexity — pick the simplest that fits the data's size, count, and access pattern:
       1. **Keep it in the DB** if it's small, there's one per row, and it's rarely fetched in bulk. *(SaunaGuide's own base64 `photo_data` is genuinely this case — a single web-optimized image per listing.)*
       2. **Split into its own table** (e.g. a `ListingPhoto` table, FK back to `Listing`) once it's one-to-many — several images per row shouldn't live in columns on the parent.
       3. **Move it to S3, keep only a URL column** once size or volume grows — decouples heavy file I/O from the DB entirely (mirrors the presigned-URL upload/download pattern elsewhere in this doc). Costs real operational overhead: lifecycle sync, orphaned files if a delete doesn't clean up both sides, per-request cost at scale, and consistency risk if the file and the DB record drift apart.
       - **Say this:** *"Depends on the data's shape and scale — measure size, count, and access pattern, then pick the simplest thing that fits. A single small image per row can live right in the DB; one-to-many wants its own table; once volume or size grows, move the bytes to S3 and keep just a URL, accepting the sync/consistency overhead that comes with a second store."*
   - **Expensive computation on the request path** → cache it, or move it to a background task.
   - **Slow external call** → cache, set timeouts, or make it async.
3. **The line to say:** *"I don't optimise blind — I measure, fix the biggest bottleneck, then re-measure."*
- **Front-end half (fullstack answer):** loading skeletons, optimistic UI, client-side caching (React Query), pagination / infinite scroll, debounced search, and **cancelling superseded requests** so a slow earlier response doesn't overwrite a newer one.

> **📝 My full answer (rehearsed):**
>
> *"I don't optimise blind — first I measure, using logs/APM/timing, to find where the time is actually going, the same way I found the bottleneck in the Irish Life OLS load. Then I match the fix to the actual cause:*
>
> - *If it's an ORM problem — one query turning into N, one per row, because a related object is lazily fetched — take a bill and its sponsors from my day job: loop over 100 bills touching `bill.primary_sponsor` and each access lazily fires another query, so 1 query becomes 101. I fix it with `select_related` for a foreign key like the primary sponsor (one query via a JOIN) or `prefetch_related` for a many-to-many like the co-sponsors (two queries stitched in Python, so a JOIN doesn't multiply the rows out). And I've seen this beyond Django: at Propylon we have an internal ORM-like layer over our legislative document store, and it has literally the same two methods — its `select_related` asks the datastore to return the related asset inline in the same response, its stand-in for a JOIN, and its `prefetch_related` collates every reference the queryset needs, batch-fetches each relation once, and stitches them back in by key. Same algorithm as Django — and it matters even more there, because each lazy access is an HTTP round trip, not just a SQL query. N+1 isn't a Django quirk; it's what lazy loading does anywhere.*
> - *If it's a missing index — I run `EXPLAIN ANALYZE`, see a sequential scan on a column I'm filtering on, and add an index for that query path. That's a database fix, not a Python one.*
> - *If it's over-fetching, there are two levels. Row-level: paginate — I did exactly this on SaunaGuide, where the homepage originally loaded every listing on every request; I added Django's `Paginator` at 10 per page plus HTMX infinite scroll, so the server only ever returns the next page of cards. Column-level: for the map view on the same page, I switched to `.values(...)` so Postgres never even selects the heavy base64 image blob stored per listing — I only need name/lat/lng for a pin, not the photo.*
> - *If it's expensive computation sitting in the request path, I move it to a background job — Celery — so the request returns immediately instead of the worker blocking on it.*
> - *If it's a slow external call, I either cache the result if it repeats, set a timeout so it fails fast, or — if I need to overlap several such calls — decide between threading (a modest number of blocking calls, bolt-on, no rewrite) and async (many concurrent calls, needs an async-native stack throughout).*
>
> *Whatever I change, I re-measure afterwards to confirm it actually moved the number — I don't assume a fix worked.*
>
> *On the front end, the same problem needs a different toolkit: loading skeletons so the layout doesn't jump and the wait feels shorter, optimistic UI where the action is low-risk and reversible, client-side caching (React Query) so a repeat view doesn't refetch, pagination/infinite scroll so I'm never asking for more than the user can see, debounced search so I'm not firing a request per keystroke, and cancelling superseded requests so a slow earlier response can't overwrite a newer one that arrived first."*

> **Follow-up: "What's the actual difference between `select_related` and `prefetch_related`, and why can't you use `select_related` for a many-to-many?"**
>
> *"`select_related` does a SQL JOIN and pulls the related row back in the same query — that only works for foreign key / one-to-one, where there's exactly one related row per row. `prefetch_related` runs a second query for the related set and stitches it back together in Python by key — needed for many-to-many or reverse FK, where a JOIN would multiply the base rows out one-per-match instead of returning them once each."*

### "We want a feature that generates a big report — what do you do?"

> **🧰 Python topics in play:** **Background tasks (Celery / SQS)** — this *is* the answer, so the whole section is the toolbox · **Concurrency** (report gen is usually I/O-bound: DB reads + file writes → threads/async, not multiprocessing — unless it's CPU-heavy number-crunching, then a worker process) · **Cache** (cache the finished report so a re-request is instant) · **Databases** (run it against a **read replica** so it doesn't load the primary).

**Core to lead with (4):** never in-request → offload to Celery, return `202` + job ID → poll/notify → store result in S3, hand back a presigned link → chunk with `.iterator()` so memory stays flat. *(Idempotency via the `Report` table + unique constraint: hold in reserve for the follow-up, don't cram into the opener.)*

**Default answer:**
- **Never generate it inside the request** — it'll block a worker and time out.
- **Offload to a background job** (Celery / SQS). Return immediately with a job ID (`202 Accepted`).
- Client **polls for status** or gets **notified** (websocket / email / webhook) when it's ready.
- **Store the result** (a file / S3) and hand back a download link.
- For very large data: **stream / paginate / generate in chunks** so you never hold it all in memory.
- **Front-end half:** never block the UI — disable the button to prevent double-submit, poll for job status (or subscribe), show progress, then surface a **download link** when it's ready.

**Only if probed further (double-submit / idempotency, read replica) — don't lead with these:**
- Consider running the report query against a **read replica** so reporting doesn't load the primary DB.
- **The `Report` table is the mechanism** behind idempotency *and* caching: a row per requested report — `user`, `params_hash`, `task_id`, `status`, `file_key`, `created_at`. On `POST`: hash the params, look up the row. In-flight → return its existing `task_id`. Done and data still fresh → return the download link, no job at all. No row → insert one and enqueue. A **unique constraint on `(user, params_hash)`** (scoped to active rows) closes the race where two simultaneous requests both find no row. This is your strongest depth point but a senior-signaling one — save it for the follow-up, don't cram it into the opener.

> **📝 My full answer (rehearsed, mid-level pitch):**
>
> *"The one thing I'd never do is generate it inside the request — a big report will block a web worker and time out long before it finishes. This is really a background-jobs problem with a notification problem attached.*
>
> - *So the view's only job is to enqueue: `POST /reports/` calls `generate_report.delay(params)` and immediately returns `202 Accepted` with the task ID. Celery generates that ID the moment I call `.delay()` and pushes the message onto the broker — Redis — which is the queue of pending work; a worker picks it up whenever it's next free. Task status and results live separately in the result backend. Same Redis, two distinct roles — broker and result backend — and neither of those is caching.*
> - *Before I parallelise anything, I make sure the report's queries are actually efficient — `EXPLAIN ANALYZE`, indexes on the filter paths, and select only the columns the report needs. On SaunaGuide I used `.values(...)` for exactly this reason — to stop Postgres pulling a heavy image blob on every row when I only needed three fields.*
> - *Inside the worker I generate in chunks — iterate the queryset with `.iterator(chunk_size=...)` and write rows to the file as I go, so memory stays flat whether it's ten thousand rows or ten million. The user still gets one file; the chunking is invisible to them.*
> - *Report generation is usually I/O-bound — DB reads and file writes — so if I need to overlap several independent queries I'd use threads or async inside the task. If it's genuinely CPU-heavy number-crunching, Celery's prefork pool is already multiprocessing, so I scale worker processes instead — true parallelism, no GIL issue across processes.*
> - *The finished file goes to S3 — object storage, not the DB and not Redis — keyed by the report parameters, with a row in the DB recording params, status, and file key. The client gets a presigned URL, a time-limited signed link, so the download goes straight from S3 to the browser and never transits my app.*
> - *For "how does the client know it's done": polling every couple of seconds is honestly fine for job status — I'd only reach for SSE if I wanted the server to push progress events, and WebSockets only if I genuinely needed bidirectional. Simplest thing that works.*
>
> *On the front end: submitting kicks off the job, so the button disables and the UI shows the job as pending — the status is just React state. A `useEffect` sets up the polling interval and cleans it up on unmount, each poll updates state, state drives the progress indicator, and when the status flips to done I render the download link from the presigned URL. The user can keep using the app the whole time — nothing blocks."*

> **Follow-up: "How do you stop two clicks on submit from kicking off two separate report jobs?"**
>
> *"A `Report` row per requested report — user, a hash of the params, task ID, status, file key. On POST I hash the params and look up the row: in-flight → return its existing task ID instead of enqueuing again; done and fresh → skip the job, return the download link; no row → insert and enqueue. A unique constraint on `(user, params_hash)` closes the race where two simultaneous requests both see no row and both try to insert. Disabling the button helps the UX but I never rely on the client to enforce it."*
>
> **Follow-up: "What if reporting load starts hurting live traffic?"**
>
> *"Point the report queries at a read replica — writes keep going to the primary, the replica absorbs the heavy reads. The one caveat I'd name is replication lag: fine for a report, wrong for a read-your-own-write flow."*

### "We want real-time notifications to users / a live progress bar — how?"

> **🧰 Python topics in play:** **Concurrency** (WebSockets / Django Channels run on **async** — a persistent connection per client is exactly what an event loop is for, cheaper than a thread each) · **Cache — Redis** (the background task writes progress to Redis; the client reads it — and Redis **pub/sub** is what fans a notification out to many connections) · **Background tasks** (the long job is what's *reporting* the progress in the first place).

**Core to lead with (3):** match the tool to the need, lightest that works — polling is fine for a bar → worker writes progress to Redis, client reads via poll or SSE → WebSockets only if genuinely bidirectional.

Match the tool to the need — lightest thing that works:
- **Polling** — client asks every few seconds. Simplest, works everywhere, more load. Fine for a progress bar.
- **Server-Sent Events (SSE)** — server pushes one-way over HTTP. Great for notifications and progress bars.
- **WebSockets** (Django **Channels**) — full bidirectional, persistent connection. Use when you genuinely need two-way real-time (chat, live collaboration).
- **Progress bar pattern:** the background task writes its progress to Redis/cache; the client polls or reads it via SSE.
- **Say this:** *"A progress bar doesn't need WebSockets — polling or SSE is simpler. I'd only reach for WebSockets when I need true bidirectional real-time."*
- **Front-end half (this one is mostly client-side):** on the client it's polling vs `EventSource` (SSE) vs WebSocket, and how you wire it — a background job writes progress somewhere, the component reads it via polling/SSE and updates state to drive the bar. **Toasts** for notifications.
- **The client never talks to Redis directly** — it's not internet-facing and doesn't speak HTTP. The client always talks to the API; the API is just doing a cheap Redis read instead of a DB query or a call to the worker.

> **📝 My full answer (rehearsed):**
>
> *"A progress bar doesn't need WebSockets — it needs the background job to report where it's at, and something for the client to read that from. Client submits, gets a task ID back, same pattern as the report case. From there the client only ever talks to the API — never to Redis directly, that's not exposed.*
>
> *The worker writes its progress into Redis as it runs — a key like `progress:{task_id}` holding something like `{"pct": 40}`, updated periodically, not on every row. For a plain progress bar the client just polls `GET /jobs/{id}/status` every couple of seconds, and that endpoint's entire job is one Redis `GET` translated to JSON — it never touches the DB and never talks to the worker process directly. Redis is the shared state sitting between the two.*
>
> *If I want push instead of poll — real notifications, not just a bar — I'd hold the connection open with SSE, and the worker `PUBLISH`es progress to a Redis channel instead of just writing a key. The SSE view `SUBSCRIBE`s to that channel and forwards each message as it arrives. That pub/sub is also what lets one event fan out to more than one listener — two tabs open on the same job, or a chat message that needs to reach several connected clients at once. I'd only reach for WebSockets if the client also needed to send data back over that same channel in real time — a progress bar doesn't."*

> **Follow-up: "Doesn't polling from thousands of clients hammer Redis or your API?"**
>
> *"Not really — each poll is a single Redis `GET`, sub-millisecond, and Redis handles tens of thousands of ops a second on modest hardware. What would actually hurt is if that endpoint touched Postgres or the worker instead. If it ever did become a problem, I'd lengthen the poll interval or move to SSE/pub-sub so it's push instead of N clients re-asking — but for one progress bar, polling a Redis key is cheap enough that I wouldn't pre-optimise it."*

### "We'll process millions of rows a day in the DB — how do you prepare?"

> **🧰 Python topics in play:** **Indices and their types** (this is where **BRIN** for append-only time data, **partial**, and **composite/leftmost-prefix** actually earn their keep) · **EXPLAIN ANALYZE** (prove which query paths need indexing instead of guessing) · **SQL vs NoSQL** (the "does analytics belong in the transactional DB or a warehouse?" premise-question) · **Background tasks** (bulk ingest via `COPY`/`bulk_create` in workers, never row-by-row in a request) · **Concurrency** (if each row needs CPU-heavy parsing, that's the **multiprocessing** case — true parallelism for CPU-bound work).

**Core to lead with (3, mid-level pitch):** index write-heavy query paths selectively → partition/archive old data out of the hot table → read replica for reporting. Frontend: virtualize + paginate, never render millions of DOM nodes. *(BRIN specifics, PgBouncer, autovacuum, and "question the premise" are real but staff-flavored — keep in reserve below, don't volunteer them in the opener.)*

**Default answer:**
- **Index the query paths** — but not everything; each index slows writes, and you're write-heavy here.
- **Partition big tables** (e.g. by date/month) so queries scan less and old partitions can be dropped cheaply.
- **Bulk operations** — `COPY` / `bulk_create`, batched, never row-by-row inserts.
- **Retention / archiving** — move old data out of the hot table.
- **Read replicas** for heavy read/analytics; keep the primary for writes.
- **Front-end half (your strongest front-end answer):** you **never render millions of DOM nodes — you virtualize/window** (render only the visible rows) and do server-side pagination/filtering so the client only fetches what's on screen. **AG Grid's server-side row model does exactly this** — worth knowing conceptually even though you haven't wired it up in DocIntel yet; say "I haven't shipped it there yet, but that's the model I'd reach for" if pressed.

**Only if probed further ("anything else?" or a specific follow-up) — don't lead with these:**
- **BRIN indexes** specifically for append-only time-ordered data (vs a plain B-tree).
- **Connection pooling** (PgBouncer) so millions of short connections don't exhaust the DB.
- **Tune autovacuum** (high write/update volume creates dead tuples that bloat tables).
- **Question the premise:** does *all* of it need to live in the transactional DB, or should analytics go to a warehouse? A strong point, but a senior-signaling one — deploy it only if the conversation has room for it, not as bullet #1.

> **📝 My full answer (rehearsed, mid-level pitch):**
>
> *"First I'd separate writes from reads, because they pull in different directions. On the write side: index only the query paths that are actually reused, since every index costs you on insert — you're write-heavy here so I'd be deliberate about which ones earn their keep. I'd also partition large tables by time, so old data can be archived or dropped cheaply instead of bloating one giant hot table, and keep ingestion as bulk operations — COPY or batched inserts — never row-by-row.*
>
> *On the read side, I'd put reporting and analytics queries against a read replica instead of the primary, so heavy reads don't compete with the writes.*
>
> *On the frontend, the same problem shows up as 'don't render millions of DOM nodes.' I hit exactly this on SaunaGuide — loading every sauna listing on the page was slow — so I paginated it, 10 at a time, and loaded the next 10 when the user scrolled to the bottom. AG Grid's server-side row model is the productionized version of that same idea: it virtualizes rows and pushes sorting/filtering/pagination to the server instead of holding the whole dataset in the browser."*

> **Follow-up: "Anything else you'd consider?" / "Why BRIN specifically, not a normal B-tree, for time-ordered data?"**
>
> *"A few things I'd add if this kept scaling: a connection pooler like PgBouncer, since real Postgres connections are expensive processes and bursts of short app connections can exhaust the DB. I'd keep an eye on autovacuum, since high write volume creates dead tuples that bloat tables if it's not tuned. For the time-ordered partitions specifically, a BRIN index suits them well — it just stores the min/max per block rather than indexing every row's exact value like a B-tree, so it's much smaller and cheaper to maintain when the data is naturally sorted by insertion time. And I'd genuinely ask whether all of this needs to live in the transactional database at all, or whether some of it belongs in a separate analytics warehouse — keeping the OLTP database lean is often the bigger win than tuning around a database that's doing two jobs."*

### "Two users edit the same record at the same time — how do you handle it?"

> **🧰 Python topics in play:** **Transactions, ACID & locking** — this *is* the answer · **Context managers** (`transaction.atomic()` + `select_for_update()`) · **SQL vs NoSQL** (strong consistency is exactly why the transactional data is relational).
>
> *Highest-probability new scenario at a fund administrator — concurrent updates to financial records is their daily reality.*

**Core to lead with (3):** ask first — how likely is a conflict, how bad is losing an edit → optimistic (version column, cheap, nothing blocks) vs pessimistic (`select_for_update`, right call for money) → name last-write-wins as the rejected default, not an accident.

**Model answer structure (this scored better live than a one-strategy answer):** name **both** strategies → state the real-world constraint that decides between them → give a day-job example of the strategy you *didn't* pick, for contrast → land on your pick and why. Structure > either answer alone.

- **First question to ask out loud:** how likely is a conflict, and how bad is silently losing an edit? That decides the strategy.
- **Optimistic locking** (conflicts rare): a `version` column; the update runs `WHERE id = ? AND version = ?`; zero rows updated means someone else won — return a conflict and let the user reload/merge. Cheap, nothing blocks.
- **Pessimistic locking** (conflicts likely, or the update must be serialized): `select_for_update()` inside `transaction.atomic()` — the second writer blocks until the first commits. Right call for anything touching money or stock.
- **Last-write-wins** is what happens if you do nothing — name it as a choice you're *rejecting*, not an accident.
> **Say this (strengthened, fund-admin-specific):** *"There are two strategies here. Optimistic — a version column, update `WHERE id = ? AND version = ?`, zero rows updated means someone else got there first. Pessimistic — `select_for_update()` inside a transaction, so the second writer blocks until the first commits. For fund administration specifically, I'd lean optimistic: this kind of record update is typically low-contention — conflicts are rare — so there's no reason to make every writer queue for something that will almost never collide. Contrast that with my day job, which is the opposite case: Propylon uses pessimistic check-out/check-in locking for legislative documents, because there conflicts are common — two drafters editing the same bill at once — and a merge would be meaningless, so we serialize instead. Same two tools, different constraint, different pick."*
- **Front-end half:** send the version the user *loaded* along with their save; on a 409 show a conflict UI (reload / show what changed) instead of silently overwriting. Disabling the save button only prevents double-submit from the *same* user — it does nothing for two different users.

**🔧 Real example (day job):** Montana runs *real* pessimistic locking — documents are **checked out / checked in** through the document service (`LockRecord`/`LockHistory` in the datastore), including from Word via the VSTO add-in. Two drafters silently merging edits to a bill is unacceptable, so the lock is explicit and held for the whole editing session — far longer-lived than a `select_for_update` row lock, same principle. **The line:** *"My day job literally runs on check-out/check-in locking — that's the pessimistic end of the spectrum, right for documents where a merge is meaningless. For rare, low-stakes conflicts I'd use an optimistic version check instead — no reason to make users queue when they'll almost never collide."* (Say "our document service", not the internal codenames.)

> **Follow-up: "What does `select_for_update` actually do at the DB level — does it block reads too, or just writes?"**
>
> *"It takes a row-level lock that blocks other writers — and other `select_for_update` readers — from touching that row until the transaction commits. A plain read without `FOR UPDATE` isn't blocked; it just sees the last-committed version, not the in-flight change."*

### "We depend on a slow / flaky third-party API — what do you do?"

> **🧰 Python topics in play:** **Concurrency** (a hung call with no timeout blocks a worker — the I/O-bound story again) · **Background tasks** (move the call off the request path; retry from the queue) · **Cache — Redis** (serve the last good response when the provider is down).

**Core to lead with (3):** timeouts, always, non-negotiable → retry with backoff, only if idempotent → circuit breaker after N failures, fail fast instead of queueing doomed calls.

- **Timeouts, always** — a call with no timeout eventually hangs every worker you have. This is the non-negotiable first line.
- **Retries with exponential backoff + jitter** — but only for idempotent calls; retrying a non-idempotent call (a payment) is how you double-charge someone. (Links to the idempotency scenario below.)
- **Circuit breaker** — after N consecutive failures, stop calling for a cooldown and fail fast, instead of queueing doomed calls behind a dead provider.
- **Off the request path** — if the user doesn't need the result synchronously, enqueue it (Celery) and let the worker absorb the retries.
- **Degrade gracefully** — cache the last good response and serve it stale with a notice, rather than erroring the whole page.
- **Say this:** *"Timeout first, retry with backoff only if it's idempotent, circuit-break if it keeps failing, and where possible move the call behind a queue so the user never waits on someone else's uptime."*

> **Follow-up: "Walk me through the circuit breaker states."**
>
> *"Closed — calls flow normally. After N consecutive failures it opens — fails fast, no calls even attempted, for a cooldown period. After the cooldown it goes half-open and lets one test call through: success closes it again, failure re-opens it and resets the cooldown."*

### "Users need to upload large files — how?"

> **🧰 Python topics in play:** **Cloud — S3 / presigned URLs** (the same presigned pattern as the report download, in reverse) · **Background tasks** (post-upload processing) · **Generators** (if the server must touch the bytes, stream them — never hold the file in memory).

**Core to lead with (3):** never proxy bytes through your app — presigned S3 URL, client uploads direct → validate cheap stuff (type/size/permissions) before issuing the URL, heavy stuff (virus scan, parsing) after, in a background task → multipart upload for very large/resumable files.

- **Don't proxy the bytes through your app server** — issue a **presigned S3 URL** and let the client upload directly to S3. Your app handles a tiny metadata request; the heavy bytes never transit it. Mirror image of the report-download answer.
- **Validate cheap things before** issuing the URL (type, declared size, permissions); **process after** — virus scan, parsing, thumbnailing — in a queued background task triggered once the upload lands.
- **Very large files:** S3 multipart upload — parallel chunks, resumable after a dropped connection.
- Record an upload row (who, key, status) so the async processing has something to update — same job-status pattern as the report.
- **Say this:** *"Presigned URL, client uploads straight to S3, my app only ever handles metadata — then a background task processes the file and updates its status."*
- **Front-end half:** progress bar from the upload's progress events, chunked/resumable for big files, client-side type/size check as UX (server still enforces — never trust the client).

> **Follow-up: "What if the upload succeeds to S3 but the client never tells your app?"**
>
> *"Two options, not mutually exclusive: an S3 event notification triggers the post-processing directly — S3 to SQS or Lambda — so it doesn't depend on the client at all. Or a periodic reconciliation job checks for upload rows stuck in 'pending' past a threshold and either re-checks S3 for the object or flags it."*

### "An endpoint is getting hammered — how do you rate limit / protect it?"

> **🧰 Python topics in play:** **Cache — Redis** (the counters must live in a *shared* store — this is exactly the per-process vs Redis distinction) · **Middlewares** (where DRF throttling plugs into the request pipeline) · **Cloud — load balancer/WAF** (defence in depth at the edge).

- **Where to limit:** edge (WAF / load balancer) for abuse and bots; app layer (DRF throttle classes) for per-user fairness. Both, ideally.
- **How:** a counter in Redis keyed by user-or-IP + time window. Fixed window is simplest; token bucket smooths bursts. It must be Redis, not process memory — with multiple app servers a local counter undercounts by a factor of N.
- Return **429 Too Many Requests** with a `Retry-After` header so well-behaved clients back off.
- **Question the premise too:** *why* is it hammered? If it's legitimate traffic, the answer might be caching the hot response, not throttling users.
- **Say this:** *"Redis counter per user per window at the app layer, WAF rules at the edge, 429 with Retry-After — and if the traffic's legitimate, I'd rather cache the response than throttle real users."*
- **Front-end half:** honour `Retry-After`, back off and queue retries instead of hammering harder; debounce inputs that fire requests.

### "A request or webhook can arrive twice — how do you avoid double-processing?"

> **🧰 Python topics in play:** **Transactions & locking** (a unique constraint + atomic insert is the real mechanism) · **Background tasks** (Celery/SQS are **at-least-once** delivery — duplicates are guaranteed eventually, not a rare accident) · your rehearsed **big-report** answer already does this (return the existing job ID for duplicate params).

- **Why it happens:** client retries after a timeout (the request *succeeded* but the response was lost), webhook providers redeliver on non-200, and queues are at-least-once by design.
- **Idempotency key:** the caller sends a unique key per logical operation (Stripe-style header, or the webhook's event ID); you store it in a table with a **unique constraint**. Duplicate arrives → the insert conflicts → return the *original* result instead of re-processing.
- **The constraint is the guard, not a lookup** — "check if seen, then act" without a unique constraint is a race between two concurrent duplicates. The DB constraint makes it atomic.
- Wrap the key-insert and the side effect in one `transaction.atomic()` so a crash can't record the key without doing the work (or vice versa).
- **The fund-administration framing:** "the payment got processed twice" is the disaster scenario in this domain — design so a replay is *safe*, don't hope it never happens.
- **Say this:** *"Retries and queues make duplicates inevitable, so I make the operation idempotent: an idempotency key with a unique constraint, insert-and-process in one transaction, and a duplicate just gets the original result back."*

### "We need search over a lot of records — how?"

> **🧰 Python topics in play:** **Indices — GIN / tsvector** (Postgres full-text — your real DocIntel example) · **EXPLAIN ANALYZE** (prove why `ILIKE '%term%'` seq-scans) · **SQL vs NoSQL** (when a dedicated search engine earns a second data store).

- **Why naive search dies:** `ILIKE '%term%'` has no left-anchored prefix, so a B-tree can't help — every query is a sequential scan.
- **Postgres full-text first:** `tsvector` column + **GIN** index — stemming, ranking, phrase search, zero new infrastructure. The right answer until requirements outgrow it.
- **Elasticsearch / OpenSearch** when you need fuzzy matching, typo tolerance, faceting, or relevance tuning at serious scale — and name the cost: a *second system* you must keep in sync with the source of truth and operate.
- **Say this:** *"I'd start with Postgres full-text and a GIN index — I've done exactly this in DocIntel — and only reach for Elasticsearch when requirements outgrow it, because it's a second data store to sync and run."*
- **Front-end half:** debounced input, cancel superseded requests (a slow response for "fun" must not overwrite results for "fund"), highlight matched terms.

---

## Quick pre-interview checklist
- [ ] Can I explain the **GIL** and the CPU-bound vs I/O-bound decision in one breath?
- [ ] Have I actually run **`EXPLAIN ANALYZE`** on a real query and seen a scan flip to an index?
- [ ] Do I have **3–4 true Propylon (or DocIntel/Irish Life) examples** ready to attach to concepts — not invented?
- [ ] For every scenario: **diagnose → options → tradeoff → my pick.**
- [ ] Can I explain **optimistic vs pessimistic locking** (`version` column vs `select_for_update`) and say when each fits?
- [ ] Can I give the **idempotency-key + unique-constraint** answer in two sentences, and say *why* duplicates are inevitable (retries + at-least-once queues)?
- [ ] Am I bridging cloud/AI gaps **honestly** ("I've done X, the AWS version is the same pattern") rather than overclaiming?
- [ ] Can I explain **`useEffect`'s dependency array + cleanup** and one pitfall, cold?
- [ ] Do I have the **Aurelia → React bridge** line ready (two-way binding + DI vs one-way flow), plus a real Montana example framed as a *concept*?
- [ ] Can I explain **row virtualization / server-side row model** for large grids from DocIntel?
- [ ] Have I actually **built** the debounced search in DocIntel with real `useEffect` cleanup — not just understood it in a prep session?
- [ ] Have I actually written and timed a **`ThreadPoolExecutor`** example, not just described the concept?
- [ ] Can I explain **why decorators beat calling a function directly** (avoiding repeated boilerplate across every view), not just define what a decorator is?