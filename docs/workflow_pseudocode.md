# EarthExplorer API — Workflow Pseudocode

Sequential description of every step in the notebook, its objective, the strategy employed, and the packages used.

---

## Step 0 — Environment Setup

**Objective:** Install all required Python packages and configure the runtime environment.

**Strategy:** A single `pip install` cell installs every dependency up front so subsequent cells can import freely. A `.env` file stores credentials outside source control.

**Packages:**
- `requests` — synchronous HTTP client for all M2M API calls
- `aiohttp` / `aiofiles` — asynchronous HTTP downloads and file I/O
- `tqdm` — progress bars (auto widget variant for Jupyter)
- `shapely` — geometric operations on AOI and scene footprints
- `python-dotenv` — load environment variables from `.env`
- `leafmap` — geospatial mapping library (installed here; plain `folium` is ultimately used for display)

---

## Step 0b — Imports & Helpers

**Objective:** Import all libraries and define the core API request helper.

**Strategy:**
1. Import standard-library modules (`json`, `time`, `os`, `pathlib`, `datetime`, `collections`, `asyncio`, `getpass`).
2. Import third-party modules (`requests`, `aiohttp`, `aiofiles`, `tqdm`, `shapely`, `dotenv`, `folium`).
3. Call `load_dotenv()` to inject `.env` values into `os.environ`.
4. Define `M2M_BASE_URL` pointing to the USGS stable JSON endpoint.
5. Define `m2m_request(endpoint, payload, api_key)` — a reusable wrapper that:
   - Builds the full URL from `M2M_BASE_URL + endpoint`.
   - Attaches the `X-Auth-Token` header when an `api_key` is supplied.
   - POSTs the JSON payload, raises on HTTP errors, and raises on M2M-level `errorCode`.
   - Returns the parsed JSON response.

**Packages:** `requests`, `python-dotenv`

---

## Step 1 — Authenticate with the M2M API

**Objective:** Obtain a session API key from USGS EROS.

**Strategy:**
1. Read `USGS_USERNAME` and `USGSTOKEN` from environment variables (populated by `load_dotenv()`).
2. If either is missing, prompt the user interactively (`input()` / `getpass.getpass()`).
3. POST to the `login-token` endpoint with `{ username, token }`.
4. Store the returned session key in `API_KEY` for all subsequent calls.

**Packages:** `python-dotenv`, `getpass` (stdlib)

---

## Step 2 — Define the Area of Interest (AOI)

**Objective:** Load or define a GeoJSON geometry to constrain all searches spatially.

**Strategy:**
1. Check for a local GeoJSON file (`GEOJSON_FILE`). If it exists, load it and extract the geometry (handles `FeatureCollection`, `Feature`, or bare `Geometry` types).
2. If no file is found, fall back to a hardcoded polygon (Denver, CO bounding box).
3. Parse the geometry with `shapely.geometry.shape()` and print bounds / area for verification.
4. Wrap the geometry in an M2M `SpatialFilterGeoJson` dict via `build_spatial_filter()`.

**Packages:** `shapely` (geometry validation & bounds), `json` / `pathlib` (stdlib)

---

## Step 3 — Search Available Datasets (Wildcard)

**Objective:** Discover which USGS datasets have coverage within the AOI.

**Strategy:**
1. Set a wildcard pattern (`DATASET_WILDCARD`, default `"naip"`).
2. POST to `dataset-search` with the wildcard and the spatial filter.
3. Print a formatted table of matching datasets (name, alias, abstract snippet).
4. Store results in `available_datasets` dict keyed by `datasetName` (falling back to `datasetAlias` via the `_ds_key()` helper).

**Packages:** `requests` (via `m2m_request`)

---

## Step 4 — Scene Search & Summary

**Objective:** Retrieve all scenes in the chosen dataset that intersect the AOI and print a statistical summary.

**Strategy:**
1. Set `DATASET_NAME` (e.g. `"naip"`).
2. `search_all_scenes()` paginates through the `scene-search` endpoint (100 results per page) until all scenes are collected. A running counter is printed during pagination.
3. `print_scene_summary()` computes and displays:
   - Total scene count
   - Acquisition date range and scenes-per-year histogram
   - Cloud-cover statistics (min, max, mean, count < 20%)

**Packages:** `requests`, `datetime`, `collections.Counter` (stdlib)

---

## Step 5 — Filter by Date Range

**Objective:** Narrow the scene list to a user-defined acquisition window.

**Strategy:**
1. Define `DATE_START` and `DATE_END` (e.g. 2019-01-01 → 2023-12-31).
2. Re-run `search_all_scenes()` with an added `acquisitionFilter` in the scene filter payload.
3. Print an updated summary via `print_scene_summary()` so the user can compare counts before and after filtering.
4. Store the filtered list as `scenes_date_filtered`.

**Packages:** `requests`

---

## Step 5b — Inspect Metadata for the Most Recent Scene

**Objective:** Display the full metadata record of one scene so the user can understand what fields are available.

**Strategy:**
1. Sort `scenes_date_filtered` by acquisition date; pick the most recent.
2. Print highlighted top-level fields (`entityId`, `displayId`, `cloudCover`, `temporalCoverage`, `spatialCoverage`, etc.).
3. Print browse/thumbnail URLs.
4. Iterate the `metadata` array and print every key–value pair.
5. Dump any remaining top-level keys not already shown.

**Packages:** `json`, `datetime` (stdlib)

---

## Step 5c — Spatial Bounds of the Most Recent Scene

**Objective:** Extract and display the geographic bounding box of the most-recent scene.

**Strategy:**
1. `get_spatial_bounds()` checks `spatialBounds` then `spatialCoverage` on the scene dict.
2. If found, print the raw JSON.
3. If absent, fall back to scanning the `metadata` array for coordinate-related fields (corner, latitude, longitude, north/south/east/west).

**Packages:** `json` (stdlib)

---

## Step 5d — Visualize Scene Footprints on a Map

**Objective:** Render an interactive map showing the AOI outline and every filtered scene's bounding polygon.

**Strategy:**
1. `scene_bounds_to_geojson()` converts each scene's spatial bounds into a GeoJSON `FeatureCollection` of Polygon features. Handles both raw GeoJSON geometries and corner-coordinate dicts.
2. Compute the map center and fit-bounds from the union of AOI + all scene geometries using Shapely's `unary_union`.
3. Create a `folium.Map`, add the AOI as a red outline layer and scene footprints as blue polygons with tooltip popups (Display ID, acquisition date, cloud cover).
4. Save the map to `scene_footprints_map.html` and display it inside an `IFrame` (works in VS Code without needing "Trust Notebook").

**Packages:** `folium`, `shapely` (`shape`, `unary_union`), `IPython.display` (`IFrame`)

---

## Step 6 — Query Download Options

**Objective:** Determine which downloadable file products exist for the filtered scenes and whether they are immediately available.

**Strategy:**
1. Collect all Entity IDs from `scenes_date_filtered`.
2. POST to `download-options` with the dataset name and entity IDs.
3. Print a formatted table for every product: entity ID, product name, availability flag, download system (`dds` vs `tram`), file size.
4. Separate products into `downloadable_products` (available DDS) and count TRAM products for later.

**Packages:** `requests`

---

## Step 7 — Request & Download DDS Products

**Objective:** Download all immediately-available (DDS) products, optionally filtered by product type.

**Strategy:**
1. **Product-type filter** — The user sets `PRODUCT_TYPE` to `"full"`, `"compressed"`, or `"both"`. The helper `_match_product_type()` filters products by checking for keywords like "full" or "compress"/"reduced" in the product name.
2. Split download-options results into `dds_products` (DDS, available, matching filter) and `tram_products` (everything else).
3. POST to `download-request` with the selected products and a timestamped label.
4. Parse the response for `availableDownloads` (ready URLs) and `duplicateProducts`.
5. **Async parallel downloader:**
   - `download_file()` streams a single URL to disk with chunked writes and progress updates.
   - `download_all()` runs up to `MAX_CONCURRENT_DOWNLOADS` (default 3–5) tasks concurrently using an `asyncio.Semaphore` and `aiohttp.ClientSession`.
   - A single `tqdm` progress bar tracks total bytes.
6. After completion, report successes and failures.

**Packages:** `aiohttp`, `aiofiles`, `asyncio`, `tqdm`, `pathlib` (stdlib)

---

## Step 8 — Verify DDS Downloads

**Objective:** Quick sanity check that downloaded files exist and look reasonable.

**Strategy:**
1. List all files in `DOWNLOAD_DIR`.
2. Print each filename with its size in MB.
3. Print a total size line.
4. Report how many TRAM products remain for Step 9.

**Packages:** `pathlib` (stdlib)

---

## Step 9 — Request & Download TRAM (Tape-Archived) Products

**Objective:** Order tape-archived products, wait for them to be staged to disk, then download.

**Strategy:**
1. If `tram_products` is empty, skip with a message.
2. POST to `download-request` with all TRAM products and a timestamped `TRAM_LABEL`.
3. Any products already staged appear in `availableDownloads`; the rest go to `preparingDownloads`.
4. **Polling loop** — call `download-retrieve` with the same label every `POLL_INTERVAL` seconds (default 30 s), up to `MAX_POLLS` attempts (default 60 ≈ 30 min). As files finish staging they transition to `availableDownloads`; the loop ends when `preparingDownloads` is empty.
5. Collect all staged URLs and download them with the same `download_all()` async downloader from Step 7.
6. Report successes and failures; advise the user to re-run the cell or check the USGS web UI if staging is still incomplete.

**Packages:** `aiohttp`, `aiofiles`, `asyncio`, `tqdm`, `time` (stdlib)

---

## Package Summary

| Package | Role |
|---|---|
| `requests` | Synchronous HTTP — all M2M API calls |
| `aiohttp` | Asynchronous HTTP — parallel file downloads |
| `aiofiles` | Async file writes during download |
| `tqdm` | Progress bars for pagination and downloads |
| `shapely` | AOI validation, bounds computation, geometry union |
| `folium` | Interactive map rendering (scene footprints + AOI) |
| `python-dotenv` | Load credentials from `.env` |
| `leafmap` | Installed as a dependency; folium used directly for VS Code compatibility |
| `getpass` | Secure interactive password/token prompt (stdlib) |
| `pathlib` | File-system path handling (stdlib) |
| `asyncio` | Concurrency primitives for parallel downloads (stdlib) |
| `json` | JSON serialization / deserialization (stdlib) |
| `IPython.display` | `IFrame` rendering of saved HTML maps in VS Code |
