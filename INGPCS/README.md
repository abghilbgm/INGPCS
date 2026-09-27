# INGPCE — Industrial Safety Operations

Real-time industrial CCTV monitoring for PPE compliance, with a live
camera wall, incident queue, PostgreSQL event history, dynamic camera
management, and configurable alert rules.

Pre-loaded with your **16 real plant cameras**, auto-organized into 9 area
groups — synced into the database automatically on every app startup.

---

## Fix: "cameras from config.py not showing in the frontend"

If you upgraded from an older build and your database only ended up with a
few cameras (e.g. just CAM02) instead of all 16, here's why and how to fix
it - **no reinstall or data loss required**.

### Why this happens
`app/config.py` is only a **seed list** — cameras are copied from it into
PostgreSQL, and the frontend only ever reads from PostgreSQL, never from
the Python file directly. In an older version, if the seeding process was
interrupted partway (app crashed, service restarted mid-seed, etc.), any
camera that hadn't been created yet was **permanently skipped** on future
restarts. This build fixes that.

### The fix
`sync_seed_data()` (in `app/db.py`) now runs automatically on **every**
app startup, and separately checks **each individual camera**: if a camera
from `config.py` is missing from the database, it gets created. If it
already exists, it's left completely untouched. This makes it self-healing
— no matter how partial the database state is, restarting the app tops it
up automatically.

### How to fix it right now, 3 ways (easiest first)

**Option A — Dashboard button (recommended):**
Open the dashboard → **Cameras** tab → click **"⟲ Sync Default Cameras"**.
You'll see a confirmation like "Added 15 missing camera(s): CAM01, CAM03, ...".
The Feeds tab will now show all 16.

**Option B — Restart the app:**
```bash
PYTHON_BIN=python3.11 ./run_server.sh
```
On Windows, run `run_server.bat` from Command Prompt. Look for this line in
the console:
```
[ppe.db] Seed sync: X area group(s) created, Y camera(s) created, Z already existed
```

**Option C — API call directly:**
```bash
curl -X POST http://localhost:8080/api/cameras/sync-seed
```
Then verify:
```bash
curl -s http://localhost:8080/api/cameras | python3 -c "import sys,json; print(len(json.load(sys.stdin)))"
```
Should print `16`.

---

## Run Locally

Requirements: Python 3.10-3.12, PostgreSQL 14 or newer, and network access
to download the detection model on first run.

1. Install PostgreSQL and start its service. Create the app role and
  database using an administrator account:

  ```sql
  CREATE ROLE ppe_user LOGIN PASSWORD 'use-a-strong-password';
  CREATE DATABASE ppe_compliance OWNER ppe_user;
  ```

2. Copy `.env.example` to `.env`, set `POSTGRES_PASSWORD` to the role
  password, set `PPE_CAMERA_PASSWORD`, and configure alert integrations.
  Use a dedicated read-only account on each camera where supported.
3. Start the app:

```bash
./run_server.sh
```

The launcher creates `.venv`, installs Python requirements, downloads the
configured model if it is missing, and starts the server. It selects an
installed Python 3.10-3.12 automatically; set `PYTHON_BIN` to override it.
For Windows,
create and activate a virtual environment, install `requirements.txt`, run
`python scripts/download_model.py`, then run `run_server.bat`.

Open **http://localhost:8080/**. The first successful startup creates the
database tables and seeds the configured cameras and area groups. Expected
startup messages include:

```
[1/3] Loading detection model ...
[2/3] Connecting to PostgreSQL...
PostgreSQL pool ready ...
[ppe.db] Seed sync: 9 area group(s) created, 16 camera(s) created, 0 already existed
[3/3] Starting camera workers...
Reconciled 16 camera(s) from DB (16 started)
STARTUP COMPLETE - application is ready.
```
If startup fails, the full traceback prints in the console. Check database
connectivity with:
```bash
.venv/bin/python scripts/check_db_connection.py
```
The model download source and model license should be reviewed before
operational use. Validate detection performance against site conditions;
automated alerts are decision support and require human review.

---

## Your 16 Cameras & Area Groups

| Area Group | Cameras |
|---|---|
| Substation | CAM01 |
| MCC | CAM02 |
| SYLOC | CAM03, CAM04 |
| Pumphouse | CAM05, CAM06, CAM07 |
| Yard | CAM08, CAM16 |
| Gates & Security | CAM09, CAM13, CAM14, CAM15 |
| Safety | CAM10 |
| Facilities | CAM11 |
| Process | CAM12 |

**Editing an RTSP URL later:** Dashboard → **Cameras** tab → **Edit** →
change URL → Save (auto-restarts just that camera).

---

## Dashboard Tabs

- **Live Wall** — live annotated video grouped by area, with per-camera
  Reconnect (↻) button and a global "Reconnect All Cameras" button
- **Incidents** — image gallery with camera/area/type filters; shows the
  actual error + Retry button if loading fails
- **Analytics** — hourly trend, violation-type breakdown, by-area-group chart
- **Cameras** — add/edit/refresh/delete cameras; manage area groups;
  **"⟲ Sync Default Cameras"** button to top up any missing seed cameras
- **Alert Rules** — create/edit/delete DB-driven alert rules live

## Other Reliability Fixes in This Build

- Throttled DB status writes (was every loop tick per camera, now once
  every few seconds) — prevents connection pool exhaustion with many cameras
- Self-healing camera loop — a transient error no longer permanently kills
  a camera's worker
- Retry-safe violation logging — a violation that fails to save is retried
  on the next frame instead of being lost
- One camera failing to start no longer blocks the other 15 from starting
- `/api/violations` and `/api/stats` return a clear error message on DB
  failure instead of a generic "Failed to load"

## Camera Management API
```
GET/POST/PUT/DELETE /api/cameras[/{camera_id}]
POST /api/cameras/{camera_id}/refresh   # force reconnect
POST /api/cameras/sync-seed              # top up missing seed cameras
GET/POST/PUT/DELETE /api/area-groups
GET/POST/PUT/DELETE /api/rules
```

## Power BI
```bash
python powerbi/build_powerbi_dataset.py --host localhost --db ppe_compliance --user ppe_user --password ppe_password --out powerbi/powerbi_data
```

## Project Structure
```
ingpce/
├── README.md
├── app/
│   ├── config.py               # SEED_CAMERAS (16 cams) + SEED_AREA_GROUPS + tunables
│   ├── db.py                     # PostgreSQL + sync_seed_data() (the missing-cameras fix)
│   ├── camera_manager.py          # dynamic lifecycle, self-healing loop, throttled writes
│   ├── stream_handler.py, detector.py, compliance_engine.py
│   ├── rules_engine.py, alert_manager.py, ws_hub.py
│   └── main.py                     # FastAPI incl. POST /api/cameras/sync-seed
├── dashboard/                        # SPA incl. "Sync Default Cameras" + per-camera Reconnect buttons
├── scripts/                           # download_model, train, seed_rules, quick_test, check_db_connection
├── powerbi/
├── requirements.txt, .env.example
├── run_server.sh, run_server.bat
```

## Notes
- Existing camera records are intentionally not overwritten by seed sync.
  Update their RTSP URLs in Cameras after rotating the previously embedded
  camera credentials; new seed URLs use the username/password in `.env`.
- The API has no built-in user authentication or TLS. The default listener
  is local-only; do not expose it to a plant network or the internet without
  adding an authenticated TLS reverse proxy and network controls.
- Decision-support tool — keep human review in the loop for disciplinary actions.
- Fine-tune the model on your own footage (`scripts/train.py`) for best accuracy.
- Ensure camera footage usage complies with your organization's privacy policies.
