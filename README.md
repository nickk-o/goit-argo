# goit-argo

GitOps repository for Argo CD manifests. This repository is the Git source
that the Terraform-installed Argo CD instance watches on the `main` branch.

## Repositories

This solution uses two repositories:

- Infrastructure repository: `terraform`
- GitOps repository: `goit-argo`

The infrastructure repository installs Argo CD and configures its
`ApplicationSet` objects. This repository contains the namespace manifests and
root-level Argo CD `Application` manifests that Argo CD synchronizes.

## Project Structure

```text
goit-argo
├── application.yaml
├── minio.yaml
├── postgres.yaml
├── pushgateway.yaml
├── namespaces
│   ├── application
│   │   └── ns.yaml
│   ├── infra-tools
│   │   └── ns.yaml
│   └── monitoring
│       └── ns.yaml
└── README.md
```

## Why The Files Are Here

Terraform configures Argo CD to watch this repository in two ways:

- `namespaces-appset` scans `namespaces/*` with recursion enabled.
- `root-application-appset` scans path `"."` for root-level manifests.

That means the namespace YAML files belong under `namespaces/*`, while the
Argo CD `Application` objects for MLflow, MinIO, PostgreSQL, and PushGateway
should live at the repository root.

## Applications

### MLflow

[application.yaml](application.yaml)
deploys the MLflow Tracking Server:

- chart: `mlflow`
- repository: `https://community-charts.github.io/helm-charts`
- namespace: `application`
- service: `ClusterIP`
- port: `5000`
- backend store: external PostgreSQL
- artifact store: MinIO bucket `mlflow-artifacts`

### MinIO

[minio.yaml](minio.yaml)
deploys MinIO in `application` with:

- bucket: `mlflow-artifacts`
- service type: `ClusterIP`

### PostgreSQL

[postgres.yaml](postgres.yaml)
deploys PostgreSQL in `application` with:

- database: `mlflow`
- username: `mlflow`
- service type: `ClusterIP`

### PushGateway

[pushgateway.yaml](pushgateway.yaml)
deploys Prometheus PushGateway in `monitoring` with:

- service type: `ClusterIP`
- port: `9091`
- in-cluster address:
  `http://pushgateway.monitoring.svc.cluster.local:9091`

## Namespaces

- [namespaces/application/ns.yaml](namespaces/application/ns.yaml)
  declares the `application` namespace.
- [namespaces/infra-tools/ns.yaml](namespaces/infra-tools/ns.yaml)
  declares the `infra-tools` namespace.
- [namespaces/monitoring/ns.yaml](namespaces/monitoring/ns.yaml)
  declares the `monitoring` namespace.

## Apply Flow

1. Commit and push changes from this repository:

```bash
git add -A
git commit -m "Prepare MLflow GitOps applications"
git push origin main
```

2. Ensure Argo CD exists from the infrastructure repository:

```bash
terraform init -reconfigure
terraform plan
terraform apply
```

After the push, Argo CD should detect the new revision and sync automatically.

This is the GitHub repository Argo CD watches:

- `https://github.com/nickk-o/goit-argo.git`

## Verification

```bash
kubectl get applications -n infra-tools
kubectl get pods -n application
kubectl get svc -n application
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Expected services:

- `mlflow`
- `minio`
- `mlflow-postgres-postgresql`
- `pushgateway`

## Port-Forward

### MLflow

```bash
kubectl port-forward svc/mlflow -n application 5000:5000
```

Open `http://localhost:5000`.

### PushGateway

```bash
kubectl port-forward svc/pushgateway -n monitoring 9091:9091
```

Open `http://localhost:9091`.
