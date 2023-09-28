# COLD Cases Export

This codebase is used to create a ML-friendly dataset out of
the [bulk data downloads](https://www.courtlistener.com/help/api/bulk-data/) available
on [courtlistener.com](https://courtlistener.com), a website run by the [Free Law Project](https://free.law/).

## Running:

- Install a relatively new python (Recommended: [pyenv](https://github.com/pyenv/pyenv)).
- Run `pip install pipx`.
- Run `pipx install poetry`.
- In the project directory, run `poetry install`.
- To obtain the latest data, run `poetry run python courtlistener_export/download.py`. This will download the latest
  versions of each of the bzip2 files needed to compile the dataset to the `data/` subdirectory.
- Install [sdkman](https://sdkman.io/).
- Run `sdk install java 11.0.18-tem`.
- Run `sdk install spark 3.4.0`
- To run the transformation process, run:

```spark-submit --driver-memory 30g --master "local[8]" cold_cases_export/export.py data```

Note that the values for the number of cores in the `--master` and `--driver-memory` interact with one another and
should be tuned to the system you run it on. These settings worked on a high-end Macbook Pro with an M2 Pro and 32 GB of
RAM.

The export script will first convert the data to [Parquet](https://parquet.apache.org/) format if this is the first time
processing it. Unfortunately, since the data is bzipped (slow to decompress) and this process cannot be parallelized
without some custom low-level code, because the csvs within (particularly the opinions data) have escaped line breaks
and the Spark CSV splitter depends on records beginning on individual lines.

The output of the export script will also show up in `data/`. In there, you will find two directories. One
called `cold.parquet` which contains the export data in Parquet, and another called `cold.jsonl` which contains the same
data in [JSON Lines](https://jsonlines.org/). The data will be split into multiple smaller files, based on the execution
plan of the Spark job and the value of `REPARTITION_FACTOR` in the export script.

## Exploring the data in Elasticsearch / Kibana:

Included in the project a rudimentary implementation of viewing the data in Kibana for exploration.

Docker is required to run the Elasticsearch and Kibana servers, unless you have a preexisting setup to use.

Additionally, you will need to have run the exporter process already, so `cold.parquet` exists in the `data` folder.

- Open a terminal window, `cd` into the `docker` folder, and run `docker compose up`. This should pull the Kibana and
  Elasticsearch containers, and run them both along with a transient contanier that establishes trust between the two.
- Wait for the containers complete initialization.
- In another terminal window, `cd` into the `cold-cases-export` directory and run this command:

```spark-submit --driver-memory 30g --master "local[8]" --packages org.elasticsearch:elasticsearch-spark-30_2.12:7.17.13 cold_cases_export/elasticsearch_load.py data```

- The indexing process takes roughly 40 minutes on this machine. This likely could be improved by tuning things like the
  number of shards in the Elasticsearch index and right sizing the number of spark executors.
- Once indexing completes, point your browser at [localhost:5601](https://localhost:5601/), accept the self-signed
  certificate, and log in with the credentials `elastic` / `changeme`.
- The easiest way to begin exploring the data is to head to the `Discover` page, under `Analytics`. You'll be prompted
  to specify a pattern for what indexes to use; you can specify `cold` which is the name of the index the indexer
  creates.
- The interface that appears will allow you to query the data and see things like facet counts for specific field values
  and build charts and visualizations.
- Hit `Control-C` `in the `docker compose` terminal to shut off the Elastic servers.