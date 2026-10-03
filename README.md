# 🎬 MovieLens Data Engineering Project

A data engineering project that builds a cloud-based analytics pipeline using **AWS S3, Snowflake, and dbt** to transform raw MovieLens datasets into structured analytical models.

The project demonstrates an end-to-end modern data engineering workflow: ingesting raw data into cloud storage, loading it into a data warehouse, transforming it with dbt, applying data quality tests, and organizing the resulting data into analytics-ready fact, dimension, and mart models.

---

## 🏗️ Architecture

```text
                    MovieLens Dataset
                           │
                           ▼
                    ┌─────────────┐
                    │   AWS S3    │
                    │ Raw Storage │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Snowflake  │
                    │  RAW Layer  │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ dbt Staging │
                    │    Layer    │
                    └──────┬──────┘
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
        ┌──────────────┐      ┌──────────────┐
        │ Dimension    │      │ Fact Models  │
        │    Models    │      │              │
        └──────┬───────┘      └──────┬───────┘
               │                     │
               └──────────┬──────────┘
                          ▼
                   ┌─────────────┐
                   │  Mart Layer │
                   │  Analytics  │
                   └─────────────┘
```

---

## 🎯 Project Goals

The main goals of this project are to:

* Build a cloud-based data pipeline using AWS and Snowflake.
* Store raw datasets in Amazon S3.
* Load raw data into Snowflake.
* Use dbt for SQL-based transformations.
* Separate raw, staging, dimensional, fact, and mart layers.
* Implement data quality testing.
* Create reusable and maintainable transformation models.
* Use dbt's dependency management and lineage through `ref()`.
* Prepare transformed data for downstream analytics and dashboards.

---

## 🛠️ Technologies Used

| Technology     | Purpose                                                  |
| -------------- | -------------------------------------------------------- |
| **AWS S3**     | Cloud object storage for raw MovieLens datasets          |
| **Snowflake**  | Cloud data warehouse                                     |
| **dbt**        | Data transformation, testing, documentation and modeling |
| **SQL**        | Data transformation and analysis                         |
| **Python**     | Environment and supporting data engineering work         |
| **Git/GitHub** | Version control and project management                   |

---

## 📂 Dataset

The project uses the **MovieLens** dataset.

The raw files include:

```text
README.txt
movies.csv
ratings.csv
tags.csv
links.csv
genome-tags.csv
genome-scores.csv
```

These files are stored in an AWS S3 bucket before being loaded into Snowflake.

### Main datasets

#### `movies`

Contains movie metadata such as:

* Movie ID
* Movie title
* Genres

#### `ratings`

Contains user ratings:

* User ID
* Movie ID
* Rating
* Timestamp

#### `tags`

Contains user-generated movie tags:

* User ID
* Movie ID
* Tag
* Timestamp

#### `genome-tags`

Contains tag identifiers and their corresponding tag names.

#### `genome-scores`

Contains the relevance of each tag to each movie.

---

# 🔄 Data Pipeline

## 1. Raw Data → AWS S3

The original MovieLens CSV files are stored in an Amazon S3 bucket.

```text
s3://learning-project-netflix-movies/
```

S3 acts as the initial cloud storage layer for the raw data.

---

## 2. AWS S3 → Snowflake

Snowflake accesses the S3 data using an external stage and storage integration.

The Snowflake environment contains a `RAW` layer where the source data is loaded.

```text
AWS S3
   │
   ▼
Snowflake External Stage
   │
   ▼
RAW Tables
```

This keeps the raw data separate from transformed analytical models.

---

# 🧱 dbt Transformation Layers

The dbt project follows a layered modeling approach.

```text
RAW
 │
 ▼
STAGING
 │
 ▼
DIMENSIONS + FACTS
 │
 ▼
MARTS
```

## Staging Layer

The staging layer creates clean interfaces to the raw data.

Examples:

```text
src_movies
src_ratings
src_tags
src_genome_tags
src_genome_scores
src_links
```

Typical staging transformations include:

* Renaming columns
* Standardizing column names
* Selecting required fields
* Basic type conversions
* Creating consistent interfaces for downstream models

For example:

```sql
SELECT
    movieId AS movie_id,
    title,
    genres
FROM raw_movies
```

---

# 📊 Dimension Models

Dimension models contain descriptive entities used by analytical queries.

### `dim_movies`

Contains cleaned movie metadata.

Example transformations include:

* Standardizing movie titles
* Trimming whitespace
* Converting genres into an array
* Providing a consistent `movie_id`

```text
dim_movies
├── movie_id
├── movie_title
├── genre_array
└── genres
```

### `dim_users`

Contains unique users derived from the available user activity.

```text
dim_users
└── user_id
```

### `dim_genome_tags`

Contains cleaned genome tag information.

```text
dim_genome_tags
├── tag_id
└── tag_name
```

---

# 📈 Fact Models

Fact models contain measurable events or relationships.

## `fct_ratings`

Stores movie-rating events.

```text
fct_ratings
├── user_id
├── movie_id
├── rating
└── rating_timestamp
```

This model can be used to analyze:

* Movie ratings
* User activity
* Rating distributions
* Movie popularity
* User behavior

---

## `fct_genome_scores`

Stores the relationship between movies and genome tags.

```text
fct_genome_scores
├── movie_id
├── tag_id
└── relevance_score
```

The model filters out zero-relevance records and rounds relevance scores for analytical use.

Example:

```sql
SELECT
    movie_id,
    tag_id,
    ROUND(relevance, 4) AS relevance_score
FROM src_scores
WHERE relevance > 0
```

---

# 📊 Mart Layer

The mart layer contains models designed specifically for downstream analytics.

For example:

```text
mart_movie_release
```

Mart models combine information from dimensions, facts, and reference data into more convenient analytical datasets.

The purpose of this layer is to make the warehouse easier for analysts and BI tools to consume without requiring them to understand the underlying raw and transformation layers.

---

# 🧪 Data Quality Testing

dbt tests are used to validate the transformed data.

Examples include:

### Not-null tests

Important identifiers are tested to ensure they contain values.

```yaml
tests:
  - not_null
```

### Relationship tests

Foreign keys are checked against their corresponding dimension tables.

For example:

```text
fct_ratings.movie_id
          │
          ▼
dim_movies.movie_id
```

This helps detect orphaned records.

### Custom SQL tests

The project also demonstrates singular dbt tests.

A dbt test should return **zero rows when the data is valid**.

For example:

```sql
SELECT
    movie_id,
    tag_id,
    relevance_score
FROM {{ ref('fct_genome_scores') }}
WHERE relevance_score <= 0
```

If the query returns rows, the test fails.

---

# 🗂️ Project Structure

```text
netflix/
│
├── analyses/
│
├── macros/
│
├── models/
│   │
│   ├── staging/
│   │   ├── src_movies.sql
│   │   ├── src_ratings.sql
│   │   ├── src_tags.sql
│   │   ├── src_genome_tags.sql
│   │   ├── src_genome_scores.sql
│   │   └── src_links.sql
│   │
│   ├── dim/
│   │   ├── dim_movies.sql
│   │   ├── dim_users.sql
│   │   └── dim_genome_tags.sql
│   │
│   ├── fct/
│   │   ├── fct_ratings.sql
│   │   └── fct_genome_scores.sql
│   │
│   ├── mart/
│   │   └── mart_movie_release.sql
│   │
│   └── sources.yml
│
├── seeds/
│
├── snapshots/
│
├── tests/
│   └── relevance_score_test.sql
│
├── dbt_project.yml
├── packages.yml
└── README.md
```

---

# 🔗 dbt Model Dependencies

dbt manages dependencies between models using `ref()`.

For example:

```sql
SELECT *
FROM {{ ref('src_movies') }}
```

This allows dbt to understand that:

```text
src_movies
    │
    ▼
dim_movies
```

Similarly:

```text
src_ratings
    │
    ▼
fct_ratings
    │
    ├──────► dim_users
    │
    └──────► dim_movies
```

This creates a dependency graph that dbt can use to determine model execution order and lineage.

---

# ⚙️ Setting Up the Project

## Prerequisites

Install:

* Python
* dbt
* Snowflake account
* AWS account
* Git

A Snowflake adapter is required:

```bash
pip install dbt-snowflake
```

---

## Clone the Repository

```bash
git clone <your-repository-url>
cd netflix
```

---

## Create a Virtual Environment

Windows:

```cmd
python -m venv venv
```

Activate it:

```cmd
venv\Scripts\activate
```

---

## Install Dependencies

```bash
pip install -r requirements.txt
```

If using dbt packages:

```bash
dbt deps
```

---

# 🔐 Snowflake Configuration

Configure your Snowflake connection in:

```text
~/.dbt/profiles.yml
```

Example:

```yaml
netflixdbt:
  target: dev

  outputs:
    dev:
      type: snowflake
      account: <your-account>
      user: <your-user>
      password: <your-password>

      role: <your-role>
      database: MOVIELENS
      warehouse: <your-warehouse>
      schema: DEV

      threads: 4
```

**Never commit passwords, private keys, access tokens, or other credentials to GitHub.**

---

# ▶️ Running the Project

Verify the dbt connection:

```bash
dbt debug
```

Install dbt packages:

```bash
dbt deps
```

Load seeds:

```bash
dbt seed
```

Run models:

```bash
dbt run
```

Run tests:

```bash
dbt test
```

Run everything:

```bash
dbt build
```

Generate documentation:

```bash
dbt docs generate
```

Start the documentation server:

```bash
dbt docs serve
```

---

# 📋 Example Analytical Questions

Once the models are built, the warehouse can be used to answer questions such as:

* Which movies have the highest average ratings?
* Which movies have received the most ratings?
* Which genres are most common?
* Which users are the most active?
* Which tags have the highest relevance to specific movies?
* How does movie popularity vary by release period?
* What relationships exist between genres, ratings, and tags?

These analytical queries can later be connected to a BI/dashboarding tool.

---

# 🚀 Future Improvements

The project can be extended with additional production-style data engineering capabilities:

* Incremental dbt models
* More analytical mart models
* dbt documentation and lineage
* Additional data quality tests
* Snowflake task-based automation
* Pipeline orchestration
* CI/CD with GitHub Actions
* Data freshness monitoring
* Dashboard integration
* Automated S3 → Snowflake ingestion
* Monitoring and alerting

---

# 📚 What This Project Demonstrates

This project demonstrates practical experience with:

* Cloud data storage
* Cloud data warehousing
* AWS S3
* Snowflake
* External stages
* Data warehouse layering
* SQL transformations
* dbt
* dbt `ref()` and `source()`
* Dimensional modeling
* Fact and dimension tables
* Data quality testing
* Custom dbt tests
* Dependency management
* Analytical data marts
* Git-based project development

---

## 👨‍💻 Author

**Syed Hamza Ali**

Computer Science Engineering Student

GitHub: [AgentHamza007](https://github.com/AgentHamza007)
