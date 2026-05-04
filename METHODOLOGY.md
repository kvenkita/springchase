# Spring Chase — Methodology

This document describes how the spring weather windows shown in Spring Chase are computed: the climate criterion used, the data source, the process of translating monthly temperature normals into a day-of-year start, the city selection and averaging approach, and the math behind the Route Planner's overlap logic.

---

## 1. The spring benchmark

The target condition is defined as:

> **Average daytime high ≤ 85 °F and average overnight low in the 50s °F (50–59 °F)**

These thresholds were chosen to match what April in North Carolina feels like — warm enough to be outdoors comfortably all day, cool enough at night to sleep with the windows open. North Carolina is the reference state (marked ★ in the tool) and its window (approximately March 31 – April 30, averaging Charlotte, Raleigh, Asheville, and Wilmington) anchors the benchmark.

The 85 °F high ceiling matters because it marks the onset of oppressive summer heat in most of the country — above that threshold, outdoor exertion and open-air travel become significantly less comfortable. The 50s low criterion captures the "crisp evening" quality of true spring: cool enough to feel seasonal, but not cold enough to require heavy gear overnight.

---

## 2. Data source

All temperature data comes from the **NOAA U.S. Climate Normals (1991–2020)**, published by the National Centers for Environmental Information (NCEI).

NOAA's 30-year normals report monthly average maximum (high) and minimum (low) temperatures for hundreds of weather stations across the United States. Using a 30-year period dampens single-year anomalies and provides a stable long-term baseline — these figures represent what a location *typically* experiences in a given month, not any particular year.

The 1991–2020 period is the current official normal, updated every decade. This dataset supersedes the previous 1981–2010 normals and reflects observed warming trends.

---

## 3. Translating monthly normals to a day-of-year start

NOAA normals are reported **by month**, not by day. To find the first day when spring conditions arrive, the following process is applied for each city:

### Step 1 — Identify the threshold month

Scan monthly normal high and low temperatures to find the first month where the average high is at or below 85 °F **and** the average overnight low has entered the 50s.

In most continental US cities, spring conditions begin somewhere between February and June. The month-pair that brackets the transition (e.g., March average still below threshold, April average above) defines the target zone.

### Step 2 — Interpolate to a day within the month

Monthly averages represent the midpoint of that month. To estimate the specific day when conditions cross the threshold, linear interpolation is applied between the adjacent months' averages:

```
frac = (threshold − prev_month_value) / (next_month_value − prev_month_value)
start_day = mid_of_prev_month + frac × days_between_midpoints
```

This gives a fractional day-of-year, which is then rounded to the nearest integer to produce `startDoy` for that city.

**Example — Charlotte, NC:**  
March normal: avg high ≈ 64 °F, avg low ≈ 44 °F → low not yet in 50s  
April normal: avg high ≈ 74 °F, avg low ≈ 54 °F → both criteria met  
Interpolation places the threshold crossing around March 31 (day 90), which is Charlotte's `startDoy = 91`.

### Step 3 — Apply both criteria simultaneously

The start day is the **later** of the two individual threshold crossings:
- The day the high crosses down through 85 °F (heading into spring from summer, or crossing up from cool spring into comfort)
- The day the low crosses up through 50 °F (nights warming out of the 40s)

In practice the low criterion typically lags the high by days to a couple of weeks, making it the binding constraint in most states. Arizona is the notable exception where the high criterion arrives later in high-elevation cities like Flagstaff.

---

## 4. The 30-day window

Each city's spring window is fixed at **30 days starting from `startDoy`**, spanning `[startDoy, startDoy + 30]` inclusive. This is a standardized estimate for comparative purposes — not a precise forecast of exactly when conditions change.

The 30-day duration is intentional: it represents a planning horizon (roughly one month) and is long enough to capture meaningful trip windows without overstating the duration of comfortable conditions. In reality, some states hold ideal conditions for 3–6 weeks; others (particularly desert states like Arizona) have a narrower comfortable window before heat sets in.

---

## 5. City selection

A single city per state can be misleading given the size and climate diversity of US states. Spring arrives in Seattle (coastal, maritime) nearly three weeks later than in Spokane (inland, continental), both in Washington state.

To capture this variation, **four cities are selected per state** with the goal of representing distinct climate zones or geographic spread — typically one each from the north, south, east, and west of the state, or from clearly differentiated climate regimes:

| Selection principle | Example |
|---|---|
| N/S/E/W geographic spread | Iowa: Des Moines (central), Sioux City (NW), Cedar Rapids (NE), Dubuque (NE, elevated) |
| Coastal vs. inland | California: San Diego (coastal), Sacramento (inland valley), San Francisco (coast fog), Los Angeles (coastal basin) |
| Valley vs. mountain | Colorado: Denver (Front Range), Colorado Springs (piedmont), Grand Junction (western slope), Pueblo (southern plains) |
| Climate zone diversity | Washington: Seattle (maritime), Spokane (inland), Yakima (rain shadow), Olympia (Puget lowlands) |

The four cities for each state are stored in the data as `[[cityName, startDoy], ...]` and are displayed in the tooltip when hovering any state row or map state.

---

## 6. State-level averaging

The state's `avgDoy` (average start day-of-year) is computed as:

```js
avgDoy = Math.round((city1_startDoy + city2_startDoy + city3_startDoy + city4_startDoy) / 4)
```

This is a simple arithmetic mean, rounded to the nearest integer. The result represents the average spring arrival date across the four sampled cities — not weighted by population, area, or any other factor.

The state window displayed in the timeline and on the map runs from `avgDoy` to `avgDoy + 30`.

**Example — North Carolina (reference state):**

| City | `startDoy` | Calendar |
|---|---|---|
| Charlotte | 91 | Apr 1 |
| Raleigh | 88 | Mar 29 |
| Asheville | 100 | Apr 10 |
| Wilmington | 82 | Mar 23 |
| **State average** | **Math.round(90.25) = 90** | **Mar 31** |

---

## 7. Day-of-year (DOY) convention

The tool uses a **day-of-year integer** as its internal date representation throughout, with:

- January 1 = day 1
- December 31 = day 365

The `doy()` function computes today's day-of-year from the system clock:

```js
function doy() {
  const n = new Date(), s = new Date(n.getFullYear(), 0, 0);
  return Math.floor((n - s) / 86400000);
}
```

The `fmt(d)` function converts a DOY integer back to a human-readable date (e.g., `fmt(90)` → `"Mar 31"`) by constructing a `Date` object anchored to January 2025 (a non-leap year used for stable display arithmetic):

```js
function fmt(d) {
  const dt = new Date(2025, 0, d);
  return dt.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}
```

The tool does not account for leap years — all windows are represented as if the year has 365 days. The resulting ±1 day discrepancy in leap years is negligible for the planning use case.

---

## 8. Route Planner overlap logic

### Matching

A state is included in the Route Planner results if its spring window **overlaps at all** with the user's selected trip window. The `getMatches()` function implements standard interval overlap detection:

```js
function getMatches() {
  const tripEnd = pStart + pLen - 1;
  return S.filter(([,,,sd]) => sd <= tripEnd && sd + 30 >= pStart);
}
```

Two intervals [A, B] and [C, D] overlap when `A ≤ D && C ≤ B`. Here:
- State window: `[sd, sd + 30]`
- Trip window: `[pStart, pStart + pLen - 1]`

Even a single day of overlap qualifies the state as a match and lights it up on the map.

### Overlap count and percentage

For each matching state, the number of overlapping days is:

```js
const overlapDays = Math.min(sd + 30, pStart + pLen - 1) - Math.max(sd, pStart) + 1;
```

This is `min(end_of_state, end_of_trip) − max(start_of_state, start_of_trip) + 1` — the standard formula for the length of the intersection of two intervals (with the +1 because both endpoints are inclusive).

The overlap percentage displayed in each match card is:

```js
const pct = Math.round(overlapDays / 30 * 100);
```

This expresses what fraction of the state's 30-day spring window falls inside the trip window — a high percentage means the trip captures most of that state's peak season.

---

## 9. State-specific caveats

### Hawaii (⚠)
Sea-level overnight lows in Hawaii stay in the mid-60s year-round due to trade winds and tropical maritime climate. The low criterion (~50–59 °F) is never strictly met at typical visited elevations. Hawaii's `startDoy` is included based on the high criterion alone, and it is flagged ⚠ to communicate that the lows caveat applies.

### Alaska (⚠)
Alaska's window is computed for the state's most populated cities including Anchorage, Juneau, and Kodiak (all of which have relatively maritime climates). Fairbanks in the interior does approach but doesn't reliably reach 50 °F overnight during the window shown. The ⚠ flag reflects this. Alaska's window (~day 152, early June) is also the latest of all 50 states.

### Arizona
Arizona presents the widest intrastate spread of any state: Phoenix (day 46, mid-February), Yuma (day 32, February), and Tucson (day 55, late February) are all desert cities that enter spring conditions early before heat arrives rapidly — while Flagstaff at ~2,100 m elevation (day 121, May 1) behaves like a Mountain West state. The four-city average lands around day 64 (early March), which is a compromise that doesn't accurately describe any single part of the state.

### North Carolina (★)
North Carolina is the reference benchmark state. Its window (avg day 90, March 31) was used to define the spring conditions criterion. It is flagged ★ in the timeline and tooltips.

### Washington
Seattle (day 145, late May) and Spokane (day 130, early May) differ by 15 days — one of the larger intrastate spreads in the dataset — due to the Olympic rain shadow and contrasting maritime vs. continental climates.

---

## 10. Limitations

**Averages, not forecasts.** The NOAA 30-year normals represent long-run average conditions. Any given year may deviate by 1–3 weeks in either direction due to El Niño, La Niña, or other interannual variability. Always verify with a live forecast service before departure.

**Elevation is undersampled.** The 4-city sample captures broad geographic variation but cannot fully represent every mountain pass, high meadow, or valley microclimate within a state. A mountain route in Colorado at 3,500 m may be 3–4 weeks behind the city averages — or inaccessible due to snow — even when the state appears "in window."

**Month-to-day interpolation is approximate.** Converting monthly normals to a single start day involves interpolation. Errors of ±3–5 days are likely for most cities; errors of ±7–10 days are possible in cities with unusual temperature curves (e.g., strong marine influence, high diurnal range, or monsoon onset effects).

**Climate change is shifting windows.** The 1991–2020 normals already reflect observed warming compared to earlier normal periods. Windows in many states are trending 1–2 weeks earlier compared to the 1981–2010 normals, and this trend is expected to continue. The data represents typical conditions over the last 30 years, not a prediction of future conditions.

**4-city averaging can obscure extremes.** When two or more cities in a state have very different spring windows (Arizona, Washington, California), the state average may not represent any single sub-region well. The tooltip city breakdown is the right tool for understanding intrastate variation.

**The 30-day window duration is fixed.** In reality, some states hold ideal conditions for 5–7 weeks while others (especially desert states) have a much shorter comfortable window. Treating all states as having a 30-day spring season is a simplification made for visual comparability.
