# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

`f22-data-infra` is the Fitness22 data infrastructure repository — Helm values, Kubernetes manifests, and migration docs for the GKE-based data platform. There is no application code, no build system, and no test runner.

## Cluster & Deployment Commands

```bash
# Add required Helm repos
helm repo add airbyte https://airbytehq.github.io/helm-charts
helm repo add apache-airflow https://airflow.apache.org
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update

# Deploy / upgrade Airbyte (namespace: airbyte-v2)
helm upgrade --install airbyte airbyte/airbyte -n airbyte-v2 -f airbyte/values.yaml

# Deploy / upgrade Airflow (namespace: airflow)
helm upgrade --install airflow apache-airflow/airflow -n airflow -f airflow/values.yaml

# Lint Helm values locally before applying
helm template airbyte airbyte/airbyte -f airbyte/values.yaml > /dev/null
helm template airflow apache-airflow/airflow -f airflow/values.yaml > /dev/null
```

## Architecture Overview

### GKE Cluster: `f22-data-gke-prod`
Regional, private cluster. All services use `ingress-nginx` (v4.15.1) with TLS from `cert-manager` (v1.20.0) via Let's Encrypt (`letsencrypt-prod` ClusterIssuer). Secrets are stored as Kubernetes secrets; never committed in plaintext.

### Airbyte (`airbyte/values.yaml`)
- Version 2.0.19 Community Edition
- URL: `https://airbyte.fitness22services.com`
- Auth: JWT + admin credentials in a Kubernetes secret
- Connectors run as ephemeral pods; resource limits are defined per container type (main, replication, sidecar)

### Airflow (`airflow/values.yaml`)
- Version 2.9.3, executor: `KubernetesExecutor` (no Celery/Redis/Flower)
- URL: `https://airflow.fitness22services.com`
- Auth: Google OAuth, restricted to `@fitness22.com` accounts
- DAGs: git-sync from Bitbucket (`clearskyapps/f22airflow.git`, `main` branch)
- Remote logging: MinIO S3 bucket (`s3://airflow-logs`)
- External DB: Cloud SQL (PostgreSQL) — bundled PostgreSQL is disabled

### MongoDB (planned migration)
- Current: MongoDB Atlas M30 (~363 GB, ~$800+/month)
- Target: self-hosted MongoDB Community on GKE via the MongoDB Community Kubernetes Operator
  - `mongodb-pool`: 3× `e2-standard-4` nodes, each with a 500 GB `pd-ssd` PV
  - Topology: 3-member replica set, no arbiter
- See `mongodb-gke-migration-plan.md` (strategy) and `mongodb-gke-migration-plan-detailed.md` (8-phase plan) for the full migration approach

### Base infrastructure prerequisites
`backups/cluster-prerequisites.md` documents the ingress-nginx, cert-manager, and storage setup required before deploying any service.

## Key Constraints

- `KubernetesExecutor` is intentional for Airflow — do not re-enable Redis or Flower.
- Airbyte uses Community Edition; features gated behind Enterprise are unavailable.
- All external-facing services terminate TLS at the nginx ingress — services themselves run HTTP internally.
- Cloud SQL is the external database for Airflow; the bundled PostgreSQL subchart is explicitly disabled.
