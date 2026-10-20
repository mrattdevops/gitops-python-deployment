# Architecture and decisions

The application and deployment repositories are separate. GitHub Actions builds/tests the Flask image and updates the dev tag in the config repo. Argo CD ApplicationSet deploys from Git; dev synchronizes automatically, while prod is deliberately promoted and synchronized manually.

**Secrets:** A three-member Vault cluster uses integrated Raft storage; each server has a `local-path` PVC. Kubernetes-auth roles are bound to ServiceAccounts and namespaces. An admission webhook injects Vault Agent into annotated Python Pods. The Agent writes `/vault/secrets/config.json`, which the Flask app reads per request.

**Reliability constraints:** Three voters tolerate one **Vault member** loss while two remain functioning/unsealed. The three Pods share one host and local storage, so this is not host- or region-level HA. Manual Shamir unseal is required following restarts; the lab uses HTTP rather than TLS.

**Ops:** Prometheus/Grafana for cluster/API metrics, Prometheus Adapter exposing per-pod requests/minute to the dev HPA (target 100, 1–4 replicas), and a manual Raft snapshot. Previous autoscaling tests observed 1→4→1 scaling; the adapter migration requires a repeat live test. See [adapter setup](prometheus-adapter.md). No automated backup CronJob.

The top-level README includes the rendered architecture; source: [`../assets/architecture.mmd`](../assets/architecture.mmd).
