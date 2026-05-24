# goit-argo

GitOps repository for Argo CD manifests. This repository is separate from the
Terraform infrastructure repository and is used as the Git source that Argo CD
watches on the `main` branch.

## Repositories

This solution uses two repositories:

- Infrastructure repository: `https://github.com/nickk-o/MLOps/tree/lesson-7`
- GitOps repository: `https://github.com/nickk-o/goit-argo.git`

The infrastructure repository installs Argo CD with Terraform. This repository
contains the Kubernetes and Argo CD manifests that Argo CD synchronizes from
GitHub.

## Project Structure

```text
goit-argo
├── application.yaml
├── namespaces
│   ├── application
│   │   ├── nginx.yaml
│   │   └── ns.yaml
│   └── infra-tools
│       └── ns.yaml
└── README.md
```

## Purpose Of The Files

`application.yaml` defines the Argo CD `Application` resource for the test
application. In this project it deploys MLflow from a Helm chart with
automated sync, pruning, self-heal, and automatic namespace creation.

`namespaces/application/ns.yaml` declares the target namespace for application
workloads.

`namespaces/infra-tools/ns.yaml` declares the namespace used by Argo CD system
components.

`namespaces/application/nginx.yaml` contains a simple nginx example manifest.

## Deployment Model

Argo CD is installed from the Terraform repository as a `helm_release` in the
`infra-tools` namespace.

Terraform also configures Argo CD to watch this GitHub repository on the
`main` branch through ApplicationSets. After changes are committed and pushed,
Argo CD detects the new revision and synchronizes the cluster state
automatically.

The repository URL configured in Terraform:

```hcl
app_repo_url    = "https://github.com/nickk-o/goit-argo.git"
app_repo_branch = "main"
```

## Test Application

The deployed test application is MLflow. The Argo CD `Application` in
`application.yaml` uses:

- `repoURL: https://community-charts.github.io/helm-charts`
- `chart: mlflow`
- `targetRevision: 1.8.1`
- inline Helm values in `spec.source.helm.values`

The manifest also enables:

- automated sync
- `prune: true`
- `selfHeal: true`
- `CreateNamespace=true`

## How To Apply The Solution

### 1. Push GitOps manifests

Run in this repository:

```bash
git add -A
git commit -m "Add GitOps manifests"
git push origin main
```

### 2. Deploy Argo CD from Terraform

Run in the infrastructure repository:

```bash
cd ~/terraform/argocd
terraform init -reconfigure
terraform plan
terraform apply
```

## Verification

Verify that Argo CD is running:

```bash
kubectl get pods -n infra-tools
```

Expected result: several pods with the `argocd-` prefix.

Verify that Argo CD created and synchronized the Application resources:

```bash
kubectl get applications -n infra-tools
```

Verify that the target workload was deployed:

```bash
kubectl get pods -n application
kubectl get svc -n application
```

## Access To Argo CD UI

Get the initial admin password:

```bash
kubectl -n infra-tools get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo
```

Start port forwarding:

```bash
kubectl port-forward svc/argocd-server -n infra-tools 8080:80
```

Open:

```text
http://localhost:8080
```

## Access To MLflow

Start port forwarding:

```bash
kubectl port-forward svc/mlflow -n application 8081:80
```

Open:

```text
http://localhost:8081
```

## Result

The solution provides:

- Argo CD deployed through Terraform in the `infra-tools` namespace
- a separate public GitOps repository with namespace and application manifests
- an Argo CD `Application` stored in Git and synchronized from GitHub
- automatic reconciliation of the MLflow Helm deployment into the
  `application` namespace

## Acceptance Criteria

- Argo CD is deployed by Terraform as a `helm_release`
- `argocd-values.yaml` contains service, RBAC, extra arguments, and timeout
  settings
- `application.yaml` describes a Helm deployment with chart source and values
- Argo CD synchronizes the application automatically from GitHub
- the `application` namespace contains the deployed workload after sync
- access is available through `kubectl port-forward`
