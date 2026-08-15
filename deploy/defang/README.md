# Deploy Buzz to your own cloud with Defang

This bundle deploys a Buzz relay to **your** cloud account (AWS, GCP, Azure, or
DigitalOcean) using [Defang](https://defang.io) — a Docker-Compose-compatible
"bring your own cloud" deployment. It's the managed-services counterpart to the
single-node [`deploy/compose`](../compose/) bundle.

## Why a separate bundle

The VPS bundle runs Postgres, Redis, and MinIO as containers with persistent
volumes. Defang runs each service as a managed cloud primitive instead, which
means **no persistent volumes are required** — Buzz's durable git state is
object-store-backed, so the relay keeps no local state.

| Dependency | VPS (`deploy/compose`) | Defang (`deploy/defang`) |
|---|---|---|
| Database | `postgres` container + volume | managed Postgres (`x-defang-postgres`, e.g. RDS) |
| Cache/pubsub | `redis` container + volume | managed Redis (`x-defang-redis`, e.g. ElastiCache) |
| Object storage | `minio` + volume | your cloud's S3-compatible store (external S3) |
| TLS | Caddy | Defang load balancer + auto-cert |

## Prerequisites

- The [Defang CLI](https://docs.defang.io/docs/getting-started/installation) and
  cloud credentials for your provider (e.g. `AWS_ACCESS_KEY_ID` /
  `AWS_SECRET_ACCESS_KEY` for AWS).
- An S3-compatible bucket in your account for media + git objects, and
  credentials scoped to it. (Defang doesn't yet provision object storage, so
  create the bucket out-of-band.)

## Configure

Generate stable secrets and set every value with `defang config set`:

```bash
cd deploy/defang

# stable secrets (generate once, keep safe)
defang config set BUZZ_RELAY_PRIVATE_KEY      # 64-char hex
defang config set BUZZ_GIT_HOOK_HMAC_SECRET   # 32+ random chars
defang config set POSTGRES_PASSWORD           # also used by managed Postgres
defang config set DATABASE_URL                # postgres://buzz:<PASSWORD>@postgres:5432/buzz?sslmode=require

# owner identity (64-char hex Nostr pubkey)
defang config set RELAY_OWNER_PUBKEY

# object storage (AWS example)
defang config set BUZZ_S3_ENDPOINT            # https://s3.<region>.amazonaws.com
defang config set BUZZ_S3_REGION              # e.g. us-east-1
defang config set BUZZ_S3_BUCKET
defang config set BUZZ_S3_ACCESS_KEY
defang config set BUZZ_S3_SECRET_KEY

# public URL (set to your custom domain, or fill in after the first deploy with
# the Defang-assigned *.defang.app URL, then redeploy — do this before members join)
defang config set RELAY_URL                   # wss://<host>
defang config set BUZZ_DOMAIN                 # <host>
defang config set BUZZ_MEDIA_BASE_URL         # https://<host>/media
defang config set BUZZ_MEDIA_SERVER_DOMAIN    # <host>
defang config set BUZZ_CORS_ORIGINS           # https://<host>
```

## Deploy

```bash
defang compose up
```

Defang provisions the managed database, cache, load balancer, and TLS cert, then
starts the relay. The first deploy prints the public URL. If you're using the
auto-assigned `*.defang.app` domain, set the `RELAY_URL`/`BUZZ_*` URL values to
it and redeploy once — before adding members, since the relay URL selects the
community.

## Notes

- The managed database/cache are reached over their compose **service names**
  (`postgres`, `redis`); Defang wires those to the managed endpoints.
- Managed Redis is reached in-VPC without a password (`redis://redis:6379`).
- The load-balancer health check targets `/_liveness` on port 3000 (served by the
  relay, no auth). The Buzz image ships no `curl`, so the healthcheck falls back
  to a bash `/dev/tcp` probe for the in-container check.
- Back up `BUZZ_RELAY_PRIVATE_KEY`, your owner key, the database, and the object
  store; the relay identity key is your community's identity.
