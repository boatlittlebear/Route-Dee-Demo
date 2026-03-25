# RouteDee — Smart Route Demo

A mobile-first demo app for **RouteDee**, a smart in-store navigation product that calculates the shortest shopping route, lets users hunt deal monsters, collect coupons, and view brand analytics. Built as a single HTML file using React 18, Tailwind CSS, and Babel (all CDN — no build step required).

---

## How to Run

Open `demo.html` directly in a browser. No server, no npm, no build needed.

Designed for mobile viewport (375px wide). Use DevTools device mode or open on a phone.

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI Framework | React 18 (CDN, UMD) |
| Styling | Tailwind CSS (CDN) + inline styles |
| JSX transpilation | Babel Standalone (CDN) |
| Font | Inter (Google Fonts) |
| Language | English UI, Thai Baht (฿) currency |

**No build pipeline.** Everything is in one file: `demo.html`.

---

## App Structure

The app has 5 main screens navigated via a bottom tab bar:

### 1. Home (`home`)
Brand dashboard — simulates a retailer/brand analytics view. Shows:
- KPI cards: Coupons Issued, Actual Redeems, Sales Lift, ROI
- Coupon channel comparison bar chart (RouteDee vs Paper/Line/Email)
- Zone heatmap (which zones redeemed most)
- Revenue model breakdown (SaaS, Brand Deal, Data Analytics)

### 2. Map (`map`) — Main Shopping Flow
Multi-step shopping flow:
1. **Product Selection** — Browse/search 77 products across 11 categories. Filter by zone.
2. **Route Map** — SVG store map showing the calculated smart route with animated yellow dashed path and step-by-step waypoints.
3. **Walking Mode** — Step through waypoints one by one. Each stop shows items to collect. Deal monsters may appear.
4. **Monster Catch** — Gamified coupon catch mini-game (throw → hit/miss → catch coupon).
5. **Summary** — Post-shopping stats: calories burned, steps, distance, time. Plus savings and coupon count.

### 3. Coupons (`coupon`)
Wallet of caught coupons with discount value, brand, expiry date, and a "Use" button.

### 4. Ranking (`rank`)
Monthly leaderboard ranked by total savings. Shows top 3 with prize tiers (฿500/฿300/฿100 coupons), ranks 4–10, and the current user's position with how much more to save to rank up.

---

## Store Map Layout

```
SVG: 312 × 316 px  (maps to physical store proportions)

┌─────────────┬──────┬─────────────────────────┬──────────┐
│  POS/CASH   │DAIRY │     PRODUCE (wide)       │MEAT/FRZN │  Row 0  y=4..82
│  (block)    │(thin)│                          │          │
├─────────────┴──────┼──────────────────────────┴──────────┤  H_AISLE[0] = y=90
│    BAKERY          │ HOUSEHOLD│BEAUTY│RICE│SEASON│SWEETS  │  Row 1  y=96..192
│    (wide left)     │ ←──────── 5 walk-through aisles ────→│
├────────────────────┴──────────────────────────────────────┤  H_AISLE[1] = y=202
│       BEVERAGES (left half)   │  COFFEE/TEA (right)       │  Row 2  y=208..246
└───────────────────────────────────────────────────────────┘  H_AISLE[2] = y=252
                                                                H_AISLE[3] = y=266 (main aisle)

Entrance: bottom-left  (x=40, y=304)
Exit/POS: top-left     (x=42, y=44)
```

### Zone Definitions (`ZONE_DEFS`)

Each zone has: `rx, ry, rw, rh` (SVG rect), `row` (0/1/2), colors (`bg/bd/tc`), `icon`, `name`.

| ID | Name | Row | Position |
|---|---|---|---|
| `fresh` | Dairy | 0 | Top, narrow (x=88) |
| `veg` | Produce | 0 | Top, wide (x=110) |
| `meat` | Meat/Frozen | 0 | Top, right (x=236) |
| `bakery` | Bakery | 1 | Left block (x=4) |
| `clean` | Household | 1 | Aisle 1 (x=86) |
| `beauty` | Beauty | 1 | Aisle 2 (x=116) |
| `grain` | Rice/Grains | 1 | Aisle 3 (x=146) |
| `spice` | Seasonings | 1 | Aisle 4 (x=176) |
| `sweet` | Sweets | 1 | Aisle 5 (x=206) |
| `drink` | Beverages | 2 | Bottom left (x=4) |
| `coffee` | Coffee/Tea | 2 | Bottom right (x=126) |

---

## Routing Algorithm

### Shortest Path: Nearest Neighbor TSP

`nearestNeighbor(zoneIds)` — greedy nearest-neighbor from `entrance`, visits each required zone once, ends at `exit` (POS).

### Direction-Aware Two-Sided Zone Routing (`aisleRoute`)

Each zone has two access aisles — top and bottom. Routing logic:

- **Going UP** (toward row 0, smaller y): exit source via `topY`, enter target via `bottomY`
- **Going DOWN** (toward row 2, larger y): exit source via `bottomY`, enter target via `topY`
- **Same row**: both use `bottomY`
- **Adjacent rows** share the same `H_AISLE` value → no vertical detour needed
- **Non-adjacent rows**: use left-wall vertical corridor at `x=2` (`VCORR=2`) to travel between aisle levels

This means the path walks **through** zones (in one end, out the other), never backtracks into a dead end.

### Horizontal Aisles (`H_AISLE`)

```js
const H_AISLE = [90, 202, 252, 266];
//               ↑    ↑    ↑    ↑
//           row0↔1  1↔2  2↔main  main(entrance level)
```

### Distance Calculation (`aisleDistance`)

Manhattan distance using the same direction-aware routing logic. Used by TSP for cost comparison.

---

## Product Catalog

77 products across 11 zones, 7 items each. Defined in `CATALOG` array:

```js
{ id: Number, name: String, emoji: String, zone: ZoneId }
```

Categories: Produce, Dairy, Meat/Frozen, Rice/Grains, Seasonings, Sweets, Beverages, Coffee/Tea, Household, Beauty, Bakery.

---

## Monster System (Gamified Coupons)

`ZONE_MONSTERS` — one potential monster per zone (6 zones have monsters):

| Zone | Brand | Discount | Rarity |
|---|---|---|---|
| veg | Dole Organic Veg | ฿20 | Rare |
| meat | CP Meats | ฿30 | Rare |
| spice | Pantai Seasonings | ฿15 | Common |
| drink | Ichitan Beverages | ฿15 | Common |
| clean | Attack Laundry | ฿50 | Legendary |
| beauty | Pantene Shampoo | ฿40 | Epic |

Rarity tiers: Common (green) → Rare (blue) → Epic (purple) → Legendary (gold).

Mini-game: 3 attempts (hearts), each throw has random success chance. Caught monsters go into coupon wallet.

---

## Key React Components

| Component | Description |
|---|---|
| `App` | Root — manages screen state, coupons, waypoints |
| `ProductPicker` | Grid browser with search + zone filter |
| `StoreMap` | SVG map with animated route + step control |
| `MonsterCatch` | Monster encounter mini-game |
| `SummaryScreen` | Post-shop stats with calorie/step/distance cards |
| `CouponWallet` | List of caught coupons |
| `LeaderBoard` | Monthly savings ranking |
| `BrandDashboard` | Brand analytics dashboard (home screen) |

### State Management

All state lives in `App` via `useState`. Key state:
- `screen` — current screen/tab
- `selected` — Set of selected product IDs
- `waypoints` — computed route (built by `buildWaypoints`)
- `stepIdx` — current walking step
- `coupons` — caught coupon array
- `collected` — Set of visited waypoint indices

---

## SVG Map Visuals

Zones are drawn as **open-ended shelves**:
- Solid left/right walls (shelf sides)
- Dashed top/bottom edges (walkable entrances)
- Horizontal shelf lines inside narrow aisles (row 1 zones with `rw < 40`)
- Vertical product dividers inside wide zones

Route path: yellow (`#ffd700`) dashed `<polyline>` with `stroke-dasharray` animation (`ani-route`), 1.8s draw-on effect. Walking position shown as a pulsing green circle.

---

## CSS Animations

Defined in `<style>` block:

| Class | Effect |
|---|---|
| `.ani-float` | Floating up/down loop |
| `.ani-bounce-in` | Elastic pop-in |
| `.ani-shake` | Horizontal shake (throwing) |
| `.ani-slide-up` | Slide up fade-in |
| `.ani-num-glow` | Glowing number pulse |
| `.ani-route` | Route path draw-on animation |

---

## Color Theme

Dark green / black background (`#050f07`) — Lotus supermarket brand colors.

- Primary: `#007934` (Lotus green)
- Accent: `#ffc300` (yellow)
- Background: `#050f07` → `#0a1a0e`
- Text: white / `#86efac` (light green) / `#4ade80`

---

## File Structure

```
demo.html          ← entire app (single file, ~1400 lines)
README.md          ← this file
```

---

## Git History Summary

```
4eb6cd4  feat: translate all UI text from Thai to English
b3f468d  style: change font from Kanit to Inter
45490b1  feat: store map matches reference floor plan
e98177b  fix: zones drawn as open-ended shelves matching routing logic
f5ccb3f  fix: direction-aware two-sided zone routing (walk-through aisles)
68ea04d  fix: aisle routing via left-wall corridor + walk-to-POS exit step
3375cb1  feat: store map matches reference floor plan
df38530  feat: realistic store map layout + 77 products
6931958  style: Lotus color theme (green/yellow/white)
1561ce5  feat: Smart Route + Realistic Store Map v0.3
84d74d7  feat: RouteDee AR Demo v0.2
4e4eb47  feat: RouteDee AR Demo v0.1
```

---

## Notes for Future AI Context

- **Single file app** — do not split into components or add a build system unless explicitly asked.
- **No npm / no bundler** — all dependencies loaded from CDN.
- **Mobile-first** — optimized for 375px viewport.
- **Routing logic is the most complex part** — `aisleRoute` and `aisleDistance` must stay in sync. If you change zone positions (`ZONE_DEFS`), update `H_AISLE` values to match new aisle Y-positions.
- **TSP is nearest-neighbor greedy** — not exact optimal, intentional for demo simplicity.
- **Monsters are per-zone, one per zone** — triggered when a waypoint zone has a matching entry in `ZONE_MONSTERS`.
- **All UI is in English** — previous versions had Thai text.
- **฿ (Thai Baht) currency symbol** is intentional — this is a Thai retail product demo.
- The `VCORR=2` left-wall corridor is important — it is kept clear of all zone blocks on purpose to allow vertical routing between non-adjacent rows.
