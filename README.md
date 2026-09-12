# AI Dynamic Pricing — Business App

A Streamlit web app that wraps your trained XGBoost demand model so a
business user can get a pricing recommendation without touching Python,
pandas, or any column names. No ML jargon appears anywhere in the UI.

## What's in this folder

| File | Purpose |
|---|---|
| `app.py` | The Streamlit app (UI + prediction logic) |
| `model.json` | Your trained XGBoost model, exported in XGBoost's portable format (not a raw pickle, so it won't break across xgboost versions) |
| `feature_names.json` | The exact 153 column names/order the model expects |
| `lookup.csv` | A small (400 KB) precomputed table with each center+meal's category, last known price, and last 12 weeks of demand — used to auto-fill the technical features (lags, rolling averages, week-of-year) so the user never has to enter them |
| `meal_elasticity.csv`, `model_metrics.csv` | Copied from your `outputs/recommendations/` folder, shown in the app for context |

## Why this approach (important)

Your `model/model.pkl` is an **XGBoost model trained on one-hot encoded
columns** (`center_id_11`, `ProductID_1109`, `ProductCategory_Pizza`, etc.)
from `notebooks/00_original_enhanced_workbook.ipynb` — not the same model
as `src/demand_model.py` (that one is a separate ridge-regression model
used by `run_pipeline.py`). There was no saved preprocessor, so `app.py`
rebuilds the exact 153-column vector by reading `model.feature_names_in_`
directly — this is the "training features vs UI features mismatch" problem
called out in your notes, solved.

The lag/rolling-demand features (`Lag_1`, `Lag_2`, `Lag_4`, `Rolling_4`,
`Rolling_12`) are calculated automatically from each product's real
history in `lookup.csv` — the business user only enters a candidate price
and two Yes/No toggles.

## Run it locally

```bash
pip install -r requirements.txt
streamlit run app.py
```

It will open at `http://localhost:8501`.

## Deploy for free (shareable URL)

1. Create a new GitHub repository, e.g. `ai-dynamic-pricing`.
2. Upload everything in this folder to that repository (keep the file names as-is).
3. Go to [share.streamlit.io](https://share.streamlit.io) and sign in with GitHub.
4. Click **New app** → select your repository → branch `main` → file `app.py` → **Deploy**.
5. After a couple of minutes you'll get a URL like:
   `https://your-app-name.streamlit.app`

Share that URL with the business user — nothing else to install on their side.

## Updating with a new model later

If you retrain the model, re-export it the same way instead of committing
a raw pickle (safer across environments):

```python
import pickle, json
with open("model/model.pkl", "rb") as f:
    model = pickle.load(f)

model.get_booster().save_model("model.json")
with open("feature_names.json", "w") as f:
    json.dump(list(model.feature_names_in_), f)
```

Then replace `model.json` and `feature_names.json` in this folder and
redeploy (Streamlit Cloud auto-redeploys on every git push).

## Limitation to keep showing users

This model reflects historical, observational pricing data — it shows
association, not a guaranteed causal effect of a price change. Treat
large recommended price changes as a hypothesis to validate with a
small controlled test, not an automatic action.
