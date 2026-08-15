# K8s Workshop - WordPress + MariaDB Migration

Migration of a WordPress + MariaDB application from `docker-compose` to Kubernetes, using Helm charts, with full monitoring via kube-prometheus-stack (Prometheus + Grafana).

## Table of Contents

- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation - Application](#installation---application)
- [Secret Management](#secret-management)
- [Installation - Monitoring](#installation---monitoring)
- [Accessing the Application and Dashboards](#accessing-the-application-and-dashboards)
- [Grafana Dashboard](#grafana-dashboard)
- [Key Architecture Decisions](#key-architecture-decisions)
- [Known Limitations](#known-limitations)

## Architecture

The project is built as an **Umbrella Helm Chart** - a top-level chart (`wordpress-app`) that contains two dependent sub-charts: `wordpress` and `mariadb`. Each sub-chart owns only its own resources, but they are connected via a Kubernetes Service (never a direct IP) and a shared Secret.

**Key principles:**

- **WordPress does not create its own Secret** - it reads directly from `mariadb-secret` (DRY - a single source of truth for the DB password, instead of two separate Secrets holding the same value).
- **MariaDB runs as a StatefulSet, not a Deployment** - it has a stable identity (`mariadb-0`) and a dedicated PVC that must "remember" who it belongs to, unlike WordPress, which is stateless and its replicas are freely interchangeable.
- **Monitoring is a completely separate Helm release** - not part of the umbrella chart. It's an external chart (`prometheus-community/kube-prometheus-stack`) installed in its own namespace (`monitoring`), connected to the rest of the cluster only via scraping (Prometheus pulls metrics, it does not receive pushes).
- **No `requirements.yaml`** - this project uses Helm 3, where sub-chart dependencies are declared directly in `Chart.yaml` (the `dependencies:` field) instead of the legacy Helm 2 `requirements.yaml`. The included `Chart.lock` file is Helm's modern dependency lock file (conceptually similar to `package-lock.json` in npm) - it is generated automatically by `helm dependency update` and pins the exact resolved versions.

## Project Structure

```
wordpress-app/
├── Chart.yaml                      # Umbrella chart, declares dependencies on 2 sub-charts
├── Chart.lock                      # Auto-generated dependency lock file (Helm 3)
├── charts/
│   ├── wordpress/
│   │   ├── Chart.yaml
│   │   ├── values.yaml             # No passwords
│   │   └── templates/
│   │       ├── deployment.yaml     # 2 replicas, imagePullSecrets, DB_PASSWORD from Secret
│   │       ├── service.yaml        # ClusterIP
│   │       └── ingress.yaml        # host: wordpress.local
│   └── mariadb/
│       ├── Chart.yaml
│       ├── values.yaml             # No passwords - only database/user names
│       ├── values-secret.yaml      # Not in git! See Secret Management
│       └── templates/
│           ├── statefulset.yaml    # Built-in PVC (volumeClaimTemplates)
│           ├── service.yaml        # headless
│           └── secret.yaml         # mariadb-secret
├── values-monitoring.yaml          # kube-prometheus-stack configuration
├── values-monitoring-secret.yaml   # Not in git! Grafana admin password
└── .gitignore
```

## Prerequisites

- EC2 (or another machine) with Docker + minikube installed
- kubectl, helm
- An AWS account with ECR repositories for the images (`roei-wordpress`, `roei-mariadb`)

## Installation - Application

1. Start minikube:

```bash
   minikube start --memory=6000mb --cpus=2
```

2. Log in to ECR and create an imagePullSecret (required because the EC2's IAM Role does not automatically propagate to pods running inside minikube):

```bash
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

```bash
   kubectl create secret docker-registry ecr-secret --docker-server=<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com --docker-username=AWS --docker-password=$(aws ecr get-login-password --region us-east-1) --namespace=default
```

3. Create the local secrets file (see Secret Management below) - `charts/mariadb/values-secret.yaml`.

4. Install:

```bash
   helm install wordpress-app . -f charts/mariadb/values-secret.yaml
```

   To update later (instead of install):

```bash
   helm upgrade wordpress-app . -f charts/mariadb/values-secret.yaml
```

## Secret Management

The project consistently separates **configuration** (in git, non-sensitive) from **secrets** (local only, in `.gitignore`):

| File | Content | In git? |
|---|---|---|
| `charts/mariadb/values.yaml` | database name, username | Yes |
| `charts/mariadb/values-secret.yaml` | root password, user password | No |
| `values-monitoring.yaml` | resources, retention | Yes |
| `values-monitoring-secret.yaml` | Grafana admin password | No |

Both secrets files must be created manually before installing. Example for `charts/mariadb/values-secret.yaml`:

```yaml
mariadb:
  auth:
    rootPassword: "<choose a password>"
    password: "<choose a password>"
```

Example for `values-monitoring-secret.yaml`:

```yaml
grafana:
  adminUser: admin
  adminPassword: "<choose a password>"
```

**Important:** values that look like numbers must be wrapped in quotes - otherwise YAML coerces them into a numeric type, which breaks the base64 encoding Helm performs on the Secret.

## Installation - Monitoring

kube-prometheus-stack is a completely separate chart (not part of the umbrella chart), installed in its own namespace:

1. Create the local secrets file `values-monitoring-secret.yaml` (see Secret Management above) if you haven't already.

2. Add the chart repository:

```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
```

3. Create the namespace and install:

```bash
   kubectl create namespace monitoring
```

```bash
   helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring -f values-monitoring.yaml -f values-monitoring-secret.yaml
```

`values-monitoring.yaml` disables monitoring of control-plane components (`kubeControllerManager`, `kubeScheduler`, `kubeEtcd`) that are not exposed by minikube by default, and limits resources (CPU/memory/storage) to fit the size of the machine.

## Accessing the Application and Dashboards

Since this runs on minikube (Docker driver) on a single EC2 instance, exposing services requires a few concurrent terminal sessions **SSH'd into that same instance** (not separate machines) - each running a long-lived foreground process:

| Terminal | Command | Purpose |
|---|---|---|
| 1 | (your normal shell) | Running kubectl/helm commands |
| 2 | `minikube tunnel` (no sudo) | Required by minikube to route traffic to LoadBalancer/Ingress-type resources |
| 3 | `kubectl port-forward svc/wordpress 8080:80 --address 0.0.0.0` | Exposes WordPress on port 8080 |
| 4 | `kubectl port-forward svc/monitoring-grafana -n monitoring --address 0.0.0.0 3000:80` | Exposes Grafana on port 3000 |

Each of these keeps running in the foreground - closing the terminal or losing the SSH connection kills the process and the corresponding service becomes unreachable until it's started again.

Once running: `http://<EC2-PUBLIC-IP>:8080` (WordPress) and `http://<EC2-PUBLIC-IP>:3000` (Grafana, user `admin` + the password from `values-monitoring-secret.yaml`).

Note: without an Elastic IP, the EC2's public IP changes on every stop/start - always verify the current IP with:

```bash
curl -s http://169.254.169.254/latest/meta-data/public-ipv4
```

## Grafana Dashboard

A dashboard named **"App Monitoring"** with 3 complementary panels:

| Panel | Type | Purpose |
|---|---|---|
| Wordpress & mariaDB uptime | Time series | How long each pod has been running continuously; a crash shows as a "sawtooth" - a drop to zero followed by a climb |
| Wordpress & mariaDB memory usage | Time series | Actual memory consumption, relative to the configured resource limits |
| Number of restarts | Stat | Cumulative restart count per pod, aggregated by pod name (`sum by (pod)`) even if the pod was recreated multiple times |

## Key Architecture Decisions

- **Umbrella chart** instead of two independent charts - allows a single `helm install` to deploy the whole application while keeping the components' code separated.
- **Secret separated from configuration** (values.yaml vs values-secret.yaml) everywhere sensitive data exists, including monitoring - not just the database.
- **DRY secrets** - WordPress reads from the existing `mariadb-secret` instead of creating a duplicate Secret with the same value.
- **Monitoring as a separate release** - requires no changes to the umbrella chart, and can be installed/removed without affecting the application itself.
- **Panel type chosen by purpose** - Time series for trends (uptime, memory), Stat for a single current value (restarts).

## Known Limitations

- Built for a workshop/learning context, not production. No TLS, no HPA, low Prometheus retention (6h).
- EC2 without an Elastic IP - the public IP address changes on every machine restart.
- ECR tokens / the imagePullSecret expire after roughly 12 hours and need to be refreshed manually.
