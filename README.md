# skills

Agent skills, installable with the [skills CLI](https://skills.sh/).

```bash
npx skills add <owner>/<repo>@use-your-opencode -g -a '*' -y
```

## `use-your-opencode`

Makes an orchestrator agent stop writing code itself: it writes a prompt file,
runs a CLI coding agent on it, verifies the result with its own measurements
instead of trusting the executor's report, and records state on disk so work
survives context compaction.

Stack-agnostic by design — no hardcoded datastore client, port, package manager,
or framework. The rules live in `AGENTS.md` (the cross-agent standard, with
symlinks for tools that insist on their own filename), and verification uses
whatever commands the project already has.

The loop it encodes:

1. **Map** the repo — rules file, memory file, the commands it runs, the files
   the change touches.
2. **Write a prompt file** to `<memory-dir>/prompts/<name>.md`: file whitelist,
   measured problem, per-file steps, prohibitions, done-gate.
3. **Run** `opencode run --auto "$(cat <memory-dir>/prompts/<name>.md)"` — or any
   other CLI executor the owner has.
4. **Verify yourself** — rendered state, datastore rows, HTTP codes, exit codes.
   The report is a claim.
5. **Record before reporting** — `<memory-dir>/STATUS.md` and
   `<memory-dir>/reports/<name>.md`.

| file | contents |
|---|---|
| `SKILL.md` | the workflow, its rules, and how it fails |
| `references/prompt-file.md` | prompt-file template and rationale |
| `references/verification.md` | how to find the project's own tools, plus the traps that cost debugging sessions |
| `references/memory-files.md` | `STATUS.md` / report skeletons and update checklists |
| `references/bootstrap.md` | `AGENTS.md`, per-tool wiring, memory layout, executor install + probe, multi-agent install |
