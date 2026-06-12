---
date: 2026-06-11
authors:
  - alismx
categories:
  - DevSecOps
tags:
  - tfenv
  - terraform
  - opentofu
  - version
  - github
  - ci
  - git
  - pin
  - shell
  - semver
pin: true
---

# DevSecOps: Manage Terraform Versions with tfenv

State file incompatibility between Terraform versions breaks your pipelines. tfenv manages multiple versions so you can pin a specific version, switch between them, and keep CI aligned with your local environment.

**TL;DR**: Install tfenv, switch to the version your project needs, and pin it in CI to prevent version drift between local and remote environments.

---

## Install tfenv

```bash
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
```

Add it to your shell profile:

```bash
# ~/.bashrc, ~/.zshrc, etc.
export PATH="$HOME/.tfenv/bin:$PATH"
```

Verify:

```bash
tfenv --version
```

---

## Check available versions

```bash
# All available versions
tfenv list-remote

# Filter for a specific major version (1)
tfenv list-remote | grep "^1\."
```

Terraform uses semantic versioning: breaking changes only happen between major versions. Minor and patch updates are compatible with existing state.

---

## Switch versions

```bash
# Install a specific version
tfenv install 1.15.0

# List installed versions
tfenv list

# Switch to a version 
tfenv use 1.15.0

# Write current version to .terraform-version
tfenv pin
```

This file is tracked in git. Anyone who runs `tfenv use` (or `terraform` with tfenv installed) will use the pinned version.

```bash
# Reads from .terraform-version by default
tfenv use
```

---

## Takeaways

- `.terraform-version` is tracked in git, so the whole team uses the same Terraform version. This is the source of truth.
- `tfenv use` reads from `.terraform-version` by default. `tfenv pin` writes to it. The primary workflow is both without arguments.
- Terraform uses semantic versioning: breaking changes only happen between major versions. Minor and patch updates are safe.
