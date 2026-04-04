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
# Analyzing NYC Taxi Data

## Introduction

## Learning Objectives

## About the Dataset

## Installation

```{code-cell} ipython3
# %pip install duckdb leafmap
```

## Library Import

```{code-cell} ipython3
import duckdb
import leafmap
```

## Installing and Loading Extensions

```{code-cell} ipython3
con = duckdb.connect()
con.install_extension("httpfs")
con.load_extension("httpfs")
con.install_extension("spatial")
con.load_extension("spatial")
```

## Loading Taxi Data

```{code-cell} ipython3
data_url = (
    "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2025-09.parquet"
)

con.sql(
    f"""
CREATE OR REPLACE TABLE trips AS
SELECT * FROM '{data_url}'
"""
)
```

### Inspecting the Data

```{code-cell} ipython3
con.sql("DESCRIBE trips")
```

```{code-cell} ipython3
con.sql("SELECT * FROM trips LIMIT 10")
```

```{code-cell} ipython3
con.sql(
    """
SELECT
    COUNT(*) as total_trips,
    MIN(tpep_pickup_datetime) as earliest_trip,
    MAX(tpep_pickup_datetime) as latest_trip,
    ROUND(AVG(trip_distance), 2) as avg_distance_miles,
    ROUND(AVG(total_amount), 2) as avg_total_amount
FROM trips
"""
)
```

## Temporal Analysis

### Trips by Hour of Day

```{code-cell} ipython3
hourly_trips = con.sql(
    """
SELECT
    EXTRACT(HOUR FROM tpep_pickup_datetime) as hour,
    COUNT(*) as trip_count,
    ROUND(AVG(trip_distance), 2) as avg_distance,
    ROUND(AVG(total_amount), 2) as avg_fare
FROM trips
GROUP BY hour
ORDER BY hour
"""
).df()
hourly_trips
```

```{code-cell} ipython3
fig = leafmap.line_chart(
    hourly_trips,
    x="hour",
    y="trip_count",
    title="NYC Taxi Trips by Hour of Day",
    x_label="Hour of Day",
    y_label="Number of Trips",
)
fig
```

```{code-cell} ipython3
fig.write_image("taxi-hourly-trips.png", width=1000, height=400, scale=2)
```

### Trips by Day of Week

```{code-cell} ipython3
daily_trips = con.sql(
    """
SELECT
    DAYNAME(tpep_pickup_datetime) as day_name,
    DAYOFWEEK(tpep_pickup_datetime) as day_num,
    COUNT(*) as trip_count,
    ROUND(AVG(trip_distance), 2) as avg_distance,
    ROUND(AVG(total_amount), 2) as avg_fare
FROM trips
GROUP BY day_name, day_num
ORDER BY day_num
"""
).df()
daily_trips
```

```{code-cell} ipython3
leafmap.bar_chart(
    daily_trips,
    x="day_name",
    y="trip_count",
    title="NYC Taxi Trips by Day of Week",
    x_label="Day of Week",
    y_label="Number of Trips",
    sort_column="day_num",
    descending=False,
)
```

### Peak vs Off-Peak Analysis

```{code-cell} ipython3
time_period_analysis = con.sql(
    """
SELECT
    CASE
        WHEN EXTRACT(HOUR FROM tpep_pickup_datetime) BETWEEN 6 AND 9 THEN 'Morning Rush (6-9 AM)'
        WHEN EXTRACT(HOUR FROM tpep_pickup_datetime) BETWEEN 16 AND 19 THEN 'Evening Rush (4-7 PM)'
        WHEN EXTRACT(HOUR FROM tpep_pickup_datetime) BETWEEN 10 AND 15 THEN 'Midday (10 AM-3 PM)'
        WHEN EXTRACT(HOUR FROM tpep_pickup_datetime) BETWEEN 20 AND 23 THEN 'Evening (8-11 PM)'
        ELSE 'Night/Early Morning'
    END as time_period,
    COUNT(*) as trip_count,
    ROUND(AVG(trip_distance), 2) as avg_distance,
    ROUND(AVG(total_amount), 2) as avg_fare,
    ROUND(AVG(trip_distance / NULLIF(EXTRACT(EPOCH FROM (tpep_dropoff_datetime - tpep_pickup_datetime)) / 3600, 0)), 2) as avg_speed_mph,
FROM trips
WHERE tpep_dropoff_datetime > tpep_pickup_datetime
GROUP BY time_period
ORDER BY trip_count DESC
"""
).df()
time_period_analysis
```

## Loading Taxi Zone Lookup Data

```{code-cell} ipython3
# Load taxi zone boundaries from CloudFront
taxi_zone_url = "https://d37ci6vzurychx.cloudfront.net/misc/taxi_zones.zip"
taxi_zones_gdf = leafmap.read_vector(taxi_zone_url)

# Display first few zones to understand the structure
taxi_zones_gdf.head()
```

```{code-cell} ipython3
# Check the structure and coordinate reference system
print(f"Number of zones: {len(taxi_zones_gdf)}")
print(f"Coordinate Reference System: {taxi_zones_gdf.crs}")
print(f"Columns: {list(taxi_zones_gdf.columns)}")
```

```{code-cell} ipython3
taxi_zones_gdf = taxi_zones_gdf.to_crs("EPSG:4326")
taxi_zones_gdf["centroid_lon"] = taxi_zones_gdf.geometry.centroid.x
taxi_zones_gdf["centroid_lat"] = taxi_zones_gdf.geometry.centroid.y
taxi_zones_df = taxi_zones_gdf.drop(columns=["geometry"])
taxi_zones_df.head()
```

```{code-cell} ipython3
con.sql(
    """
CREATE OR REPLACE TABLE taxi_zones AS
SELECT LocationID, borough, zone, centroid_lon, centroid_lat
FROM taxi_zones_df
ORDER BY LocationID
"""
)
con.sql("SELECT * FROM taxi_zones LIMIT 5")
```

```{code-cell} ipython3
taxi_zones_gdf.explore(column="borough", cmap="tab10")
```

## Spatial Analysis

### Pickup Location Hotspots

```{code-cell} ipython3
pickup_hotspots = con.sql(
    """
SELECT
    t.PULocationID,
    z.zone as zone_name,
    z.borough as borough,
    COUNT(*) as pickup_count,
    ROUND(AVG(t.trip_distance), 2) as avg_distance,
    ROUND(AVG(t.total_amount), 2) as avg_fare,
    z.centroid_lon,
    z.centroid_lat
FROM trips t
JOIN taxi_zones z ON t.PULocationID = z.LocationID
GROUP BY t.PULocationID, z.zone, z.borough, z.centroid_lon, z.centroid_lat
ORDER BY pickup_count DESC
LIMIT 50
"""
).df()
pickup_hotspots.head(5)
```

### Trip Distance Distribution

```{code-cell} ipython3
distance_analysis = con.sql(
    """
SELECT
    CASE
        WHEN trip_distance < 1 THEN 'Very Short (<1 mi)'
        WHEN trip_distance < 2 THEN 'Short (1-2 mi)'
        WHEN trip_distance < 5 THEN 'Medium (2-5 mi)'
        WHEN trip_distance < 10 THEN 'Long (5-10 mi)'
        ELSE 'Very Long (>10 mi)'
    END as distance_category,
    COUNT(*) as trip_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage,
    ROUND(AVG(total_amount), 2) as avg_fare
FROM trips
WHERE trip_distance > 0
GROUP BY distance_category
ORDER BY MIN(trip_distance)
"""
).df()
distance_analysis
```

```{code-cell} ipython3
leafmap.pie_chart(
    distance_analysis,
    values="trip_count",
    names="distance_category",
    title="Distribution of Trip Distances",
)
```

### Geographic Distribution Using H3

```{code-cell} ipython3
# Install H3 extension
con.install_extension("h3", repository="community")
con.load_extension("h3")
```

```{code-cell} ipython3
con.sql("SELECT * FROM taxi_zones")
```

```{code-cell} ipython3
# Aggregate pickups to H3 hexagons (resolution 8) using zone centroids
h3_pickups_df = con.sql(
    """
SELECT
    h3_latlng_to_cell(z.centroid_lat, z.centroid_lon, 8) as h3_index,
    COUNT(*) as pickup_count,
    ROUND(AVG(t.trip_distance), 2) as avg_distance,
    ROUND(AVG(t.total_amount), 2) as avg_fare,
    COUNT(DISTINCT t.PULocationID) as num_zones,
    h3_cell_to_boundary_wkt(h3_index) as geometry
FROM trips t
JOIN taxi_zones z ON t.PULocationID = z.LocationID
GROUP BY h3_index
HAVING COUNT(*) > 50
ORDER BY pickup_count DESC
"""
).df()
h3_pickups_df
```

```{code-cell} ipython3
h3_pickups_gdf = leafmap.df_to_gdf(h3_pickups_df)
```

```{code-cell} ipython3
m = leafmap.Map()
m.add_data(h3_pickups_gdf, "pickup_count", cmap="Reds", zoom_to_layer=True)
m
```

## Trip Flow Analysis

### Top Origin-Destination Pairs

```{code-cell} ipython3
# Analyze common origin-destination pairs using taxi zones
od_pairs = con.sql(
    """
SELECT
    t.PULocationID as origin_zone_id,
    zo.zone as origin_zone_name,
    zo.borough as origin_borough,
    t.DOLocationID as dest_zone_id,
    zd.zone as dest_zone_name,
    zd.borough as dest_borough,
    COUNT(*) as trip_count,
    ROUND(AVG(t.trip_distance), 2) as avg_distance,
    ROUND(AVG(t.total_amount), 2) as avg_fare,
    ROUND(AVG(EXTRACT(EPOCH FROM (t.tpep_dropoff_datetime - t.tpep_pickup_datetime)) / 60), 1) as avg_duration_min
FROM trips t
JOIN taxi_zones zo ON t.PULocationID = zo.LocationID
JOIN taxi_zones zd ON t.DOLocationID = zd.LocationID
WHERE t.tpep_dropoff_datetime > t.tpep_pickup_datetime
GROUP BY t.PULocationID, zo.zone, zo.Borough, t.DOLocationID, zd.zone, zd.Borough
HAVING COUNT(*) > 100
ORDER BY trip_count DESC
LIMIT 50
"""
).df()

od_pairs.head(20)
```

### Airport Trips Analysis

```{code-cell} ipython3
con.sql(
    """
SELECT
    CASE RatecodeID
        WHEN 1 THEN 'Standard Rate'
        WHEN 2 THEN 'JFK Airport'
        WHEN 3 THEN 'Newark Airport'
        WHEN 4 THEN 'Nassau/Westchester'
        WHEN 5 THEN 'Negotiated Fare'
        WHEN 6 THEN 'Group Ride'
        ELSE 'Unknown'
    END as trip_type,
    COUNT(*) as trip_count,
    ROUND(AVG(trip_distance), 2) as avg_distance,
    ROUND(AVG(total_amount), 2) as avg_fare,
    ROUND(AVG(EXTRACT(EPOCH FROM (tpep_dropoff_datetime - tpep_pickup_datetime)) / 60), 1) as avg_duration_min
FROM trips
GROUP BY trip_type
ORDER BY trip_count DESC
"""
).show()
```

```{code-cell} ipython3
leafmap.pie_chart(airport_trips, values="trip_count", names="trip_type", title="Trip Types by Rate Code")
```

## Payment and Economic Analysis

### Payment Type Analysis

```{code-cell} ipython3
con.sql(
    """
SELECT
    CASE payment_type
        WHEN 1 THEN 'Credit Card'
        WHEN 2 THEN 'Cash'
        WHEN 3 THEN 'No Charge'
        WHEN 4 THEN 'Dispute'
        ELSE 'Unknown'
    END as payment_method,
    COUNT(*) as trip_count,
    ROUND(AVG(fare_amount), 2) as avg_fare,
    ROUND(AVG(tip_amount), 2) as avg_tip,
    ROUND(AVG(total_amount), 2) as avg_total,
    ROUND(AVG(tip_amount) / NULLIF(AVG(fare_amount), 0) * 100, 1) as avg_tip_percentage,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage,
FROM trips
WHERE fare_amount > 0
GROUP BY payment_method
ORDER BY trip_count DESC
"""
).show()
```

### Tipping Patterns

```{code-cell} ipython3
con.sql(
    """
SELECT
    CASE
        WHEN trip_distance < 2 THEN 'Short (<2 mi)'
        WHEN trip_distance < 5 THEN 'Medium (2-5 mi)'
        WHEN trip_distance < 10 THEN 'Long (5-10 mi)'
        ELSE 'Very Long (>10 mi)'
    END AS distance_category,
    COUNT(*) AS trip_count,
    ROUND(AVG(tip_amount), 2) AS avg_tip,
    ROUND(AVG(tip_amount / NULLIF(fare_amount, 0)) * 100, 1) AS avg_tip_percentage,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage,
FROM trips
WHERE payment_type = 1  -- Credit card only
  AND fare_amount > 0
  AND tip_amount >= 0
GROUP BY distance_category
ORDER BY MIN(trip_distance)
"""
).show()
```

### Fare per Mile Analysis

```{code-cell} ipython3
fare_analysis = con.sql(
    """
SELECT
    EXTRACT(HOUR FROM tpep_pickup_datetime) as hour,
    COUNT(*) as trip_count,
    ROUND(AVG(fare_amount / NULLIF(trip_distance, 0)), 2) as avg_fare_per_mile,
    ROUND(AVG(trip_distance / NULLIF(EXTRACT(EPOCH FROM (tpep_dropoff_datetime - tpep_pickup_datetime)) / 3600, 0)), 2) as avg_speed_mph,
    ROUND(AVG(total_amount), 2) as avg_total_fare,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage,
FROM trips
WHERE trip_distance > 0
  AND tpep_dropoff_datetime > tpep_pickup_datetime
  AND EXTRACT(EPOCH FROM (tpep_dropoff_datetime - tpep_pickup_datetime)) > 60  -- At least 1 minute
GROUP BY hour
ORDER BY hour
"""
).df()
fare_analysis
```

```{code-cell} ipython3
leafmap.line_chart(fare_analysis, x="hour", y="avg_fare_per_mile", title="Average Fare Per Mile by Hour of Day", x_label="Hour of Day", y_label="Average Fare Per Mile")
```

## Passenger Behavior Analysis

```{code-cell} ipython3
con.sql(
    """
SELECT
    passenger_count,
    COUNT(*) as trip_count,
    ROUND(AVG(trip_distance), 2) as avg_distance,
    ROUND(AVG(total_amount), 2) as avg_total,
    ROUND(AVG(total_amount / passenger_count), 2) as avg_per_passenger,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage,
FROM trips
WHERE passenger_count BETWEEN 1 AND 6
GROUP BY passenger_count
ORDER BY passenger_count
"""
).show()
```

## Multi-Month Analysis

```{code-cell} ipython3
months = range(1, 10)
months_to_analyze = [f"yellow_tripdata_2025-{month:02d}.parquet" for month in months]

monthly_comparison = con.sql(
    f"""
SELECT
    EXTRACT(YEAR FROM tpep_pickup_datetime) as year,
    EXTRACT(MONTH FROM tpep_pickup_datetime) as month,
    COUNT(*) as trip_count,
    ROUND(AVG(trip_distance), 2) as avg_distance,
    ROUND(AVG(total_amount), 2) as avg_fare,
    SUM(total_amount) as total_revenue,
FROM read_parquet([{','.join([f"'https://d37ci6vzurychx.cloudfront.net/trip-data/{m}'" for m in months_to_analyze])}])
WHERE year = 2025
GROUP BY year, month
ORDER BY year, month
"""
).df()

monthly_comparison
```

```{code-cell} ipython3
# Visualize monthly trends
monthly_comparison["month_label"] = monthly_comparison.apply(
    lambda x: f"{int(x['year'])}-{int(x['month']):02d}", axis=1
)

leafmap.bar_chart(monthly_comparison, x="month_label",     y="trip_count",
    title="Monthly Taxi Trip Volume", x_label="Month", y_label="Number of Trips" )
```

## Visualization

### Aggregating Trip Data by Zone

```{code-cell} ipython3
pickup_df = con.sql(
    """
    SELECT PULocationID, COUNT(*) as pickup_count
    FROM trips
    GROUP BY PULocationID
    """
).df()
pickup_df
```

```{code-cell} ipython3
dropoff_df = con.sql(
    """
    SELECT DOLocationID, COUNT(*) as dropoff_count
    FROM trips
    GROUP BY DOLocationID
    """
).df()
dropoff_df
```

### Merging with Geographic Data

```{code-cell} ipython3
pickup_gdf = taxi_zones_gdf.merge(
    pickup_df, left_on="LocationID", right_on="PULocationID", how="left"
)
pickup_gdf['pickup_count'] = pickup_gdf['pickup_count'].fillna(0)

dropoff_gdf = taxi_zones_gdf.merge(
    dropoff_df, left_on="LocationID", right_on="DOLocationID", how="left"
)
dropoff_gdf['dropoff_count'] = dropoff_gdf['dropoff_count'].fillna(0)
```

```{code-cell} ipython3
pickup_gdf
```

```{code-cell} ipython3
dropoff_gdf
```

### 2D Choropleth Maps

```{code-cell} ipython3
m = leafmap.Map()
m.add_data(pickup_gdf, "pickup_count", cmap="YlOrRd", legend=True, zoom_to_layer=True)
m
```

```{code-cell} ipython3
m = leafmap.Map()
m.add_data(dropoff_gdf, "dropoff_count", cmap="YlOrRd", legend=True, zoom_to_layer=True)
m
```

### 3D Extruded Visualizations

```{code-cell} ipython3
import leafmap.maplibregl as leafmap
```

```{code-cell} ipython3
m = leafmap.Map(pitch=60, style="positron")
m.add_data(
    pickup_gdf,
    "pickup_count",
    cmap="YlOrRd",
    fit_bounds=True,
    extrude=True,
    scale_factor=10,
)
m
```

```{code-cell} ipython3
m = leafmap.Map(pitch=60, style="positron")
m.add_data(
    data=dropoff_gdf,
    column="dropoff_count",
    cmap="YlOrRd",
    fit_bounds=True,
    extrude=True,
    scale_factor=10,
)
m
```

## Performance Optimization Tips

### Filter Early

```{code-cell} ipython3
# Good: Filter before aggregation
con.sql("""
SELECT COUNT(*)
FROM trips
WHERE tpep_pickup_datetime >= '2025-09-15'
  AND tpep_pickup_datetime < '2025-09-16'
""")
```

### Sample for Exploration

```{code-cell} ipython3
con.sql("""
SELECT *
FROM trips
USING SAMPLE 1%  -- Analyze 1% of data quickly
""")
```

## Key Takeaways

## Exercises

### Exercise 1: Peak Hour Deep Dive

```{code-cell} ipython3

```

### Exercise 2: Weekend vs. Weekday Patterns

```{code-cell} ipython3

```

### Exercise 3: Airport Connection Analysis

```{code-cell} ipython3

```

### Exercise 4: Tipping Behavior Study

```{code-cell} ipython3

```

### Exercise 5: Speed and Congestion Analysis

```{code-cell} ipython3

```

### Exercise 6: Neighborhood Connectivity

```{code-cell} ipython3

```

### Exercise 7: Economic Revenue Analysis

```{code-cell} ipython3

```

### Exercise 8: Multi-Month Trend Analysis

```{code-cell} ipython3

```