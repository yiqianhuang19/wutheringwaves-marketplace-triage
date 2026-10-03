# Marketplace Operations Triage

## Purpose

This project supports marketplace operations staff who process batches of Wuthering Waves merchandise posts. It extracts key fields, identifies operational ambiguity, and routes posts that need a person to review them.

The system does not approve payments, contact users, or make final enforcement decisions. It supports a human-review workflow.

## What the system does

For each post, the system extracts product name, category, price in USD, condition, transaction type, whether human review is needed, and a short review reason. The batch interface separates routine posts from the human-review queue.

## Product architecture

```text
Excel batch with Post text column, or pasted marketplace posts
                         |
                         v
              Gradio batch triage interface
                         |
                         v
      Fixed v2 extraction prompt and OpenRouter LLM call
                 (gpt-4o-mini, temperature 0)
                         |
                         v
       Post-processing and platform-policy validation
       - normalise JSON fields
       - detect approximate or missing prices
       - preserve unknown values for unclear cases
       - attach an observable review reason
                         |
              +----------+----------+
              |                     |
              v                     v
  Automatically processed posts   Human-review queue
  structured operational fields   ambiguous, multi-item, transfer,
                                  price, or direction-risk cases

Development and evaluation path
trade_posts.xlsx + fixed ground truth --> rule baseline / LLM v1 / LLM v2
                                             |
                                             v
                         evaluation workbooks and accuracy metrics
                                             |
                                             v
                   separate 20-post holdout evaluation for LLM v2
```

The system does not use retrieval, autonomous agents, payment tools, or enforcement actions. It is a decision-support workflow: an operator remains responsible for every reviewed decision.

## Repository structure

```text
app/
  batch_triage_app.py             Standalone batch operations interface
  run_baseline.py                 Portable transparent rule-based baseline
  evaluate_baseline.py            Portable baseline evaluation script
data/
  trade_posts.xlsx                40 development posts
  ground_truth_development.xlsx   Fixed development labels
  trade_posts_holdout.xlsx        20 independent holdout posts
  ground_truth_holdout.xlsx       Fixed holdout labels
  baseline_*.xlsx                 Rule-based baseline outputs
  llm_*.xlsx                      LLM outputs and evaluations
notebooks/
  marketplace_triage_llm_development.ipynb  Colab development notebook
  marketplace_triage_colab_export.py        Read-only Python export of the notebook
docs/
  Project documentation and final submission material
```

## Run the batch interface

Install dependencies:

```bash
pip install -r requirements.txt
```

Set an OpenRouter key as an environment variable, or the program will ask for it without displaying it:

```powershell
$env:OPENROUTER_API_KEY = "your-key"
python app/batch_triage_app.py
```

Open the local address printed in the terminal. Add `--share` only when a temporary public demonstration link is required. The interface accepts an Excel file with a `Post text` column. `Post ID` is optional. It can also process posts pasted as paragraphs.

## Reproduce the rule baseline without an API key

The portable Python scripts use only the packages in `requirements.txt`; they do not make LLM calls or require Node.js. From the repository root, run:

```bash
python app/run_baseline.py
python app/evaluate_baseline.py
```

The first command recreates `data/baseline_output_development.xlsx`. The second command compares it with the fixed development labels and recreates `data/baseline_evaluation_development.xlsx`.

## Evaluation design

The development set contains 40 posts. Labels were fixed before model execution. A transparent keyword-rule baseline was evaluated before the LLM versions. The holdout set contains 20 separate, anonymised marketplace-style posts and was evaluated only after the v2 prompt and policy were fixed.

### Development results: 40 posts

| Metric | Rule baseline | LLM v1 | LLM v2 |
|---|---:|---:|---:|
| Category accuracy | 67.5% | 65.0% | 92.5% |
| Price accuracy | 72.5% | 77.5% | 75.0% |
| Condition accuracy | 87.5% | 77.5% | 82.5% |
| Transaction-type accuracy | 70.0% | 57.5% | 75.0% |
| Human-review accuracy | 82.5% | 67.5% | 80.0% |
| All tracked fields correct | 35.0% | 22.5% | 47.5% |

### Independent holdout results: 20 posts

| Metric | LLM v2 |
|---|---:|
| Category accuracy | 55.0% |
| Price accuracy | 95.0% |
| Condition accuracy | 75.0% |
| Transaction-type accuracy | 70.0% |
| Human-review accuracy | 75.0% |
| All tracked fields correct | 20.0% |

## Metrics targeted and reached

These are prototype quality targets for the submitted project, not production service-level commitments. Development results help compare approaches; the separate holdout result is the more cautious indication of generalisation.

| Metric | Prototype target | Development result: LLM v2 | Holdout result: LLM v2 | Interpretation |
|---|---:|---:|---:|---|
| Batch completion | 100% of submitted rows receive an output or explicit error | 40/40 processed; 0 model-call errors | 20/20 evaluated | Achieved in the recorded runs. |
| Category accuracy | >= 80% on development data | 92.5% | 55.0% | Met on development data, but not robust on the difficult holdout set. |
| Price accuracy | >= 75% on development data | 75.0% | 95.0% | Met; prices were often explicit in the holdout posts. |
| Transaction-type accuracy | >= 70% on development data | 75.0% | 70.0% | Met in both reported evaluations. |
| Human-review accuracy | >= 80% on development data | 80.0% | 75.0% | Met on development data; holdout result supports retaining human review. |
| Exact all-field accuracy | >= 40% on development data | 47.5% | 20.0% | Met on development data only. A row fails when any tracked field is wrong. |

The most important result is not the highest individual field score. The drop from 47.5% exact accuracy on development data to 20.0% on holdout data is evidence against full automation and supports the explicit human-review queue.

## Data and privacy boundary

The project uses English synthetic or anonymised marketplace-style posts derived from the A1 theme and user-supplied examples. It does not include usernames, private messages, payment information, purchase records, contact details, or in-person location details. Dollar amounts are test-schema values rather than currency conversions or market prices.

## Limitations

The holdout result shows weaker performance on category classification and exact all-field extraction than on the development set. The system therefore routes ambiguous posts to people rather than making final decisions automatically. Results are based on a small, task-specific dataset and should not be treated as a general marketplace-safety model.
