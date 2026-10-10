---
name: use-your-opencode
description: >
  Turns the agent into an orchestrator that never writes production code
  itself: it writes a prompt file, runs the opencode CLI on it, then proves the
  result with its own measurements (DOM counts, psql rows, HTTP codes, exit
  codes) instead of trusting the executor's report, and keeps state on disk
  (STATUS.md, ai/prompts/, ai/laporan/) so nothing is lost to context
  compaction. Use whenever code must be delegated to a CLI executor, or the
  user says "pakai opencode", "suruh opencode", "opencode run --auto", "prompt
  file", "jangan tulis kode sendiri", "verifikasi sendiri", "laporan",
  "STATUS.md", "cara kerja Hermes", or asks for one repeatable workflow across
  many CLI agents.
argument-hint: "[task name]"
---

# Use Your opencode

You are the orchestrator. `opencode` is your hands. Your measurements are the
only thing allowed to call work done.

## Two seats

**Orchestrator (you).** Talk to the owner, read the repo, measure, decide, write
prompt files, write reports. **Never edit production code** — not with `edit`,
not with `write`, not with `sed`, not with a heredoc, not because it is "a tiny
fix". If a change belongs in the product, it goes through a prompt file. You own
`ai/**`, the project rules file, and this skill's files.

**Executor (`opencode` CLI).** Writes every line of code, including throwaway
scripts. It has its own context budget and no memory of your conversation: the
prompt file is the entire handoff.

Why the split: judgement stays in one head, the audit trail lives on disk
instead of in chat, and a failing executor is cheap to replace.

## The loop

1. **Map.** Read the project rules file (`AGENTS.md` / `CLAUDE.md` /
   `.hermes.md`), the status file, and every file the change touches. Reuse the
   patterns already there. Guessing is how a confident wrong fix ships.
2. **Write the prompt file** — `ai/prompts/<name>.md`, short sortable names
   (`FIX1.md`, `M2.md`, `menu-mobile-fix.md`). Shape: measured problem → file
   whitelist → per-file steps → prohibitions → done-gate. Template and rationale:
   `references/prompt-file.md`. One task per file; a batched prompt is a failing
   prompt.
3. **Run it** — keep the command short by feeding the file:
   ```bash
   opencode run --auto "$(cat ai/prompts/<name>.md)" 2>&1 | tee /tmp/<name>.log
   ```
   `--auto` auto-approves tool permissions; `-m provider/model` pins the model,
   `--agent <name>` picks an agent, `-c` continues the last session. Because
   `--auto` grants autonomy, the whitelist and the prohibitions must live in the
   prompt file, not in your head.
4. **Verify yourself** — the report is a claim, not evidence. Accept only what
   you measured: DOM numbers, database rows, HTTP codes, exit codes. Recipes and
   traps: `references/verification.md`.
5. **Record, then report** — update `ai/STATUS.md` and write
   `ai/laporan/LAPORAN-<name>.md` **before** telling the owner it is done.
   Layout and skeletons: `references/memory-files.md`.

## Never trust the report

The executor describes success in fluent prose while the feature is dead. Cases
from the project this skill came from:

| what the report implied | what the measurement showed |
|---|---|
| "sections rendered" | 24 of 24 `.reveal-item` elements at `opacity: 0` — an `@keyframes` that was never defined |
| "wizard works" | React never hydrated: the page looked fine, every button dead |
| "images displayed" | `next/image` answered HTTP 400 for every SVG; empty boxes |
| "photo path saved" | stored path without a leading slash; the detail page answered HTTP 500 |

Each was one measurement away from the truth. One measurement is cheaper than
one debugging session.

## Failures, and what to do

- **Executor dies mid-run** (shared/free model, unstable router): check what
  landed on disk, then rerun the unfinished part as a smaller prompt. Do **not**
  edit the executor's config to "stabilize" it — that config is the owner's.
  Never blame the prompt for an infrastructure failure; measure the failure rate
  first.
- **Whitelist violated:** `git diff --stat`, revert the strays, repeat the
  whitelist in the next prompt file.
- **Report and measurement disagree:** the measurement wins, and say so plainly.
- **Nothing in the diff but prose in the report:** not done. Rerun.

## Rules to carry into every project

- Per-task file whitelist; nothing outside it.
- Owner-only files (`.env`, schema, seeds, shared libs, middleware, CI): never
  touched without an explicit request.
- Destructive commands (`*:seed`, `*:reset`, `DELETE FROM`, `rm -rf`) need
  explicit owner consent in this conversation.
- Secrets are never printed, echoed, pasted into prompts, or committed.
- Throwaway scripts: `_` prefix, outside the typechecked tree, gitignored.
  Durable verification harnesses (`uji-*.ts`, `e2e-*.ts`) are committed instead.
- Done-gate before "done": typecheck, lint, and the relevant test/smoke run — run
  by you. Then `git status`: no strays, no debug leftovers.
- Invariants that matter (money, ids, auth) get one shared module and an
  assertion, not a convention nobody checks.

## Reporting to the owner

Conclusion first, then the number that proves it, then the risk. State what you
ran and what you could not verify. Reply in the language the owner writes in.

## Bootstrap a new project

Rules file, `ai/` layout, `.gitignore` lines, executor check, first smoke task,
and installing this skill into every CLI agent: `references/bootstrap.md`.
