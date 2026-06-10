---
date: 2026-06-09
authors:
  - alismx
categories:
  - DevSecOps
tags:
  - gitleaks
  - secrets
  - git
  - ci
pin: true
---

# DevSecOps: Scan for Secrets in Your Git Repository

Accidentally committed API keys and tokens cause breaches. Gitleaks scans Git history for these patterns.

**TL;DR**: Install gitleaks, configure your repository, run a scan, and add it to your CI pipeline to catch secrets before they reach production.

---

## Install gitleaks

```bash
sudo apt install gitleaks
```

Verify the install:

```bash
gitleaks --version
```

## Configure gitleaks

Create a `gitleaks.toml` file in your repository root:

```toml
[extend]
using = "github.com/gitleaks/gitleaks"

[allowlist]
description = "global allow lists"
regexes = [
    # patterns to allow
]
```

Add a custom rule:

```toml
[[rules]]
id = "my-custom-secret"
description = "Custom secret pattern"
regex = '''(your-pattern-here)'''
```

## Scan Your Repository

```bash
# Full history scan
gitleaks detect --source .

# Verbose output (shows actual secrets)
gitleaks detect --source . --verbose

# Verbose output (redacts actual secrets)
gitleaks detect --source . --verbose --redact
```

## Add gitleaks to CI/CD

```yaml
name: gitleaks
on: [pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Takeaways

- Scan the full Git history. People commit secrets and then delete them from current files, but the values remain in commit history.
- Use `--verbose` when investigating findings. It reveals the secret value, not just the file and line.
- Add gitleaks to your CI on pull requests. Preventing the secret from reaching production is cheaper than responding after it does.
