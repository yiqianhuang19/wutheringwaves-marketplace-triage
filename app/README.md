# Marketplace triage scripts

## Rule baseline

Run the portable baseline from the repository root:

```bash
python app/run_baseline.py
python app/evaluate_baseline.py
```

`run_baseline.py` reads `data/trade_posts.xlsx` and writes `data/baseline_output_development.xlsx`. It uses only `pandas` and `openpyxl` from `requirements.txt`; it does not need an API key, Node.js, or network access.

The script implements the transparent keyword rules documented in `data/baseline_rules.xlsx`. It is intentionally limited and is the non-AI comparison for the project.

`evaluate_baseline.py` compares that output with `data/ground_truth_development.xlsx` and writes `data/baseline_evaluation_development.xlsx`.
