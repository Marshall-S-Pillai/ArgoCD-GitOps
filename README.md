# ArgoCD-GitOps

A hands-on GitOps capstone project: deploy and manage a Kubernetes application using Argo CD, with Git as the single source of truth.

## Project objectives

- Store Kubernetes application configuration in Git.
- Install and configure Argo CD in a Kubernetes cluster.
- Create an Argo CD Application that watches this repository.
- Enable automated synchronization, pruning, and self-healing.
- Demonstrate drift detection and Git-driven application updates.

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
Kubernetes cluster
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

- A Kubernetes cluster (for example, kind, minikube, or Amazon EKS).
- `kubectl` configured to access that cluster.
- Git.
- Network access from the cluster to GitHub and the Argo CD installation source.

> This repository contains the GitOps configuration. Creating a repository does not create a Kubernetes cluster or deploy the application. Complete the deployment steps below against your own cluster.

## 1. Clone the repository

```bash
git clone https://github.com/Marshall-S-Pillai/ArgoCD-GitOps.git
cd ArgoCD-GitOps
```

## 2. Verify cluster access

```bash
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

## 4. Expose the Argo CD API server (local access)

In a separate terminal:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080` and accept the local development certificate warning. Retrieve the initial admin password with:

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

Argo CD will read the Kustomize configuration under `apps/nginx/overlays/dev` and reconcile it with the cluster.

## 6. Validate deployment

```bash
kubectl get applications -n argocd
kubectl get all -n gitops-demo
kubectl get deployment nginx -n gitops-demo
kubectl rollout status deployment/nginx -n gitops-demo
```

The expected workload is an nginx Deployment with two replicas and a ClusterIP Service on port 80.

## 7. Demonstrate GitOps and drift detection

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

5. To demonstrate self-healing, manually change the live replica count:

   ```bash
   kubectl scale deployment nginx -n gitops-demo --replicas=1
   ```

   Because automated self-healing is enabled, Argo CD should reconcile the live state back to the replica count declared in Git. Allow time for the reconciliation loop.

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
- [ ] Argo CD installed and running in a Kubernetes cluster.
- [ ] Application successfully synchronized and running.
- [ ] Git commit update and drift/self-healing behavior demonstrated.

The final three items require execution and evidence from a real Kubernetes cluster; they are not claimed as completed by this repository commit.
