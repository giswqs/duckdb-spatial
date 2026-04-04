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
# Exporting and Converting Spatial Data

## Introduction

## Learning Objectives

## Sample Datasets

## Installation and Setup

```{code-cell} ipython3
# %pip install duckdb pandas
```

### Library Import and Initial Setup

```{code-cell} ipython3
import duckdb
import pandas as pd
```

```{code-cell} ipython3
print(f"DuckDB version: {duckdb.__version__}")
print(f"Pandas version: {pd.__version__}")
```

## Installing and Loading Extensions

```{code-cell} ipython3
con = duckdb.connect()
```

```{code-cell} ipython3
con.install_extension("httpfs")
con.load_extension("httpfs")
```

```{code-cell} ipython3
con.install_extension("spatial")
con.load_extension("spatial")
```

```{code-cell} ipython3
con.sql(
    "SELECT extension_name, loaded, installed FROM duckdb_extensions() WHERE extension_name IN ('httpfs', 'spatial')"
).show()
```

## Loading Sample Data

```{code-cell} ipython3
con.sql(
    """
CREATE TABLE IF NOT EXISTS cities AS
SELECT * FROM 'https://data.gishub.org/duckdb/cities.parquet'
"""
)
```

```{code-cell} ipython3
con.table("cities").show()
```

```{code-cell} ipython3
con.sql(
    "SELECT COUNT(*) AS city_count, MIN(population) AS min_pop, MAX(population) AS max_pop, AVG(population) AS avg_pop FROM cities"
).show()
```

```{code-cell} ipython3
con.sql("SUMMARIZE SELECT population FROM cities").show()
```

```{code-cell} ipython3
con.sql("SUMMARIZE SELECT population FROM cities").show()
```

## Exporting to Pandas DataFrames

### Basic DataFrame Export

```{code-cell} ipython3
cities_df = con.table("cities").df()
cities_df.head()
```

```{code-cell} ipython3
cities_df.info()
```

### Exporting Filtered Query Results

```{code-cell} ipython3
# Export only large cities in the United States
us_large_cities_df = con.sql(
    """
    SELECT name, population, latitude, longitude
    FROM cities
    WHERE country = 'USA' AND population > 500000
    ORDER BY population DESC
    """
).df()

us_large_cities_df
```

### Working with Geometry Columns in DataFrames

```{code-cell} ipython3
# Export without geometry for statistical analysis
cities_stats_df = con.sql(
    """
    SELECT name, country, population, latitude, longitude
    FROM cities
    """
).df()

cities_stats_df.head()
```

```{code-cell} ipython3
cities_stats_df = con.sql("SELECT * EXCLUDE geometry FROM cities").df()
cities_stats_df.head()
```

### Converting Geometries to Text for DataFrames

```{code-cell} ipython3
cities_with_wkt_df = con.sql(
    """
    SELECT name, country, population,
           ST_AsText(geometry) AS geometry_wkt
    FROM cities
    LIMIT 5
    """
).df()
cities_with_wkt_df
```

### Integrating with GeoPandas for Spatial Operations

```{code-cell} ipython3
# Export with WKT geometry for GeoPandas conversion
df_for_geopandas = con.sql(
    """
    SELECT name, country, population,
           latitude, longitude,
           ST_AsText(geometry) AS wkt
    FROM cities
    """
).df()
```

```{code-cell} ipython3
import geopandas as gpd
from shapely import wkt

gdf = gpd.GeoDataFrame(
    df_for_geopandas, geometry=df_for_geopandas["wkt"].apply(wkt.loads), crs="EPSG:4326"
)
gdf.head()
```

### Performance Considerations for DataFrame Export

## Exporting to CSV Files

### Basic CSV Export

```{code-cell} ipython3
con.sql("COPY cities TO 'cities.csv' (HEADER, DELIMITER ',')")
```

### Streaming Query Results to CSV

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT * FROM cities WHERE country='USA'
    ) TO 'cities_us.csv' (HEADER, DELIMITER ',')
    """
)
```

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT country,
               COUNT(*) AS city_count,
               SUM(population) AS total_population,
               AVG(population) AS avg_population
        FROM cities
        GROUP BY country
        HAVING COUNT(*) > 5
        ORDER BY total_population DESC
    ) TO 'country_stats.csv' (HEADER, DELIMITER ',')
    """
)
```

### Handling Geometries in CSV Exports

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT * EXCLUDE geometry FROM cities
    ) TO 'cities_no_geometry.csv' (HEADER, DELIMITER ',')
    """
)
```

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT name, country, population,
               ST_X(geometry) AS longitude,
               ST_Y(geometry) AS latitude
        FROM cities
    ) TO 'cities_with_coords.csv' (HEADER, DELIMITER ',')
    """
)
```

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT name, country, population,
               ST_AsText(geometry) AS geometry_wkt
        FROM cities
        LIMIT 10
    ) TO 'cities_with_wkt.csv' (HEADER, DELIMITER ',')
    """
)
```

### When to Use CSV Export

## Exporting to JSON Files

### Basic JSON Export

```{code-cell} ipython3
con.sql("COPY cities TO 'cities.json'")
```

### Exporting Filtered Query Results to JSON

```{code-cell} ipython3
con.sql("COPY (SELECT * FROM cities WHERE country='USA') TO 'cities_us.json'")
```

### Handling Geometries in JSON Exports

```{code-cell} ipython3
con.sql("COPY (SELECT * EXCLUDE geometry FROM cities) TO 'cities_attributes.json'")
```

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT name, country, population,
               ST_X(geometry) AS longitude,
               ST_Y(geometry) AS latitude
        FROM cities
    ) TO 'cities_with_coords.json'
    """
)
```

### When to Use JSON Export

## Exporting to Excel Files

### Install the excel extension

### Installing and Loading the Excel Extension

```{code-cell} ipython3
con.install_extension("excel")
con.load_extension("excel")
```

### Basic Excel Export Without Geometries

```{code-cell} ipython3
con.sql(
    "COPY (SELECT * EXCLUDE geometry FROM cities) TO 'cities.xlsx' WITH (FORMAT xlsx, HEADER true)"
)
```

### Including Spatial Information as Text Columns

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT name, country, population,
               ST_X(geometry) AS longitude,
               ST_Y(geometry) AS latitude,
               ST_AsText(geometry) AS wkt
        FROM cities
    ) TO 'cities_with_coords.xlsx' WITH (FORMAT xlsx, HEADER true)
    """
)
```

### Exporting Filtered and Aggregated Results to Excel

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT country,
               COUNT(*) AS city_count,
               SUM(population) AS total_population,
               AVG(population) AS avg_population,
               MAX(population) AS largest_city_pop
        FROM cities
        GROUP BY country
        HAVING COUNT(*) >= 5
        ORDER BY total_population DESC
    ) TO 'country_summary.xlsx' WITH (FORMAT xlsx, HEADER true)
    """
)
```

### When to Use Excel Export

## Exporting to Parquet Files

### Key Advantages of Parquet

### Basic Parquet Export

```{code-cell} ipython3
con.sql("COPY cities TO 'cities.parquet'")
```

### Exporting Filtered Query Results to Parquet

```{code-cell} ipython3
con.sql("COPY (SELECT * FROM cities WHERE country='USA') TO 'cities_us.parquet'")
```

### Partitioned Parquet Writes

```{code-cell} ipython3
con.sql(
    """
    COPY cities TO 'cities_parquet/'
    WITH (FORMAT PARQUET, PARTITION_BY (country))
    """
)
```

## Exporting to GeoJSON Format

### GeoJSON Structure and GDAL Integration

### Basic GeoJSON Export

```{code-cell} ipython3
con.sql("COPY cities TO 'cities.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON')")
```

### Exporting Filtered Subsets to GeoJSON

```{code-cell} ipython3
con.sql(
    "COPY (SELECT * FROM cities WHERE country='USA') TO 'cities_us.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON')"
)
```

```{code-cell} ipython3
con.sql(
    """
    COPY (
        SELECT name, country, population,
               CASE
                   WHEN population > 10000000 THEN 'Megacity'
                   WHEN population > 5000000 THEN 'Large City'
                   ELSE 'City'
               END AS size_category,
               geometry
        FROM cities
        WHERE population > 1000000
    ) TO 'large_cities_categorized.geojson' WITH (FORMAT GDAL, DRIVER 'GeoJSON')
    """
)
```

### Customizing GeoJSON Export Options

```{code-cell} ipython3
con.sql(
    """
    COPY (SELECT * FROM cities WHERE country='USA')
    TO 'cities_us.geojson'
    WITH (FORMAT GDAL, DRIVER 'GeoJSON', LAYER_CREATION_OPTIONS('WRITE_BBOX=YES', 'COORDINATE_PRECISION=5'))
    """
)
```

### When to Use GeoJSON

### Limitations and When to Avoid GeoJSON

## Exporting to Shapefile Format

### Shapefile Limitations and Constraints

### Why GeoPackage is Preferred

### Shapefile Export Syntax

```{code-cell} ipython3
con.sql("COPY cities TO 'cities.shp' WITH (FORMAT GDAL, DRIVER 'ESRI Shapefile')")
```

## Exporting to GeoPackage Format

### Advantages of GeoPackage Over Legacy Formats

### Basic GeoPackage Export

```{code-cell} ipython3
con.sql("COPY cities TO 'cities.gpkg' WITH (FORMAT GDAL, DRIVER 'GPKG')")
```

### Exporting Filtered Subsets to GeoPackage

```{code-cell} ipython3
con.sql(
    "COPY (SELECT * FROM cities WHERE population > 1000000) TO 'major_cities.gpkg' WITH (FORMAT GDAL, DRIVER 'GPKG')"
)
```

### When to Use GeoPackage Export

## Key Takeaways

## Exercises

### Exercise 1: Basic CSV Export with Geometry Handling

```{code-cell} ipython3

```

### Exercise 2: Filtered Exports to Multiple Formats

```{code-cell} ipython3

```

### Exercise 3: DataFrame Export and Manipulation

```{code-cell} ipython3

```

### Exercise 4: Exporting with Spatial Aggregations

```{code-cell} ipython3

```

### Exercise 5: GeoJSON Export for Web Mapping

```{code-cell} ipython3

```

### Exercise 6: GeoPackage Export for GIS Interoperability

```{code-cell} ipython3

```

### Exercise 7: Partitioned Parquet Export

```{code-cell} ipython3

```

### Exercise 8: Excel Export with Readable Geometries

```{code-cell} ipython3

```

### Exercise 9: Round-Trip Export and Import

```{code-cell} ipython3

```

### Exercise 10: Comprehensive Export Pipeline Challenge

```{code-cell} ipython3

```
