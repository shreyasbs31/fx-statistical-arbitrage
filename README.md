# FX Statistical Arbitrage — Residual Relative Momentum

This repository contains a research implementation of a cross-sectional FX momentum strategy that trades PCA-derived residuals ("Residual Relative Momentum"). The core idea is that extreme residual dislocations across FX pairs persist over multi-hour horizons and that continuation of extremes can be traded profitably.

Authors
- Shreyas B S
- Dhruv Agrawal (Team BeatCoin)

Executive summary
- Universe: EUR/USD, GBP/USD, USD/JPY, USD/CNH (mapped to output names EURUSD, GBPUSD, JPYUSD, CNHUSD)
- Signal timescale: 30-minute bars
- Execution timescale: 5-second bars
- PCA window: 48 bars (24h)
- Z-score window: 24 bars (12h)
- Entry |Z| >= 1.75, Exit |Z| <= 0.9
- Top-K extremes: 2
- Minimum holding period: 3 hours (6 bars)

Repository contents
- `fx_statistical_arbitrage_final.ipynb` — primary notebook with code, plots, backtest logic, and results.
- `strategy_output.csv` — large output time-series (very large; excluded from repo by default; see Data section).
- `FX-Statistical-Arbitrage-A-Familiar-Idea-That-Keeps-Failing.pptx` — presentation (optional supporting material).
- `Arena_stage_2.pdf` — additional documentation.

Key architecture
1. Slow signal layer (30-min): rolling PCA to remove dominant USD/global factor; compute residuals and cumulative residuals; z-score and select top-K extremes with confirmation.
2. Fast execution layer (5s): map slow signals to execution positions and compute PnL; generate `strategy_output.csv`.
3. Strictly causal calculations: all rolling windows end at t-1; positions are based on signals available at the previous bar.

How to run (local)
1. Create a Python environment (recommended: venv or conda).
2. Install dependencies:

```bash
pip install -r requirements.txt
```

3. Open and run the notebook in Jupyter Lab / Notebook:

```bash
jupyter lab fx_statistical_arbitrage_final.ipynb
```

Data and large files
- `strategy_output.csv` is very large (~3GB). It is excluded from this repo by default (see `.gitignore`) because it will bloat git and is not suitable for normal pushes.
- If you want to include the CSV in the remote repository, use Git LFS and add an attribute entry for `strategy_output.csv`:

```bash
git lfs install
git lfs track "strategy_output.csv"
git add .gitattributes
```

Recommended workflow for sharing results
- Commit all code, notebooks, and smaller assets.
- Store large time-series data (like `strategy_output.csv`) in a cloud bucket (S3/GCS) or use Git LFS.

Dependencies
See `requirements.txt`. The main libs used by the notebook are: `numpy`, `pandas`, `matplotlib`, `scikit-learn`, and Jupyter tooling.

Next steps I can do for you
- Push this repo to GitHub for you (I will need a GitHub remote URL or a Personal Access Token with repo scope).
- Optionally add a small runner script (e.g., `run_backtest.py`) to reproduce the core backtest without the full notebook.

If you want me to push, provide either:
- A GitHub repo remote URL (already created) and permission to push, or
- A GitHub Personal Access Token and the desired repo name/visibility so I can create and push to it.

If you'd like me to include `strategy_output.csv` in the remote, confirm whether you want Git LFS or cloud storage, and I will configure instructions accordingly.
