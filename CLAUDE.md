# Denver Tree Map — Project Context

## 1. Project Overview

**Denver Tree Map** is a mobile-first web app that visualizes Denver's urban tree inventory on an interactive map for field use by arborists, urban planners, and curious residents.

- **Live URL**: https://sanarteaga.github.io/denver-tree-map
- **Repo**: https://github.com/sanarteaga/denver-tree-map
- **Hosting**: GitHub Pages (main branch)
- **Developer**: Santiago (sole developer, also primary field user)
- **Deployment**: `git commit` + `git push` from local clone at `C:\Users\User\Claude_Code\denver-tree-map` — GitHub Pages auto-deploys from `main`

## 2. Architecture

**Single-file app** — everything lives in `index.html` (~1500 lines, ~63 KB):
- HTML, CSS, and JavaScript all inline
- No build step, no bundler, no framework
- All libraries loaded from CDN (cdnjs, unpkg)
- Zero dependencies installed locally

**Why this architecture**: Simplicity of deployment (drag one file into GitHub), no toolchain maintenance, no JS framework learning curve. Trade-off: one large file is harder to navigate, but grep and section comments keep it manageable.

## 3. Tech Stack

| Library | Version | Purpose |
|---------|---------|---------|
| Leaflet | 1.9.4 | Base map rendering |
| Leaflet.MarkerCluster | 1.5.3 | Tree marker clustering |
| leaflet-rotate | 0.2.8 | Map rotation for heading mode |
| CARTO Positron tiles | — | Basemap (light, clean, no referer required) |
| Space Mono + DM Sans | — | Fonts (Google Fonts) |

**Browser APIs used**:
- `navigator.geolocation` — GPS (`getCurrentPosition`, `watchPosition`)
- `SpeechSynthesisUtterance` — audio guide TTS
- `DeviceOrientationEvent` — compass for heading mode (iOS 13+ requires permission prompt)

## 4. Data Source

### 4.1 Tree Inventory API

- **Dataset**: Denver Urban Tree Inventory
- **Provider**: Colorado Open Data (data.colorado.gov)
- **Dataset ID**: `wz8h-dap6` (the map view ID `hzmx-2dfk` does NOT support API queries)
- **Endpoint**: `https://data.colorado.gov/resource/wz8h-dap6.json`
- **Auth**: None — public, CORS-open
- **Query language**: Socrata SoQL (`$select`, `$where`, `$limit`, `$group`, `$order`)

**Example query**:
```
https://data.colorado.gov/resource/wz8h-dap6.json?$limit=50000&$select=species_common,species_botanic,diameter,stems,condition,site_designation,address,street,neighbor,x_long,y_lat&$where=x_long BETWEEN '-104.995' AND '-104.985' AND y_lat BETWEEN '39.734' AND '39.744'
```

### 4.2 Key Fields

| Field | Description |
|-------|-------------|
| `species_common` | Common name ("Norway Maple") |
| `species_botanic` | Scientific name |
| `diameter` | Trunk diameter class (truncated strings — see mapping below) |
| `stems` | Number of stems |
| `condition` | Good, Fair, Poor, Dead, Critical, Excellent |
| `site_designation` | Site type (Parkway, Park, etc.) |
| `address` | Street number |
| `street` | Street name |
| `neighbor` | Neighborhood name |
| `x_long` | Longitude (WGS84) |
| `y_lat` | Latitude (WGS84) |

### 4.3 Diameter Mapping (API Quirk)

The API truncates diameter values to 8 characters. The app maps them back to their real ranges via the `DIAM_MAP` constant and `formatDiam()` function:

| API value | Display value |
|-----------|---------------|
| `0 to 6` | 0 to 6 |
| `6 to 12` | 6 to 12 |
| `12 to 1` | 12 to 18 |
| `18 to 2` | 18 to 24 |
| `24 to 3` | 24 to 30 |
| `30 to 3` | 30 to 36 |
| `36 to 4` | 36 to 42 |
| `42 to 4` | 42 to 48 |
| `48 +` | More than 48 |
| `N/A` or empty | Not available |

### 4.4 Other APIs

- **Zip geocoding**: `https://api.zippopotam.us/us/{zip}` — CORS-open, no auth (replaced Nominatim which had CORS issues with User-Agent headers)
- **GitHub commits**: `api.github.com/repos/sanarteaga/denver-tree-map/commits?path=index.html&per_page=1` — used at boot to fetch the version display info

## 5. Key Constants

```js
const TREE_API         = 'https://data.colorado.gov/resource/wz8h-dap6.json';
const GEO_API          = 'https://api.zippopotam.us/us';
const LIMIT            = 50000;           // max trees per query
const CENTER           = [39.7392, -104.9903]; // Denver center
const RADIUS_DEG       = 0.0029;          // 0.2 miles in degrees for Near Me
const RELOAD_BOUNDARY_DEG = 0.03 * 0.01449; // 0.03 miles — auto-reload threshold
const TRIGGER_METERS   = 5;               // audio guide proximity trigger
const JOG_SPEED_MS     = 2.98;            // 9 min/mile in m/s
const JOG_LOOKAHEAD    = 3.0;             // seconds to project ahead
const JOG_PROJ_M       = 8.94;            // JOG_SPEED_MS * JOG_LOOKAHEAD
const EARTH_R          = 6371000;         // Earth radius in metres
```

## 6. Feature Reference

### 6.1 Entry Screen

Three-tab interface, Near Me active by default:
- **📍 Near Me** (leftmost, default): Uses device GPS, loads trees within 0.2 miles
- **📮 Zip**: Enter a 5-digit zip code, loads trees in a ~4-mile bounding box around zip centroid
- **🏘 Neighborhood**: Dropdown populated from API on startup, queries by exact neighborhood name

Version display (commit SHA + timestamp) shown below subtitle, fetched from GitHub API.

### 6.2 Map Screen

**Layout**:
- Map fills screen
- Header (search + filter ⚙︎) at top, full width
- "Showing X trees" + "✎ Change area" pills below header, left side
- Compass button (🧭), top right, below area bar
- Locate (◎), bottom right
- Audio guide (🎧), bottom right above locate
- Jogging mode (🏃), bottom right above audio

**Marker colors** by condition:
- Green (`#2d7d4e`): Excellent/Good
- Amber (`#c47f17`): Fair
- Red (`#c0392b`): Poor/Dead/Critical
- Gray: Unknown

**Clustering** via MarkerCluster, max radius 50px, disabled at zoom 17+.

### 6.3 Detail Sheet

Opens when a marker is tapped. Shows:
- Emoji icon based on condition (🌲 🌳 🌿 🍂 🪵)
- Botanical name (small, uppercase-ish, accent color)
- Common name (large)
- Grid: Condition (with colored dot), Diameter (mapped), Site type, Stems
- Address + neighborhood at bottom

**Highlight circle**: Green filled circle (4m radius, `#5dbd6e` fill, `#2d6a3f` stroke) drawn on the tree when sheet opens, removed when it closes.

**Dismiss**: Tap map, tap handle, swipe down more than 55px.

### 6.4 Search & Filter

- **Search bar**: Matches `species_common`, `species_botanic`, or `neighbor` (case-insensitive, debounced 280ms)
- **Filter sheet** (⚙︎ button): Condition chips (multi-select) + Diameter class chips (single-select, exclusive)
- Diameter chips use all 9 mapped values (0–6, 6–12, ..., More than 48)

### 6.5 Locate Button (◎)

Gets current GPS, pans map, shows pulsing blue dot, loads trees within 0.2 miles, resets all filters. Sets `nearMeCenter` for auto-reload.

### 6.6 Audio Guide (🎧)

Field mode for hands-free tree identification while walking:
- **Triggers** when user GPS is within 5 meters of any tree (Haversine distance)
- **Reads each tree once only** (`spokenTreeIds` Set)
- **Speech format**: `"{species_common}, diameter {mapped_diameter}, {stems} stems"`
- **Stumps/vacant sites**: Only reads the name, no diameter or stems
- On trigger: map pans to user, green highlight circle on tree (auto-removes 8s)
- Button turns solid dark green (`#1a4d28`) when active
- Status pill at bottom: "Listening…" → "{species name}"
- Also starts 30-second auto-reload boundary check

### 6.7 Jogging Mode (🏃)

For audio guide while jogging (faster than walking). At 9 min/mile, GPS lag means a ~3-second delay between actual position and reading — this mode compensates:

- **Assumed speed**: 2.98 m/s (9 min/mile)
- **Lookahead**: 3 seconds
- **Projection**: ~8.94m ahead of current GPS position
- Uses last 2 GPS fixes to compute heading (`bearingTo()`), then projects position forward (`projectPoint()`)
- Triggers audio when *projected* position is within 5m of tree
- **Overrides audio guide** — stops it if running, activates same read-once logic
- Button turns dark blue (`#1a3d6e`) when active
- Same highlight circle, same status pill ("Jogging…")
- Lower `maximumAge: 1000` for fresher GPS

### 6.8 Compass / Heading Mode (🧭)

Toggles between north-up and heading-up orientation:
- **Source**: Device magnetometer via `DeviceOrientationEvent` (works standing still)
- **iOS 13+**: Requires one-time permission prompt on first tap
- **Mode B** (lock position): When active, map rotates to put direction of travel at top AND auto-centers on user position continuously
- **Compass SVG**: Red needle (north) + gray needle (south), counter-rotates to always point true north
- Tap again to return to north-up (map bearing 0, needle resets)
- Stops on "Change area"

### 6.9 Auto-Reload (when Near Me is loaded)

- Runs every **30 seconds** while audio or jogging mode is active
- Checks if user is within 0.03 miles of edge of loaded 0.2-mile area
- If yes: silently refetches trees centered on new position, clears `spokenTreeIds`, shows toast
- Resets `nearMeCenter` to null on "Change area"

### 6.10 Double-Tap Zoom

Explicitly enabled via `doubleClickZoom: true` in map init. Leaflet handles the touch gesture natively.

## 7. Code Structure in `index.html`

Rough top-to-bottom order (section comments use `─── SECTION NAME ───`):

1. `<head>` — CDN scripts, fonts, CSS variables, all styles
2. `<body>` — Entry screen, loading overlay, map div, header, area bar, filter sheet, detail sheet, buttons, toast
3. `<script>` — the app logic, organized as:
   - CONFIG (constants)
   - MAP (Leaflet init, tiles, clusters)
   - ICONS (tree marker SVGs, condition dot)
   - STATE (allTrees, filters, search)
   - TAB SWITCHING
   - FETCH NEIGHBORHOODS (runs at boot)
   - ZIP LOGIC
   - NEIGHBORHOOD LOGIC
   - GPS ENTRY HANDLER (Near Me)
   - SHARED FETCH + SHOW (zip path)
   - DRAW (drawMarkers, fitToTrees)
   - CHANGE AREA
   - DETAIL SHEET (showDetail, closeDetail, highlight circle)
   - FILTER SHEET
   - SEARCH
   - FILTERS (applyFilters)
   - GPS LOCATION (showLocationDot, locate button)
   - AUTO-RELOAD (boundary check)
   - AUDIO GUIDE (startAudioGuide, stopAudioGuide)
   - DIAMETER DISPLAY MAPPING (DIAM_MAP, formatDiam, buildSpeech)
   - JOGGING MODE (projectPoint, bearingTo, start/stop)
   - HEADING MODE (device orientation, compass button)
   - HELPERS (showLoading, setMsg, setProg, showToast)
   - BOOT (loadNeighborhoods, focus input, version display)

## 8. Development Workflow

### 8.1 Local Editing

Working copy lives at:
```
C:\Users\User\Claude_Code\denver-tree-map\index.html
```

All edits happen to this single file. Claude Code uses the `Edit` tool to modify it in place.

### 8.2 Syntax Verification

Before committing, verify the inline `<script>` block parses:
```bash
node --check <(sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d')
```

### 8.3 Deployment

From `C:\Users\User\Claude_Code\denver-tree-map`:
```bash
git add index.html
git commit -m "..."
git push
```

GitHub Pages auto-deploys from `main` within ~45 seconds. Filename must stay `index.html`.

### 8.4 When Santiago Asks for Changes

Typical flow:
1. Santiago writes a user story ("As a user I want...")
2. Claude may ask 1–2 clarifying questions for ambiguous UX decisions (especially for new interactive features)
3. Claude may research competitive behavior if explicitly requested ("check Google Maps for this")
4. Claude edits the local file, runs syntax check, verifies all changes landed with grep-based checks, and presents the file
5. Santiago field-tests, reports back

## 9. Known Quirks & Gotchas

- **API diameter truncation**: Always use `formatDiam()` — never display raw API diameter values
- **Stumps and vacant sites**: `buildSpeech()` only reads the name for these (no diameter/stems which are often bogus)
- **iOS speech priming**: First `SpeechSynthesisUtterance` must be triggered from a user gesture — Audio Guide and Jogging Mode both prime with an empty utterance after the button tap
- **iOS DeviceOrientation permission**: First tap of compass button must trigger `DeviceOrientationEvent.requestPermission()` — this is why heading mode can't auto-activate
- **Green tint bug (fixed)**: A previous dashed radius ring in `showLocationDot()` with `fillOpacity: 0.04` caused a visible green wash over the 0.2-mile area on iPhone in sunlight. The ring was removed entirely.
- **Marker cluster colors**: Use `!important`-free overrides via `.marker-cluster-small/medium/large div` selectors
- **CARTO tiles**: Chosen because OpenStreetMap tiles block requests without a Referer header when served from GitHub Pages

## 10. Developer Persona & Approach

Santiago combines data science methodology with arborist domain knowledge. He values:
- **Actionable urban forestry insights** — monoculture risk, canopy equity, species distribution
- **Field testing** — features are iterated based on real walks/jogs with the app
- **Clean UX** — wants features that work the way they look, no surprises
- **Direct feedback loop** — prefers quick iterations over lengthy discussions

When adding features, bias toward:
- Concrete constants over magic numbers (e.g., `JOG_PROJ_M` not `8.94`)
- Section comments in the JS so grep keeps working
- Respecting existing patterns (highlight circle, status pill reuse, etc.)
- Defensive checks (`if (map.setBearing)`) since this is an enhanced Leaflet setup

## 11. Species ID Field Reference

Useful facts from prior conversations for conversations about the dataset or field identification:

- **Winter ID**: leaf scar shape is most reliable — white ash has deep C-notch; green ash is flat or shallow
- **Summer ID**: leaflet underside color is easiest for ash and related genera
- **Common Denver genera** in the dataset: maple, ash, elm, oak, honeylocust, linden, crabapple, hackberry

## 12. Future Directions (Mentioned but Not Built)

- **Data analytics dashboard**: species distribution charts, monoculture risk scores, canopy equity metrics by neighborhood
- **Offline mode**: cache recent queries for field use without signal
- **Photo annotations**: attach field photos to a tree ID (would need a backend)
- **Route recording**: track where the user walked and which trees they encountered
