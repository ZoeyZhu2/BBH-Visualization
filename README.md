# BBH-Visualization

## What This Project Does

This project identifies sky localization regions of gravitational wave (GW) binary black hole (BBH) merger events that overlap with regions of high interstellar dust where X-ray scattering probability is ≥0.1 and X-ray absorption probability is ≤~0.95 (optical depth τ = 3), then searches archival eROSITA X-ray images of the highest-priority regions for a detectable dust-scattering echo ("halo") that could pinpoint the merger's source location far more precisely than the GW sky map alone.

The work is split across two notebooks:

| Notebook | Role |
|---|---|
| **`bbh_visualization.ipynb`** | Candidate identification: computes the GW/dust overlap, ranks all 167 BBH events by overlap fraction, and determines when eROSITA scanned each high-priority region relative to its GW detection time. Outputs a boundary polygon (RA/Dec) per event for the favorable overlap region. |
| **`bbh_search.ipynb`** | Archival image search: downloads the eROSITA skytile images covering each event's boundary polygon and searches them for a ring-shaped photon excess at the time-delay-predicted radius, using a direct photon-counting annulus search. Calibrated against the known dust-scattering halo of MAXI J1348–630. |

---

## Background

The physical derivations below are worked out in full in two reference notes included in this repo: [`scattering_probability.pdf`](scattering_probability.pdf) (scattering/absorption cross sections and optical depth) and [`halo_size.pdf`](halo_size.pdf) (echo radius and annular width vs. time delay). `bbh_visualization.ipynb` implements the scattering/absorption model; `bbh_search.ipynb` implements the halo-size model to predict where to search each image.

### Why X-ray halos?

X-rays scatter at much smaller angles than longer-wavelength light, making halos tight enough to be useful for localization. The scattering probability depends on:

1. **N_H column density** — the amount of neutral hydrogen along the line of sight. This will tell us how much dust is in the line of sight.
2. **Photon energy** — scattering dominates absorption at E ≥ 2 keV; at 1 keV, absorption is 3× stronger
3. **Dust grain properties** — grain radius `a` and grain material density `ρ_grain`

τ_scattering = N_d * σ_scattering

Use dn_d/da = Aa^−3.5 (Mathis, Rumpl & Nordsieck (1977)) to find dust grain size distribution.

number density: n_d = int(a_min, a_max) {dn_d/da * da} = int(a_min, a_max) {Aa^−3.5 * da} = A/2.5 * 1/a_min^2.5 * (1 - (a_min/a_max)^2.5)

mass density: ρ_d = int(a_min, a_max) {4π/3 * a^3 * ρ_grain * dn_d/da * da} = int(a_min, a_max) {A * 4π/3 * a^3 * ρ_grain * a^-3.5 * da} = A * 8π/3 * a_max^0.5 * ρ_grain * (1 - (a_min/a_max)^0.5)

We get A = (3μm_p * n_H) / (8πρ_grain * a_max^0.5 * G)

dτ_scattering =dn_d/da * da · σ_scattering · L

Integrate over a: τ_scattering = (2A* σ_0 * E_keV^-2 * L * a_max ^1.5)/(3a_0^4) * (1 - (a_min/a_max)^1.5)

Plug in for A: τ_scattering = (μ * m_p * N_H * a_max * σ_0 * E_keV^-2)/(G * 4πρ_grain * a_0^4)


Choosing G = 100, ρ_grain = 3 gr cm−3, μ = 1.4, a_0 = 0.1μm = 10^−5 cm, amax = 0.3μm = 3 × 10−5 cm, and m_p = 1.67 × 10−24 gr, the scattering optical depth is:
 
```
τ_scattering = 1.17×10⁻²² · N_H · (a_max / 0.3 μm) · E_keV⁻²
```
 
And the scattering probability is `P_scattering = 1 − exp(−τ_scattering)`.

### Why do absorption matter?

At low X-ray energies, photoelectric absorption by ISM atoms (O, Ne, Fe) removes photons before they can form a halo. The absorption cross section for ISM gas of standard cosmic composition (Morrison & McCammon 1983) gives:
 
```
τ_absorption ≈ σ_phot,ISM(E) · N_H
```
 
where σ_phot,ISM scales approximately as E⁻³. The absorption probability is `P_absorption = 1 − exp(−τ_absorption)`.
 
The ratio τ_scatter / τ_absorption ≈ 0.35·E_keV², so energies ≥ 2 keV are preferred. The absorption threshold used in this analysis is τ_absorption = 3 (P_absorption ≈ 0.95), meaning regions where ~5% of photons still survive absorption.

### Why eROSITA?
 
eROSITA conducted four consecutive all-sky X-ray surveys (eRASS1–eRASS4) between December 2019 and December 2021, scanning the entire sky every ~6 months. Its scan geometry (spin axis pointing toward the Sun, advancing ~1°/day in ecliptic longitude) makes observation times predictable for any sky position. This notebook computes those times to identify which BBH overlap regions were observed by eROSITA after their GW events were detected.

### How big is the echo, and how wide is the ring? (used by `bbh_search.ipynb`)

A dust-scattering echo traces out the locus of equal light-travel-time delay between the direct (unscattered) path and the scattered path — a paraboloid for a source effectively at infinity. Its angular radius θ grows with the time delay Δt since the unscattered light arrived and shrinks with the distance *d* to the scattering dust (`halo_size.pdf` eq. 2):

```
θ ≈ 0.49° × (d / 8500 pc)^(-1/2) × (Δt / 1 yr)^(1/2)
```

8500 pc (the distance to the Galactic Center) is used as the default assumed scattering-dust distance, since most contributing dust along a BBH sightline lies near the Galactic Plane. This is `predicted_theta_arcmin()` in `bbh_search.ipynb`, driven by each event's GW-to-eROSITA-observation time delay Δt (`parse_relative_time_to_days`).

The ring is not infinitely thin. Its dominant width contribution is the finite angular spread of the scattering probability distribution around θ, which is approximately Gaussian with characteristic width (Mauche & Gorenstein 1986, via `halo_size.pdf` eq. in "Halo Annular Width"):

```
σ ≈ 10.4 arcmin / (a / 0.1 μm) / (E / 1 keV)
```

for dust grain radius `a` and photon energy `E`. The angle Δθ past θ at which the scattering probability has dropped to 1/10 of its peak — used as the outer edge of the search annulus — is (`halo_size.pdf` eq. 5, `annulus_width_arcmin()` in code):

```
Δθ ≈ ln(10) · σ² / θ
```

Exposure-length and telescope-resolution (~15″ for eROSITA) contributions to the width are both much smaller than this term for the timescales/distances relevant here, so they're not separately modeled.

---

## Data

| Dataset | Source | What it provides |
|---|---|---|
| GW event catalog | [GWOSC REST API v2](https://gwosc.org/api/v2/) | 391 GW events across GWTC-1–4.0; component masses; skymap file URLs |
| GW skymaps | Zenodo (records: 17602505, 8177023, 6513631) | FITS HEALPix probability maps for 167 BBH events |
| HI4PI column density | `NHI_HPX.fits` — [HI4PI Collaboration 2016, A&A 594, A116](https://www.aanda.org/articles/aa/abs/2016/10/aa29341-16/aa29341-16.html) | All-sky neutral hydrogen column density N_H in cm⁻², HEALPix NSIDE=1024, galactic coordinates |
| SFD dust map* | [Schlegel, Finkbeiner & Davis 1998](https://iopscience.iop.org/article/10.1086/305772) | E(B-V) reddening — used only for qualitative background plots, not quantitative analysis |
| eROSITA survey windows | Coutinho et al. (2022) | eRASS1–4 start/end dates used in scan timing model |
| MAXI J1348–630 | [Mastroserio et al. 2021, A&A 646, A83](https://www.aanda.org/articles/aa/full_html/2021/03/aa39757-20/aa39757-20.html) | Validation source: known eROSITA dust-scattering halo at ~3.8 kpc |
| eROSITA DR1 skytile images | [eROSITA DR1 archive](https://erosita.mpe.mpg.de/dr1/) — `EXP_010` product, band `024` (0.2–2.3 keV) | 3240×3240 px images (4″/pixel) for individual sky tiles; downloaded on demand into `skytiles/{event_name}/` by `bbh_search.ipynb` |
| eROSITA skytile lookup | `SKYMAPS_052022_MPE.fits` table + [skytile search API](https://erosita.mpe.mpg.de/erodat/skyview/skytile_search_api/) | Authoritative tile IDs (`srvmap`) and true tile centers, used to find which tiles overlap a given boundary polygon and to build download URLs |
| eRASS1 source catalog | [eRASS1_Main v1.2](https://erosita.mpe.mpg.de/dr1/AllSkySurveyData_dr1/Catalogues_dr1/MerloniA_DR1/eRASS1_Main_v1.2.html) (Merloni et al. 2024) — `skytiles/eRASS1_Main.v1.2.fits`, 930,203 sources | RA/DEC/`EXT`/`SKYTILE` per catalogued X-ray source, used to identify and mask out point sources before searching (`get_tile_sources`) |
| eROSITA background maps | `DET_010` product, `BackgrImage`, band `024` | Per-tile smoothed background model (counts/pixel, same pixel grid/WCS as the `EXP_010` image), used to fill in masked point-source pixels (`remove_point_sources`) |
 
*SFD dust map is optional.
 
**You must download `NHI_HPX.fits` separately** from the HI4PI collaboration before running the notebook. Place it in the same directory as the notebook.

---

## Requirements

### Python packages

`bbh_visualization.ipynb`:

```
numpy
healpy
matplotlib
requests
astropy
ligo.skymap
dustmaps
pandas
tarfile   (standard library)
csv       (standard library)
math      (standard library)
os        (standard library)
time      (standard library)
```

`bbh_search.ipynb` (additionally):

```
scipy       (fftconvolve, gaussian_filter, maximum_filter, binary_erosion)
Pillow      (PIL — image export/annotation for candidate plots)
requests    (eROSITA skytile search API + FITS download)
astropy     (FITS I/O, WCS)
pickle      (standard library — save/load search results)
gzip        (standard library — decompress .fits.gz skytile images)
multiprocessing  (standard library — parallel per-tile search)
csv, re, os, time, shutil, tempfile, gc  (standard library)
```

Install non-standard packages with:
 
```bash
pip install numpy healpy matplotlib requests astropy ligo.skymap dustmaps pandas scipy Pillow
```

### External data files

- **`NHI_HPX.fits`** — download from the [HI4PI survey data release](https://cdsarc.cds.unistra.fr/viz-bin/cat/J/A+A/594/A116). Place it in the same directory as the notebook. Required by `bbh_visualization.ipynb`.
- **`skytiles/SKYMAPS_052022_MPE.fits`** — the eROSITA DR1 skytile lookup table (authoritative tile IDs/centers). Required by `bbh_search.ipynb` before downloading or searching any tile images.

---

## How to Run — `bbh_visualization.ipynb`

Every cell is labeled with a run condition comment. The three categories are:
 
| Label | When to run |
|---|---|
| `# Always run` | Must run every session (imports, TARGET_NSIDE, eROSITA survey windows, skymap file listing) |
| `# Run if I need to download skymaps` | Run once to download skymap files. Skip if `skymaps/` is already populated. Takes 10–30 min. |
| `# Not necessary to run` | Optional visualization or diagnostic cells — some take several minutes |
 
**Minimum sequence for a fresh session (skymaps already downloaded):**
 
```
Cell 1.1 (imports) → Cell 1.2 (constants) → skymap_files listing → Cell 6.1 → Cell 6.2 → Cell 8.1
```
 
**Full eROSITA timing analysis sequence:**
 
```
Cell 1.1 → Cell 1.2 → skymap_files listing → Cell 6.1 → Cell 6.2 → Cell 8.1
→ Cell 10 → Cell 11 → Cell 12 → Cell 14 → Cell 16.1 → Cell 16.2 → Cell 17 → Cell 18
```
 
**Section overview:**
 
| Cells | Purpose | Required? |
|---|---|---|
| 1.1–1.2 | Imports and global constants (NSIDE, eROSITA survey windows) | **Always** |
| skymap listing | `skymap_files = os.listdir("skymaps")` | **Always** |
| 2.1–2.6 | Fetch GW catalog, filter BBH events, download skymaps | Only if downloading |
| 3.1–3.3 | Test skymap download with GW150914 | After fresh download |
| 4 | Per-year plots of all BBH 90% credible regions | Optional |
| 5.1–5.2 | Download and plot SFD dust map | Optional |
| 6 | Math explanation comments | Optional reading |
| 6.1–6.2 | Define `get_hi_map()` and `get_tau_scattering_and_absorption()` | **Before any analysis** |
| 6.3–6.6 | Diagnostic window calculations; scattering, absorption, and combined probability maps | Optional |
| 7, 7.1.1–7.1.2 | Sort skymaps chronologically; plot in chunks overlaid on probability map | Optional |
| **8.1** | `calculate_overlap()` — **primary scientific result** | **For any overlap analysis** |
| 8.2.1–8.2.3 | `calculate_prob_total_failure()` — compound detection probability | After 8.1 |
| 9.1–9.4 | Top-10 overlap event plots (plain, highlighted, chunked, batch) | Optional |
| **10–12** | eROSITA timing setup: Sun positions, observation time functions, GW trigger times | **Before timing analysis** |
| 13.1 | Spot-check timing for a single sky position | Optional |
| 14 | `get_overlap_coordinates()` — overlap pixel (RA, Dec) list | Before 15–16 |
| 15.1–15.2 | Per-pixel eROSITA crossing times and delays (slow, precise) | Optional |
| **16.1–16.2** | Fast center-pixel timing approximation | **Main timing pipeline** |
| **17–18** | Filter for post-GW observations; batch run for all energies | **Main timing pipeline** |
| 19.1–19.2 | Validation: MAXI J1348–630 on probability map | Optional |

---

## Pipeline & How to Run — `bbh_search.ipynb`

**Goal:** search archival eROSITA skytiles covering each event's boundary polygon (produced by `bbh_visualization.ipynb`, Step 12) for a dust-scattering echo.

**Pipeline:**

1. **Tile discovery** — sample a grid of points inside each event's boundary polygon (`sample_grid_inside_polygon`, `step_deg=1.5`, smaller than the ~3.6° skytile size so no tile is skipped) and query the eROSITA skytile-search API per point (`find_skytiles_at_point`) to build the set of tiles overlapping the region (`find_all_skytiles_for_event`). A `margin_deg` buffer expands the polygon outward first (`expand_polygon`) so tiles near the boundary aren't missed.
2. **Image retrieval** — resolve each candidate tile's true ID/center against the authoritative `SKYMAPS_052022_MPE.fits` table (`find_real_srvmap`, since the search API's own `ra_cen`/`de_cen` can be off by up to ~1.8°), then download its `EXP_010` image product (3240×3240 px, 4″/pixel, band 024 = 0.2–2.3 keV) into `skytiles/{event_name}/` (`download_all_skytiles` / `download_skytile_image`), verifying the WCS against the expected tile center (`_verify_wcs_matches_tile`).
3. **Point-source masking** — look up each tile's catalogued sources in the `eRASS1_Main` catalog (`get_tile_sources`, filtered by `SKYTILE`) and replace every pixel within a source's exclusion radius (`build_point_source_mask`: 30″ for a point-like source with `EXT`=0, or `EXT` + 15″ margin if extended) with the corresponding pixel from that tile's own `DET_010`/`BackgrImage` background model (`remove_point_sources`), so a bright catalogued source can't inflate the annulus photon counts used for detection. Operates on an in-memory copy only — the downloaded tile images on disk are never modified — and can be disabled with `run_one_search_per_event(..., remove_sources=False)`.
4. **Predicted ring geometry** — for each event, convert its mean GW-to-observation time delay (`parse_relative_time_to_days`, `mean_dt_days_for_event`) into a predicted echo radius and annulus width using the dust-scattering geometry above (`predicted_theta_arcmin`, `annulus_width_arcmin`, `search_radius_window_arcmin`).
5. **Detection (`photon_annulus_search`)** — crop each (now point-source-masked) tile to the (margin-expanded) polygon (`crop_to_polygon`); for each radius step across the predicted ±10% window (`UNCERTAINTY_FRAC`), build a ring-shaped kernel sized to that radius's physical annulus width and convolve it with the masked image via FFT (`scipy.signal.fftconvolve`) to get a per-pixel summed-photon-count map at every possible center simultaneously. This directly sums raw photon counts in the ring rather than relying on image gradients/edges, which suits eROSITA's sparse, often single-photon-dominated data. Local maxima above `vote_threshold_frac` (default 0.6) of that radius layer's own peak are kept as raw candidates (`scipy.ndimage.maximum_filter`), with `MAX_RAW_CANDIDATES_PER_RADIUS` as a hard safety cap against flat count plateaus.
6. **Non-max suppression** — candidates from every tested radius (and, in `run_one_search_per_event`, every tile) are merged and collapsed with a greedy NMS pass (`non_max_suppress_circles` / `non_max_suppress_arcmin_results`): the highest-count candidate in each (center, radius) cluster is kept, everything within `center_dist_px`/`radius_dist_px` of it is dropped, repeated until none remain.
7. **Export & inspection** — results are pickled (`save_results`/`load_results`) and exported to CSV with per-candidate photon counts summed over the full predicted annulus, not just the fitted circle (`export_candidates_to_csv`, via `count_photons_in_annulus`). `plot_candidate`/`plot_top_candidates`/`plot_one_tile_fullres`/`plot_all_tiles_mosaic` render candidates over the raw tile image; `plot_candidate_cleaned`/`plot_top_candidates_cleaned` render the same candidates over the point-source-masked image instead, with a dotted circle marking every masked-out source. `diagnose_event` walks a zero-candidate event through each pipeline stage to show where it bottomed out.
8. **Validation** — the pipeline is calibrated against MAXI J1348–630, a source with a known, previously-published 34–47 arcmin dust-scattering ring (eRASS1: ~34–40′, eRASS2: ~40–47′), by directly reproducing its radial photon profile and running the same search on its own eROSITA tile.

> **Note on method history:** an earlier version of this pipeline (`run_halo_search`, retired but left in the notebook, commented out) used a gradient-based edge map (`simple_edge_map`) followed by a circular Hough transform (`circular_hough_transform`) instead of direct photon counting. Both `simple_edge_map` and `circular_hough_transform` are still used directly by `diagnose_event` and the MAXI J1348–630 validation cell. See `HALO_DETECTION_NOTES.md` for the Hough-transform detection logic and an open question about false-positive rejection for diffuse/random scatter that has not yet been re-evaluated for the current photon-counting method.

**Minimum sequence for a fresh session (tiles already downloaded):**

```
Step 4/5 cells (imports, constants, expand_polygon, tile lookup) → eRASS1_Main catalog cell (get_tile_sources)
→ Step 6 cell (photon_annulus_search, NMS) → point-source masking cell (remove_point_sources)
→ Step 9/10 cells (plotting helpers, run_one_search_per_event) → run_one_search_per_event(...) → save_results(...)
```

**Full sequence including tile download (first run for a new event set):**

```
Steps 1–2 (tile discovery + skytile table lookup) → Step 5 (download_all_skytiles)
→ eRASS1_Main catalog cell (get_tile_sources) → Step 6 (photon_annulus_search + NMS)
→ point-source masking cell (remove_point_sources) → Step 9 (run_one_search_per_event) → Step 10 (save_results)
→ Step 11 (count_photons_in_annulus / export_candidates_to_csv)
```

Downloads and searches are both resumable/idempotent: `download_all_skytiles` skips tiles already saved to disk (verifying with `_verify_fits`/`_verify_wcs_matches_tile` first), and `run_one_search_per_event` only reads local files — it never re-downloads.

`run_one_search_per_event` parallelizes across tiles with `multiprocessing.Pool` (`n_workers`, default `cpu_count() - 1`); pass `n_workers=1` to run single-process for debugging or to avoid pickling issues in some notebook environments.

---

## Outputs

After a full run, the following files are created:
 
```
skymaps/
    GW150914_skymap.fits
    GW200224_222234_skymap.fits.gz
    ... (167 files total)
 
{year}_bbh_events_90%_regions.png
    Per-year equirectangular sky maps of all BBH 90% credible regions,
    color-coded by event. Years: 2015, 2017, 2019, 2020, 2023, 2024.
 
x-rays E = {E}keV/
    scattering_probability_of_x-rays_(E={E}keV)_>=_{thresh}.png
        Full-sky grayscale map of X-ray scattering probability. Darker = higher probability.
    absorption_probability_of_x-rays_(E={E}keV)_<=_{thresh}.png
        Full-sky grayscale map of X-ray absorption probability.
    scattering_and_absorption_probability_of_x-rays_(E={E}keV)_scat_>={s}_absorp_<={a}.png
        Combined scattering + absorption mask.
    bbh_chunk_{N}_at_{E}keV_with_>={s}_scattering_probability_and_<={a}_absorption_probability_overlay.png
        Chronological groups of 10 BBH events overlaid on the probability background.
    bbh_overlap_percentages_>={s}_scattering_probability_and_<={a}_absorption_probability.csv
        Primary scientific output: event_name, overlap_percentage for all 167 events, sorted descending.
    bbh_area_percentages_in_order_of_greatest_overlap_with...csv
        Sky area of each event's 90% region as a fraction of the full sky, in overlap-rank order.
    top_10_overlap_bbh_events_90%_regions_E={E}keV.png
        Top 10 overlap events overlaid on probability background.
    top_10_overlap_bbh_events_overlap_highlighted_E={E}keV.png
        Top 10 events with overlap zones shown at full opacity.
    top_{N}_bbh_events_overlap_highlighted_E={E}keV.png
        One plot per top-ranked event with overlap highlighted.
    prob_total_failure_and_>=1_success_for_...csv
        P(total failure) and P(at least one success) for various event subsets.
    top_{n}_overlap_coords_...csv
        (RA, Dec) of all overlap pixels for the top-N events.
    top_{n}_overlap_regions_and_observation_time_...csv
        Center-pixel eROSITA crossing times, overlap region width in ecliptic longitude,
        eRASS survey number, and delay relative to GW detection.
    filtered_top_{n}_overlap_regions_and_observation_time_...csv
        Subset of the above: only records where eROSITA scanned AFTER the GW event.
    MAXI_J1348-630_overlapped_with_scat_prob_>=...png
        Validation plot: MAXI J1348–630 and its eROSITA halo annuli on the probability mask.
```

`bbh_search.ipynb` additionally produces:

```
skytiles/
    SKYMAPS_052022_MPE.fits
        Authoritative eROSITA DR1 tile lookup table (must be downloaded manually first).
    {event_name}/
        srvmap_{NNNNNN}.fits.gz
            One EXP_010 image per eROSITA tile overlapping that event's boundary polygon.

results.pkl  (or results_smoothed_sigma{N}px.pkl / a custom path, e.g. results_new.pkl)
    Pickled {event_name: {"dt_days", "theta_window_arcmin", "tiles", "candidates", ...}}
    dict from run_one_search_per_event() — the full search output, reloadable via load_results().

{candidates}.csv  (export_candidates_to_csv)
    event_name, blob_id, srvmap, ra_deg, dec_deg, radius_arcmin, radius_arcsec, votes,
    predicted_theta_arcmin, annulus_inner_arcmin, annulus_outer_arcmin,
    photon_count_in_annulus, n_pixels_in_annulus — one row per surviving candidate.

vote_counts.csv  (export_vote_counts)
    Per-event vote-count distributions, for inspecting how cleanly candidates separate from background.

Candidate plots (plot_candidate / plot_top_candidates / plot_one_tile_fullres / plot_all_tiles_mosaic,
and *_smoothed variants):
    Full-resolution or mosaicked tile images with candidate ring(s) and the GW boundary polygon overlaid.
```

---

## Key Results

| Metric | Value |
|---|---|
| Total GW events fetched | 391 |
| BBH events (both masses > 3 M☉) | 273 |
| BBH events with published skymaps | 167 |

---

## Coordinate System Notes
 
Four coordinate systems are used in this project:
 
| System | Convention | Used by |
|---|---|---|
| HEALPix physics | θ = colatitude (0 = north pole), φ = longitude 0–2π | healpy internally |
| Equatorial / ICRS | RA 0–360°, Dec −90° to +90° | GW skymaps, matplotlib plots |
| Galactic | l (longitude) 0–360°, b (latitude) −90° to +90° | HI4PI N_H map |
| Ecliptic | λ (longitude) 0–360°, β (latitude) −90° to +90° | eROSITA scan timing model |
 
All maps are resampled to NSIDE=1024 (pixel area ≈ 11.8 arcmin²) before any pixel-by-pixel comparison. The HI4PI galactic map is remapped to equatorial pixel ordering before overlap calculation so that pixel indices are directly comparable.

---

## References

- **`scattering_probability.pdf`**, **`halo_size.pdf`** (this repo) — internal derivation notes for the scattering/absorption optical depth model and the echo-radius/annular-width model implemented in the two notebooks.
- **HI4PI Collaboration 2016**, A&A 594, A116 — source of the N_H column density map.
- **Mathis, Rumpl & Nordsieck 1977**, ApJ 217, 425 — power-law grain size distribution used for τ_scattering.
- **Mauche & Gorenstein 1986**, ApJ 302, 371 — differential X-ray scattering cross section for dust grains and the Gaussian approximation to the scattering-angle probability distribution used for halo annular width.
- **Morrison & McCammon 1983**, ApJ 270, 119 — photoelectric absorption cross sections for ISM gas of standard cosmic composition.
- **Corrales 2015**, ApJ 805, 23 — Mie-theory corrections to the simple-diffraction scattering cross section approximation.
- **Schlegel, Finkbeiner & Davis 1998**, ApJ 500, 525 — SFD dust map used for visual reference only.
- **GWOSC** — Gravitational-Wave Open Science Center, https://gwosc.org — source of all GW event data and skymaps.
- **GWTC-1, GWTC-2.1, GWTC-3, GWTC-4.0** — the four GW transient catalogs covering O1–O4a.
- **Coutinho et al. 2022**, Proc. SPIE 12181, 2628946 — source of eROSITA eRASS1–4 survey window dates. https://doi.org/10.1117/12.2628946
- **Mastroserio et al. 2021**, A&A 646, A83 — eROSITA detection of dust-scattering halo around MAXI J1348–630; used as validation source for both the overlap-probability mask and the ring-search pipeline.
- **eROSITA DR1** — https://erosita.mpe.mpg.de/dr1/ — source of skytile images (`EXP_010` product) and the `SKYMAPS_052022_MPE.fits` tile lookup table used by `bbh_search.ipynb`.

---

## Contact / Future Work

- Extend to GWTC-5.0 events as they are released
- Incorporate a more detailed grain size distribution or composition model
- Add an automated significance test for `bbh_search.ipynb` candidates (e.g. Poisson comparison of counts in the predicted annulus vs. a similar-area background region) — currently, a candidate only has to rank in the top `vote_threshold_frac` of its own tile's peak count, with no absolute/background-subtracted significance check; see `HALO_DETECTION_NOTES.md` for the open question this leaves about diffuse/random scatter being reported as a false detection
- Re-run the full `run_one_search_per_event` pipeline (currently interrupted mid-run per the notebook's saved outputs) across all filtered high-priority events and cross-check candidates against the MAXI J1348–630 calibration

Created by Zoey Zhu zyz2000 June 2026
