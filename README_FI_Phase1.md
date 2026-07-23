# Functional Integrity — Phase 1

Global, country-level agricultural integrity from **ESA CCI / C3S Land Cover (300 m)**,
computed in Google Earth Engine for every year **2000–2015**.

For each country and year the pipeline reports:

| Field | Meaning |
|-------|---------|
| `gritM` | Fraction of that country's agricultural cells whose surrounding ~1 km ring holds ≥ `THRESHOLD_PCT` % natural land |
| `agArea_ha` | Agricultural area (hectares) |

The implementation is validated to closely reproduce an earlier arcpy pipeline, and is the
historical 300 m baseline that Phase 2 ([separate repository](#related-work)) extends to
10 m Dynamic World with a country-resolved service-shed metric.

## How it works

1. **Reclass to fractional naturalness.** Each CCI LCCS class is mapped to an integrity
   weight in 0–1:

   | Classes | Weight |
   |---------|--------|
   | Cropland (10, 11, 12, 20) and urban (190) | 0.0 |
   | Mosaic cropland (30) | 0.25 |
   | Mosaic natural (40) | 0.75 |
   | Natural vegetation (50–180) and bare (200–202) | 1.0 |
   | Water (210) / snow-ice (220) | **masked** (absent from the remap) |

   Masking water and ice by omission matches arcpy's `Reclassify(..., "NODATA")`.

2. **Agricultural mask.** Classes 10–41, i.e. any agriculture including both mosaic
   classes.

3. **Annulus focal mean (~1 km ring).** Mean naturalness in the ring around each pixel,
   computed as *outer circle − inner circle* for both sum and count:
   - outer radius `KERNEL_OUTER = 3` pixels (~900 m at 300 m grain)
   - inner radius `KERNEL_INNER = 1` pixel

   Kernels are in **pixel** units. Because Earth Engine's neighbourhood reducers skip
   masked pixels, this reproduces arcpy `FocalStatistics(..., "DATA")`. Note the inner
   radius of 1 excludes the focal cell *and* its four rook neighbours — a small, accepted
   difference from arcpy's `NbrAnnulus(0.5, 3)`, confirmed close enough against the
   original.

4. **Threshold and reduce.** Ring percentage ≥ `THRESHOLD_PCT` (20%) gives a binary
   integrity image, masked to agricultural land; `ee.Image.pixelArea()` gives true
   agricultural area. Both are reduced per country with a combined mean/sum reducer.

5. **Export.** One batch CSV export per (tile, year) to Drive, named
   `integ_<Tile>_<Year>`, with columns `GID_0, year, gritM, agArea_ha`.

## Architecture — why it's tiled, and why assignment matters

A global `reduceNeighborhood` fails at the ±180/±90 raster edge, so **tiling is mandatory**.
The globe is covered by **10 tiles** defined as lon/lat boxes in `TILES`:

```
N_America  S_America  Europe  Africa  Asia
SE_Asia_Oceania  Pacific_East  Arctic  Indian_Is  Cape_Verde
```

Antarctica is **intentionally excluded** — the polar projection fails on export, and there
are no sovereign countries and no land-cover-integrity signal under permanent ice.

Tile boxes may overlap. The single mechanism that makes this unambiguous is the
**country → tile assignment** in §6: each country is assigned to the tile whose box overlaps
it most, computed once in geopandas. That one dict governs everything downstream —

- §8 exports *only* each tile's assigned countries, so every country is computed in
  **exactly one** tile;
- §9 is therefore a plain concatenation, with no possibility of double-counting or silent
  dropping.

Each tile is processed over its box buffered by `BUFFER_KM = 25` km, so rings near tile
edges and national borders still have data. The buffer affects only the focal computation,
never attribution.

**No map projection is set anywhere, deliberately.** Mollweide and EPSG:6933 either failed
to parse or threw edge-transform errors. `pixelArea()` returns true areas and the ring uses
pixel kernels, so no CRS is required.

## Repository layout

| File | Purpose |
|------|---------|
| `IntegrityTool_Phase1_v2.ipynb` | Full pipeline: setup, config, reclass, kernel, per-tile processing, assignment, export, combine, verify |
| `README.md` | This file |

## Requirements

- A Google Earth Engine–enabled Cloud project (`ee.Initialize(project=...)`).
- Python `earthengine-api`, `geemap`, `geopandas`, `pandas`; developed and run in Google
  Colab with Drive mounted.
- Read access to the land cover collection and the GADM boundary asset.

### Datasets

| Role | Asset | Notes |
|------|-------|-------|
| Land cover | `projects/sat-io/open-datasets/ESA/C3S-LC-L4-LCCS`, band `b1` | 300 m; **asset begins in 2000**, hence `YEARS = 2000–2015` |
| Country boundaries (EE) | `projects/ee-fremier/assets/GADM_GID_0_simplified` | used by `reduceRegions` in §8 |
| Country boundaries (local) | `GADM_GID_0_simplified.shp` in Drive | same layer, read by geopandas for the §6 assignment |

Both boundary copies must be the *same* layer — §6 assigns countries using the local
shapefile, §8 reduces using the EE asset.

## Running it

**One-time (§0):** prepare and ingest the GADM boundary asset. Never again after that; the
cell is commented so *Run all* skips it.

**Every session, top to bottom:**

1. **§1** auth → **§2** config → **§3–§5** definitions
2. **§6** country → tile assignment *(the single assignment that governs everything)*
3. **§7** *(optional)* single-tile test
4. **§8** export all tiles × years → CSVs in Drive
5. **§9** *(after exports finish)* combine → long + wide CSVs
6. **§10** verify

After a runtime restart, use **Runtime → Run all**.

### Key parameters (§2)

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `YEARS` | 2000–2015 | Limited by the land cover asset's start date |
| `SCALE` | 300 | Native LCCS resolution, metres |
| `KERNEL_OUTER` / `KERNEL_INNER` | 3 / 1 | Ring radii, **pixels** |
| `THRESHOLD_PCT` | 20 | Integrity threshold, percent |
| `BUFFER_KM` | 25 | Processing overlap so edge rings have data |
| `DRIVE_FOLDER` | `EE_Integrity_v2` | **Versioned** — bump it for a clean re-run |

## Outputs

§9 writes two CSVs to the Drive folder:

- `integrity_clean_2000_2015.csv` — long form, one row per country × year. Canonical for
  time-series analysis.
- `IntegritySeries_2000_2015.csv` — wide form, one row per country with `GritM_<year>` and
  `AgArea_<year>` columns, matching the original arcpy output for joins and mapping.

Sharded tile exports repeat each country across shards, and summed areas carry last-bit
float drift, so §9 first collapses on `[GID_0, year, tile]` by mean. After that collapse an
assertion guards the one-row-per-country-year invariant — if it fires, the cause is a
genuine cross-tile straddler, not shard noise.

## Known limitations

- **Giant-country undercount.** A country larger than its assigned tile, or straddling a
  seam by more than `BUFFER_KM`, is measured only over the portion inside its assigned
  tile's buffered region. Russia and Canada may read slightly low; §10 prints them for
  inspection. The full fix — summing a country across every tile it touches — is future
  work.
- **Antarctica is excluded** by design (see above).
- **Free-tier quota** throttles the project to restricted mode; batch tasks sit in READY and
  drain slowly. Re-running §8 only queues duplicates. Phase 2's 10 m workload needs a quota
  uplift.

## Maintenance notes

- **Changing `TILES` requires re-running §6** (which reassigns countries) *and* re-exporting
  §8. §6 and §8/§9 must always be run against the same `TILES` definition.
- **Bump `DRIVE_FOLDER`** for any re-export so stale CSVs from an earlier tile design can't
  mix in.

## Background

The functional-integrity threshold and its global relevance come from:

- Mohamed, A., DeClerck, F., Verburg, P.H., …, Fremier, A., et al. (2024). Securing
  Nature's Contributions to People requires at least 20%–25% (semi-)natural habitat in
  human-modified landscapes. *One Earth* 7, 59–71.
  https://doi.org/10.1016/j.oneear.2023.12.008

This work is developed in collaboration with the
[Food Systems Countdown Initiative](https://www.foodcountdown.org/), which carries
functional integrity as a standing environmental indicator.

## Related work

- **Phase 2** — country-resolved functional integrity from Dynamic World at 10 m, with an
  explicit 150–1150 m service-shed annulus, reusing this tiling and export skeleton.
  *(Link once the Phase 2 repository is public.)*
- **1990s extension** — splicing 1992–1999 from the separate ESA-CCI asset is possible, but
  requires checking the ESA-CCI → C3S discontinuity at 2000.

## Contact

Alex Fremier, Riverine Ecosystem Ecology Lab, School of the Environment, Washington State
University — https://labs.wsu.edu/ecology/

## License

*(Add a license before making the repository public — e.g. MIT or Apache-2.0 for the code,
CC-BY-4.0 for derived data. Not yet specified.)*
