# Changes from upstream

Upstream: https://github.com/ASFHyP3/MultiRTC, branch `main` at commit 0e3609d0837b1f93e5555d1d919fb827cb3cd419 (2025-08-25), by Forrest Williams (Alaska Satellite Facility), BSD 3-Clause License (`LICENSE`, kept unchanged). That repository has no tags; the same commit is tagged `v0.5.2` in the author's own repository, `forrestfwilliams/multirtc` on GitHub, and `CHANGELOG.md` ends at 0.5.2.

This copy starts from upstream's file tree at 0e3609d without its git history. It is unchanged except for the items below. Left out: `.github/` (CI workflows, issue templates, CODEOWNERS and the Dependabot config).

## 2026-10-25: DEM tiles from DEM v1.2

`src/multirtc/dem.py` fetched DEM tiles from `https://nisar.asf.earthdatacloud.nasa.gov/STATIC/DEM/v1.1/EPSG4326/...`. Those addresses now return HTTP 404, so every Sentinel-1 RTC run stopped while fetching the DEM. Version 1.2 of the same Copernicus-based DEM is at `.../NISAR/DEM/v1.2/EPSG4326`, with an extra latitude-band folder and a `_C01` suffix on the tile names (for example `N10/N10_E120/DEM_N17_00_E121_00_C01.tif`). The tile URL builder and its test now use that layout. This is the change the author made in his own repository (commit ec62897 for `dem.py`, the test as of commit 2ca5956, released there as v0.5.4). The same tiles are also served under `.../STATIC/DEM/v1.2/` with identical bytes.

Results: the RTC still matches the official OPERA RTC-S1 product (see Check). Small differences from products made with DEM v1.1 are possible where the two DEM releases differ.

## 2026-10-25: a failed DEM tile download stops the run

`download_opera_dem_for_footprint` started the tile downloads with `executor.map` but never read the results, so an error raised in a download thread (such as the `HTTPError` from a 404) was dropped, and the run failed later with a misleading `dem.vrt: No such file or directory` from GDAL. The results are now read with `list(...)`, so the first failed download stops the run with its own error, which names the URL (`404 Client Error: Not Found for url: https://...`). No change to results when the tiles download.

## 2026-10-25: isce3 pinned to 0.22.1

`environment.yml` left isce3 unpinned, so a new environment gets the newest release (0.25.22 in October 2026). On the check burst that version gives gamma0 that differs from the official OPERA RTC-S1 product by 0.27 dB on average, up to about 1 dB on mountain slopes in a fore-slope/back-slope pattern, with 53% of pixels within 0.1 dB. Flat ground and geolocation agree.

The OPERA products were made with isce3 0.15.0, which reproduces them (0.0045 dB), but the `multirtc` command cannot start with it: `__main__.py` imports `multimetric/point_target.py` for every subcommand, and that module needs `isce3.cal.point_target_info.analyze_point_target_chip`, which isce3 added in 0.22.0. isce3 0.22.1, the oldest release with that function, gives the same agreement as 0.15.0 (see Check), so `environment.yml` now pins `isce3=0.22.1`, with a comment, and the README says why. isce3 releases between 0.22.1 and 0.25.22 were not tried.

Results: these change relative to an unpinned install made today; they go back to what OPERA publishes.

## 2026-10-25: Python 3.11 or newer

`from_isce_datetime` in `src/multirtc/base.py` passes isce3's `DateTime.isoformat()` string, which has nine fractional digits (`2024-10-27T09:59:58.375861000`), to `datetime.fromisoformat`. Python 3.10 rejects it (`ValueError: Invalid isoformat string`), so the Sentinel-1 RTC stopped on 3.10; Python 3.11 and newer accept it. `requires-python` in `pyproject.toml`, the classifier and the `environment.yml` floor are now 3.11. No change to results.

## Check

On 2026-10-25, from a fresh clone of this repository, following the README's developer setup on Linux x86_64 (2 cores, 7 GB):

```sh
conda env create -f environment.yml   # or: conda env update -n <env> -f environment.yml
conda activate multirtc
python -m pip install -e .
pytest
multirtc rtc S1 S1_146160_IW1_20241029T095958_VV_592B-BURST --resolution 30 --work-dir <work dir>
```

- The environment solved to Python 3.12.15, isce3 0.22.1 (conda-forge `cuda` build; its CPU code is what runs on a machine without a GPU), GDAL 3.9.3, numpy 1.26.4, s1reader 0.2.5, burst2safe 2.0.3. Creating it took 17 minutes, most of it in the conda solver, with 4.5 GB peak memory. `pip check` reports one mismatch that does not affect the RTC: imagecodecs 2026.8.16 asks for numpy 2.1 or newer.
- `pytest`: 2 passed.
- A tile URL that returns 404 now stops `download_opera_dem_for_footprint` with `404 Client Error: Not Found for url: ...` naming the tile.
- `multirtc rtc`: exit 0 in 4 min 7 s with 1.82 GB peak memory, downloads included (the 119 MB one-burst SAFE, a 4.7 MB orbit file and two DEM v1.2 tiles; about 0.6 GB in the work folder).
- Against the official OPERA RTC-S1 product `OPERA_L2_RTC-S1_T069-146160-IW1_20241029T095958Z_20241029T184129Z_S1A_30_v1.0` (ASF DAAC) on the shared 30 m UTM grid, VV gamma0 over the 1,940,392 pixels valid in both: mean absolute difference 0.0046 dB, median 0.00001 dB, 99.987% of pixels within 0.1 dB and 99.994% within 0.5 dB, r = 0.99999; the layover/shadow mask agrees on 99.988% of pixels.
