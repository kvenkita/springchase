# Spring Chase

**Find your perfect spring, anywhere in America.**

Spring Chase is a single-page web app that helps travelers identify the optimal 30-day spring weather window for all 50 US states — then plan road trips around those windows.

Live at **[springchase.netlify.app](https://springchase.netlify.app)**

---

## What it does

Spring is the best season, but it peaks at wildly different times across the country. In North Carolina, ideal outdoor weather (highs ≤85 °F, lows in the 50s) arrives around late March. In Montana it's late May. In Arizona it's February.

Spring Chase answers: *When does spring happen in every state, and how do I plan a trip that catches it?*

### Features

- **Climate Windows Timeline** — All 50 states displayed as horizontal bars on a year calendar, showing each state's 30-day spring window. Filter by region (Southeast, Northeast, Midwest, South, Mountain, Pacific), sort by date or alphabetically, and search by state name.
- **Route Planner** — Pick a start date and trip length (1–120 days). An interactive D3 map highlights every state whose spring window overlaps your travel dates, with a card grid showing city-level detail for each match.
- **Today banner** — The hero section shows which states are currently in their spring window right now.
- **City-level detail** — Each state's window is derived from 4 geographically spread cities (e.g., coastal vs. inland, north vs. south) so you can see the regional variation within a state.

---

## Data

Weather windows are based on **NOAA 30-Year Climate Normals (1991–2020)**. A state's spring window is defined as the period when the average high is ≤85 °F and the average overnight low is approximately 50–59 °F.

Each state's `avgDoy` (average day-of-year start) is the rounded mean of 4 representative cities. Caveats:
- **Hawaii** — sea-level lows stay mid-60s, not quite 50s (flagged ⚠)
- **Alaska** — Fairbanks interior doesn't reliably reach 50 °F overnight (flagged ⚠)
- **North Carolina** — the reference benchmark state (flagged ★), window: Mar 31 – Apr 30

---

## Tech stack

| Layer | Technology |
|---|---|
| App | Vanilla HTML / CSS / JS — zero build step |
| Map rendering | [D3.js v7](https://d3js.org/) |
| Map geometry | [TopoJSON v3](https://github.com/topojson/topojson) + [US Atlas](https://github.com/topojson/us-atlas) |
| Typography | Google Fonts — Cormorant Garamond, DM Sans, DM Mono |
| Hosting | [Netlify](https://netlify.com) |

The entire app lives in a single self-contained `index.html` file with no dependencies to install.

---

## Deployment

The app is a static file with no build step. Deploy to Netlify by dragging `index.html` into the Netlify dashboard, or via CLI:

```bash
netlify deploy --prod --dir . --file index.html
```

Or connect this GitHub repo to a Netlify site for automatic deploys on push.

---

## Potential future features

- Fall window mapping (the southward return of the season)
- Live weather API overlay for current forecasts
- Shareable trip plan URLs with encoded state
- Print / PDF export of matched states
