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

## Before you finish — the code, then the record
`<typecheck command>` and `<lint command>` clean. Report the diff of every file
you changed, verbatim.

Then write the record:

1. **`<memory-dir>/reports/<name>.md`** — create it (or append if a previous
   round exists) with the skeleton in `references/memory-files.md`. Every claim
   carries the raw command and its raw output. `not verified` is a valid entry; a
   green claim with nothing behind it is not.
2. Nothing else. Not `<memory-dir>/STATUS.md` — that file has one writer and it
   is not you. Not the orchestrator's `verified` section, not another task's
   report, not a prompt file.
```

## The write-back block

This is the block the orchestrator writes into every prompt, and it exists
because of two facts about the executor:

- **It holds the evidence at the only moment it exists.** The commands it ran,
  their exit codes, the row counts, the error text — a summary written afterwards
  from memory is a paraphrase, and paraphrase is where "it works" comes from.
- **Its context dies with the run.** Anything not on disk is gone; the next
  session, and you after a compaction, start from the files.

Assigning the report to the executor does **not** soften the verification rule.
It sharpens it: the executor's section is its claim, the `verified` section below
it is your measurement, and the two are never merged. When they disagree, the
file shows the disagreement instead of hiding it.

Verified 2026-10-11, `opencode` 1.18.35 on a two-file task: the executor created
its report with every section filled, with the raw commands and their raw output,
touched nothing outside the whitelist, and left the orchestrator's section alone.
The same test also ran the two-writer variant, and that is what settled the
design: the executor did append its line to `STATUS.md`, but nothing could prove
the body above it had survived — the memory directory is gitignored, so there is
no `git diff` to check. One writer per file.

Paste-ready, with the task's own paths and commands filled in:

````markdown
## Before you finish — the code, then the record

1. Write `<memory-dir>/reports/<name>.md`:

   ```
   # REPORT — <task>

   Executor: <cli agent, e.g. opencode 1.18.35> — model <provider/model>
   Prompt: `<memory-dir>/prompts/<name>.md`
   Files changed: <list>

   ## What was asked
   ## What I found first
   ## What I changed, per file
   ## What I ran, with raw output
   ## Not verified
   ## Traps worth keeping
   ```

2. Stop there. Do not open `<memory-dir>/STATUS.md` at all — it has one writer
   and it is not you. Do not touch the `verified` section, other reports, or the
   prompt file.
````

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
- **Write-back** — the executor records its own run while the evidence is still
  in front of it: the raw commands it ran, their raw output, and honestly what it
  did *not* verify. You audit the record instead of reconstructing it, and the
  executor's claims stay on the page next to your measurements.

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
5. Read the report the executor wrote, then append your `verified` section:
   point by point, the claim and the number that confirms or kills it. Do not
   edit its words — the unedited claim is what makes the file trustworthy.
6. `STATUS.md`: write the entry yourself — quote the report's claim verbatim,
   labelled unverified, then your measurement under it (confirmed, or the number
   that contradicts). The executor never writes this file, so the body above the
   entry is unchanged by the run and nothing unmeasured enters it.
7. Never rewrite a prompt file that has already been sent — write the next one.
   The prompt directory is history, and history that gets edited is not evidence.
8. A fix round is the same shape: `<memory-dir>/prompts/<name>-fix1.md` quotes
   the failed measurement from the previous report, and the executor appends
   `<memory-dir>/reports/<name>-fix1.md`. Failures get files too — a failed run
   with evidence stops the next session from repeating it.
