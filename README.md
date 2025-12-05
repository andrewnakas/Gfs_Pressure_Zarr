# GFS Pressure Levels → Zarr

This repository automates fetching the latest GFS pressure‑level GRIB2 data, converts it to a consolidated Zarr store, and publishes the result to a lightweight `data` branch on a 6‑hour cadence (matching the GFS 00/06/12/18 UTC cycles).

## How it works
- GitHub Actions runs at 03:30, 09:30, 15:30, and 21:30 UTC each day (starting 2025‑12‑03). These times trail the nominal GFS cycle availability by ~3–4 hours to reduce 404s while still staying current.
- The workflow downloads the most recent available cycle from the public NOAA GFS bucket (`noaa-gfs-bdp-pds`), limited to pressure-level fields.
- GRIB2 files are converted with `xarray+cfgrib` into a single consolidated Zarr store, then zipped (`gfs_latest.zarr.zip`).
- The zipped Zarr and a small metadata file are force-pushed to the `data` branch so only the latest dataset exists; history is rewritten each run to avoid repo bloat.
- Forecast hours follow the common GFS cadence: 1-hourly to 120h, then 3-hourly to 384h (16 days).

## Quick start
- Grab the latest data directly from the `data` branch:
  ```bash
  git fetch origin data:data
  git checkout data
  unzip gfs_latest.zarr.zip
  ```
- The unzipped folder (`gfs_latest.zarr/`) is a standard consolidated Zarr store ready for `xarray.open_zarr`.

## Local run
```bash
pip install -r requirements.txt
python scripts/update_gfs_zarr.py --output ./gfs_latest.zarr --zip
```

## Environment variables
You can override defaults when running locally or in CI:
- `FORECAST_HOURS`: space separated forecast hours (default: 1-hourly 0–120h plus 3-hourly 123–384h).
- `GRID`: grid resolution suffix (`0p25` or `0p50`, default `0p25` to match the dynamical.org catalog).
- `CYCLE_OFFSET_HOURS`: hours to back off from current UTC when choosing the latest cycle (default `3`).
- `PARAM_SHORTNAMES`: space separated GRIB `shortName` list used as a fallback when cfgrib encounters mixed vertical levels. Default empty = auto-discover all pressure-level variables.
- `LEVELS_HPA`: pressure levels to keep (default: empty = keep all pressure levels present).
- `DTYPE`: output dtype for data variables (default `float16` to reduce size).
- `MAX_ZARR_BYTES`: safety cutoff in bytes for the zipped store (default ~1.9 GB); run aborts if exceeded to avoid GitHub/LFS push failures.
- `BASE_URL`: source bucket (default `https://noaa-gfs-bdp-pds.s3.amazonaws.com`).
- `OUTPUT_ZARR`: output directory path (default `gfs_latest.zarr`).
- `ZIP_OUTPUT`: `1` to also emit `OUTPUT_ZARR.zip` (workflow uses this).

## Caveats
- Full pressure-level GFS files are large; even zipped they may exceed GitHub's normal file limits, so the workflow uses Git LFS on the `data` branch. Ensure LFS is enabled on your clone when pulling data.
- Defaults aim to mirror the dynamical.org pressure-level catalog: 0.25° grid, hourly→120h then 3-hourly→384h, all pressure-level variables auto-discovered, float16 to compress size, and a 1.9 GB safety cap. If a run exceeds the cap, it will fail rather than push an oversize object.
- The script falls back to the previous cycle if the newest is unavailable, up to three cycles back.
- Only the latest dataset is retained; old data is removed by rewriting the `data` branch history each run.
- Some GRIB variables only exist on a reduced pressure level set; cfgrib can choke on those. The script retries per-variable and keeps only the `PARAM_SHORTNAMES` list when needed to avoid failures.
