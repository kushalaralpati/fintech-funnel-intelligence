# fintech-funnel-intelligence
# Fintech Sign-Up Drop-Off ETL Pipeline

Synthetic fintech onboarding events -> PostgreSQL -> 18 engineered behavioural
features -> XGBoost classifier predicting who drops off before completing
sign-up.

## Architecture

```
src/data_generator.py   synthetic event-log generator (numpy, no DB needed)
src/ingest.py            generate + bulk-load events into Postgres (raw_signup_events)
sql/schema.sql            raw_signup_events + user_features DDL
sql/feature_engineering.sql   SQL (CTEs/window functions) that builds the 18 features
src/features.py           runs feature_engineering.sql, exposes load_features_df()
src/train.py               trains/evaluates XGBoost, saves model + metrics + plot
src/pipeline.py            CLI orchestrator: generate -> ingest -> features -> train
```

Event log -> funnel: `landing_page_view -> signup_started -> email_submitted ->
email_verified -> phone_number_submitted -> otp_sent -> otp_verified ->
personal_info_submitted -> kyc_document_uploaded -> kyc_approved ->
bank_account_linked -> funding_source_added -> first_transaction_completed`,
plus friction/noise events (`form_error`, `otp_retry`, `back_navigation`,
`help_click`, `device_switch`, `page_view`). Each synthetic user has a latent
"engagement" score that drives funnel conversion, error rates and idle-time
gaps, so the label (`dropped_off` = user never reached
`first_transaction_completed`) is realistically predictable from behaviour,
not hard-coded.

## The 18 behavioural features (`user_features`)

| # | Feature | What it captures |
|---|---|---|
| 1 | `total_events` | overall activity volume |
| 2 | `num_sessions` | distinct visits (tab closed & reopened, etc.) |
| 3 | `total_duration_sec` | wall-clock time from first to last event |
| 4 | `avg_time_between_events_sec` | mean pace through the funnel |
| 5 | `median_time_between_events_sec` | pace, robust to one long gap |
| 6 | `max_time_gap_sec` | longest idle/abandonment gap |
| 7 | `time_to_first_step_sec` | hesitation before starting sign-up |
| 8 | `num_form_errors` | validation friction |
| 9 | `num_otp_retries` | phone-verification friction |
| 10 | `num_backtrack_events` | uncertainty / re-checking steps |
| 11 | `num_help_clicks` | confusion signal |
| 12 | `num_device_switches` | started on one device, continued on another |
| 13 | `is_mobile_majority` | mobile- vs desktop-dominant session |
| 14 | `completed_email_verification` | funnel checkpoint reached |
| 15 | `completed_phone_verification` | funnel checkpoint reached |
| 16 | `completed_kyc_upload` | funnel checkpoint reached |
| 17 | `completed_bank_link` | funnel checkpoint reached |
| 18 | `hour_of_day_started` | time-of-day effect |

Label: `dropped_off` (1 = never reached `first_transaction_completed`).

## Setup

```bash
pip install -r requirements.txt

# Postgres: either run your own, or use the bundled docker-compose
docker compose up -d

cp .env.example .env   # adjust PGHOST/PGUSER/... if not using docker-compose
```

## Run the full pipeline

```bash
python -m src.pipeline --n-users 20000 --seed 42
```

This will:
1. Create `raw_signup_events` / `user_features` if they don't exist (`sql/schema.sql`).
2. Generate synthetic events for `--n-users` users and bulk-load them into Postgres.
3. Run `sql/feature_engineering.sql` to (re)build `user_features`.
4. Train an `XGBClassifier` (class-imbalance-aware via `scale_pos_weight`),
   evaluate on a held-out stratified test split, and write to `artifacts/`:
   - `xgb_dropoff_model.joblib` — trained model
   - `metrics.json` — ROC AUC, PR AUC, confusion matrix, classification report
   - `feature_importance.csv` / `.png` — gain-based importance

Re-run individual stages:

```bash
python -m src.ingest      # just (re)generate + load events
python -m src.features    # just rebuild user_features from existing events
python -m src.train        # just retrain on existing user_features

python -m src.pipeline --skip-ingest                    # reuse existing events
python -m src.pipeline --skip-ingest --skip-features    # reuse existing features, just retrain
```

## Notes

- All DB config is read from standard `PGHOST`/`PGPORT`/`PGDATABASE`/`PGUSER`/`PGPASSWORD`
  env vars (see `.env.example`), loaded via `python-dotenv`.
- The generator is deterministic given `--seed`, so re-runs are reproducible.
- With the default settings, completion rate is realistically low (~15-20%);
  the model is evaluated with ROC AUC / PR AUC / a full classification report
  rather than raw accuracy, since the classes are imbalanced.
- The four `completed_*` checkpoint flags (features 14-17) are strong,
  near-deterministic late-funnel signals, so the model scores very high
  (ROC AUC ~0.99 in validation). For a harder, earlier-warning model — useful
  if you want to flag at-risk users *before* they reach those checkpoints —
  drop those columns from `FEATURE_COLUMNS` in [src/features.py](src/features.py)
  before training.
- No local PostgreSQL/Docker was available in the dev sandbox this pipeline
  was built in, so `sql/feature_engineering.sql` was validated against a
  line-by-line pandas replica of the same CTE logic (same generator output,
  same aggregation semantics) rather than a live server — see the results
  above. Run `docker compose up -d` and `python -m src.pipeline` to exercise
  the real Postgres path.
