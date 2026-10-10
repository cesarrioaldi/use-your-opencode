# Bootstrap a project

Order: rules file → `ai/` memory → `.gitignore` → executor check → smoke task.

## 1. Rules file

`.hermes.md`, `AGENTS.md`, or `CLAUDE.md` — whatever the orchestrator reads
first. Keep it short, hard, and readable in one screen.

```markdown
# Project rules — <name>

Stack: <framework, package manager, runtime, database>.
State, how to run, what is done: read `ai/STATUS.md`.

## How to work
- All code is written through the opencode CLI: write the command to
  `ai/prompts/<name>.md`, then `opencode run --auto "$(cat ai/prompts/<name>.md)"`.
  The orchestrator session never writes code itself.
- Do not edit the executor's config (`~/.config/opencode/*`). If its router
  fails, that is the router's problem, not a reason to edit config.
- Do not trust the executor's report. Measure: DOM numbers, `psql` queries,
  HTTP codes. Do not rely on screenshots.
- Every finished task: update `ai/STATUS.md`, write `ai/laporan/LAPORAN-<name>.md`.
- Agent notes live in `ai/` (gitignored); the root holds project files only.
- One-shot scripts: `scripts/_*.ts` (gitignored), outside the typechecked tree.

## Hard prohibitions
- Never run `<seed/reset command>` — <blast radius, in one clause>.
- Never delete <records that are not test data>.
- Never open, print, or commit `.env`.
- Do not change without being asked: <schema, seeds, shared libs, middleware, CI>.
- Never install <banned dependencies> — <reason>.

## Code conventions
- <where mutations go: server actions, not API routes>
- <money/ids/dates: the single module that owns them>
- <auth: cookie name, what Edge runtime may not import>
- Before reporting: `<typecheck>` and `<lint>` clean, plus the relevant test.
```

Fill in the *why* and the blast radius. "Don't seed" is a rule; "the seed deletes
every table and the owner's real orders live in one of them" is a rule that
survives a tired afternoon.

## 2. `ai/` memory

```bash
mkdir -p ai/prompts ai/laporan
printf '/ai/\n/.hermes.md\nscripts/_*.ts\n' >> .gitignore
```

Then write `ai/STATUS.md` from the skeleton in `memory-files.md`: what is built,
what is not, and the numbers that define correct.

## 3. Check the executor before trusting it

```bash
command -v opencode && opencode --version
opencode models | head
```

Measure reliability instead of assuming it: send the same trivial prompt a few
times and count the failures. A free or shared model behind a local router drops
runs — that is a reason to write shorter prompts and rerun, never a reason to
edit the owner's config.

## 4. Smoke task first

One prompt file, the smallest real change, the whole loop: write → run → measure
→ `STATUS.md` → report. Whatever breaks here (missing hydration, a wrong test
command, a path the executor cannot resolve) is free to discover before real
work starts.

## 5. Install this skill for many CLI agents

```bash
npx skills add <owner>/<repo>@use-your-opencode -g -a '*' -y
```

`-a '*'` installs into every detected agent (claude-code, codex, opencode,
cursor, gemini-cli, and the rest of the family); `-g` puts it at user level so
the workflow follows you into every project. Later:

```bash
npx skills update -g
```

Working from a local checkout instead:

```bash
npx skills add /path/to/skills@use-your-opencode -g -a '*' -y
```

Project-scoped install (only this repo, kept out of everyone else's context):
drop `-g`.
