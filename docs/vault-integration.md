# Vault Agent Injector integration

## Application contract and release gate

The refactor contract was inspected in `../gitops-python-app/README.md`,
`application_config.py`, and its loader tests. The app repository owns these files;
this change does not modify them.

- Fixed file: `/vault/secrets/config`, UTF-8, plain JSON object.
- No required secret keys. `APP_ENV` is optional; when supplied it must be a
  nonempty string and is publicly returned by `/api/info`. Never put credentials
  in `APP_ENV`. `DEMO_API_KEY` is unused. `APP_VERSION` remains an env var.
- Render `.Data.data | toJSON` from KV v2, without a Vault response envelope,
  shell exports, or `KEY=value` syntax. This preserves JSON types and escaping.
- The app rereads on every `/api/info` and `/health/ready` request. Invalid JSON,
  encoding, permissions, or `APP_ENV` cause 503; liveness is independent. Missing
  files fall back to env values, so the deployment checks file presence at startup.

**Release gate:** At inspection the app refactor was uncommitted, and both
environment files and running Deployments used image `56c38ee`, which uses the
old `config.json` contract. Build/test/publish the refactor through its own repo
first. Set the real immutable image tag in `environments/dev/values.yaml` in the
same release as this filename change. Validate dev before promoting that tag and
chart to prod. Do not sync this chart against the old image or invent an image
tag. Dev auto-syncs from `main`; coordinate merging accordingly. Earlier README
and evidence claims about `config.json` describe the previous release.

## Verified existing configuration

Read-only checks on `rancher-desktop` on 2026-10-19 established:

| Setting | Dev | Prod |
| --- | --- | --- |
| Namespace | `gitops-dev` | `gitops-prod` |
| Deployment / Helm release | `python-api-dev` | `python-api-prod` |
| ServiceAccount | `python-api` | `python-api` |
| Auth mount | `auth/kubernetes` | `auth/kubernetes` |
| Role / policy | `python-dev` | `python-prod` |
| KV v2 API / policy path | `secret/data/python-api/dev` | `secret/data/python-api/prod` |
| CLI path (`-mount=secret`) | `python-api/dev` | `python-api/prod` |

`secret/` is KV v2. Both paths exist with `APP_ENV` (valid string) and
`DEMO_API_KEY`; values were not printed. Live policies exactly match
`vault/policies/*.hcl`: `read` on one data path only; no wildcard, metadata/list,
or write access. Roles bind precisely the above ServiceAccount and namespace,
attach the matching policy with a one-hour token TTL, and retain the default
policy for Agent token lifecycle operations. Reuse these without live writes.

Vault is initialized and unsealed, with three ready Raft Pods. The injector
Deployment is ready (`hashicorp/vault-k8s:1.7.6`), its service exists, and webhook
`vault-agent-injector-cfg` has a CA bundle and selects both app namespaces.
Existing app Pods have `vault-agent-init` and `vault-agent`
(`hashicorp/vault:2.0.4`). This verifies the existing installation, not the new
filename rollout or runtime rendering of the updated chart.

Auth uses the observed API endpoint `https://10.43.0.1:443`, a CA,
`disable_local_ca_jwt=false`, and no stored reviewer JWT. Vault uses its rotating
local ServiceAccount token. `vault-server-binding` grants `system:auth-delegator`
to `vault/vault`. The app needs no Kubernetes RoleBinding. Its ServiceAccount
defaults automount to false; the Pod enables it when Vault is enabled so Agent
can authenticate. No Kubernetes Secret is needed.

The observed service is `http://vault-active.vault.svc.cluster.local:8200`.
HTTP and manual unseal are existing lab choices. `vault/values.yaml` already
enables the injector. `argocd/vault-application.yaml` is empty and does not manage
Vault. Reuse the installed release; recover its exact Helm chart/version and
configuration before adopting it into Argo CD.

## Startup and rotation

Annotations are on `Deployment.spec.template.metadata.annotations`. The `config`
suffix and filename render `/vault/secrets/config`. Mode `0644` lets the image's
`appuser` read files owned by Agent's different UID. The application secrets
volume is mounted only in `python-api` (plus the Agents); no Vault token file is
shared with the app.

The init Agent authenticates and renders before app startup. Auth/path failures
leave the Pod initializing. The sidecar polls KV v2 every 30 seconds. Agent's
destination-file renderer replaces files atomically; do not add redirection/copy
hooks. The app rereads the file, so healthy rotation needs no app restart.
Refresh errors leave the last rendered file; there is no freshness guarantee
during outages. Monitor Agent health and logs.

The installed webhook uses `failurePolicy: Ignore`. A startup probe requires a
readable, nonempty config file to keep Pods unready if injection is skipped;
after roughly 150 seconds Kubernetes restarts the app container. This does not
change the cluster-wide webhook policy. After startup the app readiness handler
validates contents. If the file is later removed, its documented fallback applies.

## Inspect or recover Vault configuration

Using an existing authorized operator CLI session, inspect without printing
application secrets or tokens. For host CLI access, port-forward `svc/vault-active`
to localhost, set `VAULT_ADDR=http://127.0.0.1:8200`, and authenticate interactively
outside Git. Do not put tokens on the command line.

```sh
kubectl config current-context
kubectl get namespaces vault gitops-dev gitops-prod
kubectl get deployment vault-agent-injector -n vault
kubectl get mutatingwebhookconfiguration vault-agent-injector-cfg
kubectl get clusterrolebinding vault-server-binding -o yaml
vault status
vault secrets list -detailed
vault auth list
vault read auth/kubernetes/config
vault read auth/kubernetes/role/python-dev
vault read auth/kubernetes/role/python-prod
vault policy read python-dev
vault policy read python-prod
```

No auth/policy repair was required on this cluster. For recovery, verify the
target mount version, API/CA, reviewer RBAC, namespaces, and paths first. Only if
absent, enable `secret/` with `vault secrets enable -path=secret -version=2 kv`
and Kubernetes auth with `vault auth enable -path=kubernetes kubernetes`.
For new in-cluster auth configuration, derive `kubernetes_host` from that Vault
Pod's `KUBERNETES_SERVICE_HOST`/`KUBERNETES_SERVICE_PORT` and use local rotating
token/CA defaults. Do not overwrite working config, reinitialize storage, or
create token Secrets. See [HashiCorp Kubernetes auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes).

If the verified policy/roles need restoring, an authorized operator can run:

```sh
vault policy write python-dev vault/policies/python-dev.hcl
vault policy write python-prod vault/policies/python-prod.hcl
vault write auth/kubernetes/role/python-dev \
  bound_service_account_names=python-api \
  bound_service_account_namespaces=gitops-dev token_policies=python-dev token_ttl=1h
vault write auth/kubernetes/role/python-prod \
  bound_service_account_names=python-api \
  bound_service_account_namespaces=gitops-prod token_policies=python-prod token_ttl=1h
```

Provision missing KV data through an authorized secure workflow. Existing valid
paths need no secret writes. Keep credentials out of manifests and Git.

## Validate and deploy

After satisfying the image release gate above:

```sh
helm lint charts/python-api -f environments/dev/values.yaml --strict
helm lint charts/python-api -f environments/prod/values.yaml --strict
render_dir=$(mktemp -d)
helm template python-api-dev charts/python-api -n gitops-dev \
  -f environments/dev/values.yaml > "$render_dir/dev.yaml"
helm template python-api-prod charts/python-api -n gitops-prod \
  -f environments/prod/values.yaml > "$render_dir/prod.yaml"
kubectl apply --dry-run=server -n gitops-dev -f "$render_dir/dev.yaml"
kubectl apply --dry-run=server -n gitops-prod -f "$render_dir/prod.yaml"
```

The ApplicationSet chart source now matches its environment generator and this
checkout's origin. Confirm Argo CD can read that URL. Commit/publish the
coordinated image and manifest changes, then deploy through GitOps:

```sh
kubectl apply -f argocd/applicationset.yaml
argocd app get python-api-dev --refresh
argocd app sync python-api-dev
kubectl rollout status -n gitops-dev deployment/python-api-dev --timeout=180s
# Promote the tested tag in Git before manually syncing prod.
argocd app get python-api-prod --refresh
argocd app sync python-api-prod
kubectl rollout status -n gitops-prod deployment/python-api-prod --timeout=180s
```

New Pods trigger injection; changed templates roll out automatically. KV updates
alone do not need a rollout. Roll back the image tag and filename/template change
together if needed; the old image and new filename are incompatible.
Verify every current Pod without printing values:

```sh
for env in dev prod; do
  kubectl get pods -n "gitops-$env" -l "app=python-api-$env" \
    -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.initContainers[*].name}{" | "}{.spec.containers[*].name}{"\n"}{end}'
  for pod in $(kubectl get pods -n "gitops-$env" -l "app=python-api-$env" -o name); do
    kubectl exec -n "gitops-$env" "$pod" -c python-api -- python -c \
      'import json; from pathlib import Path; c=json.loads(Path("/vault/secrets/config").read_text(encoding="utf-8")); assert isinstance(c,dict); assert "APP_ENV" not in c or (isinstance(c["APP_ENV"],str) and c["APP_ENV"].strip()); print("config readable and valid")'
  done
done
```

For a dev rotation test, record Pod UIDs/restart counts, then use an authorized
operator to `vault kv patch -mount=secret python-api/dev APP_ENV=development-rotated`
(a public, noncredential setting). Wait for a healthy refresh and query `/api/info`
through a localhost port-forward. Confirm `vault_config_loaded=true`, the expected
public setting, and unchanged UIDs/restarts. Restore the previous setting securely.
The new filename/image rollout and rotation test remain deployment steps.

## Validation performed for this change

- Strict Helm lint passed for dev, prod, and the Vault-disabled default values.
- Server-side dry-run apply passed in `gitops-dev` and `gitops-prod`, and for the
  updated ApplicationSet. These requests persisted no resources.
- Dry-run Pod admission in both app namespaces invoked the installed webhook
  successfully. Both generated Agent configurations were checked for the exact
  destination, `0644` permissions, JSON template, role, auth mount, Vault address,
  init-versus-sidecar behavior, and 30-second interval.
- The default chart has no injection annotations/startup probe and disables Pod
  token automount. Helm rejects Vault-enabled values missing role or secret path.
- Local HCL files match Vault CLI formatting and the inspected live policies.
- No rollout, new-file runtime rendering, auth isolation test, secret write, or
  rotation test was performed. Those require the coordinated image release.

## Troubleshooting

- Missing Agents: inspect Pod annotations/`agent-inject-status`, webhook
  selectors/CA, injector availability, and admission events. The startup check
  blocks readiness if fail-open admission skips injection.
- Init waits or 403: inspect `kubectl logs -n <namespace> <pod> -c vault-agent-init`;
  check Vault is unsealed, `vault-active` endpoints, DNS/network access to 8200,
  `auth/kubernetes`, namespace/ServiceAccount bindings, TokenReview RBAC, and
  read policy on the KV v2 **data** path.
- Probe failure/503: verify the refactor image, exact `config` filename,
  readability as the app user, JSON object shape, and `APP_ENV` type without
  printing values. `config.json` is the previous contract.
- Rotation stuck: inspect `vault-agent` logs/restarts, renewal/re-auth, KV
  path/version, connectivity, and the 30-second polling interval. Stale rendered
  data can remain usable during refresh errors.
- Use `kubectl describe pod -n <namespace> <pod>` for scheduling/probe events and
  `kubectl get endpoints -n vault vault-active vault-agent-injector-svc` for service
  backends. Treat logs as sensitive before sharing them.

References: [Injector annotations](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/injector/annotations)
and [Agent templates](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template).
