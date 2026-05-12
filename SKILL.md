---
name: gemini
description: >
  Delegate a coding task to Gemini CLI and supervise the result via git diff.
  Trigger: /gemini <instruction>. Claude orchestrates, Gemini codes.
license: MIT
user-invocable: true
allowed-tools:
  - bash
  - read_file
  - grep
---

# Gemini Orchestrator

When the user invokes `/gemini <instruction>`, Claude delegates the implementation
to Gemini CLI via its headless mode (`-p/--prompt`), monitors in real time, and reports.

---

## Known Limits

Hard constraints of the Gemini CLI — not config options.

### 1. No `--max-turns` flag
Vibe lets you cap turn count (`--max-turns 8`). Gemini CLI has no equivalent.
**Timeout is the only runaway-control lever.** A stuck run burns the full timeout
before dying. Set timeouts conservatively and decompose tasks.

### 2. High context overhead (~900–10k tokens before your task starts)
Gemini CLI loads a large default system prompt on every run:
- Simple prompt → ~883 tokens before the model responds
- File-read task → ~10k tokens of context before first tool call

This means:
- Each run costs more than token-naive estimates suggest
- Short timeouts can expire during context-loading on a slow connection
- The overhead is mostly cached on repeated calls to the same model in a session

### 3. 503 backoff eats your timeout silently
On the free-tier Gemini API, the model is frequently "under high demand."
The CLI auto-retries with exponential backoff — observed taking **60–90s** before
work even begins. This is invisible until you see the first tool call.

Always add 90s buffer to your "real work" estimate:
```
Timeout budget = expected_work_secs + 90s backoff buffer + 30s context load
```

### 4. No `--agent` flag
Gemini CLI is single-mode only. There is no way to switch to a review-only or
plan-only agent. Use `plan` mode (`--approval-mode plan`) as a partial substitute.

### 5. No `--workdir` flag
The delegate script handles this by `cd`-ing into the workdir before running.

### 6. No pseudo-TTY needed (positive difference vs Vibe)
Gemini CLI works fine in a plain pipe — no `script -q -c` wrapper needed.

---

## Known projects

<!-- Customize this table for your own projects -->
| Name | Path |
|------|------|
| my-project | /path/to/my-project |

---

## Step 1 — Detect workdir

1. If the instruction mentions a known project → use its path.
2. Otherwise: `git rev-parse --show-toplevel` in the current directory.
3. If ambiguous → ask with `AskUserQuestion`.

---

## Step 2 — Choose mode

| Mode   | Flag                    | Writes files? | Use for                             |
|--------|-------------------------|---------------|-------------------------------------|
| `impl` | `--yolo`                | Yes           | Implementing changes (default)      |
| `plan` | `--approval-mode plan`  | No            | Safe exploration, reading, planning |

Use `plan` mode when you want Gemini to read the codebase and report back without
touching any files. Proposed writes appear as `[plan-write]` and are blocked.

---

## Step 3 — Decompose the task

**Critical rule**: Gemini works best on **atomic, focused tasks**.
Given the context overhead and 503 risk, keep tasks smaller than you might expect.

| Size | Definition | Approach |
|------|-----------|----------|
| **Simple** | 1 file, 1 clear change | 1 gemini call, impl mode |
| **Medium** | 2–3 related files, 1 goal | 1 gemini call with structured prompt |
| **Complex** | >3 files OR business logic OR DB migrations | **Decompose** |

**Decomposition for complex tasks:**
```
Sub-task 1: Explore relevant files — plan mode, 120s
Sub-task 2: Implement change A in file X — impl mode, 180s
Sub-task 3: Implement change B in file Y — impl mode, 180s
Sub-task 4: Verify / test — plan mode, 120s
```
→ Check git diff between sub-tasks before launching the next.

---

## Step 4 — Write the Gemini prompt

Gemini has no context from the parent conversation. The prompt must be **self-contained**.

**Structure of a good Gemini prompt:**
```
Stack: Python/Flask, SQLAlchemy, SQLite
Key files: app.py (routes + fetch), models.py (Entry)

TASK: [one single thing to do, stated as an imperative]

CONSTRAINTS:
- [what must not break]
- [expected format if relevant]

VERIFY: grep for "def function_name" in file.py and confirm it exists.
```

**Formulation rules:**
- One task per prompt — never "also do X and Y"
- Name the exact files to modify
- Include a grep-based verification criterion (not a file re-read)
- Language: English (better Gemini performance)
- Keep prompts under ~500 words — longer prompts increase context overhead

**Verification — always use grep, not file re-read:**
```
VERIFY: grep for "def extract_labels" in app.py and confirm it exists.
```
A grep is unambiguous. A file re-read can miss content outside the context window.

**Examples:**

❌ Bad (too vague, too wide):
```
Fix the API, add a signal classifier, update the UI with colored badges
```

✅ Good (atomic, verifiable):
```
Stack: Python/Flask. File: app.py

TASK: In fetch_data(), convert the date string (format "YYYY-MM-DD")
to datetime.date before returning, and convert id to str.

VERIFY: grep for "datetime.date" in app.py and confirm it exists.
```

---

## Step 5 — Launch Gemini

```bash
~/tools/gemini-delegate "<workdir>" "<prompt>" [timeout-secs] [mode]
```

| Argument       | Default | Notes                                            |
|----------------|---------|--------------------------------------------------|
| `workdir`      | —       | Absolute path, must exist                        |
| `prompt`       | —       | Self-contained task description                  |
| `timeout-secs` | `180`   | Budget: work + 90s backoff + 30s context load    |
| `mode`         | `impl`  | `impl` (writes ok) or `plan` (read-only)         |

**Recommended timeouts:**
- Plan/explore only: `120`
- Simple change (1 file): `180`
- Medium change (2–3 files): `270`
- Hard ceiling: `300` — decompose instead

**Examples:**
```bash
# Explore only — safe, no writes
~/tools/gemini-delegate "/path/to/project" "Read app.py and describe the route structure" 120 plan

# Implement a single-file change
~/tools/gemini-delegate "/path/to/project" "Stack: Flask. File: app.py. TASK: ..." 180 impl

# Background run
~/tools/gemini-delegate "/path/to/project" "..." 240 impl > /tmp/gemini_out.txt 2>&1 &
# Monitor with: tail -f /tmp/gemini_out.txt
```

---

## Step 6 — Supervise in real time

The script prints live:
```
=== GEMINI START ===
Workdir : /path/to/project
Mode    : impl (yolo)
Timeout : 180s
Prompt  : Stack: Python/Flask. File: app.py ...
====================
  [init]   model=gemini-2.5-flash
  [read]   app.py
  [write]  app.py
  [gemini] Done. Converted date to datetime.date in fetch_data().
Tool calls: 3
Gemini tokens: 1,234  (900 in + 334 out, 0 cached)  |  ~$0.0003  (8.2s)
Claude Sonnet 4.6 eq: same tokens ~$0.0077  (ratio x25.7)
=== GEMINI DONE (exit: 0) ===
=== SYNTAX OK (1 file(s) checked) ===

=== UNCOMMITTED CHANGES ===
 app.py | 4 ++--
[log] → ~/.local/share/delegate-runs.jsonl  (1234 tokens, exit 0, 42.1s)
```

In `plan` mode, proposed writes are blocked and shown as `[plan-write]`:
```
  [read]        app.py
  [plan-write]  app.py   ← proposed but blocked
  [gemini]      Here is what I would change: ...
=== PLAN MODE — no files written ===
```

**Event types emitted by the parser:**

| Event        | Meaning                                  |
|--------------|------------------------------------------|
| `[init]`     | Session started, model name shown        |
| `[read]`     | File read by Gemini                      |
| `[write]`    | File written (impl mode)                 |
| `[plan-write]` | Write proposed but blocked (plan mode) |
| `[search]`   | Grep / search tool called                |
| `[shell]`    | Shell command executed                   |
| `[gemini]`   | Assistant text response                  |
| `[WARN]`     | Tool error detected                      |

**Red flags to act on immediately:**
- `[WARN]` → Gemini hit a tool error
- `exit: 1` or non-zero → Gemini failed or left verification incomplete
- No `[write]` after 120s → looping or task too vague
- `=== SYNTAX ERRORS ===` → **fix before committing**
- `=== GEMINI TIMEOUT ===` → check what was done before retrying
- Same file read 5+ times → Gemini is circling; run likely lost

**Common issues and workarounds:**

| Issue | Cause | Fix |
|-------|-------|-----|
| Gemini 503 on startup | High API demand (free tier) | Wait 30s, retry; add 90s to timeout |
| No tool calls, empty response | Model overloaded or prompt too long | Shorten prompt, retry |
| Timeout with no writes | Stuck in 503 backoff | Retry off-peak or increase timeout to 300 |
| File not modified despite "done" | Gemini described but didn't write | Add "make the edit now, do not describe it" |
| Context load takes 30s | Large system prompt on slow connection | Normal — budget for it |
| `[plan-write]` but no change | Expected in plan mode | Switch to `impl` mode to execute |

---

## Step 7 — Iteration

- **Max 3 attempts** per sub-task before escalating to the user.
- Between attempts, **read the git diff** to avoid doubling partial work.
- If Gemini did 50% and timed out: complete the rest manually rather than relaunching.
- If 503s are eating all attempts: pause and retry in 10+ minutes.

---

## Step 8 — Report to the user

```
✓ Gemini finished — <1-line summary>

Files modified:
  - path/to/file.ext (+X / -Y lines)

[If problem]:
⚠ <description> — completing manually / retrying?

Ready to commit?
```

---

## Orchestration rules

- **Decompose before delegating** — one giant prompt + high context overhead = timeout.
- **Use plan mode first** for any task touching >2 files — safer, free exploration.
- **Check diff between sub-tasks** — never launch the next step blind.
- **Don't code in Gemini's place** unless Gemini did ≥50% and timed out.
- **Timeout is the only turn limit** — set it conservatively; decompose rather than extending.
- **VERIFY with grep, not re-read** — `grep -n "def foo" file.py` is unambiguous.
- **Add 90s to every timeout** — 503 backoff is invisible and frequent.

---

## Token economics

Gemini's tool calls (file reads, writes) consume **Gemini tokens**, not Claude tokens.
Claude receives only the compressed final output (~200–800 tokens/run).

**Approximate pricing (Gemini 2.5 Flash):**
- ~$0.15/M input tokens, ~$0.60/M output tokens
- Claude Sonnet 4.6: ~$3/M input, ~$15/M output
- Typical cost ratio: **~20–30x cheaper per token than Claude**
- Note: context overhead (~900–10k tokens/run) makes per-run cost higher than token math suggests

**Free-tier caps (Google AI Studio, no billing):**
- ~60 requests/minute
- ~1,000 requests/day
- Model availability varies (503 = cap or demand spike)

Real token counts and cost are printed after every run and appended to the run log.

---

## Run Log

Every run appends one JSON entry to `~/.local/share/delegate-runs.jsonl`.

**Fields logged:**

| Field           | Type    | Description                                          |
|-----------------|---------|------------------------------------------------------|
| `ts`            | string  | ISO 8601 UTC timestamp                               |
| `delegate`      | string  | `"gemini"`                                           |
| `workdir`       | string  | Absolute project path                                |
| `project`       | string  | `basename(workdir)`                                  |
| `prompt_words`  | int     | Word count of the prompt (complexity proxy)          |
| `mode`          | string  | `"impl"` or `"plan"`                                 |
| `timeout_secs`  | int     | Configured timeout in seconds                        |
| `exit_code`     | int     | 0=success · 124=timeout · other=error                |
| `timed_out`     | bool    | `true` if `exit_code == 124`                         |
| `tool_calls`    | int     | Total tool invocations made by Gemini                |
| `files_changed` | int     | Files modified (git diff count)                      |
| `syntax_errors` | int     | Python/JS syntax errors detected post-run            |
| `duration_secs` | float   | Total wall-clock duration                            |
| `tokens_in`     | int     | Input tokens (from Gemini `result` event)            |
| `tokens_out`    | int     | Output tokens                                        |
| `tokens_cached` | int     | Cached tokens (reduces cost on repeated context)     |
| `tokens_total`  | int     | Total tokens                                         |
| `cost_usd`      | float   | Estimated cost in USD                                |
| `model`         | string  | Model name from Gemini `init` event                  |

**Useful queries:**
```bash
# All recent runs
cat ~/.local/share/delegate-runs.jsonl | python3 -m json.tool | less

# Success rate
jq -r '[.exit_code] | @tsv' ~/.local/share/delegate-runs.jsonl | sort | uniq -c

# Timed-out runs
jq 'select(.timed_out == true)' ~/.local/share/delegate-runs.jsonl

# Total cost
jq -r '.cost_usd' ~/.local/share/delegate-runs.jsonl \
  | awk '{sum+=$1} END {printf "Total: $%.4f\n", sum}'
```

---

## See Also

A sister delegate using Mistral Vibe exists: [vibe-skill](https://github.com/pcx-wave/vibe-skill).
Both write to the same `delegate-runs.jsonl` log, making runs comparable across delegates.
