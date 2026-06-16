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

A secret in production, an insecure endpoint, a misconfigured bucket in the cloud. Pre-commit hooks are a chance to catch those mistakes before they're written to history.

**TL;DR**: Pre-commit hooks run automated checks before every `git commit`. They catch security issues at the point of authorship, the cheapest time to fix anything.

---

## Why pre-commit hooks

After `git commit`, the code exists in the repository. After `git push`, it exists in CI. After a merge, it's on its way to production. Each step out costs more to fix than the last.

Fixing a secret found in CI costs at least one round trip. Detect the issue, revert the commit, apply the fix, recommit, push again, rotate your secret! Fixing it at commit time costs near zero. The same math applies to insecure code, misconfigured infrastructure, anything that automated checks can catch.

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

This is a shell script that runs whatever you want before a commit, but it's not managed by Git itself. That adds flexibility to a developer's setup, but at the cost of a more fragile setup. This can be used by developers to add their own preferred checks to the team's pre-commit framework as long as it doesn't conflict.

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
        files: \.(tf|yaml|yml|json|bicep|template|Dockerfile|amplify\.yml|template\.yaml|openapi\.yaml|openapi\.json)$
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

## Manual git hooks

If you don't want the pre-commit framework, you can chain tools together in a shell script. This is what the individual tool documentation shows for each tool in isolation.

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running gitleaks..."
gitleaks detect --source . --verbose --redact /dev/null
if [ $? -ne 0 ]; then
  echo "gitleaks found secrets. Commit blocked."
  exit 1
fi

echo "Running checkov..."
checkov -d .
if [ $? -ne 0 ]; then
  echo "checkov found misconfigurations. Commit blocked."
  exit 1
fi

echo "All checks passed."
```

Make it executable:

```bash
chmod +x .git/hooks/pre-commit
```

The manual approach is straightforward but requires developers to install all tools manually, and requires every dev to keep their hooks up to date as the project evolves.

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
        files: \.(tf|yaml|yml|json)$
        stages: [push]
```

This runs checkov only on `git push` (pre-push hook), not on every commit. Developers can commit IaC changes and push them later for the full scan. You lose the pre-commit guarantee for checkov but gain speed during active development.

---

## CI is still a requirement

Pre-commit hooks are a developer-side safety net. CI is a vital second line. If someone bypasses the hook or forgets to install it, CI catches it later. It's not as clean because the commit has already landed, but it's better than nothing.

```yaml
# .github/workflows/security-checks.yml
name: Security Checks
on: [pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install gitleaks
        run: |
          curl -sS https://web.archive.org/web/20250101000000/https://github.com/gitleaks/gitleaks/releases/download/v8.29.0/gitleaks_8.29.0_linux_x64.tar.gz | tar xz
          sudo cp gitleaks /usr/local/bin/

      - name: Run gitleaks
        run: gitleaks detect --source . --verbose --redact
        continue-on-error: true

      - name: Install Bandit
        run: pip install bandit[sarif]

      - name: Run Bandit
        run: bandit -r . -f sarif -o bandit-results.sarif

      - name: Upload Bandit SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: bandit-results.sarif
        continue-on-error: true

      - name: Install Checkov
        run: pip install checkov

      - name: Run Checkov
        run: checkov -d . -o sarif -o checkov-results.sarif

      - name: Upload Checkov SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov-results.sarif
        continue-on-error: true
```

The `continue-on-error: true` flags prevent the job from failing entirely on each individual tool. Instead, SARIF results upload to GitHub Code Scanning regardless, which surfaces findings as inline annotations on the PR.

---

## Choose the tools

You don't need to run every check. Pick the ones that match what you ship with an eye for guarding secrets and keeping your systems secure.

| Domain | Tool | What it catches |
|--------|------|-----------------|
| Secrets | gitleaks | API keys, tokens, credentials in Git history |
| Python | bandit | `eval()`, `os.system()`, weak crypto, hardcoded secrets |
| Infrastructure | checkov | Open buckets, unencrypted DBs, permissive access policies |

Each catches a different class of mistake. Running all three means you're not leaving a category unchecked. A developer might skip secrets scanning, then push Python code with `eval()` calls, while the IaC repo has a public S3 bucket. One tool catches one thing. Two tools catch two things. Three tools catch three things.

---

## Takeaways

- Pre-commit hooks catch issues at the point of authorship — the cheapest time to fix anything
- The pre-commit framework manages hooks and installation declaratively. Manual git hooks are a shell script that requires manual setup and manual tool installation
- `language: system` hooks get no virtual environment — pre-commit won't install checkov for you because it has no official remote repo. Developers must install it themselves
- CI is a vital second line, not just a fallback. If someone bypasses the hook or forgets to install it, CI catches it later. The commit has already landed, but it's better than nothing
- Use `stages: [push]` for slow tools to avoid blocking on quick commits during active development
- Each tool catches a different class of mistake. Gitleaks catches secrets in Git history, bandit catches insecure Python patterns, checkov catches IaC misconfigurations — together they cover different attack surfaces
