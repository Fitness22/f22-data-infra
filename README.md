# f22-data-infra

Infrastructure-as-config for the Fitness22 data platform running on GKE. This repo holds the Helm values, cluster setup docs, and migration plans for the data stack — there is no application code, build system, or test runner here.

## What runs on the cluster

**Cluster:** `f22-data-gke-prod` — regional, private GKE cluster. All external-facing services are exposed through `ingress-nginx` with TLS certificates issued by `cert-manager` (Let's Encrypt, `letsencrypt-prod` ClusterIssuer). Services run plain HTTP internally; TLS terminates at the ingress.

| Service | Namespace | URL | Config |
|---|---|---|---|
| Airbyte 2.0.19 (Community) | `airbyte-v2` | https://airbyte.fitness22services.com | [`airbyte/values.yaml`](airbyte/values.yaml) |
| Airflow 2.9.3 | `airflow` | https://airflow.fitness22services.com | [`airflow/values.yaml`](airflow/values.yaml) |
| MongoDB (planned) | — | — | see migration plans below |

## Repository layout

```
airbyte/values.yaml                    Helm values for the Airbyte chart
airflow/values.yaml                    Helm values for the Apache Airflow chart
backups/
  cluster-prerequisites.md             Base cluster setup: ingress-nginx, cert-manager,
                                       ClusterIssuer, namespaces, DNS
  helm-values-all.yaml                 Computed Helm values snapshot (backup)
  airbyte-secrets.yaml                 Backed-up K8s secrets (restore before installing Airbyte)
  airbyte-auth-secrets.yaml
  airbyte-db-backup.sql                Airbyte DB dump (connections / sync history)
mongodb-gke-migration-plan.md          MongoDB Atlas → GKE migration strategy
mongodb-gke-migration-plan-detailed.md 8-phase detailed migration plan
CLAUDE.md                              Guidance for Claude Code
```

## Deploying

Bring up the base infrastructure first (ingress-nginx, cert-manager, ClusterIssuer, namespaces) — follow [`backups/cluster-prerequisites.md`](backups/cluster-prerequisites.md).

```bash
# Add Helm repos
helm repo add airbyte https://airbytehq.github.io/helm-charts
helm repo add apache-airflow https://airflow.apache.org
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update

# Deploy / upgrade Airbyte
helm upgrade --install airbyte airbyte/airbyte -n airbyte-v2 -f airbyte/values.yaml

# Deploy / upgrade Airflow
helm upgrade --install airflow apache-airflow/airflow -n airflow -f airflow/values.yaml
```

Lint values locally before applying:

```bash
helm template airbyte airbyte/airbyte -f airbyte/values.yaml > /dev/null
helm template airflow apache-airflow/airflow -f airflow/values.yaml > /dev/null
```

## Airbyte

- Community Edition, auth enabled — admin credentials and JWT signing secret come from Kubernetes secrets (`airbyte-auth-secrets`, `airbyte-secrets`; backups in `backups/`).
- Connectors run as ephemeral pods. Resource requests/limits are set per container type (main, replication, sidecar) in `airbyte/values.yaml`; connector resource defaults are disabled.
- Ingress routes `/`, `/api`, and `/connector-builder` to the appropriate backends.

## Airflow

- **Executor:** `KubernetesExecutor` — each task runs in its own pod. Redis, Celery, and Flower are intentionally disabled.
- **Metadata DB:** external Cloud SQL (PostgreSQL); the bundled PostgreSQL subchart is disabled. Connection string, Fernet key, and webserver secret key come from the `airflow-secrets` K8s secret.
- **Auth:** Google OAuth via Flask-AppBuilder, restricted to `@fitness22.com` accounts (enforced by a custom security manager in `webserverConfig`). New users register as `Viewer`. The REST API additionally accepts basic auth so service accounts can trigger DAGs programmatically.
- **DAGs:** git-sync every 60s from `git@github.com:Fitness22/f22airflow.git` (branch `main`, subpath `airflow/dags`), using the `airflow-git-ssh-secret` deploy key.
- **Logging:** remote logging to GCS (`gs://f22-airflow-logs`); local log persistence is off.
- **Pipeline secrets:** API keys (AppFigures, AppsFlyer, Slack webhook) are injected into all pods — including task pods — from the `airflow-pipeline-secrets` K8s secret.
- **Extra Python deps:** installed at runtime via `_PIP_ADDITIONAL_REQUIREMENTS` (`pymongo`, `dnspython`, `python-dotenv`).

## MongoDB migration (planned)

Migration from MongoDB Atlas M30 (~363 GB, ~$800+/month) to self-hosted MongoDB Community on GKE:

- **Target:** 3-member replica set (no arbiter) managed by the MongoDB Community Kubernetes Operator, on a dedicated `mongodb-pool` of 3× `e2-standard-4` nodes with 500 GB `pd-ssd` volumes.
- [`mongodb-gke-migration-plan.md`](mongodb-gke-migration-plan.md) — strategy: target architecture, security model, backup/restore, monitoring, 6-phase migration (discovery → build → rehearse → sync → cutover → rollback window).
- [`mongodb-gke-migration-plan-detailed.md`](mongodb-gke-migration-plan-detailed.md) — detailed week-by-week 8-phase execution plan.

## Conventions & constraints

- **Never commit secrets in plaintext.** All credentials live in Kubernetes secrets; the YAML backups in `backups/` are for cluster restore only.
- `KubernetesExecutor` is intentional for Airflow — do not re-enable Redis or Flower.
- Airbyte is Community Edition; Enterprise-gated features are unavailable.
- TLS terminates at the nginx ingress for every external service.
- Cloud SQL is Airflow's database; keep the bundled PostgreSQL subchart disabled.
