# THREATZONE frontend

React + Vite + React-Leaflet frontend for the existing DER-02 backend
(`main.py` / `der02_hazard_engine.py`). This directory contains **no
physics** -- every hazard number and every polygon on the map comes from
the real backend response.

## Setup

```bash
cd frontend
npm install
npm run dev
```

Runs at http://localhost:5173 by default. Vite's dev server has no
special backend proxy configured; the app talks directly to
`http://localhost:8000` (see `src/config.js`) using CORS, which
`main.py` already allows for `localhost:5173` / `127.0.0.1:5173`.

To point at a different backend host, set `VITE_API_BASE_URL`:

```bash
VITE_API_BASE_URL=http://localhost:8000 npm run dev
```

## Running the whole stack

```bash
# terminal 1 -- backend
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# terminal 2 -- frontend
cd frontend
npm install
npm run dev
```

Then open http://localhost:5173.

## Build

```bash
npm run build
npm run preview
```

## What talks to what

- `src/api.js` -- the only file that calls the backend. Wraps
  `GET /api/health`, `GET /api/fuel-types`, `POST /api/calculate`,
  `POST /api/compare` exactly as defined in `main.py` / `schemas.py`.
- `src/components/FacilityForm.jsx` -- builds a payload shaped exactly
  like `schemas.py::FacilityConfigIn`. No extra fields (the backend's
  Pydantic models use `extra="forbid"` and would reject them).
- `src/components/HazardMap.jsx` -- renders `band.geometry.coordinates`
  (an array of `[lat, lon]` pairs returned by
  `der02_hazard_engine.generate_hazard_geometry`) directly into
  react-leaflet `<Polygon positions={...}>`. No client-side geometry
  math, no fixed-radius circles.
