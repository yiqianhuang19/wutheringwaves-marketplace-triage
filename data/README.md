# Data and evaluation guide

This folder contains the privacy-safe datasets, fixed labels, system outputs, and evaluation workbooks for the Marketplace Operations Triage project. The project processes marketplace-style Wuthering Waves merchandise posts for an operations worker. It does not contain usernames, private messages, contact details, payment records, purchase evidence, or location data.

## Dataset split

| Split | Posts | Purpose | When it was used |
|---|---:|---|---|
| Development | 40 | Build the rule baseline, inspect errors, and improve the LLM prompt from v1 to v2. | During system development. |
| Holdout | 20 | Test the fixed v2 LLM workflow on separate posts. | Only after the v2 prompt and policy were fixed. |

The holdout set deliberately contains difficult examples such as multi-item listings, approximate prices, unclear transaction direction, and pre-order or transfer wording. It is therefore a stress test rather than a representative sample of marketplace traffic. It reduces direct tuning leakage, but it is still a small, task-specific evaluation and does not replace independent real-world annotation.

## Main input and label files

| File | What it contains | Role in the workflow |
|---|---|---|
| `trade_posts.xlsx` | 40 development posts with `Post ID` and original `Post text`. | Input for baseline and LLM development runs. |
| `ground_truth_development.xlsx` | Fixed expected labels for the 40 development posts. | Used only to compare system output after each run. |
| `trade_posts_holdout.xlsx` | 20 separate holdout posts with `Post ID` and `Post text`. | Input for the final v2 holdout run. |
| `ground_truth_holdout.xlsx` | Fixed expected labels for the 20 holdout posts. | Used only to evaluate the final v2 holdout output. |
| `product_catalogue.xlsx` | Reference catalogue inherited from the A1 theme. | Context and provenance reference; it is not a retrieval source for the LLM. |
| `baseline_rules.xlsx` | Plain-language record of the transparent keyword rules. | Makes the non-AI baseline inspectable. |

The ground-truth labels were set before the relevant model outputs were evaluated. They use the same operational schema as the application:

- `product_name`: the main product text, or `unknown`.
- `category`: `badge`, `figure`, `paper_merch`, or `unknown`.
- `price_usd`: one stated test-schema amount, or `unknown`.
- `condition`: `new`, `used`, or `unknown`.
- `transaction_type`: `sell`, `buy`, `trade`, or `unknown`.
- `needs_human_review`: whether an operations worker should check the case.
- `review_reason`: a short observable reason when review is required.

Dollar values are test-schema values for a consistent extraction field. They are not currency conversions, official market prices, or recommendations about pricing.

## Baseline workflow

```text
trade_posts.xlsx
       + baseline_rules.xlsx
       |
       v
run_baseline.py
       |
       v
baseline_output_development.xlsx
       + ground_truth_development.xlsx
       |
       v
evaluate_baseline.py
       |
       v
baseline_evaluation_development.xlsx
```

`baseline_output_development.xlsx` contains one rule-based structured prediction per development post. `baseline_evaluation_development.xlsx` compares those predictions with the development ground truth and reports the baseline metrics. The baseline exists as a transparent non-AI comparison, not as the intended final product.

## LLM workflow

```text
Development: trade_posts.xlsx --> fixed prompt and LLM --> llm_output_development_v2.xlsx
                                                    + ground_truth_development.xlsx
                                                    |
                                                    v
                                      llm_evaluation_development_v2.xlsx

Holdout: trade_posts_holdout.xlsx --> fixed v2 prompt and LLM --> llm_output_holdout_v2.xlsx
                                                         + ground_truth_holdout.xlsx
                                                         |
                                                         v
                                           llm_evaluation_holdout_v2.xlsx
```

The LLM outputs were produced by the v2 workflow documented in `notebooks/marketplace_triage_llm_development.ipynb` and implemented for batch use in `app/batch_triage_app.py`. This README explains the evidence files; the root README explains the product architecture and model configuration.

The intermediate v1 comparison is recorded in the notebook and root README. The final retained Excel outputs are v2 because they are the version tested against the holdout set.

## How the evaluation workbooks are calculated

Each evaluation workbook joins system output to the matching ground truth by `Post ID`. It reports whether the following fields are correct for each post:

- category
- price in USD
- condition
- transaction type
- needs-human-review decision

It then summarises field-level accuracy and **all tracked fields correct**. The latter is strict: a post counts as correct only when every tracked field above is correct. `product_name` remains visible in output files for operational usefulness, but it is not included as a separate exact-match accuracy metric because meaningful product names can have valid wording variations.

For the reported metric values, comparison across baseline, LLM v1, and LLM v2, and the interpretation of the holdout result, see the root `README.md`.
