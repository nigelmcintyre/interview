
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

# Behavioural
### 1. Difficult troubleshooting / debugging
- Worked on a ticket where when a public member was removed from a committee, their uses asset was also being deleted not just their membership to that committee.
- I began by adding in some logs and reproducing the bug using test data in our test environment.
- I stepped through the code to trace the flow and found that the front end tracked `removedPublicMembers`
- The backened removed assets of every UUID on that list, wheras as for other members were removed from the committee.
- I changed the logic so public members went through the same flow as other member types which resolved the issue.
- I'm guessing the logic was implemented for public members to be removed from the system and was mistakenly wired into this use case.

### 2. Ownership end-to-end
- During codification process users export the enrolled bill document to HTML.
- In some of the bills they use tabbed tables in the word document and our export tools do not support tabbed tables.
- Users had been manually converting the tabbed tables into html tables and I was tasked with automating this process.
- After some planning I settled on a design for the solution 
- I built a doc.xml tabbed table to html conversion API endpoint reusing some conversion code from other projects. 
- I added a tabbed table detection step based on real examples, to the word addin export process which called the endpoint and passed in doc.xml fragments of the tabbed tables
- And the word addin stitched the returned html into the exported html, with a fallback to the original export process if anything went wrong.
- Now when a user exports an enrolled bill containing tabbed tables they no longer need to manually convert them to html tables

### 3. Non-technical / cross-functional stakeholders (disagreement)
- Some time ago I worked on a budget bill versioning system, there were 6 types each with multiple versions.
- Drafters found the naming convention confusing and wanted it simplified.
- I pushed back on the ask as it would require a significant code rewrite close to presession code freeze and put a functioning system at risk.
- Through more discussions with the client I found that how the bill versions were displayed in the bill document was the main source of confusion
- I suggested hiding the confusing version number in the bill document untill after session when we would be able to spend the time needed for the rewrite which they agreed to.
- So I avoided a risky code rewrite close to session and they got part of the simplification they needed.

### 4. Initiative / introducing a technology or process
- Client tickets lacked proper reprodiuction steps, fullscreenshots, example documents and logs.
- Caused a lot of unecessary back and forth.
- I suggested we have the client supply fullscreenshots, local logs, full reproduction steps and example documents in every ticket.
- I updated the Gitlab bug ticket template, and I ran a training session showing the clients how to access the local logs to add to the ticket.
- Resulted in much less back and forth on bug tickets as we now nearly always had all the information we needed to fix a bug.

### 5. A difficult decision or tradeoff
- Users were manually creating shortened pdfs of amendments and wanted an option so automatically generate them.
- I narrowed it down to two options, running the generation on the server using LibreOffice to generate the PDF from the amendment word doc which would be faster but from experience less certain out the output.
- Or automate their manual process and use word to generate the pdf and guarantee exact match between .docx and .pdf
- Given these are legal documents fidelity is very important I decided to go with the word route.
- I added a new tab to the BDR portal in the broser where they could specify pages of the amendment they wanted printed to pdf.
- I added a metadata field to the amendment word doc to pass this info to the users local machine as you cannot pass data through the URI we use to open word and the .docx 
- The word addin read the metadata when the .docx opened and started the pdf print process, saving it to the datastore.
- The result was users no longer had a long manual process for printing specific pages to pdf, and could guarantee the content matched the amendment .docx

### 6. Team collaboration