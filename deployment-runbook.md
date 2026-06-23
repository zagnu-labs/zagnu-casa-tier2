# Deployment and Recovery Runbook

How the application, its configuration, and dependencies are deployed in an
automated, repeatable way, and how to recover (rollback or restore from backup)
in a timely fashion. Covers `zmail-services` (the Gmail-integration service);
other backend services follow the same pattern.

Last reviewed: 2026-06.

## 1. Everything is defined as code

| Layer | Source of truth |
|-------|-----------------|
| Application | Go source, built into a container image (Dockerfile) |
| Service config & runtime | Helm chart in the service repo (`.helm/`) |
| Cloud infrastructure | Terraform (AWS: EKS, Aurora, ElastiCache, MSK, ALB) |
| Gateway routing | Helm values in the `infra` repo (`envs/<env>/helm/api-gateway`) |
| Secrets | CI/CD encrypted secrets, injected as Kubernetes Secrets at deploy time |
| Frontend | Vercel project, deployed from Git |

No production change is applied by hand; every component is reproducible from
version control.

## 2. Automated deployment (normal path)

Backend (`zmail-services`) — GitHub Actions (`.github/workflows/deploy.yml`):

1. Trigger: push to `main`.
2. Authenticate to AWS, build the container image, push to **AWS ECR**.
3. `helm upgrade --install` against the **EKS** cluster, injecting secrets via
   `--set-string secrets.*` from the encrypted CI secret store.
4. Kubernetes performs a rolling update of the deployment.

Frontend: Vercel deploys automatically on push to `main`.

The workflow runs on every merge, so the deployment path is continuously
exercised ("tested"); the Actions run history is the evidence of repeatability.

**Typical time to deploy: a few minutes** (build + push + rolling update).

## 3. Rebuild / redeploy from scratch

1. Provision infrastructure with Terraform (`terraform apply`) — recreates EKS,
   Aurora, ElastiCache, MSK, ALB.
2. Re-run the service deploy workflow (or `helm upgrade --install`) to deploy the
   application and its config onto the cluster.
3. Apply the `infra` gateway Helm values to restore routing.
4. Restore data from backups (section 5).

## 4. Rollback (bad release)

- **Application:** `helm rollback zmail-services <previous-revision>` (or re-run
  the pipeline pinned to the previous image tag). Rolling update reverts in
  minutes.
- **Frontend:** promote the previous Vercel deployment.

## 5. Restore from backups

- **PostgreSQL (Aurora):** automated continuous backups with
  point-in-time-recovery (PITR). Restore via the AWS console / CLI
  (`aws rds restore-db-cluster-to-point-in-time`) to a chosen timestamp, then
  repoint the service. Manual snapshots can also be taken before risky changes.
- **Redis (ElastiCache):** holds only ephemeral short-link codes (30-min TTL);
  no restore needed — data is non-durable by design.
- **Kafka (MSK):** event bus; consumers are idempotent and reprocess from
  offsets as needed.
- **Secrets:** held in the CI/CD secret store (and an offline secure backup of
  the token-encryption key); re-injected on the next deploy.

**Time to restore (RTO): Aurora PITR typically completes within the low tens of
minutes**, after which a redeploy (minutes) brings the service back.

## 6. Critical dependency: token-encryption key

The OAuth token-encryption key (`TOKEN_ENCRYPTION_KEY`) is required to read
existing encrypted tokens. It is stored in the CI/CD secret store and backed up
in a secure vault. A database restore is only useful together with the matching
key, so the key is treated as part of the recovery plan.

## 7. Verification after deploy/restore

- `GET /healthz` returns 200.
- A read against a connected inbox succeeds (tokens decrypt correctly → key is correct).
- Security headers / TLS verified at the gateway (`curl -I https://api.zagnu.com/...`).
