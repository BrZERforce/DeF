# DeF

Claude Code skills.

## Skills

- [`skills-output/offensive-sqli`](skills-output/offensive-sqli/SKILL.md) —
  SQL injection testing workflow for authorized penetration tests,
  bug-bounty engagements, and CTF challenges: detection techniques,
  sqlmap usage, per-database syntax, and a reporting template.

### Using a skill as a session system prompt

Each skill's `SKILL.md` can be piped in as a standalone system prompt to
scope a fresh Claude Code session to that skill:

```bash
cat skills-output/offensive-sqli/SKILL.md | claude --system-file -
```
