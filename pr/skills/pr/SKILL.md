---
description: Create a GitHub pull request for the current branch. Invoke when the user wants to open, create, or submit a PR.
---

Create a GitHub pull request for the current branch. Follow these steps carefully:

## Rules
- **Always write the PR title and body in English**
- **Always create as a Draft PR** (`--draft` flag is mandatory)
- **Always assign the PR to yourself** using `--assignee @me`

## Arguments
Optional arguments passed via `$ARGUMENTS`:
- `--reviewer <username>` or `-r <username>`: Assign a reviewer
- `--label <label>` or `-l <label>`: Add a label
- `--base <branch>`: Override the base branch (default: main or master)
- `--title <title>`: Override the auto-generated title

Parse `$ARGUMENTS` to extract these options before proceeding.

## Step 1: Gather information

Run these commands in parallel:
- `git status` — check for uncommitted/unstaged changes
- `git branch --show-current` — get current branch name
- `git log --oneline $(git merge-base HEAD $(git remote show origin | grep 'HEAD branch' | awk '{print $NF}'))..HEAD` — list commits on this branch
- `git remote show origin | grep 'HEAD branch'` — determine default base branch (main/master)

### Handle unstaged/untracked files

If `git status` shows **unstaged changes or untracked files**, automatically stage them:
```bash
git add -A
```

### Handle staged but uncommitted changes

If there are **staged but uncommitted changes** (after the above add), warn the user and ask if they want to commit first or continue creating the PR without committing.

## Step 2: Check branch and confirm working branch

### If on the default base branch (main or master):

1. **Automatically generate a branch name** based on staged/uncommitted changes or recent commits (e.g., `feat/add-login`, `fix/null-pointer`). Use kebab-case and conventional prefix (feat/, fix/, chore/, etc.).
2. **Create and switch to the new branch** without asking:
   ```bash
   git checkout -b <branch-name>
   ```
3. Continue with the rest of the steps using this new branch.

### If on a working branch (not main or master):

Proceed automatically without asking.

## Step 3: Push the branch

Check if the current branch has a remote tracking branch:
```
git status -sb
```

If the branch is not yet pushed to remote (no upstream), push it:
```
git push -u origin <current-branch>
```

If already pushed but has unpushed commits, push:
```
git push
```

## Step 4: Analyze changes

Run these in parallel to understand what changed:
- `git log --oneline <base-branch>..HEAD` — commit history
- `git diff <base-branch>...HEAD --stat` — changed files summary
- `git diff <base-branch>...HEAD` — full diff (for understanding context)

## Step 5: Generate PR content

Based on the commits and diff, generate:

**Title**: A concise title (under 70 characters) summarizing the changes. Use conventional commit style if the commits follow it (e.g., "feat: add user authentication", "fix: resolve race condition in queue").

**Body** using this template:

```
## Summary
<2-4 bullet points describing what this PR does and why>

## Context
<1-2 sentences explaining the background/motivation, or any relevant issue references like "Closes #123">

## Changes
<Bullet list of key technical changes>

## Test plan
- [ ] <specific test steps or verification checklist>
- [ ] <add more items as needed>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Step 6: Create the PR

Build the `gh pr create` command with:
- `--title` from generated title (in English)
- `--body` from generated body (in English, use HEREDOC to preserve formatting)
- `--base` from determined base branch (or `--base` argument if provided)
- `--draft` — **always include this flag**
- `--assignee @me` — **always include this to assign yourself**
- `--reviewer` if `--reviewer` argument was given
- `--label` if `--label` argument was given

Example:
```bash
gh pr create --draft --assignee @me --title "feat: add feature" --base main --body "$(cat <<'EOF'
## Summary
- ...

## Context
...

## Changes
- ...

## Test plan
- [ ] ...

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

## Step 7: Return the PR URL

After successful creation, display the PR URL to the user.
