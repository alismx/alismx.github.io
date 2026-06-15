---
date: 2026-06-13
authors:
  - alismx
categories:
  - DevSecOps
tags:
  - trivy
  - vulnerability
  - container
  - sca
  - compliance
  - sbom
  - docker
  - kubernetes
  - security
  - oci
  - podman
  - containerd
  - sarif
  - dockerfile
  - misconfig
  - secret
  - license
  - ci
  - github
  - tar
  - registry
  - offline
  - cache
  - config
  - cyclonedx
  - spdx
pin: true
---

# DevSecOps: Scan Container Images with Trivy

Containers images ship software. They also ship vulnerabilities. Outdated base images, exposed secrets, misconfigured image instructions. Trivy scans container images for CVEs, misconfigurations, secrets, and license issues in one command.

**TL;DR**: Install Trivy, scan your container images with `trivy image`, enable additional scanners, and add it to your CI pipeline to catch vulnerabilities before deployment.

---

## Install Trivy

```bash
# Ubuntu/Debian
sudo apt install apt-transport-https ca-certificates curl gnupg
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy
```

Verify:

```bash
trivy --version
```

---

## Scan a container image

Basic usage: scan a public image for vulnerabilities and secrets:

```bash
trivy image python:3.4-alpine
```

Vulnerability and secret scanning are both enabled by default. The output groups findings by OS package type, then gets into specific packages:

```
Report Summary

┌────────────────────────────────────────────────────────────────────────────┬────────────┬─────────────────┬─────────┐
│                                   Target                                   │    Type    │ Vulnerabilities │ Secrets │
├────────────────────────────────────────────────────────────────────────────┼────────────┼─────────────────┼─────────┤
│ python:3.4-alpine (alpine 3.9.2)                                           │   alpine   │       37        │    -    │
├────────────────────────────────────────────────────────────────────────────┼────────────┼─────────────────┼─────────┤
...

python:3.4-alpine (alpine 3.9.2)

Total: 37 (UNKNOWN: 0, LOW: 4, MEDIUM: 16, HIGH: 13, CRITICAL: 4)

┌──────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬──────────────────────────────────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                            Title                             │
├──────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ expat        │ CVE-2018-20843 │ HIGH     │ fixed  │ 2.2.6-r0          │ 2.2.7-r0      │ expat: large number of colons in input makes parser consume  │
│              │                │          │        │                   │               │ high amount...                                               │
│              │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2018-20843                   │
...
```

To scan only vulnerabilities (disables the default secret scanner):

```bash
trivy image --scanners vuln python:3.4-alpine
```

To enable only misconfiguration or license scanning:

```bash
trivy image --scanners misconfig myapp:latest
trivy image --scanners license myapp:latest
```

---

## Scan container image metadata

Container images have configuration metadata, the container image instructions that built them. Trivy can scan this configuration for misconfigurations and secrets:

```bash
# Dockerfile-style misconfigurations (USER, HEALTHCHECK, ADD vs COPY)
trivy image --image-config-scanners misconfig myapp:latest

# Secrets in environment variables (credential leaks in config)
trivy image --image-config-scanners secret myapp:latest
```

These are disabled by default. The misconfiguration scanner converts the image config into Dockerfile format and runs Dockerfile checks against it. The secret scanner looks for credentials in environment variables stored in the config JSON.

---

## Scan an image tar file

Trivy can scan images saved as tar archives. Useful for offline or CI environments:

```bash
docker pull ruby:3.1-alpine3.15
docker save ruby:3.1-alpine3.15 -o ruby-3.1.tar
trivy image --input ruby-3.1.tar
```

---

## Scan by architecture

By default, Trivy scans on the host's architecture. Cross-architecture images need the `--platform` flag:

```bash
trivy image --platform=linux/arm alpine:3.16.1
```

---

## SBOM generation and vulnerability scanning

Trivy can generate Software Bill of Materials (SBOM) for container images:

```bash
# Generate SBOM in CycloneDX format
trivy image --format cyclonedx -o sbom.json python:3.4-alpine

# Generate SBOM in SPDX format
trivy image --format spdx-json -o sbom.json python:3.4-alpine
```

---

## Compliance

The `docker-cis-1.6.0` check evaluates container image configurations against the CIS Docker Benchmark:

```bash
trivy image --compliance docker-cis-1.6.0 myapp:latest
```

---

## Authentication

For private registries, authenticate with:

```bash
trivy registry login --username myuser --password mypassword registry.example.com
```

Then scan as normal. Trivy reads the registry credentials before pulling the image.

---

## Filter findings

Trivy lets you narrow scans by severity:

```bash
# Only HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL alpine:3.18

# Only HIGH severity
trivy image --severity HIGH alpine:3.18
```

---

## Add to CI/CD

```yaml
# .github/workflows/trivy.yml
name: Trivy
on: [pull_request]
jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: myapp:latest
          format: table
          exit-code: 1
          severity: HIGH,CRITICAL
```

`exit-code: 1` causes the job to fail on HIGH or CRITICAL findings. Use `format: sarif` and add the `upload-sarif` action for inline annotations.

---

## Understand what Trivy scans

Trivy checks four categories against files inside container images:

| Category     | What it finds                          | Enabled by default |
|-------------|----------------------------------------|-------------------|
| Vulnerabilities | CVEs in OS packages and language dependencies | Yes |
| Misconfigurations | IaC files inside the image (Kubernetes, Terraform) | No |
| Secrets | Hardcoded credentials, tokens, API keys | Yes |
| Licenses    | Open source license compliance        | No |

---

## Takeaways

- Trivy scans for vulnerabilities and secrets by default. Use `--scanners` to enable misconfiguration and license scanning.
- `--image-config-scanners` scans the image's configuration metadata, not the filesystem inside it.
- `--input` lets you scan tar files and OCI layout directories instead of live registries. Useful for offline CI.
- `--severity` and `--exit-code` control CI gating.
- Use `--platform` for multi-arch images. Trivy only scans the host architecture without it.
