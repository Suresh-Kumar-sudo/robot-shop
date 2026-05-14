# Robot Shop — Cart Service CI Pipeline

This pipeline automates the full CI lifecycle for the `cart` service: secret scanning, container building, security scanning, code quality analysis, artifact publishing, and GitOps-based deployment updates.

---

## Pipeline Overview

```
Checkout → Secrets Scan → Docker Build → Trivy Scan → SAST Scan → Quality Gate → OWASP Dependency Check → Docker Push → Update Helm Repo
```

---

## Environment Variables

| Variable | Value | Description |
|---|---|---|
| `GIT_REPO` | `https://github.com/Suresh-Kumar-sudo/robot-shop.git` | Source repository |
| `IMAGE_NAME` | `sureshkumarnadar1/rs-cart` | DockerHub image name |
| `APP_VERSION` | `v1.2.0` | Base application version |
| `IMAGE_TAG` | `<APP_VERSION>-<SHORT_SHA>` | Final image tag (generated at runtime) |

> `IMAGE_TAG` is dynamically generated as `APP_VERSION` + short Git commit SHA (e.g. `v1.2.0-a1b2c3d`)

---

## Stages

### 1. Checkout

- Cleans the workspace
- Clones the `main` branch
- Generates a dynamic `IMAGE_TAG` from the app version and short Git SHA

---

### 2. Secrets Scan

**Tool:** [Gitleaks](https://github.com/zricethezav/gitleaks)

Scans the repository for hardcoded secrets, credentials, and API keys before any build happens.

```
Agent: zricethezav/gitleaks:latest (Docker)
Report: reports/gitleaks/gitleaks-report.json
Archived: ✅
```

---

### 3. Docker Build

Builds the Docker image for the `cart` service from `cart/Dockerfile`.

```
Tag format: sureshkumarnadar1/rs-cart:<APP_VERSION>-<SHORT_SHA>
```

---

### 4. Trivy Scan

**Tool:** [Trivy](https://github.com/aquasecurity/trivy) (client → server mode via `CI-network`)

Runs multiple scans against the built image and generates the following reports:

| Report | Format | Scope |
|---|---|---|
| OS vulnerability report | HTML | OS packages, HIGH + CRITICAL |
| Library vulnerability report | HTML | App dependencies, HIGH + CRITICAL |
| SPDX SBOM | JSON | Full software bill of materials |
| CycloneDX SBOM | JSON | Full software bill of materials |
| Security gate | CLI | CRITICAL only — pipeline break condition |

```
Server:   http://trivy-server:4954
Reports:  reports/trivy/
Archived: ✅
```

---

### 5. SAST Scan

**Tool:** [SonarQube Scanner](https://docs.sonarqube.org/latest/)

Performs static application security testing and code quality analysis against the `cart` service source.

```
Project Key:  robot-shop-cart
Project Name: robot-shop-cart
Server:       http://sonarqube:9000 (via CI-network)
Env:          withSonarQubeEnv('sonarqube')
```

---

### 6. Quality Gate

Waits for SonarQube to evaluate the quality gate asynchronously via webhook callback.

```
Timeout:         5 minutes
On failure:      Pipeline aborted
Webhook required: http://jenkins:8080/sonarqube-webhook/
```

> Quality gate result is pushed from SonarQube to Jenkins via the configured webhook. See [CI Infrastructure Setup](../infra/README.md) for webhook configuration.

---

### 7. OWASP Dependency Check

**Tool:** [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)

Performs Software Composition Analysis (SCA) on the `cart` service — identifies known CVEs in third-party dependencies.

```
NVD Database: Pre-cached volume (dependency-check-data) — no re-download
Project:      robot-shop-cart
Scan target:  cart/
Formats:      HTML, JSON
Reports:      reports/dependency-check/
Archived:     ✅
```

> Uses `--noupdate` flag to leverage the pre-cached NVD database. See [CI Infrastructure Setup](../infra/README.md) for cache setup.

---

### 8. Docker Push

Authenticates to DockerHub and pushes the built and scanned image.

```
Credential ID: DOCKERHUB_TOKEN
Registry:      hub.docker.com
Image:         sureshkumarnadar1/rs-cart:<IMAGE_TAG>
```

---

### 9. Update Helm Repo

Updates the Helm values file in the GitOps repo with the new image tag, triggering ArgoCD to reconcile the deployment.

```
Repo:          https://github.com/Suresh-Kumar-sudo/robot-shop-helm.git
File:          robot-shop-eks/values.yaml
Field updated: cart.image.tag
Credential ID: GITHUB_TOKEN
Commit:        "Update cart image tag to <IMAGE_TAG>"
Tool:          yq (downloaded at runtime)
```

---

## Reports & Artifacts

All security reports are archived as Jenkins build artifacts.

| Stage | Artifact Location |
|---|---|
| Secrets Scan | `reports/gitleaks/gitleaks-report.json` |
| Trivy OS Scan | `reports/trivy/os-vulnerability-report.html` |
| Trivy Library Scan | `reports/trivy/library-vulnerability-report.html` |
| SPDX SBOM | `reports/trivy/spdx.json` |
| CycloneDX SBOM | `reports/trivy/cyclonedx.json` |
| Dependency Check | `reports/dependency-check/` |

---

## Required Jenkins Credentials

| Credential ID | Kind | Used In |
|---|---|---|
| `DOCKERHUB_TOKEN` | Username + Password | Docker Push |
| `GITHUB_TOKEN` | Username + Password | Update Helm Repo |
| `SONAR_TOKEN` | Secret text | SonarQube (via server config) |

---

## Prerequisites

- CI infrastructure running (Jenkins, SonarQube, Trivy Server)
- SonarQube webhook configured pointing to `http://jenkins:8080/sonarqube-webhook/`
- OWASP NVD cache pre-populated in `dependency-check-data` volume
- Docker socket mounted on Jenkins container

> See [CI Infrastructure Setup](../infra/README.md) for full infrastructure setup instructions.