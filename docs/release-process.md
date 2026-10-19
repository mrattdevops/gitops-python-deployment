# Release and rollback workflow

1. Commit Python application change to `gitops-python-app`.
2. CI runs tests/build/security checks, publishes a SHA-tagged Docker Hub image, and commits the new dev tag to `gitops-python-deployment`.
3. Argo CD dev auto-syncs; validate probes, metrics, injection and Flask endpoint.
4. Promote a verified immutable image tag to `environments/prod/values.yaml` via PR.
5. Merge, then manually sync `python-api-prod` in Argo CD; verify **both** replicas.
6. For rollback, revert the desired image tag through Git and manually sync prod again; do not hotpatch Pods as an alternative source of truth.

**Lab example:** Production-like app was on `36f432a`; promotion of tested Vault-reading image `56c38ee` yielded `vault_config_loaded: true`.
