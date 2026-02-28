# Compound Engineering Plugin setup (Codex only)

This repository environment has been configured with the Compound Engineering plugin for **Codex only**.

## What I ran

```bash
bunx @every-env/compound-plugin install compound-engineering --to codex
```

That command converts the `compound-engineering` Claude plugin into Codex-compatible prompts and skills.

## Where it installs for Codex

The converter writes to:

- `~/.codex/prompts`
- `~/.codex/skills`

In this environment, install output confirmed:

- `Installed compound-engineering to /root/.codex`

## How it works

The plugin provides a reusable compound workflow:

1. **Plan** (`/workflows:plan`) to produce implementation plans.
2. **Work** (`/workflows:work`) to execute with structured task flow.
3. **Review** (`/workflows:review`) to run multi-agent checks.
4. **Compound** (`/workflows:compound`) to record learnings for future tasks.

For Codex specifically, each converted Claude command is generated as:

- a **prompt** in `~/.codex/prompts`, and
- a corresponding **skill** in `~/.codex/skills`.

The prompt tells Codex to load and use the matching skill.

## Verify install quickly

```bash
find ~/.codex -maxdepth 2 -type d
```

You should see many `~/.codex/skills/*` directories plus `~/.codex/prompts`.
