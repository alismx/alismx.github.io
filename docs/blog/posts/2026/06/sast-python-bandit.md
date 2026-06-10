---
date: 2026-06-10
authors:
  - alismx
categories:
  - DevSecOps
tags:
  - bandit
  - sast
  - python
  - github
  - git
  - security
  - ci
pin: true
---

# DevSecOps: Run SAST on Python Code with Bandit

Insecure Python patterns (`eval()`, `os.system()`, hardcoded credentials) end up in production repos. Bandit scans Python code to find them.

**TL;DR**: Install Bandit, configure your repository, run a scan, and add it to your CI pipeline to catch security issues before they reach production.

---

## Install Bandit

```bash
pip install bandit
```

The apt package is outdated at 1.6.2. Use pip for the current version (1.9.4).

Verify:

```bash
bandit --version
```

### Extras

Bandit has optional extras for additional functionality:

```bash
# SARIF output formatter (for GitHub and other tools)
pip install bandit[sarif]

# Baseline comparison (ignore known-vulnerabilities)
pip install bandit[baseline]

# TOML configuration support
pip install bandit[toml]
```

These aren't needed to start. They become useful once you're past the basic scanning phase.

---

## Understand the rule IDs

Bandit uses alphanumeric rule IDs. The letter prefix groups them by category:

| Prefix | Category          | What it checks                                                 |
| ------ | ----------------- | -------------------------------------------------------------- |
| B1xx   | Assertion         | `assert` statements in production code                         |
| B2xx   | Exec/eval         | `eval()`, `exec()`, `pickle` usage                             |
| B3xx   | Blacklist calls   | Dangerous function calls (shell injection, SSL bypass)         |
| B4xx   | Blacklist imports | Insecure imports (pickle, xml, subprocess with shell=True)     |
| B5xx   | Crypto            | Weak crypto algorithms (MD5, DES, RC4)                         |
| B6xx   | OS                | `os.system()`, `subprocess` with shell=True, command injection |
| B7xx   | Network           | SSL/TLS misconfiguration, insecure protocols                   |

The most commonly skipped rules are B101 (assert statements) and B602 (os.system). Whether you skip them depends on your code. If you're running Bandit on test files or production code with assertions, they'll generate noise.

## Scan a project

Basic usage:

```bash
# Single file
bandit single_file.py

# Entire directory
bandit -r /path/to/project

# Limit context lines shown in output
bandit examples/*.py -n 3 --severity-level high
```

Skip specific rules by ID:

```bash
bandit -r . --skip B101,B602
```

Skip specific directories:

```bash
bandit -r . --exclude ./tests/
```

Scan from stdin:

```bash
cat examples/imports.py | bandit -
```

You can filter by severity or confidence level:

```bash
# Medium security level
bandit -r . -ll

# High security + High confidence
bandit -r . -lll -iii

# By severity name
bandit -r . --severity-level high

# By confidence name
bandit -r . --confidence-level high
```

---
## Output formats

Bandit supports several output formats. The default is plain text. For CI integration, use JSON or SARIF.

```bash
# Default: human-readable text
bandit -r .

# JSON (parseable in CI)
bandit -r . -f json -o findings.json

# SARIF (for GitHub and other tools)
bandit -r . -f sarif -o findings.sarif
```

---

## Configuration files

You can configure Bandit with a YAML or TOML file, or an INI file called `.bandit`.

### YAML (recommended, works with all extras)

```yaml
# bandit.yaml
skips:
  - B101
  - B602

exclude_dirs:
  - tests
  - migrations
```

```bash
bandit -c bandit.yaml -r .
```

### TOML

```toml
# pyproject.toml
[tool.bandit]
exclude_dirs = ["tests", "migrations"]
skips = ["B101", "B602"]
```

```bash
bandit -c pyproject.toml -r .
```

### Config generator

Bandit ships `bandit-config-generator` which generates a full configuration file with all detected plugins:

```bash
bandit-config-generator -s B101,B602 -o bandit.yaml
```

This is useful for understanding what plugins are available and their default settings. Edit the output down to what your project actually needs.

---

## Suppress individual findings

If a line triggers a finding that you've reviewed and determined is safe, add `# nosec`:

```python
# This hash is for unique IDs only, not security
the_hash = md5(data).hexdigest() # nosec
```

The whole line gets suppressed. If you only want to suppress specific checks on that line, name them:

```python
self.process = subprocess.Popen('/bin/ls *', shell=True)  # nosec B602, B607
```

You can use full test names instead of IDs:

```python
assert yaml.load("{}") == []  # nosec assert_used
```

Always add a comment explaining *why* you're suppressing the finding. Without context, future reviewers won't know if the suppression is intentional or lazy.

---

## Baseline: ignore known vulnerabilities

If you have findings that are known and not currently actionable (e.g., a cleartext password in a unit test), generate a baseline and pass it to subsequent scans:

```bash
# Generate baseline
bandit -f json -o baseline.json -r .

# Subsequent scans ignore baseline findings
bandit -b baseline.json -f json -r .
```

This is cleaner than suppressing individual lines when you have dozens of known-vulnerabilities.

---

## Add to local git hooks

Add Bandit as a pre-commit git hook:

```bash
#!/bin/sh
# .git/hooks/pre-commit

pip install -q bandit
bandit -r . -ll -f json -o /dev/null
```

Make it executable:

```bash
chmod +x .git/hooks/pre-commit
```

---

## Add to CI/CD

```yaml
# .github/workflows/bandit.yml
name: Bandit SAST
on: [pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install Bandit
        run: pip install bandit[sarif]
      - name: Run Bandit
        run: bandit -r . -f sarif -o bandit-results.sarif
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: bandit-results.sarif
```

The SARIF output format integrates directly with GitHub's code scanning UI, which is nicer than trying to parse plain-text results.

---

## Takeaways

- Bandit catches the common things: `eval()`, `os.system()`, weak crypto, hardcoded secrets
- Use `--severity-level` and `--confidence-level` to control finding noise
- `# nosec` suppresses a line. Name the specific checks if you only want to suppress certain ones
- Baseline files are cleaner than suppressing individual findings for known-vulnerabilities
- SARIF output integrates with GitHub code scanning
- Local git hooks catch findings before they reach CI
- The apt version is stale. Use pip for the current rule set.
- Always explain *why* when you skip a rule or add a `# nosec` comment
