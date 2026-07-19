
# TECH DESIGN SCENARIOS
### 1. "We have a slow API request — what do you do?"
- Measure the request to find the bottlenecks
- N+1 switch form lazy loading to eager using 'select_relate' / 'prefetch_related'
- Use explain analyze on query, look for seq_search, add an index
- Use seleted columns '.values()' or pagination instead of over-fetching (Saunaguide)
- Move heavy computation to background worker
- Cache expensive reusable responses
- Make queries multithreaded if applicaple (Propylon committe creation writes witnesses, members, meetings with threads)

### 2. "We want a feature that generates a big report"
- Don't generate report in the request
- Offload to a celery worker, view enqueues and response with job id
- Worker should stream the data using an .iterator() with a fixed chunck size so memory stays flat
- finished file saves on to a S3 bucket, keyed by query params, doubles as a cache for identical queries
- Client polls for status and receives presigned link when ready.

### 3. "We want real-time notifications / a live progress bar"
- Polling every few seconds simplest
- Server Side Events would work too, FE opens connection channel server pushes messages onto it
- Websockets only needed for bidirectional comms like webchat

### 4. “We’ll process millions of rows a day in the DB”
- Index query paths only, writes can slow.
- Partition big tables, time ordered use BRIN
- Use bulk operations over row by row
- Use connection pooling for short loved operations
- Use a read replica
- Type of Data tells us if should be a transactional DB or analytics warehouse
- Front end, virtualise render whats visible, paginate server side
Pushback Qs
- Why BRIN over B-tree? BRIN index is small in comparison for time ordered data

### 5. "Two users edit the same record at the same time"
- Optimistic for when conflicts are rare.
- Maintain version column to check if version has changed then flag conflict
- Pessimistic for when conflicts are likely or unacceptable used in propylon
Pushback Qs
- What should happen when optimistic conflict? UI show conflict or auto-merge non-overlaping fields
- Could you combine both? Yes but complex

### 6. We depend on a slow / flaky 3rd party API?
- Use timeouts, circuit-breaker after n retries.
- Move it off request path, use a backround job serve last good cached response with notice.
Pushback Qs
- how determine timeout? Measure under normal circumstances plus buffer.
- whats the risk with retries? All retry at once, try randomize retry after.

# Python
### 1. Concurrency - multithreading, multiprocessing, async
- The Global Interpreter Lock (GIL) means that only one thread executes python bytcode at a time.
- For small scale I/O bound work where while one threads waits, the GIL releases and another thread can continue
- Example is committee creation, three threads initiate writes to DB for witnesses, committee members and meetings.
- Is bolt on can wrap an existing blocking call in a ThreadPoolExecutor
- while the first thread waits for DB write to complete, GIL is release and the next can execute.
- For higher volume concurrency we use Async.
- Requires async aware stack HTTP client, DB driver, can scale to thousands concurrent connections.
- For CPU bound work parsing / computation can be done by multiple different processes.
- Each process has their own interpreter and GIL true parallelism.
- Celery worker is an example of a multiprocess.

### 2. Types
- Python is dynamically & strongly typed.
- Types hints help for readability with tools like mypy but are not enforced.
- The closest thing to enforced types is using Pydantic and FastAPI
- Type hints are used to validate incoming data at function boundary, if it doesn't match your code doesn't execute.

### 3. Mutable / Immutable
- Immutable types cannot be changed after creation. int, float, str, tuple, frozenset
- Mutable types list, dict, set and most custom objects can.
- Mutable default arguments 'def add(item, items=[])' mean every time the method is called unless the default argument is defined in the call, it'll get shared between calls.
- Fixed by setting default argument items=none and items = items or [].

### 4. Scope - LEGB
- Name lookup follows LEGB.
- Assigning a variable anywhere in function makes it local for the whole function
- At compile time python scans the whole function not in order
- using a variable that is later assigned in the variable will cause UnboundLocalError
- use nonlocal to use variable value from enclosing scope.

### 5. Iterators -> Generators
- An iterator is anything you call next() which advances its internal state.
- A Generator is an interator under the hood yield pause and returns a value
- On the next next() the generator picks up where it left off all localv variables intact.
- Can use it to stream large data of unkown size, keeps memory usage flat QuerySet.Iterator(chunk_size(2000)).
- Using list comprehension keeps whole list in memory, so we can get length and access data at will
- But if huge list all memory would get used up.

### 6. Garbage Collector
- Python uses reference counters to know when to release memory.
- When ref count of an object hits 0 it is rleased.
- For objects that onhly reference eachother, uses periodic GC to releae newer objects, keeps older cyclic objects with references

### 7. Decorators
- Wrap a function in cross cutting concerns you want repeated on each call.
- eg. auth, logging, FastAPI route registration handles parsing, validating, function call, serialising response.
- reduces boilerplate, hides functionality. 
- use functools.wraps to ensure the function keeps its name and docstring

### 8. Context managers
- 'with' guarantees setup and teardown. 
- Use it for opening a file the file will close even if an exception occurs.
- Djangos transaction.atomic() is a context manager.
- Ensures all writes commit together or roll back together.
- Mechanism behind concurrent-edits and duplicate requests.

### 9. Middlewares / Signals
- Are components in the request/response pipeline.
- every request passes through them on the way in and in reverse order on the way out.
- Auth, session management etc.
- Signals are decoupled event notifications that allow a client to send a signals without know who is listening
- like invalidating cached document records after a status change in BDR

### 10. Django ORM - N+1, `select_related` / `prefetch_related`
- One query fetches N rows
- Accessing related field lazily on each row makes one more query per row
- `select_related` fixes this by doing a join on a foreign key in one query
- Use Django's debug toolbar to identify, look for lots of identical fast queries.
- `prefetch_related` fixes this for many to many relationships with two queries/
- Results are stitched together by key in python. 
- Using a JOIN on MTM relationship would multiply out each row, a bill with 5 co-sponsors would appear 5 times in result set.

### 11. Cache - in-memory vs Redis
- In memory is fatests but is per process and doesn't survive restart
- Redis is a network hop slower but is shared across all app servers and persists so is useful when running multiple app servers
- Need to remember to invalidate cache when entry becomes stale.
- Has caused issue before in BDR when state was not cleared between drafting sessions and when bill status was updated.
- Can use caching to reuse query results to improve performance.

### 12. Background tasks - Celery
- Report generation, file processing should be done outside of request cycle
- Can block a worker and risk timing out.
- API view enqueues job on to broker (can use Redis) returns job id, workers pull tasks off queue when free.
- If worker crashes mid task, task goes back on to queue to ensure task completes.