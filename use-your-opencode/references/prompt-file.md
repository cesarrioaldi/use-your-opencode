# The prompt file

`ai/prompts/<name>.md` is the whole handoff. The executor starts blank — no
conversation, no memory of yesterday — so anything missing here is missing from
the diff, and you are the one who will verify the hole.

Write in the imperative. Short names, one task per file: `FIX1.md`, `FIX2.md`,
`M6a.md`, `menu-mobile-fix.md` — sortable, greppable, and referenced by the
report of the same name.

## Template

```markdown
# Task: <one sentence, imperative>

## Files you may change
ONLY these files: `src/a.ts`, `src/b.tsx`. Touch nothing else.

## Problem (measured, not guessed)
<Evidence with file:line and numbers.>
Measured: `<exact command>` → `<exact output: row / count / code>`.

## What to do

### 1. `src/a.ts`
- <concrete step>
- Copy the pattern at `src/c.ts:13-26`; do not invent a new one.

### 2. `src/b.tsx`
- <concrete step, including class names / API shape when they matter>

## What NOT to do
- <the traps this repo has already been bitten by>
- Do not run `pnpm db:seed` or `pnpm db:reset`.
- Do not change `.env`, config, `src/lib/*`; do not install packages.

## Before you finish
`<typecheck>` and `<lint>` clean.
Report the diff of every file you changed, verbatim.
```

## Why each block earns its place

- **Whitelist** — the executor runs with `--auto`, so the prompt file is the only
  fence. Name files explicitly. "Fix the fitting page" is an invitation to
  refactor the repo.
- **Measured problem** — makes the bug falsifiable and hands the executor a
  reproduction. The numbers you paste here become the before/after pair you use
  for verification, so measure *before* writing the prompt.
- **Per-file steps** — point at the repo's existing pattern by path. Without a
  pointer the executor invents a second convention, and conventions you did not
  choose are the bugs that come back.
- **Prohibitions** — repeat the project's hard rules inside every prompt file.
  The executor cannot read your conversation, and it *can* read the repo rules
  file only if you tell it to.
- **Done-gate** — asks for the checks you will rerun anyway, and for the diff
  verbatim so verification starts from the claimed change rather than from a
  summary of it.

## Filling it without bloating it

- Paths and line numbers beat pasted code. Paste a snippet only when the exact
  text matters (a helper to copy, a class string to match).
- State the *why* when the constraint is non-obvious ("stored paths need a
  leading slash or the detail page 500s"). Constraints with reasons survive; bare
  rules get "improved".
- Say what must stay the same: state shape, attribute names, aria roles, keys.
  Executors love to normalize things the tests and the owner depend on.
- If the task needs a decision you can make from the repo, make it in the prompt
  file. An open question in a prompt file becomes an invented answer in the diff.

## Feeding it

```bash
opencode run --auto "$(cat ai/prompts/FIX1.md)" 2>&1 | tee /tmp/FIX1.log
```

Reproducible runs pin the model and agent:

```bash
opencode run --auto -m provider/model --agent executor "$(cat ai/prompts/M6a.md)"
```

Long runs die. Split a twelve-part task into named parts and run them one at a
time, so each part lands on disk as a verified increment and a failure costs one
part, not the whole plan.

## After the run

1. `git diff --stat` — only whitelisted files changed?
2. Read the diff. The report is commentary; the diff is the work.
3. Run the gate yourself (`typecheck`, `lint`, the relevant test).
4. Measure the behavior yourself (`verification.md`).
5. Never rewrite a prompt file that has already been sent — write the next one.
   `ai/prompts/` is history, and history that gets edited is not evidence.
