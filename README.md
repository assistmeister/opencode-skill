# opencode-skill

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Your agent plans. OpenCode writes the code. You read the diff.**

A skill for [Claude Code](https://claude.ai/code) and [Google Antigravity](https://antigravity.google) that hands coding work to the [OpenCode CLI](https://opencode.ai) and supervises it. You stay in one chat. OpenCode makes the edits. Nothing gets committed without you seeing it.

## Why this exists

Two coding agents in one session get messy. You end up with two tool sets and two context windows, and each agent forms its own picture of the codebase. This skill keeps one agent in charge. The one you are already talking to plans the work, writes a single prompt, and runs OpenCode as a background job. Then it reads the output, checks the diff, and tells you what happened.

OpenCode burns its own tokens on the mechanical part, so your orchestrator sees a summary instead of a whole transcript. That usually comes to a few hundred tokens per run.

The project started as a fork of [pcx-wave/opencode-skill](https://github.com/pcx-wave/opencode-skill). The credits section covers that. This copy has drifted since: Antigravity support, portable scripts, the current CLI, and a different default model.

## What you need

- [OpenCode CLI](https://opencode.ai) v1 or v2 (`opencode --version`). The free tier needs 1.18.0 or newer.
- One host: Claude Code with skills enabled, or Antigravity (App, IDE, or the `agy` CLI).
- `python3`, for the stream parser and the syntax checks.
- GNU-style `timeout`. macOS does not ship it, so run `brew install coreutils`. The script uses `gtimeout` when it finds it.
- A git repository to work in. OpenCode never commits, so `git checkout .` is your undo button.

## Install

Clone once, then link the scripts and the skills. Because the install uses links, `git pull` updates everything.

```bash
git clone https://github.com/assistmeister/opencode-skill.git
cd opencode-skill

# Scripts
mkdir -p ~/tools
ln -sf "$PWD/tools/opencode-delegate" ~/tools/opencode-delegate
ln -sf "$PWD/tools/delegate-report" ~/tools/delegate-report
chmod +x "$PWD/tools/opencode-delegate" "$PWD/tools/delegate-report"
```

Then install for your host. You can do both; they do not conflict.

### Claude Code

Claude loads skills from `~/.claude/skills/<name>/SKILL.md`, one folder per skill. This loop links all seven:

```bash
mkdir -p ~/.claude/skills
for d in .agents/skills/*/; do
  n=$(basename "$d")
  mkdir -p ~/.claude/skills/"$n"
  ln -sf "$PWD/$d/SKILL.md" ~/.claude/skills/"$n"/SKILL.md
done
```

Start a new Claude Code session, or restart the current one, and type `/`. You should see `/opencode`, `/opencodeon`, `/opencodeoff`, `/opencodestatus`, `/opencode-report`, `/opencode-model-pick`, and `/opencode-model-clear`.

Claude's auto-mode needs one more step. Append this block to `~/.claude/CLAUDE.md` once:

```bash
grep -q "opencode auto-mode" ~/.claude/CLAUDE.md 2>/dev/null || cat >> ~/.claude/CLAUDE.md << 'EOF'

# opencode auto-mode
At the start of every user request that involves writing, editing, or fixing code:
1. Run `test -f ~/.local/share/opencode-auto.flag` (silent, no output to user).
2. If the flag exists → automatically invoke the `opencode` skill exactly as if the user had typed `/opencode <their full instruction>`. Do NOT ask first. Do NOT explain. Just delegate.
3. If the flag is absent → proceed normally.

The flag is toggled by `/opencodeon` and `/opencodeoff`.
EOF
```

`CLAUDE.md` applies to every project on the machine. `/opencodeoff` still turns the behaviour off, because it removes the flag the block checks.

### Antigravity

Antigravity reads skills from two places:

- `~/.gemini/config/skills/<name>/SKILL.md` for every project.
- `<repo>/.agents/skills/<name>/SKILL.md` for the project in that repo.

For day to day work, link the global copy:

```bash
mkdir -p ~/.gemini/config/skills
for d in .agents/skills/*/; do
  n=$(basename "$d")
  mkdir -p ~/.gemini/config/skills/"$n"
  ln -sf "$PWD/$d/SKILL.md" ~/.gemini/config/skills/"$n"/SKILL.md
done
```

Open a project in the Antigravity App, or run `agy` in its folder, then reload the skills:

```bash
/skills reload
```

They now show up as slash commands and under Customizations in the agent panel. The agent also picks one on its own when your request matches the skill description.

The CLI reads `~/.gemini/antigravity-cli/skills/` and `~/.gemini/skills/` as well, and those folders win over `~/.gemini/config/skills/`. If an old copy of this skill turns up, delete it from both.

If you open the cloned repo itself in Antigravity, the skills in `.agents/skills/` load from the workspace without any global link. That is handy for testing a change before you install it.

Auto-mode installs itself here. `/opencodeon` writes `~/.gemini/config/rules/opencode-auto.md` as an `always_on` rule, and `/opencodeoff` deletes it. There is no `CLAUDE.md` edit or snippet to paste.

#### The terminal sandbox blocks the first run

Under the default permission preset, Antigravity runs shell commands in a sandbox with no network, and writes limited to the workspace and temp dirs. OpenCode needs the network, so the delegate has to run outside that sandbox. You handle this once:

1. Say yes when the agent asks to run outside the sandbox, or
2. Add an allow rule under Settings > Permissions:

```
command(~/tools/opencode-delegate)
```

`/opencodeon` and `/opencodeoff` write under `~/.gemini`, so they may ask for the same approval once. That is expected.

## First run

Try it in any git repo you do not mind changing:

```bash
/opencode add a hello.py that prints "hi" and exits 0
```

You will see the run start, then the parsed events, the syntax check, and the diff. Output arrives when the run ends.

You can also call the delegate directly, with no host involved:

```bash
~/tools/opencode-delegate /path/to/repo "Create hello.py that prints hi." 180
```

That is the fastest way to confirm OpenCode itself is working.

## Commands

| Command | What it does |
|---|---|
| `/opencode <task>` | Plan, delegate, watch, report. The main one. |
| `/opencodeon` | Delegate coding requests automatically from now on. |
| `/opencodeoff` | Back to normal. |
| `/opencodestatus` | Reports whether auto-mode is on, for each host. |
| `/opencode-report` | Token and cost history from the run log. |
| `/opencode-model-pick <model>` | Use a different OpenCode model until you clear it. |
| `/opencode-model-clear` | Return to the default model. |

`/opencode-report` takes the same filters as the script behind it: `7d` or `30d` for a window, `project foo` for one project, and `fails` for the runs that broke.

## Auto-mode

With auto-mode on, the host agent may hand any coding request to OpenCode without asking. That suits a bug fix. It does not suit a request like "explain this function", because there is no code change to delegate.

Auto-mode is on when both of these files exist:

- Claude: `~/.local/share/opencode-auto.flag`
- Antigravity: `~/.gemini/config/rules/opencode-auto.md`

`/opencodestatus` checks both and tells you when they disagree. That happens if you toggled in one host and then switched to the other. Run the toggle again in the host you are using.

## What a run does

```
you: /opencode <task>
  └─ the skill decides: small enough to do directly, or worth delegating?
       └─ writes a self-contained prompt (stack, files, constraints, grep check)
            └─ ~/tools/opencode-delegate <workdir> <prompt> [timeout] [model]
                 ├─ writes the prompt to a temp file, so quoting cannot break it
                 ├─ runs: opencode run <prompt> --format json
                 │    --auto (or --dangerously-skip-permissions on old v1)
                 │    --model <model>, --dir <workdir> when the CLI supports it
                 ├─ prints events: [read] [edit] [write] [search] [shell] [opencode]
                 ├─ adds up real token counts and cost
                 ├─ syntax-checks every changed .py .js .json .sh file
                 ├─ prints the git diff
                 └─ appends one line to ~/.local/share/delegate-runs.jsonl
```

The script does not commit, stage, or touch files outside the workdir you give it. If a run goes sideways, `git diff` shows how far it got, and `git checkout .` puts things back.

The prompt goes through a temp file. Inline arguments break on quotes, emojis, accented text, and multi-line code.

OpenCode does not need a pseudo-TTY, unlike Mistral Vibe, so a plain pipe works.

OpenCode has no `--max-turns`, which makes the timeout your only brake. Keep it tight and split large jobs.

## Cost

OpenCode pays for the token-heavy part on its own account. Your orchestrator gets a summary, usually a few hundred tokens per run.

The default model is `opencode-go/deepseek-v4.1-flash#high`, included with an OpenCode Go subscription. The Go docs list no training on your data and 0-day retention. You need the Global region enabled under Settings > Privacy in your workspace.

If you would rather not use Go, these work too:

| Model | Notes |
|---|---|
| `opencode-go/deepseek-v4.1-flash#high` | Default. Go subscription, zero retention per the Go docs. |
| `opencode/glm-5.3-flash-free` | Free, no API key. Queues at peak times. |
| `google/gemini-2.5-flash` | Needs `GOOGLE_GENERATIVE_AI_API_KEY`. |

Switch the model for one run with the fourth argument, or for everything with `/opencode-model-pick`.

## Known limits

| Limit | What it means for you |
|---|---|
| No turn limit | The timeout is your only brake. Decompose large jobs. |
| Tools are pre-approved | OpenCode runs with `--auto`. Reviewing the diff is the safety net. |
| Free models queue | Expect slow runs at busy hours. |
| No `--dir` in v2 | The script changes into the workdir instead. Same result. |
| Needs GNU `timeout` | macOS: `brew install coreutils`. Windows: WSL, or Git Bash with coreutils. |
| Antigravity sandbox | No network inside it. The delegate must run unsandboxed, as described above. |
| Output timing | `opencode run` output is captured to a file and printed when the process exits. opencode v2 leaves service and pty daemons holding inherited stdout, so a live pipe would never close and the calling shell would hang. |
| Windows symlinks | Git Bash copies instead of linking unless Developer Mode and `MSYS=winsymlinks:nativestrict` are set. Re-run the install loop after each pull. |
| Frontend syntax coverage | `.ts` and `.tsx` are checked with `tsc` only when a `tsconfig.json` exists. |

## Run log

Every run appends one JSON line to `~/.local/share/delegate-runs.jsonl`: model, tokens, cost, duration, exit code, files changed, whether it timed out, and whether OpenCode talked but wrote nothing.

```bash
~/tools/delegate-report                 # everything
~/tools/delegate-report --since 7       # last week
~/tools/delegate-report --fails         # what went wrong
~/tools/delegate-report --project myapp # one project
```

[vibe-skill](https://github.com/pcx-wave/vibe-skill) and [gemini-skill](https://github.com/pcx-wave/gemini-skill) write to the same file, so the numbers stay comparable if you switch tools.

Under the Antigravity sandbox the log write is refused. The script prints `[log] SKIPPED` and carries on instead of failing quietly. Running the delegate unsandboxed fixes it.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/opencode` does not appear in Claude | Check that `~/.claude/skills/opencode/SKILL.md` exists, then restart the session. |
| `/opencode` does not appear in Antigravity | Run `/skills reload` and check Customizations. Delete any older copy in `~/.gemini/skills/` or `~/.gemini/antigravity-cli/skills/`, because those win. |
| The run dies instantly with a network error | You are inside the Antigravity sandbox. Approve the unsandboxed run or add the allow rule. |
| `ERROR: GNU timeout not found` | macOS needs coreutils. Windows `timeout.exe` is a different program, and the script rejects it on purpose. |
| `ERROR: python3 not found` | Install Python and make sure it is on the `PATH` your host uses. |
| The log says `wrote_nothing: true` | OpenCode answered but edited nothing. Name the file and the change more precisely, then run it again. |

## Credits

This began as [pcx-wave/opencode-skill](https://github.com/pcx-wave/opencode-skill), MIT licensed. The original design, the delegate pattern, and most of the skill text are theirs. If you want the upstream version, it is still there.

This fork adds:

- Skill folders laid out for Antigravity's `.agents/skills/` discovery, installed for both hosts from one source.
- An auto-mode toggle that writes a native Antigravity rule instead of editing a Claude-only file.
- Portability fixes: `gtimeout`, BSD `mktemp`, a Python clock that does not print `N` on macOS, and clear errors when something is missing.
- Sandbox guidance, plus a log failure that says so.
- `opencode-go/deepseek-v4.1-flash#high` as the default model.

Same lineage, and this fork is maintained here now. Bugs in this fork belong to this fork, not to upstream.

Sister projects from the same family: [vibe-skill](https://github.com/pcx-wave/vibe-skill) for Mistral Vibe, and [gemini-skill](https://github.com/pcx-wave/gemini-skill) for Gemini CLI.

## License

MIT. See [LICENSE](LICENSE) if it is present; otherwise the upstream license text applies.
