---
date: 2026-06-11
authors:
  - alismx
categories:
  - DevSecOps
tags:
  - checkov
  - iac
  - terraform
  - kubernetes
  - cloudformation
  - security
  - ci
pin: true
---

# DevSecOps: Scan Infrastructure Code for Misconfigurations with Checkov

Infrastructure code often passes syntax checks but contains security misconfigurations. Open buckets, unencrypted databases, overly permissive access policies. Checkov scans IaC files across 20+ formats for these patterns before you deploy.

**TL;DR**: Install checkov, scan your IaC directory, configure your repository, and add it to your CI pipeline to catch infrastructure misconfigurations before they reach production.

---

## Install checkov

```bash
pip install checkov
```

Verify:

```bash
checkov --version
```

---

## Scan a project

Basic usage: scan an entire directory recursively:

```bash
checkov -d /path/to/iac
```

Scan a single file:

```bash
checkov -f /path/to/main.tf
```

Checkov groups findings by resource type. AWS findings use `CKV_AWS_XX` IDs, Azure uses `CKV_AZURE_XX`, Kubernetes uses `CKV_K8S_XX`, and so on.

---

## Understand the checks

Checkov has 750+ built-in policies across 20+ IaC types. It parses HCL-based files (.tf, .tofu, .hcl) with the same rules since both Terraform and OpenTofu use HCL.

### IaC formats

| IaC Type | Format | Suppression |
|---|---|---|
| [Terraform / OpenTofu](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Terraform.md) | .tf, .tofu, .hcl, .tfvars | `# checkov:skip=CKV_AWS_21: reason` |
| [Kubernetes](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Kubernetes.md) | .yaml, .json | `checkov.io/skip1: CKV_K8S_20=reason` |
| [CloudFormation](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Cloudformation.md) | .json, .yaml, .yml, .template | `# checkov:skip=CKV_AWS_1: reason` |
| [Dockerfile](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Dockerfile.md) | Named files (Dockerfile, Dockerfile.*, dockerfile) | `#checkov:skip=CKV_DOCKER_1: reason` |
| [Bicep](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Bicep.md) | .bicep | `#checkov:skip=CKV_AZURE_1: reason` |
| [ARM Templates](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Azure%20ARM%20templates.md) | .json | CLI only (`--skip-check`) |
| [Ansible](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Ansible.md) | .yaml, .yml, .json | `#checkov:skip=CKV_ANSIBLE_1: reason` |
| [Argo Workflows](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Argo%20Workflows.md) | .yaml | `checkov.io/skip1: CKV_ARGO_1=reason` |
| [Serverless Framework](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Serverless%20Framework.md) | .yml, .yaml | `#checkov:skip=CKV_AWS_1: reason` |
| [AWS SAM](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/AWS%20SAM.md) | .yaml, .yml | `#checkov:skip=CKV_AWS_1: reason` |
| [OpenAPI](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/OpenAPI.md) | .yaml, .json | `// checkov:skip=CKV_OPENAPI_1: reason` |
| [Azure Pipelines](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Azure%20Pipelines.md) | .yml, .yaml | `#checkov:skip=CKV_AZUREPIPELINES_1: reason` |

### Auto-detected

| IaC Type | Format | Suppression |
|---|---|---|
| [Helm](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Helm.md) | Auto-detected by Chart.yaml presence | Via `#checkov:skip` on templates |
| [Kustomize](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Kustomize.md) | Auto-detected by kustomization.yaml presence | Via `#checkov:skip` on templates |
| [AWS CDK](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/CDK.md) | Synthesized from .ts, .js, .py (cdk synth to JSON first) | CDK construct metadata |

### VCS configuration

Checkov evaluates organization and repository settings from the respective APIs, not from code files. You need to provide a personal access token for each platform.

| IaC Type | Format | Suppression |
|---|---|---|
| [GitHub Actions](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Github.md) | .yml, .yaml | `#checkov:skip=CKV_GITHUB_1: reason` |
| [GitHub Configuration](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Github.md) | .json (fetched from GitHub API) | CLI only (`--skip-check`) |
| [GitLab CI](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Gitlab.md) | .yml, .yaml | `#checkov:skip=CKV_GITLABCI_1: reason` |
| [GitLab Configuration](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Gitlab.md) | .json (fetched from GitLab API) | CLI only (`--skip-check`) |
| [Bitbucket Pipelines](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Bitbucket.md) | .yml | `#checkov:skip=CKV_CIRCLECIPIPELINES_1: reason` |
| [Bitbucket Configuration](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Bitbucket.md) | .json (fetched from Bitbucket API) | CLI only (`--skip-check`) |

See the [GitHub](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Github.md), [GitLab](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Gitlab.md), and [Bitbucket](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Bitbucket.md) scan examples for required environment variables.

### Other

| IaC Type | Format | Suppression |
|---|---|---|
| [SCA](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Sca.md) | .lock, .json, .txt, .xml, .toml (varies by language) | File-level comment in lock file |
| [Git History](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Git%20History.md) | (repository commit history, not a file format) | CLI only (`--skip-check`) |
| [Terraform Plan](https://github.com/bridgecrewio/checkov/blob/main/docs/7.Scan%20Examples/Terraform%20Plan%20Scanning.md) | .tfplan (Terraform plan output, not source) | CLI only (`--skip-check`) |

Checkov's full policy index is auto-generated from the source and available at [Policy Index](https://www.checkov.io/5.Policy%20Index/all.html).

**Notes on auto-detected types**: Helm and Kustomize auto-detect config file presence, then template out to Kubernetes manifests for scanning. AWS CDK requires running `cdk synth` to generate a CloudFormation JSON template, which checkov then scans.

---

## Configuration files

You can configure checkov with a YAML file in your repo:

```yaml
# .checkov.yaml
check:
  skip-check:
    - CKV_AWS_21
    - CKV_AWS_34
compact: true
```

The `skip-check` list matches `--skip-check` on the command line. The `compact` flag collapses findings and only displays failures.

---

## Usage and output formats

Checkov supports several output formats. The default is text. For CI integration, use JSON, SARIF, or JUnit XML.

```bash
# Default: human-readable text
checkov -d .

# JSON (parseable in CI)
checkov -d . -o json -o findings.json

# SARIF (GitHub Code Scanning, GitLab SAST)
checkov -d . -o sarif -o findings.sarif

# JUnit XML (for CI systems)
checkov -d . -o junitxml -o findings.xml
```

SARIF integrates with GitHub Code Scanning and GitLab SAST which renders findings as in-line annotations on the source files.

---

## Add to CI/CD

The `checkov-action` with the `github_failed_only` output format reports findings as inline checks on the pull request.

```yaml
# .github/workflows/checkov.yml
name: Checkov
on: [pull_request]
jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          output_format: github_failed_only
```

---

## Takeaways

- Use `--skip-check` or `.checkov.yaml` for known misconfigurations. Don't skip checks blindly.
- Use `checkov:skip` comments to suppress individual findings in the code. Explain why.
- SARIF output integrates with GitHub Code Scanning for inline annotations on source files.
- Run checkov in CI on pull requests. Preventing a misconfiguration from reaching production is cheaper than fixing it later.
