# VanSetu — Full Deployment Guide
## Render (Backend) + Vercel (Frontend)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [What You Need Before Starting](#2-what-you-need-before-starting)
3. [Step 1 — Deploy Backend on Render](#3-step-1--deploy-backend-on-render)
4. [Step 2 — Set Up MongoDB Atlas (Optional)](#4-step-2--set-up-mongodb-atlas-optional)
5. [Step 3 — Deploy Frontend on Vercel](#5-step-3--deploy-frontend-on-vercel)
6. [Step 4 — Connect Frontend to Backend](#6-step-4--connect-frontend-to-backend)
7. [Step 5 — Verify Everything Works](#7-step-5--verify-everything-works)
8. [Optional — AQI API Key (Live Air Quality)](#8-optional--aqi-api-key-live-air-quality)
9. [Optional — Telegram Bot](#9-optional--telegram-bot)
10. [Re-deploying After Code Changes](#10-re-deploying-after-code-changes)
11. [Custom Domain Setup](#11-custom-domain-setup)
12. [Environment Variables Reference](#12-environment-variables-reference)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Architecture Overview

```
                    ┌─────────────────────────────────┐
                    │         GitHub Repository        │
                    │   niharika19singh/van-setu       │
                    └──────────┬──────────────────┬───┘
                               │                  │
                    auto-deploy│             auto-deploy
                               │                  │
              ┌────────────────▼───┐    ┌──────────▼────────────┐
              │   RENDER           │    │   VERCEL               │
              │   Backend API      │    │   Frontend (React)     │
              │   (FastAPI/Python) │    │   (Vite build)         │
              │                    │    │                        │
              │  /api/*  endpoints │    │  vercel.json proxies   │
              │  rootDir: backend/ │    │  /api/* → Render       │
              │  port: $PORT       │    │  serves index.html SPA │
              │  region: singapore │    │                        │
              └────────────────────┘    └────────────────────────┘
                         ▲                         │
                         │    /api/* forwarded      │
                         └──────────────────────────┘
                              (Vercel proxy rewrite)

              ┌──────────────────────┐
              │  MongoDB Atlas       │  ← Optional, for corridor
              │  (free M0 cluster)   │    suggestions feature
              └──────────────────────┘
```

**Why this architecture:**
- Vercel handles the React frontend as static files — fast global CDN, zero config
- Render handles the Python/FastAPI backend with geospatial dependencies
- Vercel's `/api/*` proxy rewrite forwards API calls to Render — **no CORS issues**
- The frontend never directly calls the Render URL, so changing backends requires
  only one line in `vercel.json`

---

## 2. What You Need Before Starting

- **GitHub account** — repo already pushed at `github.com/niharika19singh/van-setu`
- **Render account** — create free at `render.com`
- **Vercel account** — create free at `vercel.com`
- **MongoDB Atlas account** — only if you want the corridor suggestions feature
  (create free at `mongodb.com/atlas`)

---

## 3. Step 1 — Deploy Backend on Render

### 3.1 Create a New Web Service

1. Go to **https://render.com** and log in
2. Click **"New +"** in the top-right corner
3. Select **"Blueprint"** from the dropdown

   > Blueprint reads `render.yaml` from your repo and creates services automatically.
   > This is the fastest way and avoids manual configuration errors.

4. Click **"Connect a repository"**
5. If GitHub is not connected yet:
   - Click **"Connect GitHub"**
   - Authorize Render to access your GitHub account
   - Under repository access, select **"Only select repositories"**
   - Choose `niharika19singh/van-setu`
   - Click **"Install & Authorize"**
6. You will be redirected back to Render — select `van-setu` from the list
7. Click **"Connect"**

### 3.2 Review the Blueprint

Render will read `render.yaml` and show you a preview:

```
Service: van-setu-backend
Type: Web Service
Runtime: Python
Region: Singapore
Root Directory: backend/
Build Command: pip install -r requirements.txt
Start Command: uvicorn app.main:app --host 0.0.0.0 --port $PORT
```

Review this and click **"Apply"**.

### 3.3 Set Secret Environment Variables

`render.yaml` defines environment variables but marks secrets as `sync: false`
meaning you must enter them manually in the dashboard. You will see them listed
with an empty input box.

Fill in the following (explained in detail in Section 12):

| Variable | Required? | What to enter |
|---|---|---|
| `OPENAQ_API_KEY` | No | Leave blank — fallback AQI data is used |
| `MONGODB_URI` | No | Leave blank for now — set after MongoDB Atlas setup |
| `TELEGRAM_BOT_TOKEN` | No | Leave blank unless you are setting up the bot |

Click **"Apply"** to start the first deployment.

### 3.4 Watch the Build Log

Render will now:
1. Clone your GitHub repo
2. `cd` into the `backend/` directory (rootDir)
3. Run `pip install -r requirements.txt`

This takes **4–8 minutes** on the first build because it compiles:
- `rasterio` (needs GDAL, C extensions)
- `geopandas`
- `shapely`
- `osmnx`
- `scipy`
- `python-telegram-bot`

You will see output like:
```
==> Cloning from https://github.com/niharika19singh/van-setu...
==> Running build command: pip install -r requirements.txt
Collecting fastapi>=0.109.0
...
Successfully installed fastapi-0.115.x uvicorn-0.27.x rasterio-1.3.x ...
==> Build successful
==> Starting service with: uvicorn app.main:app --host 0.0.0.0 --port $PORT
```

### 3.5 Watch the Startup Log

After the build, Render starts the FastAPI server. You will see:

```
============================================================
🚀 Starting VanSetu Platform API
============================================================

📂 Loading raster data...
  📂 Loading NDVI: /opt/render/project/src/data/delhi_ndvi_10m.tif
     Shape: (57, 79)
  📂 Loading LST: /opt/render/project/src/data/delhi_lst_modis_daily_celsius.tif
     Shape (original): (57, 79)
  ⚙️  Resampling LST to match NDVI...
  🧮 Computing Green Deficit Index...
  ✅ All raster data loaded successfully

📡 Initializing AQI data...
  ⚠️  Using fallback AQI data   ← normal if no API key

🔗 Corridor aggregation service initialized
💬 Community suggestions service initialized

✅ API Ready!
   Swagger UI: http://localhost:8000/docs
   API Base: http://localhost:8000/api
```

> **Note:** The AQI warning is normal — fallback data with 10 Delhi stations is used.
> The `raster_loaded: true` line is the critical one.

### 3.6 Confirm the Service is Live

1. On the Render dashboard, wait for the status badge to turn **green** and show "Live"
2. Render assigns a URL like: `https://van-setu-backend.onrender.com`
3. Click the URL or open it manually
4. Append `/health` to the URL: `https://van-setu-backend.onrender.com/health`
5. You should see:

```json
{
  "status": "healthy",
  "raster_loaded": true,
  "bounds": { "north": 28.87, "south": 28.4, "east": 77.35, "west": 76.73 }
}
```

If `raster_loaded` is `false`, the TIF files were not found — see Troubleshooting.

6. Also check the API docs at: `https://van-setu-backend.onrender.com/docs`
   You will see the full Swagger UI with all endpoints.

### 3.7 Note Your Backend URL

Copy your Render URL, e.g. `https://van-setu-backend.onrender.com`

You will need it in Step 4.

---

## 4. Step 2 — Set Up MongoDB Atlas (Optional)

Skip this section if you don't need the **corridor community suggestions feature**
(the upvote/comment feature on corridors). All other features work without MongoDB.

### 4.1 Create a Free MongoDB Atlas Cluster

1. Go to **https://www.mongodb.com/atlas** and sign up or log in
2. Click **"Build a Database"**
3. Select **"M0 Free"** tier (512 MB, enough for thousands of suggestions)
4. Choose a cloud provider and region — pick the one closest to Singapore
   (AWS `ap-southeast-1` or GCP `asia-southeast1`)
5. Give your cluster a name, e.g. `vansetu`
6. Click **"Create Deployment"**

### 4.2 Create a Database User

1. In the setup wizard, you will see **"Create a database user"**
2. Enter a username, e.g. `vansetu_app`
3. Enter a strong password (Atlas can generate one — copy it now)
4. Click **"Create Database User"**

### 4.3 Whitelist Render's IP Addresses

MongoDB Atlas blocks connections by default. You need to allow Render's IPs.

1. In the wizard, click **"Add IP Address"**
2. For simplicity (Render uses dynamic IPs), click **"Allow Access from Anywhere"**
   which adds `0.0.0.0/0`

   > This is standard for managed PaaS platforms like Render. Atlas still requires
   > username/password authentication.

3. Click **"Finish and Close"**

### 4.4 Get the Connection String

1. On the Atlas dashboard, click **"Connect"** on your cluster
2. Select **"Drivers"**
3. Select **Driver: Python**, **Version: 3.12 or later**
4. Copy the connection string, it looks like:
   ```
   mongodb+srv://vansetu_app:<password>@vansetu.abc123.mongodb.net/?retryWrites=true&w=majority
   ```
5. Replace `<password>` with the password you created in step 4.2

### 4.5 Add the URI to Render

1. Go to your Render dashboard → `van-setu-backend` service
2. Click **"Environment"** tab
3. Find `MONGODB_URI`, click the value field
4. Paste the full connection string with the real password
5. Click **"Save Changes"**

Render will automatically redeploy. After restart, you will see in the logs:
```
💬 Community suggestions service initialized
```
(No "MongoDB connection failed" error — that means it connected.)

---

## 5. Step 3 — Deploy Frontend on Vercel

### 5.1 Import Your Repository

1. Go to **https://vercel.com** and log in
2. Click **"Add New..."** → **"Project"**
3. Under **"Import Git Repository"**, click **"Continue with GitHub"**
4. Authorize Vercel to access your GitHub account
5. Find `niharika19singh/van-setu` in the list
6. Click **"Import"**

### 5.2 Configure the Project

Vercel will try to auto-detect settings. Because `vercel.json` is at the repo root
and specifies `buildCommand` and `outputDirectory`, most settings are already set.

Verify the following on the configuration screen:

| Setting | Value | Notes |
|---|---|---|
| Framework Preset | `Other` or `Vite` | Vercel may auto-detect Vite |
| Root Directory | `.` (repo root) | Leave as root — `vercel.json` handles it |
| Build Command | `cd frontend && npm install && npm run build` | From `vercel.json` |
| Output Directory | `frontend/dist` | From `vercel.json` |

> **Do not change Root Directory to `frontend/`**. The `vercel.json` at repo root
> specifies the build command with `cd frontend`. If you set Root Directory to
> `frontend/`, Vercel will look for `vercel.json` inside `frontend/` and won't
> find it.

### 5.3 Environment Variables on Vercel

No environment variables need to be set on Vercel. The API URL is handled by the
proxy rewrite in `vercel.json` — the frontend uses `/api` (relative) and Vercel
forwards it to Render.

Click **"Deploy"**.

### 5.4 Watch the Build

Vercel builds the React app using Vite. You will see:

```
Running build in Washington, D.C., USA (East) – iad1
Cloning github.com/niharika19singh/van-setu (Branch: main, Commit: 48a69e6)
...
Running "cd frontend && npm install && npm run build"
...
vite v7.x.x building for production...
✓ 1234 modules transformed.
frontend/dist/index.html         0.50 kB
frontend/dist/assets/index-xxx.js  892.34 kB
frontend/dist/assets/index-xxx.css  45.12 kB
✓ built in 8.34s
...
Build Completed in /vercel/output [12s]
Deployment completed
```

### 5.5 Confirm the Frontend is Live

1. Vercel assigns a URL like: `https://van-setu-abc123.vercel.app`
2. Open it — you should see the VanSetu map interface
3. **The map will load but API calls may fail** until Step 4 is complete

---

## 6. Step 4 — Connect Frontend to Backend

This is the **critical linking step**. You need to tell Vercel where to forward
API requests. Currently `vercel.json` has a placeholder URL.

### 6.1 Update vercel.json

Open `/Users/user/Vansetu/vercel.json`:

```json
{
  "buildCommand": "cd frontend && npm install && npm run build",
  "outputDirectory": "frontend/dist",
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://van-setu-backend.onrender.com/api/:path*"
    },
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ]
}
```

Replace `https://van-setu-backend.onrender.com` with **your actual Render URL**
from Step 3.7.

For example, if your Render URL is `https://van-setu-xk7p.onrender.com`:

```json
"destination": "https://van-setu-xk7p.onrender.com/api/:path*"
```

### 6.2 Commit and Push

```bash
cd /Users/user/Vansetu
git add vercel.json
git commit -m "fix: set Render backend URL in vercel.json"
git push origin main
```

### 6.3 Vercel Auto-Redeploys

Vercel watches the `main` branch. Within 30 seconds of your push:
1. Vercel detects the new commit
2. Starts a new build automatically
3. Deploys the new version (the old version stays live until the new one is ready)

You can watch the build in the Vercel dashboard under **"Deployments"**.

### 6.4 How the Proxy Works

After this, every API call from the browser:

```
Browser → GET https://van-setu.vercel.app/api/corridors/priority-ranking
               ↓
          Vercel Edge Network (vercel.json rewrite rule)
               ↓
          GET https://van-setu-xk7p.onrender.com/api/corridors/priority-ranking
               ↓
          FastAPI backend processes request
               ↓
          Response flows back through Vercel to browser
```

The browser always talks to `*.vercel.app`. Render is never called directly from
the browser. This means:
- No CORS headers needed
- Backend URL can change without touching frontend code
- Vercel's global CDN handles latency for static assets

---

## 7. Step 5 — Verify Everything Works

### 7.1 Health Check

Open: `https://your-vercel-app.vercel.app/api/health`

Expected:
```json
{
  "status": "healthy",
  "raster_loaded": true,
  "bounds": {"north": 28.87, "south": 28.4, "east": 77.35, "west": 76.73}
}
```

### 7.2 Stats Check

Open: `https://your-vercel-app.vercel.app/api/stats`

Expected (non-empty values):
```json
{
  "ndvi": {"min": 0.02, "max": 0.64, "mean": 0.20, ...},
  "lst":  {"min": 22.7, "max": 29.0, "mean": 25.7, ...},
  "gdi":  {"min": 0.29, "max": 0.84, "mean": 0.62, ...},
  "shape": [57, 79]
}
```

If you see `"ndvi": {}` — the NDVI TIF file was not loaded. See Troubleshooting.

### 7.3 Priority Ranking Check

Open: `https://your-vercel-app.vercel.app/api/corridors/priority-ranking`

Expected:
```json
{
  "ranked_corridors": [ ... 2000+ items ... ],
  "metadata": { "count": 2346, "formula": "Priority = ..." }
}
```

### 7.4 UI Feature Checklist

Open your Vercel URL and verify:

- [ ] Map loads centered on Delhi
- [ ] **Statistics panel** shows NDVI/LST/GDI values (not "No statistics available")
- [ ] **Priority Ranking** loads a list of corridors (not "Failed to load ranking data")
- [ ] **Data Layers** toggle works and shows colored overlays on the map
- [ ] **Community Input** form submits without error
- [ ] **Health Data** form submits without error
- [ ] **Clicking on the map** shows a popup with NDVI/LST/AQI/Priority values

---

## 8. Optional — AQI API Key (Live Air Quality)

Without an API key, the backend uses **fallback AQI data**: 10 hardcoded Delhi
monitoring station coordinates with representative PM2.5 values (~160 µg/m³).
This is functional but not live.

To get real-time data:

### 8.1 Get a WAQI Token

1. Go to **https://aqicn.org/api/**
2. Enter your email in the "Request API Token" form
3. You will receive a token by email within minutes (free, no credit card)
4. The token looks like: `a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6`

### 8.2 Add to Render

1. Go to your Render service → **"Environment"** tab
2. Find `OPENAQ_API_KEY`, enter your WAQI token
3. Click **"Save Changes"** — Render redeploys automatically

After restart, the log will show:
```
📡 Initializing AQI data...
  ✅ WAQI fetch successful: 14 stations loaded
```

---

## 9. Optional — Telegram Bot

The Telegram bot (`backend/telegram_bot.py`) lets community users submit
environmental observations via Telegram chat. It calls the `/api/community-data`
and `/api/health-data` endpoints.

### 9.1 Create a Bot via BotFather

1. Open Telegram and search for `@BotFather`
2. Send `/newbot`
3. Enter a display name: `VanSetu Community Bot`
4. Enter a username (must end in `bot`): `vansetu_community_bot`
5. BotFather replies with a token:
   ```
   Use this token to access the HTTP API:
   7123456789:AAF_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
6. Copy the token

### 9.2 Add Token to Render

1. Go to your Render service → **"Environment"** tab
2. Find `TELEGRAM_BOT_TOKEN`, enter the token from BotFather
3. Click **"Save Changes"**

### 9.3 Run the Bot

The bot runs as a **separate process** from the API server. On Render free tier,
you can only run one process per service. To run the bot:

**Option A — Run locally:**
```bash
cd backend
source venv/bin/activate
TELEGRAM_BOT_TOKEN=your_token VANSETU_API_URL=https://your-render-url.onrender.com python telegram_bot.py
```

**Option B — Create a separate Render Worker service:**
1. In your Render dashboard, click **"New +"** → **"Background Worker"**
2. Connect the same GitHub repo
3. Root Directory: `backend`
4. Build Command: `pip install -r requirements.txt`
5. Start Command: `python telegram_bot.py`
6. Add environment variable `TELEGRAM_BOT_TOKEN` and `VANSETU_API_URL`

---

## 10. Re-deploying After Code Changes

Both Render and Vercel watch the `main` branch. Every `git push origin main`
triggers automatic redeploys on both platforms.

### Typical workflow:

```bash
# Make your changes locally
# Test locally with start.sh
# Then:
git add .
git commit -m "feat: your change description"
git push origin main
```

After push:
- **Vercel** detects change → rebuilds frontend → deploys in ~1 minute
- **Render** detects change → reinstalls dependencies → restarts server in ~3-5 minutes

### Checking deploy status:

- Vercel: dashboard at `vercel.com` → your project → **Deployments** tab
- Render: dashboard at `render.com` → your service → **Logs** tab

### Rolling back a bad deploy:

- **Vercel**: Go to Deployments tab → find a previous deployment → click the
  `...` menu → "Redeploy"
- **Render**: In service settings → **Deploys** → click a previous deploy →
  "Rollback to this deploy"

---

## 11. Custom Domain Setup

### 11.1 Add Domain to Vercel

1. Go to your Vercel project → **"Settings"** → **"Domains"**
2. Click **"Add Domain"**
3. Enter your domain, e.g. `vansetu.in`
4. Vercel shows you the DNS records to add:
   ```
   Type: A
   Name: @
   Value: 76.76.21.21

   Type: CNAME
   Name: www
   Value: cname.vercel-dns.com
   ```
5. Log in to your domain registrar (GoDaddy, Namecheap, Cloudflare, etc.)
6. Add the DNS records
7. Wait 5–30 minutes for DNS propagation
8. Vercel automatically provisions an SSL certificate (Let's Encrypt)

### 11.2 Add Domain to Render (Optional)

If you want `/api` calls in your custom domain to go directly to Render (without
Vercel proxy), you can also add a custom domain to Render. But with the Vercel
proxy setup, this is not necessary — the proxy handles it.

---

## 12. Environment Variables Reference

### Render (Backend)

| Variable | Required | Default | Description |
|---|---|---|---|
| `PORT` | Yes (auto) | Set by Render | Port the server binds to. Never set this manually. |
| `PYTHON_VERSION` | Yes | `3.11.0` | Python version for Render's build environment. |
| `OPENAQ_API_KEY` | No | `""` (empty) | WAQI API token for live AQI data. Without it, 10 fallback Delhi stations are used. Get at `aqicn.org/api`. |
| `MONGODB_URI` | No | `""` (empty) | MongoDB Atlas connection string. Without it, corridor suggestions are disabled. Format: `mongodb+srv://user:pass@cluster.mongodb.net/vansetu` |
| `MONGODB_DB` | No | `urban_green_corridors` | Database name inside MongoDB. Change only if you want a custom DB name. |
| `TELEGRAM_BOT_TOKEN` | No | `""` (empty) | Telegram bot token from BotFather. Only needed to run the community bot. |
| `DATA_DIR` | No | `backend/data` | Override path to TIF data directory. Only set this if you move the data files. |

### Vercel (Frontend)

No environment variables are required. The API URL is handled by the proxy
rewrite in `vercel.json`.

If you want to **bypass the proxy** and have the frontend call Render directly:
1. Set `VITE_API_URL` in Vercel dashboard = `https://your-service.onrender.com/api`
2. Remove the `/api/:path*` rewrite line from `vercel.json`
3. Add your Vercel domain to CORS in `backend/app/main.py`

---

## 13. Troubleshooting

### Backend Issues

**Problem:** Render build fails with `ERROR: Could not build wheels for rasterio`

Root cause: `rasterio` needs GDAL system libraries. Render's Python runtime
includes the build tools, so this should resolve itself. If not:

Fix: Add a `runtime.txt` file to `backend/`:
```
python-3.11.0
```
and a `build.sh` (Render runs this before pip install if present):
```bash
pip install --upgrade pip setuptools wheel
```

---

**Problem:** Backend starts but `raster_loaded: false`

Root cause: The TIF files were not found at startup.

Check 1 — Verify the files exist in the repo:
```bash
git ls-files backend/data/
# Should show:
# backend/data/delhi_lst_modis_daily_celsius.tif
# backend/data/delhi_ndvi_10m.tif
```

Check 2 — If missing, re-add them:
```bash
git add -f backend/data/delhi_ndvi_10m.tif
git add -f backend/data/delhi_lst_modis_daily_celsius.tif
git commit -m "fix: re-add TIF data files"
git push origin main
```

Check 3 — Verify `.gitignore` has the negation rules:
```
*.tif
*.tiff
!backend/data/delhi_ndvi_10m.tif
!backend/data/delhi_lst_modis_daily_celsius.tif
```

---

**Problem:** `"raster_loaded": true` but stats are empty `{}`

Root cause: The TIF files exist but are corrupt or zero-byte.

Fix: Check file sizes in Render's shell (Render provides a shell tab):
```bash
ls -la data/
# Should show ~20KB for each file, NOT 0 bytes
```

---

**Problem:** Priority ranking returns `{"ranked_corridors": [], "metadata": {"count": 0}}`

Root cause: Road data fetch from OpenStreetMap failed at startup, OR raster
data not loaded so all segment scores are 0 (none above 85th percentile).

Fix 1 — Check raster is loaded (see above)

Fix 2 — Trigger road data refresh via API:
```
POST https://your-render-url.onrender.com/api/roads/refresh
```

Fix 3 — Check Render logs for OSM connectivity errors:
```
ERROR: Could not connect to OSM Overpass API
```
If so, it's a network issue on Render's end — usually resolves on retry.

---

**Problem:** First request to `/api/roads` or `/api/corridors` is very slow (30–60s)

This is normal. On the first request, the backend fetches road network data
from OpenStreetMap's Overpass API for all of Delhi (~15,000 road segments).
This data is then cached in memory for the duration of the server process.

Subsequent requests are fast (in-memory cache).

On Render free tier, the service **spins down after 15 minutes of inactivity**.
The next request after spin-down pays the full cold-start cost again.

Solution (Render free tier): Use a free uptime monitor like UptimeRobot to ping
`/health` every 10 minutes, keeping the service warm:
1. Go to **uptimerobot.com** → sign up free
2. Add new monitor → HTTP(s) monitor
3. URL: `https://your-render-url.onrender.com/health`
4. Monitoring interval: 10 minutes

---

### Frontend Issues

**Problem:** Vercel build fails with `Cannot find module` or `npm install` error

Fix: Check `frontend/package.json` exists and has the correct dependencies.
Then re-trigger the build from Vercel dashboard → Deployments → "Redeploy".

---

**Problem:** Map loads but all API calls fail (network errors in browser console)

Root cause: `vercel.json` still has the placeholder URL or wrong Render URL.

Fix: Open `vercel.json`, update the destination to your actual Render URL:
```json
"destination": "https://your-actual-service.onrender.com/api/:path*"
```
Then push and wait for Vercel to redeploy.

---

**Problem:** API calls return `502 Bad Gateway` from Vercel

Root cause: Render backend is down, crashed, or still starting up.

Fix 1 — Check Render service status: go to Render dashboard, is the service
showing "Live" (green) or something else?

Fix 2 — Check Render logs for startup errors

Fix 3 — Hit the Render URL directly to confirm it's up:
```
https://your-render-url.onrender.com/health
```
If this returns healthy but Vercel still gives 502, it's a Vercel proxy config
issue — re-check the URL in `vercel.json`.

---

**Problem:** Old API URL cached in browser — changes to `vercel.json` not taking effect

Fix: Hard refresh the browser: `Ctrl+Shift+R` (Windows/Linux) or `Cmd+Shift+R` (Mac).
Or open Chrome DevTools → Network tab → check "Disable cache".

---

**Problem:** Community Input / Health Data form submits but data is not saved

These forms POST to `/api/community-data` and `/api/health-data`. Data is stored
as JSON files in `backend/data/feedback/` on the Render server.

Important: Render's free tier has an **ephemeral filesystem**. Files written
during a deployment are lost when the service restarts or redeploys. This means
community submissions are temporary.

Solution for persistent data: Set `MONGODB_URI` to save submissions to MongoDB
Atlas, OR upgrade to Render's paid tier which includes persistent disk.

---

**Problem:** Corridor suggestions show 0 / upvotes don't work

Root cause: MongoDB is not connected. Check Render logs for:
```
⚠️ MongoDB connection failed: ...
```

Fix: Follow Section 4 to set up MongoDB Atlas and add `MONGODB_URI` to Render.

---

### Local Development

To run locally after cloning:

```bash
git clone https://github.com/niharika19singh/van-setu.git
cd van-setu
./start.sh
```

`start.sh` starts:
- Backend at `http://localhost:8000`
- Frontend at `http://localhost:5173`

API docs available at `http://localhost:8000/docs`

For backend only:
```bash
cd backend
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

For frontend only:
```bash
cd frontend
npm install
npm run dev
```
