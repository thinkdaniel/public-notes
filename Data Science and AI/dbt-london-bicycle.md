# Using dbt to model a London bicycle dataset

This lesson will guide you through the process of using dbt (data build tool) to model a dataset related to bicycle rides in London. This dataset is available at BigQuery Public Datasets, specifically in the `bigquery-public-data.london_bicycles` dataset.

https://console.cloud.google.com/marketplace/product/greater-london-authority/london-bicycles

## Create a dbt project

Run the following command to create a new dbt project:

```bash
dbt init london_bicycle
```

This will create a new directory called `london_bicycle` with the necessary files and directories for a dbt project.

```bash
04:12:24  Running with dbt=1.9.2
04:12:24
Your new dbt project "london_bicycle" was created!

For more information on how to configure the profiles.yml file,
please consult the dbt documentation here:

  https://docs.getdbt.com/docs/configure-your-profile

One more thing:

Need help? Don't hesitate to reach out to us via GitHub issues or on Slack:

  https://community.getdbt.com/

Happy modeling!

04:12:24  Setting up your profile.
Which database would you like to use?
[1] bigquery

(Don't see the one you want? https://docs.getdbt.com/docs/available-adapters)

Enter a number: 1
[1] oauth
[2] service_account
Desired authentication method option (enter a number): 1
project (GCP project id): dsai-test-project
dataset (the name of your dbt dataset): london_bicycle
threads (1 or more): 4
job_execution_timeout_seconds [300]:
[1] US
[2] EU
Desired location option (enter a number): 2
04:18:54  Profile london_bicycle written to /Users/danielgoh/.dbt/profiles.yml using target's profile_template.yml and your supplied values. Run 'dbt debug' to validate the connection.
```

You will be prompted the following questions:

1. Which database would you like to use? (Select BigQuery)
2. Desired authentication method option (Select OAuth)

- oauth:
  - Uses your personal Google account credentials.
  - You will be prompted to log in via a browser.
  - Best for interactive use, development, or learning.
  - Credentials are tied to your user account and may expire.
- service_account:
  - Uses a JSON key file for a Google Cloud service account.
  - Suitable for automation, CI/CD, or production environments.
  - Credentials do not expire as quickly and are not tied to a specific user.
  - Requires you to create and manage a service account in GCP and download its key file.

3. project (GCP project id): (Enter your GCP project ID)
4. dataset (the name of your dbt dataset): (Enter `london_bicycle`)

- This is the name of the dataset where dbt will create its tables and views.

5. threads (1 or more): (Enter `4`)

- This specifies the number of threads dbt can use to run models in parallel, which can speed up execution.
- For most systems, 4 is a reasonable number, but you can adjust it based on your machine's capabilities and the size of your dataset.
- Setting this higher can improve performance, especially for larger datasets, but it may also increase resource usage.

6. job_execution_timeout_seconds [300]: (Press Enter to accept the default)

- Use the default, or set it higher if you expect long-running jobs.

7. Desired location option (Select `EU`)

- This specifies the geographic location of your BigQuery dataset.

## Analyze the dataset

To analyze the dataset, you can use the BigQuery console or any SQL client that supports BigQuery. The `london_bicycles` dataset contains 2 tables:

- `cycle_hire`:

| Field Name                     | Mode     | Type      | Description                                |
| ------------------------------ | -------- | --------- | ------------------------------------------ |
| rental_id                      | REQUIRED | INTEGER   |                                            |
| duration                       | NULLABLE | INTEGER   | Duration of the bike trip in seconds.      |
| duration_ms                    | NULLABLE | INTEGER   | Duration of the bike trip in milliseconds. |
| bike_id                        | NULLABLE | INTEGER   |                                            |
| bike_model                     | NULLABLE | STRING    |                                            |
| end_date                       | NULLABLE | TIMESTAMP |                                            |
| end_station_id                 | NULLABLE | INTEGER   |                                            |
| end_station_name               | NULLABLE | STRING    |                                            |
| start_date                     | NULLABLE | TIMESTAMP |                                            |
| start_station_id               | NULLABLE | INTEGER   |                                            |
| start_station_name             | NULLABLE | STRING    |                                            |
| end_station_logical_terminal   | NULLABLE | INTEGER   |                                            |
| start_station_logical_terminal | NULLABLE | INTEGER   |                                            |
| end_station_priority_id        | NULLABLE | INTEGER   |                                            |

- `cycle_stations`

| Field Name    | Mode     | Type    | Description |
| ------------- | -------- | ------- | ----------- |
| id            | NULLABLE | INTEGER |             |
| installed     | NULLABLE | BOOLEAN |             |
| latitude      | NULLABLE | FLOAT   |             |
| locked        | NULLABLE | STRING  |             |
| longitude     | NULLABLE | FLOAT   |             |
| name          | NULLABLE | STRING  |             |
| bikes_count   | NULLABLE | INTEGER |             |
| docks_count   | NULLABLE | INTEGER |             |
| nbEmptyDocks  | NULLABLE | INTEGER |             |
| temporary     | NULLABLE | BOOLEAN |             |
| terminal_name | NULLABLE | STRING  |             |
| install_date  | NULLABLE | DATE    |             |
| removal_date  | NULLABLE | DATE    |             |

From the schema, we can see that the `cycle_hire` table contains information about bike rentals, including the duration, bike ID, start and end stations, and timestamps. The `cycle_stations` table contains information about the bike stations, including their location, number of bikes and docks, and installation dates.

## Model the dataset

### Source Models

We start with defining source models. Source models are used to define the external raw data that we will be working with.

Add `models/sources.yml`:

```yaml
version: 2

sources:
  - name: london_bicycles # this is the name you'll use in {{ source() }}
    database: bigquery-public-data # NOT your project
    schema: london_bicycles # this is the dataset
    tables:
      - name: cycle_hire
      - name: cycle_stations
```

By defining the source models, we can reference them in our staging models using the `{{ source() }}` function. This allows us to easily manage and track the raw data we are working with. This is a better approach than referencing FROM `bigquery-public-data.london_bicycles.cycle_hire` directly in our SQL queries, as it provides a layer of abstraction and makes it easier to change the source if needed.

### Staging Models

The next concrete step is to create staging models. Staging models are used to clean and prepare the raw data for analysis. They typically include renaming columns, changing data types, and filtering out unnecessary rows.

These models give us a clean and consistent view of the data that we can use in our analysis. They make the downstream fact and dimension models easier to build and maintain.

For each raw source table, we will create a corresponding staging model which we will:

- SELECT only the columns we need
- Rename columns to follow a consistent naming convention i.e. snake_case
- Optionally, change data types to ensure they are appropriate for analysis or derive new columns as needed.

Create a new directory called `staging` inside the `models` directory:

`models/staging/stg_cycle_hire.sql`

```sql
{{ config(materialized='view') }}

SELECT
  rental_id,
  bike_id,
  bike_model,
  duration,
  duration_ms,
  start_date,
  end_date,
  start_station_id,
  start_station_name,
  end_station_id,
  end_station_name
FROM {{ source('london_bicycles', 'cycle_hire') }}
```

`models/staging/stg_cycle_stations.sql`

```sql
{{ config(materialized='view') }}

SELECT
  id AS station_id, -- to avoid confusion and be explicit
  name AS station_name,
  latitude,
  longitude,
  bikes_count,
  docks_count,
  nbEmptyDocks,
  install_date,
  removal_date,
  installed,
  locked,
  temporary
FROM {{ source('london_bicycles', 'cycle_stations') }}
```

### Fact and Dimension Models

Next, we will create fact and dimension models. Fact models are used to store the main data that we want to analyze, while dimension models are used to store the attributes that we want to use to filter or group the data.

We will store this inside the mart layer, which is typically used for the final models that are ready for analysis. For larger projects, we would typically have an intermediate layer but for simplicity, we will skip that here.

How do we decide what to put in the fact and dimension models?

- **Fact Models**: These should contain the main quantitative data that we want to analyze. In our case, the `stg_cycle_hire` model is a good candidate for a fact model because it contains information about bike rentals, including the duration and bike ID i.e. each row is a ride event This is a transactional event log, so we model it as a fact table.
- **Dimension Models**: These should contain the attributes that we want to use to filter or group the data. In our case, the `stg_cycle_stations` model is a good candidate for a dimension model because it contains information about the bike stations, including their location and number of bikes and docks. It provides context to the ride events in the fact table, so we model it as a dimension table.

`models/marts/dim_station.sql`

```sql
{{ config(materialized='table') }}

SELECT DISTINCT
  station_id,
  station_name,
  latitude,
  longitude,
  bikes_count,
  docks_count,
  nbEmptyDocks,
  install_date,
  removal_date,
  installed,
  locked,
  temporary
FROM {{ ref('stg_cycle_stations') }}
```

`models/marts/fact_cycle_hire.sql`

```sql
{{ config(materialized='table') }}

SELECT
  s.rental_id,
  s.bike_id,
  s.bike_model,
  s.duration,
  s.duration_ms,
  s.start_date,
  s.end_date,
  s.start_station_id,
  start_station.station_name AS start_station_name,
  s.end_station_id,
  end_station.station_name AS end_station_name
FROM {{ ref('stg_cycle_hire') }} s
LEFT JOIN {{ ref('dim_station') }} start_station
  ON s.start_station_id = start_station.station_id
LEFT JOIN {{ ref('dim_station') }} end_station
  ON s.end_station_id = end_station.station_id
```

### Snapshot Models

Snapshot models are used to capture the state of a table at a specific point in time. This is useful for tracking changes over time, such as the number of bikes at a station or the status of a station.

Our dimension model `dim_station` is a good candidate for a snapshot model because the number of bikes and docks at a station can change over time. We can create a snapshot model to capture the state of the station at regular intervals.

Create a new directory called `snapshots` inside the `models` directory.

```sql
{% snapshot snap_station %}

{{
  config(
    target_schema='snapshots',
    unique_key='station_id',
    strategy='check',
    check_cols=['station_name', 'latitude', 'longitude', 'installed', 'locked', 'temporary']
  )
}}

SELECT *
FROM {{ source('london_bicycles', 'cycle_stations') }}

{% endsnapshot %}
```
