# Portfolio evidence checklist

Capture real screenshots or sanitized outputs. Avoid tokens, JWTs, unseal keys, `/vault/secrets/config.json` contents, root login sessions, private hosts/IPs, and snapshot data.

- [ ] GitHub Actions tests/build/tag-push and update to GitOps repo
- [ ] Argo CD dev `Synced/Healthy`, prod manual sync configuration
- [ ] Helm values split for dev/prod and prod image promotion PR
- [ ] Dev HPA scale from 1→4→1, with relevant Grafana graph
- [ ] Prometheus ServiceMonitor and API metrics dashboard
- [ ] `vault operator raft list-peers`: 3 voters, one leader
- [ ] Leadership change before/after (`vault-0` → `vault-1`)
- [ ] Dev and both prod Pods showing Agent injection containers
- [ ] `test -s /vault/secrets/config.json` success for all target Pods
- [ ] `/api/info` showing prod environment, `vault_config_loaded: true`, version `56c38ee`
- [ ] Sanitized 403 for dev-role attempt to access prod secret
- [ ] Manual snapshot inspection and checksum `OK`
- [ ] Optional: independent prod-role authorization denial test and isolated restore exercise (not yet demonstrated)
