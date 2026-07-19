---
name: "git-tag"
description: "Reads package.json version, synthesizes a change description from git history delta, creates an annotated git tag (no \"v\" prefix), and pushes it to origin."
license: "BSD 3-Clause"
compatibility: "Requires git CLI configured with remote access and a package.json with a \"version\" field in the working directory."
metadata:
  agent: coding
---

# Git Tag

You must execute ALL steps in order. Do not stop until the tag is pushed and verified on the remote. If any step fails, report the error and stop — do not continue with incomplete work.

## Step 1: Read the Version

Read `package.json` and extract the version string. Use `node` to reliably extract the top-level `version` field (avoids matching nested "version" keys):

```bash
TAG_VERSION=$(node -e "console.log(require('./package.json').version)")
echo "TAG_VERSION=$TAG_VERSION"
```

If `node` is unavailable, fall back to a more specific grep:
```bash
TAG_VERSION=$(grep -m1 '"version"[[:space:]]*:' package.json | sed 's/.*"version"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/')
```

## Step 2: Check Pre-conditions

Before proceeding, verify:

```bash
# Ensure we're on a clean working tree
git status --porcelain
```

If there are uncommitted changes, report them to the user and stop. Do not tag a dirty tree.

**Check for detached HEAD:** If `git branch --show-current` returns empty, you're in detached HEAD state. Create a temporary branch first:
```bash
if [ -z "$(git branch --show-current)" ]; then
  echo "WARNING: Detached HEAD state. Creating temporary branch for tagging."
  git checkout -b "tag-temp-$(date +%s)"
fi
```

## Step 3: Find the Previous Tag

Find the closest prior tag to use as the delta baseline:

```bash
# Try to find previous tag in same minor version series
# Note: sort -V requires GNU sort; fall back to sort if unavailable
PREV_TAG=$(git tag -l "${TAG_VERSION%.*}.*" 2>/dev/null | sort -V 2>/dev/null | grep -v "^${TAG_VERSION}$" | tail -1)
if [ -z "$PREV_TAG" ]; then
  PREV_TAG=$(git tag -l "${TAG_VERSION%.*}.*" 2>/dev/null | sort 2>/dev/null | grep -v "^${TAG_VERSION}$" | tail -1)
fi
if [ -z "$PREV_TAG" ]; then
  PREV_TAG=$(git describe --tags --abbrev=0 HEAD 2>/dev/null || echo "")
fi
echo "PREV_TAG=$PREV_TAG"
```

If no previous tag exists, set `PREV_TAG=""` (empty).

## Step 4: Check for Existing Tag

Ensure the tag does not already exist locally or on the remote:

```bash
LOCAL_EXISTS=$(git tag -l "${TAG_VERSION}")
REMOTE_EXISTS=$(git ls-remote --tags origin "${TAG_VERSION}" 2>/dev/null)
```

If either `LOCAL_EXISTS` or `REMOTE_EXISTS` is non-empty, the tag already exists. Report this to the user and stop. Do not overwrite.

## Step 5: Gather Commits for Description

Collect the commit messages between the previous tag and HEAD. **Filter out merge commits** (messages starting with "Merge ") as they add noise:

```bash
if [ -n "$PREV_TAG" ]; then
  git log --pretty=format:"%s" "${PREV_TAG}..HEAD" | grep -v "^Merge "
else
  git log --pretty=format:"%s" --max-count=20 | grep -v "^Merge "
fi
```

## Step 6: Synthesize the Description

Read the commit messages from Step 5 and write a concise, natural-language summary:

- **Group by topic** — Use conventional commit prefixes to group: `feat:` → features, `fix:` → fixes, `chore:` → chores, `docs:` → documentation, `refactor:` → refactors. If no conventional commit prefixes are present, group by keyword matching (e.g., "Dockerfile" → Docker, "config.yaml" → config).
- **2-4 sentences.** Present tense. Specific but not verbose.
- **No commit hashes or PR numbers.** Natural language only.
- **First line** is the release identifier (e.g., "Version 1.3.5 release.").
- **Remaining lines** describe what changed.

Example output format:
```
Version 1.3.5 release.

Includes a Dockerfile fix for the HOME environment variable, expanded scheduler documentation with atomicity directives, and a TUI audit fix proposal. Documentation updates cover config.yaml tracking, TUI command expansion, and scheduler documentation improvements.
```

If no commits can be analyzed, use: `"Version ${TAG_VERSION} release."`

## Step 7: Create the Tag

Create an annotated tag. **No "v" prefix.** Use the exact `TAG_VERSION` from Step 1:

```bash
git tag -a "<TAG_VERSION>" -m "<DESCRIPTION>"
```

Replace `<TAG_VERSION>` with the actual version and `<DESCRIPTION>` with the synthesized text from Step 6.

Verify the tag exists:

```bash
git tag -n1 | grep "^<TAG_VERSION>"
```

## Step 7: Create the Tag

Create an annotated tag. **No "v" prefix.** Use the exact `TAG_VERSION` from Step 1:

```bash
git tag -a "$TAG_VERSION" -m "$DESCRIPTION"
```

Verify the tag exists:

```bash
git tag -l "$TAG_VERSION"
```

If the output is empty, the tag creation failed. Report the error and stop.

## Step 8: Push the Tag

**Check that remote exists before pushing:**
```bash
git remote get-url origin
```
If this fails, the remote is not configured. Report the error and stop.

Push to the remote:

```bash
git push origin "$TAG_VERSION"
```

**Check exit code:** If the push returns a non-zero exit code, report the error and stop. Common failures:
- **Remote not configured:** Suggest adding a remote (`git remote add origin <url>`)
- **Permission denied:** Check SSH keys or token permissions
- **Network error:** Retry once, then report

## Step 9: Verify the Push

Confirm the tag is on the remote:

```bash
git ls-remote --tags origin | grep "$TAG_VERSION"
```

If this returns nothing, the push failed. Report the error and stop.

## Step 10: Report

Print a final summary using actual variable values:

```
Tagged: $TAG_VERSION
From: ${PREV_TAG:-initial}
Commits: $(git log --oneline "${PREV_TAG:-HEAD}..HEAD" | wc -l)
Pushed: origin/$TAG_VERSION

$DESCRIPTION
```

---
