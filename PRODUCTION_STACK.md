# RouteDee — Production Development Plan

> This document outlines the real tech stack required to productionize the RouteDee demo into a deployable retail AR navigation app.

---

## What Needs to Be Real

| Demo Feature | What It Fakes | What Real Needs |
|---|---|---|
| Indoor navigation | Step-through button | BLE Beacon positioning |
| Route path | Pre-drawn SVG | Live position on real store map |
| Monster AR game | 2D overlay animation | ARKit / ARCore 3D world anchors |
| Shelf scanner | Fake confidence scores + random items | Real camera + trained CV model |
| Leaderboard | Hardcoded mock data | Backend DB + auth |

---

## 1. Mobile App

**Framework: React Native (with Expo)**

| Choice | Why |
|---|---|
| React Native | Shares JS/TS mental model with current demo code. Single codebase for iOS + Android. |
| Expo (managed workflow) | Faster bootstrapping. OTA updates. Camera, BLE, AR plugins ready-made. |
| Expo bare workflow | When Expo managed hits limits (custom native modules for BLE / ARCore) |
| TypeScript | Required for any real team |

**Key Native Modules Needed**
- `expo-camera` — Camera access for shelf scanner
- `react-native-beacons-manager` or `@config-plugins/react-native-ble-plx` — BLE beacon scanning
- `viro-community/react-viro` or `expo-three` — AR layer
- `react-native-maps` — Could be used for indoor map layers

---

## 2. Indoor Positioning (Navigation)

This is the hardest and most important piece. Replaces "step-through" with real user location.

### Technology: BLE Beacons (iBeacon / Eddystone)

**Why BLE Beacons (and not alternatives)**

| Tech | Accuracy | Cost | Verdict |
|---|---|---|---|
| BLE Beacon (iBeacon) | 1–3 m | Low ($10–30/beacon) | ✅ Best for retail |
| WiFi RTT (802.11mc) | ~1 m | Medium (needs supported APs) | Android-only, infra-heavy |
| UWB (Ultra-Wideband) | <0.5 m | High ($50–100+/node) | Overkill for now |
| GPS | 5–15 m | Free | ❌ Doesn't work indoors |
| CV-based SLAM | 0.5–1 m | Dev-heavy | Future option |

**Beacon Placement Strategy**
- Place 3–5 beacons per major zone (triangulation requires ≥3 visible beacons)
- Use RSSI (signal strength) trilateration to estimate (x, y) position
- Fingerprinting (pre-record RSSI map at known positions) gives better accuracy than pure trilateration
- Hardware recommendation: **Estimote** or **Kontakt.io** beacons (reliable SDK + dashboard)

**Positioning Pipeline**
```
Beacon RSSI readings
    → Kalman filter (smooth noisy signal)
    → Trilateration / Fingerprint matching
    → (x, y) coordinate on store grid
    → Map to nearest zone / waypoint
    → Update UI + AR overlay
```

**Beacon Management Backend**
- Store: beacon UUID → physical (x, y) → zone ID mapping
- Admin dashboard to register/move beacons when store layout changes
- Calibration tool: walk through store, record RSSI fingerprint map

---

## 3. AR Game Layer (Pokémon Go-style)

**Framework: AR Foundation (Unity) or ViroReact**

### Option A — Unity + AR Foundation (Recommended for full AR game)

| | |
|---|---|
| Platform | iOS (ARKit) + Android (ARCore) |
| Language | C# in Unity, communicate to React Native via bridge |
| Capabilities | Full 3D physics, particle effects, plane detection, world anchors |
| Monster placement | Place 3D monster at real-world position detected by plane detection |
| Throwing mechanic | Swipe gesture → physics projectile → collision detection |

**AR Game Flow (Real)**
```
1. User enters zone with beacon trigger
2. App detects flat surface (floor/shelf) via ARKit/ARCore plane detection
3. Spawn 3D monster model anchored to world coordinate
4. User swipes to throw — physics ball flies in 3D
5. Collision detection → hit/miss → coupon spawned with particle effect
6. Coupon flies into wallet (UI animation)
```

### Option B — ViroReact (Faster to ship, less capability)

- React Native native module — no separate Unity project
- 3D models (GLTF/GLB), basic physics, image tracking
- Good enough for V1 monsters — faster dev cycle
- Limitation: less control over advanced AR effects vs Unity

**3D Assets Needed**
- Monster models: low-poly GLTF (keep under 5k polys for mobile)
- Throw ball model
- Particle effects: sparkle, catch burst, coupon coin

---

## 4. Computer Vision — Shelf Scanner

**Goal:** Point camera at shelf → detect which products from shopping list are present, with bounding boxes and confidence.

### Architecture Decision

```
Option A: On-device inference (Privacy-first, no latency)
Option B: Cloud API (Easier to start, costs per call)
Option C: Hybrid (On-device for common items, cloud fallback)
```

**Recommendation: Start with Cloud API → migrate hot models on-device**

### Stack

| Layer | Technology |
|---|---|
| Camera capture | `expo-camera` or `react-native-vision-camera` |
| Barcode scan (fast path) | ML Kit Barcode Scanning (on-device, free) |
| Visual product recognition | Google Cloud Vision API → Product Search |
| Custom model (later) | YOLOv8 trained on Thai retail products |
| On-device inference | TensorFlow Lite (Android) + Core ML (iOS) via `react-native-fast-tflite` |
| Product database | PostgreSQL + product image embeddings (pgvector) |

### Detection Pipeline

```
Camera frame
    → Barcode scanner (on-device, instant)
        → Found barcode → lookup product DB → show result ✅
    → No barcode → send frame to Cloud Vision API
        → Match against product catalog embeddings
        → Return top-3 matches with confidence
        → Draw bounding boxes on frame
```

### Training Data for Custom Model (YOLOv8)

- Scrape product images from retailer catalog
- Augment with shelf photos (different lighting, angles, partial occlusion)
- Label with LabelImg or Roboflow
- Train YOLOv8n (nano, 6MB) for mobile deployment
- Export to `.tflite` (Android) and `.mlmodel` (iOS)

---

## 5. Backend

### Service Architecture

```
┌─────────────────────────────────────────────────┐
│                  Mobile App                      │
└────────┬──────────────┬──────────────┬───────────┘
         │              │              │
    REST API       WebSocket      CV API
         │              │              │
┌────────▼──────┐ ┌─────▼──────┐ ┌───▼────────────┐
│  API Server   │ │ Realtime   │ │  CV Inference  │
│  Node.js +    │ │ Server     │ │  FastAPI +     │
│  Fastify      │ │ Socket.io  │ │  TF Serving    │
└────────┬──────┘ └─────┬──────┘ └───▼────────────┘
         │              │            S3 (model)
    ┌────▼──────────────▼────┐
    │       PostgreSQL        │
    │  + Redis (cache/RT)     │
    └────────────────────────┘
```

### API Server (Node.js + Fastify)

| Endpoint group | Responsibility |
|---|---|
| `/auth` | JWT login, OAuth (LINE Login for Thai users) |
| `/store` | Store map, zone definitions, beacon config |
| `/route` | TSP route calculation (server-side, upgradeable to OR-Tools) |
| `/products` | Product catalog search, barcode lookup |
| `/coupons` | Issue, redeem, wallet |
| `/game` | Monster events, catch records, leaderboard |
| `/analytics` | Brand dashboard data (zone heatmap, redemption stats) |

### Database: PostgreSQL

```sql
-- Key tables
users, stores, zones, beacons
products, product_images
routes, route_steps
monsters, coupons, coupon_redemptions
game_sessions, game_events
brand_campaigns
```

### Real-time: Redis + Socket.io
- Live leaderboard updates
- Beacon event stream (user entered zone X)
- Multiplayer: see other shoppers on map (future)

### CV Inference: Python FastAPI
- Wraps TensorFlow Serving or runs ONNX model directly
- Accepts base64 image → returns `[{label, bbox, confidence}]`
- Rate-limited per user (avoid abuse)

---

## 6. Store Map & Route Engine

### Indoor Map

| Option | Fit |
|---|---|
| Custom SVG (current approach, scaled up) | ✅ Works for single store, easy to edit |
| Mapbox GL JS with Indoor Plugin | Good for multi-store, GeoJSON-based |
| HERE Indoor Maps | Enterprise-grade, complex |
| Apple Maps Indoor (MapsIndoors) | iOS only |

**Recommendation:** Keep custom SVG approach but make it data-driven — store the map in JSON (zones, aisles, corridors) and render dynamically. This allows store managers to edit layout without code changes.

### Route Engine (Upgrade from Nearest Neighbor TSP)

| Algorithm | Quality | Notes |
|---|---|---|
| Nearest Neighbor (current) | ~75% of optimal | Fast, good enough for <15 stops |
| 2-opt improvement | ~90% of optimal | Run after NN, same JS code |
| Google OR-Tools (Python) | Near-optimal | Run server-side for large lists |
| Dijkstra on grid graph | Exact pathfinding | Use for pixel-level path, not zone ordering |

**Real routing needs two layers:**
1. **Zone ordering** — TSP: which zones to visit in what order
2. **Path within store** — graph pathfinding on walkable grid (avoid shelves)

---

## 7. Auth & Users

- **LINE Login** — dominant social login in Thailand, mandatory for Thai retail apps
- **Apple Sign In** — required by App Store if any social login offered
- **Google Sign In** — Android fallback
- JWT access tokens (short-lived) + refresh tokens (stored in secure storage)
- User profile: name, photo, points, coupon wallet, shopping history

---

## 8. Analytics (Brand Dashboard)

**For retailers/brands to see ROI**

| Layer | Technology |
|---|---|
| Event tracking | Custom events → Kafka → ClickHouse |
| Query API | Python FastAPI on top of ClickHouse |
| Visualization | Recharts (existing demo approach) embedded in app, or separate web dashboard |
| KPIs | Coupon issuance, redemption rate, zone dwell time, route heatmap |

**Key events to track**
- `zone_entered` (from beacon)
- `monster_appeared`, `monster_caught`, `monster_escaped`
- `coupon_issued`, `coupon_redeemed`
- `shelf_scan_started`, `product_detected`
- `route_started`, `route_completed`

---

## 9. Deployment

| Layer | Service |
|---|---|
| Mobile app distribution | TestFlight (iOS beta) → App Store / Google Play |
| API Server | AWS ECS / Railway / Render |
| Database | AWS RDS (PostgreSQL) + ElastiCache (Redis) |
| CV Inference | AWS SageMaker or dedicated EC2 GPU instance |
| Object Storage | AWS S3 (model files, product images) |
| CDN | CloudFront |
| CI/CD | GitHub Actions → build + deploy on push to main |

---

## 10. Phased Build Plan

### Phase 1 — Foundation (Month 1–2)
- [ ] React Native app scaffold (Expo)
- [ ] Auth (LINE Login + JWT)
- [ ] Product catalog + search (API + DB)
- [ ] Store map (data-driven SVG)
- [ ] TSP route calculation (server-side)
- [ ] Manual step-through navigation (no beacons yet)

### Phase 2 — Beacon Navigation (Month 3–4)
- [ ] Procure and deploy BLE beacons in test store
- [ ] RSSI fingerprint calibration
- [ ] Real-time positioning pipeline
- [ ] Live position on store map
- [ ] Auto-advance waypoint when user enters zone

### Phase 3 — AR Game (Month 4–5)
- [ ] ViroReact integration
- [ ] 3D monster models (GLTF)
- [ ] AR plane detection → monster spawn
- [ ] Swipe-to-throw mechanic
- [ ] Coupon issuance + wallet

### Phase 4 — CV Shelf Scanner (Month 5–6)
- [ ] Cloud Vision API integration (V1)
- [ ] Product DB with images
- [ ] Bounding box overlay on camera feed
- [ ] Barcode scanner fast path
- [ ] Custom YOLO model training (V2)

### Phase 5 — Analytics + Brand Portal (Month 6–7)
- [ ] Event tracking pipeline
- [ ] Brand dashboard web app
- [ ] Zone heatmap
- [ ] Coupon ROI reports

---

## Estimated Hardware (Per Store)

| Item | Qty | Unit Cost | Total |
|---|---|---|---|
| BLE Beacons (Estimote) | 30–50 | $20 | $600–1,000 |
| Beacon batteries (yearly) | — | $2/beacon | ~$80/yr |
| Installation labor | 1 day | — | — |

Software costs scale with users (cloud inference, DB hosting).

---

## Key Risk Areas

| Risk | Mitigation |
|---|---|
| Beacon RSSI is noisy in metal shelving environments | Use fingerprinting + Kalman filter; test early |
| ARCore/ARKit plane detection fails on glossy floors | Test on real store floor; add fallback (image target) |
| CV model accuracy on Thai product packaging | Collect local training data; start with barcode fallback |
| LINE Login approval process | Apply early; takes 1–2 weeks |
| App Store approval (AR + camera permissions) | Write clear privacy policy; justify all permissions |
