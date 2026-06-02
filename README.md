<!-- needs a lot of updating -->
# BBH-Visualization

## What This Project Does

This notebook identifies graphs sky localization regions of gravitational wave (GW) binary black hole (BBH) merger events and regions of high interstellar dust where X-ray scattering probability is greater than 0.1 and X-ray absorption probability is less than 0.5. The goal is to find candidate events where a detectable X-ray scattering halo could be used to pinpoint the source location far more precisely than the GW sky map alone.

---

## Background

### Why X-ray halos?

X-rays scatter at much smaller angles than longer-wavelength light, making halos tight enough to be useful for localization. The scattering probability depends on:

1. **N_H column density** — the amount of neutral hydrogen along the line of sight. This will tell us how much dust is in the line of sight.
2. **Photon energy** — scattering dominates absorption at E ≥ 2 keV; at 1 keV, absorption is 3× stronger
3. **Dust grain properties** — grain radius `a` and grain material density `ρ_grain`

The scattering optical depth at reference grain parameters (a = 0.1 μm, ρ = 3 g/cm³) is:

```
optical depth to scattering = τ_scattering = 8.4×10⁻²³ · N_H · (a/0.1 μm) · (ρ/3) · (E/keV)⁻¹ = 8.4×10⁻²³ · N_H · (E/keV)⁻¹
```

And the scattering probability is `P_scattering = 1 − exp(−τ_scattering)`.

### Why do absorption matter?

At low X-ray energies, photoelectric absorption by ISM atoms (O, Ne, Fe) removes photons before they can form a halo. 

```
optical depth to photoelectric absorption = τ_(phot,ISM) = τ_absorption ≈ (2.4 × 10−22) * N_H * E_keV^-3
```

And the scattering probability is `P_absorption = 1 − exp(−τ_absorption)`.

The ratio of scattering to absorption optical depth scales as ≈ 0.35·E_keV², so energies ≥ 2 keV are preferred for halo detection.

---

## Data

| Dataset | Source | What it provides |
|---|---|---|
| GW event catalog | [GWOSC REST API v2](https://gwosc.org/api/v2/) | 391 GW events across GWTC-1–4.0; component masses; skymap file URLs |
| GW skymaps | Zenodo (records: 17602505, 8177023, 6513631) | FITS HEALPix probability maps for 167 BBH events |
| HI4PI column density | `NHI_HPX.fits` — [HI4PI Collaboration 2016, A&A 594, A116](https://www.aanda.org/articles/aa/abs/2016/10/aa29341-16/aa29341-16.html) | All-sky neutral hydrogen column density N_H in cm⁻², HEALPix NSIDE=1024 |
| SFD dust map* | [Schlegel, Finkbeiner & Davis 1998](https://iopscience.iop.org/article/10.1086/305772) | E(B-V) reddening — used only for qualitative background plots, not quantitative analysis |

*SFD dust map is optional

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
tarfile  (standard library)
csv      (standard library)
math     (standard library)
```

Install with:

```bash
pip install numpy healpy matplotlib requests astropy ligo.skymap dustmaps
```

### External data file

Download `NHI_HPX.fits` from the [HI4PI survey data release](https://cdsarc.cds.unistra.fr/viz-bin/cat/J/A+A/594/A116). Place it in the same directory as the notebook.

---

## How to Run

The notebook cells are labeled with one of three run conditions:

| Label | When to run |
|---|---|
| `# Always run` | Must run every time you open the notebook (setup, imports, TARGET_NSIDE) |
| `# Run if dir skymaps does not exist` | Run once to download skymap files. Skip if `skymaps/` already populated. |
| `# Not necessary to run` | Optional visualization cells - some take minutes to run |

**Each Section of Code:**

1*. Cell 1.1 — imports 
2*. Cell 1.2 — set `TARGET_NSIDE = 1024` 
3**. Cells 2–4 — fetch GW events from GWOSC, filter for BBH 
4**. Cells 5–7 — find skymap URLs, download and extract (~5–15 min depending on connection) 
5. Cells 8-10 - test download by graphing 1 event 
6. Cell 11 - graphing bbh events 
7^. Cell 12 - graphing SFD dust map 
5*. Cell 13.1 — define `get_tau_scattering_and_absorption()` 
6. Cell 13.2-13.4 — compute and graph scattering and absorption probability maps 
7. Cell 14 — chunk skymaps and plot overlaid on probability maps 
8. Cells 15.1–15.3 — compute overlap percentages, save CSVs 
9. Cell 15.4 — compute probability of total failure 
* - absolutely necessary to run 
** - run if skymaps need to be downloaded 
^ - extra optional 

**To re-run from the middle** (skymaps already downloaded):

```python
skymap_files = os.listdir("skymaps")  # Cell just after Cell 7
```

Run this cell and cells 1, 2, 5, and then skip to what you want to run.

---

## Outputs

After a full run, the following files will be created in the working directory:

```
skymaps/
    GW150914_skymap.fits
    GW200224_222234_skymap.fits.gz
    ... (167 files total)

scattering_probability_of_x-rays_E=1keV_>=_0.1.png
    Full-sky grayscale map of X-ray scattering probability at E=1 keV.
    Darker = higher scattering probability. Concentrated along Galactic plane.
scattering_probability_of_x-rays_E=2keV_>=_0.1.png
    Full-sky grayscale map of X-ray scattering probability at E=2 keV.
    Darker = higher scattering probability. Concentrated along Galactic plane.
absorption_probability_of_x-rays_E=2keV_<0.5.png
    Full-sky grayscale map of X-ray absorption probability at E=2 keV.
    Darker = lower absorption probability. Concentrated along Galactic plane.
scattering_and_absorption_probability_of_x-rays_E=2keV_scat_>0.1_absorp_<0.5.png
    Full-sky grayscale map of X-ray absorption probability at E=2 keV.
    Darker = lower absorption probability. Concentrated along Galactic plane.

2015_bbh_events_90%_regions.png
2017_bbh_events_90%_regions.png
... (one per active year)
    Per-year equirectangular sky maps showing all 90% credible regions,
    color-coded by event.

bbh_chunk_1_at_2keV_with_>0.1_scattering_probability_and_<0.5_absorption_probability_with_overlay.png
bbh_chunk_2_at_2keV_with_>0.1_scattering_probability_and_<0.5_absorption_probability_with_overlay.png
... (17 files)
    Groups of 10 events overlaid on the dust/scattering background map.

bbh_overlap_percentages_>0.1_scattering_probability_and_<0.5_absorption_probability.csv
    Two-column CSVs: event_name, overlap_percentage.
    Sorted descending by overlap percentage.
    These are the primary scientific outputs.
```

---

## Key Results

| Metric | Value |
|---|---|
| Total GW events fetched | 391 |
| BBH events (both masses > 3 M☉) | 273 |
| BBH events with published skymaps | 167 |
| Sky fraction with P_scatter ≥ 10% at 1 keV | 23.6% |
<!-- | Sky fraction with P_scatter ≥ 63% at 1 keV (τ > 1) | 0.6% |
| Event with highest overlap (≥10% threshold) | GW200224_222234 — **100%** overlap, ~50 deg² localization |
| P(at least one event overlaps high-scattering sky) | ≈ 1.0 (all 167 events) |
| P(at least one success) excluding top event | 0.9999994 | -->
<!-- 
GW200224_222234 dominates the result: its tiny 90% credible region (~50 deg²) falls entirely within the high-scattering part of the sky, giving it 100% overlap at the ≥10% threshold. Removing just this one event drops P(success) from ~1.0 to 0.9999994, still very high. -->

---

## Coordinate System Notes

Three coordinate systems are used in this project.

| System | Convention | Used by |
|---|---|---|
| HEALPix physics | θ = colatitude (0 = north pole), φ = longitude 0–2π | healpy internally |
| Equatorial / ICRS | RA 0–360°, Dec −90–+90° | GW skymaps, matplotlib plots |
| Galactic | l (longitude), b (latitude) | HI4PI N_H map |

The conversion pipeline is: **Equatorial → astropy SkyCoord → Galactic (l, b) → HEALPix (θ, φ) → pixel index**.

All maps are brought to NSIDE=1024 (pixel area ≈ 11.8 arcmin²) before any pixel-by-pixel comparison.

---

## References

<!-- - **Draine 2003** — "Scattering by Interstellar Dust Grains. I. Optical and Ultraviolet" — the physical model for τ_scatter used in this notebook. -->
- **HI4PI Collaboration 2016**, A&A 594, A116 — source of the N_H column density map.
- **Schlegel, Finkbeiner & Davis 1998**, ApJ 500, 525 — SFD dust map used for visual reference.
- **GWOSC** — Gravitational-Wave Open Science Center, https://gwosc.org — source of all GW event data and skymaps.
- **GWTC-1, GWTC-2.1, GWTC-3, GWTC-4.0** — the four GW transient catalogs covering O1–O4a.

---

## Contact / Future Work

This analysis treats each event independently and uses a simplified uniform grain model (fixed a, ρ_grain). Future extensions could:
<!-- - Incorporate a grain size distribution (e.g., MRN: Mathis, Rumpl & Nordsieck 1977) -->
<!-- - Use 3D dust maps (e.g., Bayestar19, Green et al. 2019) to account for dust distance along the line of sight -->
<!-- - Weight by GW distance posterior to check whether the dust is actually between us and the source -->
- Apply to future GWTC-5.0 events as they are released