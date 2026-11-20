# ArgoCD-GitOps

A hands-on GitOps capstone project: deploy and manage a Kubernetes application using Argo CD, with Git as the single source of truth.

## Project objectives

- Store Kubernetes application configuration in Git.
- Install and configure Argo CD in a Kubernetes cluster.
- Create an Argo CD Application that watches this repository.
- Enable automated synchronization, pruning, and self-healing.
- Demonstrate drift detection and Git-driven application updates.

## Implementation completed

This project has been deployed and validated on a live Amazon EC2 Ubuntu instance using Minikube with the Docker driver.

| Component | Implementation |
|---|---|
| Cloud environment | Amazon EC2 |
| Operating system | Ubuntu 26.04.1 LTS |
| Kubernetes | Minikube (Docker driver) |
| GitOps controller | Argo CD |
| Application | Nginx |
| Namespace | `gitops-demo` |
| Argo CD namespace | `argocd` |
| Git branch | `main` |
| Argo CD Application | `nginx-gitops` |

### Validation results

- Argo CD Application reported **Synced** and **Healthy** in the Argo CD dashboard.
- Argo CD completed an automated sync from the repository.
- The Nginx Deployment became Healthy in the `gitops-demo` namespace.
- **Self-healing test:** the live Deployment was manually scaled from 2 replicas to 1. Argo CD detected the drift and restored the Git-defined replica count of 2.
- **Git-driven update test:** the replica count in `apps/nginx/overlays/dev/deployment-patch.yaml` was changed from 2 to 3 and committed to `main`. Argo CD synchronized the change to the cluster.

> These results describe the deployment and tests performed in the project environment. To reproduce them, follow the setup and validation steps below. The EC2 instance and Minikube cluster are not provisioned automatically by this repository.

## Architecture

```text
Developer
   |
   | git commit / push
   v
GitHub: Marshall-S-Pillai/ArgoCD-GitOps
   |
   | Argo CD polls the configured Git repository
   v
Argo CD (argocd namespace)
   |
   | compares desired state with live cluster state
   | automated sync + prune + self-heal
   v
Kubernetes cluster (Minikube on EC2)
   |
   +-- gitops-demo namespace
         +-- nginx Deployment
         +-- nginx Service
```

## Repository structure

```text
.
├── apps/
│   └── nginx/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── namespace.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           └── dev/
│               ├── deployment-patch.yaml
│               └── kustomization.yaml
├── argocd/
│   └── application.yaml
├── docs/
│   └── validation.md
└── README.md
```

## Prerequisites

- An Ubuntu-based VM or server with sufficient CPU and memory for Minikube and Argo CD.
- Docker Engine.
- `kubectl` and Minikube.
- Git.
- Network access from the cluster to GitHub and the Argo CD installation source.

> This repository contains the GitOps configuration. The cloud VM, Docker, Kubernetes cluster, and Argo CD installation are separate environment prerequisites.

## 1. Clone the repository

```bash
git clone https://github.com/Marshall-S-Pillai/ArgoCD-GitOps.git
cd ArgoCD-GitOps
```

## 2. Start or verify the Kubernetes cluster

For the Minikube Docker-driver setup used in this project:

```bash
minikube start --driver=docker
kubectl config current-context
kubectl get nodes
```

Make sure the expected cluster is selected before installing anything.

## 3. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl get pods -n argocd
```

For production, pin and review a specific Argo CD release rather than relying on the moving `stable` manifest.

## 4. Access the Argo CD UI

For local or SSH-tunnelled access, run:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080`. When connecting to a remote EC2 instance, establish an SSH local port-forward from your workstation to the instance as well. Do not expose the Argo CD server publicly just for this demo.

Retrieve the initial admin password with:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
echo
```

Change the initial password after the first login. Do not commit credentials or Kubernetes secrets to this repository.

## 5. Register the GitOps application

From the repository root:

```bash
kubectl apply -n argocd -f argocd/application.yaml
kubectl get applications -n argocd
```

Argo CD reads the Kustomize configuration under `apps/nginx/overlays/dev` and reconciles it with the cluster.

## 6. Validate deployment

```bash
kubectl get applications -n argocd
kubectl get all -n gitops-demo
kubectl get deployment nginx -n gitops-demo
kubectl rollout status deployment/nginx -n gitops-demo
```

The initial dev overlay defines an Nginx Deployment with two replicas and a ClusterIP Service on port 80.

## 7. Demonstrate GitOps and drift detection

### A. Git-driven deployment update

1. Edit `apps/nginx/overlays/dev/deployment-patch.yaml` and change the replica count from `2` to `3`.
2. Commit and push the change:

   ```bash
   git add apps/nginx/overlays/dev/deployment-patch.yaml
   git commit -m "Scale nginx to three replicas"
   git push origin main
   ```

3. Check the Argo CD Application. Automated sync should apply the desired state.
4. Verify the live Deployment:

   ```bash
   kubectl get deployment nginx -n gitops-demo
   kubectl get pods -n gitops-demo
   ```

### B. Self-healing / drift correction

1. With the Git-defined replica count set to `2`, manually change the live Deployment:

   ```bash
   kubectl scale deployment nginx -n gitops-demo --replicas=1
   ```

2. Argo CD detects that the live state differs from Git and, with automated self-healing enabled, restores the replica count to `2`.
3. Verify the result:

   ```bash
   kubectl get deployment nginx -n gitops-demo
   kubectl get pods -n gitops-demo
   ```

Allow time for the Argo CD reconciliation loop.

## 8. Cleanup

Delete the Argo CD Application first:

```bash
kubectl delete -n argocd -f argocd/application.yaml
```

Then, when you no longer need Argo CD:

```bash
kubectl delete namespace argocd
kubectl delete namespace gitops-demo
```

These commands delete resources in the selected Kubernetes context. Verify the context before running them.

## Capstone deliverables

- [x] Git repository with Kubernetes configuration and GitOps structure.
- [x] Argo CD Application manifest with automated sync and drift correction settings.
- [x] README with setup instructions, architecture, and deployment workflow.
- [x] Argo CD installed and running in a Kubernetes cluster.
- [x] Application successfully synchronized and running.
- [x] Git commit update and drift/self-healing behavior demonstrated.

## Notes

- This is a hands-on demo environment, not a production-hardened Kubernetes deployment.
- Keep EC2 SSH access restricted to trusted source IPs and access Argo CD through an SSH tunnel or an appropriately secured ingress.
- Never commit private keys, admin passwords, tokens, or Kubernetes Secret values to Git.
