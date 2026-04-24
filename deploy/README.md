# Zenith Case Opening — Kubernetes Deployment (Kustomize)

GitOps target for the `case-opening-demo` app repo. Images are built and
published by the app repo's CI; this repo only holds the Kubernetes manifests.
A GitHub Action from the app repo runs `kustomize edit set image` on
`overlays/prod/kustomization.yaml` to pin each service to a new `sha-<7>` tag.

## Layout

```
deploy/
├── base/                    # raw manifests (env-agnostic)
│   ├── namespace.yaml
│   ├── client-{deployment,service}.yaml        # type: LoadBalancer
│   ├── auth-service-{deployment,service}.yaml
│   ├── game-service-{deployment,service}.yaml
│   ├── admin-service-{deployment,service}.yaml
│   ├── prize-service-{deployment,service}.yaml
│   ├── mongodb-{statefulset,service}.yaml      # PVC on nutanix-volume, 10Gi
│   ├── redis-{statefulset,service}.yaml        # PVC on nutanix-volume, 1Gi
│   └── kustomization.yaml
└── overlays/
    └── prod/
        └── kustomization.yaml   # namespace + images: block (edit point)
```

## Namespace

`demo-nkp-dx96j` — created by `base/namespace.yaml`.

## Prerequisites — create Secrets manually

Kustomize does NOT create the Secrets. Create them once in the
`demo-nkp-dx96j` namespace before the first apply.

### 1. `zenith-auth` — REQUIRED (auth-service will crash-loop without it)

| Key          | Value                                        |
|--------------|----------------------------------------------|
| `JWT_SECRET` | long random string (32+ chars recommended)   |

```bash
kubectl -n demo-nkp-dx96j create secret generic zenith-auth \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)"
```

### 2. `zenith-s3` — OPTIONAL (only if admin image upload is used)

| Key                     | Example                        |
|-------------------------|--------------------------------|
| `S3_ENDPOINT`           | `https://s3.example.com`       |
| `S3_REGION`             | `us-east-1`                    |
| `S3_ACCESS_KEY_ID`      | `AKIA...`                      |
| `S3_SECRET_ACCESS_KEY`  | `...`                          |
| `S3_BUCKET`             | `zenith-prize-images`          |

```bash
kubectl -n demo-nkp-dx96j create secret generic zenith-s3 \
  --from-literal=S3_ENDPOINT='https://s3.example.com' \
  --from-literal=S3_REGION='us-east-1' \
  --from-literal=S3_ACCESS_KEY_ID='...' \
  --from-literal=S3_SECRET_ACCESS_KEY='...' \
  --from-literal=S3_BUCKET='...'
```

`admin-service` mounts this secret with `optional: true`, so you can skip it
until image uploads are needed.

## Deploy

```bash
kubectl apply -k deploy/overlays/prod
```

## Image update flow (GitOps)

The app repo's GitHub Action — after Stage 4 publishes `image-refs.json` —
should clone this repo and run (one command per service):

```bash
cd deploy/overlays/prod
kustomize edit set image client=docker.io/<user>/case-opening-demo-client:sha-<7>
kustomize edit set image auth-service=docker.io/<user>/case-opening-demo-auth-service:sha-<7>
kustomize edit set image game-service=docker.io/<user>/case-opening-demo-game-service:sha-<7>
kustomize edit set image admin-service=docker.io/<user>/case-opening-demo-admin-service:sha-<7>
kustomize edit set image prize-service=docker.io/<user>/case-opening-demo-prize-service:sha-<7>

git add kustomization.yaml
git commit -m "chore: bump images to sha-<7>"
git push
```

Argo CD / Flux picks up the commit and rolls the deployments. Only the
`images:` block in `overlays/prod/kustomization.yaml` changes per release.

## External exposure

Only `client` is external. `client` Service is `type: LoadBalancer` on port
80. All microservices + MongoDB + Redis are `ClusterIP` (internal only).

Get the external IP after first apply:

```bash
kubectl -n demo-nkp-dx96j get svc client
```

## Storage

| Workload | PVC size | StorageClass     | Access  |
|----------|----------|------------------|---------|
| mongodb  | 10Gi     | `nutanix-volume` | RWO     |
| redis    | 1Gi      | `nutanix-volume` | RWO     |

Both use a StatefulSet `volumeClaimTemplate` — PVCs are created automatically
on first apply and persist across pod restarts.

## Verify

```bash
kubectl -n demo-nkp-dx96j get pods
kubectl -n demo-nkp-dx96j get svc
kubectl -n demo-nkp-dx96j get pvc
kubectl -n demo-nkp-dx96j get statefulset
```

## Runtime wiring (matches `deployment.md` §4)

| Service       | Port | MONGO_URI                                    | Other env                                            |
|---------------|------|----------------------------------------------|------------------------------------------------------|
| client        | 80   | —                                            | nginx proxies `/api/auth`, `/api/game`, `/api/admin` |
| auth-service  | 4001 | `mongodb://mongodb:27017/zenith_auth`        | `JWT_SECRET` from secret `zenith-auth`               |
| game-service  | 4002 | `mongodb://mongodb:27017/zenith_game`        | `REDIS_URL`, `PRIZE_SERVICE_URL`                     |
| admin-service | 4003 | —                                            | `AUTH_/GAME_/PRIZE_SERVICE_URL`, S3 envFrom (opt)    |
| prize-service | 4004 | `mongodb://mongodb:27017/zenith_prize`       | `REDIS_URL`                                          |
| mongodb       | 27017| —                                            | —                                                    |
| redis         | 6379 | —                                            | —                                                    |
