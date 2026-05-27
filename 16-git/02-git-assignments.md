# Git — Assignments

## Assignment 1: Git Repository Setup and Basics

```bash
# 1. Create a new project repository
mkdir myapp && cd myapp
git init
git config user.name "Your Name"
git config user.email "you@example.com"

# 2. Create initial project structure
mkdir -p src tests docs
touch src/app.py tests/test_app.py README.md .gitignore

# 3. Create a meaningful .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
.env
.venv/
*.log
dist/
build/
.pytest_cache/
EOF

# 4. Make your first commit with conventional commit message
git add .
git commit -m "feat: initial project structure"

# 5. Add some code and create multiple commits:
# feat: add user authentication module
# feat: add database connection
# test: add unit tests for auth
# docs: update README with setup instructions

# 6. View full log with graph
git log --oneline --graph --all
```

---

## Assignment 2: Branching Workflow

**Practice the GitHub Flow:**

```bash
# Starting from main branch with some commits

# 1. Create feature branch
git switch -c feature/user-registration

# 2. Make 3 commits:
# - feat: add user model
# - feat: add registration endpoint  
# - test: add registration tests

# 3. Simulate another developer's work on main
git switch main
echo "# Updated" >> README.md
git commit -am "docs: update installation guide"

# 4. Rebase feature onto updated main
git switch feature/user-registration
git rebase main

# 5. Squash your 3 feature commits into one
git rebase -i HEAD~3
# Change first to 'pick', others to 'squash'

# 6. Merge to main with --no-ff (preserve branch history)
git switch main
git merge --no-ff feature/user-registration -m "feat: add user registration (#42)"

# 7. Delete the feature branch
git branch -d feature/user-registration

# 8. View the history graph
git log --oneline --graph -10
```

---

## Assignment 3: Conflict Resolution

```bash
# Create a merge conflict scenario and resolve it

# 1. Create base file
echo "Line 1: original" > config.yaml
echo "Line 2: setting=default" >> config.yaml
echo "Line 3: timeout=30" >> config.yaml
git add config.yaml
git commit -m "feat: initial config"

# 2. Create two branches that modify the same line
git switch -c branch-a
sed -i 's/setting=default/setting=production/' config.yaml
git commit -am "config: set production setting"

git switch main
git switch -c branch-b
sed -i 's/setting=default/setting=staging/' config.yaml
git commit -am "config: set staging setting"

# 3. Merge branch-a into main first
git switch main
git merge branch-a

# 4. Try to merge branch-b (will conflict)
git merge branch-b

# 5. Resolve the conflict:
# - Open config.yaml
# - Remove conflict markers (<<<, ===, >>>)
# - Choose the correct value or combine
# - git add config.yaml
# - git commit

# 6. Verify the resolution
cat config.yaml
git log --oneline --graph -5
```

---

## Assignment 4: Git Hooks

**Set up pre-commit hooks for code quality:**

```bash
# 1. Create pre-commit hook that:
# - Runs python syntax check (python -m py_compile)
# - Checks for debug statements (print statements, debugger)
# - Prevents committing to main directly
# - Checks file size (warn if > 1MB)

cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/bash
set -e

BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Prevent direct commits to main
if [[ "$BRANCH" == "main" ]]; then
    echo "ERROR: Direct commits to main are not allowed"
    echo "Please use a feature branch and PR"
    exit 1
fi

# Check for debug statements in Python files
if git diff --cached --name-only | grep -q '\.py$'; then
    if git diff --cached | grep -E '^\+.*(print\(|debugger|import pdb)'; then
        echo "ERROR: Remove debug statements before committing"
        exit 1
    fi
fi

# Syntax check Python files
for file in $(git diff --cached --name-only | grep '\.py$'); do
    python -m py_compile "$file" 2>&1 || {
        echo "Syntax error in $file"
        exit 1
    }
done

echo "Pre-commit checks passed!"
HOOK
chmod +x .git/hooks/pre-commit

# 2. Create commit-msg hook for conventional commits
cat > .git/hooks/commit-msg << 'HOOK'
#!/bin/bash
PATTERN="^(feat|fix|docs|style|refactor|test|chore|ci|perf)(\(.+\))?: .{1,72}"
if ! grep -qE "$PATTERN" "$1"; then
    echo "ERROR: Invalid commit message format"
    echo "Use: type(scope): description"
    echo "Types: feat|fix|docs|style|refactor|test|chore|ci|perf"
    echo "Example: feat(auth): add JWT validation"
    exit 1
fi
HOOK
chmod +x .git/hooks/commit-msg

# 3. Test the hooks:
# Should fail: git commit -m "fixed stuff"  
# Should pass: git commit -m "fix(auth): correct JWT expiry validation"
```

---

## Assignment 5: Git Bisect — Find the Bug

```bash
# Simulate a regression and find it with git bisect

# Create history with a bug introduced somewhere
for i in {1..10}; do
    echo "Feature $i" >> features.txt
    git add features.txt
    if [ $i -eq 7 ]; then
        echo "BUG_INTRODUCED=true" >> config.py
        git add config.py
    fi
    git commit -m "feat: add feature $i"
done

# Now find which commit introduced the bug
git bisect start
git bisect bad HEAD
git bisect good HEAD~10   # Before any of our commits

# For each checkout, test and tell git:
python -c "import config; assert not hasattr(config, 'BUG_INTRODUCED')"
# if passes: git bisect good
# if fails:  git bisect bad

# Automate bisect with a test script
git bisect run python -c "
import subprocess
result = subprocess.run(['grep', '-l', 'BUG_INTRODUCED', 'config.py'])
exit(result.returncode)
"

git bisect reset  # Return to HEAD
```

---

## Assignment 6: Git for Teams — Collaborative Workflow

**Simulate a team workflow with conflicts:**

```bash
# Developer 1: Work on auth feature
git switch -c feature/oauth2
# ... multiple commits ...
git push -u origin feature/oauth2

# Developer 2: Review and suggest changes
# Add review comment in PR

# Developer 1: Address review, force push safely
git commit --amend -m "feat(auth): implement OAuth2 with review fixes"
git push --force-with-lease origin feature/oauth2  # Safer than --force

# After PR approval, squash merge to main
git switch main
git merge --squash feature/oauth2
git commit -m "feat(auth): implement OAuth2 authentication (#47)

Co-authored-by: Reviewer Name <reviewer@company.com>"

# Tag the release
git tag -a v1.5.0 -m "Release 1.5.0 - OAuth2 support"
git push origin v1.5.0

# Create release notes
git log v1.4.0..v1.5.0 --pretty="- %s" | sort > RELEASE_NOTES.md
```

---

## Interview Assignment: Git Recovery Scenarios

**Scenario 1: Accidentally deleted main branch**
```bash
# Recover deleted branch using reflog
git reflog --all | grep main
git checkout -b main <sha>
# OR
git branch main <sha>
```

**Scenario 2: Force push wiped out 5 commits**
```bash
# Check reflog for the missing commits
git reflog
# Find SHA before force push
git reset --hard <sha-before-force-push>
# Recover lost commits
git cherry-pick <lost-commit-sha>
```

**Scenario 3: Committed secrets (API key) to main**
```bash
# IMMEDIATELY:
# 1. Revoke the compromised credentials
# 2. Remove from history
git filter-repo --invert-paths --path secrets.env

# Force push ALL branches and tags
git push origin --force --all
git push origin --force --tags

# Notify all team members to re-clone (history has changed)
# 4. Add secrets.env to .gitignore
```

**Scenario 4: Wrong commit on wrong branch**
```bash
# If commit is last on feature-branch but should be on hotfix-branch
git log --oneline -1  # Note the SHA
git switch hotfix-branch
git cherry-pick <sha>
git switch feature-branch
git reset HEAD~1  # Remove from feature-branch
```

---

## Git Workflow Reference Card

```bash
# Daily workflow
git status          # What changed?
git diff            # What are the changes?
git add -p          # Stage selectively (review each hunk)
git commit          # Commit with editor
git push            # Push to remote

# Useful aliases to add to ~/.gitconfig
[alias]
    st = status -s
    lg = log --oneline --graph --all
    unstage = restore --staged
    last = log -1 HEAD --stat
    undo = reset --soft HEAD~1
    tags = tag -l --sort=-v:refname
    branches = branch -a --sort=-committerdate
    cleanup = "!git branch --merged | grep -v main | xargs git branch -d"
```
