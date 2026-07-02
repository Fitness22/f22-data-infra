# MongoDB Atlas to Self-Hosted GKE Migration Plan

## Goal

Move the current MongoDB workload from MongoDB Atlas to a self-hosted MongoDB replica set running in a dedicated GCP Kubernetes cluster, while preserving availability, backups, and a controlled rollback path.

This plan is optimized for:

- lower monthly infra cost than Atlas M30
- predictable operations
- minimal application downtime during cutover
- compatibility with existing Fitness22 Kubernetes deployment patterns

## Current Assumptions

- Current Atlas tier is roughly equivalent to M30
- Data size is about 363 GB
- Required storage headroom should target at least 2x working set growth buffer during migration
- The team is willing to own backups, failover validation, upgrades, and restore testing
- The MongoDB workload is primarily used by data-oriented services, not the CRM SPA
- Airbyte may need to connect to this MongoDB after migration

If any of these are wrong, adjust the plan before implementation.

## Recommended Target Architecture

### Cluster

Create a dedicated GKE cluster for stateful data services instead of mixing MongoDB into the existing app cluster.

Recommended baseline:

- GKE Standard, not Autopilot
- Regional control plane
- Zonal worker placement with explicit cross-zone replica scheduling
- 3 availability zones in one GCP region
- private cluster
- VPC-native
- Workload Identity enabled

Reasoning:

- MongoDB needs predictable storage and node placement
- Standard mode gives more control over disks, taints, disruption budgets, and upgrades
- Dedicated cluster reduces noisy-neighbor risk and simplifies blast radius control

### Node Pools

Use separate node pools.

1. `mongodb-pool`
- machine type: start with `c4-standard-4` or `c4-standard-8`
- autoscaling: disabled or very tightly bounded
- tainted for MongoDB only
- local SSD: not required initially
- minimum 3 nodes

2. `ops-pool`
- small general-purpose nodes for operator, exporters, backup jobs, and supporting agents

Initial recommendation:

- start with `c4-standard-4` if Mongo memory pressure is low to moderate
- choose `c4-standard-8` only if working set or concurrent analytical usage is materially high

### MongoDB Topology

Use a 3-member replica set.

- 3 data-bearing members
- 1 primary
- 2 secondaries
- no arbiter

Scheduling rules:

- one member per zone
- anti-affinity required
- PodDisruptionBudget enabled
- topology spread constraints enforced

### Storage

Use persistent volumes per replica member.

Recommended starting point:

- disk type: Hyperdisk Balanced or `pd-ssd`
- size per member: 500 GB to start
- provision independently per replica member

Why 500 GB each:

- current usage is already ~363 GB
- migration, index rebuilds, oplog growth, and backups need headroom
- starting too close to current size will create operational pressure immediately

### Access Pattern

Expose MongoDB only privately.

- no public MongoDB endpoints
- access from apps via private VPC networking
- if another cluster must connect, use private connectivity and firewall allowlists
- use TLS for intra-cluster and client connections

## Software Stack Recommendation

Prefer the MongoDB Community Kubernetes Operator unless you already have a strong reason to manage raw StatefulSets yourselves.

Use:

- MongoDB Community Operator
- Percona `mongodb_exporter` or equivalent metrics exporter
- GKE managed Prometheus or existing Prometheus stack
- GCS-based logical backups plus persistent-disk snapshots

Avoid on day 1:

- sharding
- hidden members
- custom backup pipelines
- cross-region replication

## Security Model

### Secrets

Store all MongoDB credentials and TLS materials in Kubernetes secrets, ideally sourced from Google Secret Manager through your preferred sync mechanism.

Secrets to manage:

- admin credentials
- application user credentials
- Airbyte or ETL credentials
- replica set keyfile or internal auth material
- TLS certs

### Network

- private nodes and private control plane access rules
- deny public ingress to MongoDB
- allow only app namespaces or approved client CIDRs
- restrict egress where practical

### Access Separation

Create separate MongoDB users for:

- application read/write
- analytics or ETL
- backup and restore automation
- admin operations

Do not reuse one broad admin credential everywhere.

## Backup and Restore Strategy

This is the part Atlas used to do for you automatically, so be strict here.

Use two backup layers:

1. Volume snapshots
- frequent disk snapshots for fast node-level recovery
- keep short retention for operational restore

2. Logical backups
- `mongodump` or `mongosync`-compatible exports to GCS
- daily full backup
- more frequent oplog-aware backup if RPO requires it

Minimum target:

- RPO: 1 hour
- RTO: 4 hours

Operational requirement:

- run restore drills in non-production before production cutover
- document restore steps before migration day

## Monitoring and Alerting

Before cutover, have alerts for:

- replica health
- replication lag
- primary elections
- disk usage
- memory pressure
- cache eviction pressure
- connection saturation
- slow queries
- backup failures

Dashboards should show:

- opcounters
- resident memory
- WiredTiger cache pressure
- replication lag
- storage consumption growth
- ticket utilization if available

## Migration Strategy

Use a staged migration with a rollback window.

### Phase 1: Discovery

- confirm exact Atlas topology, version, and feature set
- list all clients connecting to Atlas
- capture baseline metrics: ops/sec, connections, CPU, RAM, working set, read/write ratio
- identify indexes and large collections
- confirm whether change streams are used anywhere
- confirm whether Airbyte or other ETL tools depend on Atlas-specific connection settings

### Phase 2: Build Target

- create dedicated GKE cluster
- create MongoDB node pool and ops pool
- install operator
- deploy staging replica set
- enable TLS and authentication
- set up monitoring, alerting, and backups
- validate zone-aware scheduling

### Phase 3: Rehearse

- restore a production-like snapshot into staging
- run application smoke tests against staging
- measure query latency and memory pressure
- verify backup creation and restore
- test node failure and primary election

### Phase 4: Production Sync

Choose one of these paths.

Option A: Continuous sync tool
- preferred if downtime must be minimal
- use `mongosync` or another supported migration mechanism
- keep target continuously synced until cutover

Option B: Dump and restore
- acceptable if the maintenance window is larger
- take final application write pause
- run final dump
- restore into GKE
- validate counts and indexes

For this workload size, Option A is safer if near-zero downtime matters.

### Phase 5: Cutover

1. reduce write traffic where possible
2. freeze or pause non-critical writers
3. wait for sync to catch up
4. switch application secrets and connection strings
5. run smoke tests
6. monitor for replication, latency, and auth issues
7. keep Atlas intact during rollback window

### Phase 6: Rollback Window

For at least 3 to 7 days:

- do not delete Atlas
- keep rollback procedure ready
- compare data drift and error rates
- only decommission Atlas after stable production validation

## Client and Application Changes

### Connection Strings

Applications and tools must use a self-managed replica set connection string, not Atlas SRV assumptions.

Expected pattern:

```text
mongodb://host1:27017,host2:27017,host3:27017/<db>?replicaSet=<rs-name>&authSource=admin&tls=true
```

This matters for Airbyte too. The repository already contains a self-managed replica-set connection model in the saved Airbyte artifacts, so that dependency looks compatible with this migration.

### DNS

Create stable private DNS names so clients are not pinned to pod IPs.

Recommended:

- one stable name per replica member
- one application-facing service alias if needed

### Secrets Rotation

Plan for:

- dual credentials during migration if possible
- post-cutover credential rotation

## Cost-Conscious Defaults

To keep savings real, use these defaults first:

- 3 Mongo nodes only
- no extra hidden member on day 1
- moderate SSD class, not maxed IOPS
- one dedicated cluster, but small ops pool
- committed use discount only after workload stabilizes

Do not optimize too early by undersizing disks or RAM. That usually costs more later.

## Risks

### Highest-risk areas

- backups exist but restores are untested
- storage headroom is too small
- cross-zone scheduling is misconfigured and two replicas land together
- clients rely on Atlas-specific DNS or TLS behavior
- maintenance work is underestimated after go-live
- Airbyte or data pipelines keep old connection strings

## Definition of Done

The migration is complete only when all of these are true:

- production reads and writes run against GKE MongoDB
- backups are automated and verified
- restore drill completed successfully
- monitoring and alerting are active
- failover test completed
- all client connection strings updated
- Atlas decommission approval is explicitly given

## Concrete Next Actions

1. Inventory every Atlas client and connection string owner.
2. Pull Atlas metrics for 7 to 14 days and size the node pool from real usage.
3. Build the dedicated GKE cluster and MongoDB staging replica set.
4. Decide the migration method: `mongosync` vs dump/restore.
5. Rehearse restore and failover before touching production.
6. Prepare cutover runbook and rollback runbook.

## Repo Notes

Relevant repo context discovered during planning:

- [f22cloud-infra/AGENTS.md](/Users/maorshabo/Fitness22 Dropbox/Maor Shabo/Projects/f22-cloud-repos/f22cloud-infra/AGENTS.md) shows the existing team pattern is Helm-based Kubernetes operations.
- [f22-crm-dashboard/AGENTS.md](/Users/maorshabo/Fitness22 Dropbox/Maor Shabo/Projects/f22-cloud-repos/f22-crm-dashboard/AGENTS.md) confirms the CRM dashboard is not the MongoDB workload.
- [f22-data-infra/backups/cluster-prerequisites.md](/Users/maorshabo/Fitness22 Dropbox/Maor Shabo/Projects/f22-cloud-repos/f22-data-infra/backups/cluster-prerequisites.md) indicates there is already GCP-oriented data infrastructure work in this repo.
- [f22-data-infra/backups/airbyte-db-backup.sql](/Users/maorshabo/Fitness22 Dropbox/Maor Shabo/Projects/f22-cloud-repos/f22-data-infra/backups/airbyte-db-backup.sql) includes metadata showing Airbyte supports self-managed MongoDB replica set connections, which reduces migration risk for that path.
