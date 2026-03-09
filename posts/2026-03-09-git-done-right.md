---
title: "Git Done Right: Habits That Separate Senior Devs from the Rest"
author: "Matt"
date: "2026-03-09"
category: "engineering"
excerpt: "Git is on every resume, but most devs only use 20% of it. Here's how to actually use it well — commits, branching, hooks, and all."
tags: ["Git", "DevOps", "Version Control", "Best Practices", "CI/CD"]
readTime: "8 min read"
---

# Git Done Right: Habits That Separate Senior Devs from the Rest

Git is on every developer's resume. But there's a wide gap between *knowing* Git and *using* it well. Sloppy commits, bloated branches, and merge nightmares are signs of a team fighting their tooling — not leveraging it.

The good news? A handful of deliberate habits can transform your Git workflow from chaotic to clean. Here's what actually matters in 2026.

---

## ⚙️ Part 1: Commits — The Foundation of Everything

Your commit history is your project's autobiography. Make it worth reading.

### ✅ Write Atomic Commits

One commit = one logical change. Not "refactored some stuff and fixed the login bug." If you can't summarize your commit in a single sentence, it's probably too big.

Small, focused commits make code review easier, bugs easier to bisect, and rollbacks surgical instead of destructive.

### ✅ Write Meaningful Commit Messages

Follow the **imperative mood** convention — write your message as if you're giving the codebase an order:

```
# Bad
fixed stuff
updated auth

# Good
Add JWT refresh token rotation to auth service
Fix null pointer in user profile loader
```

Structure: a short subject line (50 chars max), a blank line, then an optional body explaining *why* — not just what.

### ✅ Commit Often, Clean Up Before You Push

Commit early and often locally — don't wait until "it works." Use `git stash` to keep your working tree clean when you need to context-switch. Before pushing, use interactive rebase to clean up your history:

```bash
git rebase -i HEAD~4  # Squash, reorder, or edit your last 4 commits
```

Never rebase shared branches like `main` or `develop`. Only rebase commits that haven't left your machine yet.

---

## 🌿 Part 2: Branching Strategy — Pick One and Stick to It

The best branching strategy is the one your whole team actually follows.

### The Two Dominant Models in 2026

**Git Flow** — Two long-lived branches (`main`, `develop`) plus short-lived `feature/`, `bugfix/`, and `hotfix/` branches. Good for projects with scheduled release cycles.

**Trunk-Based Development** — Everyone branches off `main`, keeps branches short-lived (ideally < 1 day), and merges back fast. Best for CI/CD-heavy teams shipping continuously.

Either way, follow a consistent naming convention:

```
feature/NYX-204-vnav-panel-redesign
bugfix/NYX-187-redux-timing-fix
hotfix/auth-token-expiry
```

### Keep Branches Short-Lived

The longer a branch lives away from `main`, the more painful the merge. Aim to merge feature branches within a few days — use **feature flags** to ship incomplete work safely without long-lived branches.

### Clean Up After Yourself

```bash
git branch -d feature/my-merged-branch   # Delete locally
git push origin --delete feature/my-merged-branch  # Delete remotely
```

Dead branches are visual noise. Kill them.

---

## 🔗 Part 3: Pull Requests & Code Review

PRs aren't just a formality — they're your team's primary knowledge-sharing mechanism.

- **Keep PRs small.** A PR that touches 20 files gets rubber-stamped. A PR that touches 3 files gets reviewed properly.
- **Write a useful PR description.** What changed? Why? Is there anything tricky to look out for?
- **Use draft PRs** to get early feedback before you're "done."
- **Link to the ticket.** Always. Your future self will thank you at 2am.

---

## 🛡️ Part 4: Automation with Git Hooks

Git hooks let you enforce standards automatically — before bad code ever hits the remote.

A `pre-commit` hook can run your linter, formatter, and unit tests before every commit:

```bash
# .git/hooks/pre-commit
#!/bin/sh
npm run lint && npm test
```

Use a tool like **Husky** (Node) or **pre-commit** (Python) to manage hooks as part of your repo — so every team member gets them automatically. No more "works on my machine."

---

## 🔒 Part 7: Security & Hygiene — The Stuff That Bites You

### Never Commit Secrets

If an API key or `.env` file ends up in your history, treat it as compromised immediately — even if the repo is private. Use tools like **git-secrets** or **truffleHog** in your CI pipeline to catch secrets before they land.

```bash
# Add .env to .gitignore BEFORE you ever create it
echo ".env" >> .gitignore
```

### Sign Your Commits

Commit signing with GPG or SSH keys verifies that commits are actually from you — critical for open source contributions and increasingly expected in enterprise/defense environments:

```bash
git config --global commit.gpgsign true
```

### Use `.gitignore` Properly

Every repo needs one. Use [gitignore.io](https://gitignore.io) or the [github/gitignore](https://github.com/github/gitignore) templates to generate a solid starting point for your stack.

---

## 🚀 Part 5: CI/CD Integration — Automate the Boring Stuff

Git doesn't live in a vacuum. In 2026, a well-wired Git workflow triggers automation at every step — so humans spend their time on code, not on ceremony.

### Every Push Should Do Something

At minimum, pushing to any branch should trigger:

- **Linting & formatting checks** — catch style issues before review
- **Unit + integration tests** — catch regressions before merge
- **Security scanning** — catch secrets and known CVEs before they hit production

A simple GitHub Actions workflow looks like this:

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

If you're on GitLab, CircleCI, or running your own runners — same idea, different syntax. The point is: **no human should be manually running tests before merging.** That's the pipeline's job.

### Protect Your Main Branch

Set branch protection rules so that:

- Direct pushes to `main` are blocked
- PRs require at least one approval
- Status checks (CI passing) are required before merge

This is a 5-minute configuration in GitHub or GitLab that prevents an entire category of "oops" moments.

### Tag Your Releases

Use annotated Git tags to mark production releases — don't rely on branch names or commit SHAs:

```bash
git tag -a v1.4.2 -m "Release v1.4.2 — VNAV panel hotfix"
git push origin v1.4.2
```

Tags are static pointers in history. They make rollbacks, changelogs, and deployment pipelines dramatically easier to reason about.

---

## ⚡ Part 6: Power Habits That Save Time Daily

Once the fundamentals are solid, these are the habits that separate developers who *fight* Git from developers who *fly* with it.

### Git Aliases — Stop Typing the Same Commands

Set up aliases for the commands you run twenty times a day:

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"
```

That last one — `git lg` — gives you a beautiful visual branch graph in the terminal. Way more useful than plain `git log`.

### Use `git bisect` to Find Bugs Fast

`git bisect` is criminally underused. It performs a binary search through your commit history to find exactly which commit introduced a bug:

```bash
git bisect start
git bisect bad                  # Current commit is broken
git bisect good v1.3.0          # This tag was working fine
# Git checks out the midpoint — you test, then mark good or bad
# Repeat until Git pinpoints the exact offending commit
git bisect reset                # Done — return to HEAD
```

This turns a "I have no idea when this broke" into a 5-minute investigation.

### Use `git reflog` to Recover from Mistakes

Accidentally deleted a branch? Rebased wrong? `git reflog` is your safety net — it records every position HEAD has been at, even if the commits are no longer reachable:

```bash
git reflog               # See recent HEAD positions
git checkout -b recovered-branch HEAD@{4}  # Restore from 4 moves ago
```

This has saved countless devs from genuine data loss. Bookmark it mentally.

### Stash with a Purpose

Instead of a pile of unnamed stashes, label them:

```bash
git stash push -m "WIP: VNAV altitude input validation"
git stash list
# stash@{0}: On feature/vnav-panel: WIP: VNAV altitude input validation
git stash pop stash@{0}
```

Unnamed stashes become mysterious and terrifying after 48 hours. Name them.

---

## ⚠️ Common Pitfalls to Avoid

1. **Force-pushing to shared branches** — Rewrites history for everyone. Only force-push to your own feature branches, never to `main`.
2. **Giant "catch-all" commits** — Hard to review, impossible to rollback cleanly.
3. **Ignoring merge conflicts** — Don't just accept "incoming" blindly. Read the diff.
4. **Skipping `.gitignore`** — `node_modules`, build artifacts, and `.env` files have no business in your repo.
5. **Meaningless branch names** — `test2`, `fix`, `mikes-branch` tell reviewers nothing.
6. **No CI integration** — Every push should trigger automated tests. If it doesn't, you're flying blind.

---

## 🎯 Conclusion: Git Is a Communication Tool

More than anything, Git is how your team communicates across time. Your commit messages and branch structure are messages to the developer who has to debug this code at midnight six months from now — and that developer might be you.

The habits here aren't advanced — they're just disciplined. Atomic commits, clean history, short-lived branches, automated hooks, and secure hygiene compound over time into a codebase that's a pleasure to work in.

Good Git hygiene isn't optional in 2026. It's table stakes.

---

**👉 What's the worst Git sin you've seen on a real project?** Drop it in the comments — we've all got war stories.
