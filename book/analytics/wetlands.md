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
# Analyzing US National Wetlands Inventory

## Introduction

## Learning Objectives

## Datasets Used in This Chapter

### National Wetlands Inventory (NWI) Dataset

### US States Boundary Dataset

### Data Access and Format Advantages

### Understanding the Analysis Scale

## Understanding the Dataset Source

### Data Download from Source

### Data Conversion to GeoParquet

## Accessing Wetland Data with DuckDB

### Environment Setup and Extension Loading

```{code-cell} ipython3
# %pip install duckdb leafmap
```

```{code-cell} ipython3
import duckdb

# Create an in-memory DuckDB connection
con = duckdb.connect()

# Install and load the spatial extension for geometry operations
con.install_extension("spatial")
con.load_extension("spatial")
```

### Querying a Single State's Wetlands

```{code-cell} ipython3
# Select a state to analyze (DC has a manageable dataset size for initial exploration)
state = "DC"  # You can change this to any two-letter state code like "FL", "CA", "TX"

# Construct the URL to the GeoParquet file hosted on Source Cooperative
url = f"https://data.source.coop/giswqs/nwi/wetlands/{state}_Wetlands.parquet"

# Query the wetland data directly from cloud storage
# DuckDB automatically handles the HTTPS connection and Parquet parsing
con.sql(f"SELECT * FROM '{url}'")
```

### Inspecting the Data Schema

```{code-cell} ipython3
con.sql(f"DESCRIBE SELECT * FROM '{url}'")
```

```{code-cell} ipython3
con.sql(f"SELECT DISTINCT WETLAND_TYPE FROM '{url}'")
```

### Alternative Access Methods

```bash
aws s3 ls s3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/
```

```bash
aws s3 ls s3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/ --recursive --summarize | grep "Total Size" | awk '{print $3/1024/1024/1024 " GB"}'
```

## Visualizing Wetland Distributions

### Loading and Preparing State-Level Data

```{code-cell} ipython3
import leafmap.maplibregl as leafmap
```

```{code-cell} ipython3
state = "DC"  # Washington DC has a manageable dataset size for visualization
url = f"https://data.source.coop/giswqs/nwi/wetlands/{state}_Wetlands.parquet"

# Read the Parquet file from cloud storage and convert to a GeoDataFrame
# src_crs='EPSG:5070' specifies the Albers Equal Area Conic projection (used in NWI data)
# dst_crs='EPSG:4326' reprojects to WGS84 latitude/longitude (required for web mapping)
# return_type='gdf' ensures we get a GeoPandas GeoDataFrame with spatial capabilities
gdf = leafmap.read_vector(
    url, return_type="gdf", src_crs="EPSG:5070", dst_crs="EPSG:4326"
)

# Display basic statistics about the loaded dataset
print(f"Number of wetland features: {len(gdf):,}")
print(f"Unique wetland types: {gdf['WETLAND_TYPE'].nunique()}")
print(f"Total wetland area: {gdf['Shape_Area'].sum() / 1_000_000:.2f} sq km")
```

### Creating Thematic Visualizations by Wetland Type

```{code-cell} ipython3
# Create a new interactive map centered on the data
m = leafmap.Map()

# Add a high-resolution satellite imagery basemap for context
# This helps identify wetland relationships with land use and landscape features
m.add_basemap("Esri.WorldImagery")

# Define colors for each wetland type using RGB values
color_map = {
    "Freshwater Forested/Shrub Wetland": (0, 136, 55),  # Dark green
    "Freshwater Emergent Wetland": (127, 195, 28),  # Bright green
    "Freshwater Pond": (104, 140, 192),  # Light blue
    "Estuarine and Marine Wetland": (102, 194, 165),  # Teal
    "Riverine": (1, 144, 191),  # Medium blue
    "Lake": (19, 0, 124),  # Dark blue
    "Estuarine and Marine Deepwater": (0, 124, 136),  # Deep teal
    "Other": (178, 134, 86),  # Brown
}


# Helper function to convert RGB tuples to RGBA strings with transparency
def rgba(rgb, a=0.85):
    r, g, b = rgb
    return f"rgba({r},{g},{b},{a})"


# Build a MapLibre GL style expression for conditional coloring
# The "match" expression looks up each feature's WETLAND_TYPE and applies the corresponding color
fill_match = ["match", ["get", "WETLAND_TYPE"]]
for wetland_type, color in color_map.items():
    fill_match += [wetland_type, rgba(color)]
# Add a default gray color for any wetland types not in our color map
fill_match += ["rgba(200,200,200,0.6)"]

# Define the paint properties for the wetland polygons
paint = {
    "fill-color": fill_match,  # Apply conditional colors
    "fill-outline-color": "rgba(0,0,0,0.25)",  # Subtle dark outlines
}

# Add the wetland GeoDataFrame to the map as filled polygons
m.add_vector(gdf, layer_type="fill", paint=paint, name="Wetlands")

# Add a legend explaining the color scheme
m.add_legend(builtin_legend="NWI", title="Wetland Type")

# Display the map
m
```

```{code-cell} ipython3
# Alternative simplified approach using leafmap's built-in NWI styling
m = leafmap.Map()
m.add_basemap("Esri.WorldImagery")
m.add_nwi(gdf, name="Wetlands")
m.add_legend(builtin_legend="NWI", title="Wetland Type")
m
```

## National-Scale Wetland Analysis

### Counting All Wetland Features Nationally

```{code-cell} ipython3
con.sql(
    f"""
SELECT COUNT(*) AS Count
FROM 's3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet'
"""
)
```

### Analyzing Wetlands by State

```{code-cell} ipython3
df = con.sql(
    f"""
SELECT filename, COUNT(*) AS Count
FROM read_parquet('s3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet', filename=true)
GROUP BY filename
ORDER BY COUNT(*) DESC;
"""
).df()
df.head()
```

```{code-cell} ipython3
df["filename"].tolist()
```

```{code-cell} ipython3
count_df = con.sql(
    f"""
SELECT SUBSTRING(filename, LENGTH(filename) - 18, 2) AS State, COUNT(*) AS Count
FROM read_parquet('s3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet', filename=true)
GROUP BY State
ORDER BY COUNT(*) DESC;
"""
).df()
count_df.head(10)
```

### Creating Tables for Spatial Joins

```{code-cell} ipython3
con.sql("CREATE OR REPLACE TABLE wetlands AS FROM count_df")
con.sql("FROM wetlands")
```

### Loading State Boundaries for Mapping

```{code-cell} ipython3
url = "https://data.gishub.org/us/us_states.parquet"
con.sql(
    f"""
CREATE OR REPLACE TABLE states AS
SELECT * FROM '{url}'
"""
)
con.sql("FROM states")
```

### Joining Wetland Statistics with State Geometries

```{code-cell} ipython3
con.sql(
    """
SELECT * FROM states INNER JOIN wetlands ON states.id = wetlands.State
"""
)
```

### Preparing Data for Visualization

```{code-cell} ipython3
df = con.sql(
    """
SELECT name, State, Count, ST_AsText(geometry) as geometry
FROM states INNER JOIN wetlands ON states.id = wetlands.State
"""
).df()
df.head()
```

```{code-cell} ipython3
gdf = leafmap.df_to_gdf(df, src_crs="EPSG:4326")
```

### Creating a Choropleth Map

```{code-cell} ipython3
m = leafmap.Map(center=[-100, 45], zoom=2, projection="globe", style="positron")
m.add_data(
    data=gdf,
    column="Count",
    scheme="Quantiles",
    cmap="Greens",
    legend_title="Wetland Count",
    fit_bounds=False,
)
m
```

### Statistical Visualizations

```{code-cell} ipython3
leafmap.pie_chart(
    count_df, "State", "Count", height=800, title="Number of Wetlands by State"
)
```

```{code-cell} ipython3
leafmap.bar_chart(count_df, "State", "Count", title="Number of Wetlands by State")
```

### Analyzing Wetland Area

```{code-cell} ipython3
con.sql(
    f"""
SELECT SUM(Shape_Area) /  1000000 AS Area_SqKm
FROM 's3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet'
"""
)
```

```{code-cell} ipython3
area_df = con.sql(
    f"""
SELECT SUBSTRING(filename, LENGTH(filename) - 18, 2) AS State, ROUND(SUM(Shape_Area) /  1000000, 0) AS Area_SqKm
FROM read_parquet('s3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet', filename=true)
GROUP BY State
ORDER BY Area_SqKm DESC;
"""
).df()
area_df.head(10)
```

```{code-cell} ipython3
leafmap.pie_chart(
    area_df, "State", "Area_SqKm", height=900, title="Wetland Area by State"
)
```

```{code-cell} ipython3
leafmap.bar_chart(area_df, "State", "Area_SqKm", title="Wetland Area by State")
```

### Classification by Wetland Type

```{code-cell} ipython3
type_df = con.sql(
    f"""
SELECT WETLAND_TYPE,
       COUNT(*) AS count,
       ROUND(SUM(Shape_Area) / 1e6, 0) AS area_sqkm
FROM 's3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet'
GROUP BY WETLAND_TYPE
ORDER BY area_sqkm DESC
"""
).df()
type_df.head(10)
```

```{code-cell} ipython3
by_state_type = con.sql(
    f"""
SELECT SUBSTRING(filename, LENGTH(filename) - 18, 0) AS State,
       WETLAND_TYPE,
       ROUND(SUM(Shape_Area) / 1e6, 2) AS area_sqkm
FROM read_parquet('s3://us-west-2.opendata.source.coop/giswqs/nwi/wetlands/*.parquet', filename=true)
GROUP BY State, WETLAND_TYPE
QUALIFY ROW_NUMBER() OVER (PARTITION BY State ORDER BY area_sqkm DESC) <= 10
ORDER BY State, area_sqkm DESC
"""
).df()
by_state_type.head(20)
```

```{code-cell} ipython3
leafmap.pie_chart(
    type_df, "WETLAND_TYPE", "area_sqkm", title="National Wetland Area by Type"
)
```

## Key Takeaways

## Exercises

### Exercise 1: State-Level Wetland Analysis

```{code-cell} ipython3

```

### Exercise 2: Regional Wetland Comparisons

```{code-cell} ipython3

```

### Exercise 3: National Wetland Type Classification

```{code-cell} ipython3

```

### Exercise 4: Focused Regional Analysis

```{code-cell} ipython3

```
