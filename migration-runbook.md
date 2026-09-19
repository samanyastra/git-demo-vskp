# Azure → AWS Migration Runbook

Expands on `migration-plan.txt`. Goal: move compute off Azure AKS to AWS Lightsail,
targeting ~500 users, spending your $500 AWS credit instead of new cash. Database
is already on AWS; Azure Blob storage stays on Azure.

## 0. Current state (confirmed from your notes)

| Item | Current | Staying / Moving |
|---|---|---|
| Compute | AKS, 2 nodes, free-tier control plane | **Moving** |
| Ingress | Self-managed nginx | **Moving** (replace with nginx on the instance, or Lightsail LB) |
| TLS | cert-manager + ClusterIssuer (Let's Encrypt) | **Moving** (replace with certbot) |
| File storage | Azure Blob | **Staying** (code depends on Azure SDK) |
| Container registry | ACR | **Decide** — see §5 |
| Database | Already AWS | **Staying**, but confirm VPC/region alignment — see §6 |

Services to migrate: `lumivis-app`, `rabbitmq-mgmt`, `samanyastra-api`, `samanyastra-ats`,
`samanyastra-auth`, `samanyastra-books`, `samanyastra-celery`, `samanyastra-mailer`,
`samanyastra-mailer-celery`, `samanyastra-public`.

## 1. Two architecture options

**Option A — Lightsail Container Service per service (managed).**
Each service gets its own Lightsail Container Service (built-in HTTPS LB + cert,
no server to patch, push-button deploys). Simplest ops, but you pay per service —
10 services × ~$7–15/mo (nano/micro tier) is roughly $70–150/mo, which burns a
$500 credit in 4–7 months. It's also awkward for RabbitMQ (needs persistent
storage + non-HTTP ports, which this product isn't built for).

**Option B — One or two Lightsail *instances*, Docker Compose (recommended).**
Plain Ubuntu VPS(es) running Docker Compose with your own nginx + certbot,
mirroring what you already run today just without the Kubernetes control plane.
A single **Large** instance (2 vCPU / 4 GB RAM / 80 GB SSD, roughly ~$20/mo at
current list pricing — verify on the Lightsail pricing page since this changes)
comfortably hosts all 10 services plus RabbitMQ for ≤500 users. Two instances
(e.g. web/API services on one, RabbitMQ + both Celery workers on the other) buys
you failure isolation for maybe ~$30–40/mo combined.

At $20–40/mo total, your $500 credit lasts 12–25 months — that's the real "$0 cost"
outcome; the credit is what makes it $0, not the architecture itself. Recommend
**Option B**, since it's closest to your existing nginx/cert-manager mental model,
is dramatically cheaper, and Celery/RabbitMQ aren't a great fit for a
container-per-service PaaS product.

Everything below assumes **Option B**.

## 2. Sizing

- Start with **one** Lightsail instance (Large: 2 vCPU/4GB) running everything
  via `docker-compose.yml`. Watch `docker stats` / CPU credits for the first
  couple weeks under real traffic.
- If RabbitMQ + Celery contend with the web services for CPU/RAM, split to a
  second instance for messaging/workers. Trivial to do later — same Compose
  file, different host, point `CELERY_BROKER_URL` at the new instance's
  private IP (see §6 on private networking).
- Keep a static IP (Lightsail "Static IP", free while attached to a running
  instance) so DNS doesn't need to change on reboot.

## 3. Networking, DNS & TLS

1. Attach a Lightsail **static IP** to the instance.
2. Open ports in the Lightsail networking tab: 80, 443, and 22 (restrict 22 to
   your IP or use Lightsail's browser SSH).
3. Install nginx + certbot on the instance directly (not in a container is
   simpler for cert renewal, but a `certbot` sidecar container works too if you
   want everything in Compose — either is fine at this scale).
4. Reverse-proxy each service by hostname/path the same way your current nginx
   ingress rules do — port these rules across; they don't need to change
   semantically, just the config format if you were using k8s Ingress objects.
5. **DNS cutover sequence** (do this in this order to avoid downtime):
   - Lower TTL on all relevant DNS records to 60–300s, at least 24h before cutover.
   - Bring up the full stack on Lightsail, smoke-test via `curl -H "Host: ..."`
     against the Lightsail IP directly (or a temp `*.sslip.io` hostname) before
     touching DNS.
   - Issue certs via certbot against the real hostnames — Let's Encrypt's HTTP-01
     challenge needs DNS already pointed at Lightsail, so certs must be issued
     *after* cutover, or use DNS-01 challenge to issue ahead of time if your DNS
     provider supports it (recommended — avoids a TLS gap during cutover).
   - Flip DNS records to the Lightsail static IP.
   - Watch error rates / logs for ~30–60 min per service.
   - Keep AKS running (scaled up) for at least 48–72h as a rollback target —
     just flip DNS back if something's wrong.
6. RabbitMQ management UI (`rabbitmq-mgmt`, port 15672) should **not** be
   internet-facing — proxy it behind nginx with basic auth, or only bind it to
   the instance's private interface and access via SSH tunnel.

## 4. Container images: ACR vs ECR

Your note says ACR is easier but you're open to switching if AWS has an
equivalent — it does (ECR). Trade-off:

- **Keep ACR**: zero migration work, Lightsail instance just needs
  `docker login <acr>.azurecr.io` and pulls over the internet. Downside: image
  pulls incur Azure egress bandwidth charges (small at this scale, but non-zero)
  and pulls are slower than an in-region AWS pull.
- **Move to ECR**: pulls become free/fast (same-cloud), and you can use
  temporary credentials via an IAM role instead of a long-lived pull secret.
  Migration is just `docker pull` from ACR → `docker tag` → `docker push` to
  ECR for each of the 10 images, then update your CI/CD push target and the
  Compose file's image references.

Recommendation: since you're moving compute to AWS anyway and CI will need to
push to *some* registry, moving to ECR is worth the one-time effort — it removes
a permanent cross-cloud dependency and a recurring (if small) egress cost.
Budget ~30–60 min per service to update the CI pipeline's push step; the actual
image copy is one script (see snippet below).

```bash
# one-off per image: copy ACR -> ECR
az acr login --name <acrname>
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
aws ecr create-repository --repository-name <service> --region <region>
docker pull <acrname>.azurecr.io/<service>:latest
docker tag <acrname>.azurecr.io/<service>:latest <account>.dkr.ecr.<region>.amazonaws.com/<service>:latest
docker push <account>.dkr.ecr.<region>.amazonaws.com/<service>:latest
```

## 5. Database & storage connectivity

- **Database (already AWS)**: confirm it's in a VPC (e.g. RDS). Lightsail
  instances live outside your default VPC by default — enable **Lightsail VPC
  peering** (one checkbox per region in the Lightsail account settings) so the
  instance can reach the DB over private IPs instead of the public endpoint.
  Also confirm the Lightsail instance's **region matches the DB's region** to
  avoid cross-region latency/egress.
- **Azure Blob storage**: no code changes needed since it's staying. The only
  new cost is Azure egress bandwidth for reads now originating from AWS instead
  of AKS in the same Azure region — likely negligible at 500 users, but worth a
  glance at your Azure bill 2–4 weeks post-migration.

## 6. Migration sequence (end to end)

1. Provision Lightsail instance(s), attach static IP, open firewall ports.
2. Enable Lightsail VPC peering; confirm DB reachability from the instance
   (`psql`/`mysql` test connection over private IP).
3. Decide ACR-vs-ECR (§5) and, if moving, copy all 10 images to ECR.
4. Write `docker-compose.yml` covering all 10 services + RabbitMQ, ported from
   your k8s manifests (env vars, volumes, resource limits, restart policies).
5. Install nginx + certbot on the instance; port ingress rules.
6. Bring the stack up on Lightsail; smoke-test each service by IP/temp hostname.
7. Issue TLS certs (prefer DNS-01 so this can happen before cutover — see §3).
8. Lower DNS TTLs (24h lead time).
9. Cut DNS over to the Lightsail static IP, service by service if you want to
   de-risk further (e.g. `samanyastra-public` first since it's lowest-stakes,
   `samanyastra-auth`/`samanyastra-api` last).
10. Monitor for 48–72h with AKS still running as rollback.
11. Decommission AKS nodes (delete the node pool / cluster) once stable —
    this is what actually stops the Azure compute bill.

## 7. Cost estimate

| Item | Est. monthly | Notes |
|---|---|---|
| Lightsail instance (Large) | ~$20 | verify current price before committing |
| Second instance (optional, RabbitMQ/Celery) | ~$10–20 | only if splitting |
| ECR storage | ~$1–2 | 10 small images |
| Data transfer | ~$0–5 | Lightsail includes a transfer allowance |
| **Total** | **~$20–45/mo** | vs. your $500 credit → **12–25 months runway** |

This is an estimate against list pricing that may have moved since my last
update — check the current Lightsail bundle prices before you commit, and
re-check after AWS's periodic price changes.

## 8. Open questions before you start

- Split across two instances now, or start with one and split only if it's
  actually needed under load?
- ACR→ECR now, or defer and accept the small cross-cloud egress cost?
- Is the AWS database in a VPC, and do you already know its region (needs to
  match wherever the Lightsail instance goes)?
- Do you want per-service DNS cutover (safer, slower) or a single cutover for
  all 10 at once (faster, riskier)?

---
Once you've decided on the open questions above, I can help write the actual
`docker-compose.yml`, nginx config, and the ACR→ECR image-copy script.
