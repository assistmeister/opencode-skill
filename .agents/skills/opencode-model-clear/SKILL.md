---
name: opencode-model-clear
description: >
  Clears the OpenCode model override and returns to the default model. Only when the
  user explicitly invokes /opencode-model-clear — never on its own.
license: MIT
user-invocable: true
disable-model-invocation: true
allowed-tools:
  - Bash
---

# /opencode-model-clear

Run: `rm -f ~/.local/share/opencode-model.flag`

Then confirm: "OpenCode model override cleared — back to default (opencode-go/deepseek-v4.1-flash#high)."
