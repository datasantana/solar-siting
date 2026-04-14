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
| `environment` | `prov_trk_species` | `common_nam`, `saro_statu`, `geom` |
| `environment` | `significant_ecological_area` | `subtype`, `geom` — includes PSW, ANSI |
| `environment` | `nhs_area` | `area_name`, `geom` |
| `environment` | `wildlife_activity_area` | `wildlife_activity_type`, `geom` |
| `environment` | `wildlife_activity_site` | `wildlife_activity_type`, `geom` |
| `environment` | `habitat_planning_range` | `subtype`, `geom` |
| `environment` | `flood_hazard` | `geom` — optional per county; if absent, `flag_flood = 0` |
| `environment` | `cli` | `cli_sys`, `geom` — Canadian Land Inventory |

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

## Phase 4 — Physical & Environmental Suitability ✅

### File: `phase4_physical_suitability.ipynb`

> **Status**: v3 refactoring complete. All 10 steps implemented with v3 schema — needs:
> identity enrichment (Steps 2a/2b), OEM grid summary (Step 3), descriptive env labels
> (Step 5), expanded raster stats (min/max), flood flag, ECI update (max 19),
> column renames (`buildable_area_ac`→`ba_net_developable_acre`, `grid_mw`→`ba_grid_capacity_mw`,
> `feeder_voltage`→`ba_voltage_3ph`, `pp_ratio`→`slopecompactness_raw`,
> `noise_count_1km`→`nr_noise_receptors`), and `trk_species_list`/`trk_species_summary` text blocks.

### Architecture

**Single buildable geometry**: `parcels_phase4_{slug}` carries `geom` = unioned buildable area
(EPSG:5321). The full parcel boundary (`parcel_geom`) is never stored here — it is joined from
`developable_parcels_{slug}` on `parcel_id` wherever needed (Steps 2, 3b, 4, 5a, 5b, 8).

**Incremental enrichment**: all columns added after Step 1 use `update_parcels_phase4()` —
SQL `ALTER TABLE` + `UPDATE` via a temp table. The geometry column (`geom`) is never touched
after Step 1.

**Output target**: `parcels_phase4_{slug}` must carry all Groups 1–6 plus triage columns, so
that Phase 5 can score and export the final GeoJSON from a single JOIN to
`developable_parcels_{slug}` for parcel boundary geometry. No scalar data may remain only in
`developable_parcels_{slug}` after Phase 4.

### Purpose
Compute all parcel-level physical, environmental, and suitability attributes required as inputs
to Phase 5 scoring, triage, and GeoJSON export. Unit of analysis: **parcels**.

### Inputs
- `analysis.developable_area_parcels_{slug}` — buildable geometry fragments (≥1 per parcel)
- `analysis.developable_parcels_{slug}` — full parcel boundaries + geocoding columns
- `analysis.oem_county_{slug}` — qualifying feeders (>10 MW) clipped to AOI
- `cadastre.parcels_{slug}` — OGF IDs, assessment roll numbers
- `cadastre.ontario_lots` — lot/concession/township identifiers
- `official_plans.land_use_{slug}` — land use designations + OP schedule metadata
- `environment.prov_trk_species` — SARO species ranges
- `environment.significant_ecological_area` — PSW, ANSI, significant woodland
- `environment.nhs_area`, `environment.wildlife_activity_area`,
  `environment.wildlife_activity_site`, `environment.habitat_planning_range`
- `environment.cli` — Canadian Land Inventory classes
- `rea.{CONSERVATION_AUTH}_regulated_area` — CA regulated area
- Raster: DEM (slope), PVout
- Optional: `environment.flood_hazard` — if present, contributes `flag_flood` to ECI

> **Source schema addition**: add `environment.flood_hazard` | `geom` to the Global Conventions
> source table. If the table does not exist for a county, set `flag_flood = 0` for all parcels.

### Config

```python
COUNTY_NAME       = "Ottawa"
COUNTY_SLUG       = COUNTY_NAME.lower().replace(" ", "_")
PG_CONN           = ...
CONSERVATION_AUTH = "rvca"          # orca | rvca | utrca — must match Phase 2
LANDUSE_COL       = "design_en"     # or "landusedesc" — varies per county
DEM_PATH          = "..."           # path to DEM GeoTIFF (EPSG:5321, slope in %)
PVOUT_PATH        = "..."           # path to PVout GeoTIFF (EPSG:5321, kWh/kWp/day)
FLOOD_TABLE       = "environment.flood_hazard"   # set None if not available for county
# Ottawa-specific OP schedule fields (set None for counties without equivalent):
OP_TITLE_COL      = "schedule_title"    # column in land_use_{slug}
OP_B9_COL         = "sched_b9_rural"    # boolean column (Ottawa Schedule B9)
```

---

### Steps

#### Step 1 — Create Phase 4 Base Table

Union buildable fragments per parcel from `developable_area_parcels_{slug}`. Carry the best
feeder attributes (highest `grid_capacity_mw`). Store `fid` as row number for export ordering.

```sql
DROP TABLE IF EXISTS analysis.parcels_phase4_{slug};

CREATE TABLE analysis.parcels_phase4_{slug} AS
WITH buildable AS (
    SELECT
        parcel_id,
        ST_MakeValid(ST_Union(geom))                                        AS geom,
        ROUND(
            (ST_Area(ST_MakeValid(ST_Union(geom))) / 4046.856)::numeric, 3
        )                                                                   AS ba_net_developable_acre,
        -- Best feeder = highest capacity among all intersecting feeders
        (array_agg(grid_capacity_mw  ORDER BY grid_capacity_mw::float DESC NULLS LAST))[1]
                                                                            AS ba_grid_capacity_mw,
        (array_agg(voltage_3ph       ORDER BY grid_capacity_mw::float DESC NULLS LAST))[1]
                                                                            AS ba_voltage_3ph,
        (array_agg(feeder_name       ORDER BY grid_capacity_mw::float DESC NULLS LAST))[1]
                                                                            AS ba_feeder_name,
        (array_agg(station_name      ORDER BY grid_capacity_mw::float DESC NULLS LAST))[1]
                                                                            AS ba_station_name,
        (array_agg(ldc_name          ORDER BY grid_capacity_mw::float DESC NULLS LAST))[1]
                                                                            AS ba_ldc_name
    FROM analysis.developable_area_parcels_{slug}
    GROUP BY parcel_id
)
SELECT
    ROW_NUMBER() OVER (ORDER BY b.parcel_id)  AS fid,
    b.parcel_id,
    b.geom,                 -- buildable area geometry, EPSG:5321
    b.ba_net_developable_acre,
    b.ba_grid_capacity_mw,  -- stored as text (Group 2 spec)
    b.ba_voltage_3ph,       -- stored as text (Group 2 spec)
    b.ba_feeder_name,
    b.ba_station_name,
    b.ba_ldc_name
FROM buildable b;

CREATE INDEX ON analysis.parcels_phase4_{slug} USING GIST (geom);
CREATE INDEX ON analysis.parcels_phase4_{slug} (parcel_id);
```

Save → `analysis.parcels_phase4_{slug}`.

---

#### Step 2 — Enrich Group 1 and Group 2 Identity Fields

Two separate point-on-surface lookups: parcel centroid for Group 1 fields, buildable area
centroid for the `ba_` fields (they may differ when parcel and buildable area span different lots).

```sql
-- Step 2a: parcel-level identity (Group 1)
-- Uses ST_PointOnSurface(parcel boundary) from developable_parcels_{slug}

UPDATE analysis.parcels_phase4_{slug} p4
SET
    parcel_acre             = sub.parcel_acre,
    lot_ident               = sub.lot_ident,
    concession_ident        = sub.concession_ident,
    geographic_township_name= sub.geographic_township_name,
    land_use_designation    = sub.land_use_designation,
    station_name            = sub.ba_station_name,  -- legacy Group 1: best feeder station
    ldc_name                = sub.ba_ldc_name,      -- legacy Group 1: best feeder LDC
    lat                     = sub.lat,
    lon                     = sub.lon,
    solar_parcel_uid        = sub.solar_parcel_uid,
    main_address            = sub.main_address,
    secondary_addresses     = sub.secondary_addresses,
    county                  = sub.county,
    township                = sub.township,
    locality                = sub.locality,
    province                = sub.province
FROM (
    SELECT
        dp.parcel_id,
        ROUND((ST_Area(dp.geom) / 4046.856)::numeric, 3)          AS parcel_acre,
        ol.lot_ident,
        ol.concession_ident,
        ol.geographic_township_name,
        lup.{LANDUSE_COL}                                          AS land_use_designation,
        dp.solar_parcel_uid,
        dp.lat, dp.lon, dp.main_address, dp.secondary_addresses,
        dp.county, dp.township, dp.locality, dp.province
    FROM analysis.developable_parcels_{slug} dp
    LEFT JOIN cadastre.ontario_lots ol
        ON ST_Contains(ol.geom, ST_PointOnSurface(dp.geom))
    LEFT JOIN official_plans.land_use_{slug} lup
        ON ST_Contains(lup.geom, ST_PointOnSurface(dp.geom))
) sub
WHERE p4.parcel_id = sub.parcel_id;
```

> **ALTER TABLE** — add all Group 1 columns before this UPDATE via `update_parcels_phase4()`
> (parcel_acre float, lot_ident text, concession_ident text, geographic_township_name text,
> land_use_designation text, station_name text, ldc_name text, lat float, lon float,
> solar_parcel_uid text, main_address text, secondary_addresses jsonb, county text,
> township text, locality text, province text).

```sql
-- Step 2b: buildable-centroid identity (Group 2 ba_ fields)
-- Uses ST_PointOnSurface(buildable geom = p4.geom)

UPDATE analysis.parcels_phase4_{slug} p4
SET
    ba_lot_ident               = sub.lot_ident,
    ba_concession_ident        = sub.concession_ident,
    ba_geographic_township_name= sub.geographic_township_name,
    ba_land_use_designation    = sub.land_use_designation,
    ba_op_title                = sub.op_title,
    ba_sched_b9_rural          = sub.sched_b9_rural,
    ba_parcel_id               = p4.parcel_id,
    ba_parcel_ogf_id           = sub.ogf_id,
    ba_assessment_roll_number  = sub.assessment_roll_number
FROM (
    SELECT
        p.parcel_id,
        ol.lot_ident,
        ol.concession_ident,
        ol.geographic_township_name,
        lup.{LANDUSE_COL}                                          AS land_use_designation,
        lup.{OP_TITLE_COL}                                         AS op_title,       -- NULL if not Ottawa
        CASE WHEN lup.{OP_B9_COL} THEN 'Yes'
             WHEN lup.{OP_B9_COL} IS NULL THEN NULL
             ELSE 'No' END                                         AS sched_b9_rural, -- NULL if not Ottawa
        cp.ogf_id,
        cp.assessment_roll_number
    FROM analysis.parcels_phase4_{slug} p
    LEFT JOIN cadastre.ontario_lots ol
        ON ST_Contains(ol.geom, ST_PointOnSurface(p.geom))
    LEFT JOIN official_plans.land_use_{slug} lup
        ON ST_Contains(lup.geom, ST_PointOnSurface(p.geom))
    LEFT JOIN cadastre.parcels_{slug} cp
        ON cp.parcel_id = p.parcel_id
) sub
WHERE p4.parcel_id = sub.parcel_id;
```

> **ALTER TABLE** — add ba_lot_ident text, ba_concession_ident text,
> ba_geographic_township_name text, ba_land_use_designation text, ba_op_title text,
> ba_sched_b9_rural text, ba_parcel_id int, ba_parcel_ogf_id int,
> ba_assessment_roll_number text.

---

#### Step 3 — OEM Grid Summary (Group 3)

Build a multi-feeder text block from `oem_county_{slug}`. The block lists **all feeders
intersecting the full parcel boundary**, ordered by capacity descending. Best-feeder scalars
(`oem_max_capacity`, `oem_max_capacity_voltage`) mirror `ba_grid_capacity_mw` and
`ba_voltage_3ph` as numeric types.

```sql
UPDATE analysis.parcels_phase4_{slug} p4
SET
    oem_grid_summary       = sub.grid_summary,
    oem_max_capacity       = sub.max_capacity,
    oem_max_capacity_voltage = sub.max_voltage
FROM (
    SELECT
        dp.parcel_id,
        STRING_AGG(
            'Capacity: '     || oem.capacity::text
            || E'\nConfiguration: ' || COALESCE(oem.configuration, 'Unknown')
            || E'\nVoltage 3ph: '   || oem.voltage_3ph::text
            || E'\nFeeder name: '   || oem.feeder_name
            || E'\nLDC name: '      || oem.ldc_name
            || E'\nStation name: '  || oem.station_name,
            E'\n---\n'
            ORDER BY oem.capacity DESC
        )                                                          AS grid_summary,
        MAX(oem.capacity)                                          AS max_capacity,
        (array_agg(oem.voltage_3ph
                   ORDER BY oem.capacity DESC))[1]                 AS max_voltage
    FROM analysis.developable_parcels_{slug} dp
    JOIN analysis.oem_county_{slug} oem
        ON ST_Intersects(dp.geom, oem.geom)
    GROUP BY dp.parcel_id
) sub
WHERE p4.parcel_id = sub.parcel_id;
```

> **ALTER TABLE** — add oem_grid_summary text, oem_max_capacity float,
> oem_max_capacity_voltage float.
>
> **Note on `configuration`**: if `oem_county_{slug}` does not have a `configuration` column,
> derive it as `CASE WHEN oem.voltage_3ph >= 27.6 THEN 'Overhead' ELSE 'Underground'
> END` or check for a `feeder_config` / `circuit_type` column added during Phase 1 enrichment.

---

#### Step 4 — SARO Species Assessment (Group 4)

Build descriptive text blocks from `environment.prov_trk_species`. Species are deduplicated by
`species_name` within each parcel before counting.

```sql
UPDATE analysis.parcels_phase4_{slug} p4
SET
    trk_species_list    = sub.species_list,
    trk_species_summary = sub.species_summary,
    end_present         = sub.end_count > 0,
    end_species         = sub.end_names,
    thr_count           = sub.thr_count,
    sc_count            = sub.sc_count,
    saro_raw            = sub.thr_count * 2 + sub.sc_count * 1
FROM (
    WITH distinct_spp AS (
        -- deduplicate by parcel + species name
        SELECT DISTINCT dp.parcel_id, ts.common_nam AS common_name, ts.saro_statu AS saro_status
        FROM analysis.developable_parcels_{slug} dp
        JOIN environment.prov_trk_species ts
            ON ST_Intersects(dp.geom, ts.geom)
    )
    SELECT
        parcel_id,
        -- trk_species_list: literal \n within block, actual newline+---+newline between blocks
        STRING_AGG(
            'Common name: ' || common_name || E'\\nSARO Status: ' || saro_status,
            E'\n---\n'
            ORDER BY saro_status, common_name
        )                                                          AS species_list,
        -- trk_species_summary: only include status lines where count > 0
        NULLIF(
            TRIM(
                CASE WHEN COUNT(*) FILTER (WHERE saro_status='END') > 0
                     THEN 'Endangered (END): '
                          || COUNT(*) FILTER (WHERE saro_status='END')::text || E'\n'
                     ELSE '' END
              || CASE WHEN COUNT(*) FILTER (WHERE saro_status='THR') > 0
                     THEN 'Threatened (THR): '
                          || COUNT(*) FILTER (WHERE saro_status='THR')::text || E'\n'
                     ELSE '' END
              || CASE WHEN COUNT(*) FILTER (WHERE saro_status='SC')  > 0
                     THEN 'Special Concern (SC): '
                          || COUNT(*) FILTER (WHERE saro_status='SC')::text
                     ELSE '' END
            ), ''
        )                                                          AS species_summary,
        COUNT(*) FILTER (WHERE saro_status='END')::int             AS end_count,
        COUNT(*) FILTER (WHERE saro_status='THR')::int             AS thr_count,
        COUNT(*) FILTER (WHERE saro_status='SC')::int              AS sc_count,
        STRING_AGG(common_name, ', ')
            FILTER (WHERE saro_status='END')                       AS end_names
    FROM distinct_spp
    GROUP BY parcel_id
) sub
WHERE p4.parcel_id = sub.parcel_id;
```

> **ALTER TABLE** — add trk_species_list text, trk_species_summary text, end_present boolean,
> end_species text, thr_count int, sc_count int, saro_raw int. Parcels with no species records
> have all SARO fields NULL (no species intersect). Set `saro_raw = 0` for NULL parcels before
> Phase 5.

---

#### Step 5 — Environmental Text Labels (Group 5)

Store env_* fields as **descriptive text labels** matching the reference schema exactly. Binary
`flag_*` columns are computed separately for ECI (Step 7) and are **not** exposed in the
output GeoJSON.

All joins use the **full parcel boundary** from `developable_parcels_{slug}`.

```sql
UPDATE analysis.parcels_phase4_{slug} p4
SET
    env_significant_ecological_area = sub.sea_label,
    env_habitat_planning_range      = sub.hpr_label,
    env_wildlife_activity_area      = sub.waa_label,
    env_wildlife_activity_site      = sub.was_label,
    env_nhs_area                    = sub.nhs_label,
    env_cli_class                   = sub.cli_label,
    -- binary flags for ECI (Step 7) — not in final GeoJSON properties
    flag_wetland                    = sub.flag_wetland,
    flag_woodland                   = sub.flag_woodland,
    flag_nhs                        = sub.flag_nhs,
    flag_waa                        = sub.flag_waa,
    flag_flood                      = sub.flag_flood
FROM (
    SELECT
        dp.parcel_id,
        -- Significant ecological area: return most severe type present, else "No..."
        COALESCE(
            MAX(CASE
                WHEN sea.subtype ILIKE '%Provincially Significant Wetland%' THEN 'Provincially Significant Wetland'
                WHEN sea.subtype ILIKE '%ANSI%'          THEN 'ANSI'
                WHEN sea.subtype ILIKE '%Wetland%'       THEN 'Significant Wetland'
                WHEN sea.subtype ILIKE '%Woodland%'      THEN 'Significant Woodland'
                ELSE sea.subtype
            END),
            'No Significant Ecological Area'
        )                                                          AS sea_label,
        COALESCE(MAX(hpr.subtype), 'No Habitat Planning Range')
                                                                   AS hpr_label,
        COALESCE(MAX(waa.wildlife_activity_type), 'No Wildlife Activity Area')       AS waa_label,
        COALESCE(MAX(was.wildlife_activity_type), 'No Wildlife Activity Site')       AS was_label,
        COALESCE(MAX(nhs.area_name), 'No NHS Area')              AS nhs_label,
        -- CLI: comma-join distinct classes present (e.g. "Class 1, Class 3")
        COALESCE(
            STRING_AGG(DISTINCT
                CASE
                    WHEN cli.cli_sys LIKE '1%' THEN 'Class 1'
                    WHEN cli.cli_sys LIKE '2%' THEN 'Class 2'
                    WHEN cli.cli_sys LIKE '3%' THEN 'Class 3'
                    WHEN cli.cli_sys LIKE '4%' THEN 'Class 4'
                    WHEN cli.cli_sys LIKE '5%' THEN 'Class 5'
                    ELSE 'Unclassified'
                END,
                ', ' ORDER BY 1
            ),
            'Unclassified'
        )                                                          AS cli_label,
        -- ECI binary flags
        MAX(CASE WHEN sea.subtype ILIKE '%Wetland%'  THEN 1 ELSE 0 END)  AS flag_wetland,
        MAX(CASE WHEN sea.subtype ILIKE '%Woodland%' THEN 1 ELSE 0 END)  AS flag_woodland,
        MAX(CASE WHEN nhs.geom IS NOT NULL        THEN 1 ELSE 0 END)  AS flag_nhs,
        MAX(CASE WHEN waa.geom IS NOT NULL        THEN 1 ELSE 0 END)  AS flag_waa,
        MAX(CASE WHEN fh.geom  IS NOT NULL        THEN 1 ELSE 0 END)  AS flag_flood
    FROM analysis.developable_parcels_{slug} dp
    LEFT JOIN environment.significant_ecological_area sea
        ON ST_Intersects(dp.geom, sea.geom)
    LEFT JOIN environment.habitat_planning_range hpr
        ON ST_Intersects(dp.geom, hpr.geom)
    LEFT JOIN environment.wildlife_activity_area waa
        ON ST_Intersects(dp.geom, waa.geom)
    LEFT JOIN environment.wildlife_activity_site was
        ON ST_Intersects(dp.geom, was.geom)
    LEFT JOIN environment.nhs_area nhs
        ON ST_Intersects(dp.geom, nhs.geom)
    LEFT JOIN environment.cli cli
        ON ST_Intersects(dp.geom, cli.geom)
    LEFT JOIN {FLOOD_TABLE} fh              -- skip JOIN if FLOOD_TABLE is None
        ON ST_Intersects(dp.geom, fh.geom)
    GROUP BY dp.parcel_id
) sub
WHERE p4.parcel_id = sub.parcel_id;
```

> **ALTER TABLE** — add env_significant_ecological_area text, env_habitat_planning_range text,
> env_wildlife_activity_area text, env_wildlife_activity_site text, env_nhs_area text,
> env_cli_class text, flag_wetland int, flag_woodland int, flag_nhs int, flag_waa int,
> flag_flood int. If `FLOOD_TABLE` is None, skip the JOIN and set `flag_flood = 0` via a
> subsequent UPDATE.

---

#### Step 6 — CA Authority Overlap Ratio

Overlap between buildable area (`p4.geom`) and CA regulated area. Stored as a float 0.0–1.0.

```sql
UPDATE analysis.parcels_phase4_{slug} p4
SET ca_overlap_ratio = ROUND(
    COALESCE(
        ST_Area(ST_Intersection(p4.geom, ca.geom)) / NULLIF(ST_Area(p4.geom), 0),
        0.0
    )::numeric, 4)
FROM (
    SELECT
        p.parcel_id,
        ST_Union(ca_r.geom) AS geom
    FROM analysis.parcels_phase4_{slug} p
    JOIN rea.{CONSERVATION_AUTH}_regulated_area ca_r
        ON ST_Intersects(p.geom, ca_r.geom)
    GROUP BY p.parcel_id
) ca
WHERE p4.parcel_id = ca.parcel_id;

-- Parcels with no CA intersection remain NULL → set to 0.0
UPDATE analysis.parcels_phase4_{slug}
SET ca_overlap_ratio = 0.0
WHERE ca_overlap_ratio IS NULL;
```

> **ALTER TABLE** — add ca_overlap_ratio float. Default 0.0 for parcels outside CA regulated area.

---

#### Step 7 — ECI Weighted Sum

ECI incorporates 8 flag components (max raw = 19). Requires Steps 4, 5, and 6 to be complete.

```sql
UPDATE analysis.parcels_phase4_{slug}
SET eci_raw = (
    COALESCE(flag_wetland,  0) * 3 +   -- env_significant_ecological_area contains "Wetland"
    COALESCE(flag_woodland, 0) * 3 +   -- env_significant_ecological_area contains "Woodland"
    COALESCE(flag_nhs,      0) * 2 +   -- env_nhs_area != "No NHS Area"
    -- flag_saro_prox: any SARO species present (thr_count + sc_count + end_present > 0)
    CASE WHEN COALESCE(thr_count,0) + COALESCE(sc_count,0) + CASE WHEN end_present THEN 1 ELSE 0 END > 0
         THEN 2 ELSE 0 END +
    COALESCE(flag_waa,      0) * 2 +   -- env_wildlife_activity_area != "No..."
    CASE WHEN ca_overlap_ratio > 0 THEN 3 ELSE 0 END +  -- flag_ca_regulated
    COALESCE(flag_flood,    0) * 3 +   -- flood hazard intersection
    -- flag_cli: Class 1 or Class 2 present
    CASE WHEN env_cli_class ILIKE '%Class 1%' OR env_cli_class ILIKE '%Class 2%'
         THEN 1 ELSE 0 END
);
```

> **ALTER TABLE** — add eci_raw int. Valid range 0–19. NULL only if prerequisite steps failed.

---

#### Step 8 — Raster Statistics (Group 6)

All raster fields are stored as **FLOAT** in PostGIS (fixing the v2 string defect).
Steps 8a–8d use Rasterio windowed reads and `update_parcels_phase4()`.

**Step 8a — PVout zonal statistics** (over buildable area, `p4.geom`):
```python
# Load buildable geometries in EPSG:5321
gdf = read_postgis(f"SELECT parcel_id, geom FROM analysis.parcels_phase4_{slug}", geom_col="geom")

import rasterio, numpy as np
from rasterio.mask import mask as rio_mask

results = []
with rasterio.open(PVOUT_PATH) as src:
    for _, row in gdf.iterrows():
        try:
            out_image, _ = rio_mask(src, [row.geom.__geo_interface__], crop=True, nodata=src.nodata)
            vals = out_image[out_image != src.nodata].flatten()
            vals = vals[~np.isnan(vals)]
            if len(vals) > 0:
                results.append({
                    "parcel_id":  row.parcel_id,
                    "pvout_mean": float(np.mean(vals)),
                    "pvout_min":  float(np.min(vals)),
                    "pvout_max":  float(np.max(vals)),
                })
        except Exception:
            results.append({"parcel_id": row.parcel_id,
                            "pvout_mean": None, "pvout_min": None, "pvout_max": None})

update_parcels_phase4(results, ["pvout_mean float", "pvout_min float", "pvout_max float"])
```

**Step 8b — Slope zonal statistics** (over buildable area):
Same pattern as 8a using DEM raster. Store `slope_mean float`, `slope_min float`,
`slope_max float`. NULL where no valid pixels overlap.

**Step 8c — Compactness (Polsby-Popper)** — computed in PostGIS on buildable area:
```sql
UPDATE analysis.parcels_phase4_{slug}
SET slopecompactness_raw = ROUND(
    (4 * PI() * ST_Area(geom)
     / NULLIF(ST_Perimeter(geom)^2, 0)
    )::numeric, 3
);
```
> Field name `slopecompactness_raw` is preserved for backwards compatibility with the HTML
> report template. In v3 the value is purely the Polsby-Popper ratio (0.0–1.0). NULL if
> geometry is degenerate (zero perimeter).

> **ALTER TABLE** — add pvout_mean float, pvout_min float, pvout_max float, slope_mean float,
> slope_min float, slope_max float, slopecompactness_raw float.

**Step 8d — Noise Receptors** (buildings within 1 km of **full parcel boundary**):
```sql
UPDATE analysis.parcels_phase4_{slug} p4
SET nr_noise_receptors = sub.cnt
FROM (
    SELECT dp.parcel_id, COUNT(b.geom) AS cnt
    FROM analysis.developable_parcels_{slug} dp
    LEFT JOIN cadastre.buildings_{slug} b
        ON ST_DWithin(dp.geom, b.geom, 1000)
    GROUP BY dp.parcel_id
) sub
WHERE p4.parcel_id = sub.parcel_id;
```
> **ALTER TABLE** — add nr_noise_receptors int. NULL if the `buildings_{slug}` table does not
> exist for the county (use `SELECT to_regclass(...)` check before running). A NULL value
> propagates to Phase 5 as the "data gap" AMBER triage flag.

---

#### Step 9 — Triage Classification

Triage is applied in Python after loading `parcels_phase4_{slug}` — env text labels and numeric
fields are all available at this point.

```python
import json, pandas as pd

df = read_postgis(
    f"SELECT * FROM analysis.parcels_phase4_{slug}", geom_col="geom"
).drop(columns="geom")   # scalar-only for triage logic

def classify_triage(row):
    flags = []
    triage = "GREEN"

    # ── RED conditions ──────────────────────────────────────────────────────
    if row.get("end_present"):
        flags.append("END_SPECIES"); triage = "RED"
    sea = row.get("env_significant_ecological_area", "") or ""
    if "Provincially Significant Wetland" in sea or "ANSI" in sea:
        flags.append("PSW_OR_ANSI"); triage = "RED"
    if (row.get("ca_overlap_ratio") or 0) >= 0.60:
        flags.append("CA_OVERLAP_GE60PCT"); triage = "RED"
    buildable_mw = (row.get("ba_net_developable_acre") or 0) / 5.0
    grid_mw  = row.get("oem_max_capacity") or 0
    grid_ratio = min(buildable_mw / grid_mw, 1.0) if grid_mw > 0 else 0.0
    if grid_ratio < 0.5:
        flags.append("GRID_RATIO_LT50PCT"); triage = "RED"

    if triage == "RED":
        return triage, json.dumps(flags)

    # ── AMBER conditions ────────────────────────────────────────────────────
    if 0 < (row.get("ca_overlap_ratio") or 0) < 0.60:
        flags.append("CA_PARTIAL_OVERLAP"); triage = "AMBER"
    if (row.get("thr_count") or 0) > 0 or (row.get("sc_count") or 0) > 0:
        flags.append("THR_OR_SC_SPECIES"); triage = "AMBER"
    if (row.get("env_nhs_area") or "") not in ("", "No NHS Area"):
        flags.append("NHS_AREA"); triage = "AMBER"
    slope = row.get("slope_mean")
    if slope is not None and slope > 8:
        flags.append("SLOPE_GT8PCT"); triage = "AMBER"
    if (row.get("nr_noise_receptors") or 0) > 8:
        flags.append("NOISE_GT8"); triage = "AMBER"
    if row.get("nr_noise_receptors") is None:
        flags.append("NOISE_DATA_GAP"); triage = "AMBER"
    if 0.5 <= grid_ratio <= 0.7:
        flags.append("GRID_RATIO_MARGINAL"); triage = "AMBER"

    return triage, json.dumps(flags)

df[["triage_class", "triage_flags"]] = df.apply(
    lambda r: pd.Series(classify_triage(r)), axis=1
)

# data_completeness: FULL_THREE_TIER if both county and OEM data present
df["data_completeness"] = df.apply(lambda r: (
    "FULL_THREE_TIER"
    if pd.notna(r.get("oem_max_capacity")) and pd.notna(r.get("pvout_mean"))
    else "PROVINCIAL_COUNTY"
    if pd.notna(r.get("pvout_mean"))
    else "PROVINCIAL_ONLY"
), axis=1)

update_parcels_phase4(
    df[["parcel_id", "triage_class", "triage_flags", "data_completeness"]].to_dict("records"),
    ["triage_class text", "triage_flags text", "data_completeness text"]
)
```

---

#### Step 10 — Verification & Map

```python
df = read_postgis(
    f"""
    SELECT p4.*, ST_Transform(dp.geom, 4326) AS parcel_geom
    FROM analysis.parcels_phase4_{slug} p4
    JOIN analysis.developable_parcels_{slug} dp USING (parcel_id)
    """, geom_col="parcel_geom")

print("=== Phase 4 Field Audit ===")
for col in df.columns:
    null_pct = df[col].isna().mean() * 100
    print(f"  {col:<40} {df[col].dtype!s:<12} null={null_pct:.0f}%")

print("\n=== Triage Counts ===")
print(df["triage_class"].value_counts())
print("\n=== ECI Range ===")
print(df["eci_raw"].describe())
print("\n=== CA Overlap ===")
print(df[df["ca_overlap_ratio"]>0]["ca_overlap_ratio"].describe())
```

Generate Folium map colored by `triage_class` (GREEN / AMBER / RED) → `map_phase4_{slug}.html`.

---

### Output Table: `analysis.parcels_phase4_{slug}`

Single geometry column (`geom` = buildable area, EPSG:5321). Full parcel boundary remains in
`developable_parcels_{slug}` and is joined in Phase 5 for GeoJSON export.

| Field | Type | Group | Notes |
|---|---|---|---|
| `fid` | integer | 1 | Row number, stable within county run |
| `parcel_id` | integer | 1 | Primary key |
| `parcel_acre` | float | 1 | Full parcel area in acres |
| `lot_ident` | text | 1 | e.g. "LOT 10" — parcel centroid lookup |
| `concession_ident` | text | 1 | e.g. "CON 10" |
| `geographic_township_name` | text | 1 | |
| `land_use_designation` | text | 1 | OP designation at parcel centroid |
| `station_name` | text | 1 | Legacy: best feeder station name |
| `ldc_name` | text | 1 | Legacy: best feeder LDC name |
| `lat` | float | 1 | WGS84 centroid latitude |
| `lon` | float | 1 | WGS84 centroid longitude |
| `solar_parcel_uid` | text | 1 | UUID-5 from geocoding |
| `main_address` | text | 1 | Mapbox reverse-geocode result |
| `secondary_addresses` | jsonb | 1 | Nullable |
| `county` | text | 1 | |
| `township` | text | 1 | |
| `locality` | text | 1 | Nullable |
| `province` | text | 1 | "Ontario" |
| `ba_lot_ident` | text | 2 | Buildable centroid lookup |
| `ba_concession_ident` | text | 2 | |
| `ba_geographic_township_name` | text | 2 | |
| `ba_grid_capacity_mw` | text | 2 | Best feeder capacity as string |
| `ba_voltage_3ph` | text | 2 | Best feeder voltage as string |
| `ba_station_name` | text | 2 | |
| `ba_feeder_name` | text | 2 | |
| `ba_ldc_name` | text | 2 | |
| `ba_land_use_designation` | text | 2 | OP designation at buildable centroid |
| `ba_op_title` | text | 2 | OP schedule title — nullable |
| `ba_sched_b9_rural` | text | 2 | "Yes"/"No"/NULL (Ottawa-specific) |
| `ba_parcel_id` | integer | 2 | = parcel_id |
| `ba_parcel_ogf_id` | integer | 2 | OGF parcel identifier |
| `ba_assessment_roll_number` | text | 2 | ARN |
| `ba_net_developable_acre` | float | 2 | Buildable area in acres |
| `oem_grid_summary` | text | 3 | Multi-feeder block, "---" separated |
| `oem_max_capacity` | float | 3 | Best feeder capacity (MW) |
| `oem_max_capacity_voltage` | float | 3 | Best feeder voltage (kV) |
| `trk_species_summary` | text | 4 | "END: N\nTHR: N\nSC: N" — only non-zero lines |
| `trk_species_list` | text | 4 | Species blocks, "---" separated |
| `end_present` | boolean | 4 | TRUE if any END species |
| `end_species` | text | 4 | CSV of END common names |
| `thr_count` | integer | 4 | Unique THR species count |
| `sc_count` | integer | 4 | Unique SC species count |
| `saro_raw` | integer | 4 | thr_count×2 + sc_count×1 |
| `env_significant_ecological_area` | text | 5 | Descriptive label |
| `env_habitat_planning_range` | text | 5 | |
| `env_wildlife_activity_area` | text | 5 | |
| `env_wildlife_activity_site` | text | 5 | |
| `env_nhs_area` | text | 5 | |
| `env_cli_class` | text | 5 | e.g. "Class 1, Class 3" or "Unclassified" |
| `pvout_mean` | float | 6 | Mean PVout over buildable area (kWh/kWp/day) |
| `pvout_min` | float | 6 | |
| `pvout_max` | float | 6 | |
| `nr_noise_receptors` | integer | 6 | Buildings within 1 km — nullable |
| `slope_mean` | float | 6 | Mean slope % over buildable area — nullable |
| `slope_min` | float | 6 | Nullable |
| `slope_max` | float | 6 | Nullable |
| `slopecompactness_raw` | float | 6 | Polsby-Popper ratio (0–1) — legacy name |
| `ca_overlap_ratio` | float | 6 | CA regulated / buildable area (0–1) |
| `eci_raw` | integer | 6 | Weighted env flag sum (0–19) |
| `flag_wetland` .. `flag_flood` | integer | internal | 0/1 — NOT exported to GeoJSON |
| `triage_class` | text | triage | GREEN / AMBER / RED |
| `triage_flags` | text | triage | JSON-serialised list |
| `data_completeness` | text | triage | PROVINCIAL_ONLY / PROVINCIAL_COUNTY / FULL_THREE_TIER |
| `geom` | geometry | — | Buildable area polygon (EPSG:5321) |

> **Flag columns** (`flag_wetland`, `flag_woodland`, `flag_nhs`, `flag_waa`, `flag_flood`,
> `end_present`, `end_species`, `thr_count`, `sc_count`, `saro_raw`) are retained in PostGIS for
> auditability but are **excluded** from the GeoJSON `properties` block (Phase 5 Step B SELECT
> list). `end_present` is used in Phase 5 SARO scoring; `thr_count`/`sc_count` drive AMBER triage.

---

## Phase 5 — Scoring, Triage Ranking & GeoJSON Export

### File: `phase5_scoring_ranking.ipynb`

> **Status**: Not yet created. Primary deliverable: `outputs/{county_slug}_parcels_v3.geojson`.

### Purpose
Apply the v3 100-point scoring model to all viable parcels. Assign scores, ranks, and triage
tiers. Produce the county GeoJSON as the canonical output consumed by the HTML report template
and ranking dashboard.

### Inputs
- `analysis.parcels_phase4_{slug}` — all physical + environmental attributes from Phase 4
- `analysis.developable_parcels_{slug}` — parcel boundary geometry for GeoJSON export
- `laurent_municipal_scores_{slug}.csv` — manual input: `parcel_id`, `muni_raw` (0–10 integer)

### Config

```python
COUNTY_NAME              = "Ottawa"
COUNTY_SLUG              = COUNTY_NAME.lower().replace(" ", "_")
PG_CONN                  = ...
ONTARIO_PVOUT_AVG        = 3.56          # Ontario mean PVout (kWh/kWp/day)
VOLTAGE_BONUS_KV         = 27.6          # feeders at this voltage earn +5 pts
GREENBELT_NCC_IDS        = [...]         # Ottawa: parcel_ids → force muni_raw = 0
CLUSTER_CONFLICTS        = [26841, 26521, 26876]   # Ottawa: JKGF4 feeder cluster
MUNI_SCORES_CSV          = f"laurent_municipal_scores_{COUNTY_SLUG}.csv"

# Output path
OUTPUT_GEOJSON = f"outputs/{COUNTY_SLUG}_parcels_v3.geojson"
```

---

### Step 0 — Configuration, Data Load & Min-Max Range Computation

Min-max ranges must be computed **before any per-parcel scoring**. They are stored as named
constants and applied consistently to all parcels in the county dataset.

```python
import pandas as pd, numpy as np, geopandas as gpd

# Load Phase 4 scalar data (no geometry needed for scoring)
df = pd.read_sql(
    f"SELECT * FROM analysis.parcels_phase4_{slug}", con=engine
)

# Merge Laurent municipal scores
muni_df = pd.read_csv(MUNI_SCORES_CSV, dtype={"parcel_id": int, "muni_raw": int})
df = df.merge(muni_df, on="parcel_id", how="left")
df["muni_raw"] = df["muni_raw"].fillna(5)   # default if missing from CSV

# Ottawa override: Greenbelt / NCC parcels → muni_raw = 0
df.loc[df["parcel_id"].isin(GREENBELT_NCC_IDS), "muni_raw"] = 0

# Fill SARO nulls (no species = no penalty)
df["saro_raw"]  = df["saro_raw"].fillna(0).astype(int)
df["thr_count"] = df["thr_count"].fillna(0).astype(int)
df["sc_count"]  = df["sc_count"].fillna(0).astype(int)
df["end_present"] = df["end_present"].fillna(False).astype(bool)

# ── County-level min-max ranges (computed ONCE, stored as constants) ──────────
area_min  = df["ba_net_developable_acre"].min()
area_max  = df["ba_net_developable_acre"].max()

pvout_min = df["pvout_mean"].min()
pvout_max = df["pvout_mean"].max()

# Noise: exclude nulls AND zeroes for range computation
noise_vals = df.loc[df["nr_noise_receptors"].notna() & (df["nr_noise_receptors"] > 0),
                    "nr_noise_receptors"]
noise_min  = noise_vals.min()
noise_max  = noise_vals.max()

compact_min = df["slopecompactness_raw"].min()
compact_max = df["slopecompactness_raw"].max()

# County irradiance labels for Group 8
irr_county_min = f"{pvout_min:.2f}"
irr_county_max = f"{pvout_max:.2f}"

print(f"  area     [{area_min:.1f}, {area_max:.1f}] ac")
print(f"  pvout    [{pvout_min:.4f}, {pvout_max:.4f}] kWh/kWp/day")
print(f"  noise    [{noise_min:.0f}, {noise_max:.0f}] buildings")
print(f"  compact  [{compact_min:.3f}, {compact_max:.3f}] PP ratio")
```

---

### Scoring Model v3 — 100 pts

| Criterion | Field | Max | Method |
|---|---|---|---|
| Buildable Area | `score_buildable_area` | 30 | Min-max(`ba_net_developable_acre`, county range) × 30 |
| Grid Capacity | `score_grid_capacity` | 5 | min(buildable_mw / oem_max_capacity, 1.0) × 5 |
| Voltage Bonus | `score_voltage_bonus` | 5 | `oem_max_capacity_voltage == VOLTAGE_BONUS_KV` → 5 else 0 |
| Irradiance | `score_irradiance` | 5 | Min-max(`pvout_mean`, county range) × 5 |
| SARO | `score_saro` | 15 | `max(0, 15 − saro_raw)`; if END: additional −5, floor 0 |
| Noise Receptors | `score_noise` | 10 | null or 0 → 10; else inverted min-max × 10 |
| Environmental Risk | `score_env_risk` | 15 | `round((1 − eci_raw / 19) × 15)`, floor 0 |
| Slope | `score_slope` | 5 | ≤4% → 5; 4–8% linear 5→2; 8–12% → 1; >12% → 0 |
| Compactness | `score_compactness` | 5 | Min-max(`slopecompactness_raw`, county range) × 5 |
| Municipal | `score_municipal` | 5 | (muni_raw / 10) × 5 |

> **Normalization**: all min-max ranges are calibrated to the current county dataset. Cross-county
> comparability requires rescoring on the pooled 4-county range — deferred to the calibration phase.

---

### Steps 1–10 — Criterion Scores

#### Step 1 — Buildable Area Score

```python
def minmax_score(series, vmin, vmax, max_pts):
    denom = vmax - vmin if vmax != vmin else 1.0
    return ((series - vmin) / denom * max_pts).clip(0, max_pts).round(0).astype(int)

df["score_buildable_area"] = minmax_score(df["ba_net_developable_acre"],
                                          area_min, area_max, 30)
```

#### Step 2 — Grid Capacity Score

```python
df["buildable_mw"] = df["ba_net_developable_acre"] / 5.0
df["grid_ratio"]   = (
    (df["buildable_mw"] / df["oem_max_capacity"].replace(0, np.nan))
    .clip(upper=1.0)
    .fillna(0.0)
)
df["score_grid_capacity"] = (df["grid_ratio"] * 5).round(0).astype(int)
```

#### Step 3 — Voltage Bonus

```python
df["score_voltage_bonus"] = (
    df["oem_max_capacity_voltage"].eq(VOLTAGE_BONUS_KV).astype(int) * 5
)
```

#### Step 4 — Irradiance Score

```python
df["score_irradiance"] = minmax_score(df["pvout_mean"], pvout_min, pvout_max, 5)
```

#### Step 5 — SARO Score

```python
df["score_saro"] = (15 - df["saro_raw"]).clip(lower=0)
# END penalty: additional −5 pt deduction after base score, floor 0
df.loc[df["end_present"], "score_saro"] = (
    df.loc[df["end_present"], "score_saro"] - 5
).clip(lower=0)
# Confirm triage class for END parcels (may have been set GREEN in Phase 4 if END-only condition)
df.loc[df["end_present"], "triage_class"] = "RED"
```

#### Step 6 — Noise Receptor Score

```python
mask_max = df["nr_noise_receptors"].isna() | (df["nr_noise_receptors"] == 0)
df["score_noise"] = 0
df.loc[mask_max, "score_noise"] = 10

mask_other = ~mask_max & df["nr_noise_receptors"].notna()
df.loc[mask_other, "score_noise"] = (
    (1 - (df.loc[mask_other, "nr_noise_receptors"] - noise_min)
         / (noise_max - noise_min))
    * 10
).clip(0, 10).round(0).astype(int)
```

#### Step 7 — Environmental Risk Score (ECI)

```python
# ECI max = 19 (v3)
df["score_env_risk"] = (
    ((1 - df["eci_raw"].fillna(0) / 19) * 15)
    .clip(lower=0)
    .round(0)
    .astype(int)
)
```

#### Step 8 — Slope Score

```python
def slope_score(s):
    if pd.isna(s):  return 0       # null slope → conservative zero
    if s <= 4:      return 5
    if s <= 8:      return max(2, round(5 - (s - 4) / (8 - 4) * 3))   # linear 5→2
    if s <= 12:     return 1
    return 0

df["score_slope"] = df["slope_mean"].apply(slope_score).astype(int)
```

#### Step 9 — Compactness Score

```python
df["score_compactness"] = minmax_score(
    df["slopecompactness_raw"].fillna(compact_min),
    compact_min, compact_max, 5
)
```

#### Step 10 — Municipal Politics Score

```python
df["score_municipal"] = ((df["muni_raw"] / 10) * 5).round(0).astype(int)
```

---

### Step 11 — Total Score, Triage Confirmation & Ranking

```python
SCORE_COLS = [
    "score_buildable_area", "score_grid_capacity", "score_voltage_bonus",
    "score_irradiance", "score_saro", "score_noise", "score_env_risk",
    "score_slope", "score_compactness", "score_municipal"
]

df["total_score"] = df[SCORE_COLS].sum(axis=1).astype(int)

# RED parcels: nullify total_score and rank
df.loc[df["triage_class"] == "RED", "total_score"] = None

# Rank within triage group (GREEN and AMBER separately; RED = null)
df["rank"] = None
for tier in ["GREEN", "AMBER"]:
    mask = df["triage_class"] == tier
    df.loc[mask, "rank"] = (
        df.loc[mask, "total_score"]
        .rank(ascending=False, method="min")
        .astype("Int64")
    )
```

---

### Step 12 — Group 8 Derived Fields

All Group 8 fields must be computed before export. Score and numeric fields are cast to int/float
as specified.

```python
# buildable_mw already computed in Step 2
# grid_ratio  already computed in Step 2

df["irradiance_vs_ontario_pct"] = (
    (df["pvout_mean"] / ONTARIO_PVOUT_AVG * 100).round(0).astype(int)
)

df["irradiance_pct_in_range"] = (
    ((df["pvout_mean"] - pvout_min) / (pvout_max - pvout_min) * 100)
    .round(0)
    .astype(int)
)

df["irr_county_min"]  = irr_county_min    # e.g. "3.69"
df["irr_county_max"]  = irr_county_max    # e.g. "3.83"
df["analyst_notes"]   = ""               # free text — default empty string
```

---

### Step 13 — Triage Rationale Notes

```python
def triage_rationale(row):
    flags = json.loads(row["triage_flags"] or "[]")
    triage = row["triage_class"]
    if triage == "GREEN":
        return "No environmental or planning flags triggered. Parcel qualifies for full scoring."
    flag_descriptions = {
        "END_SPECIES":         "Endangered species present — hard disqualifier",
        "PSW_OR_ANSI":         "Provincially Significant Wetland / ANSI intersects buildable area",
        "CA_OVERLAP_GE60PCT":  "≥60% of buildable area within CA regulated zone",
        "GRID_RATIO_LT50PCT":  "Project capacity < 50% of available feeder capacity",
        "CA_PARTIAL_OVERLAP":  "Partial CA regulated overlap (remediation possible)",
        "THR_OR_SC_SPECIES":   "Threatened or Special Concern species present",
        "NHS_AREA":            "Natural Heritage System designation intersects parcel",
        "SLOPE_GT8PCT":        "Mean slope > 8% — elevated civil cost risk",
        "NOISE_GT8":           "More than 8 noise receptors within 1 km",
        "NOISE_DATA_GAP":      "Building layer not available — noise score unverified",
        "GRID_RATIO_MARGINAL": "Grid ratio 50–70% — marginal interconnection headroom",
    }
    lines = [f"- {flag_descriptions.get(f, f)}" for f in flags]
    action = "Hard disqualifier — no score assigned." if triage == "RED" else \
             "Flagged for secondary review — scored within AMBER tier."
    return action + "\n" + "\n".join(lines)

df["analyst_notes"] = df.apply(triage_rationale, axis=1)
```

---

### Step A — Save `parcels_scored_{slug}` to PostGIS

Save all fields and all parcels (including RED with score = NULL). The geometry stored here is
the buildable area (from `parcels_phase4`), carried through `update_parcels_phase4()`.

```python
# Re-attach buildable geometry for PostGIS save
gdf_save = gpd.read_postgis(
    f"SELECT parcel_id, geom FROM analysis.parcels_phase4_{slug}",
    con=engine, geom_col="geom", crs=CRS_ONTARIO
)
df_scored = df.merge(gdf_save, on="parcel_id", how="left")
gdf_scored = gpd.GeoDataFrame(df_scored, geometry="geom", crs=CRS_ONTARIO)

save_to_postgis(gdf_scored, f"parcels_scored_{COUNTY_SLUG}", "Phase 5 scored parcels")
```

---

### Step B — Export GeoJSON

The GeoJSON uses **parcel boundary geometry** (from `developable_parcels_{slug}`), projected to
EPSG:4326. Properties must match the target v3 schema exactly — field names, types, and ordering.

**Columns must be cast to correct types before export** (scores → int, raster fields → float,
score strings inherited from Phase 4 → already float; `secondary_addresses` → list).

```python
# Ordered property list matching target v3 schema
PROPERTY_ORDER = [
    # Group 1
    "fid", "parcel_id", "parcel_acre", "lot_ident", "concession_ident",
    "geographic_township_name", "land_use_designation", "station_name", "ldc_name",
    "lat", "lon", "solar_parcel_uid", "main_address", "secondary_addresses",
    "county", "township", "locality", "province",
    # Group 2
    "ba_lot_ident", "ba_concession_ident", "ba_geographic_township_name",
    "ba_grid_capacity_mw", "ba_voltage_3ph", "ba_station_name", "ba_feeder_name",
    "ba_ldc_name", "ba_land_use_designation", "ba_op_title", "ba_sched_b9_rural",
    "ba_parcel_id", "ba_parcel_ogf_id", "ba_assessment_roll_number",
    "ba_net_developable_acre",
    # Group 3
    "oem_grid_summary", "oem_max_capacity", "oem_max_capacity_voltage",
    # Group 4
    "trk_species_summary", "trk_species_list",
    # Group 5
    "env_significant_ecological_area", "env_habitat_planning_range",
    "env_wildlife_activity_area", "env_wildlife_activity_site",
    "env_nhs_area", "env_cli_class",
    # Group 6
    "pvout_mean", "pvout_min", "pvout_max", "nr_noise_receptors",
    "slope_mean", "slope_min", "slope_max", "slopecompactness_raw",
    "ca_overlap_ratio", "eci_raw",
    # Group 7 (v3 scores — int)
    "score_buildable_area", "score_grid_capacity", "score_voltage_bonus",
    "score_irradiance", "score_saro", "score_noise", "score_env_risk",
    "score_slope", "score_compactness", "score_municipal",
    "total_score", "rank",
    # Group 8 (v3 derived — new)
    "triage_class", "triage_flags", "data_completeness",
    "buildable_mw", "grid_ratio",
    "irradiance_vs_ontario_pct", "irradiance_pct_in_range",
    "irr_county_min", "irr_county_max", "analyst_notes",
]

# Load scored scalars
df_export = pd.read_sql(
    f"SELECT * FROM analysis.parcels_scored_{COUNTY_SLUG}", con=engine
)

# Enforce integer types on all score fields (nullable int for RED parcels)
int_cols = [
    "score_buildable_area", "score_grid_capacity", "score_voltage_bonus",
    "score_irradiance", "score_saro", "score_noise", "score_env_risk",
    "score_slope", "score_compactness", "score_municipal",
    "total_score", "rank", "eci_raw", "nr_noise_receptors",
    "irradiance_vs_ontario_pct", "irradiance_pct_in_range",
]
for c in int_cols:
    if c in df_export.columns:
        df_export[c] = df_export[c].astype("Int64")   # nullable int

# secondary_addresses: JSONB → Python list
df_export["secondary_addresses"] = df_export["secondary_addresses"].apply(
    lambda v: v if isinstance(v, list) else (json.loads(v) if v else None)
)

# Join parcel boundary geometry (EPSG:4326)
gdf_boundary = gpd.read_postgis(
    f"""
    SELECT parcel_id,
           ST_Transform(geom, 4326) AS parcel_geom
    FROM analysis.developable_parcels_{COUNTY_SLUG}
    """,
    con=engine, geom_col="parcel_geom", crs=CRS_WGS84
)

gdf_export = gpd.GeoDataFrame(
    df_export.merge(gdf_boundary, on="parcel_id"),
    geometry="parcel_geom",
    crs=CRS_WGS84
)

# Re-number fid sequentially (stable within this export run)
gdf_export = gdf_export.sort_values("parcel_id").reset_index(drop=True)
gdf_export["fid"] = gdf_export.index + 1

# Apply column order (include only columns in PROPERTY_ORDER that exist)
export_cols = [c for c in PROPERTY_ORDER if c in gdf_export.columns]
gdf_export = gdf_export[export_cols + ["parcel_geom"]]

import os
os.makedirs("outputs", exist_ok=True)
gdf_export.to_file(OUTPUT_GEOJSON, driver="GeoJSON")
print(f"Exported {len(gdf_export)} features → {OUTPUT_GEOJSON}")
```

---

### Step C — Verification

```python
import json

with open(OUTPUT_GEOJSON) as f:
    gj = json.load(f)

props0 = gj["features"][0]["properties"]
geom0  = gj["features"][0]["geometry"]

print("=== Field Type Audit ===")
for k, v in props0.items():
    print(f"  {k:<40} {type(v).__name__}")

print("\n=== Geometry ===")
print(f"  Type: {geom0['type']}")
assert geom0["type"] in ("Polygon", "MultiPolygon"), "Geometry must be Polygon or MultiPolygon"

print("\n=== Score Range Check ===")
score_fields = [
    ("score_buildable_area", 0, 30), ("score_grid_capacity", 0, 5),
    ("score_voltage_bonus",  0,  5), ("score_irradiance",    0, 5),
    ("score_saro",           0, 15), ("score_noise",         0, 10),
    ("score_env_risk",       0, 15), ("score_slope",         0,  5),
    ("score_compactness",    0,  5), ("score_municipal",     0,  5),
    ("total_score",          0, 100),
]
for field, lo, hi in score_fields:
    vals = [f["properties"].get(field) for f in gj["features"]
            if f["properties"].get(field) is not None]
    if vals:
        actual_min, actual_max = min(vals), max(vals)
        ok = lo <= actual_min and actual_max <= hi
        print(f"  {field:<30} range=[{actual_min},{actual_max}]  expected=[{lo},{hi}]  {'✅' if ok else '❌'}")

print("\n=== Triage Group Counts ===")
from collections import Counter
counts = Counter(f["properties"]["triage_class"] for f in gj["features"])
print(f"  GREEN={counts.get('GREEN',0)}  AMBER={counts.get('AMBER',0)}  RED={counts.get('RED',0)}")

print("\n=== Null Check (required fields) ===")
required = ["parcel_id", "solar_parcel_uid", "triage_class", "eci_raw", "ba_net_developable_acre"]
for field in required:
    nulls = sum(1 for f in gj["features"] if f["properties"].get(field) is None)
    print(f"  {field:<40} nulls={nulls}  {'✅' if nulls == 0 else '❌'}")

print("\n=== Score Integer Type Check ===")
for field, _, _ in score_fields:
    non_int = [f["properties"][field] for f in gj["features"]
               if f["properties"].get(field) is not None
               and not isinstance(f["properties"][field], int)]
    status = "✅" if not non_int else f"❌ {len(non_int)} non-int values"
    print(f"  {field:<30} {status}")
```

---

### Step 14 — Export Scores CSV (optional)

```python
csv_cols = ["parcel_id", "solar_parcel_uid", "main_address", "triage_class", "rank",
            "total_score"] + SCORE_COLS
df_export[csv_cols].to_csv(f"scores_{COUNTY_SLUG}.csv", index=False)
```

### Step 15 — HTML Ranking Dashboard (optional)

Generate `ranking_dashboard_{slug}.html` — forest-green IBM Plex design system.
Include: summary stats (total / GREEN / AMBER / RED / top score), scoring model table,
ranked table (GREEN then AMBER then RED), per-row criterion score bars.

### Step 16 — Per-Parcel HTML Reports (optional)

One self-contained HTML per GREEN and AMBER parcel → `parcel_{parcel_id}_report.html`.
Consume the exported GeoJSON as data source (not the PostGIS table).

---

### Output Table: `analysis.parcels_scored_{slug}`

Inherits all columns from `parcels_phase4_{slug}` plus the following additions.
Flag columns (`flag_*`, `end_present`, `end_species`, `thr_count`, `sc_count`, `saro_raw`)
are retained for audit but excluded from GeoJSON export.

| Field | Type | Group | Notes |
|---|---|---|---|
| `score_buildable_area` | integer | 7 | 0–30 |
| `score_grid_capacity` | integer | 7 | 0–5 |
| `score_voltage_bonus` | integer | 7 | 0 or 5 |
| `score_irradiance` | integer | 7 | 0–5 |
| `score_saro` | integer | 7 | 0–15 |
| `score_noise` | integer | 7 | 0–10 |
| `score_env_risk` | integer | 7 | 0–15 |
| `score_slope` | integer | 7 | 0–5 |
| `score_compactness` | integer | 7 | 0–5 |
| `score_municipal` | integer | 7 | 0–5 |
| `total_score` | integer | 7 | 0–100; NULL for RED |
| `rank` | integer | 7 | Per-tier rank; NULL for RED |
| `triage_class` | text | 8 | GREEN / AMBER / RED |
| `triage_flags` | text | 8 | JSON list |
| `data_completeness` | text | 8 | |
| `buildable_mw` | float | 8 | ba_net_developable_acre / 5 |
| `grid_ratio` | float | 8 | min(buildable_mw / oem_max_capacity, 1.0) |
| `irradiance_vs_ontario_pct` | integer | 8 | round(pvout_mean / 3.56 × 100) |
| `irradiance_pct_in_range` | integer | 8 | 0–100 position in county pvout range |
| `irr_county_min` | text | 8 | Formatted county pvout min label |
| `irr_county_max` | text | 8 | Formatted county pvout max label |
| `analyst_notes` | text | 8 | Default: triage rationale text |

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
- [x] Create `phase4_physical_suitability.ipynb` — v2 implementation
- [ ] Refactor `phase4_physical_suitability.ipynb` to v3 schema (identity enrichment, OEM summary, descriptive env labels, expanded raster stats, flood flag, ECI max 19, column renames)
- [ ] Create `phase5_scoring_ranking.ipynb` — per spec above
- [ ] Run full pipeline for all target counties (PTB, OTT, OXF, BRC/SIM)