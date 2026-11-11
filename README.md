# agent-rules

Operating rules for AI coding agents (Claude Code, and later other agents
such as GPT-based ones) running on the UniSR GPU cluster.

The rules tell an agent how to behave on the cluster: never compute on the
login node, run workloads through SLURM + Apptainer (or venv), where to find
Apptainer images and data, how to log experiments to MLflow, and when to
inform the user or wait for confirmation (e.g. data moves between storage
tiers, deletions in shared folders).

On the cluster this repo lives in `/mnt/data/unisr-data/agent-rules`.

## Contents

| File | Purpose |
|---|---|
| `CLAUDE.md` | Rules for Claude Code |

## Usage (Claude Code)

Add this line to your `~/.claude/CLAUDE.md` on the cluster:

```
@/mnt/data/unisr-data/agent-rules/CLAUDE.md
```

Claude Code will load the rules at the start of every session. Your own
preferences can go below that line.

## Usage (OpenCode)

To make the custom instructions work with OpenCode, edit your existing (or
create it) file at `~/.config/opencode/opencode.json` and add the
`instructions` entry:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": [
    "~/unisr-data/agent-rules/CLAUDE.md"
  ]
}
```

If the file already exists, keep your other settings and just merge in the
`instructions` entry.

## Usage (GitHub Copilot CLI)

Link the rules to Copilot's global instructions file by running:

```bash
mkdir -p ~/.copilot
ln -s /mnt/data/unisr-data/agent-rules/CLAUDE.md ~/.copilot/copilot-instructions.md
```

Since the main global file is now taken by the symlink, use Copilot's
supplementary instructions folder for your own rules. Copilot automatically
loads any Markdown files found inside the `instructions` subdirectory:

```bash
mkdir -p ~/.copilot/instructions
```

Add as many custom files as you need (e.g.
`~/.copilot/instructions/python-style.md`,
`~/.copilot/instructions/git-rules.md`).

Copilot reads the symlinked cluster rules first, then automatically appends
all files in this directory on every prompt.

## Contributing

The copy in `/mnt/data/unisr-data/agent-rules` is shared by every user:
do not edit it in place. Change the rules in this repo and update the
cluster copy with `git pull`.
