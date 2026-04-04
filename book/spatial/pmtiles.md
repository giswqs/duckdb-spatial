---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---
# Working with Vector Tiles and PMTiles

## Introduction

## Learning Objectives

## Sample Datasets

## Installation and Setup

```{code-cell} ipython3
# %pip install -U "leafmap[pmtiles]"
```

```{code-cell} ipython3
import duckdb
import leafmap.maplibregl as leafmap
```

## Visualizing Vector Tiles Directly from Files

### Configuring JupyterHub Access (Optional)

```{code-cell} ipython3
# leafmap.configure_jupyterhub("https://your-jupyterhub-domain.com")
```

### Visualizing Data from Vector Files

```{code-cell} ipython3
url = "https://data.source.coop/giswqs/nwi/wetlands/RI_Wetlands.parquet"
filepath = leafmap.download_file(url)
```

```{code-cell} ipython3
m = leafmap.Map(style="liberty", height="600px")
m.add_basemap("Esri.WorldImagery")
m.add_duckdb_layer(
    data=filepath,
    layer_name="wetlands",
    layer_type="fill",
    paint={"fill-color": "#3388ff"},
    opacity=0.5,
    fit_bounds=True,
    use_view=False,
    min_zoom=None,
    quiet=False,
)
m
```

### Visualizing Data from Existing DuckDB Databases

```{code-cell} ipython3
url = "https://data.gishub.org/duckdb/nyc_data.db.zip"
leafmap.download_file(url, unzip=True)
```

```{code-cell} ipython3
db_path = "nyc_data.db"
con = duckdb.connect(db_path)
```

```{code-cell} ipython3
con.install_extension("spatial")
con.load_extension("spatial")
```

```{code-cell} ipython3
con.sql("SHOW TABLES;")
```

```{code-cell} ipython3
con.close()
```

```{code-cell} ipython3
m = leafmap.Map(
    center=[-73.9031255, 40.7127753], zoom=9, style="positron", height="600px"
)
m.add_basemap("Esri.WorldImagery")
m.add_duckdb_layer(
    database_path=db_path,
    table_name="nyc_census_blocks",
    layer_name="Census Blocks",
    layer_type="fill",
    paint={"fill-color": "#3388ff", "fill-outline-color": "#ffffff"},
    opacity=0.5,
    fit_bounds=False,
    src_crs="EPSG:26918",
    quiet=True,
)
m
```

## Converting Vector Data to PMTiles

### Understanding Coordinate Transformations for PMTiles

### Exporting a Global Cities Dataset to PMTiles

```{code-cell} ipython3
con = duckdb.connect()
con.install_extension("spatial")
con.load_extension("spatial")
```

```{code-cell} ipython3
con.sql(
    """
COPY (
    SELECT * EXCLUDE (geometry),
           ST_Transform(geometry, 'EPSG:4326', 'EPSG:3857', true) AS geometry
    FROM 'https://data.gishub.org/duckdb/cities.parquet'
)
TO 'cities.pmtiles'
WITH (
    FORMAT GDAL,
    DRIVER 'PMTiles',
    LAYER_CREATION_OPTIONS ('MINZOOM=0', 'MAXZOOM=14')
)
"""
)
```

## Visualizing PMTiles

### Visualizing Local PMTiles with Martin Tile Server

```{code-cell} ipython3
process = leafmap.start_martin(pmtiles=["cities.pmtiles"])
```

```{code-cell} ipython3
m = leafmap.Map(style="positron")
url = "http://localhost:3000/cities"
paint = {
    "circle-color": "#3388ff",
    "circle-radius": 4,
    "circle-opacity": 0.5,
    "circle-stroke-color": "#ffffff",
    "circle-stroke-width": 1,
}
m.add_vector_tile(
    url, layer_id="cities", layer_type="circle", name="World Cities", paint=paint
)
m
```

```{code-cell} ipython3
leafmap.stop_martin(process)
```

### Visualizing Cloud-Hosted PMTiles

```{code-cell} ipython3
release = "2025-10-22"
url = f"https://overturemaps-tiles-us-west-2-beta.s3.amazonaws.com/{release}/buildings.pmtiles"
print(url)
```

### Creating 3D Building Visualizations

```{code-cell} ipython3
m = leafmap.Map(
    center=[-74.0095, 40.7046], zoom=15, pitch=60, bearing=-17, style="positron"
)
m.add_basemap("OpenStreetMap.Mapnik")
m.add_basemap("Esri.WorldImagery", visible=False)

value_color_pairs = [0, "lightgray", 200, "royalblue", 400, "lightblue"]
style = {
    "layers": [
        {
            "id": "Building",
            "source": "buildings",
            "source-layer": "building",
            "type": "fill-extrusion",
            "filter": [
                ">",
                ["get", "height"],
                0,
            ],
            "paint": {
                "fill-extrusion-color": [
                    "interpolate",
                    ["linear"],
                    ["get", "height"],
                ]
                + value_color_pairs,
                "fill-extrusion-height": ["*", ["get", "height"], 1],
            },
        },
        {
            "id": "Building-part",
            "source": "buildings",
            "source-layer": "building_part",
            "type": "fill-extrusion",
            "filter": [
                ">",
                ["get", "height"],
                0,
            ],
            "paint": {
                "fill-extrusion-color": [
                    "interpolate",
                    ["linear"],
                    ["get", "height"],
                ]
                + value_color_pairs,
                "fill-extrusion-height": ["*", ["get", "height"], 1],
            },
        },
    ],
}

m.add_pmtiles(
    url,
    style=style,
    visible=True,
    opacity=1.0,
    tooltip=True,
    fit_bounds=False,
)
m
```

### Simplified 3D Building Visualization

```{code-cell} ipython3
m = leafmap.Map(
    center=[-74.0095, 40.7046], zoom=15, pitch=60, bearing=-17, style="positron"
)
m.add_basemap("OpenStreetMap.Mapnik")
m.add_basemap("Esri.WorldImagery", visible=False)
m.add_overture_3d_buildings(
    values=[0, 200, 400], colors=["lightgray", "royalblue", "lightblue"]
)
m
```

### Visualizing 2D Building Footprints

```{code-cell} ipython3
m = leafmap.Map(center=[-74.0095, 40.7046], zoom=15)
m.add_basemap("OpenStreetMap.Mapnik")
m.add_basemap("Esri.WorldImagery", visible=True)
m.add_overture_data(theme="buildings", opacity=0.8)
m
```

```{code-cell} ipython3
m = leafmap.Map(center=[-74.0095, 40.7046], zoom=16)
m.add_basemap("OpenStreetMap.Mapnik")
m.add_basemap("Esri.WorldImagery", visible=True)
m.add_overture_buildings(type="line", line_color="#ff0000", line_width=2, opacity=0.7)
m
```

### Visualizing Overture Transportation Networks

```{code-cell} ipython3
m = leafmap.Map(center=[-74.0095, 40.7046], zoom=16)
m.add_basemap("Esri.WorldImagery", visible=True)
m.add_overture_data(theme="transportation", opacity=0.8)
m
```

### Visualizing Overture Places

```{code-cell} ipython3
m = leafmap.Map(center=[-74.0095, 40.7046], zoom=16)
m.add_basemap("Esri.WorldImagery", visible=True)
m.add_overture_data(theme="places", opacity=0.8)
m
```

## Key Takeaways

## Exercises

### Exercise 1: Visualizing Local Vector Files with On-the-Fly Tiling

```{code-cell} ipython3

```

### Exercise 2: Visualizing Data from DuckDB Database

```{code-cell} ipython3

```

### Exercise 3: Converting Spatial Data to PMTiles

```{code-cell} ipython3

```

### Exercise 4: Serving and Visualizing Local PMTiles

```{code-cell} ipython3

```

### Exercise 5: Exporting NYC Data to PMTiles

```{code-cell} ipython3

```

### Exercise 6: Multi-Layer PMTiles Visualization

```{code-cell} ipython3

```

### Exercise 7: Visualizing Cloud-Hosted Overture Data

```{code-cell} ipython3

```

### Exercise 8: Custom 3D Building Visualization

```{code-cell} ipython3

```

### Exercise 9: Overture Transportation Network Analysis

```{code-cell} ipython3

```

### Exercise 10: Complete Spatial Analysis to PMTiles Workflow

```{code-cell} ipython3

```
