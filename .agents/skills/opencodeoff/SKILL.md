---
name: opencodeoff
description: >
  Disables OpenCode auto-delegate mode so coding requests are handled directly again.
  Only when the user explicitly invokes /opencodeoff — never on its own.
license: MIT
user-invocable: true
disable-model-invocation: true
allowed-tools:
  - Bash
---

# /opencodeoff

Run:

```bash
rm -f ~/.local/share/opencode-auto.flag ~/.gemini/config/rules/opencode-auto.md
```

Then confirm: "Auto-opencode OFF — coding requests are handled directly again."

If the removal fails with a permission error under `~/.gemini` (Antigravity's terminal
sandbox blocks writes outside the workspace), re-run the same command outside the sandbox.
