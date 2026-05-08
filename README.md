# EarthExplorer API

This repository provides tools and workflows for interacting with the USGS EarthExplorer Machine-to-Machine (M2M) API to search, download, and process satellite imagery, with a focus on NAIP (National Agriculture Imagery Program) data.

## Overview

The repository contains Jupyter notebooks that demonstrate a complete workflow for:

1. Authenticating with the USGS M2M API
2. Searching for available datasets and scenes within an Area of Interest (AOI)
3. Filtering scenes by date, cloud cover, and other metadata
4. Downloading imagery asynchronously
5. Processing downloaded NAIP data into Cloud Optimized GeoTIFFs (COGs)
6. Building XYZ tile pyramids for web visualization
7. Creating aggregate mosaics and visualizations

## Prerequisites

- Python 3.8+
- USGS EROS account with M2M access enabled: https://ers.cr.usgs.gov/profile/access
- Application Token from your EROS profile
- GDAL with Python bindings for processing notebooks

## Repository Structure

### Notebooks

- **[EarthExplorer_API.ipynb](notebooks/EarthExplorer_API.ipynb)**: Main workflow for searching and downloading satellite imagery using the USGS M2M API
- **[naip_cog_builder.ipynb](notebooks/naip_cog_builder.ipynb)**: Convert unzipped NAIP imagery into Cloud Optimized GeoTIFFs (COGs) for RGB and IRG visualization modes
- **[naip_processed_inventory_and_xyz_tiles.ipynb](notebooks/naip_processed_inventory_and_xyz_tiles.ipynb)**: Scan NAIP directories, compute inventory, and build XYZ tile pyramids
- **[naip_zip_organizer.ipynb](notebooks/naip_zip_organizer.ipynb)**: Organize downloaded NAIP zip files
- **[zips_to_sdr_prep.ipynb](notebooks/zips_to_sdr_prep.ipynb)**: Prepare zip files for SDR (Standard Data Record) processing

### Documentation

- **[docs/workflow_pseudocode.md](docs/workflow_pseudocode.md)**: Detailed pseudocode describing each step in the main API workflow
- **[docs/scene_footprints_map.html](docs/scene_footprints_map.html)**: Interactive map showing scene footprints (also generated in notebooks)

### Wildfire Data

The `wildfires/` folder contains GeoJSON files with fire perimeter data for various California wildfires in 2020, which can be used as Areas of Interest:

- Castle Fire
- Creek Fire
- CZU Lightning Complex
- North Complex
- Mortalitree (related to CZU)

Each fire has both a `.geojson` file and a `.qmd` QGIS project file.

### Other Files

- `recon_doc.qgz`: QGIS project file for reconnaissance/documentation
- `.gitignore`: Ignores downloads and environment files

## Getting Started

1. Clone this repository
2. Create a `.env` file with your USGS credentials:
   ```
   USGS_USERNAME=your_username
   USGSTOKEN=your_application_token
   ```
3. Install dependencies: `pip install requests aiohttp aiofiles tqdm shapely python-dotenv leafmap`
4. For processing notebooks, install GDAL: `pip install GDAL`
5. Start with [EarthExplorer_API.ipynb](notebooks/EarthExplorer_API.ipynb)

## Key Features

- **Asynchronous Downloads**: Efficient bulk downloading of satellite imagery
- **COG Optimization**: Advanced compression profiles for smaller file sizes (see repo memory for ZSTD optimization details)
- **Web-Ready Tiles**: XYZ tile generation for web mapping applications
- **Flexible AOIs**: Support for GeoJSON polygons, including wildfire perimeters
- **Metadata Filtering**: Cloud cover, date ranges, and other scene attributes

## License

[Add license information if applicable]

## Contributing

[Add contribution guidelines if applicable]