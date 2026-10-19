# Operations and deployment verification

## Cluster and Argo CD

```bash
kubectl config current-context
kubectl get applications.argoproj.io -n argocd
kubectl get pods -n gitops-dev
kubectl get pods -n gitops-prod
kubectl rollout status deployment/python-api-dev -n gitops-dev
kubectl rollout status deployment/python-api-prod -n gitops-prod
```

## Vault

```bash
kubectl get pods,statefulset,pvc -n vault
kubectl get deployment python-api-prod -n gitops-prod -o jsonpath='{.spec.template.metadata.annotations}'
kubectl get pods -n gitops-prod -l app=python-api-prod -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.initContainers[*].name}{" | "}{.spec.containers[*].name}{"\n"}{end}'
```

For production verify every current Pod (not just `.items[0]`). Use `test -s` on `/vault/secrets/config.json`; never print its contents. Inspect `vault-agent-init` logs when auth or rendering fails.

## Flask connectivity

Tested Gunicorn bind port is **8080**:

```bash
kubectl port-forward -n gitops-prod deployment/python-api-prod 8082:8080
# in another terminal:
curl -sS http://localhost:8082/api/info | jq .
```

## Guardrails

Do not publish credentials or secret values in screenshots. Do not delete Raft PVCs during app troubleshooting. Shamir unseal shares must be kept securely; a root token alone cannot unseal a sealed cluster. Manual snapshots are not a full DR plan.
