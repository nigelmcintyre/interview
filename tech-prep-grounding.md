# Tech Prep — Grounded in the Montana / Propylon Monorepo

> Companion to `citco-technical-prep.md`. Searched first-party app code only
> (Django apps, the `joplin`/`ascended` datastore libs, DRF serializers,
> Celery tasks, CI configs) across all `mt-*` repos + `lrms-core` under
> `/home/nigel/montana`. Excluded venvs, `node_modules`, migrations (except
> where a migration itself is the evidence), and vendored code (e.g.
> `diablo/bottle.py` is a vendored micro-framework, not ours).
>
> **Authorship note:** your git identity in these repos is
> `nigelmcintyre <nigelmcintyre1995@gmail.com>`. Across the whole monorepo you
> have a small number of commits relative to total history (e.g. 8/1405 in
> `mt-cm-common-plugin`, 9/2244 in `mt-bde-law-making`) — most concentrated in
> `mt-cm-common-plugin`, `mt-bde-law-making`, `mt-chamber-interface`,
> `mt-committee-management`, `mt-mytask-dashboard`, `mt-models`, `mt-doc-api`,
> `mt-devops`, `lrms-core`. Every citation below states whose code it is —
> don't claim authorship the blame doesn't support.

---

## PYTHON

### Concurrency — multiprocessing / multithreading / async

✅ **FOUND** — threading (I/O-bound fan-out), not authored by you.

**File:** [mt-cm-common-plugin/src/cm/cm_core/utils.py:100-136](../mt-cm-common-plugin/src/cm/cm_core/utils.py#L100-L136)
```python
class WorkerThread(threading.Thread):
    """Custom thread which raises exceptions should they be raised."""
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self._real_run = self.run
        self.run = self._run_wrapper
        ...

class WorkerThreadPool(ThreadPoolExecutor):
    """Custom thread pool which raises exceptions should they be raised."""
```

**Real usage, in a DRF view, fan-out/fan-in:** [mt-cm-common-plugin/src/cm/cm_core/views/committee.py:141-168](../mt-cm-common-plugin/src/cm/cm_core/views/committee.py#L141-L168)
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

**How to tell the story:** committee creation needs three independent writes
against the datastore (witnesses, members, meetings). Standard `threading.Thread`
doesn't propagate exceptions to the joining thread by default, so there's a
custom `WorkerThread` subclass that wraps `run`/`join` to capture and re-raise.
This is a textbook **I/O-bound** case — the GIL is released while each thread
waits on the datastore — so threads (not multiprocessing) are the right tool.
`WorkerThreadPool` (a `ThreadPoolExecutor` subclass with the same
exception-propagation fix) is defined alongside it but **I found no call site
that actually instantiates it** — the codebase consistently uses the manual
`start()`/`join()` pattern via `WorkerThread`, not the pool. Say that
precisely if asked — don't imply the pool is in active use.

This exact `WorkerThread` pattern also drives much heavier fan-out in
[legacy_meeting_minutes.py](../mt-cm-common-plugin/src/cm/cm_core/legacy_meeting_minutes.py)
(dozens of call sites processing attendees, exhibits, votesheets in parallel).

**Whose code:** `git blame` on the class → Robert Lucey, Nenad Pavlovic, Brant
Watson. `git log` on the whole file → Brendan Salmond, Robert Lucey, Bryan
Spence, others. **You have not touched this file.** Present it as "real code
in a package I work in," not "code I wrote" — be ready for "did you write
this?" with an honest "no, but I can explain what it does and why."

No genuine `multiprocessing` (CPU-bound) or `asyncio`/`async def` usage found
anywhere in first-party code — the codebase is synchronous Django/DRF +
Celery, consistent with the doc's own framing ("classic Django is
synchronous... heavy work pushed to Celery").

---

### Types

✅ **FOUND** — type hints, **authored by you**.

**File:** [mt-chamber-interface/src/mt_chamber_interface/ci_api/serializers/mixins.py:88](../mt-chamber-interface/src/mt_chamber_interface/ci_api/serializers/mixins.py#L88), also lines 68, 135
```python
def _engross_for_sponsors(self, bill: Bill, initial_data: dict):
    ...
    if getattr(bill, 'long_draft_number', None):
        try:
            bill.long_draft_number = str(int(bill.long_draft_number)).zfill(4)
        except ValueError:
            pass
```
This is inside `SponsorManagementSerializer(serializers.Serializer)` — a DRF
serializer.

**Whose code:** `git show cc33db33` (commit `cc33db33`, 2026-03-11,
"BDE-344--Update co sponsors in HB2 doc with highest version number when
edited from CI") — **that's your commit**, in a function that already carried
`bill: Bill, initial_data: dict` type hints.

**How to tell the story:** two things worth saying. First, the hints
(`bill: Bill`, `-> bool`, `-> list` elsewhere in the same file) are
documentation only — nothing stops you passing the wrong type, consistent
with "dynamically but strongly typed." Second, and more interesting: DRF's
`serializers.Serializer` is the actual runtime-enforcement layer in this
codebase — it's Django's answer to what Pydantic does for FastAPI. Field
types on the serializer are validated at runtime when `.is_valid()` runs; bare
function-signature hints elsewhere are not. That's a precise, defensible
answer to "where does this codebase get Pydantic-like enforcement?"

No Pydantic or dataclasses usage found in first-party code (two incidental
`dataclass` hits were in `mt-chamber-interface/utils/integrations.py` and a
test file — worth a quick look if you want a second example, not checked in
detail here).

---

### Mutable / immutable

✅ **FOUND** — mutable default argument, not authored by you.

**File:** [mt-cm-common-plugin/src/cm/bill_cache_utils.py:185](../mt-cm-common-plugin/src/cm/bill_cache_utils.py#L185) (identical copy also in `mt-chamber-interface/src/mt_chamber_interface/bill_cache_utils.py:138`)
```python
def _create_cache_instance(object_instance, object_type, prefilled_slots={}):
    slots = []
    for field in object_type._fields:
        if field in prefilled_slots.keys():
            slots.append(prefilled_slots[field])
        else:
            slots.append(getattr(object_instance, field))
    return object_type(*slots)
```

**How to tell the story — precisely, no overclaiming:** this is the exact
textbook anti-pattern (`def f(x={})`), and it's real, in production, called
from several `_create_*_cache_instance` helpers. **But** I checked the body:
`prefilled_slots` is only read (`.keys()`, `[field]`), never mutated, so it
isn't *currently* causing a bug. The honest framing for the interview: "here's
the anti-pattern living in real code — it happens to be safe today because
nobody writes to the shared dict, but it's one incautious edit away from a
bug where state leaks across unrelated calls. I'd flag it in review even
though it's not broken yet." That's a stronger, more credible answer than
claiming you found and fixed a live bug.

**Whose code:** `git log` → Bryan Spence, `cla826`. Not yours.

**Bonus, ties mutable/hashable together with caching:** [lrms-core/src/lrmslib/hashutils.py:96-131](../lrms-core/src/lrmslib/hashutils.py#L96-L131) `gen_key()` explicitly normalizes
lists/dicts/sets into a stable hash because raw mutable containers can't be
used as cache keys — a clean real-world illustration of "dict/set keys must
be hashable → must be immutable." Authored by Brant Watson, not you.

---

### Scope — LEGB

✅ **FOUND** — module-level `global`, not authored by you.

**File:** [mt-pcs/src/mt_pcs/settings.py:85-99](../mt-pcs/src/mt_pcs/settings.py#L85-L99)
```python
_backend_instance = None

def _get_pcs_backend():
    """Get and cache a backend instance using configured datastore settings."""
    global _backend_instance
    if _backend_instance:
        try:
            _backend_instance.auth.claims
        except ExpiredSignatureError:
            _backend_instance = None
    if _backend_instance is None:
        ...
```

**How to tell the story:** classic lazy-singleton-via-module-global. Without
`global _backend_instance`, the `_backend_instance = None` reassignment inside
the `if` branch would make Python treat `_backend_instance` as local for the
whole function — LEGB's "assigning anywhere makes it local" gotcha, and
you'd get `UnboundLocalError` on the earlier read. `global` is what lets the
function rebind the module-level name instead. Good, minimal example — no
need to embellish.

**Whose code:** not checked via blame line-by-line, but this file is a
`mt_pcs` settings module you have not committed to (your one commit there is
a changelog entry only) — present as "real code I can walk through," not
yours.

---

### Garbage collector

✅ **FOUND** — this is the strongest GC citation available: an explicit,
targeted `gc.disable()`/`gc.enable()`, not authored by you.

**File:** [lrms-core/src/lrmslib/gcutils.py](../lrms-core/src/lrmslib/gcutils.py) (full file, 22 lines)
```python
@contextmanager
def temporarily_disable_gc():
    """Context manager that temporarily disables garbage collection

    Useful for allocation-heavy operations (e.g. unserialization)
    """
    entry_state = gc.isenabled()
    gc.disable()
    try:
        yield
    finally:
        if entry_state:
            gc.enable()
```

**Real usage:** [mt-cm-common-plugin/src/cm/query_cache.py:89-101](../mt-cm-common-plugin/src/cm/query_cache.py#L89-L101)
```python
def _cache_get(key, default=None) -> Any:
    with temporarily_disable_gc():
        try:
            return DEFAULT_CACHE.get(key)
        ...

def _cache_set(key, data, *, timeout=None):
    with temporarily_disable_gc():
        DEFAULT_CACHE.set(key, data, timeout=timeout)
```

**How to tell the story:** this is a genuinely advanced, real technique
(the same trick Instagram is famous for using) — disable the cyclic GC around
an allocation-heavy operation (here, deserializing large cached querysets)
because the generational collector's periodic scans add overhead you don't
need if you're not creating reference cycles in that hot path; reference
counting still frees everything normally, you're just skipping the backup
cyclic-collection passes. It's wrapped in a context manager so it's always
re-enabled (respecting whatever the entry state was, not blindly re-enabling
if it started disabled) even on exception. This is a great "beyond the
textbook GC answer" citation.

**Whose code:** `gcutils.py` → Brant Watson, 2021. The call site in
`query_cache.py` → Bryan Spence (`SSR-9555--CM performance tune`). Not yours,
but a strong, precise example to walk through.

---

### Decorators

✅ **FOUND** — two real decorators in the same codebase with opposite
`functools.wraps` behavior, giving you a genuine before/after contrast. Both
not authored by you.

**Good example (uses `functools.wraps`):** [mt-pcs/src/mt_pcs/tasks/bill_status_tasks.py:46-59](../mt-pcs/src/mt_pcs/tasks/bill_status_tasks.py#L46-L59)
```python
def task_prep(task_function):
    """Functionality for event based tasks."""
    @wraps(task_function)
    def inner(data):
        """Create backend and get event to perform actions against."""
        backend = get_pcs_backend()
        with backend:
            task_function(BillStatus.objects.get(Q(apn=data.get('apn')), revno=data.get('revno')))
    return inner

@shared_task(queue='pcs', ignore_result=True)
@task_prep
def log_bill_action_in_session_day(bill_status: BillStatus):
    ...
```

**Gap example (skips `functools.wraps`):** [mt-cm-common-plugin/src/cm/cm_core/utils.py:83-97](../mt-cm-common-plugin/src/cm/cm_core/utils.py#L83-L97)
```python
def profile_function(func: Callable):
    """Simple decorator that times a function."""
    if settings.DEBUG is not True:
        return func
    def wrap_func(*args, **kwargs):
        start_time = time()
        result = func(*args, **kwargs)
        ...
        return result
    return wrap_func   # no @wraps — wrap_func.__name__ shadows func.__name__
```

**How to tell the story:** `task_prep` is a clean cross-cutting-concern
decorator — it wraps every event-based Celery task to open a datastore
backend connection and resolve the `BillStatus` instance before the task body
runs, so each task function only deals with business logic. It correctly uses
`@wraps(task_function)` so `inner.__name__`/`__doc__` reflect the wrapped
function, not the wrapper — important because Celery introspects task names.
`profile_function` is a good contrast: it's a real timing decorator in the
same codebase that *doesn't* use `functools.wraps`, so every decorated
function reports as `wrap_func` in stack traces/logging — a good "here's the
gotcha in the wild, both the fix and the omission" answer.

**Whose code:** `bill_status_tasks.py` — not checked line-by-line but no
commits from you in `mt-pcs` beyond a changelog entry. `cm_core/utils.py` —
you have not touched this file (confirmed via blame earlier).

---

### Middlewares / signals (Django)

✅ **FOUND** — both middleware and signals, strong matches, not authored by
you.

**Middleware — function-based and class-based, in the same file:** [lrms-core/src/ascended/webapp/middleware.py:17-33](../lrms-core/src/ascended/webapp/middleware.py#L17-L33)
```python
def version_middleware(get_response: Callable) -> Callable:
    """Implementing version as middleware to skip 'ALLOW_HOSTS' check for ELB load balancer"""
    ...
    def middleware(request: HttpRequest) -> HttpResponse:
        if request.META["PATH_INFO"] == "/__version":
            return HttpResponse(version)
        else:
            return get_response(request)
    return middleware

class AscendedAnalyticsMiddleware:  # pragma: no cover
    def __init__(self, get_response=None) -> None:
        self.get_response = get_response
    def __call__(self, request) -> HttpResponse:
        ...
```
Registered in [lrms-core/src/ascended/ascended_project/settings/base.py:79-90](../lrms-core/src/ascended/ascended_project/settings/base.py#L79-L90) — ordered right after CORS, before CSRF/session/auth middleware, so order-matters is directly demonstrable.

**Signals — cache invalidation on `post_save`/`post_delete`, exactly the "after a document is saved, invalidate its cache" case the prep doc predicted:** [lrms-core/src/ascended/authentication/signals.py:1-54](../lrms-core/src/ascended/authentication/signals.py) (full file)
```python
@receiver(post_save, sender=User)
@receiver(post_delete, sender=User)
@receiver(m2m_changed, sender=User)
def invalidate_user_cache_user(sender, instance, **kwargs):
    invalidate_user_cache_usernames((instance.username,))
```
and [lrms-core/src/ascended/virtualview/signals.py:10-20](../lrms-core/src/ascended/virtualview/signals.py#L10-L20):
```python
@receiver(post_save, sender=VVMapping)
@receiver(post_delete, sender=VVMapping)
...
def increment_virtualview_version(sender, instance, **kwargs):
    """Increment the virtualview version counter by 1"""
    try:
        cache.incr("virtualview_version")
    except (TypeError, ValueError):
        cache.set("virtualview_version", 1)
```

**How to tell the story:** the Ascended Datastore (the "Ascended" Django app
inside `lrms-core`) uses real `django.db.models.signals` — `post_save`,
`post_delete`, `m2m_changed` — stacked as multiple `@receiver` decorators on
one handler, specifically to invalidate/version a cache the moment auth data
or a "virtual view" mapping changes. This is genuinely the sender-shouldn't-
know-about-the-receiver case: `User`/`Group`/`Permission`/`Token` models don't
know a cache exists; the signal decouples that. Good place to make the
tradeoff point from the doc: decoupled, but you have to go looking in
`signals.py` to discover this side effect exists at all.

**Whose code:** `git log --format='%an'` on both files → only Brant Watson.
Not yours.

Note: I did **not** find any `@receiver`/signal usage in the Django-ORM-based
apps you've actually committed to (`mt-cm-common-plugin`, `mt-chamber-interface`,
etc.) — those apps are built on the custom `joplin` datastore ORM rather than
Django's ORM, which doesn't emit `post_save`/`pre_save` the same way. The
signal examples above live specifically in `lrms-core`'s `ascended` Django
app, which does use standard Django models.

---

### Cache — in-memory vs Redis (pros/cons)

✅ **FOUND** — layered caching, real config, and a nuance worth leading with:
production uses a **disk-backed** Django cache, not Redis, for the main
query cache. Mostly not your code, one function you did touch.

**Django cache framework, low-level API, two named caches:** [mt-cm-common-plugin/src/cm/query_cache.py:52-53, 89-101, 183-202](../mt-cm-common-plugin/src/cm/query_cache.py#L52-L101)
```python
BILL_CACHE = caches["bill-cache"]
DEFAULT_CACHE = caches["default"]

def _cache_get(key, default=None) -> Any:
    with temporarily_disable_gc():
        try:
            return DEFAULT_CACHE.get(key)
        except Exception:
            LOG.exception("Unable to get key %s from default cache", key)
            return default
```
`model_filter_fresh()` (same file, ~L103-264) checks the cache first, and on
a partial hit only re-queries rows changed since the cached `last_revno`,
merging into the cached dict rather than refetching everything — event-based
incremental cache refresh, not just TTL expiry.

**Settings — actual backend is disk, not Redis:** [mt-cm-common-plugin/src/cm/cm_proj/settings/production.py:33-50](../mt-cm-common-plugin/src/cm/cm_proj/settings/production.py#L33-L50)
```python
CACHES = {
    "default": {
        "BACKEND": "diskcache.DjangoCache",
        "LOCATION": "/var/opt/lrms/cache/cm_query_cache",
        "SHARDS": 8,
        ...
    },
    "bill-cache": {
        "BACKEND": "diskcache.DjangoCache",
        "LOCATION": "/var/opt/lrms/cache/cm_bill_cache",
        "MAX_ENTRIES": 1000,
        "TIMEOUT": None,
        ...
    },
}
```

**Redis genuinely is used elsewhere** — as the Celery broker (`mt-pcs/src/mt_pcs/settings.py:68-80`, see Background tasks below), and `lrms-core/docker-compose.yml:19-30` runs a `redis:alpine` container locally alongside `postgres:16`. And `ascended/authentication/signals.py:16-23` explicitly branches on Redis vs non-Redis cache backends (`cache.delete_many` vs a per-key fallback loop).

**How to tell the story:** this is a better, more nuanced answer than "we use
Redis" — the committee-management app's query cache is a disk-backed
(`diskcache`) Django cache, sharded across 8 files, chosen presumably for
persistence-without-a-Redis-dependency on a single-box deployment. Redis
shows up specifically as the Celery message broker, and the code is written
defensively to work whether the configured cache backend supports batch
deletes (Redis in some environments) or not. That's a genuinely interesting
real answer to "in-memory vs Redis" — it's actually "in-process vs on-disk vs
Redis, chosen per use case," which is a stronger answer than the binary the
prep doc sets up.

**Whose code:** `query_cache.py` is mostly Bryan Spence's (see Concurrency/GC
sections) with **one function you did author** —
[query_cache.py:66-87](../mt-cm-common-plugin/src/cm/query_cache.py#L66-L87)
`get_cancelled_bills()`, committed 2026-01-23 (`CM-223`). It's not a caching
function itself (it reads a JSON-ish dict field), so don't claim it as your
caching work — but it's worth knowing you've made commits in this exact file
if asked "have you worked in this part of the codebase."

---

### Background tasks (Celery / SQS)

✅ **FOUND — strong.** Real Celery app, real `@shared_task`s, Redis broker,
task-composition via `group().apply_async()`. Not your code, but this is
probably your best background-tasks story to walk through even though you
didn't write it — the pattern is exactly what the prep doc describes.

**Celery app setup:** [lrms-core/src/ascended/ascended_project/celery.py](../lrms-core/src/ascended/ascended_project/celery.py) (full file)
```python
if gevent_monkey_patch_enabled():
    patch_all()
patch_kombu_connectionpool()  # avoid kombu redis pool exhaustion spikes

from celery import Celery
app = Celery("ascended")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

**Redis as broker:** [mt-pcs/src/mt_pcs/settings.py:68-80](../mt-pcs/src/mt_pcs/settings.py#L68-L80)
```python
broker_url = 'redis://{}:{}/{}'.format(REDIS_HOST_URL, REDIS_PORT, REDIS_DB)
broker_transport_options = {'fanout_prefix': True, 'fanout_patterns': True, 'visibility_timeout': 3600}
result_backend = broker_url
```

**Task definitions:** [mt-pcs/src/mt_pcs/tasks/bill_status_tasks.py:58-60](../mt-pcs/src/mt_pcs/tasks/bill_status_tasks.py#L58-L60)
```python
@shared_task(queue='pcs', ignore_result=True)
@task_prep
def log_bill_action_in_session_day(bill_status: BillStatus):
```

**Composited dispatch — a single revision event fans out into a group of
async tasks:** [mt-pcs/src/mt_pcs/tasks/__init__.py:81-114](../mt-pcs/src/mt_pcs/tasks/__init__.py#L81-L114)
```python
@app.task(name='pcs.dispatcher', ignore_result=True)
def dispatcher(data):
    """Task dispatcher. Called once per revision action in the datastore.
    * Do *NOT* execute tasks synchronously in this function!!!
    * This composited workflow should be scheduled using .apply_async()
    """
    tasks = []
    ...
    if all(('AMENDMENTNUMBER' in data, 'STATUS' in data, data['rev_type'] == 'update')):
        tasks.append(subtask(create_amendment_available_status, args=(data,)))
    ...
    return group(*tasks).apply_async()
```

**How to tell the story:** every write to the datastore ("revision") fires a
single `dispatcher` task, which inspects what changed and composes a `group`
of the specific follow-up tasks that apply (update related refs, move
tokens, send messages, publish journals...), then dispatches them all
asynchronously via `.apply_async()` — the docstring even shouts "do NOT
execute tasks synchronously in this function." That's the request-returns-
immediately / worker-does-the-work pattern from the prep doc, except the
trigger isn't an HTTP request — it's a datastore write. Good nuance to add if
pushed: this is closer to an event-driven fan-out than the "202 Accepted +
job ID" HTTP pattern, worth distinguishing if asked directly about the web
request case.

**Whose code:** no commits from you in `mt-pcs/src/mt_pcs/tasks/`. Your one
`mt-pcs` commit is changelog-only. Present as real, understood code — not
yours.

No SQS usage found anywhere in first-party code — background work here is
Celery/Redis only.

---

## DATABASES

### SQL vs NoSQL

✅ **FOUND**, with one important correction to the prep doc's suggested
angle — flag this before the interview.

**Postgres confirmed as the production engine:** [lrms-core/src/ascended/ascended_project/settings/production.py:32](../lrms-core/src/ascended/ascended_project/settings/production.py#L32)
```python
"ENGINE": "ascended.ascended_project.db_backend.postgres",
```
`lrms-core/docker-compose.yml:8` also runs `postgres:16` for local dev.

Real multi-table joined views against Postgres exist too, e.g. [mt-birt-reports-archive/src/mt_birt_reports/views/bill_agenda_view.sql](../mt-birt-reports-archive/src/mt_birt_reports/views/bill_agenda_view.sql) — a `CREATE OR REPLACE VIEW` joining six tables (bills, members, committees, meetings, sessions, chambers) for a reporting view.

**⚠️ Correction to the prep doc's JSONB suggestion:** the doc suggests
"SaunaGuide used JSONB... DocIntel uses JSONB metadata... if the Propylon
Postgres schema stores flexible legislative metadata in JSONB, even better."
I checked. `mt_models/src/mt_models/cm/committee_meeting.py:81` does have:
```python
meeting_info = fields.JsonField('T_COMMITTEEMEETING_T_MEETINGINFO')
```
— but `JsonField` here ([lrms-core/src/joplin/models/fields/advanced.py:67-84](../lrms-core/src/joplin/models/fields/advanced.py#L67-L84)) is the custom `joplin` ORM's field type, which serializes to/from a **string** column (`valid_types = STRING_TYPES`), not a native Postgres `JSONB` column. I found **zero** uses of Django's actual `models.JSONField` or `django.contrib.postgres` JSONB features anywhere in first-party code (one incidental `CreateExtension`-pattern migration exists but isn't paired with any JSONB field).

**How to tell the story, honestly:** "we do store semi-structured JSON data
in some fields (e.g. meeting metadata) — but it's serialized into a string
column by our custom datastore ORM, not native Postgres JSONB, so we don't
get JSONB's indexing or in-database query operators on it. If I were pointing
at a genuine JSONB blurs-the-line example, it'd have to be from a different
project — that specific pattern isn't in this monorepo." That's a more
defensible answer than claiming JSONB when the underlying storage is a text
column.

---

### EXPLAIN ANALYZE

❌ **NO EXAMPLE — no code footprint, exactly as the prep doc predicted.**
Grepped for `EXPLAIN ANALYZE` / `explain(analyze` across all `.py`/`.sql`/`.md`
in every repo — zero hits outside the prep doc itself. No profiling scripts,
management commands, or tooling wrap it either. This is a "learn
conceptually and actually go run it once" topic, not a "find it in the repo"
topic — the doc's own advice stands.

---

### Indices and their types

✅ **FOUND** — strong B-tree and composite-index examples, not authored by
you. **No GIN/GiST/BRIN found anywhere** — correct the prep doc's DocIntel
suggestion (not in this repo).

**File:** [lrms-core/src/ascended/webapp/models/revision.py](../lrms-core/src/ascended/webapp/models/revision.py) — `LrmsRevision`, the core append-only revision-log table underlying the whole datastore.

Composite index (leftmost-prefix example), lines 151-153:
```python
class Meta:
    app_label = "webapp"
    ordering = ["revno"]
    get_latest_by = "revno"
    index_together = [
        ["end_revno", "rev_type"],
    ]
```

Single-column B-tree indexes throughout the same model, e.g. line 161 (`apn`), line 249 (`revno`, also PK), lines 255/261/268/275/280/297/311/318 (`end_revno`, `transaction_revno`, `rev_owner` FK, `rev_group` FK, `rev_creator`, `permissions_other`, `revised`, `rev_type`).

**How to tell the story:** this is a heavily-indexed append-only log table —
every revision write is looked up later by `apn`, filtered by `rev_type`, or
range-scanned by `revno`/`end_revno`, so nearly every commonly-filtered
column carries `db_index=True` (a standard B-tree in Postgres). The
`index_together` on `("end_revno", "rev_type")` is a genuine composite-index
example: it helps a query filtering on `end_revno` alone or on
`end_revno + rev_type` together, but not `rev_type` alone — the exact
leftmost-prefix rule from the prep doc. Good, concrete, and it's the kind of
table (append-only, heavily queried by a handful of predictable columns)
where "don't over-index" is also a fair caveat to raise.

**Whose code:** this model predates your commits; not yours.

---

## CLOUD — EC2, ECS, RDS, load balancers, deploys, CI/CD

⚠️ **Partial, and an important correction to make before the interview:** I
found **zero references to EC2, ECS, RDS, ALB/NLB, or any AWS-managed service
by name** anywhere in first-party code, settings, or CI config. What's
actually here is a **self-hosted deploy model**: RPM packages built in CI and
shipped to Propylon's own repo server, installed via Ansible (AWX/AAP),
running under systemd on hosts with `/var/opt/lrms/...` paths and `journald`
logging. Do not claim RDS/ECS examples from this repo — say what's actually
there instead, which is still a real, defensible answer.

**✅ CI/CD — this part is strong, and you have real, personally-authored
commits:**

1. [mt-devops/pipelines/linux/defaults.gitlab-ci.yml](../mt-devops/pipelines/linux/defaults.gitlab-ci.yml) — **your commit** `3f01da8`, "Replace AWX with AAP API for deploy-test":
```diff
-    awx job_templates launch ${JOB_TEMPLATE_ID} \
-      --job_type run --extra_vars ... --monitor
+    curl -X POST -H "Authorization: Bearer ${AAP_RWS_AUTH_TOKEN}" \
+      ${ANSIBLE_TOWER_HOST}/api/controller/v2/job_templates/${JOB_TEMPLATE_ID}/launch/ \
+      -d "{\"inventory\": ..., \"extra_vars\": {...}}"
```
   This is a GitLab CI `deploy-test` stage that triggers an Ansible
   Automation Platform job (via REST, previously via the `awx` CLI) to
   install the freshly-built package onto a test inventory — real automated
   deploy, real CI/CD, and it's yours.

2. [mt-bde-law-making/.gitlab-ci.yml](../mt-bde-law-making/.gitlab-ci.yml) — **your commit** `c0a5e407`, "fix publish version normalization for Nexus" — a `publish` stage (PowerShell, Windows runner, since this is the VSTO Word add-in) that pushes build artifacts to a Nexus repository; you fixed a version-string regex in that stage.

3. [lrms-core/.github/workflows/release.yml](../lrms-core/.github/workflows/release.yml) — **your commit** `eec54ff`, a revert of a broken Python-provisioning step in a GitHub Actions release workflow that builds sdists and `scp`s them to `repos.cloud.eu.propylon.com`.

**✅ Load balancer — a genuine ELB health-check endpoint (not yours):** [lrms-core/src/ascended/ascended_project/urls.py:26-31](../lrms-core/src/ascended/ascended_project/urls.py#L26-L31)
```python
def elb_check_view(request):
    return HttpResponse("ELB Check 200")

urlpatterns = [
    path("elb-check", elb_check_view),
    ...
```
and [lrms-core/src/ascended/webapp/middleware.py:18](../lrms-core/src/ascended/webapp/middleware.py#L18) — a middleware comment explicitly says it exists "to skip 'ALLOW_HOSTS' check for ELB load balancer." So an AWS ELB genuinely sits in front of this app in some environment, even though nothing else in the repo configures it — the app just knows to answer its health check. That's a small, honest, concrete thing to say: "I haven't provisioned an ALB myself, but I can show you the health-check endpoint our Django app exposes for one."

**Containers — real, local dev only:** [lrms-core/docker-compose.yml:1-40](../lrms-core/docker-compose.yml#L1-L40) runs `postgres:16`, `redis:alpine`, and an app container built from `docker/datastore/Dockerfile`, with per-service `cpus`/`mem_limit` set. Several other repos (`mt-core`, `mt-doc-api`, `mt-chamber-interface/clients/ci_webapp`) also carry their own `Dockerfile`/`docker-compose.yml`. I did not check whether any of these run in ECS/Fargate in production — nothing in-repo says either way, and given the RPM/Ansible deploy path found elsewhere, I'd guess these Dockerfiles are dev/CI conveniences rather than the production deploy target, but that's an inference, not something I verified.

**Honest bridge to use:** *"Our CI/CD is real — GitLab CI and GitHub Actions
build artifacts and I've shipped fixes to both pipelines. But our deploy
target isn't AWS-managed compute — it's RPM packages installed via Ansible
onto systemd-managed hosts, with an ELB in front of at least one app for
health checks. I understand EC2/ECS/RDS conceptually and would be trading
Ansible/RPM automation for the AWS-managed equivalents of the same
build→test→deploy pattern I've already worked in."*

---

## AI — RAG pipeline, tokenisation

❌ **NO EXAMPLE — confirmed absent, exactly as the prep doc predicted.**
Grepped for `pgvector`, `embedding` (real hits were false positives — DRF's
"embed a field" terminology, not ML embeddings), `openai`, `anthropic`,
`langchain`, and `LLM` across all first-party `.py` files — zero genuine
matches anywhere in the Montana/Propylon monorepo. This is entirely outside
this codebase; RAG and AI-retrieval patterns are a separate story to learn
conceptually rather than anchoring to Montana or to incomplete personal projects.

---

## TECH DESIGN SCENARIOS

Not a "find the topic" section, but worth noting: the concrete findings above
directly support two of these scenarios if you want to reuse them.

- **"Slow API request"** — `select_related`/`prefetch_related` genuinely
  appear ~64 times across first-party code (e.g.
  `lrms-core/src/ascended/azgoth/views.py:448`), and the custom
  `joplin`/`EagerLoadingDepth` machinery in `query_cache.py` exists
  specifically to batch-populate related fields instead of N+1-querying them
  — real eager-loading patterns exist in this codebase even though I didn't
  trace a specific "found the N+1, fixed it" incident.
- **"Big report generation"** — the Celery `dispatcher`/`group().apply_async()`
  pattern in `mt-pcs/src/mt_pcs/tasks/__init__.py` (cited above under
  Background tasks) is a real instance of "never do it synchronously, fan out
  to async workers instead."

---

## Summary — topics with NO codebase example (learn conceptually)

- **EXPLAIN ANALYZE** — no code footprint anywhere; go run it once on a real
  query before the interview, per the prep doc's own advice.
- **AI / RAG / tokenisation** — entirely absent from this monorepo; learn
  conceptually and build hands-on before claiming production experience.
- **JSONB specifically** (as opposed to JSON-as-string) — the doc's suggested
  SaunaGuide JSONB pattern does not exist in this repo; the closest
  thing (`joplin.JsonField`) serializes to a string column, not native
  Postgres JSONB.
- **GIN / GiST / BRIN indexes** — none found; only standard B-tree
  (`db_index=True`) and one composite index (`index_together`) exist here.
- **True `multiprocessing`/CPU-bound parallelism and `asyncio`/`async def`**
  — absent; only I/O-bound `threading` (via the custom `WorkerThread`) and
  Celery-based async task dispatch exist.
- **EC2 / ECS / RDS / ALB by name** — not referenced anywhere; the real deploy
  model is RPM + Ansible/AAP onto systemd hosts, with CI/CD (GitLab CI,
  GitHub Actions) as the genuinely strong, personally-authored part of this
  topic.
- **SQS** — not used; background work is Celery/Redis only.
- **Pydantic / dataclasses as primary validation** — not used; DRF
  serializers are the real runtime-validation layer here.

---

## Front-End (Montana / Aurelia)

> Montana's front end is **Aurelia**, not React — confirmed by
> `package.json` `aurelia` blocks in `mt-chamber-interface/clients/ci_webapp`
> and `mt-committee-management/client/plugins/src/*`. There is **no
> TypeScript anywhere** in either front end (no `.ts` files, no
> `tsconfig.json` — checked both apps directly) and **no AG Grid** (grepped,
> zero hits). Findings below are real Aurelia code mapped to the underlying
> *concept*, with an honest one-line bridge to how you'd say it in React —
> not a claim that Aurelia code "is" React. Authorship checked via
> `git blame`/`git log` against `nigelmcintyre <nigelmcintyre1995@gmail.com>`.

### React fundamentals (most likely to be asked)

✅ **FOUND — props/state, and computed/derived state.** Mixed authorship —
one strong example is **yours**.

**Component state + props (`@bindable`), authored by you:**
[mt-chamber-interface/clients/ci_webapp/src/session/components/bill-status/bill-status-recording.js:94-98](../mt-chamber-interface/clients/ci_webapp/src/session/components/bill-status/bill-status-recording.js#L94-L98)
```js
export class BillStatusRecording {
  @bindable billStatusFilter;
  @bindable billStatusHistory;
  @bindable billStatus;
  @bindable eventFields;
```
`@bindable` fields are the "props" — the parent element passes them in via
attribute binding (`billStatusFilter.bind="..."` on the custom element tag).
Everything else assigned in the constructor (`this.isLoading = false`,
`this.selectedBillFilter = 'all'`, etc. — same file, lines 100-164) is local
component state, conceptually identical to a batch of `useState` calls,
except Aurelia just uses plain instance properties + dirty-checking/
observation instead of a `setState` call to trigger re-render.
**Whose code:** you committed to this exact file three times (`f53e7a32`,
`a945eec8`, `dd31fb43` — chamber-based edit permission logic, see the
Conditional rendering entry below for the diff).
**Bridge:** *"`@bindable` is Aurelia's props — parent passes data down via
attribute binding. Constructor-assigned instance fields are state; Aurelia
re-renders via property observation instead of `setState`/a state-setter
function."*

**Computed/derived state — a direct `useMemo` analogue, authored by you:**
[mt-committee-management/client/plugins/src/mt-cm-common-plugin/src/models/meeting.js:148-160](../mt-committee-management/client/plugins/src/mt-cm-common-plugin/src/models/meeting.js#L148-L160), your commit `dc8def3` ("CM-317-- Fixed time change issue"):
```js
@computedFrom('day', 'referenceTime', 'time')
get timeObject() {
  const timeVal = this.time;
  const timeStr = this.referenceTime
    ? this.referenceTime
    : typeof timeVal === 'string'
    ...
  return Util.stringToDatetime(this.day + ' ' + timeStr);
}
```
**How to tell the story:** `@computedFrom('day', 'referenceTime', 'time')` is
an explicit dependency list, just like a `useMemo` dependency array — Aurelia
only recomputes `timeObject` when one of those three named properties
changes. Your fix reordered the precedence (`referenceTime` now wins over a
raw `time` value) inside that computed getter. This is a clean, honest,
personally-authored bridge to `useMemo(() => computeTime(day, referenceTime, time), [day, referenceTime, time])`.

**Keys in lists:** ⚠️ **WEAK/PARTIAL.** Aurelia's `repeat.for` (see List
rendering below) does not require or commonly use an explicit "key" the way
React does — it diffs arrays by reference/identity via its own array
observer, not a developer-supplied `key` prop. I found no `repeat.key`
usage (Aurelia 1 supports it but it's rare) anywhere in either app. Fair to
say: *"Aurelia's repeater doesn't push key management onto the developer the
way React does — that's actually one thing I'd flag as a difference, not a
similarity."*

**Controlled vs uncontrolled components:** see Forms & input handling below.

---

### Hooks, state & data

✅ **FOUND — data fetching with loading/error state**, not authored by you.

**Service layer (the "data layer"):** [mt-chamber-interface/clients/ci_webapp/src/cm/services/bill-status-service.js:10-31](../mt-chamber-interface/clients/ci_webapp/src/cm/services/bill-status-service.js#L10-L31)
```js
@inject(HttpClient, AuthService)
export class BillStatusService {
  constructor(httpClient, authService) { ... }
  getBillStatusTemplates() {
    return new Promise((resolve, reject) => {
      this.httpClient.get('/bill_status/')
        .then((r) => JSON.parse(r.response))
        .then((data) => data.items.map((d) => BillStatusTemplate.fromObject(d)))
        .then(resolve).catch(reject);
    });
  }
```

**Consumer — loading/error state driving the UI, in the `bind()` lifecycle
hook:** [bill-status-recording.js:166-216](../mt-chamber-interface/clients/ci_webapp/src/session/components/bill-status/bill-status-recording.js#L166-L216)
```js
bind() {
  this.isLoading = true;
  this.isLoadingMessage = billsLoading;
  this.isLoadingError = '';
  return Promise.all([
    this.tokenService.getTokens(),
    this.billStatusService.getBillStatusTemplates()
  ]).then(([tokens, billStatusTemplates]) => {
    ...
    this.isLoading = false;
    this.isLoadingMessage = '';
  });
}
```
Fed into a shared `<cm-loading-feedback>` component in the template (see
Components section) — a real loading/error/empty-state pattern, missing only
the "cancel stale requests" piece (no `AbortController`/cancellation token
anywhere in either app — worth naming as a gap if pushed on it).

**Bridge:** *"`bind()` here is the fetch-on-mount you'd do in `useEffect(() =>
{...}, [])` — a service class stands in for a data layer like React
Query, minus request caching/dedup/stale-request cancellation, which we don't
have. I'd reach for TanStack Query in React specifically to get that for
free."*

**`useMemo`/`useCallback`:** ⚠️ mapped above to `@computedFrom`, which is the
closer analogue. I found no separate memoization utility beyond that.

**Custom hook equivalent:** ❌ **NO EQUIVALENT.** Aurelia has no hooks
concept; the nearest thing is extracting a shared class (e.g. `Scrollable`,
`BodyWidth` below) and injecting it via DI — reuse via composition/DI, not
via a hook you call inside a component function.

---

### TypeScript (also a named requirement)

❌ **NO EQUIVALENT — confirmed absent.** Checked both `ci_webapp` and
`mt-committee-management/client` for `.ts` files and `tsconfig.json`: zero in
either. A `src/cm/types/` directory in `ci_webapp` looked promising but is
just plain-JS enum-like classes (`AllPositionsValuesType extends ValuesType`)
— naming convention, not TypeScript. **Say this plainly rather than
stretching it:** *"Montana's front end predates our TypeScript adoption —
it's plain ES6/Babel. TypeScript is a skill gap I'm actively working to close."* Don't try to
force a bridge here; there isn't one.

---

### MUI & AG Grid (spec nice-to-haves — understand the concepts, be honest about hands-on exposure)

⚠️ **WEAK/PARTIAL — a hand-rolled grid exists, and it's a genuinely good
concept-level bridge, but it is not AG Grid and you didn't write it.**

**File:** [mt-chamber-interface/clients/ci_webapp/src/cm/components/grid/grid.js](../mt-chamber-interface/clients/ci_webapp/src/cm/components/grid/grid.js) (full file, 89 lines)
```js
@customElement(`${constants.elementPrefix}grid`)
@inject(Element, Util)
export class Grid {
  @bindable({ defaultBindingMode: bindingMode.twoWay }) dataSource;
  @children('cm-col') columns;
  ...
  dataSourceChanged(newValue, oldValue) {
    logger.info('dataSourceChanged invoked');
  }
```
Template (col rendering via `<compose>`, i.e. custom cell templates): [grid.html:6-30](../mt-chamber-interface/clients/ci_webapp/src/cm/components/grid/grid.html#L6-L30)
```html
<th repeat.for="item of columns" class="text-${item.textAligment}">${item.title}</th>
...
<template if.bind="col.isTemplate" part="col-template">
  <compose view.bind="col.template.name" view-model.bind="this"></compose>
</template>
```

**How to tell the story, without overclaiming:** `@children('cm-col')`
collects `<cm-col>` child elements declared inside `<cm-grid>` in markup —
that's the direct analogue of AG Grid's `columnDefs` array, just expressed as
child elements instead of a config object. The `<compose>` element per
column that has a custom template is the direct analogue of an AG Grid
`cellRenderer`. What's **missing** compared to AG Grid: no sorting, no
filtering, no virtualization, no client-side/server-side row model — it's a
plain `<table>` with `repeat.for` over the full dataset, so it would not
hold up on a large dataset. That's an honest, useful line: *"I've worked with
a hand-rolled data-grid component that mirrors AG Grid's column-definition
and cell-renderer concepts at a basic level, but without virtualization or
built-in sorting/filtering — which is exactly why understanding AG Grid's row model and
virtualization story matters, even though I haven't built with it hands-on yet."*

**Whose code:** `git log` → Brendan Salmond, Wendel Silva. Not yours.

**MUI:** ❌ **NO EQUIVALENT.** No component-theming library in either
Aurelia app (styling is hand-written Bootstrap/SCSS — see Styling below).
Nothing here bridges to MUI's `sx` prop or theming; this is a skill gap worth studying before the interview.

---

### Framework-agnostic front end (your real strength zone — lean in here)

✅ **FOUND — debounce, throttle, async/await, and DOM event handling.** All
real, none authored by you, but this is the section where the doc says lean
on general experience rather than needing personal authorship.

**Debounce (lodash), scroll handling:** [mt-chamber-interface/clients/ci_webapp/src/cm/lib/scrollable.js](../mt-chamber-interface/clients/ci_webapp/src/cm/lib/scrollable.js) (full file)
```js
import { debounce } from 'lodash';
export class Scrollable {
  constructor(element) {
    this.scrollableElement = element;
    this.scrollToEnd = debounce(this._scrollToEnd.bind(this), 150);
    this.scrollToTop = debounce(this._scrollToTop.bind(this), 150);
  }
```

**Hand-rolled throttle (not a library), window resize:** [mt-committee-management/client/plugins/src/mt-common-frontend/src/components/body-width/body-width.js:18-38](../mt-committee-management/client/plugins/src/mt-common-frontend/src/components/body-width/body-width.js#L18-L38)
```js
constructor(eventAggregator) {
  ...
  window.addEventListener('resize', this.listen, false);
}
listen = () => {
  if (!this.timeout) {
    this.timeout = window.setTimeout(() => {
      this.timeout = null;
      this.onSizeChanged(window.innerWidth);
    }, RATE);
  }
};
```
**Worth naming honestly if asked:** this listener is registered in the
constructor with **no corresponding `removeEventListener`/cleanup anywhere
in the file** — a real leak-shaped gap. Good, specific bridge line: *"In
React, `useEffect`'s cleanup function would force you to return
`() => window.removeEventListener(...)` — this Aurelia code doesn't have an
equivalent enforced discipline, and it shows: there's no teardown here."*

**async/await + try/catch (Clipboard API), and a mixed async/`.then()`
style:** [mt-committee-management/client/plugins/src/mt-cm-committees-plugin/src/components/finalize-meeting/components/exhibits-list/exhibits-list.js:18-26](../mt-committee-management/client/plugins/src/mt-cm-committees-plugin/src/components/finalize-meeting/components/exhibits-list/exhibits-list.js#L18-L26)
```js
async handlePasteLink(index) {
  try {
    const link = await navigator.clipboard.readText();
    this.exhibits[index].exhibit.pastedLink = link;
  } catch (error) {
    logger.info('Failed to read clipboard', error);
  }
}
```
Note: `ci_webapp` (the older of the two apps) uses **zero** `async`/`await` —
it's `.then()`/`.catch()` chains throughout (see `bill-status-service.js`
above). `mt-committee-management`'s client is the newer codebase and does use
`async`/`await`. Accurate framing: *"depends which part of Montana — the
newer plugin-based app uses async/await, the older webapp is still
Promise-chain style."*

**Event handling — `click.delegate`, event delegation baked into the
template syntax:** [bill-status-recording.html:31](../mt-chamber-interface/clients/ci_webapp/src/session/components/bill-status/bill-status-recording.html#L31)
```html
<input type="radio" model.bind="item.id" checked.bind="selectedBillFilter"
  click.delegate="changeBillFilter(item.id)">
```
**Bridge:** *"`click.delegate` attaches a single delegated listener at a
parent and dispatches by event target — same idea as attaching an `onClick`
handler in JSX, except Aurelia does the delegation for you; in React you'd
just pass `onClick={() => changeBillFilter(item.id)}` per element and not
think about delegation at all."*

**Pub/sub as an alternative to prop-drilling:** `EventAggregator` (used in
both `body-width.js` above and `bill-status-recording.js:220-221`) is a
publish/subscribe bus injected via DI — components anywhere in the tree can
publish or subscribe to named events without passing callbacks down through
props. **Bridge:** *"This is the same problem Context solves in React — avoid
drilling a callback through five levels of props — but implemented as a
global pub/sub bus rather than a scoped provider tree. I'd flag the tradeoff:
Context keeps the data flow traceable to a subtree; an event bus can fire from
anywhere, which is powerful but harder to trace — the same signals-vs-direct-
call tradeoff from the Django signals topic, actually."*

**CSS/SCSS:** see Styling entry under the bridge section below (kept there to
avoid duplicating).

**Bundler / code splitting:** ✅ **FOUND**, tied to Routing below —
`PLATFORM.moduleName(...)` route definitions are Aurelia-CLI/webpack's
route-level code-splitting mechanism (each route's module becomes its own
webpack chunk, loaded on navigation). Real, not yours — see Routing.

**Browser rendering (reflow/repaint), accessibility (ARIA), XSS/CSRF:** ❌ **NO
EXAMPLE** — these are general knowledge topics with no specific line of
Montana code to point at; nothing in the repo demonstrates deliberate
reflow-avoidance, ARIA annotation, or CSRF handling that I could find and
verify. Answer conceptually.

---

### The Aurelia → React bridge (a genuine depth signal)

✅ **FOUND — components, DI, binding-mode contrast, routing.** This is where
the concept checklist's remaining items (custom elements, `@inject`, forms,
routing) land.

**Custom elements = components, `@customElement` + paired `.js`/`.html`/`.scss`:**
[mt-chamber-interface/clients/ci_webapp/src/cm/components/cm-loading-feedback/](../mt-chamber-interface/clients/ci_webapp/src/cm/components/cm-loading-feedback/) — three files, not authored by you:
```js
// cm-loading-feedback.js
const defaultBindingMode = { defaultBindingMode: bindingMode.oneWay };
@customElement('cm-loading-feedback')
export class LoadingFeedback {
  @bindable(defaultBindingMode) isLoading = false;
  @bindable(defaultBindingMode) useSpinner = true;
  @bindable(defaultBindingMode) error = '';
  @bindable(defaultBindingMode) message = '';
}
```
```html
<!-- cm-loading-feedback.html -->
<template class="cm-loading-feedback">
  <span if.bind="isLoading">
    <span if.bind="useSpinner" class="spinner-container">...</span>
    <span if.bind="message" class="message-container">${message}</span>
  </span>
  <span if.bind="error" class="error-container">${error}</span>
</template>
```
Consumed as `<cm-loading-feedback is-loading.bind="isLoading" message.bind="isLoadingMessage" error.bind="isLoadingError">` in `bill-status-recording.html:38-41` — a real, reusable presentational component, props-in only.

**The two-way vs one-way binding-mode contrast the doc wants you to state out
loud — both halves genuinely exist in this codebase:** `cm-loading-feedback.js`
above explicitly opts into `bindingMode.oneWay` for every bindable, while
[cm-filtering.js:6,11](../mt-chamber-interface/clients/ci_webapp/src/cm/components/cm-filtering/cm-filtering.js#L6-L11) explicitly opts into `bindingMode.twoWay`:
```js
const defaultBindingMode = { defaultBindingMode: bindingMode.twoWay };
export class CmFiltering {
  @bindable(defaultBindingMode) valueFilter;
```
**How to tell the story:** Aurelia's actual default (when you don't specify
a mode) is one-way for `@bindable`, but plain `value.bind`/`checked.bind` on
native form elements in templates is two-way by convention — and this
codebase shows developers being deliberate about it at the component level:
a display-only component (`cm-loading-feedback`) is explicitly one-way, an
input-driving component (`cm-filtering`) is explicitly two-way. **Bridge, use
the doc's suggested line, it's earned here:** *"I've seen both directions used
deliberately in the same Aurelia codebase — and I actually prefer React's
default of one-way data flow with explicit state setters, because you don't
have to go check a binding-mode declaration to know whether a child can
mutate a parent's state."*

**Dependency injection, `@inject` — present everywhere, contrast with
React:** [bill-status-recording.js:83-93](../mt-chamber-interface/clients/ci_webapp/src/session/components/bill-status/bill-status-recording.js#L83-L93)
```js
@inject(
  TaskQueue, Router, BillStatusService, SponsorsService,
  TokenService, MessengerService, EventAggregator, AuthService, DialogService
)
export class BillStatusRecording {
```
Nine services constructor-injected by Aurelia's built-in container — no
manual wiring, no provider tree. **Bridge, the doc's exact point, verified
real here:** *"Aurelia has DI built into the framework — decorate a class
with `@inject(...)` and list constructor dependencies, done. React has no
built-in DI; the closest equivalents are passing instances via Context,
or a library like `tsyringe`/`InversifyJS`. I'd reach for Context for this
in React, and only bring in a DI library if the dependency graph got
genuinely complex."*

**Forms & input handling — controlled-input equivalent + validation:**
[mt-chamber-interface/clients/ci_webapp/src/cm/dialogs/edit-date.js](../mt-chamber-interface/clients/ci_webapp/src/cm/dialogs/edit-date.js) (full file, 79 lines), not authored by you:
```js
activate(model) {
  this.model = model;
  ValidationRules.ensure('newDate').required().on(this.model);
}
validateForm(formField) {
  return this.validationController.validate().then((form) => {
    form.results.filter(...).map((result) => {
      const validationMessageProp = `${result.propertyName}Invalid`;
      this[validationMessageProp] = result.valid ? '' : result.message;
    });
    return form;
  });
}
```
`value.bind`/`checked.bind` on native inputs (seen throughout
`bill-status-recording.html`, e.g. line 29 `checked.bind="selectedBillFilter"`)
is Aurelia's controlled-input pattern — the bound property is the single
source of truth, same idea as React's `value={state} onChange={...}`, just
without writing the `onChange` handler yourself (the framework does it via
the binding). `aurelia-validation`'s declarative `ValidationRules.ensure(...)`
+ a `ValidationController` you inject and call `.validate()` on is the
validation-library equivalent of something like `react-hook-form` +
`zod`/`yup`. **Bridge:** *"Aurelia's two-way binding gives you controlled
inputs for free — you bind a property and the DOM stays in sync both ways.
In React I'd wire that up explicitly with `value`/`onChange`, and reach for
`react-hook-form` for the validation-rules piece `aurelia-validation` gives
me declaratively here."*

**Routing — real route table, lazy-loaded modules, role-based route
guards (guard config is yours):**
[mt-chamber-interface/clients/ci_webapp/src/session/session.js:23-60,108-114](../mt-chamber-interface/clients/ci_webapp/src/session/session.js#L23-L114)
```js
{
  route: 'bill-status/recording',
  name: BILL_STATUS_RECORDING,
  moduleId: PLATFORM.moduleName('session/components/bill-status/bill-status-recording'),
  title: 'Maintain Bill Status'
},
...
configureRouter(config, router, authService) {
  this.router = router;
  config.map(this.routes);
}
```
`PLATFORM.moduleName(...)` is what tells the webpack/Aurelia-CLI bundler to
split that route's module into its own chunk, loaded on navigation — real
route-level code splitting, not a React concept but the same underlying
mechanism as `React.lazy(() => import('./BillStatusRecording'))`.

Route-level authorization metadata, **authored by you**:
[mt-committee-management/client/plugins/src/mt-cm-admin-plugin/src/routes.js:72-85](../mt-committee-management/client/plugins/src/mt-cm-admin-plugin/src/routes.js#L72-L85), your commit `5d81b29` ("AD-groups-55"):
```js
restrictAccess: {
  groups: [
    'admin', 'role_admin', 'role_admin_secretary',
    'role_committee_secretary', 'role_developer', ...
  ]
}
```
You removed `role_committee_secretary_supervisor` from several routes'
allowed-groups lists across two plugins in this commit — real, yours, and a
clean example of route-level authorization config, conceptually the same as
a React Router loader/guard checking a role list before rendering a route
element.

**Lifecycle — `bind()`/`attached()` vs `useEffect`:** already covered above
(`bind()` ~ mount-time `useEffect`); worth adding `attached()` (used in
`grid.js:32-43` and `cm-filtering.js:15-25` to do DOM-dependent work once the
element is actually in the document) as the closer analogue to
`useEffect(() => {...}, [])` for anything that needs real DOM nodes, since
`bind()` fires before the element is attached to the document.

**Styling approach:** ✅ **FOUND.** Convention-based, component-scoped SCSS
paired 1:1 with each component (not CSS Modules/Shadow DOM scoping — just a
matching top-level class name by convention): [cm-loading-feedback.scss](../mt-chamber-interface/clients/ci_webapp/src/cm/components/cm-loading-feedback/cm-loading-feedback.scss) (full file, 19 lines)
```scss
.cm-loading-feedback {
  .spinner-container { margin: 0 5px; ... }
  .error-container { color: #f00; ... }
}
```
Plus a large shared `scss/` tree (Bootstrap-based, `scss/vendor/bootstrap`,
`scss/core`, `scss/helpers`) at the app root for global styles/mixins.
**Bridge:** *"Component-scoped SCSS by naming convention, not enforced
isolation — a component's styles could technically leak or collide since
it's just a class-name convention, not real scoping. I'd contrast that with
CSS Modules or styled-components/MUI's `sx`, which give you enforced
scoping, or Shadow DOM if using true web components."*

---

## Front-End summary — topics with NO Montana example (learn conceptually)

- **TypeScript** — confirmed absent from both Montana front ends entirely;
  learn conceptually and build hands-on to close this gap.
- **AG Grid specifically** — no library usage found; the closest thing is a
  hand-rolled `<cm-grid>` with no virtualization/sorting/filtering (cited
  above as a partial concept bridge, not a substitute).
- **MUI** — no component-theming library in either app; hand-written
  Bootstrap/SCSS instead.
- **React-specific mechanics with no Aurelia analogue at all** — Virtual DOM
  reconciliation, `StrictMode` double-render, `useTransition`/Suspense/
  concurrent features, `useContext` (closest real analogue here is
  `EventAggregator` pub/sub, not a scoped provider), cancelling stale
  requests (no `AbortController` usage found anywhere).
- **Browser rendering internals (reflow/repaint), accessibility/ARIA,
  XSS/CSRF on the front end** — no specific Montana code demonstrates these
  deliberately; answer from general knowledge.
- **Keys in lists** — Aurelia's `repeat.for` doesn't use an explicit key the
  way React does; this is a real *difference* to name, not a match to claim.
