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

For production verify every current Pod (not just `.items[0]`). Use `test -s` on `/vault/secrets/config`; never print its contents. Inspect `vault-agent-init` logs when auth or rendering fails. Deploy the app refactor image together with this filename change; see [Vault integration](vault-integration.md) for the migration gate and commands.

## Flask connectivity

Tested Gunicorn bind port is **8080**:

```bash
kubectl port-forward -n gitops-prod deployment/python-api-prod 8082:8080
# in another terminal:
curl -sS http://localhost:8082/api/info | jq .
```

## Grafana startup probes

Grafana may refuse connections on port 3000 while initializing its database and
services. `grafana.startupProbe` in `monitoring/values.yaml` checks `/api/health`
every 5 seconds and allows 60 failures (about 5 minutes). Readiness and liveness
checks start only after the startup probe succeeds. Startup probe warnings during
initialization can still occur; persistent failures require investigation.

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
kubectl logs -n monitoring deployment/monitoring-grafana -c grafana --tail=100
kubectl get events -n monitoring --sort-by=.metadata.creationTimestamp
```

If Grafana restarts, inspect the previous container's logs with `--previous`.
Look for provisioning errors, resource exhaustion, or initialization that exceeds
the startup budget before increasing the threshold further.

Grafana requests 100m CPU and 128Mi memory, with limits of 500m CPU and 512Mi
memory. The previous 200m/256Mi limits left startup running near both ceilings.
The Grafana chart derives `GOMEMLIMIT` from its memory limit; if applying a live
resource patch, also match that environment variable to the Helm rendering
(`460MiB` for the 512Mi limit).

## Guardrails

Do not publish credentials or secret values in screenshots. Do not delete Raft PVCs during app troubleshooting. Shamir unseal shares must be kept securely; a root token alone cannot unseal a sealed cluster. Manual snapshots are not a full DR plan.
