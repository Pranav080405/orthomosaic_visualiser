# Real-Time Orthomosaic & Multispectral Telemetry Visualizer

A lightweight, real-time aerial mapping pipeline engineered to stream georeferenced orthomosaic tiles from edge systems (e.g., NVIDIA Jetson Orin aboard UAVs) to an asynchronous backend and render them live on an interactive browser map via Server-Sent Events (SSE).

Supports dynamic switching between standard **RGB imagery**, **NDVI** (Normalized Difference Vegetation Index), and **NDRE** (Normalized Difference Red Edge) layers as a flight progresses.


<img width="1179" height="624" alt="Screenshot 2026-10-20 at 1 43 54 PM" src="https://github.com/user-attachments/assets/b5f9ad8e-18cb-4672-8e8d-4db0204b4485" />

<img width="1179" height="624" alt="Screenshot 2026-10-20 at 1 43 23 PM" src="https://github.com/user-attachments/assets/20702a4d-7e84-42d2-b2a6-c166ee589f40" />

---
## Demo

screen recording on local host: https://drive.google.com/file/d/1yEv0kJHcgPNapgp3P2iQ1Rcs4EX0_afe/view?usp=sharing

---

## Architecture Overview

```
 ┌─────────────────────────────────────────────────────────┐
 │                      Edge Layer                         │
 │                                                         │
 │   orin_sender.py (UAV / Jetson Orin)                    │
 │   - Computes NDVI / NDRE indices                        │
 │   - Bounded drop-queue (drops old frames on slow links) │
 │   - WebP compression with alpha transparency masks      │
 │                          OR                             │
 │   mosaic_replay.py (GeoTIFF Replayer / Simulation)      │
 │   - Slices GeoTIFF along snake/lawnmower flight paths   │
 │   - Transforms CRS to EPSG:4326 lat/long bounds         │
 └────────────────────────────┬────────────────────────────┘
                              │
                              │ HTTP POST (Multipart: RGB, NDVI, NDRE, bounds)
                              ▼
 ┌─────────────────────────────────────────────────────────┐
 │                    Server Layer                         │
 │                                                         │
 │   server.py (FastAPI + Uvicorn)                         │
 │   - Endpoint /api/tile: Saves tiles to disk & log       │
 │   - Endpoint /api/stream: Broadcasts tiles via SSE      │
 │   - Serves WebP static tiles & single-page client       │
 └────────────────────────────┬────────────────────────────┘
                              │
                              │ Server-Sent Events (SSE)
                              ▼
 ┌─────────────────────────────────────────────────────────┐
 │                   Frontend Client                       │
 │                                                         │
 │   index.html (Leaflet.js)                               │
 │   - Dynamic imageOverlay tile stitching                 │
 │   - Real-time bounding box fit and follow              │
 │   - Instant layer switching: RGB / NDVI / NDRE          │
 └─────────────────────────────────────────────────────────┘
```

---

## Vegetation Indices

Index values are computed at the edge before compression:

### 1. NDVI (Normalized Difference Vegetation Index)
$$\text{NDVI} = \frac{\text{NIR} - \text{Red}}{\max(\text{NIR} + \text{Red}, 10^{-6})}$$
*Scaled to $[0.0, 1.0]$ for display.*

### 2. NDRE (Normalized Difference Red Edge)
$$\text{NDRE} = \frac{\text{NIR} - \text{RedEdge}}{\max(\text{NIR} + \text{RedEdge}, 10^{-6})}$$
*Scaled to $[0.0, 0.6]$ for display to emphasize dense canopy variation.*

---

## Prerequisites & Installation

### 1. Clone the Repository
```bash
git clone https://github.com/Pranav080405/orthomosaic_visualiser.git
cd orthomosaic_visualiser
```

### 2. Environment Setup
Create and activate a virtual environment:
```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install Dependencies
```bash
pip install fastapi uvicorn python-multipart requests numpy opencv-python rasterio
```

---

## Usage

### 1. Start the Map Server
Launch the FastAPI ingestion and streaming server:

```bash
API_KEY=secret uvicorn server:app --host 0.0.0.0 --port 8000
```
Open your browser and navigate to:
```
http://localhost:8000
```

*(Optional production setup)* Use **Caddy** for automatic SSL/HTTPS termination:
```bash
caddy reverse-proxy --from map.yourdomain.com --to :8000
```

---

### 2. Feed Data

#### Option A: Run Synthetic Lawnmower Simulation (No Drone/GeoTIFF Needed)
Tests the end-to-end streaming and rendering pipeline with generated agricultural patterns:

```bash
python orin_sender.py http://localhost:8000 secret
```

#### Option B: Replay an Existing GeoTIFF Mosaic
Replays an existing georeferenced raster file tile-by-tile to emulate an active survey flight:

* **RGB GeoTIFF (Uses stand-in indices):**
  ```bash
  python mosaic_replay.py /path/to/mosaic.tif http://localhost:8000 secret --tile 1024 --delay 1.0
  ```

* **Multispectral GeoTIFF (True NDVI/NDRE calculation):**
  Provide 1-based band indexes matching your sensor's channel configuration:
  ```bash
  python mosaic_replay.py /path/to/multispectral.tif http://localhost:8000 secret \
      --rgb 1 2 3 \
      --red 4 \
      --re 5 \
      --nir 6 \
      --tile 1024 \
      --delay 0.8
  ```

#### Option C: Production On-Drone Deployment (Jetson Orin)
Import `Sender` into your on-board flight computer processing pipeline:

```python
from orin_sender import Sender

sender = Sender(url="http://your-server-ip:8000", key="secret")

# In your frame loop:
# rgb: HxWx3 uint8 BGR
# r, re, nir: HxW float32 reflectance arrays
# bounds: [[south, west], [north, east]] in lat/lon
# valid: HxW uint8 mask (1 = data, 0 = nodata)
sender.send(rgb, r, re, nir, bounds, valid=valid)
```

---

## API Reference

| Method | Endpoint | Description | Headers |
|---|---|---|---|
| `POST` | `/api/tile` | Ingests a tile package (`meta`, `rgb`, `ndvi`, `ndre`) | `X-Api-Key: <key>` |
| `GET` | `/api/tiles` | Returns all recorded tile metadata in JSON format | None |
| `GET` | `/api/stream` | Server-Sent Events (SSE) stream for live incoming tiles | None |
| `DELETE`| `/api/tiles` | Purges all cached tiles and resets the current flight session | `X-Api-Key: <key>` |

#### Resetting Flight Session via cURL:
```bash
curl -X DELETE -H "X-Api-Key: secret" http://localhost:8000/api/tiles
```

---

## Repository Structure

```
├── index.html          # Lightweight Leaflet-based map interface
├── mosaic_replay.py    # GeoTIFF simulator with CRS reprojection and tiling
├── orin_sender.py      # Edge streaming client with WebP compression & queue management
├── server.py           # FastAPI backend handling ingestion, SSE, and static delivery
├── requirements.txt    # Project Python dependencies
└── .gitignore          # Prevents committing data directories, TIFFs, and temp files
```

---

