---
name: opencodeon
description: >
  Enables OpenCode auto-delegate mode so coding requests go to the OpenCode CLI
  without typing /opencode each time. Only when the user explicitly invokes
  /opencodeon — never on its own.
license: MIT
user-invocable: true
disable-model-invocation: true
allowed-tools:
  - Bash
---

# /opencodeon

Run this once (it may need approval to write outside the workspace):

```bash
mkdir -p ~/.local/share ~/.gemini/config/rules
touch ~/.local/share/opencode-auto.flag

cat > ~/.gemini/config/rules/opencode-auto.md <<'EOF'
---
trigger: always_on
description: Auto-delegates coding tasks to the OpenCode CLI while auto-mode is on.
---
# OpenCode auto-mode

When a user request involves writing, editing or fixing code, invoke the `opencode`
skill as if the user had typed `/opencode <their full instruction>`. Do not ask first
and do not explain — delegate. Handle a one-line change in a known location directly.

This file exists only while auto-mode is on; `/opencodeoff` deletes it, together with
`~/.local/share/opencode-auto.flag`.
EOF
```

Then confirm: "Auto-opencode ON — coding requests will be delegated to OpenCode."

Claude Code reads the flag file through the `# opencode auto-mode` block in `CLAUDE.md`.
Antigravity reads the rule file; no `CLAUDE.md` edit is needed there.

If the write fails with a permission error under `~/.gemini` (Antigravity's terminal
sandbox blocks writes outside the workspace), re-run the same command outside the sandbox.
