# Architecture — Zenith Case Opening Demo

## Phase 1: Monolith (Local npm) — Complete

```
┌─────────────────────────────────────────────────┐
│                   Browser                        │
│  React SPA (User UI + Admin UI)                  │
└──────────────────────┬──────────────────────────┘
                       │ HTTP
┌──────────────────────▼──────────────────────────┐
│              Node.js (Express)                   │
│  ┌─────────────┐  ┌──────────────┐              │
│  │  /api/game   │  │  /api/admin  │              │
│  │  - register  │  │  - login     │              │
│  │  - spin      │  │  - prizes    │              │
│  │  - stats     │  │  - rates     │              │
│  │  - leader    │  │  - history   │              │
│  └─────────────┘  │  - operations│              │
│                    └──────────────┘              │
│  ┌─────────────┐                                 │
│  │ Static Files│  (serves React build)           │
│  └─────────────┘                                 │
└──────────────────────┬──────────────────────────┘
                       │ Mongoose
┌──────────────────────▼──────────────────────────┐
│                  MongoDB                         │
│  Collections:                                    │
│  - users        (name, created_at)               │
│  - sessions     (user_id, attempts, results[])   │
│  - prizes       (name, tier, weight, stock, img) │
│  - admin_users  (username, password_hash)        │
└─────────────────────────────────────────────────┘
```

---

## Phase 2: Containerized Frontend/Backend (Docker Compose) — Complete

```
┌──────────────────────────────────────────────────────────────┐
│                      Docker Compose                           │
│                                                               │
│  ┌────────────────────┐     ┌──────────────────────────────┐ │
│  │    client:8080      │     │       server:4000            │ │
│  │  (nginx + React)    │────▶│     (Node.js Express)        │ │
│  │                     │     │                              │ │
│  │  - User UI          │     │  /api/game/* (public)        │ │
│  │  - Admin UI         │     │  /api/admin/* (JWT auth)     │ │
│  │  - /api/* → server  │     │  gameService.js (spin logic) │ │
│  └────────────────────┘     │  s3.js (image upload)        │ │
│                              └─────────────┬────────────────┘ │
│                                             │                  │
│                                ┌────────────▼───────────────┐ │
│                                │      mongodb:27017         │ │
│                                │      (mongo:7)             │ │
│                                │      volume: mongo-data    │ │
│                                └────────────────────────────┘ │
│                                                               │
│  External reverse proxy (outside compose) handles SSL.        │
│  CI/CD: GitHub Actions → Docker Hub images.                   │
└──────────────────────────────────────────────────────────────┘
```

**Key decisions:**
- nginx in client container proxies `/api/*` to server container
- MongoDB only exposed internally in prod (`expose`, not `ports`)
- External proxy terminates SSL → `host:8080` (frontend) + `host:4000` (API)

---

## Phase 3: Microservices + Redis (Docker Compose)

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Docker Compose (7 containers)                      │
│                                                                               │
│  ┌───────────────┐                                                            │
│  │ client:8080   │  nginx routes (unchanged public API surface):              │
│  │ (nginx+React) │  /api/auth/*   → auth-service:4001                         │
│  │               │  /api/game/*   → game-service:4002                         │
│  │ Frontend does │  /api/admin/*  → admin-service:4003                        │
│  │ NOT change.   │                                                            │
│  └───────┬───────┘                                                            │
│          │                                                                    │
│  ┌───────▼───────┐  ┌────────────────┐  ┌────────────────┐                   │
│  │ auth-service  │  │  game-service  │  │ admin-service  │                   │
│  │    :4001      │  │    :4002       │  │    :4003       │                   │
│  │               │  │                │  │                │                   │
│  │  DB: zenith   │  │  DB: zenith    │  │  (no database) │                   │
│  │  _auth        │  │  _game         │  │  Orchestrator  │                   │
│  │               │  │                │  │  / BFF only    │                   │
│  │ Owns:         │  │ Owns:          │  │                │                   │
│  │  admin_users  │  │  users         │  │ Calls:         │                   │
│  │               │  │  sessions      │  │  auth-service  │                   │
│  │ Provides:     │  │                │  │  game-service  │                   │
│  │  JWT issue    │  │ Calls:         │  │  prize-service │                   │
│  │  JWT verify   │  │  prize-service │  │  S3 (upload)   │                   │
│  └───────────────┘  └───────┬────────┘  └────────┬───────┘                   │
│                              │                     │                          │
│                    ┌─────────▼─────────────────────▼──────────┐               │
│                    │           prize-service:4004              │               │
│                    │                                          │               │
│                    │  DB: zenith_prize                         │               │
│                    │  Owns: prizes collection                  │               │
│                    │                                          │               │
│                    │  SINGLE AUTHORITY for:                    │               │
│                    │  - Prize CRUD                             │               │
│                    │  - Stock management (atomic $inc)         │               │
│                    │  - Weighted random engine                 │               │
│                    │  - Rate configuration                    │               │
│                    └──────────┬───────────────────────────────┘               │
│                               │                                               │
│          ┌────────────────────┼────────────────────┐                          │
│          ▼                    ▼                     ▼                          │
│  ┌──────────────┐    ┌──────────────┐      ┌──────────────┐                  │
│  │   MongoDB    │    │    Redis     │      │   S3 / MinIO │                  │
│  │   :27017     │    │   :6379     │      │   (external) │                  │
│  │              │    │              │      │              │                  │
│  │ zenith_auth  │    │ prizes:act  │      │ Prize images │                  │
│  │ zenith_game  │    │ rates:cur   │      │              │                  │
│  │ zenith_prize │    │ feed:recent │      │              │                  │
│  └──────────────┘    └──────────────┘      └──────────────┘                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Service Ownership Table

| Service | Database | Collections | Role |
|---------|----------|-------------|------|
| **auth-service** | `zenith_auth` | `admin_users` | Admin JWT issuance + verification only |
| **game-service** | `zenith_game` | `users`, `sessions` | Player registration, session lifecycle, spin orchestration, stats, leaderboard |
| **prize-service** | `zenith_prize` | `prizes` | Single authority for all prize data, stock atomicity, weighted random |
| **admin-service** | (none) | (none) | BFF/orchestrator — authenticates admin, proxies to correct service |

### Public API Endpoints (unchanged from Phase 2)

**auth-service** — exposed at `/api/auth/*`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/login` | Admin login → JWT token |

**game-service** — exposed at `/api/game/*`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/game/prizes` | Active prizes for reel (reads Redis cache or proxies to prize-service) |
| POST | `/api/game/register` | Create player + session |
| POST | `/api/game/spin/:sessionId` | Execute spin → calls prize-service internally |
| GET | `/api/game/session/:sessionId` | Get session state |
| GET | `/api/game/stats` | Live drops, participant count, inventory summary |
| GET | `/api/game/leaderboard` | All drops sorted by tier |

**admin-service** — exposed at `/api/admin/*`

| Method | Path | Proxies to |
|--------|------|------------|
| POST | `/api/admin/login` | auth-service |
| GET | `/api/admin/dashboard` | game-service + prize-service (aggregates both) |
| GET | `/api/admin/prizes` | prize-service |
| POST | `/api/admin/prizes` | prize-service |
| PUT | `/api/admin/prizes/:id` | prize-service |
| DELETE | `/api/admin/prizes/:id` | prize-service |
| POST | `/api/admin/upload` | S3 directly (admin-service handles multipart) |
| GET | `/api/admin/rates` | prize-service |
| PUT | `/api/admin/rates` | prize-service |
| GET | `/api/admin/history` | game-service |
| POST | `/api/admin/operations/reset-sessions` | game-service |
| POST | `/api/admin/operations/reset-stock` | prize-service |
| POST | `/api/admin/operations/generate-dummy` | prize-service |

### Internal Service-to-Service API (Docker network only)

```
auth-service (called by admin-service):
  POST /internal/auth/verify       → validate JWT, return admin identity

prize-service (called by game-service + admin-service):
  GET  /internal/prizes            → list all prizes
  GET  /internal/prizes/active     → list active in-stock prizes
  POST /internal/prizes            → create prize
  PUT  /internal/prizes/:id        → update prize
  DELETE /internal/prizes/:id      → delete prize
  POST /internal/prizes/spin       → weighted random + atomic stock decrement
  GET  /internal/prizes/rates      → get all weights
  PUT  /internal/prizes/rates      → bulk update weights + invalidate cache
  POST /internal/prizes/reset-stock      → restore all stock to total
  POST /internal/prizes/generate-dummy   → seed demo prizes

game-service (called by admin-service):
  GET  /internal/game/history      → paginated drop history
  GET  /internal/game/dashboard    → participants, total opens, active sessions
  POST /internal/game/reset-sessions  → wipe users + sessions
```

### Data Flow: Spin (Phase 3)

```
User clicks Spin
       │
       ▼
  nginx → game-service
  POST /api/game/spin/:sessionId
       │
       ├── 1. Validate session exists + has attempts left (own DB)
       │
       ├── 2. POST prize-service /internal/prizes/spin
       │       ├── Load active prizes (Redis cache or DB)
       │       ├── Weighted random selection
       │       ├── Atomic $inc remainingStock: -1
       │       ├── If stock hit 0 → invalidate Redis prizes:active
       │       └── Return { prizeId, name, tier, iconKey, imageUrl }
       │
       ├── 3. Append result to session.results[] (own DB)
       │
       ├── 4. LPUSH Redis feed:recent (trim to 50)
       │
       ├── 5. If last attempt → set session.status = 'completed'
       │
       └── 6. Return { prize, attemptsLeft }
```

### Data Flow: Admin Updates Prize

```
Admin saves prize edit
       │
       ▼
  nginx → admin-service
  PUT /api/admin/prizes/:id
       │
       ├── 1. POST auth-service /internal/auth/verify (validate JWT)
       │
       ├── 2. PUT prize-service /internal/prizes/:id
       │       ├── Update prize in DB
       │       ├── Invalidate Redis prizes:active
       │       ├── Invalidate Redis rates:current
       │       └── Return updated prize
       │
       └── 3. Return updated prize to frontend
```

### Redis Cache Strategy

| Key | Type | Owner (write) | Reader(s) | TTL | Invalidation |
|-----|------|--------------|-----------|-----|--------------|
| `prizes:active` | String (JSON) | prize-service | game-service | 5 min | Prize create/update/delete/spin-to-zero |
| `rates:current` | String (JSON) | prize-service | game-service | None | `PUT /internal/prizes/rates` |
| `feed:recent` | List | game-service | game-service | Capped 50 | Each spin LPUSHes, LTRIM to 50 |
| `stats:summary` | String (JSON) | game-service | game-service | 10 sec | Time-based expiry |

### Directory Structure (Phase 3)

```
case-opening-demo/
├── client/                      # Unchanged from Phase 2
│   ├── Dockerfile
│   ├── nginx.conf               # Updated: routes to 3 backend services
│   └── src/
├── services/
│   ├── auth-service/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │       ├── index.js         # Express app, port 4001
│   │       ├── models/AdminUser.js
│   │       └── routes/auth.js   # /api/auth/login + /internal/auth/verify
│   ├── game-service/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │       ├── index.js         # Express app, port 4002
│   │       ├── models/User.js
│   │       ├── models/Session.js
│   │       ├── routes/game.js   # /api/game/* public endpoints
│   │       ├── routes/internal.js  # /internal/game/* for admin-service
│   │       └── lib/prizeClient.js  # HTTP client to prize-service
│   ├── prize-service/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │       ├── index.js         # Express app, port 4004
│   │       ├── models/Prize.js
│   │       ├── routes/internal.js  # /internal/prizes/* endpoints
│   │       ├── services/spin.js    # Weighted random + atomic stock
│   │       └── lib/redisCache.js   # Cache helpers
│   ├── admin-service/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │       ├── index.js         # Express app, port 4003
│   │       ├── routes/admin.js  # /api/admin/* proxied endpoints
│   │       ├── middleware/auth.js  # Calls auth-service to verify JWT
│   │       ├── lib/authClient.js   # HTTP client to auth-service
│   │       ├── lib/prizeClient.js  # HTTP client to prize-service
│   │       ├── lib/gameClient.js   # HTTP client to game-service
│   │       └── services/s3.js      # Image upload (stays here)
│   └── shared/                  # Shared utilities (optional npm workspace)
│       ├── package.json
│       ├── logger.js            # Structured JSON logging
│       ├── errors.js            # Common error classes
│       └── healthcheck.js       # GET /health for all services
├── docker-compose.yml           # 7 containers
├── docker-compose-prod.yml      # Pulls from Docker Hub
└── server/                      # Phase 2 monolith (kept as reference)
```

### docker-compose.yml (Phase 3)

```yaml
services:
  client:
    build: ./client
    ports: ["8080:80"]
    depends_on: [auth-service, game-service, admin-service]

  auth-service:
    build: ./services/auth-service
    expose: ["4001"]
    environment:
      MONGO_URI: mongodb://mongodb:27017/zenith_auth
      JWT_SECRET: ${JWT_SECRET}
      PORT: 4001
    depends_on:
      mongodb: { condition: service_healthy }

  game-service:
    build: ./services/game-service
    expose: ["4002"]
    environment:
      MONGO_URI: mongodb://mongodb:27017/zenith_game
      REDIS_URL: redis://redis:6379
      PRIZE_SERVICE_URL: http://prize-service:4004
      PORT: 4002
    depends_on:
      mongodb: { condition: service_healthy }
      redis: { condition: service_healthy }

  admin-service:
    build: ./services/admin-service
    expose: ["4003"]
    environment:
      AUTH_SERVICE_URL: http://auth-service:4001
      GAME_SERVICE_URL: http://game-service:4002
      PRIZE_SERVICE_URL: http://prize-service:4004
      S3_ENDPOINT: ${S3_ENDPOINT}
      S3_REGION: ${S3_REGION}
      S3_ACCESS_KEY_ID: ${S3_ACCESS_KEY_ID}
      S3_SECRET_ACCESS_KEY: ${S3_SECRET_ACCESS_KEY}
      S3_BUCKET: ${S3_BUCKET}
      PORT: 4003

  prize-service:
    build: ./services/prize-service
    expose: ["4004"]
    environment:
      MONGO_URI: mongodb://mongodb:27017/zenith_prize
      REDIS_URL: redis://redis:6379
      PORT: 4004
    depends_on:
      mongodb: { condition: service_healthy }
      redis: { condition: service_healthy }

  mongodb:
    image: mongo:7
    expose: ["27017"]
    volumes: [mongo-data:/data/db]
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    expose: ["6379"]
    volumes: [redis-data:/data]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  mongo-data:
  redis-data:
```

### nginx Routing (Phase 3)

```nginx
# client/nginx.conf — updated for Phase 3

location /api/auth/ {
    proxy_pass http://auth-service:4001;
}
location /api/game/ {
    proxy_pass http://game-service:4002;
}
location /api/admin/ {
    proxy_pass http://admin-service:4003;
}
location / {
    try_files $uri $uri/ /index.html;
}
```

No `/api/prizes/*` public route — prize-service is internal only.

---

## Phase 4: Kubernetes + Observability (Future)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                                │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     Istio Service Mesh                             │  │
│  │                                                                    │  │
│  │  ┌──────────┐  Istio   ┌──────────────┐  ┌──────────────┐        │  │
│  │  │ Ingress  │  Gateway │ frontend     │  │ auth-service │        │  │
│  │  │ Gateway  │─────────▶│ Deployment   │  │ Deployment   │        │  │
│  │  │          │          │ + Service    │  │ + Service    │        │  │
│  │  │          │          │ + HPA        │  │ + HPA        │        │  │
│  │  └──────────┘          └──────────────┘  └──────────────┘        │  │
│  │                                                                    │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│  │  │ game-service │  │admin-service │  │prize-service │            │  │
│  │  │ Deployment   │  │ Deployment   │  │ Deployment   │            │  │
│  │  │ + Service    │  │ + Service    │  │ + Service    │            │  │
│  │  │ + HPA        │  │ + Service    │  │ + HPA        │            │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │  │
│  │                                                                    │  │
│  │  Envoy sidecars on every pod → mTLS, traffic routing, telemetry   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─────────────────────── Data Layer ────────────────────────────────┐  │
│  │  ┌──────────────┐  ┌──────────────┐                               │  │
│  │  │   MongoDB    │  │    Redis     │                               │  │
│  │  │  StatefulSet │  │  StatefulSet │                               │  │
│  │  │  + PVC       │  │  + PVC       │                               │  │
│  │  └──────────────┘  └──────────────┘                               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─────────────────── Observability Stack ───────────────────────────┐  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │  │
│  │  │  Kiali   │  │  Jaeger  │  │Prometheus│  │ Grafana  │         │  │
│  │  │ Mesh UI  │  │ Tracing  │  │ Metrics  │  │Dashboards│         │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Database Schema (Current — Phase 2)

```
users
├── _id: ObjectId
├── name: String (required, trimmed)
└── createdAt: Date

sessions
├── _id: ObjectId
├── userId: ObjectId → users
├── playerName: String
├── totalAttempts: Number (1-10)
├── results: [{
│     attempt: Number
│     prizeId: ObjectId → prizes
│     prizeName: String
│     tier: enum(common, rare, epic, legendary)
│     iconKey: String
│     imageUrl: String
│     description: String
│     timestamp: Date
│   }]
├── status: enum(active, completed)
└── createdAt: Date

prizes
├── _id: ObjectId
├── name: String (required, trimmed)
├── description: String
├── tier: enum(common, rare, epic, legendary)
├── weight: Number (drop rate weight)
├── totalStock: Number
├── remainingStock: Number (atomic $inc on spin)
├── iconKey: String
├── imageUrl: String (S3 URL)
├── active: Boolean
└── createdAt: Date

admin_users
├── _id: ObjectId
├── username: String (unique)
├── passwordHash: String (bcrypt, 10 rounds)
└── createdAt: Date
```

### Phase 3 Database Ownership

```
zenith_auth (auth-service only):
  └── admin_users

zenith_game (game-service only):
  ├── users
  └── sessions

zenith_prize (prize-service only):
  └── prizes
```
