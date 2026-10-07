---
name: offensive-sqli
description: Use this skill when performing SQL injection testing during an authorized penetration test, security assessment, bug-bounty engagement, or CTF challenge. Covers the testing workflow, detection techniques (error-based, boolean-blind, time-blind, UNION, out-of-band), sqlmap usage, and database-specific syntax for MySQL, PostgreSQL, MSSQL, Oracle, and SQLite. Requires explicit written authorization or a lab/CTF context — refuse to use it against any target without one.
---

# Offensive SQL Injection Testing

SQL injection (SQLi) testing finds places where untrusted input reaches a SQL
query unsafely, letting a tester read, and sometimes write, data the
application never meant to expose. This skill is for **authorized** work:
a signed pentest engagement, a bug-bounty program's in-scope asset, or a
CTF/training lab. It is not for use against any system without that
authorization.

## 0. Authorization gate — check before anything else

Before running a single payload, confirm:

1. **Written authorization or an explicit lab/CTF context exists** — a
   signed scope document, a bug-bounty program's rules, or a
   self-contained CTF/training environment. If none of these is true,
   **stop and say so** instead of testing.
2. **The specific target is in scope.** A pentest authorization for
   `app.example.com` does not cover `partner-app.example.com`.
3. **Rules of engagement are known**: allowed testing windows, rate
   limits, whether data extraction (vs. detection only) is permitted, and
   who to contact if something breaks.
4. **Destructive statements are off-limits unless explicitly authorized.**
   Never run `DROP`, `DELETE`, `UPDATE`, `TRUNCATE`, or `xp_cmdshell`-style
   OS command execution as part of routine testing — these go far beyond
   proving the vulnerability exists and can cause real damage or outages.
   Read-only proof (e.g., `SELECT version()`, extracting a single
   non-sensitive row) is almost always enough to demonstrate impact.
5. **Keep load low.** SQLi testing (especially time-based blind and
   sqlmap's default settings) can generate heavy request volume. Throttle
   requests (`--delay`, `--threads` in sqlmap) to stay within what the
   engagement's rules of engagement allow, and never test a target
   you were not specifically told to load-test.

If any of this is missing or unclear, ask the user rather than
proceeding.

## 1. Workflow

1. **Map inputs** — every parameter, header, cookie, JSON/XML field, and
   file-upload metadata field that reaches a database query. *Why:*
   SQLi isn't limited to URL query strings; `User-Agent`, `X-Forwarded-For`,
   and JSON bodies are common blind spots.
2. **Probe for signals, not exploitation, first** — send a single quote
   `'`, a double quote `"`, and a harmless boolean pair (see
   `references/manual-payloads.md`) and compare responses. *Why:* this
   finds candidates cheaply before committing to heavier tooling.
3. **Classify the injection type** — error-based, boolean-blind,
   time-blind, UNION-based, or out-of-band (see
   `references/detection-techniques.md`). *Why:* the type determines which
   extraction technique and sqlmap flags to use.
4. **Confirm with a minimal, reversible proof** — e.g., a version
   string or a single harmless row. *Why:* proof of concept should be the
   smallest action that demonstrates impact, per the authorization gate above.
5. **Automate extraction only after manual confirmation**, using sqlmap
   (see `references/sqlmap-cheatsheet.md`) tuned to the confirmed
   injection point and database engine (see
   `references/database-syntax.md`). *Why:* pointing sqlmap at an
   unconfirmed parameter wastes requests and risks noisy, unscoped traffic.
6. **Document and report** using the template in
   `references/reporting-template.md`. *Why:* an engagement's value is the
   fix, not the shell — every finding needs reproduction steps, evidence,
   impact, and remediation.

## 2. Reference files (load as needed)

- `references/detection-techniques.md` — how to recognize each SQLi class
  from application behavior.
- `references/manual-payloads.md` — generic test strings for each
  injection context (GET/POST/JSON/headers/cookies) and why each one works.
- `references/sqlmap-cheatsheet.md` — common sqlmap invocations, from
  initial detection through `--dbs`/`--tables`/`--dump`, request-file and
  proxy usage, and risk/level tuning.
- `references/database-syntax.md` — per-engine cheatsheet (MySQL,
  PostgreSQL, MSSQL, Oracle, SQLite): comment syntax, version/string
  functions, UNION column-count discovery, time-delay functions, and
  stacked-query support.
- `references/reporting-template.md` — finding write-up structure and
  standard remediation advice (parameterized queries, least-privilege DB
  accounts, WAF, input validation).

## 3. What this skill will not help with

- Testing any target without confirmed authorization.
- Destructive payloads (data deletion/modification) outside an explicit,
  authorized scope that calls for them.
- Mass/automated scanning across hosts not individually in scope.
- Using findings for anything beyond the engagement's reporting (e.g.,
  exfiltrating data for personal use, persistence/backdoors, or lateral
  movement not covered by the engagement).
