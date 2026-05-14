# Robot Shop — Three Tier Microservice Application

A containerised microservice application built as a personal take on [instana/robot-shop](https://github.com/instana/robot-shop), extended with custom Dockerfiles, a full CI pipeline, and Helm-based deployment for both Minikube and AWS EKS.

---

## What's Different From The Original

- Removed all Instana service dependencies from Dockerfiles and Kubernetes manifests
- Custom Dockerfiles across services
- Full CI pipeline with security scanning (see [CI Pipeline](#ci-pipeline))
- Helm charts for both local (Minikube) and production (AWS EKS) deployments

---

## Repositories

| Repo | Description |
|---|---|
| [`robot-shop`](https://github.com/Suresh-Kumar-sudo/robot-shop) | Application source code & Dockerfiles |
| [`robot-shop-helm`](https://github.com/Suresh-Kumar-sudo/robot-shop-helm) | Helm charts — Minikube & EKS deployment |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | AngularJS 1.x, Nginx |
| Services | NodeJS (Express), Java (Spring Boot), Python (Flask), Golang, PHP (Apache) |
| Databases | MongoDB, MySQL , Redis |
| Messaging | RabbitMQ |
| Orchestration | Kubernetes (Minikube / AWS EKS) |
| Packaging | Helm |

---

## Run Locally (Docker Compose)

Pull all images from Docker Hub:

```bash
docker-compose pull
```

Start the application:

```bash
docker-compose up
```

Start with load generation:

```bash
docker-compose -f docker-compose.yaml -f docker-compose-load.yaml up
```

Access the store at [http://localhost:8080](http://localhost:8080)

---

## Deploy on Minikube

```bash
# Start Minikube
minikube start

# Deploy via Helm
helm install robot-shop ./robot-shop-eks -n robot-shop --create-namespace
```

Get the service URL:

```bash
minikube ip
kubectl get svc web -n robot-shop
```

### Local DNS (optional)

Map the Minikube IP to a local hostname for easier access:

```bash
# Add to /etc/hosts
sudo echo "<minikube-ip>  robot-shop.local" >> /etc/hosts
```

Then access at [http://robot-shop.local](http://robot-shop.local)

---

## Deploy on AWS EKS

```bash
# Point kubeconfig to your EKS cluster
aws eks update-kubeconfig --name <cluster-name> --region <region>

# Deploy via Helm
helm install robot-shop ./robot-shop-eks -n robot-shop --create-namespace
```

Get the Load Balancer URL from the Ingress:

```bash
kubectl get ingress -n robot-shop
```

Access the store via the Load Balancer DNS hostname shown in the `ADDRESS` field.

> Helm chart values for EKS are configured in `robot-shop-helm/robot-shop-eks/values.yaml`

---

## CI Pipeline

Each service has an automated CI pipeline running on a self-hosted Jenkins instance.

**Pipeline stages:**

```
Checkout → Secrets Scan → Docker Build → Trivy Scan → SAST → Quality Gate → OWASP Dependency Check → Docker Push → Update Helm Repo
```

On a successful build, the pipeline automatically updates the image tag in `robot-shop-helm/robot-shop-eks/values.yaml`, triggering a GitOps-style deployment.

See [`ci/README.md`](./ci/README.md) for full pipeline documentation.
See [`infra/README.md`](./infra/README.md) for CI infrastructure setup.

---

## Load Generation

A load generation utility is available in the `load-gen/` directory, built with Python and [Locust](https://locust.io).

```bash
# Build the load gen image
cd load-gen
./build.sh

# Run load generation
./load-gen.sh
```

> Load generation is not started automatically with the application. For End-User Monitoring, navigate through the store manually in the browser.

See [`load-gen/README.md`](./load-gen/README.md) for full details.

---

## Project Structure

```
robot-shop/
├── cart/               # NodeJS cart service
├── catalogue/          # NodeJS catalogue service
├── dispatch/           # Golang dispatch service
├── payment/            # Python payment service
├── ratings/            # PHP ratings service
├── shipping/           # Java shipping service
├── user/               # NodeJS user service
├── web/                # Nginx frontend
├── load-gen/           # Locust load generation
├── ci/                 # Jenkins pipeline & CI docs
├── infra/              # CI infrastructure (Docker Compose)
└── docker-compose.yaml
```

---

## Future Improvements

| Feature | Description |
|---|---|
| Jenkins Configuration as Code (JCasC) | Define Jenkins config in YAML — eliminates manual UI setup and makes Jenkins reproducible |
| `plugins.txt` | Declaratively manage Jenkins plugins as code for consistent, version-pinned installations |
| Shared Jenkins Libraries | Centralize reusable pipeline logic across multiple repos to reduce duplication |
| ArgoCD Image Updater | Automatically update Kubernetes manifests when new container images are pushed |
| DAST (OWASP ZAP) | Dynamic application security testing against running services as a pipeline stage |
| Kubernetes Policy Enforcement | Enforce cluster-wide security and compliance policies (e.g. OPA Gatekeeper / Kyverno) |