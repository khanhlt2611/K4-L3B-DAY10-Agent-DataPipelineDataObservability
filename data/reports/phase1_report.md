# Phase 1 — Baseline Data Pipeline Report

## Source summary

| Field | Value |
|---|---|
| source | Crossref REST API |
| query | agentic retrieval augmented generation large language model |
| raw_records | 24 |
| clean_records | 24 |
| embedding_model | sentence-transformers/all-MiniLM-L6-v2 |
| collection | papers-baseline |

## Evaluation metrics

| Metric | Value |
|---|---:|
| Retrieval Hit Rate | 1.0000 |
| Mean Token F1 | 1.0000 |
| Judge Accuracy | 1.0000 |
| Mean Judge Score | 5 |
| Samples | 10 |
| Ragas | Set RUN_RAGAS=1 to enable the slower Ragas pass. |

## Data quality

**Overall status:** PASS

| Expectation | Column | Status |
|---|---|---|
| expect_table_row_count_to_be_between | — | PASS |
| expect_column_values_to_not_be_null | paper_id | PASS |
| expect_column_values_to_not_be_null | title | PASS |
| expect_column_values_to_be_unique | paper_id | PASS |
| expect_column_value_lengths_to_be_between | summary | PASS |

## Freshness SLA

| Signal | Value |
|---|---:|
| Status | PASS |
| Latest published | 2026-09-15 |
| Oldest published | 2026-04-01 |
| Stale rows | 0 / 24 |
| Stale ratio | 0.0000 |
| Threshold | 180 days |

> This report is generated from pipeline artifacts; values must not be edited manually.
