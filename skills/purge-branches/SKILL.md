---
name: "purge-branches"
description: "Runs npm run purge-branches to delete local branches other than main. Use when the user wants to clean up stale branches without switching branches or pulling."
license: "BSD 3-Clause"
compatibility: "Requires git CLI configured with npm. Must be run from the project root directory."
metadata:
  agent: "coding"
---

# Purge Branches

Delete all local branches except `main`.

## Pre-flight Check

Before running, verify the npm script exists and check current state:

```bash
# Verify the script exists in package.json
grep '"purge-branches"' package.json
if [ $? -ne 0 ]; then
  echo "ERROR: purge-branches script not found in package.json"
  exit 1
fi

# Check current branch — ensure we're not on a branch that will be deleted
CURRENT_BRANCH=$(git branch --show-current)
if [ "$CURRENT_BRANCH" != "main" ]; then
  echo "WARNING: You are on branch '$CURRENT_BRANCH'. This branch will NOT be deleted (only non-main branches are purged)."
fi

# Dry-run: show what would be deleted
echo "Branches that will be deleted:"
git branch --format='%(refname:short)' | grep -v '^main$'
```

## Execute

Run the purge script:

```bash
npm run purge-branches
```

**Handle failure:** If `npm run purge-branches` returns a non-zero exit code, report the error and stop. Do not proceed.

## Report

Count deleted branches by comparing before/after:

```bash
# Count remaining branches (excluding main)
REMAINING=$(git branch --format='%(refname:short)' | grep -cv '^main$')
echo "Purge complete."
echo "Branches deleted: $((2 - REMAINING))"  # Adjust based on actual before count
echo "Remaining branches: $REMAINING"
```

If the npm script already handles reporting, use its output instead.

---
