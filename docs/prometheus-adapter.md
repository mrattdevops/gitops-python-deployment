# Request-based HPA with Prometheus Adapter

The metric path is Python `/metrics` → Prometheus → Prometheus Adapter →
`custom.metrics.k8s.io` → Kubernetes HPA → Deployment replicas.

The app continues exporting the `http_requests_total` counter. The adapter
converts its two-minute rate to requests/minute, sums routes/statuses within each
pod, and publishes `http_requests_per_minute`. Dev's `Pods` HPA targets an
`AverageValue` of 100 requests/minute per pod, with 1–4 replicas and a 120-second
scale-down stabilization window. Prod remains fixed at two replicas.

Configuration:

- `monitoring/prometheus-adapter-values.yaml`: Prometheus address and custom metric rule.
- `argocd/prometheus-adapter-application.yaml`: adapter Helm chart 5.3.0, managed by Argo CD.
- `environments/dev/values.yaml`: HPA metric and target.
- `charts/python-api/templates/hpa.yaml`: standard `autoscaling/v2` HPA.

The rule discovers pods from `up`, which exists even before an idle app has
created its labeled request counters. It supplies a per-pod zero only for a
healthy scrape target. This avoids the previous idle-pod missing-metric problem
without interpreting a missing or failed scrape as zero traffic. Namespace and
pod matchers supplied by the adapter keep each HPA's query scoped to its pods.

## Migrate an existing KEDA deployment

Use two GitOps syncs to avoid two HPAs controlling the same Deployment. The
working tree contains the final configuration with dev autoscaling enabled.

1. For the first migration commit, temporarily set dev's
   `autoscaling.enabled: false`. Include the chart's KEDA removal and the new
   adapter files. Commit/push to `main` and let Argo CD sync dev with pruning.
   This temporarily fixes dev at `replicaCount: 1`. Keep the KEDA operator running
   until its ScaledObject finalizer and owned HPA have been cleaned up. Confirm the old ScaledObject
   and its owned HPA have disappeared before proceeding:

   ```bash
   kubectl get scaledobject python-api-dev-http -n gitops-dev
   kubectl get hpa -n gitops-dev
   ```

   The old ScaledObject should be NotFound and no dev HPA should remain. If KEDA
   is no longer installed, the ScaledObject API itself may also be absent.

2. Create the adapter Application after its values file is available on `main`:

   First inspect `kubectl get apiservice v1beta1.custom.metrics.k8s.io`. If it
   already belongs to another adapter, inspect its ownership before installing
   another provider for that API group/version. NotFound is expected when no
   custom metrics provider is installed.

   ```bash
   kubectl apply -f argocd/prometheus-adapter-application.yaml
   kubectl get application prometheus-adapter -n argocd -w
   ```

   Once Argo CD reports Synced/Healthy, check adapter readiness and discovery:

   ```bash
   kubectl rollout status deployment/prometheus-adapter -n monitoring --timeout=180s
   kubectl wait --for=condition=Available apiservice/v1beta1.custom.metrics.k8s.io --timeout=180s
   kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
   kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/namespaces/gitops-dev/pods/*/http_requests_per_minute?labelSelector=app%3Dpython-api-dev'
   ```

   Discovery can take a relist interval (30 seconds). Expect one value per
   healthy dev pod, including zero for an idle pod. Kubernetes may format
   fractional values with `m`; for example, `1500m` means 1.5 requests/minute.

3. Set dev's `autoscaling.enabled: true` again, keeping the supplied `Pods`
   metric. Commit/push and let Argo CD sync. Check the HPA:

   ```bash
   kubectl describe hpa python-api-dev -n gitops-dev
   kubectl get hpa -n gitops-dev -w
   ```

   `ScalingActive` should be True and the metric target should be numeric. The
   Deployment manifest omits `spec.replicas` while autoscaling is enabled so
   GitOps does not continually overwrite the HPA's replica count. KEDA is no
   longer required for this app; removing a shared KEDA installation is separate
   from this migration.

For a fresh cluster, install the adapter and verify its custom metric before
syncing dev with autoscaling enabled; no KEDA removal step is needed.

## Verify traffic and scale-down

Forward the dev Service for test traffic:

```bash
kubectl port-forward -n gitops-dev svc/python-api-dev 8080:80
```

In another terminal, generate modest test traffic for several minutes:

```bash
while true; do
  curl -fsS http://localhost:8080/ > /dev/null
  sleep 0.2
done
```

Service port-forwarding selects one backing pod for the session. That is enough
to exercise the aggregate HPA calculation, but use an ingress or a client inside
the cluster for a test that distributes load across replicas.

Stop the loop with Ctrl-C. Allow roughly four minutes for the two-minute rate
window and 120-second stabilization, plus controller/scrape delays. Dev should
return to one replica. Health probes and `/metrics` requests are excluded from
the app's business request counter.

If values are missing, inspect the monitoring path before changing thresholds:

```bash
kubectl describe apiservice v1beta1.custom.metrics.k8s.io
kubectl logs deployment/prometheus-adapter -n monitoring --tail=100
kubectl describe hpa python-api-dev -n gitops-dev
```

In Prometheus, check `up{namespace="gitops-dev",service="python-api-dev"}` and
the per-pod query in the [runbook](runbook.md#prometheus-adapter-request-based-scale-down).
The existing `100` target is a lab setting; tune it using measured per-pod
capacity and response-time goals.

## Production and other metrics

Prod autoscaling and its ServiceMonitor remain disabled. To opt prod in, enable
its ServiceMonitor and configure the same `Pods` metric under its autoscaling
values; the adapter rule already supports the prod namespace and service.

An empty `autoscaling.metrics` list uses the chart's CPU utilization target
instead. To combine CPU and request traffic, put both metrics in the same HPA's
metrics list so one HPA owns the Deployment.

References: [adapter rule configuration](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md),
[adapter chart values](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-adapter/values.yaml).
