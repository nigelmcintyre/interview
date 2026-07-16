
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

