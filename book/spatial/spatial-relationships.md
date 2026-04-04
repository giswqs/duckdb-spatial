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
# Spatial Queries and Relationships

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

### Downloading the Sample Database

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

### Exploring the Database Structure

```{code-cell} ipython3
con.sql("SHOW TABLES;")
```

```{code-cell} ipython3
con.sql("SELECT * from nyc_subway_stations LIMIT 5")
```

## Understanding Spatial Relationships

## Testing Geometric Identity

### Finding an Exact Match

```{code-cell} ipython3
con.sql(
    """
SELECT NAME, geom, ST_AsText(geom)
FROM nyc_subway_stations
WHERE name = 'Broad St';
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT name
FROM nyc_subway_stations
WHERE ST_Equals(geom, ST_GeomFromText('POINT (583571.9059213118 4506714.341192182)'));
"""
)
```

### Practical Considerations for ST_Equals

## Topological Relationships

### Testing Spatial Intersection

### Testing Spatial Separation

### Testing Containment

### When to Use ST_Contains vs ST_Within

### Important Distinction from ST_Intersects

### Testing Boundary Contact

### Understanding Boundary vs Interior Contact

### When to Use ST_Touches

### ST_Touches vs ST_Intersects

### Detecting Linear Crossings

### Testing Partial Overlap

### Practical Applications with NYC Data

```{code-cell} ipython3
con.sql(
    """
SELECT name, ST_AsText(geom)
FROM nyc_subway_stations
WHERE name = 'Broad St';
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT name, boroname
FROM nyc_neighborhoods
WHERE ST_Intersects(geom, ST_GeomFromText('POINT(583571 4506714)'));
"""
)
```

## Distance-Based Relationships

### Understanding Distance Calculation

### Basic Distance Example

```{code-cell} ipython3
con.sql(
    """
SELECT ST_Distance(
  ST_GeomFromText('POINT(0 5)'),
  ST_GeomFromText('LINESTRING(-2 2, 2 2)')) as distance;
"""
)
```

### Distance with Real Data

```{code-cell} ipython3
con.sql(
    """
SELECT
  a.name AS from_station,
  b.name AS to_station,
  ROUND(ST_Distance(a.geom, b.geom)) AS distance_meters
FROM nyc_subway_stations a, nyc_subway_stations b
WHERE a.name = 'Broad St'
  AND b.name IN ('Wall St', 'Bowling Green', 'Fulton St')
ORDER BY distance_meters;
"""
)
```

### Critical: Coordinate System Units Matter

### When to Use ST_Distance

## Filtering by Proximity Threshold

### Finding Features Within a Distance

```{code-cell} ipython3
con.sql("SELECT name FROM nyc_streets WHERE name IS NOT NULL LIMIT 5")
```

```{code-cell} ipython3
con.sql(
    """
SELECT name
FROM nyc_streets
WHERE ST_DWithin(
        geom,
        ST_GeomFromText('POINT(583571 4506714)'),
        10
      )
AND name IS NOT NULL
ORDER BY name;
"""
)
```

### When to Use ST_DWithin

## Nearest Neighbor Queries

```{code-cell} ipython3
con.sql(
    """
SELECT name,
       ST_Distance(geom, ST_GeomFromText('POINT(583571 4506714)')) AS distance_m
FROM nyc_subway_stations
ORDER BY distance_m
LIMIT 3;
"""
)
```

## Key Takeaways

## Exercises

### Exercise 1: Testing Geometric Equality

```{code-cell} ipython3

```

### Exercise 2: Point-in-Polygon Analysis

```{code-cell} ipython3

```

### Exercise 3: Containment vs. Intersection

```{code-cell} ipython3

```

### Exercise 4: Distance Calculations

```{code-cell} ipython3

```

### Exercise 5: Proximity Filtering

```{code-cell} ipython3

```

### Exercise 6: Combining Multiple Spatial Criteria

```{code-cell} ipython3

```
