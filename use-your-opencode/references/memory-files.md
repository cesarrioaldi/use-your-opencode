# Memory on disk

Context gets compacted; files do not. Everything the next session needs must be
written down before it is needed.

## Layout

```
<repo>/
  AGENTS.md                     rules for every agent (committed, or gitignored — decide)
  <memory-dir>/                 one directory, gitignored, names fixed
    STATUS.md                   state, numbers, traps, how to verify
    BRIEF.md                    product brief, if there is one
    prompts/<name>.md           every command sent to the executor
    laporan/LAPORAN-<name>.md   what each run actually did, with evidence
```

`ai/` is the default name; `notes/`, `.agent/`, `docs/agent/` are equally fine.
Pick one, put it in the rules file's first lines, and never keep two.

`.gitignore`:

```
/<memory-dir>/
<scratch-prefix>*
```

Keeping the memory directory out of the repo is what makes it safe to write
freely: no reviewer pays for it, so nobody censors it — including the parts that
record a mistake.

## STATUS.md

Read first in every session. Written **before** reporting a finished task.

It holds: a dated headline of the latest change, the commands that run the
project, what is built and what is not, the numbers that define "correct" (row
counts, expected status codes, expected element counts), a numbered list of traps
in symptom → cause → what-to-measure form, and how verification is done in this
project.

Update checklist for every finished task, however small:

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

## LAPORAN-<name>.md

One per executor run, written by the orchestrator. Skeleton:

```markdown
# REPORT — <what was asked>

Date:
Command: `<memory-dir>/prompts/<name>.md`
Executor: <which CLI, which model/agent>
Files changed:

## The request
<verbatim, in the owner's words>

## Findings before the change
<what was already true, checked place by place — not assumed>

## What changed
<per file: what and why>

## Verification
<the commands, with their output: rows, counts, status codes, exit codes — run
by the orchestrator, never quoted from the executor>

## Other findings (not fixed)
<things noticed on the way, with paths, so the next session does not re-discover
them>

## Traps to add to STATUS.md
```

Write it even when the task failed. A failed run with evidence stops the next
session from repeating it — and "the executor said it was fine" is exactly the
sentence this workflow exists to delete.
