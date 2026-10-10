# Memory on disk

Context gets compacted; files do not. Everything the next session needs must be
written down before it is needed.

## Layout

```
<repo>/
  AGENTS.md                     rules for every agent (committed, or gitignored — decide)
  <memory-dir>/                 one directory, gitignored, names fixed
    STATUS.md                   state, numbers, traps, how to verify, and the
                                task history you append at the bottom
    BRIEF.md                    product brief, if there is one
    prompts/<name>.md           every command sent to the executor
    reports/<name>.md           what this run claimed, and what you measured
```

`ai/` is the default name; `notes/`, `.agent/`, `docs/agent/` are equally fine.
Pick one, put it in the rules file's first lines, and never keep two.

## Who writes what

The executor keeps its own record, because at that moment it is the only one
holding the evidence — the commands it just ran and their output — and its
context is about to disappear. You keep the verdict and the state file, because
you are the one who measures afterwards. Neither writes the other's part.

| file | executor | orchestrator |
|---|---|---|
| `prompts/<name>.md` | never | writes it before the run; never re-edited after it is sent |
| `reports/<name>.md` | its own section: what changed, what it ran, raw output, what it did *not* verify | appends the `verified` section: the measurements that confirm or contradict |
| `STATUS.md` | **never opens it** | its only writer: the state, plus one entry per run that quotes the claim next to your measurement |
| `AGENTS.md`, this skill's files | never | the create-or-append discipline |

One writer per file is not tidiness, it is what makes each file checkable.
`STATUS.md` is the file every session reads first, and its whole value is that
everything in it was measured — a claim written into it by the party that did not
measure would put an unverified sentence inside the trusted file. Worse, it could
not be caught: the memory directory is gitignored, so unlike `AGENTS.md` there is
no `git diff` to show that a body was rewritten under a new line.

The prompt file is what makes the report half work: the write-back block is an
assignment, not a hope. An executor that did not write its report did not finish
the task — `test -f <memory-dir>/reports/<name>.md` is part of the done-gate, and
it is the one part of the gate the executor can satisfy alone.

`.gitignore`:

```
/<memory-dir>/
<scratch-prefix>*
```

Keeping the memory directory out of the repo is what makes it safe to write
freely: no reviewer pays for it, so nobody censors it — including the parts that
record a mistake.

## STATUS.md

Read first in every session. Written by one hand — yours.

It holds: a dated headline of the latest change, the commands that run the
project, what is built and what is not, the numbers that define "correct" (row
counts, expected status codes, expected element counts), a numbered list of traps
in symptom → cause → what-to-measure form, and how verification is done in this
project.

### Task history

One entry per run, appended at the bottom, shaped source → claim → measurement:

```markdown
## Task history

- 2026-10-11 FIX1 — cart no longer drops the last item · `reports/FIX1.md`
  - claim (unverified, quoted): `pnpm test cart` → 12 passed
  - ✓ verified 2026-10-11: `pnpm test cart` → 12 passed; `git diff --name-only`
    → `src/lib/cart.ts` only
- 2026-10-12 M2 — mobile menu opens on tap · `reports/M2.md`
  - claim (unverified, quoted): "menu opens on tap"
  - ✗ contradicted 2026-10-12: after tap,
    `document.querySelector('#menu').hidden` → `true` — present, still hidden
```

Three rules make the entry worth its lines:

1. **Quote the claim verbatim** out of the report and label it unverified. You do
   not paraphrase it into the trusted file and you do not launder it into your own
   voice: the reader can see which sentence was never measured.
2. **Confirmed** → the fact also goes where it belongs in the body (built /
   data reference / traps), and the entry gets `✓ verified <date>` with the
   command and the number that confirmed it.
3. **Contradicted** → the claim stays exactly as it was quoted, and the disproving
   number goes directly under it. Deleting it would erase the record that this
   executor's claim was wrong — which is exactly the record that saves the next
   session.

The entry's value is the pairing: what was claimed, and what was true. Cleaning up
an old entry destroys the only comparison you have.

If you want the claim on disk the moment the run ends, it is already there — in
`reports/<name>.md`, written by the executor. `STATUS.md` is the second step, and
it is yours.

### Update checklist

For every finished task, however small — you write all of it, the history entry
included:

1. Date in the title.
2. Counts in the data-reference section, if they moved.
3. Move the finished feature out of "not built yet".
4. Add the new trap the moment it is discovered — with the measurement, not the
   story, and not a theory you have not tested.
5. Point at this task's prompt file and report, and append the entry that quotes
   its claim next to your measurement.
6. Correct what later turned out to be wrong, and say that it was wrong. A status
   file that repeats a disproved theory is worse than no status file.

The rule that makes this worth the effort: **before reporting to the owner**. A
report spoken only into the conversation is lost at the next compaction; the file
is not.

## `reports/<name>.md`

One per executor run. Two sections, one writer each, marker-delimited so the
seam is visible in the raw file:

```markdown
# REPORT — <what was asked>

Executor: <cli agent, e.g. opencode 1.18.35> — model <provider/model>
Prompt: `<memory-dir>/prompts/<name>.md`
Files changed:
Date:

<!-- executor:<name> — written by the executor, never edited by anyone -->

## What was asked
<verbatim, in the owner's words>

## What I found before changing anything
<what was already true, checked place by place — not assumed>

## What I changed, per file
<per file: what and why>

## What I ran, with raw output
<the commands and their output, exactly as they came out: rows, counts, status
codes, exit codes, error text. Not a summary of them.>

## Not verified
<what this run could not prove, named plainly. This section being empty is
suspicious, not impressive.>

## Traps worth keeping

<!-- verified:<name> — appended by the orchestrator, after measuring -->

## Verification
<the commands, with their output — run by the orchestrator, never quoted from the
executor>

## Where the report was wrong
<claim → the measurement that killed it, plus the correction; or "no
contradiction found">
```

Rules that keep it worth reading:

- The executor creates the file if it is missing and writes only the first
  section. It never writes the `verified` section and it never opens `STATUS.md`
  — the prompt tells it both, and one writer per file is what makes the boundary
  checkable.
- You append the second section and never edit the first, not even to fix a typo
  or soften a wrong sentence. The unedited claim is what makes your measurement
  meaningful; editing it turns a disagreement into a single voice, and the value
  of this file is that it has two.
- A fix round reuses the same naming: prompt `<name>-fix1.md`, report
  `<name>-fix1.md`, and the new prompt quotes the failed measurement out of the
  previous report verbatim.
- Write it even when the task failed. A failed run with evidence stops the next
  session from repeating it — and "the executor said it was fine" is exactly the
  sentence this workflow exists to delete.
