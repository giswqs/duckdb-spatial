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
# Geometry Operations and Functions

## Introduction

## Learning Objectives

## Sample Datasets

## Installation and Setup

```{code-cell} ipython3
# %pip install duckdb leafmap
```

### Library Import and Configuration

```{code-cell} ipython3
import duckdb
import leafmap
```

```{code-cell} ipython3
print(f"DuckDB version: {duckdb.__version__}")
```

### Downloading the Sample Data

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

```{code-cell} ipython3
con.sql("SELECT * FROM nyc_subway_stations LIMIT 5;")
```

## Understanding Geometry Types

### Creating Sample Geometries from WKT

```{code-cell} ipython3
con.sql(
    """
CREATE OR REPLACE TABLE samples (name VARCHAR, geom GEOMETRY);

INSERT INTO samples VALUES
  ('Point', ST_GeomFromText('POINT(-100 40)')),
  ('Linestring', ST_GeomFromText('LINESTRING(0 0, 1 1, 2 1, 2 2)')),
  ('Polygon', ST_GeomFromText('POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))')),
  ('PolygonWithHole', ST_GeomFromText('POLYGON((0 0, 10 0, 10 10, 0 10, 0 0),(1 1, 1 2, 2 2, 2 1, 1 1))')),
  ('Collection', ST_GeomFromText('GEOMETRYCOLLECTION(POINT(2 0),POLYGON((0 0, 1 0, 1 1, 0 1, 0 0)))'));

SELECT * FROM samples;
"""
)
```

### Converting Geometries to Readable Text

```{code-cell} ipython3
con.sql("SELECT name, ST_AsText(geom) AS geometry FROM samples;")
```

### Exporting Sample Geometries

```{code-cell} ipython3
con.sql(
    """
COPY samples TO 'samples.geojson' (FORMAT GDAL, DRIVER 'GeoJSON');
"""
)
```

## Working with Points

### Examining Point Geometry Structure

```{code-cell} ipython3
con.sql(
    """
SELECT ST_AsText(geom)
FROM samples
WHERE name = 'Point';
"""
)
```

### Extracting Coordinate Values

```{code-cell} ipython3
con.sql(
    """
SELECT name,
       ST_X(geom) AS x_coordinate,
       ST_Y(geom) AS y_coordinate
FROM samples
WHERE name = 'Point';
"""
)
```

### Analyzing Real-World Point Data

```{code-cell} ipython3
con.sql(
    """
SELECT * FROM nyc_subway_stations LIMIT 5;
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT name,
       ROUTES,
       ST_AsText(geom) AS wkt_geometry,
       ST_X(geom) AS easting,
       ST_Y(geom) AS northing
FROM nyc_subway_stations
LIMIT 10;
"""
)
```

## Working with Linestrings

### Examining Linestring Geometry Structure

```{code-cell} ipython3
con.sql(
    """
SELECT ST_AsText(geom)
  FROM samples
  WHERE name = 'Linestring';
"""
)
```

### Extracting Linestring Properties

```{code-cell} ipython3
con.sql(
    """
SELECT ST_Length(geom)
  FROM samples
  WHERE name = 'Linestring';
"""
)
```

## Working with Polygons

### Examining Polygon Geometry Structure

```{code-cell} ipython3
con.sql(
    """
SELECT name, ST_AsText(geom)
  FROM samples
  WHERE name LIKE 'Polygon%';
"""
)
```

### Analyzing Polygon Properties

```{code-cell} ipython3
con.sql(
    """
SELECT name, ST_Area(geom) AS area_sq_units
  FROM samples
  WHERE name LIKE 'Polygon%';
"""
)
```

## Working with Collections

### Examining Collection Geometry Structure

```{code-cell} ipython3
con.sql(
    """
SELECT name, ST_AsText(geom)
  FROM samples
  WHERE name = 'Collection';
"""
)
```

## Visualizing NYC Spatial Data

```{code-cell} ipython3
con.sql("SHOW TABLES;")
```

### Visualizing Subway Stations (Points)

```{code-cell} ipython3
subway_stations_df = con.sql(
    "SELECT * EXCLUDE geom, ST_AsText(geom) as geometry FROM nyc_subway_stations"
).df()
subway_stations_df.head()
```

```{code-cell} ipython3
subway_stations_gdf = leafmap.df_to_gdf(
    subway_stations_df, src_crs="EPSG:26918", dst_crs="EPSG:4326"
)
subway_stations_gdf.head()
```

```{code-cell} ipython3
subway_stations_gdf.explore()
```

### Visualizing Streets (LineStrings)

```{code-cell} ipython3
nyc_streets_df = con.sql(
    "SELECT * EXCLUDE geom, ST_AsText(geom) as geometry FROM nyc_streets"
).df()
nyc_streets_df.head()
```

```{code-cell} ipython3
nyc_streets_gdf = leafmap.df_to_gdf(
    nyc_streets_df, src_crs="EPSG:26918", dst_crs="EPSG:4326"
)
nyc_streets_gdf.head()
```

```{code-cell} ipython3
nyc_streets_gdf.explore()
```

## Geometry Processing

### Creating Buffers and Finding Centroids

```{code-cell} ipython3
con.sql(
    """
SELECT name,
       ST_AsText(ST_Centroid(geom)) AS centroid,
       ST_AsText(ST_Buffer(geom, 50)) AS buffer_50
FROM samples
WHERE name IN ('Point','Linestring')
"""
)
```

### Simplification and Envelopes

```{code-cell} ipython3
con.sql(
    """
SELECT name,
       ST_AsText(ST_Simplify(geom, 0.05)) AS simplified,
       ST_AsText(ST_Envelope(geom)) AS envelope
FROM samples
WHERE name LIKE 'Polygon%'
"""
)
```

### Overlay Operations

```{code-cell} ipython3
con.sql(
    """
WITH a AS (
  SELECT ST_GeomFromText('POLYGON((0 0, 3 0, 3 3, 0 3, 0 0))') AS g
), b AS (
  SELECT ST_GeomFromText('POLYGON((2 -1, 5 -1, 5 2, 2 2, 2 -1))') AS g
)
SELECT ST_AsText(ST_Intersection(a.g, b.g)) AS inter,
       ST_AsText(ST_Difference(a.g, b.g)) AS only_a,
       ST_AsText(ST_Union(a.g, b.g)) AS unioned
FROM a, b;
"""
)
```

## Geometry Validity and Robustness

### Understanding Geometry Validity

### Testing and Repairing Geometry Validity

#### Example 1: Self-Intersecting Polygon (Bow-Tie Shape)

```{code-cell} ipython3
con.sql(
    """
-- Create a table with a valid and an invalid polygon
CREATE TABLE polygons AS
SELECT
    1 AS id,
    ST_GeomFromText('POLYGON((0 0, 0 1, 1 1, 1 0, 0 0))') AS geom
UNION ALL
SELECT
    2 AS id,
    ST_GeomFromText('POLYGON((0 0, 1 1, 1 0, 0 1, 0 0))') AS geom;  -- self-intersecting "bow-tie"
    """
)
```

```{code-cell} ipython3
con.sql(
    """
-- Check which geometries are valid
SELECT
    id,
    ST_AsText(geom) AS wkt,
    ST_IsValid(geom) AS is_valid
FROM polygons;
    """
)
```

```{code-cell} ipython3
con.sql(
    """
-- Fix invalid geometries using ST_MakeValid()
SELECT
    id,
    ST_IsValid(geom) AS before_valid,
    ST_IsValid(ST_MakeValid(geom)) AS after_valid,
    ST_AsText(ST_MakeValid(geom)) AS repaired_geom
FROM polygons;
"""
)
```

## Key Takeaways

## Exercises

### Exercise 1: Creating Basic Geometries from WKT

```{code-cell} ipython3

```

### Exercise 2: Extracting Coordinate Information

```{code-cell} ipython3

```

### Exercise 3: Measuring Polygon Properties

```{code-cell} ipython3

```

### Exercise 4: Creating Buffers for Proximity Analysis

```{code-cell} ipython3

```

### Exercise 5: Geometry Type Inspection

```{code-cell} ipython3

```

### Exercise 6: Simplifying Complex Geometries

```{code-cell} ipython3

```

### Exercise 7: Calculating Distances Between Features

```{code-cell} ipython3

```

### Exercise 8: Performing Overlay Operations

```{code-cell} ipython3

```

### Exercise 9: Validating and Repairing Geometries

```{code-cell} ipython3

```

### Exercise 10: Practical Integration Challenge

```{code-cell} ipython3

```
