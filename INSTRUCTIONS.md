# Solar Siting — Implementation Instructions

> **Purpose**: Step-by-step instructions for an AI coding agent to implement changes
> to the existing Phase 1 notebook and create new notebooks for Phases 2–5.
> Every instruction must be applied **programmatically** without breaking existing
> working code.

---

## Global Conventions (apply to ALL notebooks)

### Database
| Key | Value |
|---|---|
| Connection | `PG_CONN` variable from Configuration cell |
| Output schema | `analysis` |
| CRS for calculations | EPSG:5321 (Ontario MNR Lambert, `CRS_ONTARIO`) |
| CRS for data exchange | EPSG:4326 (WGS84, `CRS_WGS84`) |
| Bridge CRS | EPSG:4269 (NAD83 geographic, `CRS_NAD83`) |

> ⚠️ **EPSG:3161 is deprecated** — do not use in any new or modified code. All spatial operations use EPSG:5321.

### Naming
- Output tables: `analysis.<step_label>_<county_slug>` (e.g. `analysis.aoi_county_bruce`)
- County slug: `COUNTY_NAME.lower().replace(" ", "_")`
- Unit of analysis: **Phases 1–3 operate on lots. Phase 4 onward operates on PARCELS.**
- HTML maps: `map_phase<N>_<county_slug>.html`

### Source Schemas & Tables

| Schema | Table | Key Columns |
|---|---|---|
| `administrative` | `ontario_geographic_township` | `geom` |
| `administrative` | `adm_district_township` | `name_2`, `geom` |
| `grid` | `oem_grid` | `capacityrange`, `ldc_name`, `station_name`, `feeder_name`, `capacity`, `feeder_ltl_voltage_3ph`, `geom` |
| `cadastre` | `ontario_lots` | `fid`, `lot_ident`, `concession_ident`, `geographic_township_name`, `objectid`, `refreshed_datetime`, `name`, `geom` |
| `cadastre` | `parcels_{county_slug}` | `parcel_id`, `geom` |
| `cadastre` | `buildings_{county_slug}` | `geom` |
| `official_plans` | `land_use_{county_slug}` | `design_en` (or `landusedesc`), `geom` |
| `rea` | `ansi`, `conservation_reserve`, `prov_park_zone`, `waterbody`, `watercourse`, `wetland`, `woodland` | `buffer_dist`, `geom` |
| `rea` | `disturbance_exclusions` | `geom` |
| `rea` | `{CONSERVATION_AUTH}_regulated_area` | `geom` — parametrised per county (see Phase 2 config) |
| `environment` | `prov_trk_species` | `species_name`, `saro_status`, `geom` |
| `environment` | `significant_ecological_area` | `geom` — includes PSW, ANSI |
| `environment` | `nhs_area` | `geom` |
| `environment` | `wildlife_activity_area` | `geom` |
| `environment` | `cli` | `CLI_SYS`, `geom` — Canadian Land Inventory |

### Utility Functions (reuse in every notebook)
```python
read_postgis(sql, geom_col="geom")   # Execute SQL → GeoDataFrame in WGS84
save_to_postgis(gdf, table, label)   # Write GeoDataFrame → analysis.<table> in EPSG:5321 + GiST index
```

### Rules
1. **Save-After-Every-Step** — call `save_to_postgis()` immediately after each query. The saved table is the authoritative input for the next step.
2. **Read-From-Analysis-Schema** — read inputs from `analysis.*` tables, never re-derive CTE chains from prior steps.
3. **Mark Implementation State** — append ✅ / 🔧 / ❌ to section headings after implementation.

---

## Phase 1 — Electrical Feasibility ✅

### File: `phase1_solar_siting_v2.ipynb`

All fixes implemented and verified for Bruce, Ottawa, Oxford, and Peterborough.

| Fix | Description | Key Change |
|-----|-------------|------------|
| 1 ✅ | Save after every step | Removed bulk-save cell; each step calls `save_to_postgis()` immediately |
| 2 ✅ | Chain inputs via analysis schema | Steps 2 & 3 read from `analysis.*` instead of re-deriving CTEs |
| 3 ✅ | Step 3 query improvements | `STRING_AGG(DISTINCT ...)` aggregation; `ST_Transform` on smaller OEM set for GiST index use |
| 3b ✅ | Step 2 clip-before-dump | Reordered CTEs: clip → dump → filter (ST_Dump on small clipped set) |
| 4 ✅ | Markdown updates | Step descriptions updated for schema chaining & save-after-every-step |
| 5 ✅ | Geometry validity & overlap gate | `ST_MakeValid()` in Step 2; `ST_Area(ST_Intersection(...)) > 0` in Step 3 |

**Steps**:
1. Build County AOI (dissolved + buffered township boundary) → `aoi_county_{slug}`
2. Clip OEM grid to AOI (> 10 MW, clip → dump → area filter) → `oem_county_{slug}`
3. Select lots intersecting OEM zones (STRING_AGG aggregation) → `selected_lots_{slug}`

**CRS Note**: Using EPSG:5321 (not 3161) to avoid PROJ NTv2 grid dependency on PostGIS server.

---

## Phase 2 — REA Environmental Exclusions ✅

### File: `phase2_solar_siting.ipynb`

Fully implemented. Builds unified REA exclusion layer, then subtracts from Phase 1 lots.

**Config**:
```python
COUNTY_NAME = "Ottawa"           # change per county
MIN_NET_ACRES = 40
CONSERVATION_AUTH = "rvca"       # orca (PTB) | rvca (OTT) | utrca (OXF) | tbd (BRC/SIM)
```

> ⚠️ `CONSERVATION_AUTH` must be set per county. It controls which `rea.{CONSERVATION_AUTH}_regulated_area` table is included in the exclusion layer. Never hardcode `"orca"` for all counties.

**Steps**:
1. **Build REA exclusion layer** — 8 buffered layers (`buffer_dist`) + 1 hard exclusion (`disturbance_exclusions`), clipped to AOI, dissolved, repaired (`ST_MakeValid → ST_Buffer(0) → ST_SimplifyPreserveTopology(1) → ST_SnapToGrid(0.01) → ST_MakeValid`) → `rea_exclusions_{slug}`
2. **Subtract exclusions** — `ST_Difference` per lot, compute `acre_net` + `pct_remaining`, filter ≥ `MIN_NET_ACRES` → `lots_developable_{slug}`
3. **Verification & map** — summary stats + Folium map (lots colored by `pct_remaining`, REA overlay) → `map_phase2_{slug}.html`

---

## Phase 3 — Lot Boundary Recovery & Land Use Filtering ✅

### File: `phase3_lot_parcels.ipynb`

Fully implemented. Recovers original lot boundaries, applies land-use and building exclusions, intersects with parcels.

**Config**: `COUNTY_NAME`, `MIN_LOT_ACRES` (40), `MIN_PARCEL_ACRES` (40), `LANDUSE_COL` (varies per county: `design_en` or `landusedesc`), `BUILDING_BUFFER_REA` (150 m), `BUILDING_BUFFER_DEFAULT` (30 m), `ALLOWED_LANDUSE`, `EXCLUDED_LANDUSE`

**Steps**:
1. **Step 1A — Lot boundaries**: `ST_PointOnSurface` on eroded lots → match back to `cadastre.ontario_lots` → `selected_lots_boundary_{slug}`
2. **Step 1B — Parcels**: Extract from `cadastre.parcels_{slug}` within dissolved lot envelope, ≥ `MIN_PARCEL_ACRES` → `selected_parcels_{slug}`
3. **Step 2 — Dump + land-use clip + filter** (inspection only, no DB save): `ST_Dump` → clip against `official_plans.land_use_{slug}` → ILIKE filter
4. **Step 3A — Building exclusion**: Conditional buffer (150 m in REA zones, 30 m outside) → dissolve → `building_exclusion_{slug}`
5. **Step 3B — Final developable area parcels**: Single CTE chain: dump → land-use clip → designation filter → building subtract → parcel intersection → `developable_area_parcels_{slug}`
6. **Step 3C — Developable parcel boundaries**: Attribute-join `selected_parcels` to `developable_area_parcels` on `parcel_id` → `developable_parcels_{slug}`
7. **Verification & map** → `map_phase3_{slug}.html`

**Land Use Filters** (adjust per county):
```python
ALLOWED_LANDUSE = ["%rural%", "%rural industrial%", "%industrial%", "%employment%"]
EXCLUDED_LANDUSE = ["%agricultural%", "%residential%", "%urban%", "%village%",
    "%institutional%", "%commercial%", "%environmental%", "%conservation%",
    "%open space%", "%natural%", "%heritage%"]
```

---

## Geocoding — Parcel Address Enrichment ✅

### File: `geocoding_parcels.ipynb`

> Standalone utility — runs independently after Phase 3.

Fully implemented. Reverse-geocodes parcel centroids via Mapbox v6 API, assigns deterministic UIDs, updates `developable_parcels_{slug}` in-place.

**Config**: `COUNTY_NAME`, `MAPBOX_TOKEN` (from `.env`), `GEOCODE_BATCH_DELAY` (0.05s)

**Steps**:
1. Load parcels + compute WGS84 centroids via `ST_PointOnSurface`
2. Generate `solar_parcel_uid` (UUID-5 from county + lot + concession + parcel_id)
3. Reverse-geocode: Mapbox v6 with fallback strategy (address → place/locality)
4. Merge geocoding columns into GeoDataFrame
5. Save → overwrites `developable_parcels_{slug}`
6. Verification

**Columns added**: `solar_parcel_uid`, `lat`, `lon`, `main_address`, `secondary_addresses` (JSON), `county`, `township`, `locality`, `province`

---

## Phase 4 — Physical & Environmental Suitability 🔧

### File: `phase4_physical_suitability.ipynb`

> **Status**: Refactored — single-geometry architecture.

### Architecture

**Single geometry column**: `parcels_phase4_{slug}` carries only `buildable_geom`
(one row per parcel, unioned from `developable_area_parcels_{slug}`). Steps that need
the full parcel boundary join back to `developable_parcels_{slug}` in SQL. This avoids
geopandas dual-geometry serialization issues.

**Incremental enrichment**: `update_parcels_phase4()` uses SQL `ALTER TABLE` + `UPDATE`
via a temp table — geometry columns are never touched by pandas.

### Purpose
Compute all parcel-level physical, environmental, and suitability attributes required as inputs to Phase 5 scoring and triage. Unit of analysis: **parcels** (from `developable_area_parcels_{slug}`).

### Input
- `analysis.developable_area_parcels_{county_slug}` — Phase 3 output: buildable geometry fragments (one or more per parcel)
- `analysis.developable_parcels_{county_slug}` — Phase 3 output: full parcel boundaries (one per parcel, enriched by geocoding)

### Config
```python
COUNTY_NAME       = "Ottawa"
CONSERVATION_AUTH = "rvca"       # orca | rvca | utrca — must match Phase 2
DEM_PATH          = "..."        # path to DEM raster (GeoTIFF, EPSG:5321)
PVOUT_PATH        = "..."        # path to PVout raster (GeoTIFF, EPSG:5321)
```

### Steps

#### Step 1 — Create Phase 4 Working Table
Union buildable fragments per parcel from `developable_area_parcels_{slug}`.
Carry forward grid attributes. Compute `buildable_area_ac` from the unioned geometry.
Scalar parcel metadata (address, UID) are NOT stored here — they live in `developable_parcels_{slug}`.
```sql
SELECT parcel_id,
       ST_MakeValid(ST_Union(geom)) AS geom,   -- buildable_geom, EPSG:5321
       ROUND((ST_Area(ST_Union(geom)) / 4046.856)::numeric, 3) AS buildable_area_ac,
       ... grid_capacity_mw, voltage_3ph ...
FROM analysis.developable_area_parcels_{slug}
GROUP BY parcel_id
```
Save → `analysis.parcels_phase4_{slug}` (single geometry = `geom`, which IS the buildable area).

#### Step 2 — Slope
Rasterio zonal stats on DEM raster over `geom` (buildable area). Store `slope_mean` (float, %).

#### Step 3 — PVout (Irradiance)
Rasterio zonal stats on PVout raster over `geom` (buildable area). Store `pvout_mean` (float, kWh/kWp/yr).

#### Step 4 — Noise Receptors
Count buildings within 1 km of **full parcel boundary** — join to `developable_parcels_{slug}` in SQL:
```sql
FROM analysis.parcels_phase4_{slug} p
JOIN analysis.developable_parcels_{slug} dp ON dp.parcel_id = p.parcel_id
LEFT JOIN cadastre.buildings_{slug} b
  ON ST_DWithin(dp.geom, b.geom, 1000)   -- dp.geom = full parcel, EPSG:5321
```
Store `noise_count_1km` (integer, nullable). NULL = no buildings table.

#### Step 5 — Compactness (Polsby-Popper)
Compute on `geom` (buildable area) in EPSG:5321. Store `pp_ratio` (float, 0–1).

#### Step 6 — CA Authority Overlap
Overlap ratio between `geom` (buildable) and CA regulated area. Store `ca_overlap_ratio` (float, 0–1).

#### Step 7 — Environmental Flags (ECI)
7 binary flags via spatial join to **full parcel boundary** — join to `developable_parcels_{slug}`:
```sql
FROM analysis.parcels_phase4_{slug} p
JOIN analysis.developable_parcels_{slug} dp ON dp.parcel_id = p.parcel_id
JOIN rea.wetland w ON ST_Intersects(dp.geom, w.geom)
```
`flag_ca_regulated` derived from `ca_overlap_ratio > 0`. Compute `eci_raw` weighted sum (max 16).

#### Step 8 — SARO Species Assessment
Same pattern — join to `developable_parcels_{slug}` for full parcel boundary.
Store `end_present`, `end_species`, `thr_count`, `sc_count`, `saro_raw`.

#### Step 9 — Triage Classification
RED / AMBER / GREEN logic. PSW/ANSI and NHS checks use `geom` (buildable area) directly.
Store `triage_class`, `triage_flags`, `data_completeness`.

#### Step 10 — Verification & Map
Join `parcels_phase4_{slug}` scalars to `developable_parcels_{slug}` geometry for map display.
Folium map colored by `triage_class`. Export → `map_phase4_{slug}.html`.

### Output Table: `analysis.parcels_phase4_{slug}`

Single geometry column (`geom` = buildable area in EPSG:5321). All scalar parcel
metadata (address, UID, parcel_acre) remains in `developable_parcels_{slug}` and is
joined at query time (Step 10, Phase 5).

| Field | Type | Description |
|---|---|---|
| `parcel_id` | integer | Primary key (matches `developable_parcels_{slug}`) |
| `geom` | geometry | Buildable area polygon (EPSG:5321) — unioned from fragments |
| `buildable_area_ac` | float | Buildable area in acres |
| `grid_mw` | float | Grid capacity at nearest feeder (MW) |
| `feeder_voltage` | float | Feeder voltage (kV) |
| `slope_mean` | float | Mean slope over buildable area (%) |
| `pvout_mean` | float | Mean PVout over buildable area (kWh/kWp/yr) |
| `noise_count_1km` | integer | Building count within 1 km of parcel boundary (nullable) |
| `pp_ratio` | float | Polsby-Popper compactness ratio (0–1) |
| `ca_overlap_ratio` | float | CA regulated area / buildable area (0–1) |
| `flag_wetland` .. `flag_cli` | integer | 0/1 binary env flags |
| `env_flags` | text | All flags as JSON |
| `eci_raw` | integer | Weighted sum of env flags (0–16) |
| `end_present` | boolean | TRUE if any END species in parcel |
| `end_species` | text | JSON list of END species names |
| `thr_count` | integer | Unique THR species count |
| `sc_count` | integer | Unique SC species count |
| `saro_raw` | integer | THR×2 + SC×1 |
| `triage_class` | text | GREEN / AMBER / RED |
| `triage_flags` | text | JSON list of triggered conditions |
| `data_completeness` | text | PROVINCIAL_ONLY / PROVINCIAL_COUNTY / FULL_THREE_TIER |

> **Note**: `solar_parcel_uid`, `main_address`, `parcel_acre`, `parcel_geom` remain in
> `developable_parcels_{slug}`. Join on `parcel_id` when needed.

---

## Phase 5 — Scoring, Triage Ranking & Final Output

### File: `phase5_scoring_ranking.ipynb`

> **Status**: Not yet created.

### Purpose
Apply the v3 scoring model to all parcels. Assign scores, ranks, and triage tiers. Generate all deliverables: HTML dashboard, per-parcel HTML reports, and scores CSV.

### Inputs
- `analysis.parcels_phase4_{county_slug}` — all computed attributes from Phase 4
- `laurent_municipal_scores_{slug}.csv` — manual input: `parcel_id`, `muni_raw` (0–10)
- `analysis.oem_county_{county_slug}` — grid attributes (already joined in Phase 4, but available for reference)

### Config
```python
COUNTY_NAME = "Ottawa"
COUNTY_SLUG = COUNTY_NAME.lower().replace(" ", "_")
PG_CONN     = ...
# County overrides — set per county:
GREENBELT_NCC_PARCEL_IDS = [...]   # Ottawa only: parcel_ids where muni_score forced to 0
VOLTAGE_BONUS_KV         = 27.6   # Ottawa: feeders at this voltage earn +5 pts
```

---

### Scoring Model v3 — 100 pts

| Criterion | Pts | Method |
|---|---|---|
| Buildable Area | 30 | Min-max on `buildable_area_ac`, county range → 0–30 |
| Grid Capacity | 5 | `buildable_mw = buildable_area_ac / 5` · `ratio = min(buildable_mw / grid_mw, 1.0)` · `score = ratio × 5` |
| Voltage Bonus | 5 | `feeder_voltage == VOLTAGE_BONUS_KV` → 5 pts, else 0 |
| Irradiance | 5 | Min-max on `pvout_mean`, county range → 0–5 |
| SARO | 15 | `base = 15 − saro_raw`, floor 0. If `end_present`: apply additional 5pt penalty (floor 0) and confirm `triage_class = RED` |
| Noise Receptors | 10 | `noise_count_1km IS NULL OR noise_count_1km == 0` → 10 pts (MAX). Else: inverted min-max on `noise_count_1km`, county range → 0–10 |
| Environmental Risk | 15 | `score = (1 − eci_raw / 16) × 15`, floor 0 |
| Slope | 5 | `≤4% → 5` · `4–8% → linear 5→2` · `8–12% → 1` · `>12% → 0` |
| Compactness | 5 | Min-max on `pp_ratio`, county range → 0–5 |
| Municipal Politics | 5 | `muni_score = (muni_raw / 10) × 5` (from Laurent CSV) |

> **Normalization note**: All min-max normalizations are calibrated to the current county dataset range. Cross-county comparability requires rescoring on the pooled range — deferred to the 4-county calibration phase.

---

### Steps

#### Step 0 — Configuration & Data Load
Load `parcels_phase4_{slug}` and merge Laurent municipal scores CSV on `parcel_id`.
Apply county overrides:
- Ottawa: set `muni_raw = 0` for all `parcel_id` in `GREENBELT_NCC_PARCEL_IDS`
- Ottawa: apply `VOLTAGE_BONUS_KV` check for voltage bonus

#### Step 1 — Grid Capacity Score
```python
df["buildable_mw"]   = df["buildable_area_ac"] / 5.0
df["grid_ratio"]     = (df["buildable_mw"] / df["grid_mw"]).clip(upper=1.0)
df["score_grid"]     = df["grid_ratio"] * 5
```

#### Step 2 — Buildable Area Score
Min-max normalize `buildable_area_ac` across the county dataset → scale to 0–30.

#### Step 3 — Voltage Bonus Score
```python
df["score_voltage"] = (df["feeder_voltage"] == VOLTAGE_BONUS_KV).astype(int) * 5
```

#### Step 4 — Irradiance Score
Min-max normalize `pvout_mean` across county → scale to 0–5.

#### Step 5 — SARO Score
```python
df["score_saro"] = (15 - df["saro_raw"]).clip(lower=0)
# END penalty: additional 5pt deduction, floor 0
df.loc[df["end_present"], "score_saro"] = (df.loc[df["end_present"], "score_saro"] - 5).clip(lower=0)
# Confirm triage class for END parcels
df.loc[df["end_present"], "triage_class"] = "RED"
```

#### Step 6 — Noise Receptor Score
```python
# null or zero receptors = MAX score
mask_max = df["noise_count_1km"].isna() | (df["noise_count_1km"] == 0)
df.loc[mask_max, "score_noise"] = 10
# Inverted min-max for all others
# (more buildings = lower score)
others = ~mask_max
mn = df.loc[others, "noise_count_1km"].min()
mx = df.loc[others, "noise_count_1km"].max()
df.loc[others, "score_noise"] = (1 - (df.loc[others, "noise_count_1km"] - mn) / (mx - mn)) * 10
```

#### Step 7 — Environmental Risk Score (ECI)
```python
df["score_env"] = ((1 - df["eci_raw"] / 16) * 15).clip(lower=0)
```

#### Step 8 — Slope Score
```python
def slope_score(s):
    if s <= 4:   return 5
    if s <= 8:   return 5 - (s - 4) / (8 - 4) * 3   # linear 5→2
    if s <= 12:  return 1
    return 0
df["score_slope"] = df["slope_mean"].apply(slope_score)
```

#### Step 9 — Compactness Score
Min-max normalize `pp_ratio` across county → scale to 0–5.

#### Step 10 — Municipal Politics Score
```python
df["score_muni"] = (df["muni_raw"] / 10) * 5
```

#### Step 11 — Composite Score & Ranking
```python
score_cols = [
    "score_buildable", "score_grid", "score_voltage", "score_irradiance",
    "score_saro", "score_noise", "score_env", "score_slope",
    "score_compactness", "score_muni"
]
df["total_score"] = df[score_cols].sum(axis=1)

# RED parcels: nullify score and rank
df.loc[df["triage_class"] == "RED", "total_score"] = None

# Rank within triage groups
df["rank"] = None
for triage in ["GREEN", "AMBER"]:
    mask = df["triage_class"] == triage
    df.loc[mask, "rank"] = (
        df.loc[mask, "total_score"]
        .rank(ascending=False, method="min")
        .astype(int)
    )
```

#### Step 12 — Triage Rationale Notes
Generate human-readable `triage_rationale` text field for each parcel:
- **GREEN**: "No environmental or planning flags triggered. Parcel qualifies for full scoring."
- **AMBER**: List each triggered flag with a one-line description and remediation difficulty (Low / Medium / High).
- **RED**: List all hard disqualifying conditions. Note that no score is assigned.

Also populate `remediation_rank` (integer) for AMBER parcels — ordered by ease of resolution (lower = easier to remediate).

#### Step 13 — Save Outputs
```python
# All parcels (including RED with score=NULL)
save_to_postgis(df, f"parcels_scored_{slug}", "Phase 5 scored parcels")

# Final portfolio: GREEN + AMBER ranked; RED with NON-VIABLE flag
df_final = df.copy()
df_final.loc[df_final["triage_class"] == "RED", "rank_label"] = "NON-VIABLE"
save_to_postgis(df_final, f"parcels_final_{slug}", "Phase 5 final portfolio")
```

#### Step 14 — Export Scores CSV
Export `parcel_id` + all 10 criterion scores + `total_score` + `triage_class` + `rank`.
**Do not include raw measurement columns** (slope_mean, pvout_mean, noise_count_1km, etc.).
```python
export_cols = ["parcel_id", "solar_parcel_uid", "main_address", "triage_class", "rank",
               "total_score"] + score_cols
df[export_cols].to_csv(f"scores_{slug}.csv", index=False)
```

#### Step 15 — HTML Ranking Dashboard
Generate `ranking_dashboard_{slug}.html` using the forest-green IBM Plex Mono/Sans design system.
Include:
- Summary stats: total parcels, GREEN / AMBER / RED counts, top score
- Scoring model table (all 10 criteria, pts, method)
- Ranked table: GREEN parcels first, then AMBER, then RED (NON-VIABLE)
- Per-row: parcel_id, address, triage badge, rank, total_score, all criterion scores

#### Step 16 — Per-Parcel HTML Reports
Generate one self-contained HTML report per GREEN and AMBER parcel.
File naming: `parcel_{parcel_id}_report.html`
Each report includes:
- Parcel header: ID, address, county, triage class badge, total score, rank
- Score breakdown: all 10 criteria with score, max pts, bar visualization
- Triage rationale note (from Step 12)
- Key metrics: buildable area, grid capacity, grid ratio, feeder voltage, CA overlap ratio
- Environmental summary: ECI raw, triggered flags, SARO species list
- Data completeness flag

> **Design system**: Forest-green (#1a3c2d / #2e6b47), IBM Plex Mono for labels and code, IBM Plex Sans for body. No external dependencies — all CSS inline, base64-encode any map images.

---

## Analysis Schema Tables (per county)

| Phase | Table | Description |
|---|---|---|
| 1 | `aoi_county_{slug}` | Dissolved county AOI + buffer |
| 1 | `oem_county_{slug}` | OEM grid clipped to AOI, > 10 MW |
| 1 | `selected_lots_{slug}` | Lots intersecting OEM (aggregated attributes) |
| 2 | `rea_exclusions_{slug}` | Dissolved REA environmental exclusion geometry |
| 2 | `lots_developable_{slug}` | Lots after REA erase (net area + pct_remaining) |
| 3 | `selected_lots_boundary_{slug}` | Original lot boundaries via ST_PointOnSurface |
| 3 | `selected_parcels_{slug}` | Parcels within lot boundaries (≥ MIN_PARCEL_ACRES) |
| 3 | `building_exclusion_{slug}` | Dissolved conditional building buffers (150 m REA / 30 m default) |
| 3 | `developable_area_parcels_{slug}` | Final: dump → land-use → building erase → parcel intersection |
| 3 | `developable_parcels_{slug}` | Clean parcel outlines (+ geocoding columns after enrichment) |
| 4 | `parcels_phase4_{slug}` | Parcels with all physical + environmental attributes, triage class |
| 5 | `parcels_scored_{slug}` | Parcels with all 10 criterion scores, total score, rank, rationale notes |
| 5 | `parcels_final_{slug}` | Final portfolio: GREEN + AMBER ranked; RED with NON-VIABLE flag |

---

## Implementation Checklist

- [x] Phase 1 fixes (1–5) applied to `phase1_solar_siting_v2.ipynb`
- [x] Phase 1 verified for Bruce, Ottawa, Oxford, Peterborough
- [x] Phase 2 created (`phase2_solar_siting.ipynb`) — REA exclusions + lot subtraction
- [x] Phase 3 created (`phase3_lot_parcels.ipynb`) — lot recovery, land use, buildings, parcels
- [x] Geocoding created (`geocoding_parcels.ipynb`) — Mapbox reverse-geocode + UIDs
- [x] Create `phase4_physical_suitability.ipynb` — per spec above
- [ ] Create `phase5_scoring_ranking.ipynb` — per spec above
- [ ] Run full pipeline for all target counties (PTB, OTT, OXF, BRC/SIM)
