# opencode-skill

> Fork of [pcx-wave/opencode-skill](https://github.com/pcx-wave/opencode-skill) (MIT). Adds upstream [PR 1](https://github.com/pcx-wave/opencode-skill/pull/1) (OpenCode v1/v2 support), LF line endings for Windows checkouts, `opencode-go/deepseek-v4.1-flash#high` as the default model, and first-class Google Antigravity support (skills under `.agents/skills/`, always-on rule for auto-mode, sandbox guidance).

![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)

**The host agent orchestrates. OpenCode codes. You review the diff.**

A skill for **Claude Code** and **Google Antigravity** (App, IDE, CLI) that delegates coding tasks to **OpenCode CLI** and supervises the result — the same pattern as [vibe-skill](https://github.com/pcx-wave/vibe-skill) and [gemini-skill](https://github.com/pcx-wave/gemini-skill).

---

## What it does

When you type `/opencode <instruction>` in Claude Code or Antigravity, this skill:

1. Decomposes the task into atomic sub-tasks (if needed)
2. Writes a self-contained prompt for OpenCode
3. Runs `opencode-delegate` — a script that launches OpenCode CLI in non-interactive mode
4. Streams structured output: `[read]`, `[edit]`, `[write]`, `[search]`, `[shell]`, `[opencode]`
5. Runs post-run syntax checks on modified `.py`, `.js`, `.json`, `.sh` files automatically
6. Reports the git diff and any issues to you
7. Appends a structured JSON entry to `~/.local/share/delegate-runs.jsonl` (tokens, cost, duration, exit code)

---

## Prerequisites

- [OpenCode CLI](https://opencode.ai) v1 or v2 installed (`opencode --version`; the free tier needs ≥1.18.0)
- One host: [Claude Code](https://claude.ai/code) with skills enabled, or [Google Antigravity](https://antigravity.google) (App, IDE, or CLI `agy`)
- `python3` available (for the streaming parser and syntax checks)
- GNU-style `timeout` — on macOS: `brew install coreutils` (the script picks up `gtimeout`)
- A git repository to work in

---

## Installation

Antigravity discovers skills in `.agents/skills/`; Claude Code reads `~/.claude/skills/`.
Both installs are symlinks, so `git pull` keeps them current.

```bash
git clone https://github.com/assistmeister/opencode-skill.git
cd opencode-skill

# 1. Install the scripts (symlinks — stay in sync with git pull)
mkdir -p ~/tools
ln -sf "$PWD/tools/opencode-delegate" ~/tools/opencode-delegate
ln -sf "$PWD/tools/delegate-report" ~/tools/delegate-report
chmod +x "$PWD/tools/opencode-delegate" "$PWD/tools/delegate-report"

# 2. Install the skills for Antigravity (App, IDE and CLI)
mkdir -p ~/.gemini/config/skills
for d in .agents/skills/*/; do n=$(basename "$d"); mkdir -p ~/.gemini/config/skills/"$n"
  ln -sf "$PWD/$d/SKILL.md" ~/.gemini/config/skills/"$n"/SKILL.md; done

# 3. Install the skills for Claude Code
mkdir -p ~/.claude/skills
for d in .agents/skills/*/; do n=$(basename "$d"); mkdir -p ~/.claude/skills/"$n"
  ln -sf "$PWD/$d/SKILL.md" ~/.claude/skills/"$n"/SKILL.md; done
```

### Antigravity quickstart

```bash
agy             # inside the project OpenCode should work on
/skills reload  # after installing or updating
/opencode add a dark mode toggle to the settings page
```

Every skill becomes a slash command (`/opencode`, `/opencodeon`, `/opencodestatus`,
`/opencode-report`, …). Skills committed under `.agents/skills/` are also discovered when
you open this repository itself.

Antigravity additionally loads `~/.gemini/antigravity-cli/skills/` and `~/.gemini/skills/`,
which **override** `~/.gemini/config/skills/`. Remove stale copies there if an old version
wins.

Auto-mode needs no `CLAUDE.md` edit in Antigravity: `/opencodeon` writes
`~/.gemini/config/rules/opencode-auto.md` (an `always_on` rule) and `/opencodeoff` deletes it.

### Antigravity terminal sandbox

The default permission preset runs agent shell commands in a sandbox with no network and
writes limited to the workspace and temp dirs. The delegate needs network access, so run it
outside the sandbox — approve the agent's request, or add to the Allow list:

```
command(~/tools/opencode-delegate)
```

Toggling auto-mode writes under `~/.gemini`, so `/opencodeon` and `/opencodeoff` may need
one approval too.

### Claude Code auto-mode

Append the auto-mode block once (Claude only — Antigravity uses the rule file above):

```bash
grep -q "opencode auto-mode" ~/.claude/CLAUDE.md 2>/dev/null || cat >> ~/.claude/CLAUDE.md << 'EOF'

# opencode auto-mode
At the start of every user request that involves writing, editing, or fixing code:
1. Run `test -f ~/.local/share/opencode-auto.flag` (silent, no output to user).
2. If the flag exists → automatically invoke the `opencode` skill exactly as if the user had typed `/opencode <their full instruction>`. Do NOT ask first, do NOT explain — just delegate.
3. If the flag is absent → proceed normally.

The flag is toggled by `/opencodeon` and `/opencodeoff`.
EOF
```

### Verify the install

```bash
opencode --version
~/tools/opencode-delegate /tmp "Say hello in one sentence." 60
# Should print: [opencode] Hello! ...

test -f ~/.gemini/config/skills/opencode/SKILL.md && echo "Antigravity skill linked"
```

### Updating

Because the install uses symlinks, a `git pull` is all you need:

```bash
cd ~/opencode-skill && git pull
```

`~/tools/opencode-delegate` and all skill files are automatically up to date — no re-copy needed.
On Windows, Git Bash copies instead of symlinking unless Developer Mode and
`MSYS=winsymlinks:nativestrict` are set; re-run the install loop after each pull.

---

## Usage

In a Claude Code or Antigravity session, just describe what you want:

```
/opencode add a dark mode toggle to the settings page
```

```
/opencode the login form is not validating the email field — fix it
```

Claude will decompose the task, write the OpenCode prompt, supervise execution, and report the diff.

---

## Auto-delegate mode

For frictionless delegation, enable auto-mode once in your session:

```
/opencodeon      — every code request is automatically delegated to OpenCode
/opencodeoff     — return to normal Claude behaviour
/opencodestatus  — check if auto-mode is ON or OFF
```

With `opencodeon` active, just talk to Claude normally — coding tasks go to OpenCode automatically.

---

## How opencode-delegate works

```
Claude Code
  └─ /opencode <instruction>
       └─ SKILL.md logic
            └─ ~/tools/opencode-delegate <workdir> <prompt> [timeout] [model]
                 ├─ writes prompt to temp file (avoids shell injection)
                 ├─ runs: opencode run <prompt> --format json
                 │   --model <model> + version-specific flags (run from inside <workdir>)
                 │   └─ --dangerously-skip-permissions (v1 ≤1.15) or --auto (v1.19+, v2),
                 │      plus --dir <workdir> when supported (detected from `opencode run --help`)
                 │   └─ no pseudo-TTY needed (unlike Vibe)
                 ├─ pipes JSON events through Python parser
                 │   └─ handles: step_start / tool_use / text / step_finish / error
                 │   └─ prints [read] / [edit] / [write] / [search] / [shell] / [opencode]
                 ├─ aggregates real token counts from step_finish events
                 ├─ runs syntax checks on modified .py, .js, .json, .sh files
                 ├─ prints git diff --stat
                 └─ appends JSON entry to ~/.local/share/delegate-runs.jsonl
```

### Why no pseudo-TTY?

Unlike Mistral Vibe, OpenCode CLI works fine in a plain pipe — no TTY check on startup.

### Why prompt via temp file?

Inline shell arguments break when the prompt contains Python dict syntax, emojis, accented characters, or multi-line code. Writing to a temp file avoids all shell injection issues.

### Why a timeout instead of `--max-turns`?

OpenCode CLI has no `--max-turns` flag. The timeout is the only runaway-control lever.

---

## Token economics

OpenCode's internal tool calls consume **OpenCode tokens**, not Claude tokens.
Claude only receives the compressed final output (~200-800 tokens/run).

**Pricing depends on the model used:**
- `opencode-go/deepseek-v4.1-flash#high` — **default**, OpenCode Go subscription; the Go docs list no training and 0-day retention
- `opencode/glm-5.3-flash-free` — **free**, no API key needed (may have queue delays)
- `google/gemini-2.5-flash` — requires `GOOGLE_GENERATIVE_AI_API_KEY` env var

Real token counts and cost are printed after every run and appended to the run log.

---

## Known limitations

| Limitation | Detail |
|-----------|--------|
| No `--max-turns` | Timeout is the only runaway control |
| `--auto` / `--dangerously-skip-permissions` (v1 ≤1.15) | All tools auto-approved (safe with post-run review); flag auto-detected |
| Free model delays | May have queue during peak usage |
| No `--dir` flag in v2 | Script `cd`s into workdir (v1 also gets `--dir`) |
| Needs GNU `timeout` | Not in macOS by default — `brew install coreutils`; the script uses `gtimeout` when present |
| Antigravity terminal sandbox | Default preset has no network and no write access outside the workspace; the delegate must run unsandboxed (`command(~/tools/opencode-delegate)`) |
| Windows | Git Bash copies instead of symlinking unless Developer Mode + `MSYS=winsymlinks:nativestrict`; `timeout.exe` is rejected, so use WSL or Git Bash with GNU coreutils |

---

## Run Log

Every run appends one JSON entry to `~/.local/share/delegate-runs.jsonl`.
The shared `~/tools/delegate-report` script can query all runs across delegates.

```bash
~/tools/delegate-report                  # full report
~/tools/delegate-report --since 7        # last 7 days
~/tools/delegate-report --fails          # failures only
```

---

## Sister projects

- [vibe-skill](https://github.com/pcx-wave/vibe-skill) — delegate to Mistral Vibe
- [gemini-skill](https://github.com/pcx-wave/gemini-skill) — delegate to Gemini CLI

All three write to the same `delegate-runs.jsonl` log, making runs comparable across delegates.

---

## License

MIT
