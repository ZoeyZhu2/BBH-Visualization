# Halo Detection: How Results Are Discovered

Notes on the ring-detection logic in `bbh_search.ipynb` (Step 13, cell
"Searching for halos using CHT"), and an open question about what
counts as a valid detection.

## Pipeline

1. **`simple_edge_map`** — computes the local gradient magnitude of
   the (optionally smoothed) image and keeps pixels above a robust
   threshold (`median + threshold_sigma * MAD`, restricted to the
   polygon mask). This responds to *sharp local structure* — a
   boundary/edge — not simply to "there are more counts here than
   average."
2. **`circular_hough_transform`** — every edge pixel casts a vote for
   every circle of radius `r` (within the predicted radius window)
   it could lie on, at `n_angles=360` candidate centers per pixel. A
   genuine ring's edge pixels all lie on the *same* circle, so their
   votes all land on the *same* accumulator cell — the true center —
   producing a sharp peak. Random/scattered points, even if numerous,
   are geometrically inconsistent with one shared circle, so their
   votes spread thin across many different candidate centers instead
   of piling up on one.
3. **`vote_threshold_frac`** (default 0.6) — keeps only accumulator
   cells with votes >= 60% of that radius layer's *own maximum*.
4. **`non_max_suppress_circles`** — collapses near-duplicate peaks
   (same ring, off by a pixel or two) into one candidate, ranked by
   vote count.

## Question: does diffuse/random scatter within the annulus count as a valid result?

Asked 2026-07-20: "like a plentiful random looking scatter within the
annulus should correspond to a valid result at the center point" —
i.e., if there's a lot of noisy-looking but statistically-excess
scatter spread through the predicted annulus, does the pipeline treat
that as a detection?

**No, not under the current design**, and that's intentional as far
as it goes: a diffuse scatter of counts with no sharp circular
boundary generally will not produce a strong Hough peak, because nothing
about it is geometrically consistent with one shared circle. That's
what lets random noise get rejected instead of flooding every tile
with false positives.

**However, there's a real gap**: `vote_threshold_frac` is relative to
that image's *own* maximum vote, not an absolute or
background-subtracted significance test. On a faint/sparse tile, a
handful of coincidentally-aligned random points can still be "60% of
max" and get reported as a candidate — nothing currently checks that
against expected background (e.g. a Poisson comparison of counts in
the predicted annulus vs. counts in a similar-area region outside
it). The only place an absolute photon-count check exists today is
`count_photons_on_circle`, used manually in the MAXI J1348-630
validation cell's "Test A" radial profile — it is not wired into
`run_halo_search`/`non_max_suppress_arcmin_results` as an automated
gate.

**Open follow-up**: consider adding an automated significance check
(e.g. total counts within `[theta_min, theta_max]` around a candidate
center vs. expected background, tested against Poisson statistics)
before a Hough candidate is reported as a real detection, rather than
relying on relative vote rank alone.
