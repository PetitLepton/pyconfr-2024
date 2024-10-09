---
title: "Python & kedro<br>pour<br>l'agriculture de précision"
# subtitle: "Étude de cas Inclusive Brains"
author: "Paul Arnaud & Flavien Lambert<br>Data Engineering@Sencrop"
format:
  revealjs:
    title-block-banner: true
    width: 1244
    toc: true
    toc-depth: 1
    toc-title: Agenda
    toc-location: right
    theme: [custom.scss]
    logo: sencrop_logo_horizontal_RVB.png
    menu: false
    progress: false
    scrollable: true
    navigation-mode: grid
    highlight-style: github
    code-line-numbers: false
---
# sencrop

## sencrop {background-opacity=0.25 background-image="raincrop.jpg"}

- 35000 stations météorolgiques réparties sur toute l'Europe

```{=html}
<div class="container">
  <img src="./grey-temp-hum-o.png" width="70em"/>
  <p>température de l'air & hygrométrie</p>
  <img src="./grey-wind.png" width="70em"/>
  <p>direction et vitesse du vent</p>
  <img src="./grey-rain-o.png" width="70em"/>
  <p>pluviométrie</p>
  <img src="./grey-dew-point.png" width="70em"/>
  <p>point de rosée</p>
</div>
```

## spatialisation 

Est-ce que l'on peut fournir de la donnée météorologique de qualité sur n'importe quel localisation sur le territoire ?

:::{.columns}

:::{.column #vcenter}

- comparaison des mesures de stations avec les médianes sur les grilles `h3`

:::
:::{.column #vcenter}

![](./pentagon_hexagon_children.png)

:::
:::

# kedro en trois mots

## kedro en trois mots

:::{.fragment}

- librairie de transformations de données
  - découplage entre les sources de données et les transformations
:::

:::{.fragment}

- trois concepts principaux
  - `node` : fonction — au sens Python — avec un/des `dataset` d'entrée et un/des `dataset` de sortie
  - `pipeline` : Direct Acyclic Graph composé de `node`
  - `catalog` : un ensemble de `dataset`

:::

## nodes & pipelines

:::: {.columns}
:::{.column}

```{.python filename="src/pipelines/weather_conditions/pipeline.py"}
from kedro.pipeline import Pipeline, node, pipeline

from .nodes import create_model_input_table, preprocess_companies, preprocess_shuttles


def create_pipeline(**kwargs) -> Pipeline:
    return pipeline(
        [
            node(
                func=preprocess_companies,
                inputs="companies",
                outputs="preprocessed_companies",
                name="preprocess_companies_node",
            ),
            ...
            node(
                func=create_model_input_table,
                inputs=["preprocessed_shuttles", "preprocessed_companies", "reviews"],
                outputs="model_input_table",
                name="create_model_input_table_node",
            ),
        ]
    )
```

:::
:::{.column}

```{.python filename="src/pipelines/weather_conditions/nodes.py"}
import pandas as pd

...

def preprocess_companies(companies: pd.DataFrame) -> pd.DataFrame:
    """Preprocesses the data for companies.

    Args:
        companies: Raw data.
    Returns:
        Preprocessed data, with `company_rating` converted to a float and
        `iata_approved` converted to boolean.
    """
    companies["iata_approved"] = _is_true(companies["iata_approved"])
    companies["company_rating"] = _parse_percentage(companies["company_rating"])
    return companies
```

:::
:::

## catalog & environment

- catalog : définition des `dataset` d'entrée et de sortie 🌱🍂
- environnement : ensemble du catalog et d'un jeu de paramètres

:::{.r-stack}
:::{.fragment .fade-in-then-out}

```{.yaml filename=/conf/base/catalog.yaml width="200px"}
companies:
  type: pandas.CSVDataset
  filepath: data/01_raw/companies.csv

preprocessed_companies:
  type: pandas.Parquedivataset
  filepath: data/02_intermediate/preprocessed_companies.pq
```

---

```sh
kedro run --pipeline my_pipeline --env base
```

:::

:::{.fragment .fade-in}

```{.yaml filename=/conf/test-local/catalog.yaml }
companies:
  type: pandas.CSVDataset
  filepath: data/01_raw/companies.csv

preprocessed_companies:
  type: pandas.CSVDataset
  filepath: data/02_intermediate/preprocessed_companies.csv
```

---

```sh
kedro run --pipeline my_pipeline --env test-local
```

:::
:::

## structure

:::

```bash
.
├── conf
│   ├── base
│   │   ├── catalog.yml
│   │   └── parameters.yml
│   └── local
│       └── credentials.yml
├── data
│   ├── 01_raw
│   └── ...
├── notebooks
└── src
    └── my_project
        ├── pipeline_registry.py
        └── pipelines
            └── my_pipeline
                ├── nodes.py
                └── pipeline.py
```

:::

# kedro: du développement à la production

## Dumas du tuyau : trois écrivains et un lecteur

## des tests à la production : une histoire de sources

:::{.columns}

:::{.column}

```{.yaml filename="/conf/test-measures/catalog.yml"}
locations:
  type: pandas.JSONDataset
  filepath: data/01_raw/test_measures/locations.json

formatted_measures_on_grids:
  type: pandas.CSVDataset
  filepath: data/03_primary/test_measures/measures_on_grids.csv
  load_args:
    parse_dates:
      - timestamp
```

:::
:::{.column}

```{.yaml filename="/conf/production/catalog.yml"}
locations:
  type: pandas.JSONDataset
  filepath: s3://virtual-stations/virtual-stations.json

formatted_measures_on_grids:
  type: datasets.ConfluentKafkaAvroDataset
  save_args:
    topic: h3_cells_time_series
    cluster:
      bootstrap_servers: ...
    schema_registry:
      url: ...
    schema:
      name: h3_cells_time_series
      type: record
      fields:
        - name: h3_cell_id
          type: string
        - name: h3_cell_resolution
          type: int
        ...
```

:::
:::

# vers l'agnoticisme ?

## la perspective du manager

- structure rigide : expérience de développement excellente
  - PoC to production in a few weeks
- réactivité de la communauté : Slack avec 2200 inscrit·es
- vers un pipeline agnostique
  - code des `node` dépend encore des libraries : `pandas`/`polars`/`spark`
  - nouvelles librairies pour palier cette dépendance : [`ibis`](https://ibis-project.org/), [`narwhal`](https://github.com/narwhals-dev/narwhals)
