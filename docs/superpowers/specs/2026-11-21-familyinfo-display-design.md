# FamilyInfoDisplay — Design Spec

Status: complete v12 (pending approval)
Date: 2026-11-21

## 1. Intent & Why
A shared family info display on one tablet: weather, today's/tomorrow's appointments, and sailing weather (Segelwetter) visible at a glance.

## 2. Goals
- Tile-based home screen: one tile per topic
- Weather tile: today's forecast 06:00–16:00, tomorrow's forecast 16:00–06:00
- Today's + tomorrow's appointments from the family's Google Calendar
- Segelwetter page embedded
- New topics easy to add; every tile hideable, hidden tiles restorable

## 3. Non-Goals
- Not iOS, not multi-device — one tablet
- No calendar editing (read-only display)
- No multi-location or extended-range weather
- No in-app accounts or login
- No plugin system — new topics are code additions (a tile entry in a list), not runtime plugins

## 4. Anti-Goals (never do)
- TBD

## 5. Platform & Constraints
- Target: Lenovo Tab TB311FU — 10.1-inch IPS WUXGA (1920×1200 landscape, 16:10), 224 ppi, 60 Hz, MediaTek Helio G85, 4 GB RAM, 64 GB storage (+ microSD), Android 16 (API 36) on ZUI 17.5, USB-C, LTE/WiFi
- minSdk/targetSdk: 36
- Language: Kotlin, single activity, Jetpack Compose
- Orientation: landscape only, no portrait
- Normal app (no kiosk, no boot autostart); screen stays on while the app is in the foreground (KEEP_SCREEN_ON)
- UI language: German (all labels, tile titles, and user-facing strings)
- Permissions: READ_CALENDAR, ACCESS_FINE_LOCATION (both requested once at first use)
- WebView for the Segelwetter tile
- AndroidX DataStore for tile visibility and settings
- Third-party resources and licenses documented in `THIRD_PARTY_LICENSES.md` (repo) and shown on the about screen (section 7.7)

## 6. Users
- Whole family, shared read-only display, no per-user profiles

## 7. Features

### 7.1 Tile framework
- Layout (fixed, 2 columns):
  - Column 1: weather tile on top (~1/3 of column height), calendar tile below (scrollable, fills the rest)
  - Column 2: Segelwetter WebView, full column height
- Each topic = a tile entry (id, title, content builder, default-visible) in a central list
- Visibility stored per tile id; a hidden tile's slot collapses and its column's remaining tiles expand
- New topics beyond the 3 are appended to the grid (layout for 4+ tiles chosen at that time)
- Settings screen lists all tiles with on/off switches; hidden tiles are re-enabled there

### 7.2 Weather tile
- Source: Open-Meteo (free, no API key), single configurable location
- 06:00–16:00: today — current temperature, max temperature, chance of rain during the day
- 16:00–06:00: tomorrow — min and max temperature, chance of rain
- Numbers plus unicode icons
- Refresh: on launch + every 15 minutes (default, adjustable later)

### 7.3 Calendar tile
- Reads locally synced Google Calendar via the Android Calendar ContentProvider (one-time permission grant; no API key)
- Shows today's and tomorrow's events (time + title), all synced calendars
- Constraint: the family's Google account must be signed in and calendar sync enabled on the tablet

### 7.4 Segelwetter tile
- WebView loading https://superfloh247.github.io/Segelwetter/
- Needs network; simple offline fallback text

### 7.5 Settings screen
- Entry: settings icon on the home screen
- Tile visibility: all tiles with on/off switches, persisted to DataStore
- Weather location:
  - "Locate me" button: gets current device location (ACCESS_FINE_LOCATION, one-time prompt) and fills the coordinate field
  - Input field for coordinates (lat, lon), editable by hand
  - Saved to DataStore; weather tile uses it

### 7.6 Visual design (Grafana 13 inspired)
- Dark theme: near-black app background, panels slightly lighter (Grafana's default dark palette)
- Tiles styled as Grafana panels: 1px subtle border, small rounded corners, header row with title left-aligned
- Accent: Grafana orange (#F46800) for active elements (settings icon, buttons, highlights)
- Font: Inter (bundled as res/font resource, OFL license), compact Grafana-like hierarchy: panel title small, values large
- Status colors like Grafana: green = good (no rain), yellow = moderate, orange/red = heavy rain or busy day
- Flat, minimal shadows, readable at a distance (tablet sits on a table, not held)

### 7.7 About screen
- Entry: "Info" item on the settings screen
- Shows: app name + version, purpose (one line), app license (Beerware with ko-fi link)
- Third-party resources with licenses:
  - Inter font — SIL Open Font License 1.1 (https://openfontlicense.org/), by Rasmus Andersson
  - Open-Meteo weather API — Creative Commons Attribution 4.0 for data products (https://open-meteo.com/#disclaimer), free for non-commercial use
  - Segelwetter page (florian500) — Beerware license (kaffee: https://ko-fi.com/florian500); the page also states: OpenStreetMap contributors (ODbL), Leaflet (BSD-2-Clause), Open-Meteo terms, EOT20 tide model (DGFI/TUM). We only embed the page by URL, so these apply to the page itself
- Licenses kept in a single source of truth: `THIRD_PARTY_LICENSES.md` in the repo; the about screen shows the short list and links, full license texts not bundled in-app (out of scope — one line per resource is enough for this app)

## 8. Data
- Tile visibility, weather location, settings: DataStore (local)
- Weather: fetched from Open-Meteo, last response cached in memory (tile shows stale data offline until next successful fetch)
- Calendar: Android Calendar ContentProvider (device-synced data)
- Offline: calendar and tile layout work offline; weather and Segelwetter need network
- First launch: no weather location set → weather tile shows "set location in settings" until user acts

## 9. Open Questions
(none — all resolved)

## 10. Resolved Facts
- Family Google Calendar is synced on the tablet; system calendar app shows all relevant appointments (calendar tile will work via ContentProvider)
- Tablet: Android 16 / ZUI 17.5, 4 GB RAM, 64 GB storage, Helio G85 — user's device is the 4/64 variant; Compose confirmed feasible, no Views fallback needed
- Display is 16:10 (not 16:9): grid design must account for the taller screen

## 11. Planning & Execution Rules (hard requirements)
- The build/implementation/test plan must be broken down into the smallest possible and sensible tasks and subtasks. Every plan step must be a single bounded action with a clear done-criterion; no combined steps like "build X and test Y."
- The task executor must never combine tasks or subtasks: one task (or one subtask) per execution step, no merging, no doing a second task "while at it." Each executed item is checked off on its own.

## 12. App License
Beerware — Wenn dir diese App gefällt, spendiere mir einen Kaffee: https://ko-fi.com/florian500
(Stated in-app on the about screen and in the repo `LICENSE.md`.)
