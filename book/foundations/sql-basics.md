---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.1
kernelspec:
  name: duckdb_kernel
  display_name: DuckDB
  language: ''
---
# Essential SQL for Spatial Analysis

## Introduction

## Learning Objectives

## Sample Datasets

## Setting Up the Environment

```bash
pip install duckdb duckdb-engine jupyter-duckdb jupysql
```

```bash
python -c "import duckdb_kernel, os; print(os.path.dirname(duckdb_kernel.__file__))"
```

```bash
jupyter kernelspec install <path_to_duckdb_kernel> --user
```

```bash
jupyter lab
```

## Connecting To DuckDB

```{code-cell} ipython3
%LOAD :memory:
```

## Install Extensions

```{code-cell} ipython3
SELECT * FROM duckdb_extensions();
```

```{code-cell} ipython3
INSTALL httpfs;
LOAD httpfs;
```

## Reading CSV Files from URLs

```{code-cell} ipython3
SELECT * FROM 'https://data.gishub.org/duckdb/cities.csv';
```

```{code-cell} ipython3
SELECT * FROM 'https://data.gishub.org/duckdb/countries.csv';
```

## Creating Tables for Better Performance

```{code-cell} ipython3
CREATE TABLE cities AS SELECT * FROM 'https://data.gishub.org/duckdb/cities.csv';
```

```{code-cell} ipython3
CREATE TABLE countries AS SELECT * FROM 'https://data.gishub.org/duckdb/countries.csv';
```

```{code-cell} ipython3
FROM cities;
```

```{code-cell} ipython3
FROM countries;
```

## The SQL SELECT Statement

```{code-cell} ipython3
SELECT * FROM cities;
```

### Limiting Results with LIMIT

```{code-cell} ipython3
SELECT * FROM cities LIMIT 10;
```

### Selecting Specific Columns

```{code-cell} ipython3
SELECT name, country, population FROM cities LIMIT 10;
```

### Finding Unique Values with DISTINCT

```{code-cell} ipython3
SELECT DISTINCT country FROM cities LIMIT 10;
```

### Basic Aggregate Functions

#### Counting Features

```{code-cell} ipython3
SELECT COUNT(*) FROM cities;
```

#### Counting Unique Categories

```{code-cell} ipython3
SELECT COUNT(DISTINCT country) FROM cities;
```

#### Finding Extremes

```{code-cell} ipython3
SELECT MAX(population) FROM cities;
```

```{code-cell} ipython3
SELECT MIN(population) FROM cities;
```

#### Calculating Totals

```{code-cell} ipython3
SELECT SUM(population) FROM cities;
```

#### Computing Averages

```{code-cell} ipython3
SELECT AVG(population) FROM cities;
```

### Sorting Results with ORDER BY

#### Basic Sorting

```{code-cell} ipython3
SELECT name, country, population FROM cities ORDER BY country LIMIT 10;
```

#### Multi-Level Sorting

```{code-cell} ipython3
SELECT name, country, population FROM cities
ORDER BY country ASC, population DESC LIMIT 15;
```

#### Spatial Sorting Applications

```{code-cell} ipython3
SELECT name, country, population FROM cities
ORDER BY population DESC LIMIT 10;
```

```{code-cell} ipython3
SELECT name, latitude, longitude FROM cities
ORDER BY latitude DESC, longitude ASC LIMIT 10;
```

### Calculated Columns and Expressions

#### Basic Calculations

```{code-cell} ipython3
SELECT name, population,
       population / 1000000 AS population_millions,
       ROUND(population / 1000000, 2) AS population_millions_rounded
FROM cities
WHERE population > 5000000
ORDER BY population DESC LIMIT 10;
```

#### Geographic Calculations

```{code-cell} ipython3
SELECT name, latitude, longitude,
       CASE
           WHEN latitude > 0 THEN 'Northern Hemisphere'
           ELSE 'Southern Hemisphere'
       END AS hemisphere,
       ABS(latitude) AS distance_from_equator
FROM cities
ORDER BY ABS(latitude) DESC LIMIT 10;
```

## Filtering Data with the WHERE Clause

```{code-cell} ipython3
SELECT * FROM cities WHERE country='USA';
```

```{code-cell} ipython3
SELECT * FROM cities WHERE country='USA' OR country='CAN';
```

```{code-cell} ipython3
SELECT * FROM cities WHERE country='USA' AND population>1000000;
```

## Pattern Matching with LIKE

```{code-cell} ipython3
SELECT * FROM cities WHERE country LIKE 'U%';
```

```{code-cell} ipython3
SELECT * FROM cities WHERE country LIKE '%A';
```

```{code-cell} ipython3
SELECT * FROM cities WHERE country LIKE '_S_';
```

## The IN Operator

```{code-cell} ipython3
SELECT * FROM cities WHERE country IN ('USA', 'CAN');
```

## The BETWEEN Operator

```{code-cell} ipython3
SELECT * FROM cities WHERE population BETWEEN 1000000 AND 10000000;
```

## Combining Data with SQL Joins

```{code-cell} ipython3
SELECT COUNT(*) FROM cities;
```

```{code-cell} ipython3
SELECT * FROM cities LIMIT 10;
```

```{code-cell} ipython3
SELECT COUNT(*) FROM countries;
```

```{code-cell} ipython3
SELECT * FROM countries LIMIT 10;
```

### SQL Inner Join

```{code-cell} ipython3
SELECT * FROM cities INNER JOIN countries ON cities.country = countries."Alpha3_code";
```

```{code-cell} ipython3
SELECT name, cities."country", countries."Country" FROM cities INNER JOIN countries ON cities.country = countries."Alpha3_code";
```

### SQL Left Join

```{code-cell} ipython3
SELECT * FROM cities LEFT JOIN countries ON cities.country = countries."Alpha3_code";
```

### SQL Right Join

```{code-cell} ipython3
SELECT * FROM cities RIGHT JOIN countries ON cities.country = countries."Alpha3_code";
```

### SQL Full Join

```{code-cell} ipython3
SELECT * FROM cities FULL JOIN countries ON cities.country = countries."Alpha3_code";
```

### SQL Union

```{code-cell} ipython3
SELECT country FROM cities
UNION
SELECT "Alpha3_code" FROM countries;
```

## Query Plans and Performance

```{code-cell} ipython3
EXPLAIN SELECT country, COUNT(*) FROM cities GROUP BY country;
```

```{code-cell} ipython3
EXPLAIN ANALYZE SELECT country, COUNT(*) FROM cities GROUP BY country;
```

## Aggregating Data for Summary Statistics

### Understanding GROUP BY

```{code-cell} ipython3
SELECT COUNT(name), country
FROM cities
GROUP BY country
ORDER BY COUNT(name) DESC;
```

```{code-cell} ipython3
SELECT countries."Country", COUNT(name)
FROM cities
LEFT JOIN countries ON cities.country = countries."Alpha3_code"
GROUP BY countries."Country"
ORDER BY COUNT(name) DESC;
```

### Filtering Aggregated Results with HAVING

```{code-cell} ipython3
SELECT COUNT(name), country
FROM cities
GROUP BY country
HAVING COUNT(name) > 40
ORDER BY COUNT(name) DESC;
```

```{code-cell} ipython3
SELECT countries."Country", COUNT(name)
FROM cities
LEFT JOIN countries ON cities.country = countries."Alpha3_code"
GROUP BY countries."Country"
HAVING COUNT(name) > 40
ORDER BY COUNT(name) DESC;
```

## Conditional Statements

```{code-cell} ipython3
SELECT name, population,
CASE
    WHEN population > 10000000 THEN 'Megacity'
    WHEN population > 1000000 THEN 'Large city'
    ELSE 'Small city'
END AS category
FROM cities;
```

## Saving Results

```{code-cell} ipython3
CREATE TABLE cities2 AS SELECT * FROM cities;
```

```{code-cell} ipython3
FROM cities2;
```

```{code-cell} ipython3
DROP TABLE IF EXISTS cities_usa;
CREATE TABLE cities_usa AS (SELECT * FROM cities WHERE country = 'USA');
```

```{code-cell} ipython3
FROM cities_usa;
```

```{code-cell} ipython3
INSERT INTO cities_usa (SELECT * FROM cities WHERE country = 'CAN');
```

## Advanced SQL Features for Spatial Analysis

### Window Functions

#### Ranking Cities by Population

```{code-cell} ipython3
SELECT name, country, population,
       ROW_NUMBER() OVER (ORDER BY population DESC) AS global_rank,
       RANK() OVER (PARTITION BY country ORDER BY population DESC) AS country_rank
FROM cities
WHERE population > 1000000
ORDER BY population DESC LIMIT 15;
```

#### Moving Averages and Spatial Trends

```{code-cell} ipython3
SELECT name, country, population,
       AVG(population) OVER (
           PARTITION BY country
           ORDER BY population
           ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
       ) AS local_avg_population
FROM cities
WHERE country IN ('USA', 'CHN', 'IND')
ORDER BY country, population DESC;
```

### Common Table Expressions (CTEs)

#### Analyzing Urban Hierarchies

```{code-cell} ipython3
WITH country_stats AS (
    SELECT country,
           COUNT(*) AS city_count,
           SUM(population) AS total_urban_population,
           AVG(population) AS avg_city_size,
           MAX(population) AS largest_city_population
    FROM cities
    GROUP BY country
),
urban_categories AS (
    SELECT *,
           CASE
               WHEN city_count >= 20 THEN 'Highly Urbanized'
               WHEN city_count >= 10 THEN 'Moderately Urbanized'
               WHEN city_count >= 5 THEN 'Developing Urban'
               ELSE 'Limited Urban'
           END AS urbanization_level
    FROM country_stats
)
SELECT urbanization_level,
       COUNT(*) AS countries_in_category,
       AVG(avg_city_size) AS avg_city_size_in_category,
       SUM(total_urban_population) AS total_pop_in_category
FROM urban_categories
GROUP BY urbanization_level
ORDER BY AVG(avg_city_size) DESC;
```

### Subqueries for Spatial Comparisons

#### Finding Cities Above Country Average

```{code-cell} ipython3
SELECT c1.name, c1.country, c1.population,
       country_avg.avg_population,
       ROUND((c1.population - country_avg.avg_population) / country_avg.avg_population * 100, 1) AS pct_above_avg
FROM cities c1
JOIN (
    SELECT country, AVG(population) AS avg_population
    FROM cities
    GROUP BY country
) country_avg ON c1.country = country_avg.country
WHERE c1.population > country_avg.avg_population
ORDER BY pct_above_avg DESC LIMIT 15;
```

### String Functions for Spatial Data

#### Cleaning and Standardizing Place Names

```{code-cell} ipython3
SELECT name,
       UPPER(name) AS name_upper,
       LENGTH(name) AS name_length,
       SUBSTRING(name, 1, 3) AS name_prefix,
       CASE
           WHEN name LIKE '%City%' THEN REPLACE(name, 'City', '')
           WHEN name LIKE '%town%' THEN REPLACE(name, 'town', '')
           ELSE name
       END AS cleaned_name
FROM cities
WHERE name LIKE '%City%' OR name LIKE '%town%'
ORDER BY name LIMIT 10;
```

### Date and Time Functions

```{code-cell} ipython3
-- Example patterns for temporal spatial data
SELECT
    -- Current date and time functions
    CURRENT_DATE AS today,
    CURRENT_TIMESTAMP AS now,

    -- Date arithmetic examples
    CURRENT_DATE - INTERVAL '30 days' AS thirty_days_ago,
    CURRENT_DATE + INTERVAL '1 year' AS next_year,

    -- Date formatting
    STRFTIME(CURRENT_DATE, '%Y-%m') AS year_month,
    DATE_TRUNC('month', CURRENT_DATE) AS month_start;
```

## SQL Comments and Documentation

### Single-Line Comments

```{code-cell} ipython3
SELECT name, population FROM cities
WHERE population > 1000000 -- Filter for major cities only
ORDER BY population DESC LIMIT 10;
```

### Multi-Line Comments

```{code-cell} ipython3
SELECT COUNT(name), country
FROM cities
/*
 * This query analyzes urban distribution by country
 * Useful for understanding global urbanization patterns
 * Results can be used for comparative demographic analysis
 */
GROUP BY country
ORDER BY COUNT(name) DESC
LIMIT 10;
```

## Key Takeaways

## Exercises

### Exercise 1: Loading Remote Data and Creating Tables

### Exercise 2: Basic Column Selection

### Exercise 3: Filtering Rows with WHERE

### Exercise 4: Sorting Results with ORDER BY

### Exercise 5: Finding Unique Values with DISTINCT

### Exercise 6: Counting and Grouping Data

### Exercise 7: Sorting Aggregated Results

### Exercise 8: Joining Related Tables

### Exercise 9: Pattern Matching with LIKE

### Exercise 10: Multiple Conditions with AND
