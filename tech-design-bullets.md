4. “We’ll process millions of rows a day in the DB”
- Index query paths only, writes can slow.
- Partition big tables, time ordered use BRIN
- Use bulk operations over row by row
- Use connection pooling for short loved operations
- Use a read replica
- Does all the data need to live in a transactional DB
- Front end, virtualise render whats visible, paginate