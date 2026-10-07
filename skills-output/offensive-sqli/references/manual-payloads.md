# Manual Test Payloads

Generic strings for finding candidate injection points by hand, before
reaching for sqlmap. Always start with the least invasive probe and only
escalate once a signal confirms the parameter is worth pursuing.

## Universal first probes

| Payload | Why it's useful |
|---|---|
| `'` | Breaks string-quoted queries; if unescaped, often triggers a syntax error. |
| `"` | Same, for double-quote-delimited contexts (less common in SQL, but seen in some drivers). |
| `\` | Can break escaping logic in naive sanitizers that strip `'` but not `\`. |
| `''` (two single quotes) | If the app "fixes" a lone `'` by doubling it, this confirms the escaping behavior so you can craft around it. |
| `id=1` vs `id=1 AND 1=1` vs `id=1 AND 1=2` | Baseline / true / false triple — the core of boolean detection. If `1=1` matches baseline and `1=2` doesn't, the parameter reaches a boolean condition. |
| `id=1 OR 1=1` | If this returns *more* rows than the baseline (e.g., all records instead of one), the input reaches a WHERE clause without proper parameterization. |
| `id=1--` / `id=1#` / `id=1/*` | Comment-based truncation: if appending a comment marker changes behavior, you've confirmed you can terminate the rest of the original query. Marker depends on engine — see `database-syntax.md`. |

## Context-specific placement

SQLi isn't limited to `?id=` query parameters. Test every one of these
the same way:

- **POST body fields** (form-encoded and JSON) — don't assume JSON fields
  are "safer"; many ORMs still build raw query fragments from JSON input.
- **HTTP headers** — `User-Agent`, `X-Forwarded-For`, `Referer`,
  `X-Forwarded-Host` are frequently logged into a database with string
  concatenation and are a classic blind spot.
- **Cookies** — session or preference cookies that get used in a `WHERE`
  clause (e.g., `lang=en` driving a lookup table query).
- **Order-by / sort parameters** — `?sort=name` passed directly into an
  `ORDER BY` clause can't always be parameterized by the app's ORM,
  making it a common injection point even in otherwise-safe apps.
- **Second-order sinks** — data stored safely on write (e.g., a
  parameterized `INSERT`) but later concatenated unsafely into a
  *different* query on read (e.g., an admin report). Test by storing a
  payload in one feature and observing where else it's used.

## Encoding for filtered inputs

When a WAF or naive filter blocks obvious payloads, authorized testing
may include confirming whether encoding bypasses the filter — this
demonstrates the control's actual coverage, which is itself a finding to
report (the WAF is incomplete), not a way to quietly avoid detection on a
system you're not authorized to test.

| Technique | Example |
|---|---|
| URL encoding | `%27` for `'`, `%23` for `#` |
| Double URL encoding | `%2527` for a layer that decodes twice |
| Case variation | `UnIoN SeLeCt` to dodge case-sensitive keyword filters |
| Inline comments as whitespace | `UNION/**/SELECT` where spaces are stripped |
| Equivalent operators | `OR 'a'='a'` instead of `OR 1=1` if digits are filtered |

## Confirming, not over-extracting

Once a signal confirms injection, the smallest useful proof is usually:

```
SELECT @@version          -- MySQL/MSSQL
SELECT version()          -- PostgreSQL
SELECT banner FROM v$version WHERE rownum=1   -- Oracle
SELECT sqlite_version()   -- SQLite
```

Pulling the full `users` table isn't "more thorough" — it's extra risk
and extra load for no added proof value. Extract only what the
engagement's rules of engagement call for.
