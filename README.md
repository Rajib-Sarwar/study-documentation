# GPS Snap-to-Road: Study Documentation
## Journey from Raw GPS Snapping to HMM Map-Matching with Bearing Correction
---
## 1. The Problem
A fleet of limousine drivers sends GPS coordinates from mobile devices to the `LsTrafficApi` backend every ~10 seconds. The backend must "snap" these raw GPS points to the actual road network so that trip paths display cleanly on maps, distances are accurate, and driver behavior analysis is reliable.
### Why It's Hard
| Challenge | Description |
|---|---|
| **GPS noise** | Smartphone GPS has 10-20m typical error in NYC urban canyons |
| **Divided roads** | Highways have parallel carriageways going opposite directions — GPS can't distinguish them |
| **Underpasses/overpasses** | Stacked roads at the same lat/lng |
| **10-second batches** | Only 3-4 GPS points per upload — very little context per batch |
| **Real-time requirement** | Snap must complete in <100ms per batch for live tracking |
### The Core Tension
```
Full-trip context (hundreds of points)  →  HMM can find the correct road globally
10-sec batch context (3-4 points)       →  HMM can get stuck on the wrong road
```
The entire journey was about bridging this gap: how to give a 10-second batch enough "memory" to make the same quality decision as a full-trip analysis.
---
## Phase 1: GraphHopper Route API + Polyline Matching
### Approach
1. Collect raw GPS points in a 10-second batch
2. Thin/select waypoints (avoid sending too many points to GraphHopper)
3. Send waypoints to GraphHopper's **Route API** (`GHRequest`)
4. GraphHopper returns a polyline (the road path through those waypoints)
5. Project each raw GPS point onto the nearest segment of that polyline
6. If projection distance is too far, fall back to identity (keep raw)
### Key Concepts
- **Corridor Snapping**: Filter GraphHopper polyline segments too far from the raw trace
- **Waypoint Thinning**: `ROUTE_WP_SPACING_M = 80.0` — space waypoints so GraphHopper isn't forced through dense clumps
- **Identity Fallback**: `IDENTITY_MAX_JUMP_FROM_LAST_M = 1500m` — keep raw if it doesn't jump too far
### Problems Encountered
| Problem | Symptom | Root Cause |
|---|---|---|
| **PointOutOfBoundsException** | HTTP 500 from GraphHopper | Raw point outside map bounds |
| **"V" shapes** | Zigzag at trip start (Battery Park) | Fallback to full polyline instead of identity |
| **"Flying" gaps** | Teal line showing huge jumps | Corridor filtering removed too many vertices |
| **Data loss vs other backend** | Fewer points than RPC backend | Different data feeds (app vs telematics) |
### Fixes Applied
- Pre-filter out-of-bounds points before sending to GraphHopper
- Removed `WIDEN-FALLBACK` — fall back to `IDENTITY` or `HOLD_LAST` instead of full polyline
- Ensure all polyline vertices between first and last snapped points are included
- Null-safe `filterClusteredLocationData`
---
## Phase 2: Guards and Heuristics
As testing revealed edge cases, threshold-based guards were added:
### Stationary Clamping
```
STATIONARY_RAW_M = 8.0    → if raw moved < 8m, vehicle is stopped, don't gap-cross
MAX_ADVANCE_SLACK_M = 15  → snap cannot lead raw movement by more than 15m
HOLD_MAX_DRIFT_M = 10.0   → if holding last position and raw drifted > 10m, fall back to identity
```
**Problem solved**: Broadway stop case (seq 848-859) where live snap teleported the vehicle forward while stopped at a light.
### Slow-Speed Drift Rejection
```
SLOW_MOVE_M = 12.0        → raw movement < 12m means crawling
SLOW_SNAP_MAX_M = 10.0    → reject snaps > 10m from raw while crawling
```
**Problem solved**: Flatbush/Queens Blvd (seq 438-446) where snap error grew monotonically during slow movement.
**Problem created**: This guard caused "V" shapes in urban canyons because a consistent GPS bias exceeded the fixed 10m threshold, forcing points back to raw coordinates (zigzag).
### DBSCAN Clustering (Later Removed)
Initially used DBSCAN to classify outliers (`cluster == -1` → drop). **Removed** because:
1. A haversine bug (`Math.cos(lng)` instead of `Math.cos(lat)`) caused incorrect East-West distance
2. DBSCAN rejected valid single-point uploads
3. Replaced with explicit bounds + jump rejection:
   - Points outside GraphHopper map bounds → `SKIPPED / NO_MATCH`
   - Points implying jumps > `IDENTITY_MAX_JUMP_FROM_LAST_M` → `SKIPPED`
   - Raw coordinates always preserved in the database regardless
### The Realization: Threshold Tuning Is Not the Answer
> *"Can you do a proper research and tell me which way will be enough for this to improve, as if we do this it might create issues and result in a bad performance than a raw GPS data"*
The "V" shapes and zigzags were not caused by wrong thresholds — they were caused by the fundamental approach: **geometric projection of individual points cannot disambiguate parallel carriageways**. No amount of threshold tuning would fix this.
This led to the decision: **"let's do it right"** — implement a proper map-matching algorithm.
---
## Phase 3: Hazelcast Context Trail
### The Insight
GraphHopper's Route API only sees the current batch's waypoints. It has no memory of where the vehicle was before. On a divided road, each batch independently decides which carriageway to use — and can flip between them.
### The Solution: `DriverSnapContextService`
An in-memory Hazelcast cache storing the **last N good snaps** per device, providing cross-batch road memory.
### Key Parameters (Final Tuning)
| Parameter | Value | Purpose |
|---|---|---|
| `MAX_POINTS` | 8 | Max context points stored per device |
| `MAX_TRAIL_M` | 150.0 | Rolling window: keep last 150m of road behind newest snap |
| `GAP_RESET_MS` | 30 min | Treat as new trip after 30 min of silence |
| `MAX_JUMP_M` | 16,000m | Teleport detection (covers tunnels/bridges) |
| `MIN_SPACING_M` | 40.0 | Space older context points so GraphHopper isn't forced through dense clumps |
| `MIN_TAIL_SPACING_M` | 5.0 | Always keep the newest snap (pins the batch seam) |
| `TTL_MINUTES` | 30 | Last-resort Hazelcast expiry |
| Condition | Action |
|---|---|
| Raw distance < 5m | Bearing unreliable -> skip correction, keep HMM |
| No correct-direction edge nearby | Fall back to identity (keep raw) — never force bad snap |
| Bearings match (<=90 deg diff) | Normal HMM projection — no correction needed |
| Raw < 2 points | Can't compute bearing -> skip |
### Constants
```java
BEARING_WRONG_SIDE_DEG = 90.0   // Trigger threshold for correction
BEARING_MAX_DIFF_DEG   = 90.0   // Max diff for edge to be "same direction"
BEARING_MIN_RAW_DIST_M = 5.0    // Min raw distance for reliable bearing
BEARING_SNAP_MAX_M     = 50.0   // Max snap distance for correction
```
### Logging
```
[SNAP-DBG] dev=... BEARING-CORRECT rawBearing=275, hmmBearing=95, diff=180 > 90 -> re-snap with bearing filter
[SNAP-DBG] dev=... BEARING-OK rawBearing=270, hmmBearing=265, diff=5 <= 90
[SNAP-DBG] dev=... BEARING-SKIP rawDist=3.2m < 5.0m (bearing unreliable)
[SNAP-DBG] pt dev=... raw=40.767023,-73.897149, projDist=6.2m, decision=BEARING-SNAP
[SNAP-DBG] pt dev=... raw=40.767023,-73.897149, projDist=0.0m, decision=BEARING-IDENTITY
```
---
## Dashboard Tools
### 1. Trip Dashboard (`trip-dashboard.html`)
Displays individual trip GPS data with three layers:
- **Green**: Raw GPS
- **Orange**: DB Snap (stored in database — 10-sec batch result)
- **Purple/Teal**: Live Snap (full-trip resnap with current algorithm)
Features:
- Bidirectional highlighting between map coordinates and report rows
- Polyline of waypoints used by GraphHopper (context + raw)
- Layer toggles (show/hide raw, dbSnap, liveSnap)
- "Report this point" feature to flag coordinates for analysis
- "Select range" tool to pick start/end points and generate a detailed JSON report
- "Clear range" button to reset selection
- "Share map" button generating a URL with all filters/toggles/view state
- Draggable popups
### 2. Discrepancy Dashboard (`gps-discrepancy-dashboard.html`)
Day-level driver trip summaries for data quality analysis:
- Trip list with time column (first/last upload times in local HH:mm)
- Sustained divergence detection (`SUSTAINED_DIVERGE`)
- Per-trip metrics: raw path length, db path length, live path length, max errors
### 3. Range Report (`/tripgps/resnap`)
JSON report for a selected coordinate range including:
- Raw, DB snap, and live snap coordinates per point
- `dbToLiveM` and `rawToLiveM` distances
- Path lengths and detected issues
---
## Key Lessons Learned
### 1. Threshold Tuning Has Limits
Adding more guards (stationary clamp, slow-speed drift, overshoot rejection) fixed specific cases but created new problems ("V" shapes). **Geometric projection fundamentally cannot disambiguate parallel carriageways** — no threshold fixes that.
### 2. Context Memory Is Essential but Can Trap
The 150m Hazelcast trail provides cross-batch road memory, but if a wrong snap enters the trail, it anchors subsequent batches to the wrong road. **Context is only as good as the snaps that populate it.**
### 3. Full-Trip vs 10-sec Batch: The Fundamental Gap
The full-trip HMM (live snap) consistently outperforms the 10-sec batch HMM (DB snap) because it has global visibility. The 10-sec batch must make local decisions with limited context. **Bearing correction bridges this gap** by adding direction information the HMM lacks.
### 4. Never Lose Raw Data
Every snap decision preserves the raw GPS coordinates in the database. Snaps can be wrong, but raw is never lost. This allows reprocessing and verification.
### 5. Production Logging Is Crucial
`[SNAP-DBG]` logs with `ctxCoords`, `batchRaw`, per-point decisions, and bearing checks allow diagnosing production behavior without a dashboard resnap. **Hardcoding `SNAP_DEBUG=true`** ensures logs are always available.
### 6. The Divided-Road Feedback Loop
```
Batch N: HMM chooses wrong carriageway -> wrong snap stored in context
    v
Batch N+1: context on wrong road -> HMM follows context -> wrong snap again
    v
Batch N+2: context still wrong -> HMM follows -> repeats
    v
... persists until road geometry forces a switch back
```
This is why a 6m GPS drift can cause a 16m error lasting 8 batches. The bearing correction breaks this loop by rejecting the initial wrong choice.
---
## Current Architecture
```
                    Raw GPS (10-sec batch)
                           |
                           v
              +---------------------------+
              | DeviceGpsBatch            |
              |  - partition in/out bounds |
              |  - build observations     |
              |    (context + raw)        |
              +---------------------------+
                           |
                           v
              +---------------------------+
              | GraphHopperService        |
              |  .matchTrack() -- HMM     |
              |  (Newson & Krumm Viterbi) |
              +---------------------------+
                           |
                           v
              +---------------------------+
              | Post-HMM Bearing Check    |
              |  rawBearing vs hmmBearing |
              |  if diff > 90 deg:        |
              |    re-snap with bearing   |
              |    filter (LocationIndex)  |
              +---------------------------+
                           |
                           v
              +---------------------------+
              | Project raw onto road     |
              |  (HMM polyline or        |
              |   bearing-corrected edge)|
              +---------------------------+
                           |
                           v
              +---------------------------+
              | Store snaps in Hazelcast  |
              |  trail (150m, 8 pts max)  |
              |  + persist to database    |
              +---------------------------+
```
### Key Files
| File | Role |
|---|---|
| `DeviceGpsBatch.java` | Core snap logic: HMM + bearing correction + projection |
| `GraphHopperService.java` | GraphHopper wrapper: `matchTrack()`, `snapToRoadWithBearing()` |
| `GraphHopperUtils.java` | Utilities: `computeBearing()`, `bearingDiff()`, `snapToRoad()` |
| `DriverSnapContextService.java` | Hazelcast context trail management |
| `GpsProcessingService.java` | Orchestrates snapping + stores context |
| `trip-dashboard.html` | Trip visualization frontend |
| `gps-discrepancy-dashboard.html` | Day-level discrepancy dashboard |
---
## Glossary
| Term | Definition |
|---|---|
| **Snap** | Mapping a raw GPS point to a road network position |
| **HMM** | Hidden Markov Model — probabilistic sequence model |
| **Viterbi** | Algorithm to find the most likely HMM state sequence |
| **Emission probability** | P(GPS point \| road position) — Gaussian on distance |
| **Transition probability** | P(next road position \| current) — favors continuity |
| **measurementErrorSigma** | GPS error standard deviation (50m) |
| **Context trail** | Hazelcast cache of previous good snaps per device |
| **Bearing** | Direction of travel in degrees (0=N, 90=E, 180=S, 270=W) |
| **Divided road** | Highway with separate carriageways for each direction |
| **Carriageway** | One side of a divided road (one direction of travel) |
| **OOB** | Out of bounds — GPS point outside GraphHopper map |
| **Identity fallback** | Keep raw GPS point when snap fails |
| **DB snap** | Snap stored in database (10-sec batch result) |
| **Live snap** | Full-trip resnap with current algorithm |
| **projDist** | Projection distance — distance from raw to snapped point |
| **SNAP-DBG** | Debug log tag for snap pipeline diagnostics |
---
## Conclusion
The journey from raw GPS snapping to HMM map-matching with bearing correction represents a progression from **geometric heuristics** to **probabilistic sequence modeling** to **direction-aware correction**.
Each phase solved the limitations of the previous one:
- Phase 1 (Route API) couldn't disambiguate parallel roads
- Phase 2 (Guards) created "V" shapes from fixed thresholds
- Phase 3 (Context) provided memory but could trap on wrong roads
- Phase 4 (HMM) solved sequence scoring but lacked direction sense
- Phase 5 (Bearing) added direction to break the divided-road feedback loop
The result is a snap pipeline that handles urban canyons, divided roads, tunnels, and 10-second batches while preserving raw data and providing full traceability through production logs.
