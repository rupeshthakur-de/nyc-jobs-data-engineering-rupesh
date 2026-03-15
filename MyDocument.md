# MyDocument.md — NYC Jobs Assessment: Learnings, Challenges & Assumptions

---

## Overview

This document captures the key considerations, design decisions, challenges encountered, and assumptions made during the NYC Jobs data engineering assessment.

---

## Assumptions

| # | Assumption | Rationale |
|---|-----------|-----------|
| 1 | Salary comparisons use **Annual** frequency only for direct KPIs (2–5) | Mixing hourly/daily with annual figures without normalisation would be misleading |
| 2 | Hourly salary annualised at **2,080 hrs/year** (40 hrs × 52 weeks) | Standard US full-time working year |
| 3 | Daily salary annualised at **260 days/year** (5 days × 52 weeks) | Standard US business days |
| 4 | "Last 2 years" (KPI 5) is computed relative to the **latest Posting Date in the dataset**, not today's date | The dataset is historical; using today would exclude all data |
| 5 | `Salary Range To` is used as the ceiling for KPI 4 (highest salary per agency) | It represents the maximum earnable, which is the most relevant measure for "highest salary" |
| 6 | KPI 6 (highest paid skills) is scoped to the NYC government market | The source data is exclusively NYC public sector; broader US market generalisation requires additional datasets |
| 7 | Rows with `Salary Range From = 0` are **not filtered out** in all queries | Some legitimate roles (e.g. volunteer/intern-adjacent) may have a zero lower bound; filtering is applied selectively per KPI |

---

## Design Decisions

### Data Pipeline Architecture
- A **chain of composable transform functions** was chosen over a monolithic processing block. This makes each step independently testable and the pipeline easy to extend.
- Spark's `.transform()` chaining pattern is used to keep the pipeline readable and avoid intermediate variable proliferation.

### Feature Engineering Choices

| Feature | Technique | Justification |
|---------|-----------|---------------|
| `annual_salary_mid` | Derived / normalised numerical | Enables like-for-like salary comparison across all posting types |
| `salary_band` | Discretisation / binning | Creates an ordinal target useful for segmentation and ML classification |
| `posting_age_days` | Date arithmetic | Proxy for position difficulty or urgency; useful for demand modelling |
| `posting_year` / `posting_quarter` | Temporal decomposition | Enables time-series trend analysis without string parsing |
| `min_degree` + `degree_rank` | Text extraction + ordinal encoding | Converts free-text qualification requirements into a structured, rankable feature |

### Column Removal Rationale

| Removed Column | Reason |
|---------------|--------|
| `To Apply` | Administrative instructions; no analytical value |
| `Additional Information` | Largely duplicates `Job Description`; high null rate |
| `Recruitment Contact` | PII; not relevant to analysis |
| `Hours/Shift` | >85% null; non-standard formatting |
| `Work Location 1` | Duplicate of `Work Location` |

---

## Challenges

### 1. Multi-line CSV Parsing
Job descriptions frequently contain embedded newlines and quoted commas, causing standard single-line CSV readers to mis-parse rows. **Solution:** `multiLine=true` and `escape='"'` options in `spark.read.csv()`.

### 2. Degree Extraction from Free Text
The `Minimum Qual Requirements` column uses diverse phrasings ("baccalaureate", "bachelor's", "B.A.") for the same degree level. **Solution:** regex patterns with multiple synonyms per degree level. A more robust approach would use an NLP model (SpaCy NER), but regex is sufficient for this scale.

### 3. Multi-valued Job Categories
Some postings list multiple categories in a single cell (e.g. `"Engineering, Architecture, & Planning Finance, Accounting, & Procurement"`). **Solution:** these composite values are treated as a single distinct category, which inflates the long tail. A proper fix would split on a canonical delimiter, but the data format doesn't provide a reliable one.

### 4. Salary = 0 Edge Cases
~12 rows have `Salary Range From = 0`. These appear to be data quality issues (missing values stored as 0) rather than true zero-salary roles. They are excluded from average salary KPIs but retained in the processed dataset with a `null` `annual_salary_mid`.

### 5. No Spark ML Available in 2.4.5
PySpark 2.4.5's MLlib was available but formal correlation testing (Pearson) was performed using `scipy.stats` after collecting the aggregated summary to the driver — an acceptable trade-off for a small aggregated result (~5 rows).

---

## Learnings

- **Parquet as output format** is strongly preferred over CSV for downstream consumption: columnar storage gives 5–10× better read performance on filtered queries, and schema is preserved without re-inference.
- **Window functions** (e.g. `row_number().over(window)`) are the idiomatic Spark way to compute per-group top-N without a self-join, which would be expensive at scale.
- **UDFs** carry a serialisation cost in Python (data leaves JVM → Python → JVM). For production at large scale, these should be rewritten using native Spark `F.regexp_extract` or Pandas UDFs (vectorised) to keep data in the JVM.
- The `.transform()` pattern significantly improved testability — each function could be unit-tested with a small `createDataFrame` mock independently of the full pipeline.

---

## Proposed Trigger & Scheduling Approach

### Manual / Ad-hoc
```bash
# From within the Docker environment:
spark-submit --master spark://master:7077 \
             --py-files /app/utils.py \
             /app/nyc_jobs_pipeline.py
```

### Scheduled via Apache Airflow DAG
```python
# Conceptual DAG structure (not part of submission code)
with DAG("nyc_jobs_daily", schedule_interval="@daily") as dag:
    ingest  = BashOperator(task_id="ingest",   bash_command="...")
    process = SparkSubmitOperator(task_id="process", application="/app/pipeline.py")
    validate = PythonOperator(task_id="validate", python_callable=run_tests)
    ingest >> process >> validate
```

### CI/CD Recommendation
- Store the pipeline as a versioned `.py` module in the repository.
- On each push to `main`, a GitHub Actions workflow runs the test suite against a local SparkContext.
- Passing builds trigger a Docker image rebuild and push to a container registry for deployment.

---

*Document maintained by: Candidate | Assessment Date: March 2026*
