---
date: 2026-06-15
authors:
  - alismx
categories:
  - DevSecOps
tags:
  - pre-commit
  - security
  - git
  - ci
  - shift-left
  - bandit
  - gitleaks
  - checkov
  - sast
  - secrets
  - python
  - terraform
  - iac
pin: true
---

# DevSecOps: Pre-commit Hooks for Automated Checks

A secret committed to Git leaks to anyone with read access to the repo. Pre-commit hooks are a chance to catch those mistakes before they're written to history.

**TL;DR**: Pre-commit hooks run automated checks before every `git commit`. They catch security issues before it's committed, the cheapest time to fix anything.

---

## Why pre-commit hooks

After `git commit`, the code exists in the repository. After `git push`, it exists in CI. After a merge, it's on its way to production. Each step out costs more to fix than the last.

---

## How pre-commit hooks work

A pre-commit hook is a script stored in `.git/hooks/pre-commit`. Git runs it before each commit. If the script exits with a non-zero code, the commit is blocked and the developer sees the output.

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running security checks..."
gitleaks detect --source . --verbose --redact /dev/null
if [ $? -ne 0 ]; then
  echo "Found a secret. Commit blocked."
  exit 1
fi

echo "All checks passed."
```

Make it executable:

```bash
chmod +x .git/hooks/pre-commit
```

This is a shell script that runs whatever you want before a commit, but it's not committed to the repo. This should be used by developers to add their preferred checks on top of the team's agreed upon pre-commit framework.

This approach is straightforward but requires developers to install all tools manually, and requires every dev to keep their hooks up to date as the project evolves.

---

## pre-commit framework (recommended)

The pre-commit framework ([pre-commit.com](https://pre-commit.com/)) manages hooks declaratively and handles installation automatically (or notifies you if something isn't).

### Install

```bash
pip install pre-commit
```

### Configuration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.29.0
    hooks:
      - id: gitleaks
        args: [--verbose, --redact]
  ...
  - repo: local
    hooks:
      - id: checkov
        name: checkov
        entry: checkov -d .
        language: system
        pass_filenames: false
```

Some repos like Gitleaks and Bandit have official remote hooks and can be install automatically. Checkov doesn't have an official pre-commit remote repo, so you define it as a local hook that runs the `checkov` binary directly. That also means pre-commit won't install checkov for you, so developers need to install checkov before `pre-commit install` works.

### How it works

After installing with `pre-commit install`, the hooks run automatically on every commit:

```bash
# Run install
pre-commit install
```

```bash
# Run all hooks on the entire repo
pre-commit run --all-files

# Run a specific hook
pre-commit run gitleaks

# Run hooks on staged files only
pre-commit run
```

The commit is blocked if any hook exits non-zero. The developer sees the output and can fix the issue.

---

## Execution

Gitleaks on staged files (not full history) runs fast. Bandit on staged Python files runs fast. Checkov on all IaC files in the repo takes longer because it scans the entire directory.

If you want to avoid blocking on slow findings during quick commits, use `pre-commit`'s `stages` feature:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.29.0
    hooks:
      - id: gitleaks
        stages: [commit]
  ...
  - repo: local
    hooks:
      - id: checkov
        name: checkov
        entry: checkov -d .
        language: system
        pass_filenames: false
        stages: [push]
```

This runs checkov only on `git push` (pre-push hook), not on every commit. Developers can commit IaC changes and push them later for the full scan. You lose the pre-commit guarantee for checkov but gain speed during active development.

---

## CI is still a requirement

CI catches things the developer missed. If someone bypasses the hook or forgets to install it, CI blocks the merge. It's not as clean because the commit has already landed, but it's better than nothing.

---

## Choose the tools

You don't need to run every check. Pick the ones that match what you ship with an eye for guarding secrets and keeping your systems secure.

Consider using any of these posts for inspiration:

- [Bandit](./sast-python-bandit.md): Python SAST
- [Checkov](./sast-iac-checkov.md): IaC SAST
- [Gitleaks](./sast-git-secrets-gitleaks.md): Git Secret Scanning
- [Trivy](./sast-container-trivy.md): Container SAST

---

## Takeaways

- The pre-commit framework manages hooks and installation declaratively. Manual git hooks are a shell script that requires manual setup and manual tool installation.
- `language: system` hooks get no virtual environment: pre-commit won't install checkov for you because it has no official remote repo. Developers must install it themselves.
- CI is a vital second line, not just a fallback. If someone bypasses the hook or forgets to install it, CI catches it later. The commit has already landed, but it's better than nothing.
- Use `stages: [push]` for slow tools to avoid blocking on quick commits during active development.
- Each tool catches a different class of mistake.
