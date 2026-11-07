# Petroleum import data pipeline

Reproducible collection and validation of monthly refined petroleum import data for Kyrgyzstan and other landlocked economies. This stage covers data only: there are no forecasting models here.

Start with [reports/phase2/research_feasibility_report.md](reports/phase2/research_feasibility_report.md). It supersedes the Phase 1 [data feasibility assessment](reports/data_feasibility_assessment.md), which is kept as a record.

For modelling, use `data/research/`, described in [docs/data_dictionary.md](docs/data_dictionary.md). Method and rules are in [docs/methodology.md](docs/methodology.md), [docs/missing_data_policy.md](docs/missing_data_policy.md) and [docs/data_sources.md](docs/data_sources.md).

## Setup

```bash
python -m venv .venv
.venv/Scripts/python -m pip install -r requirements.txt
```

Copy `.env.example` to `.env` and add your keys from <https://comtradedeveloper.un.org> (product `comtrade - v1`, free tier). Without a key the pipeline falls back to the keyless preview endpoint, which is limited to 500 records a call and throttled. The panel needs about 324 calls with a key, or about 3,780 without one.

## Running

Each stage resumes from its checkpoint, so it is safe to rerun.

```bash
.venv/Scripts/python run.py refs
```
```bash
.venv/Scripts/python run.py availability
```
```bash
.venv/Scripts/python run.py pilot
```
```bash
.venv/Scripts/python run.py audit
```
```bash
.venv/Scripts/python run.py external
```
```bash
.venv/Scripts/python run.py process
```
```bash
.venv/Scripts/python run.py panel
```
```bash
.venv/Scripts/python run.py process
```
```bash
.venv/Scripts/python run.py reports
```

Phase 2 follows, and writes only to `data/research/`, `docs/` and `reports/phase2/`:

```bash
.venv/Scripts/python run.py validation_pull
```
```bash
.venv/Scripts/python run.py phase2
```

`process` runs before `panel` because the panel reporter list comes from the coverage report. `run.py load` needs `DATABASE_URL` and a database with `sql/schema.sql` applied.

To take a new data vintage, change `snapshot_id` in `config/settings.yaml`. Older snapshots stay untouched.

## Tests

```bash
.venv/Scripts/python -m pytest -q tests
```
```bash
.venv/Scripts/python -m tests.db_smoke
```

`db_smoke` applies the schema and loads all data into a throwaway embedded PostgreSQL.

## Layout

| Path | Contents |
|---|---|
| `config/` | Settings, HS code dictionary, candidate countries |
| `src/ingestion/` | Comtrade client, task plans, external sources |
| `src/preprocessing/` | Cleaning rules, Datasets A-C, validation reports |
| `src/database/`, `sql/` | PostgreSQL schema and loader |
| `data/raw/` | Unmodified responses and a JSONL manifest per snapshot |
| `data/processed/` | Tidy source tables and CSV copies |
| `data/analytical/` | Parquet datasets |
| `reports/` | Quality report, inventory, coverage, feasibility |

Notebooks from the original plan were replaced by `run.py` stages and generated reports, which rerun without manual steps.

## Outputs

| File in `data/analytical/` | Grain |
|---|---|
| `dataset_a_bilateral` | Reporter x HS6 x partner x flow x month |
| `dataset_b_monthly` | Reporter x product group x flow x month |
| `supplier_concentration` | Reporter x product group x month, imports only |
| `dataset_c_global_monthly` | Global x month |
| `dataset_c_country_monthly` | Country x month |
| `dataset_c_country_annual` | Country x year; annual values, never interpolated |
| `dataset_d_integrated` | Eligible series x month, from 2015, joined to supplier and external data |

Dataset D holds same-month values on one calendar. Lag every feature to its release date before modelling, or the results will leak future information. Annual covariates are left out for the same reason.

`reports/series_eligibility.csv` states which series may be used. `eligible` is the full panel; `core` also drops transit or export markets and 271019 series that miss blended diesel.

## Key `dataset_b_monthly` columns

| Column | Meaning |
|---|---|
| `mass_t` | Clean mass in tonnes. Use this one |
| `mass_t_reported` | Before the unit-value check |
| `mass_t_incl_volume_estimate` | Adds rejected rows at litres times observed density. For annual comparison only |
| `total_source` | `partner_sum`, or `world_row` where partner rows are not available |
| `month_status` | `usable`, `unit_value_rejected`, `value_without_mass`, `published_product_absent`, `not_published` |
| `flag_uv_low`, `flag_uv_high` | Unit value checks. Low removes mass; high only flags |
| `first_released` | First Comtrade release, for as-of replay |

Absent months stay absent. Nothing is interpolated and no month is filled with zero.

## Security

Keys are read from the environment only, sent as a header rather than in URLs, and masked in logs. `.env` is git-ignored.
