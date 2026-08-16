# K8s Workshop - WordPress + MariaDB Migration

WordPress + MariaDB, moved from `docker-compose` to Kubernetes with Helm. Also has monitoring set up with Prometheus and Grafana.

## Table of Contents

- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation - Application](#installation---application)
- [Secret Management](#secret-management)
- [Installation - Monitoring](#installation---monitoring)
- [Accessing the Application and Dashboards](#accessing-the-application-and-dashboards)
- [Grafana Dashboard](#grafana-dashboard)
- [More information](#more-information)

## Architecture

It's one umbrella Helm chart (`wordpress-app`) with two sub-charts inside: `wordpress` and `mariadb`. Each one only deals with its own stuff. They connect through a Kubernetes Service, not a hardcoded IP.

WordPress doesn't have its own password Secret - it just reads `mariadb-secret`, which already exists. Otherwise there'd be two places with the same password that could get out of sync.

MariaDB is a StatefulSet because it needs a fixed name and its own disk. WordPress is a normal Deployment since any replica can handle any request.

WordPress also has its own PersistentVolumeClaim, mounted at `/var/www/html`, so uploaded media, themes and plugins survive pod restarts - matching how the original docker-compose set this up. Both replicas share the same volume (see More information for the trade-off there).

Monitoring is a separate Helm install, not part of the umbrella chart. It's an existing chart (`kube-prometheus-stack`), in its own namespace, and it just reads metrics from outside - nothing pushes data into it.

## Project Structure

```
wordpress-app/
├── Chart.yaml                      # umbrella chart, points to the 2 sub-charts
├── Chart.lock                      # auto-generated, locks dependency versions
├── charts/
│   ├── wordpress/
│   │   ├── Chart.yaml
│   │   ├── values.yaml             # no passwords here
│   │   └── templates/
│   │       ├── deployment.yaml     # 2 replicas, mounts pvc.yaml at /var/www/html
│   │       ├── pvc.yaml            # persists uploads/themes/plugins
│   │       ├── service.yaml
│   │       └── ingress.yaml
│   └── mariadb/
│       ├── Chart.yaml
│       ├── values.yaml             # no passwords here either
│       ├── values-secret.yaml      # not in git! see Secret Management
│       └── templates/
│           ├── statefulset.yaml    # has its own PVC
│           ├── service.yaml
│           └── secret.yaml
├── values-monitoring.yaml          # config for kube-prometheus-stack
├── values-monitoring-secret.yaml   # not in git! Grafana admin password
└── .gitignore
```

## Prerequisites

- EC2 (or another machine) with Docker + minikube
- kubectl, helm
- AWS account with ECR repos for the images (`roei-wordpress`, `roei-mariadb`)

## Installation - Application

1. Start minikube:

   ```bash
   minikube start --memory=6000mb --cpus=2
   ```

2. Log into ECR and create an imagePullSecret (needed because the EC2's AWS permissions don't automatically reach pods inside minikube):

   ```bash
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
   ```

   ```bash
   kubectl create secret docker-registry ecr-secret --docker-server=<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com --docker-username=AWS --docker-password=$(aws ecr get-login-password --region us-east-1) --namespace=default
   ```

3. Create `charts/mariadb/values-secret.yaml` - see Secret Management below.

4. Install:

   ```bash
   helm install wordpress-app . -f charts/mariadb/values-secret.yaml
   ```

   To update later, use `upgrade` instead of `install`:

   ```bash
   helm upgrade wordpress-app . -f charts/mariadb/values-secret.yaml
   ```

## Secret Management

Passwords and normal config are kept in separate files. Config goes in git, passwords don't (they're in `.gitignore`).

| File | What's in it | In git? |
|---|---|---|
| `charts/mariadb/values.yaml` | database name, username | Yes |
| `charts/mariadb/values-secret.yaml` | passwords | No |
| `values-monitoring.yaml` | resource settings | Yes |
| `values-monitoring-secret.yaml` | Grafana admin password | No |

Both password files need to be created by hand before installing anything.

`charts/mariadb/values-secret.yaml`:

```yaml
mariadb:
  auth:
    rootPassword: "<pick a password>"
    password: "<pick a password>"
```

`values-monitoring-secret.yaml`:

```yaml
grafana:
  adminUser: admin
  adminPassword: "<pick a password>"
```

Note: if your password is only numbers, put it in quotes. Otherwise YAML reads it as a number and it breaks the encoding Helm does on the Secret.

## Installation - Monitoring

Separate chart, separate namespace, doesn't touch the umbrella chart above.

1. Create `values-monitoring-secret.yaml` first if you haven't (see Secret Management above).

2. Add the chart repo:

   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

3. Install:

   ```bash
   kubectl create namespace monitoring
   ```

   ```bash
   helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring -f values-monitoring.yaml -f values-monitoring-secret.yaml
   ```

`values-monitoring.yaml` turns off monitoring for control-plane pieces minikube doesn't expose by default (`kubeControllerManager`, `kubeScheduler`, `kubeEtcd`), and lowers resource limits to fit a small machine.

## Accessing the Application and Dashboards

Everything runs on one EC2 with minikube. SSH into that machine and run:

```bash
kubectl port-forward svc/wordpress 8080:80 --address 0.0.0.0 &
kubectl port-forward svc/monitoring-grafana -n monitoring --address 0.0.0.0 3000:80 &
```

Then: `http://<EC2-PUBLIC-IP>:8080` for WordPress, `http://<EC2-PUBLIC-IP>:3000` for Grafana (`admin` + the password from `values-monitoring-secret.yaml`).

## Grafana Dashboard

Dashboard called **"App Monitoring"**, 3 panels:

| Panel | Type | What it shows |
|---|---|---|
| Wordpress & mariaDB uptime | Time series | How long each pod has been running without restarting. A crash shows as a drop to zero and a climb back up. |
| Wordpress & mariaDB memory usage | Time series | Memory each pod is using, compared to its limit. |
| Number of restarts | Stat | Total restarts per pod. |

## More information

Built for a workshop, not production:

- No TLS, no autoscaling, Prometheus only keeps 6 hours of history.
- EC2's public IP changes on every restart (no Elastic IP).
- ECR tokens and the imagePullSecret expire after about 12 hours, need manual refresh.
- WordPress replicas share a single `ReadWriteOnce` PVC for `/var/www/html` (uploads/themes/plugins). This works because minikube only has one node - on a real multi-node cluster this would need `ReadWriteMany` storage instead.
