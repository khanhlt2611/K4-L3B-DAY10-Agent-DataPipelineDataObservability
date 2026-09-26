# Corruption Impact and Repair Report

## Baseline vs Corrupted vs Repaired

| Metric | Baseline | Corrupted | Repaired | Corruption delta | Repair delta |
|---|---:|---:|---:|---:|---:|
| Retrieval Hit Rate | 1.0000 | 0.8000 | 1.0000 | -0.2000 | +0.2000 |
| Mean Token F1 | 1.0000 | 0.8108 | 1.0000 | -0.1892 | +0.1892 |
| Judge Accuracy | 1.0000 | 0.8000 | 1.0000 | -0.2000 | +0.2000 |
| Mean Judge Score | 5 | 4.2000 | 5 | -0.8000 | +0.8000 |

## Quality and freshness recovery

Baseline quality and freshness are recorded in `phase1_report.md`. The table below compares the failure and repair stages supplied to this report.

| Signal | Corrupted | Repaired |
|---|---:|---:|
| Quality Gate | FAIL | PASS |
| Failed expectations | 3 | 0 |
| Freshness SLA | FAIL | PASS |
| Stale rows | 8 | 0 |

## Interpretation

- A negative corruption delta indicates degradation relative to the baseline.
- A positive repair delta indicates recovery after rebuilding from the trusted raw snapshot.
- Conclusions should only be made when the generated values show a measurable change.

> This report is generated from pipeline artifacts; values must not be edited manually.
