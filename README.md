<p align="center">
  <img src="https://img.shields.io/badge/status-in%20development-blue?style=for-the-badge" alt="Status" />
  <img src="https://img.shields.io/badge/port-8081-informational?style=for-the-badge" alt="Port" />
  <img src="https://img.shields.io/badge/Java-21%2B-orange?style=for-the-badge&logo=openjdk&logoColor=white" alt="Java" />
  <img src="https://img.shields.io/badge/Spring%20Boot-4.x-6DB33F?style=for-the-badge&logo=springboot&logoColor=white" alt="Spring Boot" />
  <img src="https://img.shields.io/badge/database-PostgreSQL%2016-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL" />
  <img src="https://img.shields.io/badge/messaging-Apache%20Kafka-231F20?style=for-the-badge&logo=apachekafka&logoColor=white" alt="Kafka" />
</p>

# Trip & Media Service

> **The core service of the Navio platform** — a single Spring Boot 3 application housing three co-located modules: the **Trip Engine** (trip CRUD, revisions, sharing, forking), the **IAM Module** (identity, roles, ACLs, audit logging), and the **Media Module** (upload pipeline, thumbnails, safe rendering URLs). Runs on **port 8081** behind the NGINX reverse proxy.

All traffic from the frontend reaches this service through NGINX at paths `/v1/trips`, `/v1/share`, `/v1/media`, `/v1/me`, and `/v1/mod`. Other backend services communicate with it via internal HTTP on `localhost:8081/internal/v1/...`.

---

## Table of Contents

1. [Why One Service, Three Modules](#why-one-service-three-modules)
2. [Module Overview](#module-overview)
3. [API Reference](#api-reference)
4. [Database Schemas](#database-schemas)
5. [Events Produced](#events-produced)
6. [Kafka Topics](#kafka-topics)
7. [Internal Communication](#internal-communication)
8. [Tech Stack](#tech-stack)
9. [Running Locally](#running-locally)
10. [Implementation Phases](#implementation-phases)

---

## Why One Service, Three Modules

Trip access control is the most latency-sensitive operation in the platform. A user loading their trip triggers an ACL check on every request. Co-locating the **IAM module** in the same JVM as the **Trip Engine** means those checks are direct Java method calls — zero network overhead and zero serialization cost.

Media uploads are tightly coupled to trips: images are attached to stops, and the media processing worker needs to validate and update trip-linked media records. Keeping media in-process avoids a round-trip and simplifies transactionality.

This is a deliberate architectural choice — not an accident. The three modules share a JVM but own separate Postgres schemas (`trip`, `iam`, `media`) with no cross-schema joins.

---

## Module Overview

### Trip Engine

The product's core. Owns everything related to creating, editing, and sharing travel itineraries.

| Capability               | Description                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- |
| **Trip CRUD**            | Create, read, update, soft-delete trips. Stops and route data stored as JSONB columns (no MongoDB needed).    |
| **Revision History**     | Every edit snapshots the full trip document into `trip.revisions`. Rollback restores any prior state.         |
| **Visibility Control**   | Three-tier: `private` (owner only) → `unlisted` (anyone with link) → `public` (discoverable).                |
| **Share Links**          | Generate cryptographically random tokens. Optional expiry. Revocable. Public access via `GET /v1/share/{token}` requires no JWT. |
| **Forking**              | Deep-copy a trip with full attribution chain. Fork stores a snapshot of origin metadata so attribution survives even if the original trip is deleted. |
| **Collaborators**        | Grant/revoke `editor` or `viewer` permissions on a trip. ACL checks run in-process (same JVM as IAM).        |
| **Maps Integration**     | On stop change: calls Google Maps / Mapbox to compute route distance, duration, and encoded polyline. Stored in `route_legs` JSONB. Circuit-broken via Resilience4j — trip saves even if Maps is down. |
| **Full-Text Search**     | `tsvector` generated column on `trip.trips` (title, description, notes, stop names, tags). Maintained by a Postgres trigger. Exposed via an internal search endpoint consumed by the Community Service. |
| **Caffeine Cache**       | Recently-accessed trips cached in-process (max 200 entries, 5-minute TTL). No Redis.                         |

### IAM Module

The identity and access control backbone for the entire platform. Embedded in this service, not a standalone microservice.

| Capability           | Description                                                                                                   |
| -------------------- | ------------------------------------------------------------------------------------------------------------- |
| **User Profile Sync**| On first authenticated request, mirrors Keycloak JWT claims (`sub`, `email`, `name`) into `iam.users`.       |
| **Roles**            | Three-tier: `user` (default) → `mod` → `admin`. Stored in `iam.users.role`.                                  |
| **Ban / Unban**      | Moderators and admins can ban users. Full ban history tracked in `iam.bans`. Active ban = latest record with `unbanned_at IS NULL`. |
| **ACL Engine**       | Generic resource-level access control via `iam.acl_entries`. Permissions: `owner`, `editor`, `viewer`. One entry per user per resource (upsert on grant). |
| **Audit Log**        | Append-only `iam.audit_log`. Records every security-sensitive action: bans, ACL grants/revocations, role changes, visibility changes, share link operations, mod deletions. Never updated or deleted. |
| **Event Publishing** | Publishes `UserRegistered.v1`, `UserProfileUpdated.v1`, `UserBanned.v1`, `UserUnbanned.v1` to Kafka via the Transactional Outbox pattern. |

**Why IAM lives here:** Other services (Community, EV Intelligence) validate JWTs independently via Spring Security + Keycloak JWKS. They don't need to call IAM for auth. But they do consume IAM Kafka events (e.g., `UserBanned.v1`) for downstream enforcement. The Trip Engine, however, needs ACL checks on every trip operation — and by being in the same JVM, those checks are free.

### Media Module

Owns the safe upload pipeline for images attached to trip stops and community posts.

| Capability              | Description                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Pre-Signed Uploads**  | Client requests an upload URL → service validates MIME type and size → generates pre-signed MinIO URL → inserts `media.media` record with `status=pending`. Client uploads directly to MinIO. |
| **Processing Worker**   | Embedded `@Scheduled` component polls `media.media WHERE status = 'uploaded'` every 5 seconds. Pipeline: download → MIME magic-byte validation → optional virus scan → thumbnail generation (200px + 600px) → upload thumbnails to MinIO → mark `status=ready`. |
| **Safe Rendering**      | Only `status=ready` media records return URLs. Blocked or pending media returns 404.                              |
| **Size & Type Limits**  | Max 10 MB. Allowed: JPEG, PNG, WebP, GIF.                                                                         |
| **Cross-Module Linking**| Trip Engine validates `mediaIds[]` on trip stop updates in-process. Community Service validates media via internal HTTP call to this service. |

---

## API Reference

### Public Endpoints (via NGINX)

All public endpoints require a JWT in `Authorization: Bearer <token>` unless noted.

#### Trip Engine

| Method   | Path                                         | Auth         | Description                                             |
| -------- | -------------------------------------------- | ------------ | ------------------------------------------------------- |
| `POST`   | `/v1/trips`                                  | JWT          | Create trip. Auto-grants `owner` ACL. Default visibility: `private`. |
| `GET`    | `/v1/trips`                                  | JWT          | List caller's trips (paginated, newest first).          |
| `GET`    | `/v1/trips/{tripId}`                         | JWT          | Load trip. ACL check: owner / editor / viewer.          |
| `PATCH`  | `/v1/trips/{tripId}`                         | JWT          | Update trip. ACL: owner or editor. Creates revision snapshot. |
| `DELETE` | `/v1/trips/{tripId}`                         | JWT          | Soft-delete trip. ACL: owner only. Forks are NOT deleted — they retain an origin snapshot. |
| `GET`    | `/v1/trips/{tripId}/revisions`               | JWT          | List revisions (newest first). ACL: owner or editor.    |
| `POST`   | `/v1/trips/{tripId}/rollback`                | JWT          | Body: `{ "revisionId": "..." }`. Restores snapshot. ACL: owner or editor. |
| `POST`   | `/v1/trips/{tripId}/visibility`              | JWT          | Body: `{ "visibility": "private" \| "unlisted" \| "public" }`. ACL: owner only. |
| `POST`   | `/v1/trips/{tripId}/share-links`             | JWT          | Generate share token. Returns `{ tokenId, token, url }`. |
| `DELETE` | `/v1/trips/{tripId}/share-links/{tokenId}`   | JWT          | Revoke share link.                                      |
| `GET`    | `/v1/share/{token}`                          | **None**     | Resolve share token → read-only trip view.              |
| `POST`   | `/v1/trips/{tripId}/fork`                    | JWT          | Fork trip. Must be public/unlisted or caller has viewer+. |
| `GET`    | `/v1/trips/{tripId}/forks`                   | JWT          | List public forks of a trip.                            |
| `GET`    | `/v1/trips/{tripId}/origin`                  | JWT          | Return fork origin metadata.                            |
| `POST`   | `/v1/trips/{tripId}/permissions`             | JWT          | Grant `editor` or `viewer` to a user. ACL: owner only.  |
| `GET`    | `/v1/trips/{tripId}/permissions`             | JWT          | List trip collaborators. ACL: owner only.               |

#### IAM Module

| Method   | Path                              | Auth       | Description                                  |
| -------- | --------------------------------- | ---------- | -------------------------------------------- |
| `GET`    | `/v1/me`                          | JWT        | Current user profile from `iam.users`.       |
| `PATCH`  | `/v1/me/preferences`              | JWT        | Update notification / theme preferences (JSONB). |
| `POST`   | `/v1/mod/users/{userId}/ban`      | Mod/Admin  | Ban user. Body: `{ reason }`. Writes to `iam.bans`. |
| `POST`   | `/v1/mod/users/{userId}/unban`    | Mod/Admin  | Lift active ban. Sets `unbanned_at`.         |

#### Media Module

| Method | Path                      | Auth | Description                                                                       |
| ------ | ------------------------- | ---- | --------------------------------------------------------------------------------- |
| `POST` | `/v1/media/upload-url`    | JWT  | Body: `{ filename, mimeType, sizeBytes }`. Returns `{ mediaId, uploadUrl, expiresIn }`. |
| `POST` | `/v1/media/complete`      | JWT  | Body: `{ mediaId }`. Signals upload done; triggers processing pipeline.           |
| `GET`  | `/v1/media/{mediaId}`     | JWT  | Returns metadata and URLs. Only resolves when `status=ready`.                     |

---

### Internal Endpoints (service-to-service, `localhost:8081` only)

Protected by `X-Internal-Auth` shared secret header. Never exposed through NGINX.

#### IAM

| Method   | Path                              | Called By              | Description                                                    |
| -------- | --------------------------------- | ---------------------- | -------------------------------------------------------------- |
| `POST`   | `/internal/v1/users/sync`         | Self (on JWT arrival)  | Upsert Keycloak user profile into `iam.users`.                 |
| `GET`    | `/internal/v1/users/{userId}`     | Community Service      | Get user profile (display name, role, status).                 |
| `POST`   | `/internal/v1/acl/check`          | Community Service      | Body: `{ resourceType, resourceId, userId, requiredPermission }` → `allow` / `deny`. |
| `POST`   | `/internal/v1/acl/grant`          | Self (Trip Engine)     | Grant permission entry.                                        |
| `DELETE` | `/internal/v1/acl/revoke`         | Self (Trip Engine)     | Remove permission entry.                                       |
| `GET`    | `/internal/v1/acl/list`           | Self                   | `?resourceType=trip&resourceId=...` — list all permissions.    |

#### Trip Engine

| Method | Path                               | Called By         | Description                                              |
| ------ | ---------------------------------- | ----------------- | -------------------------------------------------------- |
| `GET`  | `/internal/v1/trips/search`        | Community Service | `?q=...&page=...&size=...` — full-text search against `trip.trips` (public/unlisted only). |
| `GET`  | `/internal/v1/trips/{tripId}`      | AI Orchestrator   | Fetch full trip for LLM prompt context.                  |

#### Media

| Method | Path                              | Called By         | Description                                  |
| ------ | --------------------------------- | ----------------- | -------------------------------------------- |
| `GET`  | `/internal/v1/media/{mediaId}`    | Community Service | Validate media exists and `status=ready`.    |

---

## Database Schemas

This service exclusively owns three Postgres schemas on the shared `pg-primary` instance. No other service reads from or writes to these schemas.

### `trip` — Trip Engine

| Table              | Purpose                                                                                              |
| ------------------ | ---------------------------------------------------------------------------------------------------- |
| `trip.trips`       | Core trip records. JSONB columns: `stops`, `route_legs`, `ev_profile`, `forked_from`, `dates`, `stats`. `tsvector` column maintained by trigger. |
| `trip.revisions`   | Full trip document snapshots. Capped at last 100 per trip. `revision_number` is monotonic per trip. |
| `trip.share_tokens`| Share link tokens (URL-safe, 32+ chars). Support optional `expires_at` and `revoked_at`.            |
| `trip.outbox`      | Transactional Outbox for trip domain events → Kafka `trip.events.v1`.                               |

**Key indexes:**
- `idx_trips_owner` — paginated trip list per user
- `idx_trips_visibility` — public/unlisted feed queries
- `idx_trips_forked_from` — JSONB expression index for fork lineage
- `idx_trips_search` — GIN index on `tsvector` (partial: public/unlisted, active only)
- `idx_trips_tags` — GIN index on `tags TEXT[]`

### `iam` — IAM Module

| Table             | Purpose                                                                                             |
| ----------------- | --------------------------------------------------------------------------------------------------- |
| `iam.users`       | Keycloak profile mirror (`sub`, `email`, `display_name`) + app fields (`role`, `status`, `bio`, `preferences` JSONB). |
| `iam.bans`        | Ban/unban history. Active ban = latest record with `unbanned_at IS NULL`.                           |
| `iam.acl_entries` | Resource-level permissions. One row per user per resource (enforced via unique constraint).         |
| `iam.audit_log`   | Append-only security audit trail. Covers bans, ACL changes, role changes, visibility changes, mod actions. |
| `iam.outbox`      | Transactional Outbox for IAM domain events → Kafka `iam.events.v1`.                                |

**Key indexes:**
- `idx_bans_active` — partial index for "is user currently banned?" (fast path)
- `idx_acl_resource` — "who has access to this trip?"
- `idx_acl_user` — "what resources does this user have access to?"
- `idx_audit_log_actor`, `idx_audit_log_resource`, `idx_audit_log_target` — audit queries

### `media` — Media Module

| Table          | Purpose                                                                                                   |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| `media.media`  | Upload tracking. States: `pending` → `uploaded` → `processing` → `ready` / `blocked`. Stores original and thumbnail URLs, dimensions, MIME type, size. |

**Processing worker** polls `WHERE status = 'uploaded'` every 5 seconds. All state transitions are written back to this table.

---

## Events Produced

All events follow the standard Navio envelope:

```json
{
  "eventId": "evt_01J...ulid",
  "eventType": "TripCreated.v1",
  "occurredAt": "2026-03-27T10:00:00Z",
  "producer": "trip-media-service",
  "partitionKey": "trip_8b1f2c",
  "trace": { "traceId": "...", "spanId": "..." },
  "payload": {}
}
```

### Trip Events → `trip.events.v1`

| Event Type                 | Partition Key | Trigger                                          | Consumers                             |
| -------------------------- | ------------- | ------------------------------------------------ | ------------------------------------- |
| `TripCreated.v1`           | `tripId`      | New trip saved                                   | Community (search denorm)             |
| `TripUpdated.v1`           | `tripId`      | Trip fields / stops changed                      | Community (search denorm)             |
| `TripDeleted.v1`           | `tripId`      | Trip soft-deleted                                | Community (remove from search)        |
| `TripVisibilityChanged.v1` | `tripId`      | Visibility transition (private/unlisted/public)  | Community (feed eligibility)          |
| `TripForked.v1`            | `tripId`      | Fork created                                     | Community (notify original owner)     |
| `TripShared.v1`            | `tripId`      | Share link created or shared with a user         | Community (notify recipient)          |

### IAM Events → `iam.events.v1`

| Event Type              | Partition Key | Trigger                        | Consumers                                        |
| ----------------------- | ------------- | ------------------------------ | ------------------------------------------------ |
| `UserRegistered.v1`     | `userId`      | First JWT arrival, user synced | Community (author name cache)                    |
| `UserProfileUpdated.v1` | `userId`      | Display name / avatar changed  | Community (author name sync in posts/comments)   |
| `UserBanned.v1`         | `userId`      | Mod/admin bans a user          | Community (enforce ban, send in-app notification) |
| `UserUnbanned.v1`       | `userId`      | Mod/admin lifts a ban          | Community (restore access)                       |

**Delivery guarantee:** Events are written to the outbox table within the same database transaction as the domain write. An embedded `@Scheduled` publisher polls every 500 ms, publishes to Kafka, and marks rows as published. At-least-once delivery; consumers deduplicate on `eventId` (ULID).

---

## Kafka Topics

| Topic           | Produced By    | Consumed By                                              |
| --------------- | -------------- | -------------------------------------------------------- |
| `trip.events.v1`| This service   | Community Service (search denorm, notifications)         |
| `iam.events.v1` | This service   | Community Service (author name sync, ban enforcement)    |

This service does **not** consume any Kafka topics.

---

## Internal Communication

### This service calls:

| Dependency            | How         | What For                                                       |
| --------------------- | ----------- | -------------------------------------------------------------- |
| **Google Maps / Mapbox** | HTTP (external) | Route distance, duration, encoded polyline on stop changes. Resilience4j: 5s timeout + circuit breaker. Fallback: trip saves with `route_legs` marked unavailable. |
| **MinIO**             | SDK         | Pre-signed upload URL generation; media file download for processing. |

### Other services call this service:

| Caller              | Endpoint                           | Purpose                               |
| ------------------- | ---------------------------------- | ------------------------------------- |
| Community Service   | `GET /internal/v1/trips/search`    | Full-text trip search                 |
| Community Service   | `GET /internal/v1/users/{userId}`  | Fetch author display name             |
| Community Service   | `POST /internal/v1/acl/check`      | Permission check on trips             |
| Community Service   | `GET /internal/v1/media/{mediaId}` | Validate media before attaching to post |
| AI Orchestrator     | `GET /internal/v1/trips/{tripId}`  | Fetch trip context for LLM prompt     |

---

## Tech Stack

| Layer              | Technology                      | Notes                                                            |
| ------------------ | ------------------------------- | ---------------------------------------------------------------- |
| **Runtime**        | Java 21+ / Spring Boot 4.x      |                                                                  |
| **Web**            | Spring Web MVC                  | Synchronous request handling                                     |
| **Persistence**    | Spring Data JPA + Flyway        | Schema migrations run on startup                                 |
| **Database**       | PostgreSQL 16                   | Schemas: `trip`, `iam`, `media`                                  |
| **Messaging**      | Apache Kafka                    | Transactional Outbox → `trip.events.v1`, `iam.events.v1`         |
| **Caching**        | Caffeine (in-process JVM)       | Trips: max 200, 5 min TTL. No Redis.                             |
| **Auth**           | Keycloak (OIDC/JWT)             | JWT validated via JWKS endpoint (`spring-security-oauth2-resource-server`) |
| **Object Storage** | MinIO / local filesystem        | Uploads, thumbnails, exports                                     |
| **Resilience**     | Resilience4j                    | Circuit breakers + timeouts on Maps API and MinIO calls          |
| **Config**         | Spring Cloud Config Server      | Reads from `localhost:8888`. Stores feature flags, topic names, timeouts, refresh intervals. Secrets come from environment variables — not Config Server. |
| **Boilerplate**    | Lombok                          |                                                                  |

---

## Running Locally

### Prerequisites

- Java 21+
- PostgreSQL 16 with `trip`, `iam`, `media` schemas created
- Kafka broker at `localhost:9092` with topics `trip.events.v1` and `iam.events.v1`
- Keycloak at `localhost:8180/realms/navio`
- MinIO at `localhost:9000`
- Spring Cloud Config Server at `localhost:8888`

### Environment Variables

```bash
POSTGRES_URL=jdbc:postgresql://localhost:5432/navio
POSTGRES_USER=navio
POSTGRES_PASSWORD=<your-password>
KEYCLOAK_ISSUER_URI=http://localhost:8180/realms/navio
MINIO_URL=http://localhost:9000
MINIO_ACCESS_KEY=<your-key>
MINIO_SECRET_KEY=<your-secret>
INTERNAL_AUTH_SECRET=<shared-secret>
MAPS_API_KEY=<google-or-mapbox-key>
```

### Build & Run

```bash
./mvnw clean package -DskipTests
java -Xmx256m -jar target/Trip-Media-Service-*.jar
```

### Health Check

```
GET http://localhost:8081/actuator/health
```

---

## Implementation Phases

| Phase | What Gets Built                                                                                           |
| ----- | --------------------------------------------------------------------------------------------------------- |
| **1** | IAM Module: Keycloak JWT validation, user profile sync, roles, ban/unban, ACL engine, audit log, IAM outbox → Kafka |
| **2** | Trip Engine: Trip CRUD with JSONB stops/route_legs, revision history & rollback, Maps integration, Caffeine cache |
| **3** | Trip Engine: Share links, forking with attribution, visibility transitions, collaborator permissions, trip outbox → Kafka |
| **6** | Media Module: Pre-signed upload URLs, embedded processing worker, thumbnail generation, safe rendering URLs |

> Phases 1–3 are the critical path. Phase 6 (media) can run in parallel with Phases 4–5 (EV + Community) since it only depends on Phase 1 being complete.

---

<p align="center"><sub>Part of <a href="https://github.com/bestprappy/Navio-Server">Navio-Server</a> — built by <a href="https://github.com/bestprappy">bestprappy</a></sub></p>
