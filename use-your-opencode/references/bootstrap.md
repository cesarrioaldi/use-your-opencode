# Bootstrap a project

Order: discover the stack → rules file → memory directory → `.gitignore` →
executor check → first smoke task → install the skill everywhere.

## 0. Discover, do not assume

Write down what the project actually uses, from files in the repo:

```bash
ls                                  # manifest, Makefile, compose file, CI config
cat package.json pyproject.toml go.mod Makefile 2>/dev/null | head -60
grep -rn "listen\|PORT\|localhost" --include=*.json --include=*.toml --include=*.yml -m3 . | head
```

You need four facts before writing anything: install command, run/dev command,
test + typecheck commands, and how data is stored and queried (datastore client,
file format, or HTTP API). Where the project already names a port or a host, use
that; never invent one. These facts go into the rules file, so no later session
has to rediscover them.

## 1. Rules file: `AGENTS.md`

`CLAUDE.md`, `GEMINI.md`, `.cursor/rules`, and any house name a harness invents
are per-tool names — a repo that uses one of them as its only source of truth is
readable by exactly one tool. Use `AGENTS.md` and point the others at it (shown
below). Keep it short, hard, and readable in one screen.

```markdown
# <project> — rules for coding agents

Stack: <language/framework, package manager, runtime>.
State, what is built, how to run: read <memory-dir>/STATUS.md.

## Commands
- install: `<...>`   run: `<...>`   test: `<...>`   typecheck/lint: `<...>`

## How to work
- All code is written through a CLI executor: write the task to
  `<memory-dir>/prompts/<name>.md`, then `<executor command> "$(cat <memory-dir>/prompts/<name>.md)"`.
  The orchestrator never writes code itself.
- Do not edit the executor's own configuration. If its provider/router is
  unreliable, shorten prompts and rerun; that is not a config problem.
- Do not trust the executor's report. Measure: rendered state, datastore rows,
  HTTP codes, exit codes. Screenshots are illustration, not evidence.
- Every finished task: update `<memory-dir>/STATUS.md` and write
  `<memory-dir>/reports/<name>.md`.
- Agent notes live in <memory-dir> (gitignored); the root holds project files.
- Scratch scripts: `<prefix>` and outside the typechecked/tested tree.

## Hard prohibitions
- Never run `<destructive command>` — <blast radius in one clause>.
- Never delete <the records that are not test data>.
- Never open, print, or commit secrets (`.env`, credentials, tokens).
- Do not change without being asked: <schema, migrations, seed data, shared
  libraries, auth/middleware, CI>.
- Never install <banned dependencies> — <reason>.

## Conventions
- <where mutations go (server actions / service layer / API handler)>
- <the single module that owns money/ids/dates/permissions>
- <how auth and sessions work, and what must not import what>
- Before reporting: `<check commands>` clean, plus the relevant test.
```

Fill in the *why* and the blast radius. "Don't run the seeder" is a rule; "the
seeder truncates every table and the owner's real records live in one of them"
is a rule that survives a tired afternoon.

### Point other tools at it

```bash
ln -s AGENTS.md CLAUDE.md      # Claude Code
ln -s AGENTS.md GEMINI.md      # Gemini CLI (or set context.fileName to AGENTS.md)
ln -s AGENTS.md .junie/guidelines.md
ln -s AGENTS.md <house-name>   # any harness with a name of its own
```

Prefer symlinks over copies: one file, every reader, no drift. Claude Code also
accepts a `CLAUDE.md` whose only content is `@AGENTS.md` — but note that its
presence changes the default: with a `CLAUDE.md` in the tree, Claude reads
`CLAUDE.md` and ignores `AGENTS.md`, so keep them consistent (import, symlink,
or set project instructions to read both).

### Check what each tool actually loads

The obvious test — put a codeword in `AGENTS.md` and ask the CLI for it — is
**not** a test of loading. Agents have file tools: verified 2026-10-11,
`opencode run` in a directory containing only `random-notes-xyz.md` globbed the
directory, read the file, and answered the codeword. A right answer can mean
"I searched and found it", which is exactly the case where a rule you placed
matters least.

What actually distinguishes the two:

```bash
# the tool's own accounting, when it has one
codex exec "List the instruction sources you loaded."
claude -p "List the instruction sources you loaded."
# or run with debug output and grep for the rules filenames
opencode run --print-logs --log-level DEBUG "say ok" | grep -iE 'agents|instruction'
```

If a tool cannot tell you, assume it may not have read the rules: the binding
constraints belong in the prompt file too, which is the one document the executor
is guaranteed to read. Repeat the check inside the real repo after adding a nested
rules file, because nested discovery differs per tool.

## 2. Memory directory

One gitignored directory, fixed names inside, chosen once:

```
<repo>/
  AGENTS.md                    rules for every agent (committed)
  <memory-dir>/
    STATUS.md                  state, numbers, traps, how to verify
    BRIEF.md                   product brief, if there is one
    prompts/<name>.md          every command sent to the executor
    reports/<name>.md          what each run actually did, with evidence
```

`ai/` is a fine default; `notes/`, `.agent/`, `docs/agent/` work just as well.
Rename freely — but pick one, use it in the rules file, and never keep two.

Default `.gitignore` lines (adjust to match the names you picked):

```
/<memory-dir>/
<scratch-prefix>*
```

Whether `AGENTS.md` itself is committed is a real choice: committed means every
teammate's agent inherits the rules; gitignored means the rules are yours alone
and nobody reviews them. Solo projects differ from shared repos — decide, and
say which you chose in the rules file header.

## 3. Check the executor before trusting it

```bash
command -v opencode claude codex gemini cursor-agent
<cli> --version
```

Then measure reliability instead of assuming it: send the same trivial prompt a
few times and count the failures. A free or shared model behind a local router
drops runs; that is a reason to write shorter prompts and rerun, never a reason
to edit the owner's config.

## 4. Smoke task first

One prompt file, the smallest real change, the whole loop: write → run → measure
→ `STATUS.md` → report. Whatever breaks here (a wrong command in the rules file,
a port that is not what the README said, the datastore client missing) is free to
discover before real work starts.

## 5. Install this skill for many CLI agents

```bash
npx skills add <owner>/<repo>@use-your-opencode -g -a '*' -y
npx skills add /path/to/skills@use-your-opencode -g -a '*' -y   # from a local checkout
```

`-a '*'` covers every detected agent; `-g` installs at user level so the workflow
follows you into every project. Drop `-g` for a project-scoped install. Update
with `npx skills update -g`.
