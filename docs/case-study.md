# Case study: building a GitOps delivery platform with HA secrets management

## Problem

A Flask API needed a reproducible delivery workflow with traceable releases, safe environment promotion, autoscaling and centralized secrets instead of credentials in manifest files.

## Solution

Separated application CI from Kubernetes desired-state Git. Used GitHub Actions for tests, build, image scanning and SHA tagging, with Helm + Argo CD for dev auto-sync and manual prod promotion. Added Prometheus/Grafana and a three-member Vault Raft cluster using Kubernetes auth and Vault Agent injection.

## Engineering lessons

1. **Container image completeness:** A new `vault_config.py` import crashed Gunicorn because the Docker build omitted the file. Correcting the Dockerfile and promoting a known tested image (`56c38ee`) restored the feature.
2. **Application and infrastructure readiness are separate:** Argo CD could report healthy workloads while the Python Deployment Pod template lacked Vault annotations. Rendering Helm and checking injected containers localized the gap.
3. **GitOps promotion matters:** The prod Pods successfully received Vault secrets but the old app version (`36f432a`) did not read the file. Promoting the tested image yielded `vault_config_loaded: true` on `/api/info`.
4. **Least privilege is testable:** A dev-bound Vault role could read dev KV v2 data but was denied prod access with HTTP 403.
5. **HA has a scope:** Leadership transferred between Vault peers, but all replicas share a single Rancher Desktop node.

## Observed results

Dev HPA reached four replicas and later returned to one; Vault Raft formed a three-voter cluster; injection was observed in both prod replicas; Flask reported production environment from Vault on the promoted image; a manual Raft snapshot was completed. Automated backups were intentionally excluded.

## Next improvements

TLS/mTLS, cloud KMS auto-unseal, distinct failure domains, off-node encrypted backup and isolated restore drills, token hardening, end-to-end monitoring and alerting. None should be presented as already delivered.
