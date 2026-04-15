# Claude Code Skills

A collection of Claude Code skills for streamlined development workflows.

## Skills

| Skill | Description |
|-------|-------------|
| **pr** | Create GitHub Draft PRs with auto-generated title, body, and branch management |
| **review** | Cross-review code with Codex MCP and Claude — mutual critique by default |
| **review-guide** | Generate human-friendly review guides with recommended reading order |

## Installation

Add this marketplace to your Claude Code:

```
/plugin marketplace add takuuum/claude-code-skills
```

Then install individual skills:

```
/plugin install pr@claude-code-skills
/plugin install review@claude-code-skills
/plugin install review-guide@claude-code-skills
```

## Skill Details

### /pr

Creates a GitHub Draft PR from the current branch. Automatically:
- Detects base branch and generates branch name if on main
- Pushes the branch to remote
- Generates PR title and body from commits and diff
- Always creates as Draft with `--assignee @me`

Options: `--reviewer`, `--label`, `--base`, `--title`

### /review

Runs a mutual cross-review between Claude and Codex MCP by default. Both reviewers critique each other's findings, producing a high-confidence consolidated report.

Options: `--diff`, `--pr <number>`, `--focus <area>`, `--no-cross-review`

### /review-guide

Generates a reading guide for code changes — categorizes changes, extracts test specifications, and recommends a file reading order optimized for human reviewers.

Options: `--pr <number>`, `--diff`, `--base <branch>`

## License

MIT
