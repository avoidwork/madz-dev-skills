---
name: commit-push
description: Automates the complete git workflow: scans project rules for conventions, stages all changes, commits, pushes to the remote, and opens a Pull Request. Ensures strict compliance with documentation rules, including rule 5.4.
license: BSD 3-Clause
compatibility: Requires git CLI configured with remote access and PR creation tools (e.g., gh).
metadata:
  agent: coding
---

# Commit & Push

You are the final step in the craft. Precision matters. Follow these steps in order.

## 1. Read the Rules

Before touching a single file, scan the project root and any specified paths for a `.agent` rules file and read its contents.

- Identify the required commit message format.
- Identify the required PR title/body format.
- **Crucially:** Locate and strictly follow **Rule 5.4**. If Rule 5.4 is not found, log a WARN and proceed without it — do not block on missing documentation.

## 2. Create & Checkout a Branch

Check if already on a feature branch. If the current branch matches `feat/*`, `fix/*`, `docs/*`, or `chore/*`, skip branch creation and proceed to Step 3:

```bash
CURRENT_BRANCH=$(git branch --show-current)
if echo "$CURRENT_BRANCH" | grep -qE '^(feat|fix|docs|chore)/'; then
  echo "Already on feature branch: $CURRENT_BRANCH. Skipping branch creation."
else
  # Not on a feature branch — create one
  SYNTHESIZE_BRANCH=true
fi
```

If branch creation is needed, synthesize the branch name from the staged files. Run:

```bash
git diff --cached --name-only
```

**Branch name strategy:** Prefer meaningful file paths over the first changed file. Use this priority:
1. If any file matches a pattern like `src/<module>/*` or `tests/<module>/*`, use the module name as the slug.
2. If no module pattern matches, use the first changed file's basename (strip directories, extensions, special characters).
3. If no files are staged yet, use a generic slug from the commit type (e.g., `feat/update-config`).

Prepend the appropriate prefix based on the commit type (infer from the scanned project rules' commit message format, or default to `feat`):

- `feat/<slug>` for new features
- `fix/<slug>` for bug fixes
- `docs/<slug>` for documentation changes
- `chore/<slug>` for maintenance

Then create and check out the branch:

```bash
git checkout -b "<branch-name>"
```

If the changes are mixed, default to `feat/<slug>`.

## 3. Stage Everything

```bash
git add -A
```

Check the status to ensure nothing unexpected is included:

```bash
git status
```

If there are untracked files that shouldn't be committed, ask the user or exclude them via `.gitignore` before proceeding.

**Check for no changes:** If `git status --porcelain` returns nothing after staging, there are no changes to commit. Report this and stop — do not create an empty commit.

## 2.5. Check for Detached HEAD

Before creating a branch, verify git is not in a detached HEAD state:

```bash
git branch --show-current
```

If the output is empty (detached HEAD), create a branch from the current commit first:
```bash
git checkout -b "detached-fix-$(date +%s)"
```

## 4. Commit

Craft the commit message according to the scanned project rules.

```bash
git commit -m "<your-message-here>"
```

## 5. Push

Push the current branch to the remote.

```bash
git push origin HEAD
```

**Handle push failures:** If the push fails:
- **Remote not configured:** Report the error and suggest adding a remote (`git remote add origin <url>`).
- **Permission denied:** Report the error and suggest checking SSH keys or token permissions.
- **Merge conflicts:** If the remote has commits not present locally, suggest rebasing (`git rebase origin/main`) before pushing.
- **Other errors:** Report the error and stop. Do not proceed to PR creation.

## 6. Check for Existing PR

Before creating a new PR, check whether one already exists for this branch.

```bash
gh pr list --head "$(git branch --show-current)" --base main --state open --json url --jq '.[0].url'
```

- If a URL is returned, **do not create a new PR**. Report the existing PR URL to the user and skip to step 8.
- If no URL is returned, proceed to step 7.

## 7. Open a Pull Request

**Verify `gh` CLI is available:** Before creating a PR, ensure the `gh` CLI is installed and authenticated:
```bash
gh auth status 2>&1
```
If `gh` is not authenticated, suggest running `gh auth login` and stop.

Use the project's standard PR tool (usually `gh pr create`).

- Title: Follow the scanned project rules format.
- Body: Summarize the changes, reference any related issues, and ensure **Rule 5.4** is visibly addressed in the description.
- Assign the PR to the repository owner (Jason Mulligan / `avoidwork`).

**Finding related issues:** Search for issue references in commit messages (e.g., "fixes #123", "closes #456") and include them in the PR body. If no issue references are found, note "No related issues referenced."

```bash
gh pr create --title "<title>" --body "<body>" --assignee avoidwork
```

## 8. Verification

- Confirm the PR was created successfully (or note the existing PR URL from step 6).
- Print the PR URL for the user.