---
name: opencode-report
description: >
  Shows the delegation run log — tokens, cost, duration and failures across past
  OpenCode and other delegate runs. Only when the user explicitly invokes
  /opencode-report — never on its own.
license: MIT
user-invocable: true
disable-model-invocation: true
allowed-tools:
  - Bash
---

# /opencode-report

Run `~/tools/delegate-report` with the flags that match the user's arguments, print the
output verbatim, and stop.

| User says | Flag |
|-----------|------|
| last 7 days, 7d | `--since 7` |
| last 30 days, 30d | `--since 30` |
| project foo | `--project foo` |
| only failures, fails | `--fails` |
| (nothing) | (no flags, full report) |

```bash
~/tools/delegate-report
```
