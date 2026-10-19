# Vault operations: Raft failover and manual backup

## Raft inspection

```bash
kubectl get pods,statefulset,pvc -n vault
vault operator raft list-peers
```

A leadership transfer via `vault operator step-down` moved leadership from `vault-0` to `vault-1`; the cluster retained three voting peers. This validates leader transfer, not node-failure protection. If a Shamir-sealed Vault Pod restarts, it requires two distinct original unseal shares. Do not reinitialize existing storage.

## Manual snapshot (Mac, local-only Vault port-forward)

```bash
export VAULT_ADDR=http://127.0.0.1:8200
mkdir -p "$HOME/vault-backups/lab02"
chmod 700 "$HOME/vault-backups/lab02"
umask 077
BACKUP_FILE="$HOME/vault-backups/lab02/vault-raft-$(date +%Y%m%d-%H%M%S).snap"
vault operator raft snapshot save "$BACKUP_FILE"
vault operator raft snapshot inspect "$BACKUP_FILE"
shasum -a 256 "$BACKUP_FILE" > "$BACKUP_FILE.sha256"
shasum -a 256 -c "$BACKUP_FILE.sha256"
```

The command requires an authenticated, sufficiently privileged operator; enter credentials interactively. Keep the snapshot and Shamir key shares secure and separate from Git. The SHA-256 verifies file integrity, not recoverability.

## Restore caution

**Do not restore onto the running lab Vault cluster casually.** `vault operator raft snapshot restore` overwrites storage state. Test restoration only on an isolated recovery cluster after preserving compatible seal material and following the official recovery procedure. No automated Kubernetes CronJob, additional PVC, or retention process was implemented in this project.
