+++
author = "Andreas Lay"
title = "Ibis: Build your SQL Queries via Python ‒ One API for nearly 20 Backends"
date = "2025-03-30"
description = "Build testable & maintainable SQL with Ibis"
tags = ["Python", "ibis"]
categories = ["Python", "Data Engineering"]
ShowToc = true
TocOpen = true
+++

## What is Ibis and Why Would You Use It?

[Ibis](https://ibis-project.org/) is a backend agnostic query builder / dataframe API for Python. Think of it as an interface for dynamically generating & executing SQL queries via Python. `Ibis` provides a unified API for a variety of backends like Snowflake, BigQuery, DuckDB, polars or PySpark.

But why would this be useful? I mainly use it for:

- Generating complex dynamic queries where (Jinja-)templating would becomes too messy
- Writing SQL generation as self-contained, easily testable Python functions
- Switching out a production OLAP database (like Snowflake) at test time with a local & version controlled DuckDB instance containing test data

If you know SQL you already know Ibis.

For those who want to skip ahead directly to the code example: [Here you can find the correspoding gist](https://gist.github.com/layandreas/d2050e3113c4cbd1627a3cca58040eb3). You may run this example directly using [uv](https://github.com/astral-sh/uv) with:

```shell
uv run https://gist.githubusercontent.com/layandreas/d2050e3113c4cbd1627a3cca58040eb3/raw/1acc913dd1042a06017fce16cffa94c4b67d870b/ibis-penguins.py
```

## Query Builder vs. Object-relational Mapper (ORM)

If you're coming from a web development background you're probably familiar with ORM's like for instance [SQLAlchemy](https://docs.sqlalchemy.org/en/20/orm/) or the [Django ORM](https://docs.djangoproject.com/en/5.1/topics/db/queries/). `Ibis` is **not** an ORM but a query builder. While ORM's will map your database objects to objects in your program `Ibis` will not do such a thing! Rather think of using `Ibis` in terms of _"writing raw SQL queries but in Python"_.

## An Example using the DuckDB Backend

### Create a New DuckDB Table

Let's use the [DuckDB backend](https://duckdb.org/docs/stable/guides/python/ibis.html) and load an example dataset into a table:

```python
# This will create a new DuckDB database in your working directory
# if it doesn't exist yet
con = ibis.connect("duckdb://penguins.ddb")
con.create_table(
    "penguins", ibis.examples.penguins.fetch().to_pyarrow(), overwrite=True
)

print("Tables in duckdb:")
print("\n")
print(con.list_tables())
print("\n")

penguins = con.table("penguins")
print("Initial penguins table:")
print("\n")
print(penguins.to_pandas())
print("\n")
```

Output:

```shell
Tables in duckdb:

['penguins']


Initial penguins table:

       species     island  bill_length_mm  bill_depth_mm  flipper_length_mm  body_mass_g     sex  year
0       Adelie  Torgersen            39.1           18.7              181.0       3750.0    male  2007
1       Adelie  Torgersen            39.5           17.4              186.0       3800.0  female  2007
2       Adelie  Torgersen            40.3           18.0              195.0       3250.0  female  2007
3       Adelie  Torgersen             NaN            NaN                NaN          NaN    None  2007
4       Adelie  Torgersen            36.7           19.3              193.0       3450.0  female  2007
..         ...        ...             ...            ...                ...          ...     ...   ...
339  Chinstrap      Dream            55.8           19.8              207.0       4000.0    male  2009
340  Chinstrap      Dream            43.5           18.1              202.0       3400.0  female  2009
341  Chinstrap      Dream            49.6           18.2              193.0       3775.0    male  2009
342  Chinstrap      Dream            50.8           19.0              210.0       4100.0    male  2009
343  Chinstrap      Dream            50.2           18.7              198.0       3775.0  female  2009

[344 rows x 8 columns]
```

### Window Functions: Row Number Over Partition

My most used feature when it comes to SQL is window functions, in particular `row_number() over(partition by ... order by ...)` to select one observation within a group. Now we are going to select the penguins with the longest bill length by species:

```python
penguins = (
    penguins.mutate(
        rn=ibis.row_number().over(
            group_by=[ibis._["species"]],
            order_by=ibis.desc(ibis._["bill_length_mm"]),
        ),
    )
    .filter(ibis._["rn"] == 0)
    .drop("rn")
    .alias("select_longest_bill_by_species")
)

print("Penguins with longest bill length by species:")
print("\n")
print(penguins.to_pandas())
print("\n")
```

Output:

```shell
Penguins with longest bill length by species:


     species     island  bill_length_mm  bill_depth_mm  flipper_length_mm  body_mass_g     sex  year
0     Gentoo     Biscoe            59.6           17.0                230         6050    male  2007
1  Chinstrap      Dream            58.0           17.8                181         3700  female  2007
2     Adelie  Torgersen            46.0           21.5                194         4200    male  2007
```

### Add a JSON Column

You might need to prepare your data for publishing via an API endpoint or produce a Kafka message. Usually this data is served as JSON. Let's add a new JSON column to our table:

```python
penguins = penguins.mutate(
    json_payload=ibis.struct(
        [("species", ibis._["species"]), ("island", ibis._["island"])]
    )
).alias("add_json_payload_column")

print("Penguins data with JSON payload column:")
print("\n")
print(penguins.to_pandas())
print("\n")
```

Output:

```shell
Penguins data with JSON payload column:

     species     island  bill_length_mm  bill_depth_mm  ...  body_mass_g     sex  year                                  json_payload
0     Gentoo     Biscoe            59.6           17.0  ...         6050    male  2007     {'species': 'Gentoo', 'island': 'Biscoe'}
1  Chinstrap      Dream            58.0           17.8  ...         3700  female  2007   {'species': 'Chinstrap', 'island': 'Dream'}
2     Adelie  Torgersen            46.0           21.5  ...         4200    male  2007  {'species': 'Adelie', 'island': 'Torgersen'}

[3 rows x 9 columns]
```

### The Compiled SQL

So what's happening under the hood? `Ibis` uses [SQLGlot](https://github.com/tobymao/sqlglot) to compile your expressions into SQL. Running `print(ibis.to_sql(penguins))` will give you the compiled raw query string. The compiled SQL for our example looks like this:

```sql
WITH "select_longest_bill_by_species" AS (
  SELECT
    "t2".*
    EXCLUDE ("rn")
  FROM (
    SELECT
      *
    FROM (
      SELECT
        "t0"."species",
        "t0"."island",
        "t0"."bill_length_mm",
        "t0"."bill_depth_mm",
        "t0"."flipper_length_mm",
        "t0"."body_mass_g",
        "t0"."sex",
        "t0"."year",
        ROW_NUMBER() OVER (PARTITION BY "t0"."species" ORDER BY "t0"."bill_length_mm" DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) - 1 AS "rn"
      FROM "penguins" AS "t0"
    ) AS "t1"
    WHERE
      "t1"."rn" = 0
  ) AS "t2"
), "add_json_payload_column" AS (
  SELECT
    "t4"."species",
    "t4"."island",
    "t4"."bill_length_mm",
    "t4"."bill_depth_mm",
    "t4"."flipper_length_mm",
    "t4"."body_mass_g",
    "t4"."sex",
    "t4"."year",
    {'species': "t4"."species", 'island': "t4"."island"} AS "json_payload"
  FROM "select_longest_bill_by_species" AS "t4"
)
SELECT
  *
FROM "add_json_payload_column"
```

_**Note:** The CTE's (`"select_longest_bill_by_species" AS`, `"add_json_payload_column" AS`) are compiled as we used the `alias`-method. This was done mostly for readability purposes and come in handy when you're looking at the generated SQL for debugging purposes. Otherwise `ibis` would have just generated nested subqueries._

Now the nice thing is that you could switch out the `DuckDB` backend with any of the other supported backends without any code changes. The generated SQL will be valid for the chosen backend. In our example in particular the generation of the JSON column

```sql
{'species': "t4"."species", 'island': "t4"."island"} AS "json_payload"
```

will most likely compile to a different expression for other backends.

### Full Example

Here is the full example. To make the data pipeline more composable I wrapped the transformations into the respective functions `_select_longest_bill_by_species` and `_add_json_payload_column`. The `_pipe` helper function was just added for convenience & readability.

_**Tip:** Using [uv](https://github.com/astral-sh/uv) you can run this script with `uv run https://gist.githubusercontent.com/layandreas/d2050e3113c4cbd1627a3cca58040eb3/raw/1acc913dd1042a06017fce16cffa94c4b67d870b/ibis-penguins.py`_

[Gist](https://gist.github.com/layandreas/d2050e3113c4cbd1627a3cca58040eb3)

```python
# /// script
# requires-python = ">=3.13"
# dependencies = [
#     "duckdb==1.2.1",
#     "gcsfs>=2025.3.0",
#     "ibis-framework[duckdb]==10.4.0",
#     "pins==0.8.7",
# ]
# ///

import ibis
from typing import Callable


def main() -> None:
    con = ibis.connect("duckdb://penguins.ddb")
    con.create_table(
        "penguins", ibis.examples.penguins.fetch().to_pyarrow(), overwrite=True
    )

    print("Tables in duckdb:")
    print("\n")
    print(con.list_tables())
    print("\n")

    penguins = con.table("penguins")
    print("Initial penguins table:")
    print("\n")
    print(penguins.to_pandas())
    print("\n")

    penguins = _pipe(
        initial_table=penguins,
        funcs=[_select_longest_bill_by_species, _add_json_payload_column],
    )

    print("Compiled query:")
    print("\n")
    print(ibis.to_sql(penguins))
    print("\n")

    con.disconnect()


def _select_longest_bill_by_species(penguins: ibis.Table) -> ibis.Table:
    penguins = (
        penguins.mutate(
            rn=ibis.row_number().over(
                group_by=[ibis._["species"]],
                order_by=ibis.desc(ibis._["bill_length_mm"]),
            ),
        )
        .filter(ibis._["rn"] == 0)
        .drop("rn")
        .alias("select_longest_bill_by_species")
    )

    print("Penguins with longest bill length by species:")
    print("\n")
    print(penguins.to_pandas())
    print("\n")

    return penguins


def _add_json_payload_column(penguins: ibis.Table) -> ibis.Table:
    penguins = penguins.mutate(
        json_payload=ibis.struct(
            [("species", ibis._["species"]), ("island", ibis._["island"])]
        )
    ).alias("add_json_payload_column")

    print("Penguins data with JSON payload column:")
    print("\n")
    print(penguins.to_pandas())
    print("\n")

    return penguins


def _pipe(
    initial_table: ibis.Table, funcs: list[Callable[[ibis.Table], ibis.Table]]
) -> ibis.Table:
    processed_table = initial_table
    for func in funcs:
        processed_table = func(processed_table)

    return processed_table


if __name__ == "__main__":
    main()
```

## Usage Patterns

### Ibis vs. dbt?

For most pre-processing our data I will generally stick to using raw SQL with [dbt](https://docs.getdbt.com/docs/introduction) with some minimal Jinja templating. `Ibis` will then come in at the _"last mile"_: Some final preparations before loading data into memory e.g. for model fitting or dashboarding.

Especially for the dashboarding / frontend tool use cases where a user might have a lot of input options it's extremely useful to be able to programmatically generate queries. While Jinja templating is an option I find using `Ibis` is much easier to write a testable & maintainable code base.

### Testing

Last but not least for me `Ibis` has proven itself extremely valuable for testing use cases:

- I will create a local DuckDB database and ingest some test data taken corresponding to our production database (like Snowflake or BigQuery).

- This database is version controlled and checked into our repository.

- At test time we will replace the production database connection with a DuckDB connection and run our tests against the DuckDB test data.
