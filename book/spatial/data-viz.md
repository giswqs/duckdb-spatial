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
# Interactive Data Visualization

## Introduction

## Learning Objectives

## Sample Datasets

## Installation and Setup

```{code-cell} ipython3
# %pip install -U leafmap
```

```{code-cell} ipython3
import duckdb
import leafmap.maplibregl as leafmap
```

## Downloading Sample Data

```{code-cell} ipython3
url = "https://data.gishub.org/duckdb/nyc_data.db.zip"
leafmap.download_file(url, unzip=True)
```

## Connecting to DuckDB and Loading Extensions

```{code-cell} ipython3
con = duckdb.connect("nyc_data.db")
```

### Installing and Loading the Spatial Extension

```{code-cell} ipython3
con.install_extension("spatial")
con.load_extension("spatial")
```

### Exploring the Database Contents

```{code-cell} ipython3
con.sql("SHOW TABLES;")
```

## Visualizing Point Data

### Querying and Converting Point Data

```{code-cell} ipython3
subway_stations_df = con.sql(
    "SELECT * EXCLUDE geom, ST_AsText(geom) as geometry FROM nyc_subway_stations"
).df()
subway_stations_df.head()
```

### Transforming to GeoDataFrame with CRS Conversion

```{code-cell} ipython3
subway_stations_gdf = leafmap.df_to_gdf(
    subway_stations_df, src_crs="EPSG:26918", dst_crs="EPSG:4326"
)
subway_stations_gdf.head()
```

### Creating a Basic Point Map

```{code-cell} ipython3
m = leafmap.Map(style="liberty")
m.add_vector(subway_stations_gdf, name="Subway Stations")
m
```

### Customizing Point Symbology

```{code-cell} ipython3
m = leafmap.Map(style="liberty")
paint = {
    "circle-radius": 5,
    "circle-color": "#ff0000",
    "circle-opacity": 0.5,
    "circle-stroke-color": "#ffffff",
    "circle-stroke-width": 1,
}
m.add_vector(
    subway_stations_gdf, layer_type="circle", paint=paint, name="Subway Stations"
)
m
```

## Visualizing Line Data

### Querying and Converting Line Data

```{code-cell} ipython3
streets_df = con.sql(
    "SELECT * EXCLUDE geom, ST_AsText(geom) as geometry FROM nyc_streets"
).df()
streets_df.head()
```

### Transforming Streets to GeoDataFrame

```{code-cell} ipython3
streets_gdf = leafmap.df_to_gdf(streets_df, src_crs="EPSG:26918", dst_crs="EPSG:4326")
streets_gdf.head()
```

### Creating a Basic Line Map

```{code-cell} ipython3
m = leafmap.Map(style="liberty")
m.add_vector(streets_gdf, name="Streets")
m
```

### Customizing Line Symbology

```{code-cell} ipython3
m = leafmap.Map(style="liberty")
paint = {"line-color": "#ff0000", "line-width": 2, "line-opacity": 0.5}
m.add_vector(streets_gdf, layer_type="line", paint=paint, name="Streets")
m
```

## Visualizing Polygon Data

### Querying and Converting Polygon Data

```{code-cell} ipython3
census_blocks_df = con.sql(
    "SELECT * EXCLUDE geom, ST_AsText(geom) as geometry FROM nyc_census_blocks"
).df()
census_blocks_df.head()
```

### Transforming Census Blocks to GeoDataFrame

```{code-cell} ipython3
census_blocks_gdf = leafmap.df_to_gdf(
    census_blocks_df, src_crs="EPSG:26918", dst_crs="EPSG:4326"
)
census_blocks_gdf.head()
```

### Creating a Basic Polygon Map

```{code-cell} ipython3
m = leafmap.Map(style="liberty")
m.add_vector(census_blocks_gdf, name="Census Blocks")
m
```

### Customizing Polygon Symbology

```{code-cell} ipython3
m = leafmap.Map(style="liberty")
paint = {
    "fill-color": "#ff0000",
    "fill-opacity": 0.5,
    "fill-outline-color": "#ffffff",
}
m.add_vector(census_blocks_gdf, layer_type="fill", paint=paint, name="Census Blocks")
m
```

### Creating Choropleth Maps

```{code-cell} ipython3
m = leafmap.Map(style="positron")
m.add_data(census_blocks_gdf, column="POPN_TOTAL")
m
```

### Creating 3D Extrusion Maps

```{code-cell} ipython3
m = leafmap.Map(
    style="positron", pitch=60, bearing=73.5, center=(-73.923449, 40.694178), zoom=11
)
m.add_data(
    census_blocks_gdf,
    column="POPN_TOTAL",
    cmap="coolwarm",
    extrude=True,
    fit_bounds=False,
)
m
```

## Key Takeaways

## Exercises

### Exercise 1: Basic Point Visualization

```{code-cell} ipython3

```

### Exercise 2: Attribute-Based Filtering and Visualization

```{code-cell} ipython3

```

### Exercise 3: Line Network Visualization

```{code-cell} ipython3

```

### Exercise 4: Basic Polygon Visualization

```{code-cell} ipython3

```

### Exercise 5: Choropleth Mapping

```{code-cell} ipython3

```

### Exercise 6: Custom Choropleth Color Schemes

```{code-cell} ipython3

```

### Exercise 7: 3D Extrusion Visualization

```{code-cell} ipython3

```

### Exercise 8: Multi-Layer Visualization

```{code-cell} ipython3

```

### Exercise 9: Spatial Query Visualization

```{code-cell} ipython3

```

### Exercise 10: Custom Analysis and Visualization Pipeline

```{code-cell} ipython3

```
