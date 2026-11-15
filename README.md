# go-train-skills

<img src="gopher-claude.png" alt="Gopher Express" width="300" />

## Installation

This repo is a Claude Code plugin and its own marketplace. Run these in a terminal (the VS Code extension doesn't support the `/plugin` command):

```sh
claude plugin marketplace add Shitcode-Swamp/go-train-skills
claude plugin install go-train-skills@go-train-skills
```

This installs it for your user, so it works in every repo. To install it only for one repo and share it with everyone working there, run the install from that repo with `--scope project` (saved to its `.claude/settings.json`).

In the Claude Code terminal app, the same works as `/plugin marketplace add ...` and `/plugin install ...`.

Plugins load when a session starts, so start a new Claude conversation (in VS Code: **Developer: Reload Window**). Type `/go-train-skills:` to check the skills are there.

### Updating

```sh
claude plugin marketplace update go-train-skills
claude plugin update go-train-skills@go-train-skills
```

Then start a new session.

### Trying local changes

```sh
claude --plugin-dir /path/to/go-train-skills
```

### Always use the development loop

Add this line to the `CLAUDE.md` of a repo:

```
For non-trivial Go tasks, follow the go-train-skills:dev-loop skill.
```

## About

Architecture tips and go-train tool skills for Claude.

Includes:

- Service architecture guidelines
- Coding agents guidelines (writing, review error surfacing, agentic loops)
- Go-Train framework usage guidelines

## Contents

- `agents/`: `planner`, `go-test-writer`, `coder`, `reviewer`, `attention-gatherer`
- `skills/dev-loop`: how the main chat runs the agents
- `skills/`: code style (`minimal-comments`, `go-naming`, `go-nesting-depth`, `go-oop`, `go-interfaces`) and architecture (`service-layers`, `go-train-di`, `go-train-controller`, `go-train-repository`)
