# use-your-opencode

A skill that turns a coding agent into an **orchestrator**. It stops writing
production code itself: each task goes to a CLI coding agent through a prompt
file, the result is proven with the orchestrator's own measurements instead of
the executor's report, and state lives on disk so nothing is lost to context
compaction. The prompt file also assigns the record-keeping: the executor writes
its own run report, while everything that claims to be true — the status file and
the append-only verification — is written by the orchestrator alone, so a wrong
claim is contradicted on the page instead of overwritten. It is also a cost
decision: the executor does the typing on a cheap or free model, so the costly
session never pays for the edits.

Works with any agent that can read `AGENTS.md` — Claude Code, Codex, Cursor,
Gemini CLI, opencode, Copilot, Cline, Windsurf, Zed, and the rest of the
[skills](https://skills.sh/) ecosystem. Stack-agnostic: no hardcoded datastore
client, port, framework, or package manager.

## Why: the typing is the part a cheap model can do

Most of what an agent burns in a coding session is not judgement, it is mechanical
work — reading files back, retyping them, running the edit → run → edit loop. That
is where the tokens go, and it is exactly the part you can hand to a model that
costs nothing.

So the split is also a budget. You — the model in the session the owner actually
talks to — keep the context, the plan, the prompt file and the measurement. The
executor burns the bulk tokens on the edits. Point it at a free or cheap endpoint
and the expensive session stops paying for the typing:

```bash
opencode models                        # what is reachable from this machine
opencode run --auto -m <provider>/<model> "$(cat <memory-dir>/prompts/FIX1.md)"
opencode stats                         # tokens and cost, from the executor's own store
```

opencode reaches free models (`opencode/*-free`), anything you already pay for, or
a local router, through `opencode auth login` (`opencode providers`). Measured on
the machine this was written on, `opencode 1.18.35`: `opencode models` lists ten
`opencode/*-free` models plus `9router/freemodel`, and `opencode stats` reports 228
sessions over 54 days at **$0.00** total cost — 1.0M average tokens per session.
The workflow's own test runs (a two-file Node task, `freemodel` behind a local
router) took 29–69 s each, and every run produced a report good enough to verify
against.

Two rules keep the saving honest:

- **Cheap runs fail more often.** Free and shared endpoints drop, rate-limit, and
  stall. Keep prompts short, one task per file, rerun the unfinished part as a
  smaller prompt — and if failures persist, measure the rate (same prompt, N times,
  count the failures) instead of blaming the prompt. Never "stabilize" it by
  editing the executor's own provider config; that belongs to the owner.
- **The bar does not move.** A free executor does not make its report evidence.
  Exit codes, stored rows, rendered state, `git diff --stat` — all measured by you.
  A cheap executor costs reruns, not correctness, but only for as long as you
  actually measure.

Three runs, zero cost, and the costly session never typed a line of product code.

## Install

These commands use this repository's path. A fork or a rename substitutes the same
way — swap in your own `owner/repo`.

```bash
npx skills add cesarrioaldi/use-your-opencode
```

Project scope by default: the skill lands in `.agents/skills/` for the current
project, alongside a `skills-lock.json`. In a terminal the CLI may ask which
agents and scope you want before it installs; `-y` answers everything and is what
agents and scripts must use. Add `-g` to install at user level instead.

| flag | effect |
|---|---|
| `-g` | user-level install, so the workflow follows you into every project |
| `-a '*'` | install into every agent the CLI detects, without asking |
| `-s use-your-opencode` | install this skill only, if the repo ever holds more than one |
| `--copy` | copy the files instead of symlinking them |
| `-y` | no prompts — for scripts and for agents |

Other forms, all checked against the `skills` CLI:

```bash
npx skills add https://github.com/cesarrioaldi/use-your-opencode                # full URL
npx skills add cesarrioaldi/use-your-opencode -g -y                             # user level, unattended
npx skills add cesarrioaldi/use-your-opencode -a opencode -y                    # one agent, project scope
npx skills add cesarrioaldi/use-your-opencode -l                                # list skills, install nothing
npx skills use cesarrioaldi/use-your-opencode@use-your-opencode                 # print SKILL.md, install nothing
# npx skills add cesarrioaldi/use-your-opencode@use-your-opencode               # ✗ not supported — use -s
```

### Manual install, no CLI

```bash
git clone https://github.com/cesarrioaldi/use-your-opencode /tmp/skills
mkdir -p ~/.agents/skills
cp -R /tmp/skills/use-your-opencode ~/.agents/skills/           # user level
# or: cp -R /tmp/skills/use-your-opencode .agents/skills/       # project level
```

A tarball works the same way:
`curl -fsSL https://github.com/cesarrioaldi/use-your-opencode/archive/refs/heads/main.tar.gz | tar xz -C /tmp`.

### Verify, update, remove

```bash
npx skills list                              # use-your-opencode should appear
test -f ~/.agents/skills/use-your-opencode/SKILL.md && echo installed
npx skills update -g                         # update
npx skills remove use-your-opencode          # uninstall
```

## Requirements

- A shell and file tools — the workflow is prompt file → command → measurement.
- One CLI coding agent. The default is `opencode`; if it is not installed:

  ```bash
  curl -fsSL https://opencode.ai/install | bash   # or: npm install -g opencode-ai
  opencode --version
  opencode run --auto "Reply with exactly: ok"    # probe: binary, PATH, provider auth
  ```

  Any other agent with a non-interactive mode works: `claude -p`, `codex exec`,
  `gemini -p`, `cursor-agent -p`. Put the one you use in the project rules file.

## What it does

1. **Map** the repo — rules file, memory file, install/run/test commands, and
   every file the change touches. Discover, never guess.
2. **Write a prompt file** — `<memory-dir>/prompts/<name>.md`: measured problem,
   file whitelist, per-file steps, prohibitions, done-gate, and the write-back
   block that assigns the executor its run report.
3. **Run** `opencode run --auto "$(cat <memory-dir>/prompts/<name>.md)"` — or any
   other CLI executor the owner has.
4. **Verify yourself** — rendered state, datastore rows, HTTP codes, exit codes.
   The report is a claim, never evidence.
5. **Audit the record, then write the state** — the executor wrote
   `<memory-dir>/reports/<name>.md` because the prompt asked it to; you append the
   `verified` section, then write the `STATUS.md` entry yourself, quoting its claim
   verbatim next to your measurement.

Two writers, one file each: the executor records what it just ran while the
evidence is in front of it, and you record whether that is true. `STATUS.md` — the
file every session reads first, and a gitignored one, so nothing would catch a
rewrite — is yours alone, and a disagreement stays visible in the report instead of
being merged into one voice.

| file | contents |
|---|---|
| `SKILL.md` | the workflow, its rules, and how it fails |
| `references/prompt-file.md` | prompt-file template and rationale |
| `references/verification.md` | how to find the project's own tools, plus the traps that cost debugging sessions |
| `references/memory-files.md` | `STATUS.md` / report skeletons and update checklists |
| `references/bootstrap.md` | `AGENTS.md`, per-tool wiring, memory layout, executor install + probe, multi-agent install |

## Layout of this repo

```
use-your-opencode/
  SKILL.md
  references/
    bootstrap.md
    memory-files.md
    prompt-file.md
    verification.md
README.md
```

## Using it

Say the task normally — "delegate this to opencode", "write a prompt file for
this", "don't write the code yourself", "verify it yourself" — or invoke the
skill directly:

```
/use-your-opencode <task name>
```

`AGENTS.md` is created if the project has none. If one already exists, the skill
appends a marker-delimited block **below** the existing content and changes
nothing above it — never a rewrite, never a deletion. That is measurable: in a
git repo, `git diff --numstat AGENTS.md` must report `0` deletions.

## For AI agents

If you are an agent asked to install this skill, do exactly this and nothing else:

1. Install it, non-interactively:

   ```bash
   npx skills add cesarrioaldi/use-your-opencode -g -a '*' -y      # swap in your fork's owner/repo if you forked this
   ```

   Those flags are not optional for you: a bare `npx skills add` waits at a prompt
   when a terminal is attached. If `-a '*'` is rejected by an older CLI, install
   per agent or copy the folder to `~/.agents/skills/use-your-opencode/` (manual
   install above).

2. Verify, then load it:

   ```bash
   npx skills list | grep use-your-opencode
   ```

   Read `use-your-opencode/SKILL.md` (runtime: `skill://use-your-opencode`) and
   follow it as your instructions. Its references are loaded on demand.

3. Before touching the target repo's `AGENTS.md`: create it if missing, otherwise
   append your block below the existing content. Never delete, truncate, rename,
   re-sort, or reformat it, and never remove a rule you found there — if it
   contradicts you, the file wins and you report the conflict to the owner.

4. Never edit the executor's own configuration to work around a failure.

5. Never report success without a measurement of your own. A fluent executor
   report is a claim, not evidence.

6. If installation is impossible (no network, no `npx`), read
   `use-your-opencode/SKILL.md` from this repo and follow it directly — the skill
   is plain markdown, and the rules-file template in `references/bootstrap.md`
   carries the workflow on its own.
