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
# Developing Interactive Dashboards with Voilà and Solara

## Introduction

### Understanding Voilà

### Understanding Solara

### Integration with the Geospatial Python Ecosystem

### Choosing the Right Framework

### Why Not Streamlit or Gradio?

### What You'll Learn

## Learning Objectives

## Installing Voilà and Solara

```{code-cell} ipython3
# %pip install voila solara leafmap duckdb
```

```{code-cell} ipython3
import voila
import solara
```

## Introduction to Hugging Face Spaces

### Creating a Hugging Face Account

### Installing the Hugging Face CLI

```{code-cell} ipython3
# %pip install -U "huggingface_hub"
```

### Logging in to Hugging Face

```bash
hf auth login
```

## Creating a Basic Voilà Application

### Creating a new Hugging Face Space

### Embedding the Hugging Face Space in Your Website

### Running the Voilà Application

### Understanding the Application Code

#### Setting Up the Map and Imports

```{code-cell} ipython3
import json
import duckdb
import ipywidgets as widgets
import leafmap.maplibregl as leafmap
import matplotlib.pyplot as plt

m = leafmap.Map(sidebar_visible=True, layer_manager_expanded=False, height="800px")
m.add_basemap("Esri.WorldImagery", before_id=m.first_symbol_layer_id, visible=False)
m.add_draw_control(controls=["polygon", "trash"])
m
```

#### Initializing the DuckDB Connection

```{code-cell} ipython3
con = duckdb.connect()
con.install_extension("httpfs")
con.install_extension("spatial")
con.install_extension("h3", repository="community")
con.load_extension("httpfs")
con.load_extension("spatial")
con.load_extension("h3")
```

#### Loading the Building Data

```{code-cell} ipython3
url = "https://data.gishub.org/duckdb/h3_res4_geo.parquet"

con.sql(
    f"CREATE TABLE IF NOT EXISTS h3_res4_geo AS SELECT * FROM read_parquet('{url}');"
)
```

#### Creating the User Interface Controls

```{code-cell} ipython3
colormaps = sorted(plt.colormaps())

checkbox = widgets.Checkbox(
    description="3D Map",
    value=True,
    style={"description_width": "initial"},
    layout=widgets.Layout(width="initial"),
)
outline_chk = widgets.Checkbox(
    description="Add Hexagon Outline",
    value=False,
    style={"description_width": "initial"},
    layout=widgets.Layout(width="initial"),
)

colormap_dropdown = widgets.Dropdown(
    options=colormaps,
    description="Colormap:",
    value="inferno",
    style={"description_width": "initial"},
)
class_slider = widgets.IntSlider(
    description="Class:",
    min=1,
    max=10,
    step=1,
    value=5,
    style={"description_width": "initial"},
)
apply_btn = widgets.Button(description="Apply")
close_btn = widgets.Button(description="Close")
output_widget = widgets.Output()
with output_widget:
    print("Draw a polygon on the map. Then, \nclick on the 'Apply' button")
```

#### Implementing the Query Logic

```{code-cell} ipython3
def on_apply_btn_click(change):
    with output_widget:
        try:
            output_widget.clear_output()
            if len(m.draw_features_selected) > 0:
                geojson = m.draw_features_selected[0]["geometry"]
            df = con.sql(
                f"""
            SELECT * EXCLUDE (geometry), ST_AsText(geometry) AS geometry FROM h3_res4_geo
            WHERE ST_Intersects(geometry, ST_GeomFromGeoJSON('{json.dumps(geojson)}'));
            """
            ).df()
            gdf = leafmap.df_to_gdf(df)
            if "H3 Hexagon" in m.layer_dict:
                m.remove_layer("H3 Hexagon")

            if outline_chk.value:
                outline_color = "rgba(255, 255, 255, 255)"
            else:
                outline_color = "rgba(255, 255, 255, 0)"

            if checkbox.value:
                m.add_data(
                    gdf,
                    column="building_count",
                    scheme="JenksCaspall",
                    cmap=colormap_dropdown.value,
                    k=class_slider.value,
                    outline_color=outline_color,
                    name="H3 Hexagon",
                    before_id=m.first_symbol_layer_id,
                    extrude=True,
                    fit_bounds=False,
                    add_legend=False,
                )
            else:
                m.add_data(
                    gdf,
                    column="building_count",
                    scheme="JenksCaspall",
                    cmap=colormap_dropdown.value,
                    k=class_slider.value,
                    outline_color=outline_color,
                    name="H3 Hexagon",
                    before_id=m.first_symbol_layer_id,
                    fit_bounds=False,
                    add_legend=False,
                )

            m.remove_from_sidebar(name="Legend")
            m.add_legend_to_sidebar(
                title="Building Count",
                legend_dict=m.legend_dict,
            )
        except Exception as e:
            with output_widget:
                print(e)


def on_close_btn_click(change):
    m.remove_from_sidebar(name="H3 Hexagonal Grid")
```

#### Wiring Up the Interface

```{code-cell} ipython3
apply_btn.on_click(on_apply_btn_click)
close_btn.on_click(on_close_btn_click)

widget = widgets.VBox(
    [
        widgets.HBox([checkbox, outline_chk]),
        colormap_dropdown,
        class_slider,
        widgets.HBox([apply_btn, close_btn]),
        output_widget,
    ]
)
m.add_to_sidebar(widget, label="H3 Hexagonal Grid", widget_icon="mdi-hexagon-multiple")
```

#### How Voilà Transforms This Notebook

### Exploring the File Structure of the Space

### Updating the Hugging Face Space

#### Updating the Space from the Hugging Face Website

#### Updating the Space from the Command Line

```bash
git clone https://huggingface.co/spaces/YOUR-USERNAME/duckdb-voila
cd duckdb-voila
```

```bash
voila notebooks/
```

```bash
voila notebooks/ --strip_sources=False
```

## Creating an Advanced Web Application with Solara

### Understanding Solara's Architecture

### Development and Deployment Flexibility

### Reactive Programming Model

### Native ipywidgets Integration

### Using a Solara Template for DuckDB

### Exploring the File Structure of the Solara Web App

### Introduction to Solara Components

#### Creating Your First Solara Component

```{code-cell} ipython3
import solara
import leafmap.maplibregl as leafmap


def create_map():

    m = leafmap.Map(
        style="liberty",
        projection="globe",
        height="750px",
        zoom=2.5,
        add_sidebar=True,
        sidebar_visible=True,
    )
    return m


@solara.component
def Page():
    m = create_map()
    return m.to_solara()


Page()
```

#### Understanding the Component Structure

#### The Solara Development Pattern

### Creating a New Page

#### Page Naming Convention

#### Creating a Building Density Visualization Page

```{code-cell} ipython3
import solara
import leafmap.maplibregl as leafmap


def create_map():

    m = leafmap.Map(
        style="dark-matter",
        projection="globe",
        height="750px",
        zoom=2.5,
        add_sidebar=True,
        sidebar_visible=True,
    )
    m.add_basemap("Esri.WorldImagery", before_id=m.first_symbol_layer_id, visible=False)
    url = "https://data.gishub.org/duckdb/h3_res4_geo.parquet"
    gdf = leafmap.read_vector(url)
    m.add_data(
        gdf,
        column="building_count",
        scheme="JenksCaspall",
        cmap="inferno",
        outline_color="rgba(255, 255, 255, 0)",
        name="H3 Hexagon",
        before_id=m.first_symbol_layer_id,
    )

    return m


@solara.component
def Page():
    m = create_map()
    return m.to_solara()
```

#### Understanding the Buildings Page

### Running the Solara Web App Locally

```bash
cd duckdb-solara
solara run pages/
```

### Deploying to Hugging Face Spaces

#### Making Changes and Deploying

```bash
git add .
git commit -m "Add buildings visualization page"
git push
```

#### Monitoring the Deployment

#### Sharing Your Application

## Key Takeaways

## Exercises

### Exercise 1: Create a World Population Voilà Dashboard

### Exercise 2: Build a Multi-Dataset Solara Application

### Exercise 3: Interactive DuckDB Query Dashboard (Advanced)
