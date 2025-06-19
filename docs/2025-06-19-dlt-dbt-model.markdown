---
layout: post
title: "Podcasts, Pipelines & Chill 🎧"
# date: 2025-06-19
nav_order: 1
# permalink: /posts/podcasts-pipelines-chill/
---

# Podcasts, Pipelines & Chill 🎧🛠️

I was scrolling Reddit on `r/dataengineering` on a lazy Saturday and found a post about podcast analytics. It was really hot that day 🔥

so I thought, why not wire up a **tiny** data platform to see the numbers for myself?

---

## Stack TL;DR

| Layer         | Tool               | Why I picked it                                          |
| ------------- | ------------------ | -------------------------------------------------------- |
| Orchestration | **Apache Airflow** | The de‑facto “cron‑on‑steroids”.                         |
| Ingest        | **dlt**            | Point → click → data in Snowflake with almost no config. |
| Transform     | **dbt**            | SQL + version control + tests = ❤️                       |
| Warehouse     | **Snowflake**      | Handles scale so I don’t have to.                        |
| Glue          | **Docker Compose** | One `up` and everything runs.                            |
| Meta DB       | **Postgres**       | Airflow needs somewhere to write.                        |

---

## High‑Level Diagram

![Overall architecture](/assets/images/global_assignment_architecure.jpg)

---

## Warehouse Layers

![Model lineage](/assets/images/global_data_warehouse_layers.png)

- **Bronze** → Raw dumps from `dlt`
- **Silver** → Clean tables (`int__*`)
- **Gold** → Facts & marts (`fct__*`, `mart__*`)

---

## The DAG

1. **Load** – `dlt` pulls the podcast API and lands rows in Snowflake.
2. **Transform** – `dbt run` cleans everything up.
3. **Test** – `dbt test` makes sure I didn’t break anything.

All wrapped in a single file: `podcast_dag.py`.

---

## Running It Yourself

```bash
git clone <this‑repo>
cd <this‑repo>
docker-compose up --build      # grab a coffee ☕
```

Open Airflow at `http://localhost:8080`, enable **podcast_dag**, hit _Trigger_, watch the logs fly.

---

## Quick Peek at a Gold‑Layer Query

```sql
SELECT
  ep.title,
  COUNT(*) AS completed_plays
FROM {{ ref('fct__completed') }}
JOIN {{ ref('int__episodes') }} ep USING (episode_id)
GROUP BY 1
ORDER BY completed_plays DESC
LIMIT 10;
```

Boom – top‑played episodes in one query.

---

## What I Liked

- 🏎️ **Speed** – `dlt` + Snowflake moved ~50 k rows in seconds.
- 🔍 **Transparency** – dbt lineage graph shows exactly where data flows.
- 🎛️ **Control** – Airflow UI lets me replay any task with a click.

## What Could Be Better

- The local Airflow image takes a minute to build.
- Secrets in `docker-compose.yaml` are fine for a demo, but use a vault in prod.

---

That’s all – a weekend well spent and a nice little pipeline to show for it.  
Questions? Suggestions? Hit me up or find out more about me on my [about page](/about).
