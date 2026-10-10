# skills

Agent skills, installable with the [skills CLI](https://skills.sh/).

```bash
npx skills add <owner>/skills@use-your-opencode -g -a '*' -y
```

## `use-your-opencode`

Makes an orchestrator agent stop writing code itself: it writes a prompt file,
runs the `opencode` CLI on it, verifies the result with its own measurements
instead of trusting the executor's report, and records state on disk so work
survives context compaction.

The workflow it encodes:

1. **Map** the repo — rules file, status file, the files the change touches.
2. **Write a prompt file** to `ai/prompts/<name>.md`: file whitelist, measured
   problem, per-file steps, prohibitions, done-gate.
3. **Run** `opencode run --auto "$(cat ai/prompts/<name>.md)"`.
4. **Verify yourself** — DOM counts, `psql` rows, HTTP codes, exit codes. The
   report is a claim.
5. **Record before reporting** — `ai/STATUS.md` and
   `ai/laporan/LAPORAN-<name>.md`.

| file | contents |
|---|---|
| `SKILL.md` | the workflow, its rules, and how it fails |
| `references/prompt-file.md` | prompt-file template and rationale |
| `references/verification.md` | measurement recipes and the traps that cost debugging sessions |
| `references/memory-files.md` | `STATUS.md` / `LAPORAN` skeletons and update checklists |
| `references/bootstrap.md` | rules file, `ai/` layout, executor check, multi-agent install |
