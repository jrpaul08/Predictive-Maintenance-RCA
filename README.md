# Finding Precursors to Rare Events
Take-home assignment for Oden Technologies: find the signals that show up before rare component failures, turn them into simple detectors, and test them with lift, precision and coverage.

## Data

[Microsoft Azure Predictive Maintenance](https://www.kaggle.com/datasets/arnabbiswas1/microsoft-azure-predictive-maintenance)

## How to Run

Requires Python 3.9+.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Then open the notebooks and run all cells in order.

## Notebooks

All analysis, design choices and findings are written up inside the notebooks.

| Notebook | What it covers |
|---|---|
| `01_data_exploration.ipynb` | EDA of every dataset along with its findings |
| `02_initial_precursor_analysis.ipynb` | Design choices (H, W, normal operation, metrics). Four candidate precursors built and tested in detail on comp1 |
| `03_final_precursor_analysis_and_findings.ipynb` | The Precursor design applied to all four components, the results table, final findings, shortlist and what I would do if I had more time|


## AI Usage

I used Claude Code to help with code and write-up. Transcript is in `ai_transcript.md`.

