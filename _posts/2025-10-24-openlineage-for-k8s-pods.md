---
title: OpenLineage + Airflow on Kubernetes: Getting Real Lineage into Atlan/Marquez
date: 2025-10-24
categories: [Airflow, OpenLineage, Kubernetes, Atlan, Marquez, Data Engineering]
description: “A practical guide to wiring OpenLineage for Airflow DAGs that use the KubernetesPodOperator, with manual inlets/outlets and emitting lineage from inside the pod. Works with Atlan or Marquez.”
tags: [airflow, openlineage, kubernetes, kpo, atlan, marquez, lineage]
---

# OpenLineage + Airflow on Kubernetes: Getting e2e Lineage into Atlan/Marquez

If you’re running Airflow on Kubernetes and want your runs to show up in Atlan or Marquez with proper dataset lineage, you’ll eventually hit a snag: the KubernetesPodOperator (KPO) is a black box. It launches a container and Airflow has no idea what inputs/outputs you touched. That means OpenLineage can’t emit the right dataset edges.

This post shows two reliable ways to fix that:

    1. Manual lineage in the DAG using inlets / outlets (fastest, table-level).
    2. Emit lineage from inside the container (richer; link runs with a ParentRun facet).

> Assumptions: Airflow 2.9.x with the OpenLineage provider 1.9. If you’re on Airflow ≥2.10 + provider 2.x, everything here still applies, but the provider has some newer niceties you can adopt later.

## Step 0 — Point Airflow’s OpenLineage at Atlan/Marquez (or the console)

You can configure the provider via env vars or a small YAML. For local testing, start with console so you can see JSON events in logs:

```bash
# print OpenLineage events to Airflow logs
export AIRFLOW__OPENLINEAGE__TRANSPORT='{"type":"console"}'
export AIRFLOW__OPENLINEAGE__NAMESPACE='airflow-dev'
```

## Option 1 — Manual Lineage with Inlets/Outlets

This is the quickest way to make Atlan/Marquez show your KPO task’s inputs/outputs. You annotate the task with what it reads/writes. Here’s an example:

```python
from airflow import DAG
from datetime import datetime
from airflow.lineage.entities import Table
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator

with DAG(
    dag_id="kpo_openlineage_demo",
    start_date=datetime(2025, 1, 1),
    schedule=None,
    catchup=False,
    tags=["lineage", "kubernetes"],
) as dag:

    k8s_task = KubernetesPodOperator(
        task_id="ingest_accounts",
        name="ingest_accounts",
        namespace="airflow-tasks",
        image="localhost:5000/data-ingest:dev",
        cmds=["python", "-m", "app.main"],
        get_logs=True,
        is_delete_operator_pod=True,

        # 👇 manual lineage — table-level
        inlets=[Table("source.analytics.accounts_raw")],
        outlets=[Table("bq.analytics.accounts_raw")],
    )
```

Pros:

- Dead simple; just add inlets/outlets to your KPO.
- Immediately visible as an edge: source.analytics.accounts_raw → bq.analytics.accounts_raw.

Cons:

- You need to know the table names ahead of time.
- No way to link the KPO run with what happened inside the pod.

## Option 2 — Emit lineage inside the container 

This method gives you the richest lineage. You emit OpenLineage events from inside your containerized task using the OpenLineage Python client. This way, you can capture exactly what datasets were read/written, and link the KPO run as a parent.

```python
from openlineage.client import OpenLineageClient, RunEvent, RunState, Dataset
from openlineage.client.facet import ParentRunFacet
import os
import time

def emit_start_event(client, run_id, job_name):
    event = RunEvent(
        eventType=RunState.START,
        eventTime=time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        run={"runId": run_id},
        job={"namespace": "airflow-dev", "name": job_name},
        inputs=[Dataset(namespace="source.analytics", name="accounts_raw")],
        outputs=[Dataset(namespace="bq.analytics", name="accounts_raw")],
        facets={
            "parentRun": ParentRunFacet(
                runId=os.getenv("OPENLINEAGE_PARENT_RUN_ID"),
                jobName=os.getenv("OPENLINEAGE_PARENT_JOB_NAME"),
            )
        },
    )
    client.emit(event)

def emit_complete_event(client, run_id, job_name):
    event = RunEvent(
        eventType=RunState.COMPLETE,
        eventTime=time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        run={"runId": run_id},
        job={"namespace": "airflow-dev", "name": job_name},
    )
    client.emit(event)

def main():
    client = OpenLineageClient()
    run_id = os.getenv("OPENLINEAGE_RUN_ID")
    job_name = os.getenv("OPENLINEAGE_JOB_NAME")
    emit_start_event(client, run_id, job_name)
    # ... your data processing logic here ...
    emit_complete_event(client, run_id, job_name)
if __name__ == "__main__":
    main()

```

## When to pick which approach?

- Pick Manual inlets/outlets when you need something working today and table-level edges are enough.
- Pick Emit-inside-the-pod (+ ParentRun) when you need richer metadata (schemas, stats, potentially column-level) and you control the job’s image/entrypoint.

Many teams start with manual, validate the experience in Atlan/Marquez, then upgrade selective tasks to emit-inside where the extra detail is worth it.

## Closing thoughts

Airflow on Kubernetes isn’t a barrier to lineage — it just means you decide where lineage comes from:

- from your DAG (declared), or
- from your code (emitted).

Atlan/Marquez will happily ingest either via OpenLineage. Start simple with inlets/outlets, then layer in ParentRun-linked events from inside the pod where you need deeper visibility.