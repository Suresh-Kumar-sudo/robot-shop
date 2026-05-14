# CI Infrastructure Setup

A reusable, containerized CI platform with Jenkins, SonarQube, Trivy, and OWASP Dependency Check.

---

## Overview

This setup creates a complete, production-like CI infrastructure using Docker Compose.

### Components

| Service | Purpose |
|---|---|
| Jenkins | CI orchestration |
| SonarQube | Centralized SAST / code quality |
| Trivy Server | Shared vulnerability DB & image scanning |
| OWASP Dependency Check | SCA with pre-cached NVD database |
| CI-network | Shared Docker network for service discovery |
| Persistent volumes | Data retention across restarts |

---

## Step 1 — Build Custom Jenkins Image

A custom Jenkins image pre-installs the tools needed inside the CI environment.

**Tools installed:** Docker CLI, curl, git, wget, yq

### `Dockerfile`

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && apt-get install -y \
    docker.io \
    curl \
    git \
    wget

# Install yq
RUN wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
    -O /usr/local/bin/yq && \
    chmod +x /usr/local/bin/yq

USER jenkins
```

```bash
docker build -t jenkins .
```

---

## Step 2 — Docker Compose Infrastructure

### `docker-compose.yml`

```yaml
version: '3'

networks:
  CI-network:

volumes:
  jenkins_home:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
  dependency-check-data:

services:

  jenkins:
    build: .
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    networks:
      - CI-network
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock

  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    ports:
      - "9000:9000"
    networks:
      - CI-network
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions

  trivy-server:
    image: aquasec/trivy:latest
    container_name: trivy-server
    command: server
    ports:
      - "4954:4954"
    networks:
      - CI-network
```

### Start the infrastructure

```bash
docker compose -f docker-compose.yml up -d
```

All containers share `CI-network`, enabling internal service discovery:

- `http://sonarqube:9000`
- `http://trivy-server:4954`
- `http://jenkins:8080`

---

## Step 3 — Pre-cache OWASP NVD Database

Dependency Check downloads 350k+ NVD records. Pre-cache it once into a shared volume to avoid repeated downloads on every scan.

```bash
docker run --rm \
  -v dependency-check-data:/usr/share/dependency-check/data \
  owasp/dependency-check:latest \
  --updateonly
```

---

## Step 4 — Docker Socket Permission Fix

Jenkins needs access to the host Docker daemon (Docker-outside-of-Docker pattern).

```
Jenkins Container → Docker CLI → Docker Socket → Host Docker Daemon
```

```bash
sudo chmod 666 /var/run/docker.sock
```

---

## Step 5 — Jenkins Initial Setup

### Get the initial admin password

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Install required plugins

| Plugin | Purpose |
|---|---|
| Docker Pipeline | Run pipeline steps inside Docker containers |
| Docker | Docker build/push integration |
| SonarQube Scanner | Trigger SonarQube analysis from pipelines |

---

## Step 6 — Configure Credentials in Jenkins

Go to **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

### GitHub

- **Kind:** Username with password
- **Username:** GitHub username
- **Password:** Fine-grained personal access token (minimum required permissions, selected repos only)

### DockerHub

- **Kind:** Username with password
- **Username:** DockerHub username
- **Password:** DockerHub access token

---

## Step 7 — SonarQube Setup

### Default credentials

Open `http://localhost:9000`

```
Username: admin
Password: admin
```

### Generate a token

**My Account → Security → Generate Tokens**

Use a descriptive name like `jenkins-token`. **Copy the token immediately.**

### Add token to Jenkins

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Field | Value |
|---|---|
| Kind | Secret text |
| Secret | `<your sonar token>` |
| ID | `SONAR_TOKEN` |

### Configure SonarQube server in Jenkins

**Manage Jenkins → System → SonarQube Servers**

| Field | Value |
|---|---|
| Name | `sonarqube` |
| URL | `http://sonarqube:9000` |
| Token | `SONAR_TOKEN` |

> **Important:** The name `sonarqube` must match the value used in `withSonarQubeEnv('sonarqube')` in your Jenkinsfile.

### Configure SonarQube webhook

**Administration → Configuration → Webhooks → Create**

```
http://jenkins:8080/sonarqube-webhook/
```

> **Important:** The trailing slash is required. This enables asynchronous quality gate callbacks and `waitForQualityGate()` support in pipelines.

---

## Appendix

### Why Trivy Server mode?

Without server mode, every pipeline run re-downloads the vulnerability database, slowing builds significantly. Trivy Server maintains a shared, cached DB:

```
Trivy Client → Trivy Server → Shared Vulnerability DB
```

### Why SonarQube Server?

Running SonarQube as a persistent server (rather than local CLI scans) provides:
- Historical code analysis & trend tracking
- Centralized quality gates
- Security dashboards
- Maintainability metrics across all projects

### Reset Jenkins password (if forgotten)

```bash
# Enter the container
docker exec -it jenkins bash

# Create the init scripts directory
mkdir -p /var/jenkins_home/init.groovy.d

# Write a password reset script
cat > /var/jenkins_home/init.groovy.d/basic-security.groovy <<'EOF'
#!groovy
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.get()
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.createAccount("admin","admin")
instance.setSecurityRealm(hudsonRealm)

def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
instance.setAuthorizationStrategy(strategy)
instance.save()
EOF

# Restart Jenkins
docker restart jenkins
```

Login with `admin / admin`, then remove the reset script:

```bash
docker exec -it jenkins rm /var/jenkins_home/init.groovy.d/basic-security.groovy
```

---

## Future Improvements

### Planned Enhancements

| Feature | Description |
|---|---|
| Jenkins Configuration as Code (JCasC) | Define Jenkins config in YAML — eliminates manual UI setup and makes Jenkins reproducible |
| `plugins.txt` | Declaratively manage Jenkins plugins as code for consistent, version-pinned installations |
| Shared Jenkins Libraries | Centralize reusable pipeline logic across multiple repos to reduce duplication |
| ArgoCD Image Updater | Automatically update Kubernetes manifests when new container images are pushed |
| DAST (OWASP ZAP) | Dynamic application security testing against running services as a pipeline stage |
