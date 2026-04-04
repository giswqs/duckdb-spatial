---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.18.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---
# Analyzing Global Building Footprints

## Introduction

## Learning Objectives

## About the Dataset

## Installation and Setup

```{code-cell} ipython3
# %pip install duckdb "leafmap[duckdb]"
```

```{code-cell} ipython3
import duckdb
import leafmap.maplibregl as leafmap
```

## Installing and Loading Extensions

```{code-cell} ipython3
con = duckdb.connect("buildings.db")
con.install_extension("httpfs")
con.load_extension("httpfs")
con.install_extension("spatial")
con.load_extension("spatial")
```

## Exploring Available Data

### Understanding the Dataset Scale

```{code-cell} ipython3
release = leafmap.get_overture_latest_release(patch=True)
print(release)
```

```{code-cell} ipython3
url = f"s3://overturemaps-us-west-2/release/{release}/theme=buildings/type=building/*.parquet"
```

```{code-cell} ipython3
# Examine the first few records to understand the schema
con.sql(
    f"""
SELECT *
FROM read_parquet('{url}')
LIMIT 10
"""
)
```

### Counting the Complete Dataset

```{code-cell} ipython3
con.sql(
    f"""
SELECT COUNT(*) as count
FROM read_parquet('{url}')
"""
)
```

### Visualizing Buildings in 3D

```{code-cell} ipython3
m = leafmap.Map(
    center=[-74.009789, 40.709945], zoom=14.65, pitch=60, bearing=-17, style="positron"
)
m.add_basemap("OpenStreetMap.Mapnik")
m.add_basemap("Esri.WorldImagery", visible=False)
m.add_overture_3d_buildings(
    values=[0, 200, 400], colors=["lightgray", "royalblue", "lightblue"]
)
m
```

## Regional Analysis with Bounding Box Filtering

### Loading Regional Data with Spatial Filters

```{code-cell} ipython3
# Define bounding box for Colorado (approximate)
min_lon, max_lon = -109.1, -102.0
min_lat, max_lat = 37.0, 41.0

# Create a table for the region's buildings
con.sql(
    f"""
CREATE TABLE IF NOT EXISTS buildings AS
SELECT
    id,
    geometry,
    bbox,
    height,
    num_floors,
    class,
    sources,
    REPLACE(CAST(sources->0->'dataset' as VARCHAR), '"', '') as source,
    ROUND(ST_Area(ST_Transform(geometry, 'EPSG:4326', 'EPSG:32613', true)), 2) as area_sq_meters
FROM read_parquet('{url}')
WHERE bbox.xmin >= {min_lon} AND bbox.xmax <= {max_lon}
  AND bbox.ymin >= {min_lat} AND bbox.ymax <= {max_lat}
"""
)
```

```{code-cell} ipython3
con.sql("FROM buildings LIMIT 10")
```

### Exporting Regional Data

```{code-cell} ipython3
con.sql("COPY buildings TO 'buildings.parquet'")
```

### Basic Statistics

```{code-cell} ipython3
# Count total buildings in Colorado
con.sql("SELECT COUNT(*) as total_buildings FROM buildings")
```

```{code-cell} ipython3
# Calculate comprehensive area statistics
con.sql(
    """
SELECT
    COUNT(*) as building_count,
    ROUND(SUM(area_sq_meters) / 1000000, 2) as total_area_sq_km,
    ROUND(AVG(area_sq_meters), 2) as avg_building_area,
    ROUND(MIN(area_sq_meters), 2) as min_area,
    ROUND(MAX(area_sq_meters), 2) as max_area,
    ROUND(MEDIAN(area_sq_meters), 2) as median_area
FROM buildings
"""
)
```

### Building Size Distribution

```{code-cell} ipython3
# Categorize buildings by size
con.sql(
    """
SELECT
    CASE
        WHEN area_sq_meters < 50 THEN 'Small (<50 m²)'
        WHEN area_sq_meters < 100 THEN 'Medium (50-100 m²)'
        WHEN area_sq_meters < 200 THEN 'Large (100-200 m²)'
        ELSE 'Very Large (>200 m²)'
    END as size_category,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage
FROM buildings
GROUP BY size_category
ORDER BY MIN(area_sq_meters)
"""
)
```

### Spatial Sampling and Visualization

```{code-cell} ipython3
# Sample 10,000 buildings for visualization
sample_gdf = con.sql(
    """
SELECT area_sq_meters, ST_AsText(geometry) as geometry
FROM buildings
USING SAMPLE 10000
"""
).df()

# Convert to GeoDataFrame
gdf = leafmap.df_to_gdf(sample_gdf, src_crs="EPSG:4326")
```

```{code-cell} ipython3
# Visualize the building sample
m = leafmap.Map(center=[-104.978833, 39.740018], zoom=12)
m.add_data(
    gdf, column="area_sq_meters", cmap="RdYlGn", legend_title="Building Area (m²)", fit_bounds=False
)
m
```

```{code-cell} ipython3
con.close()
```

### Visualizing Buildings Directly from DuckDB

```{code-cell} ipython3
m = leafmap.Map(
    center=[-104.991531, 39.742043], zoom=14, style="positron", height="600px"
)
m.add_basemap("Esri.WorldImagery")
m.add_duckdb_layer(
    database_path="buildings.db",
    table_name="buildings",
    geom_column="geometry",
    layer_name="Buildings",
    layer_type="fill",
    paint={"fill-color": "#3388ff", "fill-outline-color": "#ffffff"},
    opacity=0.5,
    fit_bounds=False,
    src_crs="EPSG:4326",
    quiet=False,
)
m
```

### Data-Driven Styling by Building Area

```{code-cell} ipython3
m = leafmap.Map(
    center=[-104.991531, 39.742043], zoom=14, style="positron", height="600px"
)
m.add_basemap("Esri.WorldImagery")

paint = {
    "fill-color": [
        "interpolate",
        ["linear"],
        ["get", "area_sq_meters"],
        0,
        "#000004",
        50,
        "#781c6d",
        100,
        "#ed6925",
        200,
        "#fcffa4",
    ],
}
m.add_duckdb_layer(
    database_path="buildings.db",
    table_name="buildings",
    geom_column="geometry",
    layer_name="Buildings",
    layer_type="fill",
    paint=paint,
    opacity=0.8,
    fit_bounds=False,
    src_crs="EPSG:4326",
    quiet=False,
)
legend_dict = {
    "Small (<50 m²)": "#000004",
    "Medium (50-100 m²)": "#781c6d",
    "Large (100-200 m²)": "#ed6925",
    "Very Large (>200 m²)": "#fcffa4",
}
m.add_legend(title="Build Category", legend_dict=legend_dict)
m
```

```{code-cell} ipython3
m.close_db_connections()
```

## Multi-Region Comparison

### Setting Up Multi-County Analysis

```{code-cell} ipython3
con = duckdb.connect()
con.load_extension("httpfs")
con.load_extension("spatial")
```

```{code-cell} ipython3
con.sql("CREATE TABLE buildings AS FROM 'buildings.parquet'")
```

```{code-cell} ipython3
con.sql("DESCRIBE buildings")
```

### Loading County Boundaries

```{code-cell} ipython3
# Load Colorado county boundaries
counties_url = "https://data.gishub.org/us/us_counties.parquet"
counties_gdf = leafmap.read_vector(counties_url)
counties_gdf.crs = "EPSG:4326"
counties_gdf = counties_gdf[counties_gdf["STATE"] == "08"]
counties_gdf.head()
```

```{code-cell} ipython3
counties_gdf.explore()
```

```{code-cell} ipython3
# First, create a temporary table with county boundaries
con.sql(
"""
CREATE OR REPLACE TABLE counties AS
SELECT
    NAME as county_name,
    geometry
FROM read_parquet('https://data.gishub.org/us/us_counties.parquet')
WHERE STATE = '08'
"""
)

# Perform spatial join and calculate statistics by county
comparison_df = con.sql(
"""
SELECT
    c.county_name,
    COUNT(b.*) as building_count,
    ROUND(SUM(b.area_sq_meters) / 1000000, 2) as total_area_sq_km,
    ROUND(AVG(b.area_sq_meters), 2) as avg_building_size,
    ROUND(MEDIAN(b.area_sq_meters), 2) as median_building_size
FROM buildings b
JOIN counties c ON ST_Intersects(b.geometry, c.geometry)
GROUP BY c.county_name
ORDER BY building_count DESC
"""
).df()
comparison_df.head()
```

```{code-cell} ipython3
# Create a bar chart comparing building counts for top 20 counties
top_counties = comparison_df.head(20)
leafmap.bar_chart(
    top_counties,
    "county_name",
    "building_count",
    x_label="County",
    y_label="Building Count",
    title="Building Count by County (Top 20)",
)
```

```{code-cell} ipython3
# Compare average building sizes for top counties
leafmap.bar_chart(
    top_counties,
    "county_name",
    "avg_building_size",
    x_label="County",
    y_label="Average Building Size (m²)",
    title="Average Building Size by County (m²)",
)
```

## Data Source Filtering and Quality Assessment

```{code-cell} ipython3
# Examine data sources in the buildings
source_query = """
SELECT
    source,
    COUNT(*) as building_count,
    ROUND(AVG(area_sq_meters), 2) as avg_area,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage
FROM buildings
WHERE sources IS NOT NULL
GROUP BY source
ORDER BY building_count DESC
"""
source_df = con.sql(source_query).df()
source_df
```

```{code-cell} ipython3
leafmap.pie_chart(source_df, names="source", values="building_count", title="Building Count by Source")
```

### Filtering by Data Source Quality

```{code-cell} ipython3
# Filter to OSM buildings only
con.sql(
    """
SELECT
    COUNT(*) as osm_building_count,
    ROUND(AVG(area_sq_meters), 2) as avg_area,
    COUNT(CASE WHEN height IS NOT NULL THEN 1 END) as buildings_with_height,
    COUNT(CASE WHEN num_floors IS NOT NULL THEN 1 END) as buildings_with_floors,
    COUNT(CASE WHEN class IS NOT NULL THEN 1 END) as buildings_with_class
FROM buildings
WHERE source='OpenStreetMap'
"""
).show()
```

```{code-cell} ipython3
con.close()
```

### Visualizing Data Source Distribution

```{code-cell} ipython3
m = leafmap.Map(
    center=[-105.012019, 39.675236], zoom=15, style="positron", height="600px"
)
m.add_basemap("Esri.WorldImagery")

paint = {
    "fill-color": [
        "case",
        ["==", ["get", "source"], "OpenStreetMap"],
        "#FF0000",
        ["==", ["get", "source"], "Microsoft ML Buildings"],
        "#00FF00",
        ["==", ["get", "source"], "Esri Community Maps"],
        "#0000FF",
        "#000000",
    ],
}
m.add_duckdb_layer(
    database_path="buildings.db",
    table_name="buildings",
    geom_column="geometry",
    layer_name="Buildings",
    layer_type="fill",
    paint=paint,
    opacity=0.5,
    fit_bounds=False,
    src_crs="EPSG:4326",
    quiet=False,
)
legend_dict = {
    "OpenStreetMap": "#FF0000",
    "Microsoft ML Buildings": "#00FF00",
    "Esri Community Maps": "#0000FF",
}
m.add_legend(title="Data Source", legend_dict=legend_dict)
m
```

```{code-cell} ipython3
m.close_db_connections()
```

## Grid-Based Spatial Aggregation

### Setting Up Grid Analysis

```{code-cell} ipython3
con = duckdb.connect()
con.load_extension("httpfs")
con.load_extension("spatial")
```

```{code-cell} ipython3
con.sql("CREATE TABLE buildings AS FROM 'buildings.parquet'")
```

### Generating a Regular Grid

```{code-cell} ipython3
extent = (-109.1, 37.0, -102.0, 41.0)
grid = leafmap.generate_latlon_grid(extent, dx=0.1, dy=0.1, output="grid.geojson")
grid.head()
```

```{code-cell} ipython3
print(f"Number of grid cells: {len(grid)}")
```

```{code-cell} ipython3
grid.explore()
```

### Loading Grid into DuckDB

```{code-cell} ipython3
con.sql("""
CREATE OR REPLACE TABLE grid AS
SELECT * FROM ST_Read('grid.geojson')
""")
```

```{code-cell} ipython3
con.sql("SHOW TABLES")
```

### Performing Spatial Aggregation

```{code-cell} ipython3
con.sql("""
CREATE OR REPLACE TABLE grid_building_counts AS
SELECT
    g.id,
    g.lon_min,
    g.lat_min,
    g.lon_max,
    g.lat_max,
    g.geom,
    COUNT(b.*) AS building_count
FROM grid AS g
LEFT JOIN buildings AS b
    ON ST_Intersects(g.geom, b.geometry)
GROUP BY
    g.id,
    g.lon_min,
    g.lat_min,
    g.lon_max,
    g.lat_max,
    g.geom
ORDER BY
    g.id;
""")
```

### Visualizing Grid Aggregation Results

```{code-cell} ipython3
grid_df = con.sql("SELECT * EXCLUDE (geom), ST_AsText(geom) AS geometry FROM grid_building_counts").df()
grid_gdf = leafmap.df_to_gdf(grid_df, src_crs="EPSG:4326")
grid_gdf.head()
```

```{code-cell} ipython3
m = leafmap.Map()
m.add_basemap("Esri.WorldImagery", before_id=m.first_symbol_layer_id)
m.add_data(grid_gdf, column="building_count", cmap="inferno", name="Building Count", before_id=m.first_symbol_layer_id)
m
```

```{code-cell} ipython3
con.close()
```

## Aggregating Building Data with H3 Hexagonal Grids

### Understanding H3's Advantages Over Traditional Grids

### Installing and Loading the H3 Extension

```{code-cell} ipython3
con = duckdb.connect()
con.install_extension("h3", repository="community")
con.load_extension("h3")
```

```{code-cell} ipython3
con.load_extension("httpfs")
con.load_extension("spatial")
```

### Choosing Appropriate H3 Resolutions

### Creating H3-Based Building Aggregations

```{code-cell} ipython3
# Aggregate buildings into H3 resolution 6 hexagons
h3_res4_query = f"""
CREATE TABLE IF NOT EXISTS h3_res4_aggregation AS
SELECT
    h3_latlng_to_cell(
        ST_Y(ST_Centroid(geometry)),
        ST_X(ST_Centroid(geometry)),
        4
    ) as h3_cell,
    COUNT(*) as building_count,
FROM read_parquet('{url}')
WHERE bbox.xmin BETWEEN -178.5 AND 178.5
GROUP BY h3_cell
ORDER BY building_count DESC
"""
# con.sql(h3_res4_query)
```

```{code-cell} ipython3
h3_geo_query = f"""
CREATE OR REPLACE TABLE h3_res4_geo AS
SELECT
    CAST(h3_cell AS VARCHAR) as h3_cell,
    building_count,
    ST_GeomFromText(h3_cell_to_boundary_wkt(h3_cell)) as geometry
FROM h3_res4_aggregation
"""
# con.sql(h3_geo_query)
```

```{code-cell} ipython3
con.sql("""
CREATE TABLE IF NOT EXISTS h3_res4_geo AS
SELECT * FROM 'https://data.gishub.org/duckdb/h3_res4_geo.parquet';
""")
```

```{code-cell} ipython3
con.sql("SHOW TABLES")
```

```{code-cell} ipython3
con.sql("COPY h3_res4_geo TO 'h3_res4_geo.parquet'")
```

```{code-cell} ipython3
h3_res4_gdf = leafmap.read_vector("h3_res4_geo.parquet")
h3_res4_gdf
```

```{code-cell} ipython3
print(f"Number of H3 cells: {len(h3_res4_gdf)}")
```

### Visualizing H3 Aggregations

```{code-cell} ipython3
m = leafmap.Map()
m.add_basemap("Esri.WorldImagery", before_id=m.first_symbol_layer_id, visible=False)
m.add_data(
    data=h3_res4_gdf,
    column="building_count",
    scheme="JenksCaspall",
    cmap="inferno",
    outline_color="rgba(255, 255, 255, 0)",
    name="H3 Hexagon",
    before_id=m.first_symbol_layer_id,
)
m
```

### 3D Visualization with Extruded Hexagons

```{code-cell} ipython3
m = leafmap.Map(center=[7.062832, -4.144790], pitch=45.5, zoom=1.57)
m.add_basemap("Esri.WorldImagery", before_id=m.first_symbol_layer_id, visible=False)
m.add_data(
    data=h3_res4_gdf,
    column="building_count",
    scheme="JenksCaspall",
    cmap="inferno",
    outline_color="rgba(255, 255, 255, 0)",
    name="H3 Hexagon",
    before_id=m.first_symbol_layer_id,
    extrude=True,
    fit_bounds=False,
)
m
```

### Advanced H3 Operations

```{code-cell} ipython3
# Find the densest H3 cell (urban core)
densest_cell = h3_res4_gdf.iloc[0]["h3_cell"]

# Get neighboring cells using H3 grid_disk function
neighbors_query = f"""
SELECT unnest(h3_grid_disk({densest_cell}, 1)) as h3_cell
"""

neighbors_df = con.sql(neighbors_query).df()
print(f"Cell {densest_cell} has {len(neighbors_df)-1} neighbors")
neighbors_df
```

```{code-cell} ipython3
neighbors_df["h3_cell"] = neighbors_df["h3_cell"].astype(str)
neighbors_df = neighbors_df.merge(h3_res4_gdf, on="h3_cell", how="left")
neighbors_df["geometry"] = neighbors_df["geometry"].astype(str)
neighbors_gdf = leafmap.df_to_gdf(neighbors_df, src_crs="EPSG:4326")
neighbors_gdf
```

```{code-cell} ipython3
neighbors_gdf.explore(column="building_count", cmap="inferno")
```

### When to Use H3 vs. Other Aggregation Methods

## Key Takeaways

## Exercises

### Exercise 1: Regional Deep Dive

```{code-cell} ipython3

```

### Exercise 2: Multi-Resolution Grid Analysis

```{code-cell} ipython3

```

### Exercise 3: Cross-Regional Building Size Analysis

```{code-cell} ipython3

```

### Exercise 4: Data Source Quality Analysis

```{code-cell} ipython3

```

### Exercise 5: Building Height Analysis

```{code-cell} ipython3

```
