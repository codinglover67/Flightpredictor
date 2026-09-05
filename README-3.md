# FlightSense — Flight Risk Intelligence

A flight delay and cancellation prediction system: dual ML models (Random Forest + XGBoost) served through a Dockerized R API, with a web frontend for live predictions and a Power BI integration for dashboarding.

## What it does

Given flight details (airline, route, scheduled times, distance, etc.), FlightSense predicts:
1. **Delay probability** — likelihood the flight's arrival is delayed 15+ minutes
2. **Cancellation probability** — likelihood the flight is cancelled

Each prediction is scored by **two independent models** (Random Forest and XGBoost) so the results can be compared side by side, rather than trusting a single model blindly.

## Architecture

```
┌─────────────────┐
│   index.html      │  ← Frontend — collects flight details, calls the API,
│                    │     displays delay/cancel risk
└─────────┬───────────┘
          │ REST (POST)
┌─────────▼───────────┐
│      api.R           │  ← Plumber API: /predict/delay, /predict/cancel
│   entrypoint.R        │  ← Starts the Plumber server on container boot
└─────────┬───────────┘
          │ loads at startup
┌─────────▼─────────────────────────────────────────┐
│  models/                                             │
│   powerbi_rf_arrdel15.rds   powerbi_xgb_arrdel15.json │
│   powerbi_rf_cancelled.rds  powerbi_xgb_cancelled.json│
│   powerbi_del_level_map.rds powerbi_can_level_map.rds │
└─────────────────────────────────────────────────────┘
          ▲
          │ same trained models
┌─────────┴───────────┐
│      Power BI          │  ← Business-facing risk dashboards, using the
│                         │     same .rds / .json model exports
└─────────────────────────┘
```

Everything runs inside a single Docker container (`rocker/r-ver` base image), so the API, models, and dependencies ship together as one deployable unit.

## Modules

### `api.R` — Prediction API
- Built with **Plumber** (R's REST API framework)
- Loads all four models once at server startup (not per-request) for performance
- **`POST /predict/delay`** — takes airline, origin, destination, scheduled departure time, elapsed time, distance, month, and day of week; engineers features (departure hour, time-of-day bucket, weekend flag) on the fly; returns delay probability from both RF and XGBoost
- **`POST /predict/cancel`** — similar flow, plus a high-traffic-origin flag (checked against the top 20 busiest US airports) and distance-group bucketing; returns cancellation probability from both models
- Handles unseen categorical values gracefully by collapsing them to an `"OTHER"` level rather than erroring out
- CORS-enabled so the frontend (or any external client) can call it directly from the browser

### `entrypoint.R` — Server bootstrap
- Loads `api.R` via Plumber and starts the server on `0.0.0.0`, using the `PORT` environment variable if set (defaults to 8000) — this makes it deployable as-is on platforms that inject their own port (Render, Railway, Fly.io, etc.)

### `models/` — Trained model artifacts
- `powerbi_rf_arrdel15.rds`, `powerbi_rf_cancelled.rds` — Random Forest models (delay, cancellation)
- `powerbi_xgb_arrdel15.json`, `powerbi_xgb_cancelled.json` — XGBoost models (delay, cancellation)
- `powerbi_del_level_map.rds`, `powerbi_can_level_map.rds` — factor-level maps used to encode categorical inputs (airline, origin, destination) consistently with training, and to collapse rare/unseen categories to `"OTHER"`
- These same model files are also consumed directly by **Power BI** for dashboard-side scoring, so the API and the BI dashboard stay consistent with each other

### `index.html` — Frontend
- Calls `/predict/delay` and `/predict/cancel` and displays the returned risk scores
- Talks to the API over plain REST/JSON, so it can be hosted separately from the backend (e.g. static hosting) as long as `API` points at the deployed container

### `Dockerfile`
- Base image: `rocker/r-ver:4.3.1`
- Installs system libraries (`libssl-dev`, `libcurl4-openssl-dev`, `libxml2-dev`) and R packages (`plumber`, `randomForest`, `xgboost`)
- Copies the full project into the image and runs `entrypoint.R` on container start
- Exposes port 8000

## Running it locally

### With Docker (recommended)

```bash
git clone https://github.com/codinglover67/flight-api.git
cd flight-api
docker build -t flightsense .
docker run -p 8000:8000 flightsense
```

The API is now live at `http://localhost:8000`.

### Without Docker

```bash
# Install R packages
R -e "install.packages(c('plumber','randomForest','xgboost'))"

# Run the API
Rscript entrypoint.R
```

### Using the frontend

Open `index.html` in a browser (or serve it via any static host), making sure it points at your running API's URL.

## Example request

```bash
curl -X POST http://localhost:8000/predict/delay \
  -H "Content-Type: application/json" \
  -d '{
    "Airline": "AA",
    "Origin": "JFK",
    "Dest": "LAX",
    "CRSDepTime": 1430,
    "CRSElapsedTime": 360,
    "Distance": 2475,
    "Month": 7,
    "DayOfWeek": 5,
    "DepDel15": 0
  }'
```

Returns:
```json
{
  "rf_delay_prob": 0.31,
  "xgb_delay_prob": 0.28,
  "input_echo": { "airline": "AA", "origin": "JFK", "dest": "LAX", "month": 7, "day_of_week": 5 }
}
```

## Tech stack

- **Backend:** R, Plumber (REST API)
- **Models:** Random Forest, XGBoost (dual-model comparison per prediction)
- **Deployment:** Docker
- **BI integration:** Power BI (consumes the same trained model artifacts)
- **Frontend:** HTML/CSS/JS
