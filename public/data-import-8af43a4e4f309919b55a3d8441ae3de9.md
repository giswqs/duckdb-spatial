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
# Loading Spatial Data Formats

## Introduction

## Learning Objectives

## Sample Datasets

## Installation and Setup

```{code-cell} ipython3
# %pip install duckdb leafmap
```

### Library Import and Initial Setup

```{code-cell} ipython3
import duckdb
import leafmap
import pandas as pd
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

## Downloading Sample Data

```{code-cell} ipython3
url = "https://data.gishub.org/duckdb/cities.zip"
leafmap.download_file(url, unzip=True)
```

## Loading CSV Files with Coordinates

### Basic CSV Loading with Automatic Detection

```{code-cell} ipython3
con.read_csv("cities.csv")
```

### Overriding Automatic Detection

```{code-cell} ipython3
con.read_csv("cities.csv", header=True, sep=",")
```

### Parallel CSV Reading for Large Files

```{code-cell} ipython3
con.read_csv("cities.csv", parallel=True)
```

### Querying CSV Files Directly in SQL

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.csv'")
```

```{code-cell} ipython3
con.sql("SELECT * FROM read_csv_auto('cities.csv')")
```

### Performance Considerations and Best Practices

```{code-cell} ipython3
con.sql("COPY (SELECT * FROM 'cities.csv') TO 'cities-demo.parquet'")
```

```{code-cell} ipython3
con.sql("DESCRIBE SELECT * FROM 'cities.csv'").show()
```

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.csv' LIMIT 100").show()
```

## Loading JSON Files

### Reading JSON with Automatic Schema Detection

```{code-cell} ipython3
con.read_json("cities.json")
```

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.json'")
```

```{code-cell} ipython3
con.sql("SELECT * FROM read_json_auto('cities.json')")
```

### Understanding JSON Format Variants

### Working with Nested JSON Structures

### Using the JSON Extension for Advanced Operations

```{code-cell} ipython3
con.install_extension("json")
con.load_extension("json")
con.sql(
    "SELECT json('{\"a\":1,\"b\":[2,3]}') AS js, json_extract(js,'$.b[0]') AS first"
)
```

## Querying Pandas DataFrames Directly

### Loading Data into Pandas and Querying with SQL

```{code-cell} ipython3
df = pd.read_csv("cities.csv")
df
```

```{code-cell} ipython3
con.sql("SELECT * FROM df").fetchall()
```

### When to Use DataFrame Querying

## Loading Parquet Files for Performance

### Reading Parquet Files in Python

```{code-cell} ipython3
con.read_parquet("cities.parquet")
```

### Querying Parquet Files Directly in SQL

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.parquet'")
```

```{code-cell} ipython3
con.sql("SELECT * FROM read_parquet('cities.parquet')")
```

### Cloud Storage and Remote Parquet Files

```{code-cell} ipython3
con.sql("SELECT * FROM 'https://data.gishub.org/duckdb/cities.parquet'")
```

```{code-cell} ipython3
url = "s3://us-west-2.opendata.source.coop/vida/google-microsoft-osm-open-buildings/geoparquet/by_country/country_iso=USA/USA.parquet"
con.sql(f"SELECT * FROM '{url}' LIMIT 10")
```

### When to Convert to Parquet

```{code-cell} ipython3
con.sql("COPY (SELECT * FROM 'cities.csv') TO 'cities-demo.parquet'")
```

## Loading GeoJSON Files with Spatial Geometries

### Discovering Available Spatial Formats

```{code-cell} ipython3
con.sql("SELECT * FROM ST_Drivers()").show(max_rows=100)
```

### Loading GeoJSON with ST_Read()

```{code-cell} ipython3
con.sql("SELECT * FROM ST_Read('cities.geojson')")
```

### DuckDB's Shorthand SQL Syntax

```{code-cell} ipython3
con.sql("FROM ST_Read('cities.geojson')")
```

### Creating Persistent Spatial Tables

```{code-cell} ipython3
con.sql("CREATE TABLE cities AS SELECT * FROM ST_Read('cities.geojson')")
```

### Querying the Spatial Table

```{code-cell} ipython3
con.table("cities")
```

```{code-cell} ipython3
con.sql("SELECT * FROM cities")
```

### Understanding GDAL Dependencies

## Loading Shapefiles into Modern Workflows

### The Shapefile Multi-File Structure

### Loading Shapefiles with ST_Read()

```{code-cell} ipython3
con.sql("SELECT * FROM ST_Read('cities.shp')")
```

```{code-cell} ipython3
con.sql("FROM ST_Read('cities.shp')")
```

### Creating Tables from Shapefiles

```{code-cell} ipython3
con.sql(
    """
        CREATE TABLE IF NOT EXISTS cities2 AS
        SELECT * FROM ST_Read('cities.shp')
    """
)
```

```{code-cell} ipython3
con.table("cities2")
```

```{code-cell} ipython3
con.sql("SELECT * FROM cities2")
```

### Shapefile Limitations and Modern Alternatives

```{code-cell} ipython3
con.sql("COPY cities2 TO 'cities2.parquet'")
```

## Loading GeoParquet for Cloud-Native Spatial Analysis

### Reading Local GeoParquet Files

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.parquet'")
```

### Converting WKB Geometries to DuckDB's Spatial Type

```{code-cell} ipython3
con.sql("FROM 'cities.parquet'")
```

### Loading GeoParquet from Cloud Storage

```{code-cell} ipython3
con.sql(
    """
        SELECT *  FROM 's3://us-west-2.opendata.source.coop/fused/overture/2025-06-25-0/theme=divisions/type=division/*.parquet'
    """
)
```

```{code-cell} ipython3
con.sql(
    """
        SELECT COUNT(*)  FROM 's3://us-west-2.opendata.source.coop/fused/overture/2025-06-25-0/theme=divisions/type=division/*.parquet'
    """
)
```

## Data Loading Performance Strategies

```{code-cell} ipython3
con.sql("FROM 'cities.parquet' USING SAMPLE 10%")
```

```{code-cell} ipython3
con.sql("SELECT name, population FROM 'cities.parquet'")
```

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.parquet' WHERE population > 1000000")
```

```{code-cell} ipython3
con.sql("SELECT * FROM 'cities.parquet' LIMIT 100")
```

## Troubleshooting Common Data Loading Issues

## Key Takeaways

## Exercises

```{code-cell} ipython3
url = "https://data.gishub.org/us/us_counties.zip"
leafmap.download_file(url)
```

### Exercise 1: CSV Loading and Schema Inspection

```{code-cell} ipython3

```

### Exercise 2: Format Conversion for Performance

```{code-cell} ipython3

```

### Exercise 3: Loading GeoJSON with Spatial Geometries

```{code-cell} ipython3

```

### Exercise 4: Working with Shapefiles

```{code-cell} ipython3

```

### Exercise 5: Querying Pandas DataFrames with SQL

```{code-cell} ipython3

```

### Exercise 6: Cloud Data Access with GeoParquet

```{code-cell} ipython3

```

### Exercise 7: Format Comparison with Your Own Data

```{code-cell} ipython3

```
