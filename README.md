# Kajaria Visualizer Pro

An in-store tile visualization tool. A customer uploads one photo of their room and sees Kajaria tiles rendered onto their own floor, with the room's real lighting and shadows preserved. Built as a full-stack prototype (Flask + React) with AI room analysis, photo-realistic rendering, a 3D room planner and built-in lead capture.


---

## What it does

The app has four modes, all sharing one tile catalogue and one lead-capture flow.

| Mode | What it does |
|---|---|
| **Auto Visualize** (agent) | Upload a room photo and do nothing else. Claude vision reads the room (type, lighting, wall tone, style, floor texture, mood), a scoring function picks the best-suited tile from the catalogue, and fal.ai renders it onto the floor. Zero clicks after upload. |
| **Photo Studio** | Manual control. The floor is detected automatically around furniture, then the user flips through tiles live, changes layout (straight or diagonal), grout colour and wall + floor together. Erase/Restore brushes and a manual "Trace floor" outline are available to refine the mask. Export a photo-real render through fal.ai. |
| **3D Planner** | No photo needed. Enter the room size in square feet, pick a room type and a style (Modern, Minimalist, Traditional, Trending India) and a furnished 3D room is generated in Three.js with the selected tile underfoot. |
| **Projection** | Preview of an in-store bay where a floor projector and a wall LED act as one continuous canvas. |

Across modes:

- **Tile catalogue** with filters by group (Marble, Wood, Stone & Studio, Heritage, Bath), finish, size and price, plus a shortlist.
- **Custom tile upload.** Any image becomes a tile, cropped to a square texture and added to the shelf.
- **Instant estimate** in rupees from area and tile price.
- **Lead capture + CRM drawer.** Save the customer, their phone, buying intent, approximate area, the shortlisted tiles and a snapshot of the render.

---

## How Auto Visualize works

```
Room photo
   │
   ▼
Claude vision  ──►  { roomType, lighting, wallTone, style, floorTexture,
   │                  furnitureDensity, spaceSize, mood, warmth, luxuryLevel, ... }
   │   (falls back to an offline heuristic analysis if the vision API is unreachable)
   ▼
recommendTiles()  ──►  ranked tiles with human-readable reasons
   │
   ▼
Prompt builder  ──►  tile size in cm, finish, layout (plank / brick-bond / stacked grid),
   │                  light response (matte vs polished), "preserve everything else"
   ▼
fal.ai image-edit model (room photo + tile texture as inputs)
   │
   ▼
Rendered room  ──►  Save as lead
```

The prompt tells the image model to replace only the floor, infer real-world scale from architectural cues, keep furniture, walls and windows unchanged, and cast the room's natural shadows onto the new floor.

---

## Architecture

```
┌──────────────────────────┐        ┌─────────────────────────────┐
│  Frontend (Vite + React) │        │  Backend (Flask)            │
│  Home · Auto · Photo     │ /api/* │  POST /api/claude    ──► Anthropic Messages API
│  3D Planner · Projection ├───────►│  POST /api/fal/render ──► fal.ai queue (submit, poll, fetch)
│  Tile rail · Leads / CRM │        │  GET  /*  serves the built frontend
└──────────────────────────┘        └─────────────────────────────┘
```

- The frontend builds into `backend/public`, so one Flask process serves the whole app in production.
- In development, Vite proxies `/api` to Flask on port 8765.
- Both AI keys can stay **server-side**: the backend proxies Claude and fal.ai so the browser never sees them.

### Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Three.js (3D planner), Canvas 2D (warping, textures, masks) |
| Backend | Python, Flask, Flask-CORS, Requests |
| Vision / analysis | Anthropic Claude (vision) |
| Rendering | fal.ai image-edit model (nano-banana-pro) |

### Project structure

```
backend/
  app.py                  Flask server: Claude proxy, fal.ai proxy, static hosting
  requirements.txt
  public/                 Built frontend output
frontend/
  src/
    App.jsx               App shell, mode routing, shortlist, leads state
    components/
      Home.jsx            Landing screen
      AutoVisualize.jsx   Agentic zero-click flow
      PhotoStudio.jsx     Floor detection, brushes, layout/grout controls, render export
      Room3D.jsx          Three.js furnished room generator
      Projection.jsx      Floor + wall projection preview
      TileRail.jsx        Catalogue shelf, filters, custom upload
      LeadModal.jsx       Lead capture form
      LeadsDrawer.jsx     CRM drawer
    utils/
      vision.js           Floor detection, room analysis, tile recommendation, fal.ai client
      texture.js          Procedural tile textures (marble, wood, stone, terrazzo, patterns)
      graphics.js         Perspective warping helpers
      helpers.js
    data/tiles.js         Tile catalogue and lookup
  vite.config.js          /api proxy and build output directory
```

---

## Run locally

**Prerequisites:** Python 3.11+, Node.js 18+, an Anthropic API key and a fal.ai key.

### 1. Backend

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

export ANTHROPIC_API_KEY=your_anthropic_key
export FAL_KEY=your_fal_key        # or put both in backend/.env
python3 app.py                     # http://localhost:8765
```

### 2. Frontend (development)

```bash
cd frontend
npm install
npm run dev                        # Vite dev server, proxies /api to :8765
```

### 3. Production-style build

```bash
cd frontend && npm run build       # outputs to backend/public
cd ../backend && python3 app.py    # Flask now serves the full app on :8765
```

### Environment variables

| Variable | Used by | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | backend | Room analysis and floor detection through `/api/claude` |
| `FAL_KEY` | backend | Photo-real rendering through `/api/fal/render` |
| `PORT` | backend | Port to listen on (default `8765`) |

Never commit keys. Keep them in the environment or in an ignored `.env`.

---

## API

| Endpoint | Method | Description |
|---|---|---|
| `/api/claude` | POST | Forwards the request body to the Anthropic Messages API and returns the response as-is. |
| `/api/fal/render` | POST | Body: `{ prompt, roomUri, tileUri }`. Submits to fal.ai, polls the queue every 2 s (up to 3 min), and returns `{ url }` of the rendered image. |
| `/` and `/<path>` | GET | Serves the built frontend. |

---


