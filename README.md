# Financial Reconciliation Engine

Matching transactions across two ledgers when the invoice references **don't line up**: mistyped, missing, or split across several payments. A join on invoice number can't handle these cases, and they're where manual reconciliation time goes after a system migration, a merger or a month-end close.

The engine uses **probabilistic record linkage** to link the records, labels every record with a break type a finance team can act on, and measures its own accuracy against known ground truth.

![Confusion matrix](images/Confusion_matrix.png)

---

## Results

Run on 41,280 ledger records built from real UK retail invoices, with simulated discrepancies so that accuracy can be measured exactly (see [How it was tested](#how-it-was-tested)).

| | |
|---|---|
| Records a join on invoice number **cannot** link | 5,209 (12.6%) |
| Of those, correctly linked by the engine | **5,166** (≈ £2.0M) |
| Linking accuracy (record attached to the right invoice) | **99.9%** |
| Misclassified records, first version → final version | **348 → 71** (−80%) |

Per break type, final version:

| Break type | Precision | Recall |
|---|---|---|
| Matched | 1.00 | 1.00 |
| Amount mismatch | 1.00 | 1.00 |
| Timing difference | 1.00 | 1.00 |
| Duplicate posting | 1.00 | 1.00 |
| Key corrupted (mistyped ID) | 0.98 | 0.99 |
| Key missing (blank ID) | 1.00 | 0.99 |
| Split payment | 1.00 | 1.00 |
| Missing in source | 0.91 | 0.93 |

---

## How it works

The engine runs four stages, **in this order on purpose**:

1. **Exact match, with a sanity check.** Records are linked on invoice number, but only if customer, amount (±10%) and date (±7 days) also roughly agree. Invoice numbers are sequential, so a mistyped ID is often *another real invoice number*. Without this check, the record is linked to a stranger's invoice.
2. **Split detection.** For each invoice still unmatched, look for 2–3 loose payments that add up to it (within 2p, posted within 5 days). Pairs use a one-pass "two-sum" lookup. Each part must be smaller than the invoice, so a full-size record can't be used as a "part". This runs *before* fuzzy matching, because otherwise fuzzy matching grabs half-payments and links them to the wrong invoices.
3. **Fuzzy matching.** Remaining records are scored 0–1 against candidate invoices on four fields:

   | Field | Similarity | Weight |
   |---|---|---|
   | Amount | 1 − relative difference | 0.45 |
   | Date | 1 − days apart ÷ 7 | 0.25 |
   | Customer | same / different (left out if missing) | 0.15 |
   | Product signature | Jaccard overlap of top product codes | 0.15 |

   This is a simplified Fellegi–Sunter record-linkage model. Pairs are accepted from the highest score down, above a threshold of 0.80. A missing customer counts as *no information*, not as a mismatch.
4. **Classification.** Every record gets a break type (matched, amount mismatch, timing difference, duplicate, key corrupted, key missing, split, missing in source/target), and the report totals records and £ value for each.

**Blocking** keeps it fast: records are only compared within the same customer, or the same month when the customer is unknown, instead of every record against every invoice.

---

## How it was tested

Real reconciliation data never comes with the right answers, so accuracy can't be measured on it. Instead:

- **System A** is built from the [UCI Online Retail II](https://archive.ics.uci.edu/dataset/502/online+retail+ii) dataset: ~1M real invoice lines rolled up to ~40,000 invoices.
- **System B** is a copy of A with known breaks injected into ~19% of invoices: amount changes, late postings, duplicates, records missing on either side, mistyped IDs, blank IDs and split payments. Each invoice gets at most one break, so its correct answer is unambiguous.
- The correct answers are kept separate and never shown to the engine. They are only used to score it.

---

## Error analysis: from 348 errors to 71

The first version scored 99% overall but still made 348 mistakes. Grouping them by cause:

| Cause | Errors | Fix |
|---|---|---|
| Invoice-ID collisions: a mistyped ID matched another real invoice | 219 (63%) | sanity check on exact matches |
| Bug in the test data: swapping two identical digits left the "corrupted" ID unchanged | 52 (15%) | corruption now always changes the ID |
| Identical orphan entries competing for the same invoice | 63 (18%) | not fixable by scoring (see below) |
| Split payments not reassembled | 14 (4%) | — |

A second issue showed up in the threshold sweep: records with no customer ID could never score above 0.85, so any threshold above that silently excluded them. Treating a missing customer as neutral removed that ceiling.

Of the **71 errors remaining**, 70 are explained:

| Cause | Errors |
|---|---|
| Orphan entries identical to a real invoice (28 wrong links + 24 records pushed out) | 52 |
| Split payments not reassembled | 12 |
| Coincidental splits: two unrelated amounts that happened to add up (4 wrong links + 2 records pushed out) | 6 |
| Unexplained | 1 |

Most errors come in pairs: a wrong link takes an invoice, and the record that really belonged to it is left unmatched. That's an argument for sending ties and borderline scores to **manual review** rather than letting the engine guess.

![Threshold sweep](images/threshold_sweep.png)

---

## Limitations

- **The discrepancies are simulated.** Mistyped and blank-ID records keep their exact amount, date and customer, which is why the threshold sweep is nearly flat: true matches score close to 1.0. Real data is messier, so these results are an upper bound.
- **Identical records can't be told apart.** When two candidates match an invoice equally well, no similarity score can pick the genuine one.
- **The weights are set by hand.** With labelled matches they could be learned, e.g. logistic regression on the four field similarities.
- **Cancellations and credit notes are excluded** from this version.

## Next steps

- Add noise to the simulated breaks (small amount and date shifts) so the threshold has a real trade-off to justify
- A manual-review queue for ties and near-threshold scores
- Learn the field weights from labelled pairs
- Handle cancellations and credit notes as their own break type

---

## Run it

```bash
git clone https://github.com/<your-username>/financial-reconciliation-engine.git
cd financial-reconciliation-engine
pip install -r requirements.txt
```

Download the Online Retail II `.xlsx` from the [UCI page](https://archive.ics.uci.edu/dataset/502/online+retail+ii), save it in this folder as `online_retail_II.xlsx`, then open `reconciliation_engine.ipynb` and run all cells. Loading the Excel file takes a few minutes.

## Data

Online Retail II, UCI Machine Learning Repository: https://archive.ics.uci.edu/dataset/502/online+retail+ii

## Tech

Python · pandas · NumPy · scikit-learn · matplotlib · Jupyter
