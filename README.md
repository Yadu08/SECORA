<div align="center">

# SECORA

**Secure ESG and Cyber Operations Risk Analyzer**

Two independent anomaly detectors, one auditable risk score, and a hash-chained ledger that makes any later edit detectable.

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-1.5-F7931E?logo=scikitlearn&logoColor=white)](https://scikit-learn.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[Live demo](https://secora-rpaz.onrender.com) · [How it works](#how-it-works) · [Screenshots](#screenshots) · [Quick start](#quick-start-with-docker) · [API](#api-reference)

</div>

![SECORA landing page](docs/screenshots/landing.png)

SECORA scores companies on two kinds of risk at once: what their ESG disclosures look like, and what their cyber incident history looks like. Two independent Isolation Forests flag anomalies in each domain, a power mean combines them into one 0-100 risk score, and every score is written to a SHA-256 hash-chained audit ledger, so any later edit to a stored result can be detected.

The point of the design is that an unexplained score is not worth much to a risk analyst. Every number the dashboard shows can be traced back: which features drove it, which model version produced it, and whether the stored record still matches its own hash.

## Contents

- [Why it is built this way](#why-it-is-built-this-way)
- [Features](#features)
- [Screenshots](#screenshots)
- [How it works](#how-it-works)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Quick start with Docker](#quick-start-with-docker)
- [Local development without Docker](#local-development-without-docker)
- [Scraping data and retraining](#scraping-data-and-retraining)
- [Deploying to Render](#deploying-to-render)
- [API reference](#api-reference)
- [CSV format](#csv-format)
- [Demonstrating tamper detection](#demonstrating-tamper-detection)
- [Design decisions and trade-offs](#design-decisions-and-trade-offs)
- [Roadmap](#roadmap)
- [License](#license)

## Why it is built this way

Three constraints shaped the architecture.

**One model per domain, not one model over everything.** ESG disclosure features and cyber incident features live on different scales and mean different things. Training a single detector over the union of both lets a clean record in one domain mask an outlier in the other. Two independent detectors keep the domains separable, and the aggregation step decides explicitly how much a strong signal in one domain should count.

**Aggregation that does not average away a crisis.** A plain mean of 0.95 and 0.05 is 0.50, which reads as unremarkable. A cubic power mean over the same pair returns about 0.75, because cubing before averaging keeps the dominant term visible. That is the behaviour a risk function wants: one serious problem should not be cancelled by good news elsewhere.

**A result that can be checked later.** Scores that feed a decision need to be defensible after the fact. Each stored result is hashed together with the hash before it, so editing any record invalidates that record and every record after it. Verification recomputes the chain from the genesis hash and reports the first record that does not match.

## Features

**Landing page (`/`)** explains the pipeline for first-time visitors. It walks through the four steps (ingest, detect, combine, record) and lists the features each detector uses. An interactive slider compares the power mean with a plain average, and a hash-chain diagram shows how tampering is caught. "Try it with demo data" opens the dashboard and scores the demo set in one click. A badge shows the live ledger status.

**Dashboard (`/app`)**
- **Overview:** headline numbers, a risk-level donut, a 0-100 risk histogram, a sector heatmap, and the ten highest-risk flagged companies.
- **Flagged companies:** search and filter by level and sector. The inspector shows the company's risk gauge, both domain scores, its ledger request ID, and per-feature z-scores that explain why it was flagged.
- **Audit trail:** every ledger record with copyable hashes, plus one-click verification of the full chain. The first broken record is highlighted.
- **Data in:** load the demo data, upload a CSV (button or drag and drop, with structural validation and row-level error messages), download a CSV template, or re-run the analysis.
- **Failure fallback:** if scoring fails (missing or tampered model, database error), the API serves the last committed results and the UI labels them as stale instead of showing nothing.

**UI:** responsive down to phone width, light and dark themes (follows the system setting, with a toggle in the header), loading placeholders, and keyboard-accessible controls.

## Screenshots

### Dashboard overview

Headline counts, risk-level split and the distribution of the aggregate score across the scored universe.

![Dashboard overview](docs/screenshots/dashboard-overview.png)

### Flagged companies and the inspector

Selecting a company shows both domain scores, its ledger request ID, and the per-feature z-scores that explain the flag. Bars beyond 2σ are highlighted, so the reason for a score is visible rather than asserted.

![Company inspector with z-score explanations](docs/screenshots/inspector.png)

### Audit trail

Every record carries its tuple, its own hash and the previous hash. "Verify chain now" recomputes all of them from the genesis hash.

![Audit trail with hash chain verification](docs/screenshots/audit-trail.png)

### Tamper detection

After editing one stored value directly in the database, verification fails at exactly that record and the badge turns red. The record was altered without going through the API, and the chain still caught it.

![Tamper detected at record 5](docs/screenshots/tamper-detected.png)

### Dark theme and mobile

<table>
<tr>
<td width="72%"><img src="docs/screenshots/dashboard-dark.png" alt="Dashboard in dark theme"></td>
<td width="28%"><img src="docs/screenshots/mobile-dashboard.png" alt="Dashboard on a phone-width screen"></td>
</tr>
</table>

### Explainer sections on the landing page

<table>
<tr>
<td width="50%"><img src="docs/screenshots/how-it-works.png" alt="How it works section"></td>
<td width="50%"><img src="docs/screenshots/hash-chain.png" alt="Hash chain diagram"></td>
</tr>
</table>

## How it works

```
CSV upload / demo data
        │
        ▼
 validation ──► ESG features (9) ──► StandardScaler ──► Isolation Forest ──► s_E ┐
            └─► Cyber features (6) ─► StandardScaler ──► Isolation Forest ──► s_C ┤
                                                                                  ▼
                                        R = (α·s_E^p + (1-α)·s_C^p)^(1/p) × 100, α = 0.5, p = 3
                                                                                  │
                                         LOW < 25 ≤ MEDIUM < 45 ≤ HIGH < 65 ≤ CRITICAL
                                                                                  │
                                                                                  ▼
                              τ = ⟨t, R, s_E, s_C, rid, mh⟩ ──► h = SHA256(CanonicalJSON(τ) ‖ h_prev)
```

| Piece | Detail |
|---|---|
| ESG features (9) | 3 pillar scores, industry-normalised composite (z within sector), YoY change, emissions intensity (emissions / revenue), expenditure share (ESG spend / revenue), sector-relative position (composite minus sector median), score consistency (composite minus mean of pillars) |
| Cyber features (6) | Incident count, breach count, average severity, mean days to detect, log10 records exposed, critical vulnerabilities. These are company-level; to use CICIDS2017 flow statistics instead, edit `CYBER_FEATURES` and `build_cyber_features` |
| Detectors | One `StandardScaler` + `IsolationForest(n_estimators=200, contamination=0.15)` per domain, trained and stored separately |
| Domain score | `-score_samples` calibrated to [0, 1]: the training median maps to 0, the training 99.5th percentile to 1 |
| Aggregate | Cubic power mean with α = 0.5. Cubing before averaging keeps one strong anomaly visible instead of letting a clean score in the other domain average it away |
| Risk bands | LOW < 25, MEDIUM < 45, HIGH < 65, CRITICAL ≥ 65 (edit `RISK_BANDS` in `backend/app/config.py`) |
| Explanations | Per-feature z-scores from each domain's scaler, sorted by magnitude |
| Model manifest | `models/manifest.json` records features, parameters, calibration and the SHA-256 of each model file. Models refuse to load if a file does not match. `mh` is the SHA-256 of the canonical manifest JSON |
| Ledger | One record per company per run. Genesis `h_prev` is 64 zeros. `rid` is `<run id>-<n>` and links the record to its `anomaly_results` row. Verification recomputes every hash from genesis and reports the first mismatch |
| Fallback | If scoring fails, `/api/analyze` returns HTTP 200 with `status: "fallback"` and the committed `backend/data/fallback_results.json`. `/api/dashboard` and `/api/results` do the same when no completed run exists |

## Tech stack

| Layer | Tools |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy, scikit-learn, pandas, numpy |
| Frontend | React 18, Vite, Tailwind CSS, Chart.js (react-chartjs-2) |
| Database | PostgreSQL 16 (SQLite when `DATABASE_URL` is unset, for local development) |
| Scraping | BeautifulSoup, yfinance |
| Runtime | Docker, Docker Compose, nginx (serves the built frontend) |
| Hosting | Render Blueprint (`render.yaml`) |

Database tables: `datasets`, `companies`, `cyber_threats`, `analysis_runs`, `anomaly_results`, `blockchain_audit`.

## Project structure

```
secora/
├── docker-compose.yml            # db + backend + frontend
├── render.yaml                   # Render Blueprint: Postgres + 2 Docker web services
├── FEATURE_IDEAS.md              # add-ons that fit the current architecture
├── docs/screenshots/             # images used in this README
├── backend/
│   ├── Dockerfile, start.sh, requirements.txt, requirements-scraper.txt
│   ├── app/
│   │   ├── main.py               # FastAPI app, CORS, startup (DB init, model load)
│   │   ├── routes.py             # all /api routes
│   │   ├── services.py           # CSV validation, ingestion, analysis, fallback logic
│   │   ├── ledger.py             # hash chain: append + verify
│   │   ├── dashboard.py          # aggregation for charts and heatmap
│   │   ├── fallback.py           # committed stale results
│   │   ├── models.py, database.py, config.py, utils.py
│   │   └── ml/ features.py, pipeline.py, demo_data.py
│   ├── scripts/ scrape_data.py, train.py
│   └── data/fallback_results.json   # generated by train.py, commit it
└── frontend/
    ├── Dockerfile, nginx.conf.template, 40-runtime-config.sh   # API URL injected at container start
    ├── index.html, tailwind.config.js, vite.config.js
    └── src/
        ├── pages/ Landing.jsx (/), Dashboard.jsx (/app)
        ├── components/           # Header, ControlPanel, Overview, Companies, Inspector, AuditTrail, ui.jsx
        ├── lib/ router.jsx, theme.jsx (light/dark tokens), risk.js, chart.js
        └── api.js                # fetch wrapper for every /api route
```

## Quick start with Docker

Requires Docker Desktop (or Docker Engine with the Compose plugin).

```bash
git clone https://github.com/Yadu08/SECORA.git
cd SECORA
docker compose up --build
```

- App: http://localhost:3000 (landing page), http://localhost:3000/app (dashboard)
- API docs (Swagger): http://localhost:8000/docs

The backend image trains both models during `docker build`, so it starts ready. Click **Try it with demo data** on the landing page, or **Load demo data** in the dashboard.

Quick API check:

```bash
curl -X POST localhost:8000/api/demo
curl -X POST localhost:8000/api/analyze -H 'content-type: application/json' -d '{}'
curl localhost:8000/api/blockchain/verify
```

Reset everything, including the database volume: `docker compose down -v`.

## Local development without Docker

**Backend** (from `backend/`, Python 3.12):

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m scripts.train              # creates models/ and data/fallback_results.json
uvicorn app.main:app --reload        # http://localhost:8000
```

Leave `DATABASE_URL` unset to use a local SQLite file, or point it at Postgres:
`DATABASE_URL=postgresql://secora:secora@localhost:5432/secora`.

**Frontend** (from `frontend/`, Node 18+):

```bash
npm install
npm run dev                          # http://localhost:5173, proxies /api to :8000
```

`npm run build` writes the production bundle to `frontend/dist/`.

## Scraping data and retraining

```bash
cd backend
pip install -r requirements.txt -r requirements-scraper.txt
python -m scripts.scrape_data --max-esg 120   # writes data/training_companies.csv
python -m scripts.train                       # writes models/*.joblib, manifest.json, data/fallback_results.json
```

What the scraper collects:
- The S&P 500 universe, its sectors, and a breach history, from Wikipedia (BeautifulSoup).
- Best-effort Yahoo Finance ESG scores and revenue through `yfinance`, for the first `--max-esg` tickers. Yahoo returns nothing for many tickers, and its layout changes.

Per-company emissions, ESG spend and operational cyber metrics are not freely available. Missing values are imputed from the seeded generator and marked by the `esg_real` and `breach_real` columns in the CSV. Treat the imputed columns as synthetic, and check the scraped sites' terms of use before running the scraper at scale.

`train.py` always adds 600 synthetic companies to the scraped rows. After retraining, rebuild the backend image (`docker compose build backend`), and commit `data/fallback_results.json` and `data/training_companies.csv`.

## Deploying to Render

1. Push the repo to GitHub, including `backend/data/fallback_results.json`.
2. In Render, choose **New > Blueprint** and select the repo. `render.yaml` creates `secora-db` (Postgres), `secora-backend` and `secora-frontend`.
3. After the first deploy, note each service's public URL. If Render added a suffix because a name was taken, update the URLs:
   - `secora-backend` > Environment > `CORS_ORIGINS`: set it to the frontend URL.
   - `secora-frontend` > Environment > `API_BASE_URL`: set it to the backend URL, with no trailing slash.

   Then redeploy both services. The frontend needs no rebuild, because the URL is written into `config.js` when the container starts.
4. Open the frontend URL and click **Try it with demo data**. Check that `https://<backend>/api/health` returns `"status": "ok"`.

Free-plan notes:
- Free web services sleep after inactivity, so the first request can take about a minute.
- Free Postgres databases expire after 30 days.
- Backend build time includes model training, which takes under a minute.

## API reference

| Method | Route | Purpose |
|---|---|---|
| GET | `/api/health` | Database check and whether the models loaded |
| GET, POST | `/api/demo` | Create (once) and return the built-in synthetic demo dataset |
| POST | `/api/upload` | Upload a CSV (multipart `file`). Validates structure and ranges; returns 422 with a list of errors on failure |
| POST | `/api/analyze` | Body `{"dataset_id": n}` or `{}` for the latest dataset. Scores companies, stores results, appends ledger records. Returns `status: "completed"` or `status: "fallback"` |
| GET | `/api/results` | Scored companies with explanations. Query: `run_id`, `flagged_only`, `level`, `sector`, `q`, `limit`, `offset` |
| GET | `/api/dashboard` | Summary, histogram, sector heatmap and top flagged companies. Query: `run_id` |
| GET | `/api/blockchain` | Ledger records, newest first. Query: `limit`, `offset` |
| GET | `/api/blockchain/verify` | Recomputes the chain. Returns `valid`, `length`, `head_hash`, `first_invalid_idx`, `reason` |

Interactive docs are served at `/docs` on the backend.

## CSV format

Header names are case-insensitive. You need at least 20 rows with unique company names. Score columns must be 0-100 and `avg_severity` 0-10. `revenue` must be above 0, and counts must not be negative. The dashboard's **CSV template** link downloads a file with this header.

| Column | Meaning |
|---|---|
| `company`, `sector` | Name and sector (required) |
| `environmental_score`, `social_score`, `governance_score`, `composite_score` | ESG scores, 0-100 |
| `emissions` | tCO2e |
| `revenue`, `esg_expenditure` | USD millions |
| `incident_count`, `breach_count`, `critical_vulns` | Counts |
| `avg_severity` | 0-10 |
| `mttd_days` | Mean days to detect |
| `records_exposed` | Records exposed in breaches |
| `ticker`, `prior_composite_score` | Optional; the prior score drives the YoY feature |

## Demonstrating tamper detection

```bash
docker compose exec db psql -U secora -d secora -c "UPDATE blockchain_audit SET risk = 1 WHERE idx = 5;"
curl localhost:8000/api/blockchain/verify    # valid: false, first_invalid_idx: 5
```

The ledger badge turns red, and the Audit trail tab highlights record 5 as the first mismatch. This is the state captured in the [tamper detection screenshot](#tamper-detection) above.

To see the fallback path, change one byte of `backend/models/esg_detector.joblib` and restart the backend. The model then fails its manifest check, and **Run analysis again** serves the committed results with a stale-results banner.

## Design decisions and trade-offs

Some choices are worth stating plainly, including where the current design stops short.

**The hash chain is tamper-evident, not tamper-proof.** It lives in the same database it protects. Someone with write access to `blockchain_audit` can alter a record and recompute every hash after it, and verification will then report an intact chain. What the chain does guarantee is that a partial edit — a value changed in place, a row deleted, a record reordered — is caught immediately and localised. Closing the remaining gap needs an anchor outside the database; that is the first item in [FEATURE_IDEAS.md](FEATURE_IDEAS.md).

**Isolation Forest, not a supervised classifier.** There is no labelled ground truth for "this company's disclosures are misleading". Unsupervised outlier detection answers the question that is actually answerable from the data: which records are unlike the rest of the population. The cost is that `contamination` sets the flag rate by assumption rather than by evidence, and that scores are only meaningful relative to the training population.

**Scores are calibrated, not probabilistic.** Mapping the raw anomaly score through the training median and 99.5th percentile gives a stable 0-100 range that is comparable across runs of the same model. It is not a probability of anything, and the README avoids presenting it as one.

**Training data is partly synthetic.** Per-company emissions, ESG spend and operational cyber metrics are not freely available at scale. The scraper collects what is public and the generator fills the rest, with the imputed columns marked. The pipeline is built to be retrained on real data by replacing the CSV; the detector code does not change.

**The write endpoints are open.** This is a demo deployment with no authentication, so anyone who finds the URL can append to the ledger. Adding an API key is a small change and is item 2 in the feature list, but it is not in the current build and the README does not imply otherwise.

## Roadmap

[FEATURE_IDEAS.md](FEATURE_IDEAS.md) lists 11 additions that fit the current design, with effort estimates and the files each one touches. Among them: anchoring the ledger head hash outside the database, API-key protection, results export, run-to-run comparison, what-if sliders for α and p, and a data-drift warning.

## License

Released under the [MIT License](LICENSE).

## Author

**Yadunandan Srinivasa**
[GitHub](https://github.com/Yadu08) · [LinkedIn](https://linkedin.com/in/yadunandan-s)
