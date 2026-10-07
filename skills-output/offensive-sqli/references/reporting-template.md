# Finding Report Template

A SQL injection finding is only as useful as its write-up. Use this
structure for each confirmed injection point.

```markdown
## [Severity] SQL Injection in <endpoint/parameter>

**Location:** `POST /api/items` — `category` parameter (JSON body)
**Injection type:** Boolean-blind
**Database engine:** MySQL 8.0 (confirmed via @@version)

### Description
One-paragraph plain-language explanation: what the parameter does, how
it reaches a SQL query, and why that's exploitable.

### Reproduction steps
1. Send `POST /api/items` with `category=electronics' AND '1'='1`
   → response matches baseline (200, 14 items).
2. Send the same request with `category=electronics' AND '1'='2`
   → response differs (200, 0 items).
3. This confirms the parameter reaches an unparameterized WHERE clause.

### Evidence
- Request/response pair for the true condition (redact session tokens
  before including in the report).
- Request/response pair for the false condition.
- The minimal extracted proof (e.g., `@@version` string), not a full
  data dump, unless the engagement's scope calls for deeper extraction.

### Impact
What an attacker could actually do with this: read other users' data,
bypass authentication, enumerate the schema, pivot to OS access (only if
actually demonstrated and in scope) — be concrete and avoid
overstating beyond what was proven.

### Remediation
1. **Use parameterized queries / prepared statements** for every query
   built from user input — this is the primary fix, not a secondary one.
2. **Avoid string concatenation into SQL**, including in ORM "raw query"
   escape hatches.
3. **Apply least privilege to the DB account** the application uses — a
   read-only, narrowly-scoped account limits blast radius even if another
   injection is found later.
4. **Turn off verbose database error messages** in production responses.
5. **A WAF is defense-in-depth, not a substitute** for fixing the query —
   note this explicitly if the client's instinct will be to just add a
   WAF rule.

### References
- OWASP SQL Injection Prevention Cheat Sheet
- CWE-89
```

## Severity guidance

Base severity on demonstrated impact, not theoretical worst case:

- **Critical**: unauthenticated, leads to full database read/write, or
  chains to OS command execution (actually demonstrated, in scope).
- **High**: authenticated but low-privilege account can read other
  users' sensitive data, or any account can read highly sensitive data
  (credentials, PII) via the injection.
- **Medium**: blind injection confirmed but extraction is slow/limited,
  or requires an already-privileged account to reach.
- **Low**: injection confirmed in a context with minimal data exposure
  (e.g., a non-sensitive lookup table) and no practical path to more.

## What not to include

- Full dumps of sensitive tables (credentials, PII) — reference that
  they were extracted and where, without pasting raw sensitive data into
  the report body. Hand sensitive evidence over through the engagement's
  agreed secure channel instead.
- Any action taken outside the authorized scope or rules of engagement —
  if something out-of-scope was noticed incidentally, report it
  separately and ask before testing it further.
