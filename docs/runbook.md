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

## Prometheus Adapter request-based scale-down

Dev uses a target of 100 requests/minute per replica, with a minimum of one
replica. After stopping load, allow roughly four minutes for the two-minute
Prometheus rate window and the 120-second HPA scale-down stabilization window,
plus scrape and controller delays. Prometheus Adapter exposes the per-pod
`http_requests_per_minute` metric through `custom.metrics.k8s.io`; the HPA handles
scaling. See [adapter installation and migration](prometheus-adapter.md) before
the first sync of this configuration.

```bash
kubectl get apiservice v1beta1.custom.metrics.k8s.io
kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/namespaces/gitops-dev/pods/*/http_requests_per_minute?labelSelector=app%3Dpython-api-dev'
kubectl get hpa -n gitops-dev
kubectl describe hpa python-api-dev -n gitops-dev
kubectl get deployment python-api-dev -n gitops-dev -w
```

If the HPA target is `<unknown>`, inspect HPA events, the custom metrics API, and
adapter logs. Unavailable metrics block or limit scale-down. Idle replacement
pods may not export `http_requests_total`: the app
creates its labeled counters on the first business request, and excludes health
probes and `/metrics` from counting.

The adapter discovers the metric from the always-present `up` scrape series.
Its query supplies zero for an absent request rate only when that pod has a
healthy scrape target. Down or missing targets receive no zero fallback.
Check these queries in Prometheus when troubleshooting:

```promql
up{namespace="gitops-dev",service="python-api-dev"}
http_requests_total{namespace="gitops-dev",service="python-api-dev"}
(sum by (namespace, pod) (rate(http_requests_total{namespace="gitops-dev",service="python-api-dev"}[2m])) * 60)
  or on (namespace, pod) (0 * (max by (namespace, pod) (up{namespace="gitops-dev",service="python-api-dev"}) == 1))
```

The HPA metric should be numeric, and the
deployment should return to one replica after stabilization when traffic is idle.

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
