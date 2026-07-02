# MongoDB Atlas → Self-Hosted GKE Migration Plan (Detailed)

## Summary

| Parameter | Value |
|---|---|
| Atlas tier | M30 (2 vCPU, 8 GB RAM) |
| Data size | ~363 GB, growing ~5-20 GB/month |
| Current cost | ~$800+/mo |
| Workload | Write-focused, low volume, no change streams |
| Downtime tolerance | Near-zero (mongosync) |
| Writer service | Currently on DO, will connect cross-cloud initially |
| GCP region | us-east1 |
| Timeline | 1-2 months |
| Team K8s experience | Experienced (Helm, operators, StatefulSets) |

---

## Phase 1: GKE Cluster Provisioning (~Week 1)

Create a dedicated GKE Standard cluster in `us-east1`:

```
Cluster: f22-data-gke-prod
Mode: GKE Standard (not Autopilot)
Region: us-east1 (regional control plane)
Network: Existing VPC, VPC-native
Private cluster: yes (private nodes, authorized networks for control plane)
Workload Identity: enabled
Release channel: Regular
```

### Node Pools

| Pool | Machine Type | Nodes | Purpose |
|---|---|---|---|
| `mongodb-pool` | `e2-standard-4` (4 vCPU, 16 GB) | 3 (one per zone) | MongoDB replicas |
| `ops-pool` | `e2-small` (0.5 vCPU, 2 GB) | 1-2 | Operator, exporter, backup jobs |

**Why `e2-standard-4` instead of `c4-standard-4`:**

- The workload is low-volume writes, not CPU-intensive
- e2 is ~30-40% cheaper than C4 for the same spec
- 16 GB RAM (vs Atlas M30's 8 GB) gives comfortable headroom for WiredTiger cache with 363 GB dataset
- Can resize later if needed

### Storage

| Per replica | Spec |
|---|---|
| Disk type | `pd-ssd` (SSD persistent disk) |
| Size | 500 GB each |
| StorageClass | `premium-rwo` |

**Why 500 GB:** Current 363 GB + migration headroom + oplog + 12+ months growth at 20 GB/mo.

---

## Phase 2: MongoDB Deployment (~Week 1-2)

### Operator

MongoDB Community Kubernetes Operator

- Namespace: `mongodb`
- Operator scheduled on `ops-pool` via node selector
- MongoDB replicas on `mongodb-pool` via taints/tolerations

### Replica Set Configuration

```
Members: 3 (data-bearing, no arbiter)
Version: Match Atlas version (verify first — likely 7.0 or 6.0)
Topology: 1 member per zone (anti-affinity + topology spread)
PDB: minAvailable: 2
Auth: SCRAM-SHA-256
TLS: enabled (internal + client)
```

### Storage per member

```yaml
storageClass: premium-rwo
size: 500Gi
```

### Users to create

1. `admin` — cluster admin (break-glass only)
2. `app-writer` — write service credentials
3. `backup-agent` — backup automation
4. `monitoring` — exporter read-only

### Secrets

Create via `kubectl create secret generic`, annotate with `helm.sh/resource-policy: keep` (matching existing DO pattern). Do NOT embed in Helm values.

---

## Phase 3: Network Access (~Week 2)

**Approach:** Public MongoDB endpoint with strict firewall rules (temporary, until writer service migrates to GCP).

- Create a `LoadBalancer` service pointing to the MongoDB primary (or headless service for replica-set-aware clients)
- GCP firewall rule: allow inbound on port 27017 **only from the DO cluster's egress IPs**
- TLS required for all connections
- MongoDB user auth (SCRAM) as second layer
- Remove public access once the writer migrates to GKE

This is acceptable as a temporary measure because:

- TLS + SCRAM auth provides strong transport and authentication security
- IP-restricted firewall limits exposure to known sources only
- The public endpoint is temporary (until writer moves to GCP)

### Connection String Pattern

```text
mongodb://host1:27017,host2:27017,host3:27017/<db>?replicaSet=<rs-name>&authSource=admin&tls=true
```

### DNS

Create stable DNS names (e.g., `mongo-0.data.fitness22services.com`) so clients are not pinned to LoadBalancer IPs.

---

## Phase 4: Observability (~Week 2-3)

### Monitoring Stack (on `ops-pool`)

1. **MongoDB Exporter** (Percona `mongodb_exporter`)
   - Scrapes MongoDB metrics
   - Exposes `/metrics` endpoint

2. **Prometheus** — GKE Managed Prometheus (Google Cloud Managed Service for Prometheus)
   - Zero maintenance
   - Free for first 50M samples ingested/mo
   - Can switch to self-hosted kube-prometheus-stack later if needed

3. **Grafana** — deploy on ops-pool or use Google Cloud Monitoring dashboards

### Alerts

| Alert | Condition |
|---|---|
| `MongoReplicaSetUnhealthy` | Fewer than 3 members |
| `MongoReplicationLag` | Secondary lag > 10s |
| `MongoPrimaryElection` | Primary changed |
| `MongoDiskUsage` | > 75% capacity |
| `MongoConnectionSaturation` | > 80% max connections |
| `MongoBackupFailed` | Backup job failed |
| `MongoSlowQueries` | Operations > 100ms threshold |

### Dashboard Panels

- opcounters (insert/update/delete/query)
- WiredTiger cache usage vs configured
- Replication lag per secondary
- Disk usage growth
- Connection pool utilization
- Resident memory

---

## Phase 5: Backup Strategy (~Week 2-3)

### Two-Layer Backup

| Layer | Method | Frequency | Retention | RTO |
|---|---|---|---|---|
| Volume snapshots | GCE disk snapshots via CronJob | Every 6 hours | 7 days | ~30 min |
| Logical backup | `mongodump` to GCS bucket | Daily | 30 days | ~2-4 hours |

### Implementation

1. **Volume snapshots**: K8s CronJob using `gcloud compute disks snapshot` (runs on ops-pool)
2. **Logical backup**: K8s CronJob running `mongodump --gzip --archive` → upload to GCS bucket `f22-mongodb-backups`
3. **GCS bucket**: Standard storage class, lifecycle policy to move to Nearline after 7 days, delete after 30 days

### Targets

- RPO: 6 hours (snapshot) / 1 day (full logical)
- RTO: 30 min (snapshot) / 4 hours (logical restore)

### Mandatory Pre-Cutover

Run a full restore drill from both snapshot and logical backup into a test namespace.

---

## Phase 6: Rehearsal (~Week 3)

Before touching production data:

1. Deploy the full stack (operator + replica set + exporter + backups) in a `mongodb-staging` namespace
2. Load a representative dataset (~50-100 GB sample from Atlas `mongodump`)
3. Run the writer service against staging
4. Verify:
   - Write latency from DO → GCP (expect ~20-40ms)
   - Backup creation and restore
   - Kill a pod → verify automatic failover + PDB behavior
   - Kill a node → verify rescheduling
   - Alert firing on simulated failures
5. Size validation: confirm memory pressure, disk I/O, and CPU are within bounds

---

## Phase 7: Production Migration (~Week 4-5)

### Method: `mongosync` (near-zero downtime)

#### Pre-requisites

- Verify Atlas MongoDB version (mongosync requires 6.0+ on both sides)
- Atlas must allow the GKE cluster's egress IP to connect
- GKE MongoDB must allow Atlas's sync connection

#### Steps

1. **Start mongosync** — continuous sync from Atlas → GKE replica set
   - mongosync runs as a pod on the ops-pool
   - Syncs all collections, indexes, and data
   - Initial sync: ~363 GB, expect 12-24 hours depending on bandwidth
   - After initial sync: continuous oplog tailing keeps target in sync

2. **Validate sync state**
   - Compare document counts per collection
   - Compare index lists
   - Run application smoke tests against GKE (read-only)

3. **Cutover (maintenance window of ~5-15 minutes)**
   - Pause/reduce writes from the writer service
   - Wait for mongosync lag to reach 0
   - Run `mongosync commit` to finalize
   - Update writer service connection string (new Kubernetes secret)
   - Restart writer service pods
   - Verify writes land in GKE MongoDB
   - Monitor for errors

4. **Post-cutover validation**
   - Confirm all writes go to GKE
   - Verify replication health across 3 members
   - Check backup jobs are running
   - Verify alerts are firing correctly on test scenarios

---

## Phase 8: Rollback Window (~Week 5-7)

- **Keep Atlas running for 7 days** after cutover
- If critical issues: revert connection string back to Atlas
- Compare error rates, latency, and data integrity daily
- After 7 stable days: snapshot GKE data, then decommission Atlas

---

## Cost Summary

| Item | Monthly Cost |
|---|---|
| 3x `e2-standard-4` (mongodb-pool) | ~$290 |
| 1x `e2-small` (ops-pool) | ~$15 |
| GKE cluster management fee | $0 (Standard, no Autopilot surcharge) |
| 3x 500 GB pd-ssd | ~$255 |
| GCS backup storage (~500 GB) | ~$10-15 |
| Network egress (DO <> GCP, temporary) | ~$10-20 |
| GKE Managed Prometheus | ~$0 (under free tier) |
| LoadBalancer IP | ~$18 |
| **Total** | **~$600-615/mo** |
| **Atlas current** | **~$800+/mo** |
| **Savings** | **~$200+/mo (~25%)** |

After the writer moves to GCP and the LoadBalancer is removed, savings increase. With 1-year committed use discounts (CUD) on the VMs, total could drop to ~$450/mo.

---

## Deliverables & File Structure

```
f22-data-infra/
├── mongodb-gke-migration-plan.md              # Original high-level plan
├── mongodb-gke-migration-plan-detailed.md     # This document
├── mongodb/
│   ├── gke-cluster-setup.sh                   # gcloud commands to create cluster
│   ├── operator/
│   │   └── install.sh                         # Install MongoDB Community Operator
│   ├── manifests/
│   │   ├── namespace.yaml
│   │   ├── mongodb-replicaset.yaml            # MongoDBCommunity CR
│   │   ├── mongodb-users-secret.yaml          # Template (not committed with values)
│   │   ├── tls-secret.yaml
│   │   ├── pdb.yaml
│   │   └── loadbalancer-service.yaml          # Temporary public access
│   ├── monitoring/
│   │   ├── exporter-deployment.yaml
│   │   ├── servicemonitor.yaml
│   │   └── prometheusrules.yaml
│   ├── backups/
│   │   ├── snapshot-cronjob.yaml
│   │   ├── mongodump-cronjob.yaml
│   │   └── gcs-bucket-setup.sh
│   ├── migration/
│   │   ├── mongosync-pod.yaml
│   │   ├── cutover-runbook.md
│   │   └── rollback-runbook.md
│   └── scripts/
│       ├── deploy-mongodb.sh
│       ├── verify-replication.sh
│       └── restore-drill.sh
```

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| mongosync incompatible with Atlas version | Verify Atlas version first; fallback to mongodump/restore with ~1hr window |
| Cross-cloud latency impacts writer | Monitor p99 latency; temporary — resolves when writer moves to GCP |
| Public endpoint exposure | Strict IP allowlist + TLS + SCRAM; temporary until writer migrates |
| Backup restores untested | Mandatory restore drill in Phase 6 before any production cutover |
| Disk undersizing | Starting at 500 GB/member (38% headroom + growth buffer) |
| Zone scheduling misconfigured | Anti-affinity + topology spread constraints; verified in rehearsal |

---

## Concrete Next Steps (Ordered)

1. **Verify Atlas MongoDB version** — determines mongosync compatibility and target version
2. **Inventory all Atlas connection strings** — which services/tools connect and where they get their connection string
3. **Provision the GKE cluster** — `gcloud` commands, ~30 min
4. **Deploy MongoDB Community Operator + staging replica set** — ~2 hours
5. **Set up monitoring + backups** — ~1 day
6. **Rehearsal with sample data** — ~2 days
7. **mongosync production migration** — ~1-2 days (sync + cutover)
8. **Rollback window** — 7 days monitoring

---

## Open Decisions

These should be resolved before implementation begins:

1. **GKE Managed Prometheus vs self-hosted kube-prometheus-stack?** — Plan defaults to GKE Managed for simplicity
2. **Node sizing: `e2-standard-2` (match Atlas M30) vs `e2-standard-4` (headroom)?** — Plan defaults to e2-standard-4
3. **MongoDB version: latest 7.0 or match Atlas exactly?** — Depends on Atlas version check
