# The prompt file

`<memory-dir>/prompts/<name>.md` is the whole handoff. The executor starts blank
— no conversation, no memory of yesterday, and no guarantee it read the rules
file — so anything missing here is missing from the diff, and you are the one who
will verify the hole.

Write in the imperative. Short names, one task per file: `FIX1.md`, `FIX2.md`,
`M6a.md`, `menu-mobile-fix.md` — sortable, greppable, and matched by the report
of the same name.

## Template

```markdown
# Task: <one sentence, imperative>

## Files you may change
ONLY these files: `<path/a>`, `<path/b>`. Touch nothing else.

## Problem (measured, not guessed)
<Evidence with file:line and numbers.>
Measured: `<exact command>` → `<exact output: row / count / code>`.

## What to do

### 1. `<path/a>`
- <concrete step>
- Copy the pattern at `<path/c>:13-26`; do not invent a new one.

### 2. `<path/b>`
- <concrete step, including the exact names/shapes that matter>

## What NOT to do
- <the traps this project has already been bitten by>
- Do not run `<destructive command>`.
- Do not change <owner-only files> or secrets; do not install packages.

## Before you finish
`<typecheck command>` and `<lint command>` clean.
Report the diff of every file you changed, verbatim.
```

## Why each block earns its place

- **Whitelist** — the executor usually runs with auto-approved tool permissions,
  so this file is the only fence. Name files explicitly. "Fix the checkout page"
  is an invitation to refactor the repo.
- **Measured problem** — makes the bug falsifiable and hands the executor a
  reproduction. The numbers you paste here become the before/after pair for your
  own verification, so measure *before* writing the prompt.
- **Per-file steps** — point at the project's existing pattern by path. Without a
  pointer the executor invents a second convention, and conventions you did not
  choose are the bugs that come back.
- **Prohibitions** — repeat the project's hard rules inside every prompt file. It
  is the only document the executor is guaranteed to read: do not assume it
  loaded the rules file, and never assume it can see this conversation.
- **Done-gate** — asks for the checks you will rerun anyway, and for the diff
  verbatim so verification starts from the claimed change instead of a summary of
  it.

## Filling it without bloating it

- Paths and line numbers beat pasted code. Paste a snippet only when the exact
  text matters (a helper to copy, a class string to match).
- State the *why* when the constraint is non-obvious ("stored paths need a
  leading slash or the detail page 500s"). Constraints with reasons survive; bare
  rules get "improved".
- Say what must stay the same: state shape, attribute names, wire formats, keys,
  public signatures, routes. Executors normalize things the tests and the owner
  depend on.
- If a decision can be made from the repo, make it here. An open question in a
  prompt file becomes an invented answer in the diff.
- Keep it to one screen or two. A prompt nobody can read is a prompt nobody
  follows — split the task instead.

## Feeding it

```bash
opencode run --auto "$(cat <memory-dir>/prompts/FIX1.md)" 2>&1 | tee /tmp/FIX1.log
```

Any CLI executor takes the same shape — `claude -p "$(cat ...)"`,
`codex exec "$(cat ...)"`, `gemini -p "$(cat ...)"`. Reproducible runs pin the
model or agent:

```bash
opencode run --auto -m provider/model --agent executor "$(cat <memory-dir>/prompts/M6a.md)"
```

Long runs die. Split a twelve-part task into named parts and run them one at a
time, so each part lands on disk as a verified increment and a failure costs one
part, not the whole plan. Run one executor at a time: two concurrent runs of the
same CLI can fight over its local session store (verified: concurrent
`opencode run` fails with "database is locked").

## After the run

1. `git diff --stat` — only whitelisted files changed?
2. Read the diff. The report is commentary; the diff is the work.
3. Run the gate yourself (typecheck, lint, the relevant test).
4. Measure the behavior yourself (`verification.md`).
5. Never rewrite a prompt file that has already been sent — write the next one.
   The prompt directory is history, and history that gets edited is not evidence.
