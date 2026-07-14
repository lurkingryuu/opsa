# OPSA Deployment Rehaul: docker-compose/SSH to Talos + Flux GitOps

Status: **implemented locally, cutover pending**

This document is the complete design for moving OPSA off the old SSH + docker-compose
homelab deployment and onto the new homelab: a single-node Talos Linux cluster
(`hermes`, i5-7200U / 12GB) managed by Flux GitOps in the
[`homenet`](https://github.com/lurkingryuu/homenet) repo, with Cilium Gateway API,
CloudNativePG, SOPS-encrypted secrets, and in-cluster MinIO.

---

## 1. Background

### 1.1 What OPSA is

| Component | Tech | Role |
|-----------|------|------|
| `tummy` | PostgreSQL | The database. Schema: `users`, `channels`, `messages` |
| `digester` | Go | One-shot, idempotent ingestion job: reads a Slack export `.zip` (`ZIPFILE_PATH`) and inserts users/channels/messages |
| `excretor` | Rust (Axum + SQLx) | Backend API on `EXCRETOR_PORT`; runs `sqlx::migrate!("../migrations")` on startup; optional Slack OAuth auth |
| `garnisher` | React + Vite on nginx | The SPA frontend; nginx proxies `/api/` to excretor (compose-era wiring) |

### 1.2 How deployment works today (and why it must go)

`.github/workflows/deploy-homelab.yaml` (push to `rewrite`):

1. A self-hosted runner builds the three images locally (`:latest`, no registry).
2. `docker save | gzip` each image, `scp` the tarballs plus `docker-compose.yml`,
   a dotenvx-decrypted `.env`, `migrations/`, and `tummy/` to the old server.
3. A long SSH script `docker load`s images, runs compose, manually checks/loads
   `tummy/init.sql` with ad-hoc `psql` containers, `DROP TABLE _sqlx_migrations`
   as a checksum-mismatch workaround, runs the digester if the DB is empty, and
   greps `docker compose ps` for health.

None of this can work against the new homelab: Talos has no SSH shell or Docker
daemon, and the cluster is pull-based GitOps (a git push to `homenet` *is* the
deploy). The rehaul therefore has two halves:

- **Part A (this repo)**: make OPSA cluster-deployable - registry-hosted versioned
  images, self-contained DB bootstrap, a health endpoint, and removal of the old
  deploy machinery and production dotenvx key material.
- **Part B (homenet repo)**: a new Flux layer that runs OPSA following the
  cluster's existing conventions.

### 1.3 Decisions already made

| Question | Decision |
|----------|----------|
| Exposure | Public at `opsa.cloud.karthikeyay.com` via the cluster's `external` Gateway |
| Auth | excretor's built-in Slack OAuth (`SLACK_AUTH_ENABLE=true`), not Authentik |
| Images | GitHub Actions builds and pushes `ghcr.io/lurkingryuu/opsa-*`; semver image tags cut from `v*` git tags; homenet pins tags with `# Renovate-managed` comments and Renovate PRs bump them |
| Archive ingestion | One-shot Kubernetes Job pulls the export zip from the in-cluster MinIO (`minio.storage.svc.cluster.local:9000`) |

### 1.4 Verified facts the design rests on

- excretor's Dockerfile needs the **repo root as build context** because it copies
  the sibling `migrations/` dir (compile-time `sqlx::migrate!("../migrations")`
  path parity, `excretor/src/db/tummy.rs:51`).
- In `excretor/src/api/routes.rs`, routes registered **before** `route_layer(...)`
  (`/api/*` and `/`) are guarded by `verify_token_middleware`; routes registered
  after (`/auth`, `/auth/callback`, `/assets/*file`) are not. A new `/health`
  route must be registered after the layer.
- excretor's `/` handler is a placeholder (`"Hello app!"`); the real SPA,
  including its logged-out view and Vite `/assets` bundle, is served by garnisher.
  Therefore `/assets` must **never** be routed to excretor in-cluster (its
  `/assets` route serves a different, legacy static dir).
- `Tummy`'s connection pool (`tummy_conn_pool`) is private; a DB-backed health
  check needs a small `ping()` method on `Tummy`.
- digester reads exactly `TUMMY_HOST`, `TUMMY_PORT`, `TUMMY_USERNAME`,
  `TUMMY_PASSWORD`, `TUMMY_DB`, `ZIPFILE_PATH` (`digester/main.go`). It queries
  `messages` immediately and dies if tables are missing; it is idempotent over
  existing rows.
- excretor calls `https://slack.com/api/*` via reqwest, so the runtime image
  needs `ca-certificates`.
- `STATIC_ASSETS_DIR` is canonicalized at excretor boot and the process panics
  if the directory does not exist - the image must create it.
- MinIO's in-cluster app-template service is `minio` in namespace `storage`,
  S3 API on port 9000.
- CNPG pattern in homenet (`kubernetes/apps/auth/authentik/database.yaml`): a
  `Cluster` auto-generates secret `<name>-app` with keys
  `username/password/dbname/host/port/uri` and a read-write service `<name>-rw`.
- The dotenvx-encrypted `.env.prod` is safe to track, but it is no longer needed
  after production secret management moves to SOPS in homenet. The matching
  `.env.keys` file is private key material and must remain untracked.

---

## 2. Part A - opsa repo changes

Branch off `main` and open the PR against `main`.

### A1. Fold `tummy/init.sql` into sqlx migrations

Today the schema is split: base tables come from `tummy/init.sql` (only applied by
the postgres image's initdb on a fresh volume) and search indexes come from the
sqlx migration `migrations/20240603132234_index.sql` (applied by excretor at
startup). This split is exactly why the old deploy script needed manual `psql`
bootstrapping and the `DROP TABLE _sqlx_migrations` hack. CNPG will not run
`init.sql`, so excretor's migrations must fully bootstrap a fresh database alone.

- Create `migrations/20240603000000_init_tables.sql` with the exact contents of
  `tummy/init.sql` (tables `users`, `channels`, `messages`; all statements are
  already `CREATE TABLE IF NOT EXISTS`, so it is a no-op over databases that
  were initialized the old way).
- The timestamp `20240603000000` sorts **before** the existing
  `20240603132234_index.sql`, so ordering is tables first, indexes second.
- Delete the `tummy/` directory and remove both `./tummy/init.sql` mounts from
  `docker-compose.yml` (services `tummy` and `tummy-dev`). Local dev keeps
  working: `make dev` already runs `cargo sqlx migrate run` which now creates
  the tables too.

Migration file content (verbatim from `tummy/init.sql`):

```sql
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    real_name TEXT NOT NULL,
    display_name TEXT NOT NULL,
    email TEXT NOT NULL,
    deleted BOOLEAN NOT NULL DEFAULT FALSE,
    is_bot BOOLEAN NOT NULL DEFAULT FALSE,
    image_url TEXT
);

CREATE TABLE IF NOT EXISTS channels (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    topic TEXT,
    purpose TEXT
);

CREATE TABLE IF NOT EXISTS messages (
    channel_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    ts TIMESTAMP(6) NOT NULL,
    msg_text TEXT NOT NULL,
    thread_ts TIMESTAMP(6),
    parent_user_id TEXT,
    PRIMARY KEY (channel_id, user_id, ts),
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (channel_id) REFERENCES channels(id)
);
```

SQLx 0.7 applies any pending migration in sorted order even when a newer version
is already recorded, so existing dev databases do not need a volume reset. The
`IF NOT EXISTS` statements make the new base migration a no-op over their tables.
Never renumber or edit an applied migration after the first production deploy.

### A2. Add a `/health` endpoint to excretor

Kubernetes probes and Gatus need an unauthenticated health URL; none exists today
(excretor's `/` is auth-guarded when Slack auth is on).

`excretor/src/db/tummy.rs` - add a ping method:

```rust
pub async fn ping(&self) -> Result<(), sqlx::Error> {
    sqlx::query("SELECT 1")
        .execute(&self.tummy_conn_pool)
        .await
        .map(|_| ())
}
```

`excretor/src/api/handlers/misc.rs` - add the handler (DB-checked, so Gatus and
readiness catch a dead DB, not just a live process):

```rust
pub async fn health(State(state): State<RouterState>) -> StatusCode {
    match state.tummy.ping().await {
        Ok(()) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

Re-export via `excretor/src/api/handlers/mod.rs` alongside the other misc
handlers.

`excretor/src/api/routes.rs` - register **after** `route_layer(...)` so it
bypasses `verify_token_middleware`:

```rust
        .route("/auth", get(handlers::auth))
        .route("/auth/callback", get(handlers::auth_callback))
        .route("/assets/*file", get(handlers::assets))
        .route("/health", get(handlers::health))
```

Verify with `cargo check` (with `SQLX_OFFLINE=true`).

### A2.1 Repair the SPA authentication handoff

The SPA previously treated its login button as a local state toggle and never
navigated to excretor's `/auth` endpoint. It also tried to parse the middleware's
redirected login page as JSON. Make authentication state derive from API results:

- unauthenticated `/api/*` requests return `401 Unauthorized`;
- the SPA shows a loading state until the initial API request resolves;
- API success marks the session authenticated and `401` returns it to login;
- the login button navigates to `/auth`, which starts Slack OAuth.

### A3. Multi-stage excretor Dockerfile

The current `excretor/Dockerfile` is single-stage `rustlang/rust:nightly-slim`
and ships the entire Rust toolchain plus source. Replace it (build context stays
the **repo root**):

```dockerfile
FROM rustlang/rust:nightly-slim AS builder

WORKDIR /usr/src/opsa
COPY excretor ./excretor
COPY migrations ./migrations

WORKDIR /usr/src/opsa/excretor
ENV SQLX_OFFLINE=true
RUN cargo build --release

FROM debian:bookworm-slim

# ca-certificates: excretor calls https://slack.com/api/* (OAuth + auth.test)
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /usr/src/opsa/excretor/target/release/excretor /app/excretor

# STATIC_ASSETS_DIR is canonicalized at boot and must exist
RUN mkdir -p /app/assets
ENV STATIC_ASSETS_DIR=/app/assets

EXPOSE 3000
CMD ["/app/excretor"]
```

On the first build, check shared-library cleanliness of the binary in the slim
runtime (`ldd /app/excretor`); if reqwest was built with native-tls, add
`libssl3` to the runtime apt install (rustls needs nothing).

The digester Dockerfile remains multi-stage. Garnisher now builds with Bun from
its locked dependency graph, bundles Tailwind through Vite instead of loading the
development CDN in production, and copies only the generated SPA into nginx.
This keeps the runtime image static while making frontend builds reproducible
and dependency-auditable.

### A4. New CI workflow `.github/workflows/build-images.yaml`

- **Registry/owner**: `ghcr.io/lurkingryuu/opsa-{excretor,garnisher,digester}`.
  The deploy target is the personal homelab consuming from the fork, and the
  fork's `GITHUB_TOKEN` can push to `lurkingryuu/*` GHCR packages with zero PAT
  setup. If upstream (`kossiitkgp`) later adopts this, only the repository
  string changes in homenet (one Renovate-visible line per image).
- **Tagging**: semver from `v*` git tags (`type=semver,pattern={{version}}` so
  `v1.1.0` publishes `1.1.0`) plus `type=sha` for traceability. Renovate orders
  semver docker tags natively, which is what drives deploys in homenet.
- **Triggers**: tag push `v*` builds and pushes; `pull_request` builds only
  (CI validation of all three Dockerfiles).
- **Single arch** `linux/amd64` (the node is amd64; skipping QEMU keeps the
  Rust build fast).
- Coexists with `release.yaml`: both fire on `v*` tags independently
  (release.yaml cuts the GitHub release/changelog; this workflow pushes images).

```yaml
name: Build & Push Images

on:
    push:
        tags: ["v*"]
    pull_request:

permissions:
    contents: read
    packages: write

jobs:
    build:
        runs-on: ubuntu-latest
        strategy:
            matrix:
                include:
                    - name: excretor
                      context: .
                      dockerfile: excretor/Dockerfile
                    - name: garnisher
                      context: ./garnisher
                      dockerfile: garnisher/Dockerfile
                    - name: digester
                      context: ./digester
                      dockerfile: digester/Dockerfile
        steps:
            - name: Checkout repository
              uses: actions/checkout@v4

            - name: Set up Docker Buildx
              uses: docker/setup-buildx-action@v3

            - name: Log in to GHCR
              if: github.event_name != 'pull_request'
              uses: docker/login-action@v3
              with:
                  registry: ghcr.io
                  username: ${{ github.actor }}
                  password: ${{ secrets.GITHUB_TOKEN }}

            - name: Extract image metadata
              id: meta
              uses: docker/metadata-action@v5
              with:
                  images: ghcr.io/${{ github.repository_owner }}/opsa-${{ matrix.name }}
                  tags: |
                      type=semver,pattern={{version}}
                      type=sha

            - name: Build and push
              uses: docker/build-push-action@v6
              with:
                  context: ${{ matrix.context }}
                  file: ${{ matrix.dockerfile }}
                  push: ${{ github.event_name != 'pull_request' }}
                  tags: ${{ steps.meta.outputs.tags }}
                  labels: ${{ steps.meta.outputs.labels }}
                  cache-from: type=gha,scope=${{ matrix.name }}
                  cache-to: type=gha,mode=max,scope=${{ matrix.name }}
```

After the first tagged push, set all three GHCR packages to **public**
(GitHub package settings) so the cluster needs no imagePullSecret.

### A5. Cleanup

- Delete `.github/workflows/deploy-homelab.yaml` (the SSH/tarball deploy).
- `git rm --cached .env.prod` and add it to `.gitignore`. The encrypted file is
  not itself a leak; it is removed because SOPS in homenet now owns production
  secrets and the deployment no longer consumes dotenvx configuration.
- Delete `.env.keys` from the worktree. It is the private dotenvx decryption key
  and is no longer needed by the production deployment path.
- Delete `test-deployment.sh` and `TEST_DEPLOYMENT.md`: they simulate the
  removed SSH deploy, reference a migration image that no longer exists, and
  contain the `DROP TABLE _sqlx_migrations` hack (grep confirms the hack lives
  nowhere else).
- `garnisher/nginx.conf` and its envsubst entrypoint stay unchanged - the
  `/api/` proxy is still used by local docker-compose; in-cluster it simply
  never receives traffic (the Gateway splits `/api` first).

---

## 3. Routing design

**Path-split at the Gateway** rather than proxying through garnisher's nginx:

```
https://opsa.cloud.karthikeyay.com
    /api/*     -> opsa-excretor:3000   (auth-guarded JSON API)
    /auth/*    -> opsa-excretor:3000   (Slack OAuth start + callback)
    /health    -> opsa-excretor:3000   (probes + Gatus)
    everything else -> opsa-garnisher:80   (SPA shell, /login, /assets bundle)
```

Rationale:

- One less hop for API traffic; no nginx template/envsubst coupling in-cluster.
- nginx.conf stays untouched for local compose dev.
- `/assets` must stay on garnisher (Vite bundle); excretor's `/assets` is a
  different, legacy static dir. This is encoded as a comment in the HTTPRoute.
- Same origin for SPA and API, so the `Secure; HttpOnly` JWT cookie set at
  `/auth/callback` flows to `/api` calls. TLS terminates at the Gateway
  (wildcard `*.cloud.karthikeyay.com` cert), which satisfies the Secure flag.
- `SLACK_REDIRECT_URI` becomes `https://opsa.cloud.karthikeyay.com/auth/callback`
  and the same URL must be added to the Slack app's OAuth redirect URLs
  (manual step, see runbook).

---

## 4. Part B - homenet repo changes

A new top-level Flux layer `kubernetes/apps/opsa/` with root Kustomization
`cluster-opsa`, rather than folding into an existing layer. OPSA has its own
database, Job, namespace, and public hostname; its only dependency is
`cluster-storage` (CNPG operator + MinIO). No Authentik dependency since auth is
Slack OAuth. This mirrors the `cluster-headscale` precedent for a
special-dependency app.

### B1. `kubernetes/flux/cluster/ks.yaml` - append

```yaml
---
# OPSA (KOSS Slack archive) - needs CNPG operator + MinIO (archive zip), both in
# the storage layer. Public at opsa.cloud.karthikeyay.com; auth is Slack OAuth
# inside excretor, so no dependency on cluster-auth.
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cluster-opsa
  namespace: flux-system
spec:
  interval: 1h
  retryInterval: 2m
  timeout: 10m
  path: ./kubernetes/apps/opsa
  prune: true
  wait: false
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: cluster-storage
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### B2. New files under `kubernetes/apps/opsa/`

```
kubernetes/apps/opsa/
├── kustomization.yaml
├── namespace.yaml
├── database.yaml
├── excretor.yaml
├── garnisher.yaml
├── digester-job.yaml
├── httproute.yaml
├── secret.sops.yaml            (created at cutover, committed encrypted)
└── secret.sops.yaml.example
```

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./namespace.yaml
  - ./database.yaml
  # Cutover gate: enable only after creating the encrypted production secret.
  # - ./secret.sops.yaml
  - ./excretor.yaml
  - ./garnisher.yaml
  - ./digester-job.yaml
  - ./httproute.yaml
```

`namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: opsa
  labels:
    kustomize.toolkit.fluxcd.io/prune: disabled
```

`database.yaml` - CNPG Cluster. Database name stays `tummy` (matches the app's
vocabulary; the generated secret's `dbname` key wires it regardless). Schema
comes entirely from excretor's sqlx migrations at boot (A1), so no init SQL:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: opsa-postgres
  namespace: opsa
spec:
  instances: 1
  storage:
    size: 5Gi
    storageClass: local-path
  bootstrap:
    initdb:
      database: tummy
      owner: tummy
```

CNPG auto-generates secret `opsa-postgres-app` (keys
`username/password/dbname/host/port/uri`) and service `opsa-postgres-rw`.

`excretor.yaml` - bjw-s app-template HelmRelease via `chartRef` (the cluster's
standard for first-party apps, sonarr pattern):

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: opsa-excretor
  namespace: opsa
spec:
  interval: 1h
  chartRef:
    kind: OCIRepository
    name: app-template
    namespace: flux-system
  values:
    controllers:
      opsa-excretor:
        containers:
          app:
            image:
              repository: ghcr.io/lurkingryuu/opsa-excretor
              tag: "1.1.0" # Renovate-managed
            env:
              EXCRETOR_PORT: "3000"
              TUMMY_USERNAME:
                valueFrom:
                  secretKeyRef:
                    name: opsa-postgres-app
                    key: username
              TUMMY_PASSWORD:
                valueFrom:
                  secretKeyRef:
                    name: opsa-postgres-app
                    key: password
              TUMMY_HOST:
                valueFrom:
                  secretKeyRef:
                    name: opsa-postgres-app
                    key: host
              TUMMY_PORT:
                valueFrom:
                  secretKeyRef:
                    name: opsa-postgres-app
                    key: port
              TUMMY_DB:
                valueFrom:
                  secretKeyRef:
                    name: opsa-postgres-app
                    key: dbname
              SLACK_AUTH_ENABLE: "true"
              SLACK_REDIRECT_URI: "https://opsa.cloud.karthikeyay.com/auth/callback"
              KEEP_LOGGED_IN_FOR_DAYS: "30"
              TITLE: "OPSA"
              DESCRIPTION: "Our Precious Slack Archive"
            envFrom:
              - secretRef:
                  name: opsa-secrets # SLACK_CLIENT_ID / SECRET / SIGNING_SECRET
            probes:
              liveness:
                enabled: true
                custom: true
                spec:
                  tcpSocket:
                    port: 3000
                  initialDelaySeconds: 10
                  periodSeconds: 15
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /health
                    port: 3000
                  initialDelaySeconds: 5
                  periodSeconds: 10
            resources:
              requests:
                cpu: 25m
                memory: 64Mi
              limits:
                memory: 256Mi
    service:
      app:
        controller: opsa-excretor
        ports:
          http:
            port: 3000
```

(`STATIC_ASSETS_DIR` is baked into the image as `/app/assets`.)

`garnisher.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: opsa-garnisher
  namespace: opsa
spec:
  interval: 1h
  chartRef:
    kind: OCIRepository
    name: app-template
    namespace: flux-system
  values:
    controllers:
      opsa-garnisher:
        containers:
          app:
            image:
              repository: ghcr.io/lurkingryuu/opsa-garnisher
              tag: "1.1.0" # Renovate-managed
            env:
              # nginx's /api proxy is unused in-cluster (the Gateway splits /api
              # first), but envsubst needs a resolvable upstream to boot.
              EXCRETOR_HOST: opsa-excretor
              EXCRETOR_PORT: "3000"
            probes:
              liveness: &probe
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /
                    port: 80
              readiness: *probe
            resources:
              requests:
                cpu: 10m
                memory: 16Mi
              limits:
                memory: 64Mi
    service:
      app:
        controller: opsa-garnisher
        ports:
          http:
            port: 80
```

`httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: opsa
  namespace: opsa
spec:
  parentRefs:
    - name: external
      namespace: network
      sectionName: https
  hostnames: [opsa.cloud.karthikeyay.com]
  rules:
    # NOTE: /assets is the Vite SPA bundle served by garnisher - do NOT send it
    # to excretor (its /assets route serves a different, legacy static dir).
    - matches:
        - path: { type: PathPrefix, value: /api }
        - path: { type: PathPrefix, value: /auth }
        - path: { type: PathPrefix, value: /health }
      backendRefs:
        - name: opsa-excretor
          port: 3000
    - backendRefs:
        - name: opsa-garnisher
          port: 80
```

The `external` Gateway carries the `external-dns: public` label, so the
Cloudflare DNS record for `opsa.cloud.karthikeyay.com` is published
automatically; the wildcard TLS cert already covers it.

`digester-job.yaml` - the one-shot ingestion Job. Design choices:

- **Schema wait**: init container 1 polls with `psql` until
  `to_regclass('public.messages')` is non-null, i.e. until excretor has booted
  and run its migrations. This encodes the real dependency (tables exist)
  rather than a proxy for it.
- **Fetch tooling**: `minio/mc` init container talking straight to the
  in-cluster S3 service - no presigned-URL choreography, credentials from the
  SOPS secret.
- **Re-run pattern**: `kustomize.toolkit.fluxcd.io/force: Enabled` annotation.
  Jobs are immutable; with this annotation Flux delete-and-recreates the Job
  whenever its spec changes, so bumping `ARCHIVE_VERSION` (or the image tag)
  re-runs ingestion. Deliberately **no** `ttlSecondsAfterFinished`: TTL-deleting
  a Flux-managed Job would make Flux recreate it every reconcile, forever.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: opsa-digester
  namespace: opsa
  annotations:
    # Jobs are immutable - this lets Flux replace the Job when the spec changes.
    # To re-ingest: upload a new zip to the MinIO "opsa" bucket and bump
    # ARCHIVE_VERSION below. The digester is idempotent over existing rows.
    kustomize.toolkit.fluxcd.io/force: Enabled
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      initContainers:
        - name: wait-for-schema
          image: ghcr.io/cloudnative-pg/postgresql:17.5 # Renovate-managed
          env:
            - name: DB_URI
              valueFrom:
                secretKeyRef:
                  name: opsa-postgres-app
                  key: uri
          command: ["/bin/sh", "-c"]
          args:
            - |
              until [ "$(psql "$DB_URI" -tAc "SELECT to_regclass('public.messages') IS NOT NULL")" = "t" ]; do
                echo "waiting for excretor to run migrations..."
                sleep 5
              done
        - name: fetch-archive
          image: quay.io/minio/mc:RELEASE.2025-05-21T01-59-54Z # Renovate-managed
          envFrom:
            - secretRef:
                name: opsa-secrets # MINIO_ACCESS_KEY / MINIO_SECRET_KEY
          env:
            - name: ARCHIVE_VERSION
              value: "2026-07" # bump to re-ingest a new export
          command: ["/bin/sh", "-c"]
          args:
            - |
              mc alias set homelab http://minio.storage.svc.cluster.local:9000 "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
              mc cp "homelab/opsa/slack-export-${ARCHIVE_VERSION}.zip" /archive/export.zip
          volumeMounts:
            - name: archive
              mountPath: /archive
      containers:
        - name: digester
          image: ghcr.io/lurkingryuu/opsa-digester:1.1.0 # Renovate-managed
          env:
            - name: ZIPFILE_PATH
              value: /archive/export.zip
            - name: TUMMY_USERNAME
              valueFrom:
                secretKeyRef:
                  name: opsa-postgres-app
                  key: username
            - name: TUMMY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: opsa-postgres-app
                  key: password
            - name: TUMMY_HOST
              valueFrom:
                secretKeyRef:
                  name: opsa-postgres-app
                  key: host
            - name: TUMMY_PORT
              valueFrom:
                secretKeyRef:
                  name: opsa-postgres-app
                  key: port
            - name: TUMMY_DB
              valueFrom:
                secretKeyRef:
                  name: opsa-postgres-app
                  key: dbname
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 512Mi
          volumeMounts:
            - name: archive
              mountPath: /archive
      volumes:
        - name: archive
          emptyDir: {}
```

`secret.sops.yaml.example` (copy to `secret.sops.yaml`, fill in, then
`sops -e -i secret.sops.yaml`; the repo's `.sops.yaml` rule for `\.sops\.yaml$`
already covers it and Flux decrypts in-cluster via `sops-age`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: opsa-secrets
  namespace: opsa
type: Opaque
stringData:
  SLACK_CLIENT_ID: REPLACE_SLACK_CLIENT_ID
  SLACK_CLIENT_SECRET: REPLACE_SLACK_CLIENT_SECRET
  # Also the HMAC key for the JWT session cookie
  SLACK_SIGNING_SECRET: REPLACE_SLACK_SIGNING_SECRET
  # Dedicated read-only MinIO key scoped to the "opsa" bucket
  MINIO_ACCESS_KEY: REPLACE_MINIO_ACCESS_KEY
  MINIO_SECRET_KEY: REPLACE_MINIO_SECRET_KEY
```

### B3. Gatus

Add to the endpoint list in
`kubernetes/apps/observability/gatus/app.yaml` (same shape as the jellyfin
entry):

```yaml
              - name: opsa
                url: https://opsa.cloud.karthikeyay.com/health
                interval: 1m
                conditions: ["[STATUS] == 200"]
```

### B4. Renovate

No `renovate.json5` change is required: the `helm-values` and `kubernetes`
managers already scan `kubernetes/**.yaml`, and semver tags on GHCR order
natively - Renovate will PR bumps for the app-template `image.tag` values and
the Job's container tags. Keep the inline `# Renovate-managed` comment on every
pinned tag (convention). Optional nicety - group all three images into one PR:

```json5
    { matchPackageNames: ["/^ghcr.io\\/lurkingryuu\\/opsa-/"], groupName: "opsa" },
```

---

## 5. Part C - Cutover runbook (order matters)

1. **opsa PR** with all of Part A -> merge to `main`.
2. **Tag `v1.1.0`** -> `build-images.yaml` pushes
   `ghcr.io/lurkingryuu/opsa-{excretor,garnisher,digester}:1.1.0`; mark the three
   GHCR packages **public**; `release.yaml` independently cuts the GitHub
   release.
3. **Rotate production secrets**: issue a new Slack client secret and signing
   secret for the new deployment rather than copying credentials from the old
   dotenvx setup. The DB password is CNPG-generated, so nothing to rotate there.
4. **Slack app config**: add `https://opsa.cloud.karthikeyay.com/auth/callback`
   to the OAuth redirect URLs.
5. **MinIO prep** (console at `minio.home.karthikeyay.com` or `mc`):
   create bucket `opsa`, create a read-only access key scoped to it, upload the
   export as `slack-export-<version>.zip` where `<version>` matches
   `ARCHIVE_VERSION` in `digester-job.yaml`.
6. **homenet PR** with all of Part B (create and encrypt `secret.sops.yaml`
   from the example) -> merge to `main`.
7. **Reconcile**: `flux reconcile kustomization cluster-opsa --with-source`
   (or wait for the interval).

### Verification

- `kubectl -n opsa get cluster opsa-postgres` healthy; secret
  `opsa-postgres-app` exists.
- excretor pod Ready; logs show sqlx migrations applied. garnisher pod Ready.
- digester Job: `wait-for-schema` and `fetch-archive` init containers complete,
  main container logs the ingestion phases, Job reaches `Complete`.
- `curl https://opsa.cloud.karthikeyay.com/health` returns 200; Gatus shows
  the opsa endpoint green.
- Browser end-to-end: SPA loads its logged-out view -> `/auth` starts Slack OAuth
  -> callback returns to the SPA -> channels, messages, and search all work; the session
  cookie is `Secure; HttpOnly` and persists per `KEEP_LOGGED_IN_FOR_DAYS`.
- Cleanup after success: delete the old SSH-deploy GitHub repo secrets
  (`HOMELAB_SSH_*`, `DOTENV_PRIVATE_KEY_PROD`, `SLACK_ARCHIVE_URL`,
  `HOMELAB_HEALTH_CHECK_URL`, `DEPLOYMENT_WEBHOOK_URL`); confirm
  `deploy-homelab.yaml` is gone from the default branch; decommission the old
  server's containers.

### Re-ingestion procedure

Upload the new export to the MinIO `opsa` bucket as
`slack-export-<new-version>.zip`, bump `ARCHIVE_VERSION` in
`digester-job.yaml`, merge. Flux force-replaces the Job; the digester is
idempotent over existing rows.

---

## 6. Risks and gotchas

| Risk | Assessment |
|------|------------|
| Earlier pending SQLx migration on pre-existing dev DBs | SQLx 0.7 applies it safely; `IF NOT EXISTS` preserves existing tables. Never renumber or edit applied migrations |
| Routing `/assets` to excretor | Would break the SPA bundle; encoded as a comment in the HTTPRoute |
| `Secure` cookie behind TLS-terminating Gateway | Fine - browser<->gateway is HTTPS; the plaintext hop is cluster-internal |
| `verify_token` calls Slack `auth.test` on every `/api` request | Latency + rate-limit exposure under load; acceptable for a homelab archive. Future opsa improvement: cache verification for the JWT lifetime |
| excretor slim-runtime shared libs | Check `ldd` on first image build; add `libssl3` only if reqwest uses native-tls |
| Job + TTL + Flux interaction | Avoided: `force` annotation without `ttlSecondsAfterFinished` |
| Memory on the 12GB node | ~excretor 256Mi + garnisher 64Mi + CNPG ~512Mi + transient Job 512Mi - comfortable fit |
| Reusing legacy production credentials | Avoided by issuing fresh Slack secrets in step 3 instead of copying the old dotenvx values |
