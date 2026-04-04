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
# Advanced Spatial Joins

## Introduction

## Learning Objectives

## Sample Datasets

## Installation

```{code-cell} ipython3
# %pip install duckdb leafmap
```

## Library Import and Configuration

```{code-cell} ipython3
import duckdb
import leafmap
```

```{code-cell} ipython3
url = "https://data.gishub.org/duckdb/nyc_data.db.zip"
leafmap.download_file(url, unzip=True)
```

## Connecting to DuckDB

```{code-cell} ipython3
con = duckdb.connect("nyc_data.db")
```

### Loading the Spatial Extension

```{code-cell} ipython3
con.install_extension("spatial")
con.load_extension("spatial")
```

### Exploring the Database Contents

```{code-cell} ipython3
con.sql("SHOW TABLES;")
```

## Intersection Joins

### Examining the Input Tables

```{code-cell} ipython3
con.sql("SELECT * FROM nyc_neighborhoods LIMIT 5;")
```

```{code-cell} ipython3
con.sql("SELECT * FROM nyc_subway_stations LIMIT 5;")
```

### Performing a Basic Spatial Join

```{code-cell} ipython3
con.sql(
    """
SELECT
  subways.name AS subway_name,
  neighborhoods.name AS neighborhood_name,
  neighborhoods.boroname AS borough
FROM nyc_neighborhoods AS neighborhoods
JOIN nyc_subway_stations AS subways
ON ST_Intersects(neighborhoods.geom, subways.geom)
WHERE subways.NAME = 'Broad St';
"""
)
```

### Scaling to All Features

```{code-cell} ipython3
con.sql(
    """
SELECT
  subways.name AS subway_name,
  neighborhoods.name AS neighborhood_name,
  neighborhoods.boroname AS borough
FROM nyc_neighborhoods AS neighborhoods
JOIN nyc_subway_stations AS subways
ON ST_Intersects(neighborhoods.geom, subways.geom)
"""
)
```

### Combining Spatial and Attribute Filters

```{code-cell} ipython3
con.sql(
    """
SELECT DISTINCT COLOR FROM nyc_subway_stations;
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT
  subways.name AS subway_name,
  neighborhoods.name AS neighborhood_name,
  neighborhoods.boroname AS borough
FROM nyc_neighborhoods AS neighborhoods
JOIN nyc_subway_stations AS subways
ON ST_Intersects(neighborhoods.geom, subways.geom)
WHERE subways.color = 'RED';
"""
)
```

### Pattern Summary for Point-in-Polygon Joins

## Distance Within Joins

### Understanding ST_DWithin Syntax

### Analyzing Demographics Near Transit: A Case Study

```{code-cell} ipython3
con.sql("FROM nyc_census_blocks LIMIT 5")
```

```{code-cell} ipython3
con.sql(
    """
SELECT
  100.0 * Sum(popn_white) / Sum(popn_total) AS white_pct,
  100.0 * Sum(popn_black) / Sum(popn_total) AS black_pct,
  Sum(popn_total) AS popn_total
FROM nyc_census_blocks;
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT DISTINCT routes FROM nyc_subway_stations;
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT DISTINCT routes
FROM nyc_subway_stations AS subways
WHERE strpos(subways.routes,'A') > 0;
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT
  100.0 * Sum(popn_white) / Sum(popn_total) AS white_pct,
  100.0 * Sum(popn_black) / Sum(popn_total) AS black_pct,
  Sum(popn_total) AS popn_total
FROM nyc_census_blocks AS census
JOIN nyc_subway_stations AS subways
ON ST_DWithin(census.geom, subways.geom, 200)
WHERE strpos(subways.routes,'A') > 0;
"""
)
```

## Advanced Join

```{code-cell} ipython3
con.sql(
    """
CREATE OR REPLACE TABLE subway_lines ( route char(1) );
INSERT INTO subway_lines (route) VALUES
  ('A'),('B'),('C'),('D'),('E'),('F'),('G'),
  ('J'),('L'),('M'),('N'),('Q'),('R'),('S'),
  ('Z'),('1'),('2'),('3'),('4'),('5'),('6'),
  ('7');
"""
)
```

```{code-cell} ipython3
con.sql(
    """
SELECT
  lines.route,
  100.0 * Sum(popn_white) / Sum(popn_total) AS white_pct,
  100.0 * Sum(popn_black) / Sum(popn_total) AS black_pct,
  Sum(popn_total) AS popn_total
FROM nyc_census_blocks AS census
JOIN nyc_subway_stations AS subways
ON ST_DWithin(census.geom, subways.geom, 200)
JOIN subway_lines AS lines
ON strpos(subways.routes, lines.route) > 0
GROUP BY lines.route
ORDER BY black_pct DESC;
"""
)
```

## Coordinate System Transformations

### Transforming Geographic to Projected Coordinates

```{code-cell} ipython3
url = "https://data.gishub.org/duckdb/cities.parquet"
con.sql(f"SELECT * FROM '{url}'")
```

### Applying the Transformation

```{code-cell} ipython3
con.sql(
    f"""
SELECT * EXCLUDE geometry, ST_Transform(geometry, 'EPSG:4326', 'EPSG:5070', true) AS geometry FROM '{url}'
"""
)
```

### When and How to Use ST_Transform

### Practical Transformation Example in Joins

```{code-cell} ipython3
con.sql(
    """
-- Join web-sourced cities (EPSG:4326) with local NYC neighborhoods (EPSG:26918)
SELECT
  cities.name AS city_name,
  neighborhoods.name AS neighborhood_name
FROM nyc_neighborhoods AS neighborhoods
JOIN (
  SELECT name, ST_Transform(geometry, 'EPSG:4326', 'EPSG:26918', true) AS geom
  FROM read_parquet('cities.parquet')
) AS cities
ON ST_Intersects(neighborhoods.geom, cities.geom);
"""
)
```

## Spatial Relationship Functions Reference

### Core Spatial Predicates

### Choosing the Right Predicate

## Key Takeaways

## Exercises

### Exercise 1: Basic Point-in-Polygon Containment Join

```{code-cell} ipython3

```

### Exercise 2: Distance-Based Proximity Join with Aggregation

```{code-cell} ipython3

```

### Exercise 3: Finding Nearest Features with Distance Calculation

```{code-cell} ipython3

```

### Exercise 4: Multi-Table Join with Complex Filtering

```{code-cell} ipython3

```

### Exercise 5: Performance Analysis with Filtering Strategies

```{code-cell} ipython3

```

### Exercise 6: Demographic Analysis Across Multiple Transit Lines

```{code-cell} ipython3

```

### Exercise 7: Coordinate System Transformation Practice

```{code-cell} ipython3

```

### Exercise 8: Transit Accessibility Scoring

```{code-cell} ipython3

```
