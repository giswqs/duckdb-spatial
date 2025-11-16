# Spatial Data Management with DuckDB

[![image](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/giswqs/duckdb-spatial/HEAD)
[![image](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/giswqs/duckdb-spatial/blob/main)

<!-- [![Docker Image](https://img.shields.io/badge/docker-giswqs%2Fpygis%3Abook-blue?logo=docker)](https://hub.docker.com/r/giswqs/pygis/tags) -->

<!-- [![Amazon](https://img.shields.io/badge/Buy%20on-Amazon-orange?logo=amazon&logoColor=white)](https://amazon.com/dp/B0FFW34LL3) -->

## Introduction

Welcome to the official website for **_Spatial Data Management with DuckDB: From SQL Basics to Advanced Geospatial Analytics_**. This website contains all the code examples featured in the book, designed to help you learn and apply DuckDB for geospatial analysis.

## Get the Book

- 🇺🇸 **English Full-Color Print Edition (446 pages):** Available on Amazon ([link](https://www.amazon.com/dp/B0G2JFMFFC))

- 🇺🇸 **English PDF Edition (441 pages):** Available on Leanpub ([link](https://leanpub.com/duckdb))

<!-- - 🇨🇳 **Chinese PDF Edition (430 pages):** 中文电子版可在 Leanpub 购买 ([link](https://leanpub.com/duckdb-zh))

- 🇲🇽 **Spanish PDF Edition (430 pages):** Edición en español disponible en Leanpub ([link](https://leanpub.com/duckdb-es))
 -->

## Cite the Book

If you use this book in your research or teaching, please consider citing it as follows:

> Wu, Q. (2025). _Spatial Data Management with DuckDB: From SQL Basics to Advanced Geospatial Analytics_. Independently published. PDF edition ISBN 979-8993859705; Print edition ISBN 979-8274710572. [https://duckdb.gishub.org](https://duckdb.gishub.org)

![book cover](https://assets.gishub.org/images/duckdb-book-cover.webp)

## Table of Contents

<!-- To download a PDF version of the Table of Contents, please visit <https://duckdb.gishub.org/book-toc.pdf>. -->

- **Preface**

  - Introduction
  - Who This Book Is For
  - What This Book Covers
  - Getting the Most Out of This Book
  - Conventions Used in This Book
  - Downloading the Code Examples
  - Video Tutorials and Supplementary Resources
  - Community and Feedback
  - Acknowledgments
  - About the Author
  - Licensing and Copyright

- **DuckDB Foundations**

  - Getting Started with DuckDB
  - Essential SQL for Spatial Analysis
  - DuckDB Python Integration

- **Spatial Data Operations**

  - Loading Spatial Data Formats
  - Exporting and Converting Spatial Data
  - Geometry Operations and Functions
  - Spatial Queries and Relationships
  - Advanced Spatial Joins
  - Interactive Data Visualization
  - Working with Vector Tiles and PMTiles

- **Real-World Geospatial Analytics**

  - Analyzing the US National Wetlands Inventory
  - Analyzing Global Building Footprints
  - Analyzing NYC Taxi Data
  - Developing Interactive Dashboards with Voilà and Solara

## How to Run Code Examples

The code examples are organized into folders, each corresponding to a chapter in the book. The code examples are written in Python and can be run using MyBinder, Google Colab, or Docker.

<!-- Follow this [video tutorial](https://www.youtube.com/embed/6GwMoV4LOiU) to learn how to run the code examples. -->

### Using MyBinder

The code examples can be run using MyBinder.

[![image](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/giswqs/duckdb-spatial/HEAD)

### Using Google Colab

The code examples can be run using Google Colab.

[![image](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/giswqs/duckdb-spatial/blob/main)

<!-- ### Using Docker

The code examples can be run using Docker. There are two Docker [images](https://hub.docker.com/r/giswqs/pygis/tags) available:

A lightweight docker image without Apache Sedona:

```bash
docker pull giswqs/pygis:book
docker run -it -p 8888:8888 -v $(pwd):/app/workspace giswqs/pygis:book
```

A docker image with Apache Sedona:

```bash
docker pull giswqs/pygis:sedona
docker run -it -p 8888:8888 -p 4040:4040 -p 8080:8080 -p 8081:8081 -p 7077:7077 -p 8085:8085 -v $(pwd):/app/workspace giswqs/pygis:sedona
``` -->

## Video Tutorials

Complementing the written content, this book is supported by a comprehensive series of video tutorials that walk through key concepts and provide additional examples: <https://tinyurl.com/duckdb-spatial-videos>.

The videos are designed to complement, not replace, the written material. They're particularly helpful for:

- Visual learners who benefit from seeing code being written and executed
- Understanding complex concepts through multiple explanations
- Learning about the development workflow and best practices
- Seeing how to approach problems and debug issues

The playlist is organized to follow the book's structure. You can watch them in order as you progress through the book, or jump to specific topics as needed.

The videos were created in Fall 2023 when I was teaching the [**Spatial Data Management**](https://geog-414.gishub.org) course at the University of Tennessee. Although the course has concluded, the videos remain relevant and can be used as a reference for the book. Additional videos will be added in the future.

## About the Author

Dr. Qiusheng Wu is an Associate Professor in the Department of Geography & Sustainability at the University of Tennessee, Knoxville. He is also an Amazon Scholar. Dr. Wu’s research focuses on advancing open-source geospatial analytics through cloud computing and GeoAI. He is the creator and maintainer of several widely used open-source Python packages, including [Geemap](https://geemap.org), [Leafmap](https://leafmap.org), [SAMGeo](https://samgeo.gishub.org), and [GeoAI](https://opengeoai.org), which integrate cloud-based geospatial platforms with AI-powered analysis and visualization. Dr. Wu’s work bridges remote sensing, Earth observation, and artificial intelligence to make large-scale geospatial data more accessible, reproducible, and intelligent for researchers, educators, and practitioners worldwide. His open-source projects can be found on GitHub at <https://github.com/opengeos>.

## Licensing and Copyright

This book embraces the principles of open science and open education. To support transparency, learning, and reuse, the **code examples** in this book are released under a [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) license. This means you are free to copy, modify, and distribute the code, even for commercial purposes, as long as appropriate credit is given.

Please attribute code usage by citing the book or linking to the GitHub repository:

> Wu, Q. (2025). _Spatial Data Management with DuckDB: From SQL Basics to Advanced Geospatial Analytics_. Independently published. PDF edition ISBN 979-8993859705; Print edition ISBN 979-8274710572. [https://duckdb.gishub.org](https://duckdb.gishub.org)

Let me know if you want APA, Chicago, MLA, BibTeX, or multiple styles.

While the code is freely available, the **text, figures, and images** in this book are **copyrighted** by the author and may not be reproduced, redistributed, or modified without explicit permission. This includes all written content, custom diagrams, and embedded visualizations unless otherwise noted.

If you wish to reuse or adapt any non-code material from the book (for example, for teaching, presentations, or publications), please contact the author to request permission.

This dual licensing approach helps balance open access to learning materials with the protection of original creative work. Thank you for respecting these terms and supporting the open-source geospatial community.
