# Functional Integrity — Phase 2

Service-shed functional integrity of human-modified landscapes, computed at national
resolution from **Google Dynamic World (10 m)** and **ESA CCI Land Cover (300 m)** in
Google Earth Engine.

For each pixel of agricultural land, the pipeline measures the fraction of (semi-)natural
habitat within a surrounding **service shed** — an annulus running from 150 m to 1150 m
from the focal cell — and reports, per country, the share of agricultural land whose
service shed meets a minimum functional-integrity threshold (default 20% natural habitat).
The 20–25% threshold follows Mohamed et al. (2024, *One Earth*); see
[Background](#background).

Phase 2 produces the country-resolved, annually repeatable layer. Phase 1
([separate repository](#related-work)) established the global wall-to-wall integrity
surface from ESA CCI on a continental-tile scheme.

## What the pipeline computes

1. **Annual land-cover label.** For a given year, Dynamic World scenes are composited to a
   mean per-class probability, and the annual label is the argmax across the nine class
   probabilities. ESA CCI provides an equivalent 300 m label for the historical baseline.
2. **Naturalness surface.** Each land-cover class is crosswalked to an integrity weight
   (0–1): natural vegetation = 1.0, mosaic-natural = 0.75, mosaic-cropland = 0.25, and so
   on. Water and ice are masked.
3. **Service-shed percentage.** A fixed annulus kernel (150–1150 m) computes, per pixel,
   the mean naturalness across the ring — the ecological capacity delivered to that
   location.
4. **Functional integrity.** The service-shed percentage is thresholded (≥ 20%), masked to
   agricultural land, and reduced per country to (a) the fraction of agricultural land that
   meets the threshold and (b) total agricultural area (ha).

Output is one table per country-year, with fields `fi` (integrity fraction), `agArea_ha`,
`year`, and `product`.

## Repository layout

| File | Purpose |
|------|---------|
| `IntegrityTool_Phase2.ipynb` | Main pipeline: setup, config, kernel, crosswalks, composite, processing, task harness, smoke test, staged submission |
| `README.md` | This file |

> **Note:** the notebook is organized so that cells 1–7 only *define* functions (no
> compute). Nothing is submitted to Earth Engine until the smoke test (section 8) or the
> staged submission harness (section 9) is run.

## Requirements

- A Google Earth Engine–enabled Cloud project (`ee.Initialize(project=...)`).
- Python `earthengine-api`; developed and run in Google Colab.
- Read access to the datasets below and to the boundary asset.

### Datasets

| Role | Asset | Resolution |
|------|-------|------------|
| Land cover (primary) | `GOOGLE/DYNAMICWORLD/V1` | 10 m |
| Land cover (baseline) | `projects/sat-io/open-datasets/ESA/C3S-LC-L4-LCCS` (band `b1`) | 300 m |
| Country boundaries | `projects/<your-project>/assets/GADM_GID_0_simplified` (field `GID_0`) | — |

Update `GADM_ASSET` and `PROJECT` in the configuration cell to point at your own project
and boundary asset.

## Running it

1. **Authenticate and configure.** Run cell 1 (`ee.Authenticate` / `ee.Initialize`) and
   cell 2 (configuration). Set `PROJECT`, `GADM_ASSET`, `ANCHOR_YEAR`, and the threshold.
2. **Define everything.** Run cells 3–7. No compute is triggered.
3. **Smoke test first.** Run section 8. It launches three small countries
   (`SMOKE_TEST = ['PRY','ZMB','PNG']`) at 10 m. If these resolve, the reducer
   configuration is sound. If they fail, the problem is upstream in the composite, not the
   reduction — and you have found out in minutes rather than overnight.
4. **Check the queue.** Use the task-summary helper to poll task state. Failed tasks print
   their error message.
5. **Scale up.** Run the staged submission harness (section 9) with throttling
   (`MAX_INFLIGHT`) to keep the queue shallow under noncommercial-tier limits.

### Key parameters (configuration cell)

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `THRESHOLD_PCT` | 20 | Minimum service-shed % natural to count as integral |
| `INNER_RADIUS_M` / `OUTER_RADIUS_M` | 150 / 1150 | Service-shed annulus |
| `ANCHOR_YEAR` | 2021 | Year to composite |
| `SCALE_10` / `SCALE_300` | 10 / 300 | Native scales for the two products |
| `CRS` | `EPSG:4326` | Pinned everywhere; never inherited |
| `TILESCALE_10` | 16 | `tileScale` for large-extent countries |
| `MAX_INFLIGHT` | 3 | Concurrent tasks under the noncommercial tier |
| `GIANTS` | see cell | Countries that need the higher `tileScale` |

## Design notes and known constraints

These reflect fixes made after an initial global run failed with memory and timeout
errors; they are worth understanding before modifying the pipeline.

- **No array operations in the composite.** The annual label is computed with a band-wise
  max, not `toArray().arrayArgmax()`. Array primitives are the most memory-hungry operation
  in Earth Engine and caused the original 300 m timeout. Tie-breaking differs from
  `arrayArgmax` (this resolves ties to the highest class index; set `ties='low'` to match
  the array version exactly). Ties in a mean-probability surface are vanishingly rare.
- **Single annulus kernel.** The service shed is one fixed kernel (two neighborhood
  passes), not an outer-minus-inner difference (four passes). This roughly halves the
  per-task memory footprint and was necessary to keep large countries within the memory
  ceiling at 10 m.
- **Every computed image is anchored to a real projection.** The naturalness surface is
  seeded from the input land cover, not from `ee.Image(0)` (a projection-less constant that
  caused earlier failures). Any `reduceResolution` step sets an explicit default projection
  first.
- **`bestEffort` is deliberately off.** With it on, Earth Engine silently coarsens scale to
  make a reduction fit, which would compute different countries at different effective
  resolutions. A loud failure is preferred over silent heterogeneity.
- **Large-extent, low-density countries can still time out at 10 m.** Focal-kernel cost
  scales with bounding-box extent, not land area, so sparsely-populated but geographically
  large countries are the hardest cases. `preaggregate_m` (coarsening naturalness before
  the kernel) is the escape hatch; it is an approximation and should be reported when used.
- **Task cancellation uses `cancelOperation`, not `cancelTask`.** On the Cloud-API client,
  `cancelTask` silently no-ops; tasks are Operations addressed by resource name.

## Background

The functional-integrity threshold and its global relevance come from:

- Mohamed, A., DeClerck, F., Verburg, P.H., …, Fremier, A., et al. (2024). Securing
  Nature's Contributions to People requires at least 20%–25% (semi-)natural habitat in
  human-modified landscapes. *One Earth* 7, 59–71.
  https://doi.org/10.1016/j.oneear.2023.12.008
- Schneider, K.R., Remans, R., …, Fremier, A., et al. (2025). Governance and resilience as
  entry points for transforming food systems in the countdown to 2030. *Nature Food.*
  https://doi.org/10.1038/s43016-024-01109-4

This work is developed in collaboration with the
[Food Systems Countdown Initiative](https://www.foodcountdown.org/), which carries
functional integrity as a standing environmental indicator.

## Related work

- **Phase 1** — global wall-to-wall integrity surface from ESA CCI on an 11-tile
  continental scheme. *(Link once the Phase 1 repository is public.)*

## Contact

Alex Fremier, Riverine Ecosystem Ecology Lab, School of the Environment, Washington State
University — https://labs.wsu.edu/ecology/

## License

*(Add a license — e.g. MIT for code, CC-BY 4.0 for derived data — before making the
repository public. Not yet specified.)*
