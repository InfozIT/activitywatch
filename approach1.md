# InfozIT Tracker — Integration Plan
## Based on real `Notifications_BACKEND` codebase analysis

---

## Backend Codebase Summary (What We Found)

| Aspect | Detail |
|---|---|
| **Language / Runtime** | Node.js (ESM — `type: "module"`) |
| **Framework** | Express.js v5 |
| **Database** | **MySQL** (via Sequelize ORM v6) |
| **Queue System** | BullMQ + Redis |
| **Auth** | JWT (`jsonwebtoken`) + API Key (dual mode in `authVerify` middleware) |
| **API Key System** | Already exists — AES-256-CBC encrypted, bcrypt-hashed, stored in `project_keys` table |
| **Migration Tool** | Sequelize CLI (`.cjs` migration files) |
| **Port** | 3030 |

---

## Critical Discovery: Infrastructure Already Exists!

> [!IMPORTANT]
> **The backend already has almost everything we need.** We do NOT need to build a key system from scratch. Here's what's already there:

| Existing Piece | How We Reuse It |
|---|---|
| `projects` table | Each client project = one row here |
| `project_keys` table | The API key the desktop app uses to authenticate |
| `project_services` table | Tracks which services a project has enabled (EMAIL, PUSH, etc.) → **we add TRACKER** |
| `core_services` table | Registry of available services → **we add "ActivityTracker" as a new core service** |
| `verifyProjectKey` middleware | Reuse as-is to authenticate tracker sync requests |
| `authVerify` middleware | Reuse for dashboard/admin queries |
| BullMQ queue system | **We create a new `tracker.queue.js`** to process incoming event batches async |
| AES-256 key utilities | Reuse `projectKey.util.js` — no new key format needed |
| `logActivity` utility | Reuse for audit logging device registrations |

---

## Database Plan: MySQL (Keep What We Have)

> [!NOTE]
> **We stick with MySQL** — no need to add a new database engine. MySQL handles the volume of activity events well with proper indexing. TimescaleDB would only be necessary at millions of events/day per user, which is far beyond initial scale.

### New Tables to Add

**Table 1: `tracker_devices`** — Registered desktop machines
```sql
CREATE TABLE tracker_devices (
  id            CHAR(36)     NOT NULL PRIMARY KEY,  -- UUID
  project_id    CHAR(36)     NOT NULL,
  device_name   VARCHAR(255),             -- e.g. "Johns MacBook Pro"
  hostname      VARCHAR(255),
  os            ENUM('macOS','Windows','Linux'),
  os_version    VARCHAR(100),
  device_token  TEXT         NOT NULL,   -- JWT issued to this device
  status_id     CHAR(36)     NOT NULL,   -- ACTIVE/INACTIVE enum
  last_seen_at  DATETIME,
  registered_at DATETIME     NOT NULL    DEFAULT NOW(),
  FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

**Table 2: `tracker_events`** — Raw activity events from desktop
```sql
CREATE TABLE tracker_events (
  id            BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  device_id     CHAR(36)     NOT NULL,
  project_id    CHAR(36)     NOT NULL,
  event_time    DATETIME(3)  NOT NULL,   -- millisecond precision
  bucket        VARCHAR(100),            -- 'aw-watcher-window', 'aw-watcher-afk'
  app           VARCHAR(255),
  title         TEXT,
  url           TEXT,                    -- for browser watchers
  duration_ms   INT,                     -- milliseconds
  is_afk        TINYINT(1),
  extra_data    JSON,                    -- catch-all for future fields
  synced_at     DATETIME     NOT NULL    DEFAULT NOW(),
  INDEX idx_project_time (project_id, event_time),
  INDEX idx_device_time  (device_id, event_time),
  FOREIGN KEY (device_id)  REFERENCES tracker_devices(id),
  FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

**No new migration for projects/project_keys** — they already exist and we reuse them.

---

## New Routes to Add to `server.js`

```js
// In server.js — add these 3 lines:
import trackerDeviceRouter from "./src/routes/tracker/trackerDevice.routes.js";
import trackerSyncRouter   from "./src/routes/tracker/trackerSync.routes.js";
import trackerQueryRouter  from "./src/routes/tracker/trackerQuery.routes.js";

app.use("/api/tracker/devices", trackerDeviceRouter);  // Device registration
app.use("/api/tracker/sync",    trackerSyncRouter);    // Desktop → backend data push
app.use("/api/tracker/query",   trackerQueryRouter);   // Dashboard queries
```

---

## Complete File Structure to Create

```
src/
├── routes/
│   └── tracker/
│       ├── trackerDevice.routes.js    NEW
│       ├── trackerSync.routes.js      NEW
│       └── trackerQuery.routes.js     NEW
│
├── controllers/
│   └── tracker/
│       ├── trackerDevice.controller.js   NEW
│       ├── trackerSync.controller.js     NEW
│       └── trackerQuery.controller.js    NEW
│
├── services/
│   └── tracker/
│       ├── trackerDevice.service.js      NEW
│       ├── trackerSync.service.js        NEW
│       └── trackerQuery.service.js       NEW
│
├── models/
│   └── tracker/
│       ├── trackerDevice.model.js        NEW
│       └── trackerEvents.model.js        NEW
│
├── middlewares/
│   └── trackerAuth.middleware.js         NEW  (verifies device JWT token)
│
└── queues/
    └── tracker/
        └── tracker.queue.js              NEW  (async event ingestion via BullMQ)

migrations/
├── 20260916000100-create-tracker-devices.cjs   NEW
└── 20260916000200-create-tracker-events.cjs    NEW
```

---

## API Endpoints Design

### 1. Device Registration
```
POST /api/tracker/devices/register
Header: x-project-key: prj_<encrypted_payload>_<random>   ← existing key format

Body: {
  "deviceName": "Johns MacBook Pro",
  "hostname":   "johns-macbook.local",
  "os":         "macOS",
  "osVersion":  "14.5"
}

Response 201: {
  "success": true,
  "deviceToken": "<JWT valid for 1 year>",
  "deviceId": "uuid-xxx"
}
```

**How it works:**
- Uses the **existing** `projectKeyVerify` middleware to validate the API key
- Creates a row in `tracker_devices`
- Issues a **device-specific JWT** (separate from user JWT) containing `{ deviceId, projectId }`
- Desktop stores this JWT and uses it for all future sync requests

---

### 2. Event Sync (Desktop → Backend)
```
POST /api/tracker/sync/events
Header: Authorization: Bearer <device_token>

Body: {
  "batchId": "uuid-batch",    ← for deduplication
  "events": [
    {
      "time":       "2026-09-15T10:00:00.000Z",
      "bucket":     "aw-watcher-window",
      "duration_ms": 45200,
      "data": {
        "app":   "Google Chrome",
        "title": "GitHub - InfozIT Tracker"
      }
    }
  ]
}

Response 200: {
  "success": true,
  "accepted": 142,
  "rejected": 0
}
```

**Processing flow:**
- `trackerAuth` middleware verifies the device JWT → attaches `req.device`
- Controller validates payload
- Pushes to **BullMQ tracker queue** (non-blocking — responds immediately to desktop)
- Worker processes and bulk-inserts into `tracker_events`
- `batchId` used for idempotency (duplicate detection)

---

### 3. Dashboard Queries
```
GET /api/tracker/query/summary?projectId=xxx&from=2026-09-01&to=2026-09-15
Header: Authorization: Bearer <user_JWT>   ← existing user auth

GET /api/tracker/query/timeline?deviceId=xxx&date=2026-09-15

GET /api/tracker/query/devices?projectId=xxx
```

---

## Authentication Flow — How Pieces Connect

```
[Admin Dashboard]
     │
     │ Creates project → generates project API key (existing flow)
     │
     ▼
[InfozIT Tracker Desktop App]
     │
     │ First launch: User enters API key (prj_xxx)
     │
     ├──► POST /api/tracker/devices/register
     │         ├── verifyProjectKey middleware (EXISTING — no changes)
     │         └── Issues device JWT → saved to ~/.config/infozit/config.toml
     │
     │ Every 60s sync:
     ├──► POST /api/tracker/sync/events
     │         ├── trackerAuth middleware (NEW — verifies device JWT)
     │         └── BullMQ job → bulk insert into tracker_events
     │
[Admin Dashboard]
     │
     └──► GET /api/tracker/query/summary
               ├── authVerify middleware (EXISTING — user JWT)
               └── Aggregated data from tracker_events
```

---

## Service Registration: ActivityTracker as a Core Service

The backend has a `core_services` table that lists available services. We need to:

1. **Seed a new core service** called `"ActivityTracker"` in `core_services`
2. When a project wants to use the tracker, they enable it via `project_services` (this already works)
3. The `checkProjectServiceActive` middleware (already exists) can then gate access

This means the tracker is naturally part of the platform's **service billing/enablement model** — consistent with email, push notifications, etc.

---

## Desktop App Changes

In the ActivityWatch/InfozIT Tracker codebase, we need to build:

### New file: `aw-client/aw_client/infozit_sync.py`
```python
# Background thread that:
# 1. Reads config: device_token, backend_url
# 2. Queries local SQLite via aw-client for new events since last_synced_at
# 3. POSTs batch to /api/tracker/sync/events with device_token
# 4. On success: updates last_synced_at in config
# 5. On failure: queues events in persistqueue for retry
```

### New file: `aw-client/aw_client/infozit_config.py`
```python
# Reads/writes ~/.config/infozit/config.toml containing:
# [tracker]
# backend_url  = "https://api.infozit.com"
# project_key  = "prj_xxx"          # entered by user at setup
# device_token = "eyJhbGc..."       # received after registration
# device_id    = "uuid-xxx"
# last_synced_at = "2026-09-15T..."
```

### First-launch Qt Dialog
- New Qt dialog in `aw-qt` that shows when `device_token` is missing from config
- User enters API key → calls `/api/tracker/devices/register` → saves device_token
- After success: dialog closes, normal tracking starts

---

## Implementation Phases

### Phase 1 — Backend Foundation (2–3 days)
- [ ] Create 2 Sequelize migrations (`tracker_devices`, `tracker_events`)
- [ ] Create 2 Sequelize models
- [ ] Register models in `src/models/index.js`
- [ ] Create `trackerAuth.middleware.js` (device JWT verification)
- [ ] Seed `ActivityTracker` into `core_services`

### Phase 2 — Backend API (2–3 days)
- [ ] Create `trackerDevice.routes.js` + controller + service (device registration)
- [ ] Create `trackerSync.routes.js` + controller (event ingestion)
- [ ] Create `tracker.queue.js` + worker (BullMQ async processing)
- [ ] Create `trackerQuery.routes.js` + controller (dashboard queries)
- [ ] Register all routes in `server.js`

### Phase 3 — Desktop App (3–4 days)
- [ ] Create `infozit_config.py` (read/write config file)
- [ ] Create `infozit_sync.py` (background sync thread)
- [ ] Create first-launch Qt API key entry dialog
- [ ] Integrate sync thread into `aw-server/main.py` startup
- [ ] Test offline queue + retry

### Phase 4 — Installer (1–2 days)
- [ ] Update `aw.spec` → rename bundle to `InfozIT Tracker.app`
- [ ] Update `Makefile` → rename DMG/EXE targets
- [ ] Build and verify `.dmg` (macOS)
- [ ] Add Windows packaging script

---

## Open Questions Before We Start

> [!IMPORTANT]
> Please confirm these before we write the first line of code:

1. **Backend URL**: What is the production/staging URL? (e.g., `https://api.infozit.com`) — needed for the desktop app config.

2. **`ActivityTracker` service name**: Should this appear in the `core_services` table as `"ActivityTracker"`, `"Tracker"`, or something else?

3. **Device JWT expiry**: How long should a registered device stay valid? (Suggested: 1 year, but you can revoke manually)

4. **Data volume estimate**: Roughly how many events/minute does one user generate? (AFK watcher polls every ~10s, window watcher every ~1s during activity)

5. **Shall we start with Phase 1 (backend) or Phase 3 (desktop app) first?**
