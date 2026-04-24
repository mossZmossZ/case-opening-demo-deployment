# Deployment Reference — Zenith Case Opening Demo

> Single source of truth for the **next repo** (GitOps / Kustomize) that will
> consume images from this one. A future AI tool reads this file to generate
> the Kubernetes manifests / Kustomize overlays. Keep it up to date whenever
> an image, port, or env var changes.
>
> **Last updated: 2026-04-25** — Phase 3 microservices complete, CI/CD pipeline
> operational, Docker images scanned and published, ready for Phase 4 (Kubernetes).

---

## 1. Registry & image names

- **Registry:** Docker Hub (`docker.io`)
- **Namespace:** value of the `DOCKERHUB_USERNAME` GitHub secret
- **Visibility:** repos should exist on Docker Hub; auth uses `DOCKERHUB_TOKEN`

| Service        | Image (repo path)                                       | Context in repo              |
|----------------|---------------------------------------------------------|------------------------------|
| client         | `${DOCKERHUB_USERNAME}/case-opening-demo-client`        | `./client`                   |
| auth-service   | `${DOCKERHUB_USERNAME}/case-opening-demo-auth-service`  | `./services/auth-service`    |
| game-service   | `${DOCKERHUB_USERNAME}/case-opening-demo-game-service`  | `./services/game-service`    |
| admin-service  | `${DOCKERHUB_USERNAME}/case-opening-demo-admin-service` | `./services/admin-service`   |
| prize-service  | `${DOCKERHUB_USERNAME}/case-opening-demo-prize-service` | `./services/prize-service`   |

> The legacy `./server/` monolith is **not** part of the microservice
> deployment and is not built by the release pipeline.

---

## 2. Tag strategy

Every push to **`main`** produces two tags per image:

| Tag           | Example       | Nature    | Use                                       |
|---------------|---------------|-----------|-------------------------------------------|
| `sha-<7char>` | `sha-8865e29` | Immutable | **GitOps — always pin to this**           |
| `main-latest` | `main-latest` | Floating  | Human sanity-check / quick dev pulls only |

**Never pin Kustomize or Argo/Flux to a floating tag.** Floating tags are
overwritten on every merge and defeat GitOps reproducibility.

---

## 3. GitOps handoff artifact — `image-refs.json`

At the end of every successful Stage 4 run, the workflow uploads an artifact
named **`image-refs`** (see `.github/workflows/ci.yml` → `publish-refs` job).
Download it from the run and feed it to the GitOps repo's workflow.

Shape:

```json
{
  "commit":  "8865e29abc...full sha...",
  "commit7": "8865e29",
  "branch":  "main",
  "repo":    "<org>/case-opening-demo",
  "runUrl":  "https://github.com/.../actions/runs/123456",
  "qualityGates": {
    "loadTest": true,
    "dast":     true
  },
  "services": [
    {
      "service": "client",
      "image":   "docker.io/<user>/case-opening-demo-client",
      "tag":     "sha-8865e29",
      "ref":     "docker.io/<user>/case-opening-demo-client:sha-8865e29"
    }
    // ... one entry per service (5 total)
  ]
}
```

**Recommended GitOps repo flow** (next AI tool builds this):
1. Listen for a `repository_dispatch` event from this repo's pipeline.
2. Download the `image-refs` artifact using `actions/download-artifact`.
3. For each entry in `services[]`, run:
   ```
   kustomize edit set image <service-name>=<ref>
   ```
4. Commit & push the overlay change; ArgoCD / Flux syncs to the cluster.

---

## 4. Runtime contract per service

Ports and env vars below mirror `docker-compose-prod.yml`. Treat these as
the Deployment spec for Kubernetes manifests.

### client (frontend, nginx)
- **Port (container):** `80`
- **Port (service):** recommend `80` (ClusterIP) + Ingress on `/`
- **Env:** none
- **Depends on:** `auth-service`, `game-service`, `admin-service` (via
  in-cluster DNS — see `client/nginx.conf`)
- **Note:** nginx proxies `/api/auth/` → auth, `/api/game/` → game,
  `/api/admin/` → admin. In K8s these must resolve as
  `http://auth-service:4001`, `http://game-service:4002`,
  `http://admin-service:4003` — keep the K8s Service names identical.

### auth-service
- **Port:** `4001`
- **Env:**
  - `MONGO_URI` → `mongodb://mongodb:27017/zenith_auth`
  - `JWT_SECRET` → **required secret** (K8s Secret)
  - `PORT` → `4001`
- **Depends on:** `mongodb`

### game-service
- **Port:** `4002`
- **Env:**
  - `MONGO_URI` → `mongodb://mongodb:27017/zenith_game`
  - `REDIS_URL` → `redis://redis:6379`
  - `PRIZE_SERVICE_URL` → `http://prize-service:4004`
  - `PORT` → `4002`
- **Depends on:** `mongodb`, `redis`, `prize-service`

### admin-service  (BFF — no own DB)
- **Port:** `4003`
- **Env:**
  - `AUTH_SERVICE_URL` → `http://auth-service:4001`
  - `GAME_SERVICE_URL` → `http://game-service:4002`
  - `PRIZE_SERVICE_URL` → `http://prize-service:4004`
  - `S3_ENDPOINT`, `S3_REGION`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`,
    `S3_BUCKET` → **K8s Secret** (optional; admin image upload)
  - `PORT` → `4003`
- **Depends on:** `auth-service`, `game-service`, `prize-service`

### prize-service  (internal only — no Ingress)
- **Port:** `4004`
- **Env:**
  - `MONGO_URI` → `mongodb://mongodb:27017/zenith_prize`
  - `REDIS_URL` → `redis://redis:6379`
  - `PORT` → `4004`
- **Depends on:** `mongodb`, `redis`

### mongodb (stateful)
- Image: `mongo:7`
- Port: `27017`
- Needs a PersistentVolumeClaim mounted at `/data/db`.
- Databases used (logical): `zenith_auth`, `zenith_game`, `zenith_prize`.

### redis (stateful, but ephemeral-ok)
- Image: `redis:7-alpine`
- Port: `6379`
- A PVC at `/data` is nice-to-have (cache warmup); acceptable as emptyDir.

---

## 5. Ingress surface

Only two things are externally reachable:

| Path          | Backend       | Notes                                     |
|---------------|---------------|-------------------------------------------|
| `/*`          | `client:80`   | SPA + nginx API proxy (everything)        |

All `/api/*` traffic enters through the `client` nginx — the microservices
themselves should remain `ClusterIP` (no LoadBalancer, no Ingress).
`prize-service` in particular is internal-only by design.

---

## 6. Secrets the GitOps repo will need to create

Kubernetes `Secret` objects (names are suggestions):

| Secret           | Keys                                                                          | Consumed by        |
|------------------|-------------------------------------------------------------------------------|--------------------|
| `zenith-auth`    | `JWT_SECRET`                                                                  | auth-service       |
| `zenith-s3`      | `S3_ENDPOINT`, `S3_REGION`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_BUCKET` | admin-service (optional) |
| `dockerhub-pull` | docker-registry type, from `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`           | all Deployments (`imagePullSecrets`) |

---

## 7. Health checks (recommended for K8s probes)

All Node services expose nothing special yet — use a TCP probe on their
port as a minimum. The client exposes HTTP:

- `client`: `GET /` → 200  (readiness + liveness)
- `auth/game/admin/prize-service`: TCP probe on their port until a
  dedicated `/healthz` is added (tracked as future work).

---

## 8. CI/CD pipeline — single file, four stages

All pipelines are consolidated into **`.github/workflows/ci.yml`**.  
Trigger: **push to `main` only**. There are no PR pipelines and no manual dispatch.

```
push to main
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 1 · Build & Scan (parallel, one job per service)       │
│                                                              │
│  [client]  [auth-service]  [game-service]  [admin-service]  [prize-service]
│  Hadolint → buildx build (local) → Trivy CVE gate           │
│  EXIT-CODE 1 on HIGH/CRITICAL with fix → blocks Stage 2     │
└──────────────────────────┬──────────────────────────────────┘
                           │ all 5 must pass
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 2 · Push to Docker Hub (parallel, one job per service) │
│                                                              │
│  [client]  [auth-service]  [game-service]  [admin-service]  [prize-service]
│  Cache hit from Stage 1 → build+push:                        │
│    sha-<7char>   ← immutable (GitOps / kustomize uses this)  │
│    main-latest   ← floating  (humans / quick dev pulls)      │
│  Each job uploads a refs-<service>.json artifact             │
└──────┬───────────────────────────────────────────────────────┘
       │
       ├──────────────────────┐
       ▼                      ▼
┌─────────────┐     ┌─────────────────────┐
│ Stage 3a    │     │ Stage 3b            │
│ Load Test   │     │ DAST (OWASP ZAP)    │
│ [k6 smoke]  │     │ [zap-baseline.py]   │
│             │     │                     │
│ Pulls sha-  │     │ Pulls sha-tagged    │
│ tagged imgs │     │ imgs from DockerHub │
│ via compose │     │ via compose         │
│             │     │                     │
│ continue-on │     │ continue-on-error:  │
│ -error: true│     │ true (soft gate)    │
└──────┬──────┘     └──────────┬──────────┘
       │                       │
       └──────────┬────────────┘
                  │ waits for both (pass or fail)
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 4 · Publish image-refs.json                            │
│                                                              │
│  Combines refs-*.json → image-refs.json (includes           │
│  qualityGates.loadTest + qualityGates.dast status)          │
│  Uploads as artifact `image-refs` (retained 90 days)        │
│  Writes run summary with stage results table                 │
│                                                              │
│  Only runs if Stage 2 (push) succeeded.                     │
└─────────────────────────────────────────────────────────────┘
```

### Stage gates summary

| Stage | Job | Hard gate? | Blocks if fails? |
|-------|-----|-----------|-----------------|
| 1 | Build & Scan (Trivy) | **Yes** | Stops all downstream stages |
| 2 | Push to Docker Hub | Yes | Stops Stage 3 & 4 |
| 3a | Load Test (k6) | No | Reports failure, Stage 4 still runs |
| 3b | DAST (ZAP) | No | Reports failure, Stage 4 still runs |
| 4 | Publish refs | — | Final step, publishes regardless of Stage 3 |

### Artifacts produced per run

| Artifact name | Contents | Retention |
|---|---|---|
| `refs-<service>` (×5) | Per-service image ref JSON | 30 days |
| `image-refs` | Combined refs for GitOps (all 5 services) | 90 days |
| `k6-report` | k6 summary JSON | 7 days |
| `zap-report` | ZAP HTML + JSON + Markdown | 7 days |

GitOps / Kustomize updates are **not** done here — that's the next repo's
job. Download the `image-refs` artifact from the run and feed it to the
GitOps repo workflow. This repo's contract ends at "images pushed to Docker
Hub + `image-refs.json` artifact available".

---

## 9. Docker image security hardening

All four microservice runner stages (`auth`, `game`, `admin`, `prize`) include:

```dockerfile
RUN rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/bin/npm \
           /usr/local/bin/npx
```

**Why:** `node:20-alpine` ships with npm@10.8.2, which bundles its own copies
of `tar@6.2.1`, `cross-spawn@7.0.3`, `glob@10.4.2`, and `minimatch@9.0.5` —
all with known HIGH CVEs. These packages belong to npm's own internals; our
application code never uses them. Removing npm from the runner stage:

1. Clears all 11 Trivy HIGH findings so the CI gate passes clean.
2. Reduces the production container attack surface (no package manager present).

The `client` image (`FROM nginx:alpine`) is unaffected — nginx has no npm.

All five services also have a `package-lock.json` (committed) and their
Dockerfiles use `npm ci` (instead of `npm install`) to guarantee
reproducible, deterministic builds regardless of when the CI runner executes.

---

## 10. Current state — 2026-04-25

### What is complete

| Area | Status |
|------|--------|
| Phase 3 microservices | **Complete** — auth, game, admin, prize services running; client nginx proxy configured |
| Docker images | **Published** — all 5 images on Docker Hub, SHA-tagged |
| CI/CD pipeline | **Operational** — `.github/workflows/ci.yml`, push-to-main trigger |
| Trivy CVE gate | **Passing** — 0 HIGH/CRITICAL after npm removal from runner stages |
| Lock files | **Committed** — all 5 services have `package-lock.json` for reproducible builds |
| Kubernetes setup | **In progress** — cluster provisioned, GitOps repo not yet created |

### What is next (Phase 4)

1. **Create the GitOps repo** — Kubernetes manifests + Kustomize overlays.
   Use this `deployment.md` (§§1–7) as the input spec. The AI tool building
   the GitOps repo needs:
   - The 5 image names from §1
   - The tag format `sha-<7char>` from §2
   - The `image-refs.json` shape from §3
   - The env vars, ports, and secrets from §4 and §6
   - The ingress surface from §5
   - The health check guidance from §7

2. **Add a GitOps trigger step to `ci.yml`** — after Stage 4 (`publish-refs`),
   fire a `repository_dispatch` to the GitOps repo so it pulls `image-refs.json`
   and runs `kustomize edit set image` automatically.

3. **Add `/healthz` endpoints** to all four Node services (§7) so K8s readiness
   and liveness probes work properly without relying on TCP-only checks.

4. **Istio + observability** (Phase 4 roadmap) — service mesh, Kiali, Jaeger.
