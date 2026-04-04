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
# Getting Started with DuckDB

## Introduction

## Learning Objectives

## What Makes DuckDB Different from Traditional Databases

## When (and When Not) to Use DuckDB for Spatial Work

### When to Use DuckDB

### When _Not_ to Use DuckDB

## Installing DuckDB CLI and Running Your First Query

### Windows Installation

   ```bash
   duckdb --version
   ```

### macOS or Linux Installation

```bash
curl https://install.duckdb.org | sh
```

```bash
duckdb
```

## Installing the DuckDB Python Client

### Installing DuckDB Using pip

```bash
pip install duckdb
```

#### Installing uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```bash
pip install uv
```

#### Basic uv Usage

```bash
# Navigate to your project directory
cd /path/to/your/project

# Create a virtual environment
uv venv

# Create with specific Python version
uv venv --python 3.12

# Activate the environment (varies by OS)
# On macOS/Linux:
source .venv/bin/activate
# On Windows:
.venv\Scripts\activate
```

```bash
# Install packages
uv pip install jupyterlab leafmap duckdb duckdb-engine jupyter-duckdb jupysql
```

### Installing DuckDB Using Conda

#### Installing Miniconda on Windows

```bash
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe -o .\miniconda.exe
start /wait "" .\miniconda.exe /S
del .\miniconda.exe
```

#### Installing Miniconda on macOS

```bash
mkdir -p ~/miniconda3
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh -o ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
```

```bash
mkdir -p ~/miniconda3
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh -o ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
```

```bash
source ~/miniconda3/bin/activate
conda init --all
```

#### Installing Miniconda on Linux

```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
```

```bash
source ~/miniconda3/bin/activate
conda init --all
```

#### Installing DuckDB in the Conda environment

```bash
conda create -n geo python=3.12
conda activate geo
conda install -c conda-forge python-duckdb duckdb-engine jupysql leafmap
```

### Verifying Installation

```bash
jupyter lab
```

## Installing Visual Studio Code

### Download and Installation

### Installing VS Code Extensions

```bash
code --install-extension ms-python.python
code --install-extension ms-toolsai.jupyter
code --install-extension RandomFractalsInc.duckdb-sql-tools
```

### Installing DuckDB Extensions

```bash
duckdb
```

## Using the DuckDB UI

```bash
duckdb -ui
```

## Installing DBeaver SQL IDE

## Key Takeaways

## Exercises

### Exercise 1: Installation Verification

### Exercise 2: Installing and Loading Extensions

### Exercise 3: Python Environment Setup

### Exercise 4: DBeaver SQL Editor Setup

### Exercise 5: Exploring the DuckDB UI
