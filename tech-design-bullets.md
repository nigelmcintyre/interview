4. “We’ll process millions of rows a day in the DB”
- Index query paths only, writes can slow.
- Partition big tables, time ordered use BRIN
- Use bulk operations over row by row
- Use connection pooling for short loved operations
- Use a read replica
- Type of Data tells us if should be a transactional DB or analytics warehouse
- Front end, virtualise render whats visible, paginate server side
Pushback Qs
- Why BRIN over B-tree? BRIN index is small in comparison for time ordered data

5. "Two users edit the same record at the same time"
- Optimistic for when conflicts are rare.
- Maintain version column to check if version has changed then flag conflict
- Pessimistic for when conflicts are likely or unacceptable used in propylon
Pushback Qs
- What should happen when optimistic conflict? UI show conflict or auto-merge non-overlaping fields
- Could you combine both? Yes but complex

6. We depend on a slow / flaky 3rd party API?
- Use timeouts, circuit-breaker after n retries.
- Move it off request path, use a backround job serve last good cached response with notice.
Pushback Qs
- how determine timeout? Measure under normal circumstances plus buffer.
- whats the risk with retries? All retry at once, try randomize retry after.

