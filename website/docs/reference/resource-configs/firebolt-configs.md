---
title: "Firebolt configurations"
id: "firebolt-configs"
---

## Using dbt-firebolt



### Model Configuration for Fact Tables

A dbt model can be created as a Firebolt fact table and configured using the following syntax:

<Tabs
  groupId="config-languages"
  defaultValue="project-yaml"
  values={[
    { label: 'Project file', value: 'project-yaml', },
    { label: 'Config block', value: 'config', },
  ]
}>

<TabItem value="project-yaml">
<File name='dbt_project.yml'>

```yaml
models:
  <resource-path>:
    +materialized: table
    +table_type: fact
    +primary_index: [ <column-name>, ... ]
    +indexes:
      - type: aggregating
        key_column: [ <column-name>, ... ]
        aggregation: [ <agg-sql>, ... ]
      ...
```

</File>
</TabItem>

<TabItem value="config">
<File name='models/<model_name.sql>'>

```jinja
{{ config(
    materialized = "table"
    table_type = "fact"
    primary_index = [ "<column-name>", ... ],
    indexes = [
      {
        type = "aggregating"
        key_column = [ "<column-name>", ... ],
        aggregation = [ "<agg-sql>", ... ],
      },
      ...
    ]
) }}
```

</File>
</TabItem>
</Tabs>


#### Fact Table Configurations

| Configuration     | Description                                                                               |
|-------------------|-------------------------------------------------------------------------------------------|
| `materialized`    | How the model will be materialized into Firebolt. Must be `table` to create a fact table. |
| `table_type`      | Whether the materialized table will be a [fact or dimension](https://docs.firebolt.io/concepts/working-with-tables#fact-and-dimension-tables) table. |
| `primary_index`   | Sets the primary index for the fact table using the inputted list of column names from the model. Required for fact tables. |
| `indexes`         | A list of aggregating indexes to create on the fact table. |
| `type`            | Specifies whether the index is an aggregating index or join index. Join indexes only apply to dimension tables, so for fact tables set to `aggregating`. |
| `key_column`      | Sets the grouping of the aggregating index using the inputted list of column names from the model. |
| `aggregation`     | Sets the aggregations on the aggregating index using the inputted list of SQL agg expressions. |

#### Example of a Fact Table With an Aggregating Index (Config Block Syntax)

```
{{ config(
    materialized = "table",
    table_type = "fact",
    primary_index = "id",
    indexes = [
      {
        type: "aggregating",
        key_column: "order_id",
        aggregation: ["COUNT(DISTINCT status)", "AVG(customer_id)"]
      }
    ]
) }}
```

### Model Configuration for Dimension Tables

A dbt model can be materialized as a Firebolt dimension table and configured using the following syntax:

<Tabs
  groupId="config-dimension"
  defaultValue="project-yaml"
  values={[
    { label: 'Project file', value: 'project-yaml', },
    { label: 'Property file', value: 'property-yaml', },
    { label: 'Config block', value: 'config', },
  ]
}>

<TabItem value="project-yaml">
<File name='dbt_project.yml'>

```yaml
models:
  <resource-path>:
    +materialized: table
    +table_type: dimension
    +indexes:
      - type: join
        join_column: <column-name>
        dimension_column: [ <column-name>, ... ]
      ...
```

</File>
</TabItem>

<TabItem value="property-yaml">
<File name='models/properties.yml'>
Property file syntax

```yaml
models:
  - name: <model-name>
    config:
      materialized: table
      table_type: dimension
      indexes:
        - type: join
          join_column: <column-name>
          dimension_column: [ <column-name>, ... ],
        ...
```

</File>
</TabItem>

<TabItem value="config">
<File name='models/<model_name>.sql'>

Config block syntax

```jinja
{{ config(
    materialized = "table",
    table_type = "dimension",
    indexes = [
      {
        type = "join",
        join_column = "<column-name>",
        dimension_column: [ "<column-name>", ... ]
      },
      ...
    ],
) }}
```

</File>
</TabItem>
</Tabs>

#### Dimension Table Configurations

| Configuration      | Description                                                                               |
|--------------------|-------------------------------------------------------------------------------------------|
| `materialized`     | How the model will be materialized into Firebolt. Must be `table` to create a dimension table. |
| `table_type`       | Whether the materialized table will be a [fact or dimension](https://docs.firebolt.io/concepts/working-with-tables#fact-and-dimension-tables) table. |
| `indexes`          | A list of join indexes to create on the dimension table. |
| `type`             | Specifies whether the index is an aggregating index or join index. Aggregating indexes only apply to fact tables, so for dimension tables set to `join`. |
| `join_column`      | Sets the join key of the join index using the inputted column name from the model. |
| `dimension_column` | Sets the columns to be loaded into memory on the join index using the inputted list of column names from the mode. |

#### Example of a Dimension Table With a Join Index (Config Block Syntax)

```
{{ config(
    materialized = "table",
    table_type = "dimension",
    indexes = [
      {
        type: "join",
        join_column: "order_id",
        dimension_column: ["customer_id", "status"]
      }
    ]
) }}
```

### How Aggregating Indexes and Join Indexes Are Named

In dbt-firebolt, you do not provide names for aggregating indexes and join indexes; they are named programmatically. dbt will generate index names using the following convention:
```
<table-name>__<key-column>__<index-type>_<unix-timestamp-at-execution>
```

For example, a join index could be named `my_users__id__join_1633504263` and an aggregating index could be named `my_orders__order_date__aggregating_1633504263`.

### Managing Ingestion via External Tables

`dbt-firebolt` supports dbt's [external tables feature](https://docs.getdbt.com/reference/resource-properties/external), which allows dbt to manage the table ingestion process from S3 into Firebolt. This is an optional feature but can be highly convenient depending on your use case.

More information on using external tables including properly configuring IAM can be found in the Firebolt [documentation](https://docs.firebolt.io/sql-reference/commands/ddl-commands#create-external-table).

#### Installation of External Tables Package

To install and use `dbt-external-tables` with Firebolt, you must:

1. Add this package to your packages.yml:

    ```yaml
    packages:
      - package: dbt-labs/dbt_external_tables
        version: <version>
    ```

2. Add these fields to your `dbt_project.yml`:

    ```yaml
    dispatch:
      - macro_namespace: dbt_external_tables
        search_order: ['dbt', 'dbt_external_tables']
    ```

3. Pull in the `packages.yml` dependencies by calling `dbt deps`.

