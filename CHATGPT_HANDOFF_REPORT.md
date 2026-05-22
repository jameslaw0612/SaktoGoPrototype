# ChatGPT Handoff Report

Date: 2026-05-22
Project: SaktoGoPrototype / Olongapo Ride-Matching Demo

## 1. Executive Summary

`SaktoGoPrototype` is the current canonical app folder. It contains a working ride-request, route-visualization, and driver-simulation demo centered on Olongapo City.

The system is built from:

- a single-page frontend in [index.html](/E:/SaktoGov3/SaktoGoPrototype/index.html:1), [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:1), and [styles.css](/E:/SaktoGov3/SaktoGoPrototype/styles.css:1)
- a Python standard-library backend in [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:1)
- a dedicated passenger-driver intelligent matching module in [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:1)

The intelligent matching is not machine learning. It is a deterministic, explainable, weighted scoring system that ranks candidate drivers using road-network routing, live traffic when available, live weather when available, driver quality attributes, and movement behavior.

The most important current-state points are:

- the backend matcher is intended to be the primary source of truth
- live TomTom traffic is the primary traffic input for passenger-driver matching
- live Open-Meteo weather is the primary weather input
- heuristic traffic and fallback weather behavior remain as graceful fallback paths
- the user-mode phone UI now has its own in-phone “Finding the best driver” loading overlay

## 2. Current Architecture

### Frontend

The frontend in [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:1) is responsible for:

- rendering the Leaflet map and UI
- running the simulated fleet
- handling pickup/drop-off selection
- calling the backend for intelligent matching
- showing route, weather, and traffic summaries
- using browser-side ranking only as an emergency fallback

Important frontend entry points:

- intelligent-match request: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:1912)
- matcher source badges rendering: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:667)
- phone-only match loading overlay logic: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:330)
- pickup/drop-off traffic loading: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:3926)
- route traffic summary loading: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:3953)
- backend traffic proxy caller: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:4037)
- traffic formatter: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:4093)

### Backend

The backend in [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:1) is a lightweight HTTP server that:

- bootstraps Olongapo map data and landmarks
- builds and stores an in-memory routable road graph
- loads live weather
- looks up live traffic from TomTom
- exposes routing and matching APIs
- delegates intelligent ranking to `passenger-driver.py`

Important backend pieces:

- default TomTom fallback key constant: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:28)
- live weather loader: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:206)
- live traffic capability check: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:232)
- single-point live traffic lookup: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:235)
- batched live traffic lookup: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:289)
- A* routing: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:620)
- intelligent ranking bridge: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:702)
- bootstrap payload: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:721)
- intelligent match endpoint: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:1063)
- traffic sample endpoint: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:1086)

### Intelligent Matcher Module

The main intelligent system is isolated in [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:1). The primary entry point is:

- ranking entry: [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:20)

This file currently contains the core logic for:

- candidate filtering
- route-aware driver evaluation
- traffic and weather context building
- ETA calculation
- weighted score calculation
- explanation generation
- debug field generation

## 3. Passenger-Driver Matching: Current State

### End-To-End Flow

The current backend-first ride-matching flow is:

1. The frontend collects ride type, pickup, and drop-off.
2. The frontend sends the request to `/api/intelligent-match`.
3. The backend snaps pickup and drop-off onto the road graph.
4. The backend verifies a valid A* route exists.
5. The matcher filters nearby eligible drivers.
6. Each candidate driver is evaluated using route distance, traffic, weather, driver attributes, and movement behavior.
7. The backend returns ranked suggestions, explanation text, source metadata, and debug breakdowns.
8. The frontend displays the suggested driver, source badges, and explanation.
9. If backend matching is unavailable, the browser can still rank drivers in `browser_fallback` mode, but that path is explicitly secondary.

### Candidate Filtering

Current filtering behavior is still preserved:

- selected vehicle type only
- `standby_available` or `moving_available` only
- exclude locked and held drivers
- exclude `assigned_pickup`, `assigned_ontrip`, and inactive drivers by status gate
- search within `3 km` first
- expand to `5 km` if needed
- skip drivers that do not produce a valid A* pickup route

Filtering and evaluation are centered in [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:282).

### Scoring Formula

The current final weighted score is:

```text
final_score =
  distance_score * 0.30 +
  traffic_score * 0.20 +
  weather_score * 0.10 +
  rating_score * 0.10 +
  cancellation_score * 0.10 +
  route_efficiency_score * 0.15 +
  movement_score * 0.05
```

This is applied in [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:385).

### What Each Score Uses

`distance_score`

- uses local A* route distance on the in-memory OSM-based road graph
- not a live external API at scoring time

`traffic_score`

- primary source: TomTom live traffic
- fallback: heuristic traffic estimation
- current route traffic context builder: [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:451)

`weather_score`

- primary source: Open-Meteo live weather
- fallback: neutral/default weather behavior
- current weather context builder: [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:211)

`rating_score`

- simulated driver rating from the frontend fleet model
- not a live API

`cancellation_score`

- simulated driver cancellation rate from the frontend fleet model
- not a live API

`route_efficiency_score`

- local routing/geometric comparison between direct distance and actual routed pickup distance
- not a live API

`movement_score`

- based on driver state and motion direction within the simulation
- not a live API
- movement logic: [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:548)

## 4. Live Environment Data In Matching

### Traffic: TomTom Is Primary

Live traffic is now handled in the backend and fed into the matcher consistently.

Backend traffic capability:

- env-var-first key selection with demo fallback key in code
- live traffic enabled check: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:232)
- TomTom lookup helpers: [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:235) and [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:289)

Matcher traffic behavior:

- candidate pickup routes are sampled along the route
- the matcher asks the backend service for live traffic samples
- `trafficRatio` is derived from `currentSpeed / freeFlowSpeed`
- ratios are clamped to a safe range
- road closures are heavily penalized
- if live data is unavailable, the matcher falls back gracefully to the heuristic estimator

Core traffic logic:

- traffic context builder: [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:451)
- heuristic fallback estimator: [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:524)

Result metadata now includes:

- `trafficSource`
- `trafficNotice`
- `trafficSamplesUsed`

Aggregate source/notice helpers:

- [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:637)
- [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:641)

Current traffic source values are:

- `tomtom_live`
- `heuristic_fallback`

### Weather: Open-Meteo Is Primary

Weather is still loaded by the backend and used by the matcher as the main weather input.

Backend weather loader:

- [app.py](/E:/SaktoGov3/SaktoGoPrototype/app.py:206)

Matcher weather behavior:

- uses live weather when available
- computes weather severity and multiplier
- uses fallback behavior if weather is unavailable

Current weather context builder:

- [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:211)

Current weather source values are:

- `open_meteo_live`
- `weather_fallback`

Result metadata now includes:

- `weatherSource`
- `weatherNotice`

## 5. ETA Composition

ETA is still calculated and displayed, but it is not directly part of the weighted score.

ETA calculation:

- [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:542)

The ETA is composed from:

- route distance
- driver speed
- traffic effect
- weather effect

More specifically:

- `distance_meters`
  - comes from the local A* route on the road graph
- `speed_kph`
  - comes from the simulated driver state
- `traffic_ratio`
  - comes from TomTom live traffic when available, otherwise heuristic fallback
- `weather_multiplier`
  - comes from Open-Meteo live weather when available, otherwise fallback weather behavior

So ETA is only partially real-API-based:

- real API component: traffic
- real API component: weather
- non-API component: route distance
- non-API component: simulated driver speed

## 6. Matching Output And Explainability

For each result, the backend can provide:

- selected driver
- pickup and trip ETAs
- final score
- explanation text
- `matchingMode`
- `trafficSource`
- `trafficNotice`
- `weatherSource`
- `weatherNotice`
- per-driver `scoreBreakdown`
- per-driver `debug`

Important matching output behavior:

- backend-primary results use `matchingMode: "backend_primary"`
- browser emergency fallback uses `matchingMode: "browser_fallback"`

The explanation generator is in [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:426).

The relative weighted scores are applied in [passenger-driver.py](/E:/SaktoGov3/SaktoGoPrototype/passenger-driver.py:385).

## 7. Frontend Matching Behavior

### Backend Is Intended Source Of Truth

The frontend calls `/api/intelligent-match` first and treats the backend result as primary:

- [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:1912)

If backend matching fails or is unavailable, the browser still has an emergency fallback path. That fallback is clearly marked as browser fallback mode in the returned offer payloads.

### Source Visibility In The UI

The suggested driver card has explicit matcher-source badges:

- markup: [index.html](/E:/SaktoGov3/SaktoGoPrototype/index.html:127)
- styles: [styles.css](/E:/SaktoGov3/SaktoGoPrototype/styles.css:446)
- render logic: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:667)

These badges distinguish:

- backend primary vs browser fallback
- TomTom live traffic vs heuristic fallback
- Open-Meteo live weather vs fallback

### Traffic Cards And Route Summaries

Pickup, drop-off, and route traffic summaries use backend traffic sampling rather than direct frontend TomTom calls:

- point traffic: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:3926)
- route traffic: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:3953)
- backend traffic proxy fetch: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:4037)

The traffic summary formatter was also adjusted so road-closure messaging is clearer:

- [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:4093)

### User-Mode Loading Experience

When the user is in phone mode and the app is ranking candidates, the loading animation now appears inside the phone screen rather than over the whole browser window:

- phone overlay markup: [index.html](/E:/SaktoGov3/SaktoGoPrototype/index.html:45)
- styles: [styles.css](/E:/SaktoGov3/SaktoGoPrototype/styles.css:1042)
- show/hide logic: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:330) and [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:346)
- ranking trigger: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:1852)

### Debug Output

Browser debug logging for candidate details is gated behind:

- `?debugMatch=1`
- switch: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:11)

This avoids constant console spam during normal use while still exposing scoring details when needed.

## 8. Driver Simulation And Ride Lifecycle

The app still has a fairly rich simulated fleet. Driver initialization remains in the frontend:

- driver creation uses simulated rating and cancellation behavior
- rating builder: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:3724)
- cancellation builder: [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:3728)

Drivers still move through the expected lifecycle:

- available standby
- moving available
- assigned to pickup
- assigned on-trip
- released back into general simulation after drop-off

This means the recent intelligent matching changes did not replace the original ride flow. They improved how the best driver is chosen, explained, and presented.

## 9. Route Alignment / Suggested Driver Preview

One recent frontend fix addressed a route-preview mismatch where the suggested driver’s preview could appear slightly off the expected street alignment.

Current sync step:

- [app.js](/E:/SaktoGov3/SaktoGoPrototype/app.js:2477)

This resynchronizes the pending suggested driver with the held driver state before drawing the preview, so the driver-to-pickup route should better match the actual road-network start point used by the simulation.

## 10. Current Strengths

- The intelligent matcher is cleanly separated into its own Python module.
- Passenger-driver matching is now backend-first instead of split across unrelated live-traffic code paths.
- TomTom live traffic is integrated into the backend matching path.
- Open-Meteo weather remains integrated into the backend matching path.
- The matching output is explainable and includes explicit source metadata.
- The UI makes backend-vs-fallback mode more visible.
- The user-mode phone UI has a better localized loading experience during matching.
- The driver simulation and ride lifecycle are already integrated end-to-end.
- The road graph and routing approach are more realistic than straight-line-only matching.

## 11. Current Limitations And Risks

### 11.1 Still Rule-Based, Not Learned

The system is still an explainable heuristic scorer, not a trained ML model. It does not learn from historical trips, acceptance behavior, supply-demand patterns, or real operational KPIs.

### 11.2 Browser Fallback Still Exists

The backend is the intended source of truth, but there is still fallback ranking logic in the browser. That is useful for resilience, but it still creates maintenance drift risk over time.

### 11.3 External API Reliability Still Matters

Live behavior depends on external services such as:

- TomTom
- Open-Meteo
- Nominatim
- Overpass
- OSM tile/CDN sources

The app now fails more gracefully than before, but it is still sensitive to network issues, quotas, and third-party downtime.

### 11.4 TomTom Key Handling Is Demo-Friendly, Not Production-Safe

The backend prefers `TOMTOM_API_KEY` from the environment, but there is also a code-level fallback key for convenience/demo use. That makes local sharing easier, but it is not appropriate for production or public repository exposure.

### 11.5 No Persistence

The app still runs entirely in memory:

- no trip database
- no driver database
- no user accounts
- no historical analytics

Restarting resets the state.

### 11.6 No Formal Regression Test Suite

The code structure is inspectable and the scripts can be syntax-checked, but there is still no visible automated regression coverage for:

- matching logic
- route correctness
- source/fallback behavior
- backend/browser parity

## 12. Best Short Description For Another ChatGPT Session

Use this description:

"This is an Olongapo ride-matching and route-visualization demo. The current canonical app folder is `SaktoGoPrototype`. The frontend is a Leaflet single-page app with a simulated driver fleet, a phone-style user UI, and a ride lifecycle from suggestion to pickup to drop-off. The backend is a Python HTTP server that loads OSM-based road data, builds an in-memory routable graph, fetches live weather from Open-Meteo, and fetches live traffic from TomTom. The main intelligent matching logic is in `passenger-driver.py`. It ranks candidate drivers with a deterministic weighted formula using routed pickup distance, live-traffic-derived traffic score, live-weather-derived weather score, rating, cancellation risk, route efficiency, and movement behavior. ETA is computed from route distance, simulated speed, traffic ratio, and weather multiplier, but ETA is not directly scored. The backend is the intended source of truth, and the frontend only falls back to browser ranking in emergency `browser_fallback` mode. Matching results expose `trafficSource`, `weatherSource`, notices, score breakdowns, and debug details." 

## 13. Validation Status

What was validated directly in source:

- backend live weather flow exists
- backend live TomTom traffic flow exists
- `/api/traffic-samples` route exists in code
- passenger-driver matching contains live traffic and weather source metadata
- frontend exposes matcher source badges and browser fallback labeling
- point and route traffic UI call the backend traffic proxy
- the phone-mode match loading overlay exists in the current UI

What was not fully revalidated in this report update:

- live third-party API success at runtime in this environment
- full browser interaction from cold boot through completed ride
- exact parity across every edge case between backend-primary and browser fallback behavior
