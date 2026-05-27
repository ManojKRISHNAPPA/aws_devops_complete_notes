# Git for DevOps Engineers — Complete Guide

## Table of Contents
1. [Git Fundamentals](#1-git-fundamentals)
2. [Git Architecture](#2-git-architecture)
3. [Core Commands](#3-core-commands)
4. [Branching and Merging](#4-branching-and-merging)
5. [Remote Repositories](#5-remote-repositories)
6. [Advanced Git](#6-advanced-git)
7. [Git Workflows](#7-git-workflows)
8. [Git Hooks](#8-git-hooks)
9. [Git in CI/CD](#9-git-in-cicd)
10. [GitHub/GitLab Operations](#10-githubgitlab-operations)
11. [Git Troubleshooting](#11-git-troubleshooting)
12. [Common Interview Questions](#12-common-interview-questions)

---

## 1. Git Fundamentals

**Git** is a distributed version control system. Every developer has a complete copy of the entire history.

```
Centralized VCS (SVN)          Distributed VCS (Git)
──────────────────────────     ──────────────────────────────
     Central Server                Central Server (bare repo)
          │                              │
    ┌─────┼─────┐               ┌───────┼───────┐
    │     │     │               │       │       │
  Dev1  Dev2  Dev3           Dev1    Dev2    Dev3
(checkout)(checkout)       (full repo)(full repo)(full repo)

  • Single point of failure    • Work offline
  • Slow — every op hits server • Full history everywhere
  • No offline work            • Fast local operations
```

**Three States of Files:**
```
Working Directory    Staging Area (Index)    Repository (.git)
─────────────────    ────────────────────    ──────────────────
Modified files       git add → staged        git commit → stored
(untracked/modified) changes                 as commit objects
      │                    │                        │
      └─── git add ───────►│                        │
                           └─── git commit ────────►│
      ◄─── git checkout ───────────────────────────┘
      ◄─── git restore ────────────────────────────┘
```

---

## 2. Git Architecture

```
Git Object Model:
──────────────────────────────────────────────────────
BLOB     → Content of a file (no filename, just content)
TREE     → Directory listing (filenames + references to blobs/trees)
COMMIT   → Snapshot pointer (tree + parent + author + message)
TAG      → Named reference to a commit

Commit History:
  A ──► B ──► C ──► D (main)
                     │
                     └──► E ──► F (feature-branch)

Each commit stores:
  • SHA-1 hash (40 hex chars, unique ID)
  • Pointer to tree (snapshot)
  • Pointer to parent commit(s)
  • Author + timestamp
  • Commit message

Git References:
  HEAD         → Current commit/branch
  main         → Pointer to latest commit on main
  origin/main  → Remote tracking branch
  ORIG_HEAD    → Previous HEAD (before merge/rebase)
```

---

## 3. Core Commands

### Initial Setup

```bash
# Identity (required before first commit)
git config --global user.name "Your Name"
git config --global user.email "you@company.com"
git config --global core.editor "vim"
git config --global init.defaultBranch "main"
git config --global pull.rebase false    # merge on pull
git config --list                        # Show all settings

# Initialize repository
git init my-project           # Create new repo
git init .                    # Initialize current dir
git clone git@github.com:org/repo.git  # Clone existing

# Clone options
git clone --depth 1 <url>     # Shallow clone (faster, CI/CD use)
git clone --branch dev <url>  # Clone specific branch
git clone --single-branch --branch main <url>  # Only main history
```

### Daily Workflow Commands

```bash
# Status and diff
git status                    # Show working tree status
git status -s                 # Short format
git diff                      # Unstaged changes
git diff --staged             # Staged changes
git diff HEAD~1               # Changes from last commit
git diff main feature-branch  # Compare branches

# Staging
git add file.txt              # Stage specific file
git add .                     # Stage all changes in current dir
git add -p                    # Interactive staging (patch mode)
git add -u                    # Stage modified/deleted (not new)
git restore --staged file.txt  # Unstage (modern)
git reset HEAD file.txt        # Unstage (older syntax)

# Committing
git commit -m "feat: add user authentication"
git commit -am "fix: update config"    # Stage and commit (tracked files only)
git commit --amend -m "corrected msg"  # Amend last commit (before push only!)
git commit --amend --no-edit           # Amend without changing message

# Log
git log                       # Full log
git log --oneline             # Compact
git log --oneline --graph     # With ASCII branch graph
git log --oneline -10         # Last 10 commits
git log --author="Name"       # Filter by author
git log --since="2024-01-01" --until="2024-12-31"
git log -p                    # Show patches (diffs)
git log --stat                # Show file change stats
git log main..feature         # Commits in feature but not main
git shortlog -sn              # Commit count per author
```

---

## 4. Branching and Merging

```bash
# Branch management
git branch                    # List local branches
git branch -a                 # List all branches (local + remote)
git branch -v                 # Branches with last commit
git branch feature/login      # Create branch (stay on current)
git checkout -b feature/login  # Create and switch (older)
git switch -c feature/login    # Create and switch (modern, git 2.23+)
git switch main               # Switch branch
git branch -d feature/login   # Delete merged branch
git branch -D feature/login   # Force delete (unmerged)
git branch -m old-name new-name  # Rename branch

# Merging
git merge feature/login       # Merge into current branch
git merge --no-ff feature     # Force merge commit (no fast-forward)
git merge --squash feature    # Squash all commits into one staged change
git merge --abort             # Abort merge in conflict

# Merge strategies:
#
# Fast-forward (default when possible):
# main: A--B--C
# feature:       D--E
# After merge: A--B--C--D--E (linear history)
#
# Three-way merge (--no-ff):
# main: A--B--C---F (F is merge commit)
#              \ /
# feature: D--E
# Preserves branch history

# Rebasing
git rebase main               # Rebase current branch onto main
git rebase -i HEAD~5          # Interactive rebase (squash, reorder, edit)
git rebase --continue         # Continue after resolving conflict
git rebase --abort            # Abort rebase

# Interactive rebase actions:
# pick   - keep commit as-is
# reword - keep commit, change message
# edit   - pause to amend commit
# squash - combine with previous commit
# fixup  - like squash but discard message
# drop   - remove commit entirely

# Conflict resolution
git status                    # See conflicted files
# Edit files to resolve conflicts
# Conflict markers:
# <<<<<<< HEAD
# your changes
# =======
# their changes
# >>>>>>> feature-branch

git add resolved-file.txt     # Mark resolved
git commit                    # Complete merge
# or
git rebase --continue         # Complete rebase

# Stash (save work temporarily)
git stash                     # Stash current changes
git stash save "work in progress"  # With description
git stash list                # List stashes
git stash pop                 # Apply and remove latest stash
git stash apply stash@{2}     # Apply specific stash (keep it)
git stash drop stash@{0}      # Delete stash
git stash branch feature-name # Create branch from stash
```

---

## 5. Remote Repositories

```bash
# Remote management
git remote -v                           # List remotes with URLs
git remote add origin git@github.com:org/repo.git  # Add remote
git remote remove origin               # Remove remote
git remote rename origin upstream      # Rename
git remote set-url origin <new-url>    # Change URL

# Fetch and Pull
git fetch origin              # Download but don't merge
git fetch --all               # Fetch all remotes
git pull origin main          # Fetch + merge
git pull --rebase origin main # Fetch + rebase (cleaner history)

# Push
git push origin main          # Push to remote
git push -u origin feature    # Push and set upstream
git push origin --delete feature-branch  # Delete remote branch
git push --force-with-lease   # Safer force push (fails if remote changed)
git push --tags               # Push all tags

# Tracking branches
git branch -u origin/main main  # Set upstream
git branch --track feature origin/feature  # Create tracking branch

# Tags
git tag                       # List tags
git tag v1.0.0                # Lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"  # Annotated tag (preferred)
git tag -a v1.0.0 abc1234    # Tag specific commit
git push origin v1.0.0        # Push specific tag
git push origin --tags        # Push all tags
git tag -d v1.0.0             # Delete local tag
git push origin --delete v1.0.0  # Delete remote tag
```

---

## 6. Advanced Git

```bash
# Cherry-pick (apply specific commits)
git cherry-pick abc1234                   # Apply one commit
git cherry-pick abc1234..def5678         # Range of commits
git cherry-pick -n abc1234               # Stage without committing

# Reset (be careful!)
git reset --soft HEAD~1    # Move HEAD, keep staged changes
git reset --mixed HEAD~1   # Move HEAD, unstage (default)
git reset --hard HEAD~1    # Move HEAD, discard all changes (DESTRUCTIVE!)
git reset --hard origin/main  # Match remote state (DESTRUCTIVE!)

# Revert (safe — creates new commit)
git revert abc1234         # Revert commit (create new commit)
git revert HEAD~3..HEAD    # Revert last 3 commits
git revert -n abc1234      # Revert without committing

# Reflog (recovery tool)
git reflog                 # History of HEAD movements
git reflog show feature    # Reflog for specific branch
# Recover deleted branch:
git reflog                 # Find the SHA of the last commit
git checkout -b recovered-branch <sha>

# Bisect (find commit that introduced a bug)
git bisect start
git bisect bad              # Current commit is bad
git bisect good v2.0.0     # This version was good
# Git checks out middle commit
# Test it, then:
git bisect good             # Or:
git bisect bad
# Git narrows down until finding the culprit
git bisect reset            # End bisect session

# Blame (find who changed what)
git blame file.txt          # Line-by-line author info
git blame -L 10,20 file.txt # Lines 10-20 only
git blame --follow -p file  # Follow through renames

# Submodules
git submodule add git@github.com:org/lib.git libs/lib
git submodule update --init --recursive  # Initialize after clone
git submodule update --remote            # Update to latest

# Worktrees (work on multiple branches simultaneously)
git worktree add ../hotfix hotfix-branch  # Separate working dir
git worktree list
git worktree remove ../hotfix
```

---

## 7. Git Workflows

### Git Flow

```
main ────────────────────────────────────────────────► (stable releases)
  │                                                   ▲
  └── develop ─────────────────────────────────────► (integration)
         │          ▲                │           ▲
         └── feature/A         release/1.0 ────┘
         └── feature/B               │
                                 hotfix/1.0.1 ─► main + develop
```

```bash
# Git Flow commands (git-flow tool)
git flow init
git flow feature start user-authentication
git flow feature finish user-authentication
git flow release start 1.0.0
git flow release finish 1.0.0
git flow hotfix start 1.0.1
git flow hotfix finish 1.0.1
```

### GitHub Flow (Simpler)

```
main ─────────────────────────────────────────────────►
  │                                                   ▲
  └── feature/auth ─► PR Review ─► Merge ─► Deploy ──┘
```

```bash
# GitHub Flow
git switch -c feature/user-auth
# ... make changes ...
git push -u origin feature/user-auth
# Create PR on GitHub
# After merge:
git switch main && git pull
git branch -d feature/user-auth
```

### Trunk-Based Development

```
main ─────────────────────────────────────────────────►
  │         │         │
  └─ short  ┘  direct │  short
  feature         commit  branch
  (< 1 day)              (1-2 days)
```

---

## 8. Git Hooks

Git hooks are scripts triggered by Git events:

```
Client-side hooks:
  pre-commit      → Run before commit (linting, tests)
  commit-msg      → Validate commit message format
  pre-push        → Run before push (tests)

Server-side hooks:
  pre-receive     → Validate before accepting push
  post-receive    → Deploy after push
  update          → Per-branch access control
```

```bash
# Location: .git/hooks/ (must be executable)

# pre-commit: Run linting before every commit
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
echo "Running pre-commit checks..."

# Run linter
if ! eslint src/; then
    echo "ESLint failed. Commit aborted."
    exit 1
fi

# Check for debug statements
if grep -r "debugger\|console\.log" src/; then
    echo "Remove debug statements before committing"
    exit 1
fi

exit 0
EOF
chmod +x .git/hooks/pre-commit

# commit-msg: Enforce conventional commits
cat > .git/hooks/commit-msg << 'EOF'
#!/bin/bash
COMMIT_MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|ci)(\(.+\))?: .+"
if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
    echo "ERROR: Commit message must follow Conventional Commits format"
    echo "Example: feat(auth): add JWT validation"
    exit 1
fi
EOF
chmod +x .git/hooks/commit-msg
```

### Sharing Hooks (using pre-commit framework)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
```

```bash
pip install pre-commit
pre-commit install          # Install hooks
pre-commit run --all-files  # Run manually
```

---

## 9. Git in CI/CD

```bash
# CI/CD common patterns

# 1. Get current branch name
BRANCH=$(git rev-parse --abbrev-ref HEAD)
COMMIT_SHA=$(git rev-parse --short HEAD)
COMMIT_MSG=$(git log -1 --pretty=%B)
TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "untagged")

# 2. Check if working tree is clean
if ! git diff --quiet; then
    echo "Working tree is dirty!"
    exit 1
fi

# 3. Get changed files (for smart CI — only run affected tests)
CHANGED_FILES=$(git diff --name-only HEAD~1)
if echo "$CHANGED_FILES" | grep -q "^frontend/"; then
    run_frontend_tests
fi
if echo "$CHANGED_FILES" | grep -q "^backend/"; then
    run_backend_tests
fi

# 4. Semantic versioning from tags
git tag -a v$(cat VERSION) -m "Release $(cat VERSION)"
git push origin --tags

# 5. Generate changelog
git log --pretty="- %s" v1.0.0..v2.0.0 > CHANGELOG.md

# .github/workflows/ci.yml
```

```yaml
# GitHub Actions CI
name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Full history for versioning
      
      - name: Get version
        run: echo "VERSION=$(git describe --tags --abbrev=0)" >> $GITHUB_ENV
      
      - name: Run tests
        run: make test
      
      - name: Build Docker image
        run: docker build -t app:${{ github.sha }} .
      
      - name: Push to ECR
        if: github.ref == 'refs/heads/main'
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URL
          docker push app:${{ github.sha }}
```

---

## 10. GitHub/GitLab Operations

```bash
# GitHub CLI (gh)
gh auth login                      # Authenticate
gh repo create my-repo --public    # Create repo
gh pr create --title "Add feature" --body "..."  # Create PR
gh pr list                         # List PRs
gh pr merge 42 --squash           # Merge PR
gh pr review 42 --approve         # Approve PR
gh issue create --title "Bug" --body "..."
gh issue list --assignee @me
gh release create v1.0.0 --notes "Release notes"
gh workflow run deploy.yml         # Trigger workflow
gh run list                        # List workflow runs

# GitLab API examples
# Create MR via API
curl --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --data "source_branch=feature&target_branch=main&title=My MR" \
  "https://gitlab.com/api/v4/projects/123/merge_requests"
```

---

## 11. Git Troubleshooting

```bash
# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Undo last commit (keep changes unstaged)
git reset --mixed HEAD~1

# Completely undo last commit (DESTRUCTIVE)
git reset --hard HEAD~1

# Recover accidentally deleted branch
git reflog
git checkout -b recovered <sha>

# Recover accidentally committed sensitive file
git filter-repo --path secret.txt --invert-paths
# or (older method)
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secret.txt' \
  --prune-empty --tag-name-filter cat -- --all

# Fix wrong commit author
git commit --amend --author="Correct Name <correct@email.com>"

# Remove file from ALL history (nuclear option)
git filter-repo --invert-paths --path-glob '*.credentials'
git push origin --force --all

# Resolve "refusing to merge unrelated histories"
git pull origin main --allow-unrelated-histories

# Clean untracked files
git clean -fd               # Remove untracked files and directories
git clean -fdx              # Also remove gitignored files
git clean -n                # Dry run (see what would be deleted)

# Ignore already-tracked file
git rm --cached file.txt    # Remove from tracking, keep file
echo "file.txt" >> .gitignore
git commit -m "stop tracking file.txt"
```

---

## 12. Common Interview Questions

**Q1: What is the difference between git merge and git rebase?**
> `merge` creates a new merge commit, preserving complete history of both branches (non-destructive). `rebase` rewrites history by replaying commits on top of another branch — creates linear history but changes commit SHAs. Rule: never rebase shared/public branches; rebase local feature branches before merging.

**Q2: What is the difference between `git fetch` and `git pull`?**
> `fetch` downloads remote changes to remote-tracking branches (`origin/main`) without modifying your working directory. `pull` = `fetch` + `merge` (or `rebase`). Best practice: `fetch` first, review, then `merge` or `rebase` — gives you control.

**Q3: What is a detached HEAD state?**
> HEAD normally points to a branch which points to a commit. In detached HEAD, HEAD points directly to a commit (not a branch). Happens when you checkout a tag or old commit. Commits made in this state are orphaned — save them: `git switch -c new-branch`.

**Q4: How do you revert a commit that was already pushed?**
> Use `git revert <sha>` — creates a new commit that undoes changes. Never `git reset --hard` on pushed commits (rewrites history, breaks others). `revert` is safe for shared branches.

**Q5: What is `git stash` and when do you use it?**
> `stash` temporarily saves uncommitted changes so you can switch branches or pull without committing half-done work. `git stash` saves; `git stash pop` restores. Use when interrupted mid-feature for a hotfix.

**Q6: How do you squash commits before merging a PR?**
> `git rebase -i HEAD~N` (N = number of commits to squash). Mark all but first as `squash` or `fixup`. Or use `git merge --squash feature-branch` on main — stages all changes as one commit. GitHub/GitLab also have "Squash and merge" option on PRs.

**Q7: What is a fast-forward merge and when does it happen?**
> Fast-forward happens when the target branch hasn't diverged from the source — Git just moves the pointer forward. No merge commit created. Happens when: checked out main, created feature, main got no new commits. Use `--no-ff` to always create merge commit.

**Q8: How do you configure Git to use a proxy?**
> `git config --global http.proxy http://proxy:8080`. Or per-repo: `git config http.proxy http://proxy:8080`. For HTTPS: `git config --global https.proxy https://proxy:8080`. Unset: `git config --global --unset http.proxy`.

**Q9: What is the .gitignore and what are its rules?**
> .gitignore patterns: `*.log` ignores all .log files; `logs/` ignores directory; `/root-only.txt` ignores only in root; `!important.log` negates (don't ignore). Patterns apply from .gitignore location. `git check-ignore -v file.txt` shows which rule ignores a file.

**Q10: Explain the difference between a tag and a branch.**
> Branch is a movable pointer to a commit — moves forward with new commits. Tag is a fixed pointer to a specific commit (a release snapshot) — never moves. Annotated tags have metadata (tagger, date, GPG signature); use for releases. Lightweight tags are just aliases.

---

*Next: [Git Assignments](02-git-assignments.md)*
