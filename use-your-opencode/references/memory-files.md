# Memory on disk

Context gets compacted; files do not. Everything the next session needs must be
written down before it is needed.

## Layout

```
<repo>/
  AGENTS.md                     rules for every agent (committed, or gitignored — decide)
  <memory-dir>/                 one directory, gitignored, names fixed
    STATUS.md                   state, numbers, traps, how to verify
                                + the dated executor log below it
    BRIEF.md                    product brief, if there is one
    prompts/<name>.md           every command sent to the executor
    reports/<name>.md           what this run claimed, and what you measured
```

`ai/` is the default name; `notes/`, `.agent/`, `docs/agent/` are equally fine.
Pick one, put it in the rules file's first lines, and never keep two.

## Who writes what

The executor keeps its own record, because at that moment it is the only one
holding the evidence — the commands it just ran and their output — and its
context is about to disappear. You keep the verdict, because you are the one who
measures afterwards. Neither writes the other's part.

| file | executor | orchestrator |
|---|---|---|
| `prompts/<name>.md` | never | writes it before the run; never re-edited after it is sent |
| `reports/<name>.md` | its own section: what changed, what it ran, raw output, what it did *not* verify | appends the `verified` section: the measurements that confirm or contradict |
| `STATUS.md` | appends dated lines under `## Executor log (unverified)` | promotes confirmed facts into the body; never deletes a log line |
| `AGENTS.md`, this skill's files | never | the create-or-append discipline |

The prompt file is what makes this work: the write-back block is an assignment,
not a hope. An executor that did not write its report did not finish the task —
`test -f <memory-dir>/reports/<name>.md` is part of the done-gate, and it is the
one part of the gate the executor can satisfy alone.

`.gitignore`:

```
/<memory-dir>/
<scratch-prefix>*
```

Keeping the memory directory out of the repo is what makes it safe to write
freely: no reviewer pays for it, so nobody censors it — including the parts that
record a mistake.

## STATUS.md

Read first in every session. Both seats write it, in different places: the
executor appends to the log at the bottom, you edit the body.

It holds: a dated headline of the latest change, the commands that run the
project, what is built and what is not, the numbers that define "correct" (row
counts, expected status codes, expected element counts), a numbered list of traps
in symptom → cause → what-to-measure form, and how verification is done in this
project.

### The executor log

```markdown
## Executor log (unverified)

- 2026-10-11 FIX1 — cart total no longer drops the last item — `pnpm test cart`
  → 12 passed — reports/FIX1.md
```

One line per run, appended by the executor at the end of the file, never by you
in the middle of it. Every line is a claim, not a fact. Promotion is your job and
it is the only route into the body:

1. Measure it yourself.
2. **Confirmed** → write the fact where it belongs in the body (built /
   data reference / traps), then append `✓ verified <date>` to that log line.
3. **Contradicted** → leave the line untouched, put your correction directly
   under it with the number that disproves it, and fix the body. Deleting it
   would erase the record that this executor's claim was wrong — which is
   exactly the record that saves the next session.

The log's value is that it is one writer's unfiltered claim next to the other
writer's measurement. Cleaning it up destroys the only comparison you have.

### Update checklist

For every finished task, however small — the executor logs; you do the rest:

1. Date in the title.
2. Counts in the data-reference section, if they moved.
3. Move the finished feature out of "not built yet".
4. Add the new trap the moment it is discovered — with the measurement, not the
   story, and not a theory you have not tested.
5. Point at this task's prompt file and report.
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
  section. It never writes the `verified` section — the prompt tells it so.
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
