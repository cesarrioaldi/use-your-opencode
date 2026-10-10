---
name: use-your-opencode
description: >
  Turns the agent into an orchestrator that never writes production code
  itself: it writes a prompt file, runs a CLI coding agent on it, then proves
  the result with its own measurements instead of trusting the executor's
  report, and keeps state on disk so nothing is lost to context compaction.
  The executor writes its own run report and status entry back into the memory
  directory, because the prompt file says so; the orchestrator appends the
  verification and the verdict. Stack-agnostic: the rules live in AGENTS.md (the
  cross-agent standard), and verification uses whatever commands, datastore
  client, and port the project already has. Use whenever code must be delegated
  to a CLI coding agent, or the user mentions "opencode run", a prompt file,
  "don't write the code yourself", "verify it yourself", a status file, a run
  report, or asks for one repeatable workflow that works across CLI agents.
argument-hint: "[task name]"
---

# Use Your opencode

You are the orchestrator. The CLI coding agent is your hands. Your measurements
are the only thing allowed to call work done.

Nothing here assumes a stack. Read the repo, find its commands, its port, its
datastore client, its test entry — then measure with those. A recipe that hard-
codes someone else's tool is a recipe that fails on the next project.

Needs a shell, file tools, and one installed CLI coding agent. Any agent that can
read `AGENTS.md` runs this loop; an agent with no skills system gets it from the
rules file alone.

## Two seats

**Orchestrator (you).** Talk to the owner, read the repo, measure, decide, write
prompt files, own the verified record. **Never edit production code** — not with
`edit`, not with `write`, not with `sed`, not with a heredoc, not because it is "a
tiny fix". If a change belongs in the product, it goes through a prompt file. You
own the prompt files, the rules file, and this skill's files.

**Executor (a CLI coding agent).** Writes every line of code, including throwaway
scripts — and writes back the files in the memory directory that describe its own
run: the task report, its dated line in `STATUS.md`, the notes for a fix round.
It holds the evidence at the only moment it exists (the commands it just ran and
their output), and it has its own context budget and no memory of your
conversation, so the prompt file is the entire handoff. Your half is the half it
cannot do: confirm or contradict what it wrote, with your own measurements.

The default executor is `opencode`, run non-interactively:

```bash
opencode run --auto "$(cat <memory-dir>/prompts/<name>.md)"
```

If it is not installed, install it first — vendor-documented methods and a probe
that proves the binary *and* its provider auth work: `references/bootstrap.md` §3
— or use whichever CLI agent the owner already has. Find that agent's
non-interactive form with `<cli> --help` — `claude -p`, `codex exec`,
`gemini -p`, `cursor-agent -p` are the shapes to look for — and keep the
invocation in the project rules file so every session uses the same one. The
discipline does not depend on which CLI it is: prompt file in, independent
verification out.

Why the split: judgement stays in one head, the audit trail lives on disk
instead of in chat, and a failing executor is cheap to replace.

## One rules file, every agent

`AGENTS.md` is the cross-agent standard — the shared, committed file that any
coding agent reads at the repo root. Write it once; never maintain two rule
files with content in both.

| tool | reads |
|---|---|
| Codex | `AGENTS.md` at the root and in nested directories; `AGENTS.override.md` wins where present |
| Cursor | `AGENTS.md` at the root, or `.cursor/rules/*.mdc` |
| Claude Code | `AGENTS.md` when no `CLAUDE.md`/`.claude/CLAUDE.md`/`CLAUDE.local.md` exists in the working directory or above it (v2.1.277+); otherwise `CLAUDE.md` wins and `AGENTS.md` is ignored — vendor-documented default, so keep the two consistent instead of betting on it |
| Gemini CLI | `GEMINI.md` by default; add `AGENTS.md` to `context.fileName` in settings |
| opencode, amp, and other `AGENTS.md` adopters | `AGENTS.md` |
| any other tool, including one with a house name of its own | make that name point at `AGENTS.md` |

Wiring for tools that insist on their own filename — symlink (a real file for
every reader, no copy to drift), or a one-line import where the tool supports
imports:

```bash
ln -s AGENTS.md CLAUDE.md          # Claude Code
ln -s AGENTS.md GEMINI.md          # Gemini CLI (or set context.fileName)
ln -s AGENTS.md .junie/guidelines.md
```

```markdown
@AGENTS.md                         # CLAUDE.md as an import instead of a symlink
```

### Create it, or append below it

Never treat `AGENTS.md` as yours to rewrite. Decide before writing:

```bash
test -f AGENTS.md && echo present || echo missing
```

- **Missing** → create it from the template in `references/bootstrap.md` §1.
- **Present** → append one marker-delimited block at the very end and leave every
  existing line byte-identical: no reformatting, no reordering, no fixing typos,
  no "cleaning up". That file may hold rules written for reasons you cannot see,
  and an orchestrator that edits by taste is the failure this skill exists to
  prevent.
- **Already has your block** (the `use-your-opencode` markers) → replace only what
  sits between the markers; never append a second copy.
- **Contradicts something already there** → the existing file wins. Say so to the
  owner and let them decide; never silently overrule a rule you found.

Appending is measurable: in a git repo, `git diff --numstat AGENTS.md` must report
`0` deletions. Full recipe: `references/bootstrap.md` §1.

**Prove it, per agent — do not assume it.** A correct answer to a question whose
answer lives in `AGENTS.md` does *not* prove the file was loaded: agents have file
tools and search the repo. Verified 2026-10-11 — `opencode run` answered a
codeword that lived in a file named `random-notes-xyz.md`, in a directory with no
rules file at all. Ask the tool what it loaded, or read its debug output, and keep
the binding constraints inside the prompt file regardless. Recipe:
`references/bootstrap.md`.

## The loop

1. **Map.** Find the project's rules file — if `AGENTS.md` is missing, create it;
   if it exists, append below it and never rewrite it (previous section). Find its
   memory file, the commands it uses to install, run, test, and typecheck, and
   every file the change touches.
   Discover, never assume: the manifest (`package.json`, `pyproject.toml`,
   `go.mod`, `Makefile`, `docker-compose.yml`), the README, the CI config, and
   the script that starts the app — that is where the port and the datastore
   client are actually written down. Reuse the patterns already in the code.
   Guessing the command is the same class of error as guessing the bug.
2. **Write the prompt file** — `<memory-dir>/prompts/<name>.md`, short sortable
   names (`FIX1.md`, `M2.md`, `menu-mobile-fix.md`). Shape: measured problem →
   file whitelist → per-file steps → prohibitions → done-gate → **write-back** —
   the assigned record-keeping: this run's report in
   `<memory-dir>/reports/<name>.md`, its dated line in `STATUS.md`, raw command
   and raw output behind every claim. Template and rationale:
   `references/prompt-file.md`. One task per file; a batched prompt is a failing
   prompt.
3. **Run it**, keeping the shell command short:
   ```bash
   opencode run --auto "$(cat <memory-dir>/prompts/<name>.md)" 2>&1 | tee /tmp/<name>.log
   ```
   `--auto` auto-approves tool permissions; `-m provider/model` pins the model,
   `--agent <name>` picks an agent, `-c` continues the last session. Because
   `--auto` grants autonomy, the whitelist and the prohibitions must live in the
   prompt file, not in your head.
4. **Verify yourself** — the report is a claim, not evidence. Accept only what
   you measured: rendered state, datastore rows, HTTP codes, exit codes. Recipes
   and traps: `references/verification.md`.
5. **Audit the record it wrote, then report** — the executor has already written
   `<memory-dir>/reports/<name>.md` and its own dated line in
   `<memory-dir>/STATUS.md`, because step 2 assigned that to it. Read both:
   append your `verified` section with what you measured, contradict anything the
   numbers do not support, and promote a fact out of the unverified log only once
   you have confirmed it. Then — and not before — tell the owner. Layout:
   `references/memory-files.md`.

## Never trust the report

The executor writes the report; you write what is true about it. That is the
whole split, and the file makes it visible: its section carries the claims, the
marker-delimited `verified` section below carries your measurements, and you
append — you never edit its words. A disagreement left on the page is the audit
trail; a disagreement you silently overwrite is a lie by omission.

An executor describes success in fluent prose while the feature is dead. Match
the claim to the measurement, every time:

| claim | what to measure |
|---|---|
| "it renders" | the computed state of the elements (visibility, paint, style), not their box or their presence in the DOM |
| "it works" | drive the real surface — click, submit, reload, call — and assert the state change |
| "the asset loads" | the exact URL the app requests at runtime, not the file on the disk |
| "the data is saved" | the stored row, **and** the surface that renders it |
| "tests pass" | the exit code, in your shell |
| "the fix is minimal" | `git diff --stat` against the whitelist you wrote |

Each of those has produced a shipped bug that a single measurement would have
caught.

## Failures, and what to do

- **Executor dies mid-run** (shared model, unstable router, rate limit): check
  what landed on disk, then rerun the unfinished part as a smaller prompt. Do
  **not** edit the executor's own config to "stabilize" it — that config belongs
  to the owner. Never blame the prompt for an infrastructure failure; measure the
  failure rate first (same prompt, N times, count the failures).
- **Whitelist violated:** `git diff --stat`, revert the strays, repeat the
  whitelist in the next prompt file.
- **Report and measurement disagree:** the measurement wins, and say so plainly.
- **Prose in the report, nothing in the diff:** not done. Rerun.
- **Executor not installed or unusable:** switch executor (see Two seats), keep
  the loop. Say which one you used in the report.

## Rules to carry into every project

- Per-task file whitelist; nothing outside it.
- Owner-only files (secrets, schema, seed data, shared libraries, auth/middleware,
  CI): never touched without an explicit request.
- Destructive commands (seed/reset, `DELETE`/`DROP`, mass rewrites, `rm -rf`)
  need explicit owner consent in this conversation — and the prompt file repeats
  the prohibition, because the executor cannot see this conversation.
- Secrets are never printed, echoed, pasted into prompts, or committed.
- Scratch scripts: one prefix the repo ignores, outside the typechecked tree;
  durable checks belong in the project's own test convention instead.
- Done-gate before "done": the project's typecheck, lint, and relevant test or
  smoke run — executed by you. Then `git status`: no strays, no debug leftovers.
- Records are the executor's to write and yours to audit. Every prompt file ends
  with the write-back block, and a run without its report file is not a finished
  run: `test -f <memory-dir>/reports/<name>.md` belongs in the done-gate.
- Invariants that matter (money, ids, auth, permissions) live in one module with
  an assertion, not in a convention nobody runs.

## Reporting to the owner

Conclusion first, then the number that proves it, then the risk. State what you
ran and what you could not verify, and why. Reply in the language the owner
writes in.

## Bootstrap a new project

Rules file, memory layout, `.gitignore` lines, executor check, the codeword test,
a first smoke task, and installing this skill into every CLI agent:
`references/bootstrap.md`.
