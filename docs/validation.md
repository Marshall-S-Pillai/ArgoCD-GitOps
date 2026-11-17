# Validation checklist

Run these checks after Argo CD has been installed and the Application manifest has been applied.

## Argo CD health and sync

```bash
kubectl get applications -n argocd
kubectl describe application nginx-gitops -n argocd
```

Expected: the Application reports `Synced` and `Healthy` after the first successful reconciliation.

## Workload

```bash
kubectl get deployment,service,pods -n gitops-demo
kubectl rollout status deployment/nginx -n gitops-demo
```

Expected: nginx Deployment has two available replicas; Service exposes port 80 internally.

## Git-driven change

Change `spec.replicas` in `apps/nginx/overlays/dev/deployment-patch.yaml` to 3, commit, and push to `main`. Confirm Argo CD synchronizes the new desired state and Kubernetes reports three replicas.

## Drift and self-healing

Scale the live Deployment to one replica:

```bash
kubectl scale deployment nginx -n gitops-demo --replicas=1
```

Confirm Argo CD detects the difference and restores the Git-declared replica count. Capture the Argo CD Application status and Deployment output as project evidence.

## Evidence to collect

- Argo CD Application showing Synced / Healthy.
- Pods and Deployment showing the expected replica count.
- Git commit showing the manifest change.
- Argo CD reconciliation after the manual drift test.

Do not mark these checks complete until they have been run against the target cluster.
