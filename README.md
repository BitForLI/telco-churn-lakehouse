# Telco Lakehouse Analytics

A Databricks-ready analytics product that turns a telecommunications customer snapshot into
quality-controlled tables, reusable retention metrics, and an evidence-based stakeholder brief.

The project focuses on a practical business question: **which customer segments should a
retention team investigate first, and how can that analysis run repeatedly with minimal manual
work?**

## Product at a glance

| | |
| --- | --- |
| **Users** | Retention analysts and business stakeholders |
| **Input** | Repeated customer snapshots containing account, service, contract, and churn data |
| **Output** | Governed tables, dashboard-ready KPIs, segment priorities, and a stakeholder brief |
| **Trust controls** | Invalid-row quarantine, duplicate checks, quality gates, and traceable run manifests |
| **Runs in** | Databricks with Delta Lake, or locally through the tested Python runner |

## What it delivers

- A Bronze/Silver/Gold lakehouse workflow implemented as four Databricks notebook tasks.
- Data-contract checks, invalid-row quarantine, and a configurable quality gate.
- Hash-keyed Bronze inserts and customer/date-keyed Silver Delta `MERGE` operations for repeated snapshot loading.
- Overall and segment-level churn, revenue, tenure, and revenue-at-risk measures.
- SQL queries for KPI cards, retention priorities, snapshot trends, and quality monitoring.
- A deterministic stakeholder report whose numbers come directly from Gold metrics.
- An optional Databricks `ai_query` summary that receives aggregated evidence only.
- A dependency-light local Python runner, automated tests, linting, and GitHub Actions CI.

## Architecture

```text
CSV in a Unity Catalog Volume
              |
              v
 Bronze Delta table  -- raw values, source path, record hash
              |
              v
 Silver customer model  ---->  Quarantine table
 typed fields and features       explicit quality issues
              |
              v
 Gold KPIs + segment metrics + retention priority view
              |
        +-----+------+
        |            |
        v            v
   AI/BI SQL    Stakeholder brief
                deterministic or optional aggregate-only AI
```

See [the architecture notes](docs/architecture.md) for design and reliability decisions and
[the data dictionary](docs/data_dictionary.md) for table definitions.

## Results on the included sample

The complete IBM sample contains 7,043 customer records. A validated local run produced:

| Measure | Result |
|---|---:|
| Quality pass rate | 100.0% |
| Overall churn rate | 26.5% |
| Monthly revenue represented | $456,116.60 |
| Monthly revenue attached to churned customers | $139,130.85 |
| Month-to-month churn rate | 42.7% |
| One- and two-year contract churn rate | 6.8% |

The generated brief identifies the month-to-month group as the largest revenue-at-risk segment
and recommends testing an early-tenure retention intervention. These are descriptive
associations from a static educational dataset, not causal findings or claims about commercial
impact. See the [sample stakeholder brief](reports/sample_stakeholder_brief.md).

## Run locally

Python 3.11 or later is required. The local workflow uses only the standard library; development
tools are installed separately.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"
telco-analytics --input data/raw/telco_customer_churn.csv --output data/output
```

Generated files include the curated customer snapshot, quarantined rows, Gold metrics, KPI JSON,
stakeholder brief, and a run manifest containing row counts, the quality result, and the input
SHA-256 fingerprint.

```powershell
ruff check src tests scripts notebooks
pytest
python scripts/check_outputs.py data/output/run_manifest.json
```

## Deploy to Databricks

The deployment requires a Databricks workspace with Unity Catalog access and permission to create
tables in the chosen catalog and schema.

1. Create a Unity Catalog Volume and upload the CSV to its landing directory.
2. Authenticate the Databricks CLI for the target workspace.
3. Validate and deploy the bundle, overriding values where necessary:

```powershell
databricks bundle validate
databricks bundle deploy -t dev
databricks bundle run telco_customer_analytics -t dev
```

The job definition starts with its weekly schedule paused. Set `catalog`, `schema`, `source_path`, and
`minimum_quality_rate` in bundle variables before enabling it. Supplying `ai_endpoint` enables
the optional aggregate-only AI summary; leaving it empty keeps reporting deterministic.

## Data quality and governance

- The source contract requires 21 columns and validates numeric fields and selected categories,
  including churn, contract, internet service, and four Yes/No flags; it does not validate every
  categorical column.
- Invalid numeric values, missing IDs, duplicates, and unsupported values in checked categories
  are quarantined. A blank total charge is accepted only for a zero-tenure customer.
- In Databricks, a failed Silver quality gate prevents dependent Gold and report tasks from
  running, but Silver and quarantine writes have already occurred. The local runner writes its
  outputs and manifest before raising on a failed gate; inspect `quality_gate_passed` before
  consuming those files.
- Customer identifiers never enter the optional AI prompt.
- Numeric claims in the default report are generated from current Gold-layer results.
- The run manifest makes each local execution traceable to an exact input file.

## Implementation details

The local runner and Spark notebooks cover the same business workflow, but are not identical
implementations. Locally, the first valid record for a customer ID is retained and later
duplicates are quarantined. Bronze uses a hash of the snapshot date and source fields to skip
previously inserted records; Spark Silver quarantines all remaining records with a duplicate
customer ID in that snapshot. Silver merges on customer ID and date, while Gold overwrites the
selected date partition. These are insert/update workflows, not source-deletion reconciliation.

Gold groups customers by contract, internet service, tenure band, payment group, and support
status. `monthly_revenue_at_risk` means the sum of monthly charges for records already labelled
as churned, not a forecast of future losses. Segments overlap across dimensions, so their values
must not be added together as a single portfolio total.

The default brief is built directly from calculated metrics. Optional AI reporting sends only
aggregate evidence and asks the model to avoid unsupported numbers and causal claims; the code
does not automatically verify the generated text. The local brief selects segments with at
least the larger of 10 customers or 1% of the snapshot (falling back when none qualify); the
Databricks priority view uses a fixed 10-customer minimum.

## Code references

- [Source checks](src/telco_analytics/quality.py) and
  [customer features](src/telco_analytics/transformations.py).
- [Bronze ingestion](notebooks/01_bronze_ingestion.py),
  [Silver modelling](notebooks/02_silver_customer_model.py), and
  [Gold metrics](notebooks/03_gold_retention_metrics.py).
- [Local KPIs](src/telco_analytics/metrics.py),
  [brief generation](src/telco_analytics/insights.py), and
  [output manifest](src/telco_analytics/pipeline.py).
- [Task dependencies and retries](resources/telco_analytics.job.yml),
  [tests](tests/), and [CI workflow](.github/workflows/ci.yml).

## Repository guide

| Path | Purpose |
|---|---|
| `notebooks/` | Databricks Bronze, Silver, Gold, and reporting tasks |
| `src/telco_analytics/` | Tested local implementation and reusable business logic |
| `sql/dashboard_queries.sql` | Databricks AI/BI dashboard queries |
| `resources/` and `databricks.yml` | Declarative Automation Bundle job definition |
| `tests/` | Quality, transformation, metric, report, and end-to-end tests |
| `reports/` | Example business-facing output from a validated local run |
| `docs/` | Architecture, data dictionary, and role-capability mapping |

## Attribution and scope

This project was forked from
[`tolgahancepel/telco-customer-churn`](https://github.com/tolgahancepel/telco-customer-churn)
under the MIT License and uses the public
[`Telco-Customer-Churn.csv`](https://github.com/IBM/telco-customer-churn-on-icp4d/blob/master/data/Telco-Customer-Churn.csv)
sample. See [NOTICE.md](NOTICE.md) for details.

The lakehouse workflow, quality controls, reusable analytics package, automated reporting, tests,
CI, SQL, and documentation in this fork were added by Reese Lee. The sample is not current data
from a telecommunications provider, and a live Databricks deployment requires the operator's own
workspace and permissions.

