# Good Gemini Prompts

Patterns that reliably work with `gemini-delegate`.

---

## Template

```
Stack: <language/framework, key dependencies>
Key files: <file1> (<what it does>), <file2> (<what it does>)

TASK: <single imperative sentence — one thing only>

CONSTRAINTS:
- <what must not break>
- <format/interface contract if relevant>

VERIFY: grep for "<unique string that proves the change>" in <file> and confirm it exists.
```

---

## Single-file edit

```
Stack: Python/Flask. File: app.py

TASK: In fetch_data(), convert the date string (format "YYYY-MM-DD")
to datetime.date before returning, and convert id to str.

CONSTRAINTS:
- Do not change the function signature
- Return type must remain a dict

VERIFY: grep for "datetime.date" in app.py and confirm it exists.
```

---

## Add a route

```
Stack: Python/Flask. File: app.py

TASK: Add a GET /health route that returns {"status": "ok"} with HTTP 200.
Place it after the existing routes, before the if __name__ == "__main__" block.

CONSTRAINTS:
- Do not modify any existing routes
- Response must be JSON

VERIFY: grep for "/health" in app.py and confirm it exists.
```

---

## Read-only exploration (plan mode)

Run with `plan` as the mode argument — no files will be written.

```
Stack: Python/Flask. File: app.py

TASK: Read app.py and list all route definitions with their HTTP methods and paths.
Do not modify any files.

OUTPUT: Return a markdown table with columns: Method, Path, Function name.
```

---

## Fix a bug

```
Stack: Node.js/Express. File: middleware/auth.js

TASK: The verifyToken() function does not handle expired tokens — it throws
an unhandled exception instead of returning a 401 response. Fix it to catch
jwt.TokenExpiredError and return res.status(401).json({error: "Token expired"}).

CONSTRAINTS:
- Do not change the function signature
- All other error types should still throw

VERIFY: grep for "TokenExpiredError" in middleware/auth.js and confirm it exists.
```

---

## Tips

- Keep TASK under 3 sentences — longer = higher chance of partial completion
- Always include a VERIFY step — Gemini sometimes describes a change without making it
- Use `plan` mode first on unfamiliar codebases before committing to `impl`
- Name specific line ranges if you know them: "in the block starting at line 42"
