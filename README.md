# Data Lakehouse

## Tools

* Docker + Docker-Compose

## Setup

* `docker-compose up --build`

### First Time Tasks

In order to bootstrap superset you need to migrate the DB and create a initial admin user:

```
docker-compose exec superset superset db upgrade
docker-compose exec superset superset fab create-admin \
    --username admin \
    --firstname Admin \
    --lastname User \
    --email admin@example.com \
    --password admin
docker-compose exec superset superset init
```

### Minio

Before you begin goto http://localhost:9001 using `minioadmin / minioadmin` as login and create a Bucket `iceberg-warehouse` and an Access Keys. Store the secret for later use.

Assign the policy to allow full access to minio:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::*"
            ]
        }
    ]
}
```

### Dremio

Login via http://localhost:9047 and begin creating the inital setup for an admin user. `dremioadmin / dremioadmin1` is what I used to pass Dremio password requirement.

To be able to interact with Apache Iceberg, we use Nessie.
Goto Data Sources to add a Nessie Catalog.

#### Nessie

Quote: "Nessie is a native Apache Iceberg catalog that provides Git-like data management. As a result, data engineering teams can use commits, branches, and tags to be able to experiment on Apache Iceberg tables."

https://docs.dremio.com/24.3.x/reference/sql/commands/nessie/


##### General

| Key                     | Value                                |
|-------------------------|--------------------------------------|
| Name                    | iceberg                              |
| Nessie endpoint URL     | [http://nessie:19120/api/v2](http://nessie:19120/api/v2) |
| Nessie authentication type | None                             |

##### Storage

| Key                        | Value                              |
|----------------------------|------------------------------------|
| AWS root path              | /iceberg-warehouse                 |
| AWS access key | minioadmin |
| AWS access secret | minioadmin |

###### Connection properties

| Key                        | Value                              |
|----------------------------|------------------------------------|
| fs.s3a.path.style.access   | true                               |
| fs.s3a.endpoint            | minio:9000                         |
| dremio.s3.compat           | true                               |
| Encrypt connection         | disabled                           |

Ensure you use the exact values as shown!

#### Data Example

* `git lfs fetch` to fetch data files
* `bunzip2` the datafiles

Upload the file `./data/TMDB_movie_dataset_v11.csv` into Dremio via UI (475MB)
Enable `Extract Column Names` to get the column names from the first row.

Upload `./data/country_codes.csv` to have a list of alpha-2 codes available to join.

Check if your local data is queryable:
```
SELECT * FROM "@dremioadmin"."TMDB_movie_dataset_v11"
```

Create a feature branch:
```
CREATE BRANCH TMDB_movie_dataset_v11 IN iceberg;
```

Load local CSV into Iceberg

```
USE BRANCH TMDB_movie_dataset_v11 in iceberg;
CREATE TABLE iceberg.kaggle.TMDB_movie_dataset_v11 AS ( SELECT * FROM "@dremioadmin"."TMDB_movie_dataset_v11" );
```

Datatype corrected Query
```
SELECT
  CONVERT_TO_INTEGER(id, 1, 1, 0) AS id,
  title,
  CONVERT_TO_FLOAT(vote_average, 1, 1, 0) AS vote_average,
  CONVERT_TO_INTEGER(vote_count, 1, 1, 0) AS vote_count,
  status,
  TO_DATE(release_date, 'YYYY-MM-DD', 1) AS release_date,
  CONVERT_TO_INTEGER(revenue, 1, 1, 0) AS revenue,
  CONVERT_TO_INTEGER(runtime, 1, 1, 0) AS runtime,
  cast("adult" as boolean) AS adult,
  backdrop_path,
  CONVERT_TO_INTEGER(budget, 1, 1, 0) AS budget,
  homepage,
  imdb_id,
  original_language,
  original_title,
  overview,
  CONVERT_TO_FLOAT(popularity, 1, 1, 0) AS popularity,
  poster_path,
  tagline,
  regexp_split(genres, ',\s?', 'ALL', 25) AS genres,
  regexp_split(production_companies, ',\s?', 'ALL', 25) AS production_companies,
  regexp_split(production_countries, ',\s?', 'ALL', 10) AS production_countries,
  regexp_split(spoken_languages, ',\s?', 'ALL', 25) AS spoken_languages,
  regexp_split(keywords, ',\s?', 'ALL', 100) AS keywords
FROM
  iceberg.kaggle."TMDB_movie_dataset_v11" AT BRANCH "TMDB_movie_dataset_v11";
```

! There appears to be a problem in splitting strings. Last element is cut off. Odd!

Load to another iceberg tables
```
CREATE TABLE iceberg.kaggle.TMDB_movie_dataset_v11_clean AS (
...copy-above-conversion-query...
);
```

Merge change back to main
```
MERGE BRANCH TMDB_movie_dataset_v11 INTO main IN iceberg
```

### Superset

(See Setup to ensure DB was migrated and an admin user was created)

Goto http://localhost:8088/
In the upper right. Click on `+` >> `Data` and `Connect Database`
On "Supported Databases" select "Dremio"
Use the Connection String: `dremio+flight://dremioadmin:dremioadmin1@dremio:32010/?UseEncryption=false`

### Jupyter + PySpark

TBA
