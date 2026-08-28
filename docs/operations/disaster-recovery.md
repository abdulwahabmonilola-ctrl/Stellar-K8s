# Disaster Recovery & Quorum Loss Runbook

**Audience:** On-call operators and SREs running Stellar-K8s validator fleets.
**Scope:** Single-node and fleet-level recovery. Three scenarios:

1. Complete Pod failure
2. Corrupted PVC recovery
3. Total quorum loss (forced sync from history archives)

**Format:** Each scenario follows **Symptom → Diagnosis → Mitigation → Resolution**.
Commands assume the `kubectl stellar` plugin is installed (see
[docs/kubectl-plugin.md](../kubectl-plugin.md)) and the operator is the current
stable release.

> **Golden rule — never fork the ledger.** If you are unsure whether a node is
> caught up, do **not** start it with stale state against a live network. Start
> in Recovery Mode first (Scenario 2 / [Recovery Mode manifest](#recovery-mode)).
> A node that signs with an old ledger hash can create a fork and cost the
> network real money.

---

## Pre-Incident Checklist

Have these ready *before* you need them:

```bash
# The current StellarNode inventory
kubectl stellar list -A

# Quorum + sync snapshot of every node
kubectl stellar summary -A
kubectl stellar status -A

# Verify backups exist and are recent (per node)
kubectl get volumesnapshots -A -l stellar.org/snapshot-of=<node-name>
kubectl get forensicsnapshots -A

# Who is the seed secret? (you must be able to retrieve it for a rebuild)
kubectl get secret <seed-secret-name> -n <namespace> \
  -o jsonpath='{.data.STELLAR_CORE_SEED}' | base64 -d | head -c 4
# ^ never print the whole seed
```

Store your seed secret in a vault (Vault/AWS KMS) and **test the restore path in
staging at least once per quarter**. A runbook you have never executed is a wish
list.

---

## Scenario 1 — Complete Pod Failure

### Symptom

- `kubectl stellar summary` shows the node as `Degraded` or `Pending`.
- PagerDuty/Alertmanager fires `AnchorPointApiTargetDown`-style alerts for the
  validator pod.
- Stellar Core peers stop hearing from the node; quorum begins shedding.

### Diagnosis

```bash
# 1. What does the operator think is wrong?
kubectl stellar status <node-name>
kubectl stellar events <node-name> --watch   # watch in a second terminal

# 2. What does Kubernetes think happened? (CrashLoopBackOff, ImagePullBackOff, OOMKilled, Evicted...)
kubectl get pods -l stellar.org/managed-node=<node-name> -n <namespace> -o wide
kubectl describe pod -l stellar.org/managed-node=<node-name> -n <namespace> | tail -80

# 3. Why did the process die?
kubectl stellar logs <node-name> -c stellar-core --tail 200
kubectl stellar logs <node-name> -c stellar-core --tail 500 | grep -E "FATAL|ERROR|panic|abort"

# 4. Explain any Stellar error codes in the logs
kubectl stellar explain <error-code>
```

**Expected outcome:** a root cause (OOM, config error, node eviction, or a
transient restart). If the pod is stuck in `CrashLoopBackOff`, capture the core
dump first — see [docs/forensic-snapshot.md](../forensic-snapshot.md).

### Mitigation

For a **transient failure** (node drain, spot reclaim, one-off OOM):

```bash
# Scale to 0 cleanly, then let the operator bring it back
kubectl patch stellarnode <node-name> -n <namespace> --type=merge -p '{"spec":{"suspended":true}}'
kubectl rollout status statefulset/<node-name> -n <namespace> --timeout=120s
kubectl patch stellarnode <node-name> -n <namespace> --type=merge -p '{"spec":{"suspended":false}}'
```

If the pod is **unhealthy but present**, force a clean restart:

```bash
kubectl delete pod -l stellar.org/managed-node=<node-name> -n <namespace>
kubectl stellar status <node-name>   # wait for phase: Ready
```

### Resolution

- Confirm the node rejoined consensus **without** re-syncing:
  ```bash
  kubectl stellar status <node-name>
  kubectl stellar logs <node-name> -c stellar-core --tail 50 | grep -i "ledger closed"
  ```
- Verify it is signing and within a few ledgers of the network.
- File a post-mortem using [docs/incident-response/post-mortem.md](../incident-response/post-mortem.md).
- If this pod is killed repeatedly, it is **not** a pod failure — it is capacity
  or scheduling. Move to Scenario 2 only if data is involved.

---

## Scenario 2 — Corrupted PVC Recovery

### Symptom

- Pod starts but `stellar-core` logs corruption errors:
  `database is not consistent`, `corrupt`, `fsync failed`, or `checkpoint` failures.
- `kubectl stellar status` stays `Creating`/`Syncing` and never reaches `Ready`.
- Disk I/O errors in `dmesg`/events, or the CSI driver reports an unhealthy volume.

### Diagnosis

```bash
kubectl stellar status <node-name>
kubectl stellar events <node-name> --watch
kubectl stellar logs <node-name> -c stellar-core --tail 300 | grep -iE "corrupt|integrity|fsync|sqlite|database"

# Inspect the data volume read-only (never mount rw while investigating)
kubectl get pvc -l stellar.org/managed-node=<node-name> -n <namespace> -o wide
kubectl get volumesnapshots -A -l stellar.org/snapshot-of=<node-name>

# Check the node's app database is reachable (read-only query through the plugin)
kubectl stellar sql <node-name> "SELECT 1;" -o json
```

**Decision point:** is the corruption limited to the DB (recover from a
snapshot/archive) or the PVC itself (recreate the volume)? If the CSI snapshot
is healthy, prefer **snapshot restore** (fast). If no snapshot exists, fall back
to **archive re-sync** (slow but always valid).

### Mitigation — Recovery Mode

Put the node in Recovery Mode *before* touching data: disable the readiness
probe (so the operator stops kicking an unhealthy pod), suspend normal traffic,
and run with read-only DB checks. Apply
[`examples/recovery-mode-validator.yaml`](../../examples/recovery-mode-validator.yaml)
or patch the live node:

```bash
kubectl patch stellarnode <node-name> -n <namespace> --type=merge -p '
{
  "spec": {
    "suspended": true,
    "maintenanceMode": true,
    "probes": {
      "readiness": { "failureThreshold": 100 },
      "liveness":   { "failureThreshold": 100 }
    }
  }
}'
```

This keeps the pod around for diagnosis without the operator marking it Ready.

**Option A — restore from CSI VolumeSnapshot (fastest, preferred):**

```bash
# 1. Confirm the newest snapshot is < your RPO
kubectl get volumesnapshots -A -l stellar.org/snapshot-of=<node-name> --sort-by=.metadata.creationTimestamp

# 2. Point a fresh StellarNode at it (name must differ from the broken one)
kubectl apply -f - <<'EOF'
apiVersion: stellar.org/v1alpha1
kind: StellarNode
metadata:
  name: <node-name>-restored
  namespace: <namespace>
spec:
  nodeType: Validator
  network: <network>           # e.g. Mainnet
  version: "v21.0.0"
  storage:
    storageClass: <your-ssd-class>
    size: 500Gi
  restoreFromSnapshot:
    volumeSnapshotName: <snapshot-name>   # e.g. <node-name>-data-20250224-020000
  validatorConfig:
    seedSecretRef: <seed-secret-name>     # same seed! otherwise the key changes
    enableHistoryArchive: true
    historyArchiveUrls:
      - "https://history.stellar.org/prd/core-live/core_live_001"
EOF

# 3. Watch it catch up (snapshot restore is near-instant)
kubectl stellar status <node-name>-restored
watch -n 10 kubectl stellar status <node-name>-restored   # or poll in a loop
```

**Option B — re-sync from history archives (slow, always valid):**

When no snapshot exists. Wipe the data volume and let the node sync from the
archive:

```bash
# 1. Delete the broken node (retain the PVC per retentionPolicy: Retain)
kubectl delete stellarnode <node-name> -n <namespace> --wait=false

# 2. Delete ONLY the corrupt data PVC (do not delete the backup snapshots)
kubectl delete pvc <node-name>-data -n <namespace> --wait=false
kubectl get pvc -n <namespace>   # confirm gone

# 3. Recreate the node (fresh volume + same seed + archives)
kubectl apply -f examples/validator-mainnet.yaml   # adjust name/seed/network

# 4. Watch the long catch-up (can take hours on Mainnet)
kubectl stellar status <node-name>
kubectl stellar logs <node-name> -c stellar-core -f | grep -i "ledger"
```

### Resolution

```bash
kubectl stellar status <node-name>          # phase: Ready, sync state: Synced
kubectl stellar summary
kubectl stellar sql <node-name> "SELECT 1;" -o json   # app DB reachable
```

- Verify the last ledger is within ~1–3 ledgers of the network leader.
- Remove the Recovery Mode overrides once healthy (clear `maintenanceMode`
  and probe overrides):
  ```bash
  kubectl patch stellarnode <node-name> -n <namespace> --type=merge \
    -p '{"spec":{"maintenanceMode":false,"probes":null}}'
  ```
- **Critical:** confirm the node has not signed a forked ledger. If it restarted
  from an old snapshot on a live network, the archive re-sync (Option B) is the
  only safe path.

---

## Scenario 3 — Total Quorum Loss (forced sync from archives)

### Symptom

- **All** of your validators are down or desynced; SCP cannot reach quorum.
- `kubectl stellar summary` shows every node `Degraded`/`Pending`.
- Stellar Core logs `Not enough nodes to reach agreement` / `SCP ... timed out`.
- Network-wide symptoms may also appear if you are a large portion of the quorum set.

### Diagnosis

```bash
kubectl stellar summary -A
kubectl stellar status -A
kubectl stellar events -A --watch

# Which nodes are actually alive?
kubectl get stellarnodes -A -o wide

# Is the app DB present and internally consistent on each survivor?
for n in <node1> <node2> <node3>; do
  echo "== $n =="
  kubectl stellar sql "$n" "SELECT 1;" -o json
done
```

**Critical question:** do the survivors agree on a ledger sequence, or have they
diverged? Divergent DBs + a live network = fork risk. Do not let them all start
simultaneously.

### Mitigation

1. **Freeze.** Suspend **every** managed validator so nothing signs:
   ```bash
   kubectl get stellarnodes -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | \
     while read n; do kubectl patch stellarnode "$n" -A --type=merge -p '{"spec":{"suspended":true}}'; done
   ```

2. **Identify the canonical ledger.** Pick the survivor with the highest,
   internally-consistent ledger as the source of truth (best: the newest CSI
   snapshot across the fleet):
   ```bash
   kubectl get volumesnapshots -A --sort-by=.metadata.creationTimestamp | tail -20
   ```

3. **Re-establish quorum from the canonical state.** Bring the canonical node up
   first and let it reach `Ready`; only then bring peers back one at a time.
   ```bash
   kubectl patch stellarnode <canonical> -A --type=merge -p '{"spec":{"suspended":false}}'
   kubectl stellar status <canonical>              # poll until phase: Ready
   kubectl stellar events <canonical> -A --watch   # live view of reconcile events
   ```

4. **Forced archive re-sync for any node that cannot trust its own DB.** Follow
   Scenario 2 / Option B, but *stagger* the restarts. Re-sync from archives:
   - SDF Mainnet: `https://history.stellar.org/prd/core-live/core_live_001`
   - SDF Testnet: `https://history.stellar.org/prd/core-testnet/core_testnet_001`

   ```bash
   for n in <peer1> <peer2> <peer3>; do
     kubectl patch stellarnode "$n" -A --type=merge -p '{"spec":{"suspended":false}}'
     until kubectl stellar status "$n" -o json | grep -q '"phase": "Ready"'; do sleep 10; done
   done
   ```

### Resolution

```bash
kubectl stellar summary -A        # all nodes Ready + Synced
kubectl stellar status -A
kubectl stellar events -A | tail -20
```

- Confirm each node is within a few ledgers of the network and **all agree on
  the current ledger sequence**.
- Watch for at least one full quorum window (a few ledger closes) with no
  `SCP` timeouts.
- Only then re-enable any automation (quorum optimizer, auto-failover, DR
  drills).

---

## Recovery Mode

A Recovery Mode StellarNode bypasses normal readiness/liveness gating so the
operator leaves a broken pod alone while you diagnose it. Use it **before**
touching data in Scenario 2.

Reference manifest:
[`examples/recovery-mode-validator.yaml`](../../examples/recovery-mode-validator.yaml)

Key differences from a normal validator:
- `maintenanceMode: true` — operator keeps the object but stops treating it as Ready.
- `probes.readiness.failureThreshold` and `probes.liveness.failureThreshold` set
  very high so the pod is never restarted/killed mid-diagnosis.
- `suspended: true` in the manifest if you want it scaled to 0 while you inspect.
- Optional `restoreFromSnapshot` to bootstrap from a known-good CSI snapshot.

> **Always** remove `maintenanceMode` and probe overrides after recovery.

---

## Verification / DR Drill

Run quarterly, in staging first:

```bash
# Simulate: suspend a node, corrupt nothing, restore from snapshot
kubectl patch stellarnode <node> --type=merge -p '{"spec":{"suspended":true}}'
kubectl stellar status <node>
kubectl patch stellarnode <node> --type=merge -p '{"spec":{"suspended":false}}'

# Time it: snapshot restore should be minutes, archive re-sync tracked in hours
start=$(date +%s)
until kubectl stellar status <node> -o json | grep -q '"phase": "Ready"'; do sleep 10; done
echo "RTO: $(( $(date +%s) - start ))s"
```

Log actual restore times (RTO) and compare against your SLO. Update this runbook
if any command changed in a release.

---

## References

- [kubectl stellar plugin reference](../kubectl-plugin.md)
- [Volume snapshots & restore](../volume-snapshots.md)
- [DR failover between regions](dr-failover.md)
- [Forensic snapshots](../forensic-snapshot.md)
- [Quorum set optimization](../quorum-optimization.md)
- [Incident post-mortem template](../incident-response/post-mortem.md)
- Stellar Core admin guide: <https://developers.stellar.org/docs/validators/admin-guide>
