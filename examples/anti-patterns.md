# Gemini Anti-Patterns

What fails with `gemini-delegate` and why.

---

## Prompt too wide — multiple tasks

**Bad:**
```
Fix the login bug, add email validation to the signup form, and update the error messages to be friendlier.
```

**Why it fails:** Gemini will attempt all three, partially complete each, and the
result is a half-broken state that's hard to recover from.

**Fix:** One prompt per task. Run three separate `gemini-delegate` calls, checking
`git diff` between each.

---

## No VERIFY step

**Bad:**
```
Stack: Flask. File: app.py
TASK: Add rate limiting to the /login route.
```

**Why it fails:** Gemini may respond "Done, I added rate limiting" while having only
described what to do without actually writing the file. Without a VERIFY step there
is no check.

**Fix:** Always add `VERIFY: grep for "limiter.limit" in app.py and confirm it exists.`

---

## Timeout too short — ignores 503 backoff

**Bad:**
```bash
~/tools/gemini-delegate /path/to/project "TASK: ..." 60
```

**Why it fails:** The free-tier Gemini API frequently returns 503. The CLI retries
with backoff, which can silently consume 60–90 seconds before work begins. A 60s
timeout expires before Gemini even starts.

**Fix:** Minimum 120s for any task. Budget formula:
`timeout = real_work_estimate + 90s backoff + 30s context load`

---

## Giant prompt — context overhead

**Bad:**
```
Stack: Django, DRF, PostgreSQL, Redis, Celery
Key files: models.py (500 lines), serializers.py (300 lines), views.py (800 lines),
tasks.py (200 lines), signals.py (150 lines)

TASK: Refactor the entire authentication flow to use JWT tokens instead of
session cookies, update all related serializers, add token refresh endpoints,
update the Celery tasks that depend on user sessions, and add tests.
```

**Why it fails:** Gemini's context overhead is already ~10k tokens before the prompt.
This task will hit the timeout before completing a quarter of the changes.

**Fix:** Decompose.
```
Call 1 (plan, 120s): Read all 5 files and describe the current auth flow
Call 2 (impl, 180s): Update models.py only — add JWT token fields
Call 3 (impl, 180s): Update serializers.py only — add token serializers
Call 4 (impl, 180s): Update views.py only — add token endpoints
...
```

---

## Relaunching without reading the diff

**Bad:** Gemini times out. Immediately relaunch with the same prompt.

**Why it fails:** Gemini may have completed 60% of the task before timing out. A
blind relaunch will attempt the same changes on already-modified files, producing
duplicated code, merge conflicts, or overwritten partial work.

**Fix:** Always `git diff` before relaunching. If Gemini did ≥50%, complete the rest
manually rather than relaunching.

---

## Using `impl` mode for exploration

**Bad:**
```bash
~/tools/gemini-delegate /path/to/project "Read all files and tell me how auth works" 180 impl
```

**Why it matters:** In `impl` mode (`--yolo`), any write Gemini decides to make is
executed immediately with no confirmation. An "exploration" prompt can still trigger
writes if Gemini decides to "improve" something along the way.

**Fix:** Use `plan` mode for read-only tasks:
```bash
~/tools/gemini-delegate /path/to/project "Read all files and tell me how auth works" 120 plan
```

---

## Verify with file re-read instead of grep

**Bad:**
```
VERIFY: Read app.py and confirm the change is present.
```

**Why it fails:** If `app.py` is long, Gemini's re-read may not reach the modified
section within its context window. It will then hallucinate confirmation.

**Fix:**
```
VERIFY: grep for "def new_function" in app.py and confirm it exists.
```
A grep returns a single matching line — unambiguous, context-window-independent.
