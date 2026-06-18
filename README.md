<!-- needs a lot of updating -->
# BBH-Visualization

## What This Project Does

This notebook identifies sky localization regions of gravitational wave (GW) binary black hole (BBH) merger events and overlaps them with regions of high interstellar dust where X-ray scattering probability is ≥0.1 and X-ray absorption probability is ≤~0.95 (optical depth τ = 3). The goal is to find candidate events where a detectable X-ray dust-scattering halo could help pinpoint the source location far more precisely than the GW sky map alone. For high-priority events, the notebook also computes when the eROSITA X-ray telescope scanned those regions relative to the GW detection time, enabling retrospective archival searches.

---

## Background

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
 
*SFD dust map is optional.
 
**You must download `NHI_HPX.fits` separately** from the HI4PI collaboration before running the notebook. Place it in the same directory as the notebook.

---

## Requirements

### Python packages

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

Install non-standard packages with:
 
```bash
pip install numpy healpy matplotlib requests astropy ligo.skymap dustmaps pandas
```

### External data file

Download `NHI_HPX.fits` from the [HI4PI survey data release](https://cdsarc.cds.unistra.fr/viz-bin/cat/J/A+A/594/A116). Place it in the same directory as the notebook.

---

## How to Run

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

---

## Key Results

| Metric | Value |
|---|---|
| Total GW events fetched | 391 |
| BBH events (both masses > 3 M☉) | 273 |
| BBH events with published skymaps | 167 |
---

## Coordinate System Notes

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

 **HI4PI Collaboration 2016**, A&A 594, A116 — source of the N_H column density map.
- **Mathis, Rumpl & Nordsieck 1977**, ApJ 217, 425 — power-law grain size distribution used for τ_scattering.
- **Morrison & McCammon 1983**, ApJ 270, 119 — photoelectric absorption cross sections for ISM gas of standard cosmic composition.
- **Schlegel, Finkbeiner & Davis 1998**, ApJ 500, 525 — SFD dust map used for visual reference only.
- **GWOSC** — Gravitational-Wave Open Science Center, https://gwosc.org — source of all GW event data and skymaps.
- **GWTC-1, GWTC-2.1, GWTC-3, GWTC-4.0** — the four GW transient catalogs covering O1–O4a.
- **Coutinho et al. 2022**, Proc. SPIE 12181, 2628946 — source of eROSITA eRASS1–4 survey window dates. https://doi.org/10.1117/12.2628946 — source of eROSITA eRASS1–4 survey window dates. 
- **Mastroserio et al. 2021**, A&A 646, A83 — eROSITA detection of dust-scattering halo around MAXI J1348–630; used as validation source.

---

## Contact / Future Work

- Extend to GWTC-5.0 events as they are released
- Incorporate a more detailed grain size distribution or composition model

Created by Zoey Zhu zyz2000 June 2026
