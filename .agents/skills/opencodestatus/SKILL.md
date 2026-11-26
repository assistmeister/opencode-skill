---
name: opencodestatus
description: >
  Shows whether OpenCode auto-delegate mode is ON or OFF in Claude Code and Antigravity.
  Only when the user explicitly invokes /opencodestatus — never on its own.
license: MIT
user-invocable: true
disable-model-invocation: true
allowed-tools:
  - Bash
---

# /opencodestatus

Run:

```bash
test -f ~/.local/share/opencode-auto.flag && echo "Claude Code flag : ON" || echo "Claude Code flag : OFF"
test -f ~/.gemini/config/rules/opencode-auto.md && echo "Antigravity rule: ON" || echo "Antigravity rule: OFF"
```

Print the two lines verbatim. If they disagree, say so in one line and suggest running
`/opencodeon` or `/opencodeoff` to sync them.
