# Smart Landslide Risk Monitoring System

A working full-stack prototype dashboard for monitoring slope conditions
using soil moisture, rainfall, vibration, distance and other sensors, with
a live risk score, alerting, historical analytics, and a built-in sensor
simulator (for testing without physical hardware).

> ⚠️ **Prototype risk indicator — not a scientifically validated landslide
> prediction system.** The risk formula is an adjustable engineering-course
> demonstration, not a geotechnical model. See "About This Configuration"
> on the Configuration page in the app.

---

## What's inside

```
landslide-monitor/
│
├── frontend/
│   ├── index.html        Your original dashboard UI, preserved as-is
│   └── app.js             All the JavaScript that talks to the backend
│
├── backend/
│   ├── server.js           Entry point — starts everything
│   ├── db.js                SQLite schema + connection
│   ├── riskEngine.js        Centralized risk-score formula
│   ├── simulator.js         Realistic, gradually-changing sensor generator
│   ├── alerts.js            Alert generation with cooldown/de-duplication
│   ├── ingest.js             Shared pipeline: simulator AND real hardware
│   │                          both flow through this one function
│   ├── validate.js           Input validation
│   ├── events.js             Server-Sent Events (live update) hub
│   ├── http-utils.js         Small helpers for the raw Node http server
│   ├── env.js                  Loads backend/.env if present
│   ├── routes/
│   │   ├── sensors.js       GET latest/history, POST new reading
│   │   ├── alerts.js         GET/POST/DELETE/acknowledge alerts
│   │   ├── analytics.js      Historical stats for the Analytics page
│   │   ├── config.js          GET/PUT/reset configuration
│   │   └── system.js          health, status, SSE stream, simulator control
│   ├── tests/
│   │   └── api.test.js       Automated end-to-end test (10 checks)
│   ├── scripts/
│   │   └── seed.js            Optional: manually (re)generate demo history
│   ├── data/                   landslide.db lives here (auto-created)
│   ├── package.json
│   └── .env.example
│
├── .gitignore
└── README.md   ← you are here
```

### Why no Express, no `npm install` step that can fail

This project deliberately uses only **Node.js's own built-in modules**
(`http` for the web server, `node:sqlite` for the database) instead of
Express, cors, body-parser, socket.io, etc. That means:

- `npm install` has **nothing to download** — it can't fail because a
  package registry is unreachable, a version conflicts, or a native module
  fails to compile.
- The whole app — frontend, backend, database, live updates — runs with
  **one command** and **zero internet access** required at runtime (the
  only optional internet use is loading the Chart.js *charting* library
  from a CDN for the graphs; if that's blocked, every number, card, alert
  and the live risk score still work — only the line/bar charts fall back
  to a text message).

It still behaves exactly like an Express app from the outside: JSON REST
API, CORS headers, and a `/api/events` Server-Sent-Events stream for live
updates (no page reloads).

---

## Requirements

- **Node.js v22.13.0 or newer** (LTS "Jod" or newer). This project uses
  Node's built-in `node:sqlite` module, added in Node 22.5 and enabled by
  default (no flag needed) from Node 22.13 onward.
  Check your version:
  ```
  node -v
  ```
  If you're on an older Node 22.x and see an error mentioning
  `node:sqlite`, either upgrade Node, or temporarily run:
  ```
  node --experimental-sqlite server.js
  ```
- A modern browser (Chrome, Edge, Firefox, Safari).
- No database server, no Docker, no build step.

---

## Installation & running (2 commands)

```bash
cd backend
npm install
npm start
```

Then open your browser to:

```
http://localhost:3000
```

That's it — the backend serves the frontend too, so you don't run a
separate frontend server. On first run it automatically:

1. Creates `backend/data/landslide.db` and the database tables.
2. Pre-populates about 4 minutes of realistic simulated history so the
   dashboard and charts aren't empty the very first time you open it.
3. Starts the sensor simulator (scenario: **normal**, one reading every
   2 seconds) and starts broadcasting live updates over SSE.

Stop the server any time with `Ctrl+C`.

### Optional: `.env` file

Copy `backend/.env.example` to `backend/.env` to change the port,
simulation interval, alert cooldown, CORS origins, an API key for
`POST /api/sensors`, etc. All settings are optional — sensible defaults
are used if you skip this.

### Optional: regenerate demo history on demand

```bash
cd backend
npm run seed
```

This wipes and rebuilds ~2 hours of realistic history. Restart the server
afterward to see it.

---

## How to test the complete system

### Automated test (10 checks, ~1 second)

```bash
cd backend
npm test
```

This boots a real copy of the server on a throwaway port and database,
then checks: health check, default config, sensor validation (rejects bad
data), a valid hardware-style reading is stored and retrievable, a
high-risk reading produces an alert, configuration can be updated and
persists, reset-to-defaults works, analytics returns proper stats,
alerts can be cleared, and unknown routes return 404.

### Manual end-to-end walkthrough

1. `npm start`, then open `http://localhost:3000`.
2. The header should show **● LIVE** and **Backend: ONLINE** within a
   couple of seconds.
3. Sensor cards should show real (non-`undefined`, non-`—`) numbers that
   change every ~2 seconds.
4. Click a scenario button (e.g. **High Risk**) — soil moisture, rain and
   vibration should trend up over the next ~15–30 seconds, the risk score
   should climb, and new entries should appear under **Alerts** in the
   sidebar and on the Dashboard's "Recent Alerts" panel.
5. Open the **Alerts** page — filter by sensor/severity, then **Clear
   All** and confirm the list empties.
6. Open **Analytics** — current/min/max/average/trend should be populated
   from real historical data; switch the time range (1H/6H/24H).
7. Open **Configuration**, change a threshold (e.g. Soil Moisture
   Threshold), click **Save Settings** — you should see a confirmation
   toast, and the Dashboard's sensor card threshold label should update
   immediately (proving the new value round-tripped through SQLite and
   back).
8. Click **Reset to Defaults** and confirm the values return to 70 / 50 /
   3 / 25 / 50 / 75.
9. Open **System Health** — Backend/Database should read ONLINE/CONNECTED,
   and "Total readings" / "Last successful data received" should be live.
10. Stop the server (`Ctrl+C`) — within a few seconds the header should
    switch to **● OFFLINE** and a red "Backend Offline" banner should
    appear, while the dashboard keeps showing the last known values
    (nothing goes blank or shows `undefined`).
11. Restart the server (`npm start`) — the dashboard should automatically
    reconnect within a few seconds, no page refresh needed.

---

## API reference

All endpoints are under `http://localhost:3000/api/...` and return JSON.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | Basic liveness check |
| GET | `/api/status` | Full system status (db, simulator, uptime, counts) |
| GET | `/api/sensors/latest` | Most recent reading + risk |
| GET | `/api/sensors/history?minutes=60&limit=200` | Downsampled history |
| POST | `/api/sensors` | Ingest one reading (simulator *or* real hardware) |
| GET | `/api/alerts?sensor=&severity=&status=&limit=` | List alerts |
| POST | `/api/alerts` | Manually create an alert |
| DELETE | `/api/alerts` | Clear all alerts |
| PATCH | `/api/alerts/:id/acknowledge` | Acknowledge/resolve one alert |
| GET | `/api/analytics?minutes=60` | Stats + chart series for a time window |
| GET | `/api/config` | Current configuration |
| PUT | `/api/config` | Update one or more thresholds/weights |
| POST | `/api/config/reset` | Reset configuration to defaults |
| GET | `/api/simulation/scenarios` | List available scenario names |
| POST | `/api/simulation/scenario` | Set active scenario `{ "scenario": "rain" }` |
| POST | `/api/simulation/start` \| `/stop` \| `/reset` | Control the simulator |
| GET | `/api/events` | Server-Sent Events stream (`reading`, `alert`, `status`) |

### Connecting real hardware later

An Arduino/ESP device can POST directly to the same endpoint the simulator
uses — there is no separate "real" pipeline to build later:

```
POST /api/sensors
Content-Type: application/json

{
  "soilMoisture": 65,
  "temperature": 30,
  "humidity": 75,
  "rainfall": true,
  "vibration": false,
  "distance": 48.7,
  "ir": false
}
```

Response:
```json
{ "success": true, "message": "Sensor data received", "data": { ... } }
```

Values are validated (range-checked) and rejected with `400` and a
descriptive `errors` array if malformed. If you set `API_KEY` in
`backend/.env`, the device must also send header `x-api-key: <value>`.

---

## Database

SQLite, stored as a single file at `backend/data/landslide.db` (created
automatically). Three tables: `sensor_readings`, `alerts`,
`configuration`. Column types are plain `TEXT` / `REAL` / `INTEGER`,
chosen so the schema maps cleanly onto PostgreSQL later
(`TEXT`→`TIMESTAMP`/`VARCHAR`, `REAL`→`DOUBLE PRECISION`,
`INTEGER` 0/1 →`BOOLEAN`) if you ever outgrow SQLite.

Readings older than `RETENTION_DAYS` (default 7) are cleaned up
automatically every few hours.

---

## Troubleshooting

**"Cannot find module 'node:sqlite'" or a `node:sqlite` error on startup**
Your Node.js version is too old. Install Node 22.13+ from
[nodejs.org](https://nodejs.org) and run `node -v` to confirm.

**Port 3000 already in use**
Set a different port: create `backend/.env` from `.env.example` and change
`PORT=3000` to e.g. `PORT=4000`, or run `PORT=4000 npm start`.

**Dashboard shows "● OFFLINE" right after starting**
Give it a few seconds — the browser needs a moment to connect. If it
persists, check the terminal running `npm start` for errors, and confirm
you're opening `http://localhost:3000` (not a different port).

**Charts show "Chart library unavailable"**
The page tried to load Chart.js from a CDN (`cdnjs.cloudflare.com`) and
your network blocked it. Every number on the dashboard is still live —
only the line/bar charts are affected. Check your internet connection, or
firewall/proxy settings, if you want the charts too.

**I changed a threshold and nothing happened**
Make sure you clicked **Save Settings** (not just typed in the box). A
green confirmation toast should appear in the bottom-right.

**I want to start over with a clean database**
Stop the server, delete `backend/data/landslide.db` (and any
`-wal`/`-shm` files next to it), then run `npm start` again — it will
recreate everything and reseed initial history automatically.
