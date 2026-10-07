# Detection Techniques

Each class of SQL injection shows up through a different observable signal.
Identify the signal first; it tells you which extraction method to use.

## Error-based

**Signal:** the application leaks a database error message (stack trace,
driver error, or a generic 500 that changes shape) when you send a
syntax-breaking character.

**Probe:** send `'`, `"`, `\`, or `'||'` in each parameter and diff the
response against a baseline request. A response that changes from "200 OK
normal page" to a SQL error, a blank page, or a different HTTP status is
a strong signal.

**Why it works:** the query becomes syntactically invalid, and the
database driver's error surfaces either directly (verbose error pages) or
indirectly (a 500 vs. 200 status difference).

**Extraction:** many engines let you force the error message itself to
contain data (e.g., MySQL's `extractvalue()`/`updatexml()` XPath error
tricks, MSSQL's conversion errors via `CONVERT(int, (SELECT ...))`). See
`database-syntax.md` for exact functions.

## Boolean-blind

**Signal:** no visible error, but the page content, length, or HTTP
status reliably differs between a condition that's true and one that's
false, even though nothing else changed.

**Probe:** compare `id=1 AND 1=1` vs `id=1 AND 1=2` (or the equivalent in
the actual query context). If the first behaves like the baseline and the
second returns an empty/different result, the input reaches a boolean
condition.

**Why it works:** the application doesn't tell you *why* the result
differs, but the binary true/false signal is enough to ask the database
yes/no questions — including about data you don't yet know (see "blind
extraction" below).

**Extraction:** binary search. To read a character at a given position:
ask `AND SUBSTRING(table.column, N, 1) > 'm'` and narrow the ASCII range
with each response, like a guessing game — ~7 requests reveal one
character. sqlmap automates this with `--technique=B`.

## Time-blind

**Signal:** the page content and status are identical for true/false
conditions, but response time differs when you inject a conditional
delay.

**Probe:** inject `AND IF(1=1, SLEEP(5), 0)` (MySQL) or the engine's
equivalent (see `database-syntax.md`) and compare response latency against
a `1=2` control. A ~5-second difference confirms the injection point
*without* needing any visible difference in output.

**Why it works:** even when the application shows zero behavioral
difference, the database still executes the delay, and wall-clock time is
observable from outside.

**Extraction:** same binary-search idea as boolean-blind, but each
character guess swaps the comparison into a conditional `SLEEP`. This is
the slowest technique — budget accordingly and respect the engagement's
rate limits. sqlmap: `--technique=T`.

## UNION-based

**Signal:** the query's result set is reflected directly in the response
(a table, a list), and you can control the number of columns.

**Probe:** determine column count with `ORDER BY N` (increment N until it
errors) or by appending `UNION SELECT NULL,NULL,...` and adjusting the
NULL count until it stops erroring. Then replace NULLs one at a time with
a marker string (e.g., a random token) to find which columns are
reflected in the visible output.

**Why it works:** `UNION SELECT` appends your own result set to the
original query's output, as long as the column count and (for some
engines) compatible types match. Once you know which output column is
rendered on the page, you put your own data in that slot.

**Extraction:** direct — put `table_name`, `column_name`,
`concat(user,0x3a,password)` etc. into the matching column and read it
straight from the page. Fastest technique when applicable.

## Out-of-band (OOB)

**Signal:** none of the above works (output isn't reflected, no visible
timing/boolean difference, e.g., behind an async worker), but the database
can make outbound network calls.

**Probe/extraction:** trigger a DNS or HTTP request to a domain you
control, encoding data into the subdomain/path (e.g., MSSQL's
`xp_dirtree`, `xp_fileexist` and OPENROWSET-based requests to a UNC path;
Oracle's `UTL_HTTP`/`UTL_INADDR` via XXE-adjacent tricks). This needs
infrastructure (a listener you own) and is only appropriate when it's
within the engagement's scope and your authorization covers egress
testing.

**Why it works:** the database server, not the web app, resolves the
DNS/HTTP request, so you confirm (and exfiltrate via) a side channel the
application never intended to expose.
