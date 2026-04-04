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
# DuckDB Python Integration

## Introduction

## Learning Objectives

## Sample Datasets

## Installation and Setup

```{code-cell} ipython3
# %pip install duckdb pandas 'polars[pyarrow]'
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
con.install_extension("httpfs")
con.load_extension("httpfs")
```

```{code-cell} ipython3
con.sql("SELECT extension_name, loaded, installed FROM duckdb_extensions()").show()
```

```{code-cell} ipython3
# Common extensions for spatial data analysis
extensions = ["httpfs", "spatial"]
for ext in extensions:
    con.install_extension(ext)
    con.load_extension(ext)
print("Extensions loaded successfully")
```

## Reading Data from Multiple Sources

### Verifying the Connection

```{code-cell} ipython3
con.sql("SELECT 42").show()
```

### Reading CSV Files from URLs

```{code-cell} ipython3
con.read_csv("https://data.gishub.org/duckdb/cities.csv")
```

```{code-cell} ipython3
con.read_csv("https://data.gishub.org/duckdb/countries.csv")
```

### Understanding DuckDB Relations

```{code-cell} ipython3
cities = con.read_csv("https://data.gishub.org/duckdb/cities.csv")
large_cities = cities.filter("population > 1000000").order("population DESC").limit(10)
large_cities.show()
```

## Seamless Integration with Pandas DataFrames

### Querying DataFrames with SQL

```{code-cell} ipython3
pandas_df = pd.DataFrame({"a": [42]})
con.sql("SELECT * FROM pandas_df")
```

```{code-cell} ipython3
# Load cities data into a pandas DataFrame
cities_df = con.read_csv("https://data.gishub.org/duckdb/cities.csv").df()
print(f"Loaded {len(cities_df)} cities into pandas DataFrame")
cities_df.head()
```

### The Bidirectional Workflow

```{code-cell} ipython3
# Query the pandas DataFrame using SQL
large_cities = con.sql(
    """
    SELECT name, country, population
    FROM cities_df
    WHERE population > 5000000
    ORDER BY population DESC
    LIMIT 10
"""
)
large_cities.show()
```

```{code-cell} ipython3
# Step 1: Use SQL for initial filtering (DuckDB's strength)
cities_rel = con.sql(
    """
    SELECT * FROM read_csv_auto('https://data.gishub.org/duckdb/cities.csv')
    WHERE population > 100000
"""
)

# Step 2: Convert to DataFrame for pandas operations
cities_df = cities_rel.df()

# Step 3: Use pandas for complex transformations
cities_df["pop_millions"] = cities_df["population"] / 1_000_000
cities_df["hemisphere"] = cities_df["latitude"].apply(
    lambda x: "North" if x > 0 else "South"
)

# Step 4: Query the transformed DataFrame with SQL
result = con.sql(
    """
    SELECT hemisphere, COUNT(*) as city_count,
           AVG(pop_millions) as avg_pop_millions
    FROM cities_df
    GROUP BY hemisphere
"""
)
result.show()
```

### Performance Considerations

## Polars Interoperability

```{code-cell} ipython3
# %pip install polars
import polars as pl

# Convert DuckDB result to Polars via pandas
pl_df = pl.from_pandas(con.sql("SELECT * FROM range(10)").df())
pl_df.head()

# Query Polars DataFrame in DuckDB
con.sql("SELECT COUNT(*) AS n FROM pl_df")
```

## Result Conversion and Output Formats

### Converting to Python Objects

```{code-cell} ipython3
con.sql("SELECT 42").fetchall()  # Returns [(42,)]
```

```{code-cell} ipython3
# Fetch a single row as a tuple
con.sql("SELECT 42, 43").fetchone()  # Returns (42, 43)
```

```{code-cell} ipython3
# Fetch many rows (useful for batch processing)
result = con.sql("SELECT * FROM range(100)")
result.fetchmany(10)  # Returns first 10 rows as list of tuples
```

### Converting to Pandas DataFrames

```{code-cell} ipython3
con.sql("SELECT 42 AS answer").df()
```

### Converting to NumPy Arrays

```{code-cell} ipython3
result = con.sql("SELECT range as id, range * 2 as doubled FROM range(5)").fetchnumpy()
print(f"Keys: {result.keys()}")
print(f"ID array: {result['id']}")
print(f"Doubled array: {result['doubled']}")
```

### Converting to Apache Arrow Tables

```{code-cell} ipython3
tbl = con.sql("SELECT range as id FROM range(5)").arrow()
print(tbl.read_next_batch())
```

### Choosing the Right Format

## Writing Data To Disk

```{code-cell} ipython3
con.sql("SELECT 42").write_parquet("out.parquet")  # Write to a Parquet file
con.sql("SELECT 42").write_csv("out.csv")  # Write to a CSV file
con.sql("COPY (SELECT 42) TO 'out.parquet'")  # Copy to a parquet file
```

## Persistent Storage and Database Files

### Creating a Persistent Database

```{code-cell} ipython3
# Create or open a connection to a file called 'spatial_analysis.db'
con = duckdb.connect("spatial_analysis.db")

# Create a table and load data into it
con.sql(
    """
    CREATE TABLE IF NOT EXISTS cities AS
    FROM read_csv_auto('https://data.gishub.org/duckdb/cities.csv')
"""
)

# Query the table to verify it was created
con.table("cities").show()
```

### Connecting to Existing Databases

```{code-cell} ipython3
# In a new Python session or later in your workflow
con = duckdb.connect("spatial_analysis.db")

# All your tables are still there
con.sql("SHOW TABLES").show()

# Query your persisted data
con.sql("SELECT COUNT(*) FROM cities").show()
```

### Connection Management and Cleanup

```{code-cell} ipython3
# Do your work with the connection
con.sql("CREATE TABLE test AS SELECT 42 AS value")

# Explicitly close when done
con.close()

# After closing, attempting to use the connection will raise an error
# con.sql("SELECT * FROM test")  # This would fail
```

```{code-cell} ipython3
# The with statement guarantees cleanup
with duckdb.connect("spatial_analysis.db") as con:
    con.sql(
        """
        CREATE TABLE IF NOT EXISTS countries AS
        FROM read_csv_auto('https://data.gishub.org/duckdb/countries.csv')
    """
    )
    con.table("countries").show()
    # Connection closes automatically when the block ends
# Even if an error occurred above, the connection is now closed
```

### Use Cases for Persistent vs In-Memory Databases

## Prepared Statements and Parameters

```{code-cell} ipython3
con = duckdb.connect()
query = "SELECT ?::INT AS a, ?::INT AS b, ?::INT + ?::INT AS sum"
con.execute(query, [2, 3, 2, 3]).fetchall()
```

```{code-cell} ipython3
min_pop = 1000000
con.execute(
    "SELECT COUNT(*) FROM read_csv_auto('https://data.gishub.org/duckdb/cities.csv') WHERE population > ?",
    [min_pop],
).fetchone()
```

## Key Takeaways

## Exercises

### Exercise 1: Installation and Basic Queries

```{code-cell} ipython3

```

### Exercise 2: Loading Remote Data

```{code-cell} ipython3

```

### Exercise 3: SQL to DataFrame Conversion

```{code-cell} ipython3

```

### Exercise 4: Querying DataFrames with SQL

```{code-cell} ipython3

```

### Exercise 5: Bidirectional Workflows

```{code-cell} ipython3

```

### Exercise 6: Result Format Conversion

```{code-cell} ipython3

```

### Exercise 7: Persistent Storage

```{code-cell} ipython3

```

### Exercise 8: Joining SQL and Python Logic

```{code-cell} ipython3

```

### Exercise 9: Writing Results to Files

```{code-cell} ipython3

```

### Exercise 10: Practical Integration Challenge

```{code-cell} ipython3

```
