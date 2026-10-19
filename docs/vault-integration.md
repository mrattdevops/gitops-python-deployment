# Vault HA and secret injection runbook

## Secrets and access boundaries

- KV v2 mount: `secret/`.
- Dev path: `secret/data/python-api/dev`, policy `python-dev`, role bound to `python-api` in `gitops-dev`.
- Prod path: `secret/data/python-api/prod`, policy `python-prod`, role bound to `python-api` in `gitops-prod`.
- Each environment's Helm values controls `.Values.vault.enabled`, role, and secret path. Pod template annotations enable the Agent Injector. Application runs with the `python-api` ServiceAccount.

## Verify injection, without revealing secret values

```bash
kubectl get pods -n vault
kubectl get pods -n gitops-dev -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.initContainers[*].name}{" | "}{.spec.containers[*].name}{"\n"}{end}'
kubectl get pods -n gitops-prod -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.initContainers[*].name}{" | "}{.spec.containers[*].name}{"\n"}{end}'
```

For a selected Python Pod, use `kubectl exec -n <namespace> <pod> -c python-api -- test -s /vault/secrets/config.json`. Avoid `cat` and log output containing secrets. The `/api/info` endpoint exposes only environment, image version, hostname, and whether config file exists.

## Rotation

Change a harmless dev key using an authorized Vault identity (`vault kv patch -mount=secret python-api/dev APP_ENV=development-rotated`). Verify the API updates once Vault Agent rerenders the file; compare Pod UID and Python container restart count before and after to prove no restart. Restore a sensible dev value after the test. Never echo a real secret into logs.

## Authorization evidence

A dev ServiceAccount token successfully authenticated and read the dev path; attempting the prod path returned `403 permission denied`. Production authentication and reverse isolation require separate evidence before being marked fully verified.

## Troubleshooting

- Annotations must be under `Deployment.spec.template.metadata.annotations`, not top-level metadata.
- Agent sidecar present but file missing: inspect `vault-agent-init` logs, ServiceAccount role, namespace binding, policies and path.
- Python crashes with `ModuleNotFoundError: vault_config`: ensure Dockerfile copies `vault_config.py` to `/app` and CI builds a new immutable image.
- Port-forward: Gunicorn in the tested prod image listens on `0.0.0.0:8080`, so `kubectl port-forward -n gitops-prod deployment/python-api-prod 8082:8080`.
- CLI from Mac: start a local-only port-forward to `svc/vault-active` and set `VAULT_ADDR=http://127.0.0.1:8200` (lab HTTP only).
