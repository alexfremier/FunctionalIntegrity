# Functional Integrity — Phase 1

Global, wall-to-wall functional-integrity surface of human-modified landscapes, computed
from **ESA CCI Land Cover (300 m)** in Google Earth Engine.

Phase 1 produces the global integrity layer: for every ice-free land pixel, the fraction of
(semi-)natural habitat in the surrounding neighborhood, crosswalked from ESA CCI land-cover
classes. Because a single global reduction at this grain exceeds per-task limits, the
computation is partitioned into **11 continental tiles** and run per tile per year, then
recombined.

Phase 2 ([separate repository](#related-work)) extends this into a country-resolved,
Dynamic World (10 m) product with an explicit service-shed kernel.

> **Fill in before publishing:** confirm the exact naturalness crosswalk, neighborhood
> radius, year range, and output format below match the final notebook. Sections marked
> _[confirm]_ are reconstructed from project notes and should be checked against
> `IntegrityTool_Phase1_final.ipynb`.

## What the pipeline computes

1. **Naturalness crosswalk.** Each ESA CCI land-cover class is mapped to an integrity
   weight (0–1): natural vegetation = 1.0, mosaic-natural and mosaic-cropland at
   intermediate weights, cropland and built at low weights, water and ice masked.
   _[confirm exact weights against the notebook.]_
2. **Neighborhood integrity.** A moving-window reduction computes, per pixel, the mean
   naturalness over the surrounding neighborhood. _[confirm radius / kernel.]_
3. **Continental tiling.** The globe is split into 11 continental tiles (Africa, Asia,
   Europe, N. America, S. America, SE Asia / Oceania, Pacific East, Arctic, Indian Ocean
   islands, Cape Verde, Antarctic). Each tile is processed independently to stay within
   per-task memory limits, and boundaries are handled so tiles mosaic cleanly without seam
   artifacts.
4. **Export.** One export per tile per year (`integ_<Tile>_<Year>`). _[confirm: exported to
   Drive / as an Earth Engine asset / other, and the output type — raster or table.]_

## Repository layout

| File | Purpose |
|------|---------|
| `IntegrityTool_Phase1_final.ipynb` | Final validated pipeline (11-tile continental scheme, server-side) |
| `README.md` | This file |

_[If an earlier arcpy version is included for reference, list it here and note it is
superseded — see [History](#history).]_

## Requirements

- A Google Earth Engine–enabled Cloud project (`ee.Initialize(project=...)`).
- Python `earthengine-api`; developed and run in Google Colab.
- Read access to ESA CCI Land Cover and to the tile-boundary geometries.

### Datasets

| Role | Asset | Resolution |
|------|-------|------------|
| Land cover | ESA CCI Land Cover (LCCS) _[confirm exact asset ID / collection]_ | 300 m |
| Continental tiles | _[confirm: asset, or defined inline in the notebook]_ | — |

## Running it

1. **Authenticate and configure.** Run the setup cell (`ee.Authenticate` /
   `ee.Initialize`), then set the project, year range, and output target.
2. **Run per tile per year.** The submission loop iterates the 11 tiles across the year
   range and submits one export each. _[confirm year range — project notes indicate
   2000–2015.]_
3. **Monitor tasks.** Poll task state; failed tasks print their error. Note that the
   **Antarctic tile is expected to fail / is excluded** — it carries no agricultural land
   and is not part of the product.
4. **Recombine.** _[confirm how tiles are mosaicked into the global surface — in this
   notebook, downstream, or in Phase 2.]_

## Design notes

- **Server-side throughout.** No client-side loops over pixels or features; all reduction
  happens server-side, with the client only submitting and monitoring tasks. This is what
  makes the global computation tractable and the notebook portable.
- **Continental tiling is a memory strategy, not a scientific one.** The tile scheme exists
  purely to stay within per-task limits; results are identical to an (infeasible) single
  global pass, and tiles are designed to mosaic without seams.
- **Explicit projection.** Computed images are anchored to a real projection rather than a
  projection-less constant, which avoids the reprojection failures that can otherwise
  appear at tile boundaries. _[confirm this matches the final notebook.]_
- **Migrated from arcpy.** The analysis was originally prototyped in an arcpy workflow and
  rewritten for Earth Engine to reach global extent and to make the released tool
  reproducible without proprietary software. See [History](#history).

## Validation

The 11-tile scheme completes reliably: across the full year range, every continental tile
succeeds except Antarctic (expected, no agricultural land). This clean, repeated completion
at 300 m is what establishes the pipeline as sound — the compute limits encountered at
Phase 2's 10 m resolution are a matter of allocation, not pipeline design.

## History

- **arcpy prototype** — original desktop-GIS implementation. Superseded; retained only for
  reference. _[Include or omit as you prefer.]_
- **GEE / Colab migration** — rewritten for global extent, portability, and open release.
- **`IntegrityTool_Phase1_final.ipynb`** — current validated version (11 continental tiles,
  server-side, seam-free mosaic).

## Background

The functional-integrity threshold and its global relevance come from:

- Mohamed, A., DeClerck, F., Verburg, P.H., …, Fremier, A., et al. (2024). Securing
  Nature's Contributions to People requires at least 20%–25% (semi-)natural habitat in
  human-modified landscapes. *One Earth* 7, 59–71.
  https://doi.org/10.1016/j.oneear.2023.12.008

This work is developed in collaboration with the
[Food Systems Countdown Initiative](https://www.foodcountdown.org/).

## Related work

- **Phase 2** — country-resolved functional integrity from Dynamic World (10 m) with an
  explicit 150–1150 m service-shed kernel. *(Link once the Phase 2 repository is public.)*

## Contact

Alex Fremier, Riverine Ecosystem Ecology Lab, School of the Environment, Washington State
University — https://labs.wsu.edu/ecology/

## License

*Public after the publication is published*
