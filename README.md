# gemini-skill

A Claude Code skill that delegates coding tasks to **Gemini CLI** and supervises the result.

Claude orchestrates. Gemini codes. You review the diff.

---

## What it does

When you type `/gemini <instruction>` in Claude Code, this skill:

1. Decomposes the task into atomic sub-tasks (if needed)
2. Chooses a mode: `impl` (writes files) or `plan` (read-only exploration)
3. Runs `gemini-delegate` — a shell script that launches Gemini CLI in headless mode
4. Streams structured output: `[read]`, `[write]`, `[plan-write]`, `[WARN]`, `[SYNTAX ERROR]`
5. Runs post-run syntax checks on modified `.py` and `.js` files automatically
6. Reports the git diff and any issues to you
7. Appends a structured JSON entry to `~/.local/share/delegate-runs.jsonl` (tokens, cost, duration, exit code)

---

## Prerequisites

- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed and authenticated (`gemini --version`)
- [Claude Code](https://claude.ai/code) with skills enabled
- `python3` available (for the streaming parser and syntax checks)
- `node` available (optional — for JS syntax checks)
- A git repository to work in

---

## Installation

```bash
git clone https://github.com/pcx-wave/gemini-skill.git && cd gemini-skill && mkdir -p ~/tools ~/.claude/skills/gemini && ln -sf "$(pwd)/tools/gemini-delegate" ~/tools/gemini-delegate && chmod +x ~/tools/gemini-delegate && ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/gemini/SKILL.md
```

### Step-by-step

```bash
# 1. Clone this repo
git clone https://github.com/pcx-wave/gemini-skill.git
cd gemini-skill

# 2. Install the delegate script (symlink — stays in sync with git pull)
mkdir -p ~/tools
ln -sf "$(pwd)/tools/gemini-delegate" ~/tools/gemini-delegate
chmod +x ~/tools/gemini-delegate

# 3. Install the skill for Claude Code
mkdir -p ~/.claude/skills/gemini
ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/gemini/SKILL.md

# 4. (Optional) Enable auto-mode — Claude delegates all code tasks automatically
#    without requiring /gemini each time. Toggle with /geminion and /geminioff.
grep -q "gemini auto-mode" ~/.claude/CLAUDE.md 2>/dev/null || cat >> ~/.claude/CLAUDE.md << 'EOF'

# gemini auto-mode
At the start of every user request that involves writing, editing, or fixing code:
1. Run `test -f ~/.local/share/gemini-auto.flag` (silent, no output to user).
2. If the flag exists → automatically invoke the `gemini` skill exactly as if the user had typed `/gemini <their full instruction>`. Do NOT ask first, do NOT explain — just delegate.
3. If the flag is absent → proceed normally.

The flag is toggled by `/geminion` and `/geminioff`.
EOF
```

### Verify the install

```bash
# Check gemini is available
gemini --version

# Test the delegate script (plan mode — no writes)
~/tools/gemini-delegate /tmp "Say hello in one sentence." 120 plan
# Should print: [gemini] Hello! ...
```

---

## Usage

In a Claude Code session, just describe what you want:

```
/gemini add a dark mode toggle to the settings page
```

```
/gemini the login form is not validating the email field — fix it
```

```
/gemini read the route structure in app.py and summarise it
```

Claude will decompose the task, choose the right mode, write the Gemini prompt,
supervise execution, and report the diff.

---

## Two modes

| Mode   | Flag                    | Writes files? | Use for                           |
|--------|-------------------------|---------------|-----------------------------------|
| `impl` | `--yolo`                | Yes           | Implementing changes (default)    |
| `plan` | `--approval-mode plan`  | No            | Safe exploration, read-only tasks |

In `plan` mode, any write Gemini proposes is shown as `[plan-write]` but never
executed — useful for understanding a codebase before touching it.

---

## How gemini-delegate works

```
Claude Code
  └─ /gemini <instruction>
       └─ SKILL.md logic
            └─ ~/tools/gemini-delegate <workdir> <prompt> [timeout] [mode]
                 ├─ writes prompt to temp file (avoids shell injection with UTF-8/emoji)
                 ├─ selects --yolo (impl) or --approval-mode plan
                 ├─ runs: gemini -p "$PROMPT" $APPROVAL_FLAG -o stream-json
                 │         └─ no pseudo-TTY needed (unlike Vibe)
                 ├─ pipes stream-json events through Python parser
                 │         └─ handles: init / message / tool_call / tool_result / result
                 │         └─ prints [init] / [read] / [write] / [plan-write] / [gemini] / [WARN]
                 ├─ reads real token counts from the stream-json result event
                 ├─ runs syntax checks on modified .py and .js files (skipped in plan mode)
                 ├─ prints git diff --stat (skipped in plan mode)
                 └─ appends JSON entry to ~/.local/share/delegate-runs.jsonl
```

### Why no pseudo-TTY?

Unlike Mistral Vibe, Gemini CLI works fine in a plain pipe — no TTY check on startup.
This simplifies the script significantly compared to `vibe-delegate`.

### Why prompt via temp file?

Inline shell arguments break when the prompt contains Python dict syntax, emojis,
accented characters, or multi-line code. Writing to a temp file avoids all shell
injection issues.

### Why a timeout instead of `--max-turns`?

Gemini CLI has no `--max-turns` flag. The timeout is the only runaway-control lever.
The 503 backoff on the free tier can silently consume 60–90s before work begins —
always budget for this in your timeout value.

---

## Token economics

Gemini's internal tool calls consume **Gemini tokens**, not Claude tokens.
Claude only receives the compressed final output (~200–800 tokens/run).

**Approximate pricing (Gemini 2.5 Flash):**
- ~$0.15/M input tokens, ~$0.60/M output tokens
- Claude Sonnet 4.6: ~$3/M input, ~$15/M output
- Typical ratio: **~20–30x cheaper per token than Claude**

Note: Gemini CLI loads a large system prompt (~900–10k tokens) before each task,
so actual per-run cost is higher than token math alone suggests.

**Free-tier caps (Google AI Studio):**
- ~60 requests/minute, ~1,000 requests/day
- 503 errors = rate cap or demand spike → auto-retried with backoff

Real token counts and cost are printed after every run and appended to the run log.

---

## Known limitations

| Limitation | Detail |
|-----------|--------|
| No `--max-turns` | Timeout is the only runaway control |
| High context overhead | ~900–10k tokens loaded before your task starts |
| 503 backoff | Free tier retries silently for 60–90s |
| No agent selection | Single mode only — use `plan` mode for read-only |
| No `--workdir` flag | Script `cd`s into workdir instead |

---

## Customization

Edit `~/.claude/skills/gemini/SKILL.md`:

- **Known projects table** — list your repos with their absolute paths
- **Timeout defaults** — adjust per project complexity
- **Mode preference** — default to `plan` for exploratory projects

---

## Sister project

A parallel delegate using **Mistral Vibe** instead of Gemini is available at
[pcx-wave/vibe-skill](https://github.com/pcx-wave/vibe-skill).
Same orchestration pattern, same run log format — different model and trade-offs.

---

## License

MIT
