# Git Workflow for OpenCode

## Remote Setup

| Remote   | URL                                      | Purpose                  |
|----------|------------------------------------------|--------------------------|
| origin   | https://github.com/hw92/opencode.git     | Your fork (push here)    |
| upstream | https://github.com/anomalyco/opencode.git| Source repo (pull from)  |

## Branch Strategy

```
dev  ← stays in sync with upstream
 └── lab  ← your experiments
```

- **dev**: Keep clean, only for syncing with upstream
- **lab**: Your experiments and tests

## Daily Workflow

### Push your changes
```bash
git add .
git commit -m "your message"
git push origin lab
```

### Sync with upstream
```bash
# Update dev from upstream
git checkout dev
git fetch upstream
git merge upstream/dev

# Rebase lab onto updated dev
git checkout lab
git rebase dev
```

### If rebase has conflicts
```bash
# Fix conflicts in files, then:
git add .
git rebase --continue
```

## Quick Reference

```bash
git remote -v          # Check remotes
git branch             # See current branch
git status             # See changes
git log --oneline -5   # Recent commits
```

---

*Written by Claude (Opus 4.5) | 2026-01-17 PST*
