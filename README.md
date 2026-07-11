# fintech-funnel-intelligence
Fintech Sign-Up Drop-Off Pipeline

End-to-end ETL + ML pipeline that predicts user drop-off in a fintech onboarding funnel — and demonstrates how to catch data leakage before it ships.

Synthetic event logs → PostgreSQL → 18 SQL-engineered behavioural features → XGBoost early-warning classifier.


The Problem

Fintech products lose most users silently — between onboarding steps, KYC checks, and first transactions. Product teams need to know where users drop off and, more importantly, who is about to drop off early enough to intervene (a push notification, a simplified step, proactive support).

This project builds that capability end to end:

Synthetic Event Logs (20,000 user journeys, 13-step funnel)
        │
        ▼
ETL  (Python + SQLAlchemy → PostgreSQL, batched inserts, idempotent schema)
        │
        ▼
Feature Engineering  (18 behavioural features, built in pure SQL —
        │             window functions, PERCENTILE_CONT, FILTER clauses)
        ▼
XGBoost Classifier  (drop-off prediction + leakage analysis)
        │
        ▼
Artifacts  (model, metrics.json, feature importance plot)


The Key Finding: Data Leakage

The first model scored 0.99 ROC AUC. That's not a win — that's a red flag.

What was leaking

FeatureWhy it leakstotal_eventsCompleters touch all 13 funnel steps (~13+ events); droppers have ~1–5. Directly encodes funnel depth = the label.total_duration_secCompleters always spend more total time.num_sessionsCompleters are more likely to have returned across sessions.completed_email_verificationPost-hoc checkpoint flag (step 4 of 13).completed_phone_verificationPost-hoc checkpoint flag (step 7 of 13).completed_kyc_uploadPost-hoc checkpoint flag (step 9 of 13).completed_bank_linkPost-hoc checkpoint flag (step 11 of 13).

These features are only knowable after the user's outcome is largely determined. A model built on them describes the past — it can't warn you about the future.

The honest model

After removing all 7 leaky features, the early-warning model uses only 11 upstream friction signals that are measurable from the very first events of a session:


num_form_errors, num_otp_retries — friction
num_backtrack_events, num_help_clicks — hesitation & confusion
num_device_switches, is_mobile_majority — context
avg / median time between events, max_time_gap_sec, time_to_first_step_sec — pace & intent
hour_of_day_started — temporal signal


Results

ModelFeaturesROC AUCVerdictNaive (post-hoc)18 (incl. funnel-depth features)0.995❌ Rejected — data leakageEarly-warning11 pure behavioural signals0.912✅ Deployable at sign-up start

Early-warning PR AUC: 0.982


Honest caveat: 0.91 is still optimistic. The synthetic generator drives both friction signals and drop-off from the same latent engagement score, so the signal is cleaner than production data would be. On real user data I'd expect roughly 0.70–0.80 — still highly actionable for triggering interventions.




Repo Structure

fintech_signup_etl/
├── sql/
│   ├── schema.sql                # raw_signup_events + user_features tables, indexes
│   └── feature_engineering.sql   # 18 features built in SQL (idempotent UPSERT)
├── src/
│   ├── config.py                 # env-driven settings (.env supported)
│   ├── db.py                     # SQLAlchemy engine + DDL helpers
│   ├── data_generator.py         # latent-engagement funnel simulator
│   ├── ingest.py                 # batched ETL into Postgres
│   ├── features.py               # runs SQL feature job, loads feature frame
│   └── train.py                  # XGBoost training + evaluation + artifacts
├── artifacts/                    # model, metrics.json, feature importance
└── fintech_signup_dropoff_pipeline.ipynb   # fully self-contained Colab notebook


The Synthetic Data Generator

Most synthetic datasets hard-code the label into a feature. This one doesn't.

Each user gets a latent engagement score (mixture of two Beta distributions, shifted by acquisition channel). That single latent variable drives:


Step-to-step conversion via a logit model: advance_logit = logit(base_p) + 2.6·(engagement − 0.5) − 0.35·friction
Friction events — form errors, OTP retries, back-navigation (which accumulate friction that decays between steps)
Timing — lognormal inter-event gaps; low-engagement users have longer, heavier-tailed gaps and "walked away" pauses
Realistic context — device/OS/browser distributions, 24-hour traffic curve, 90-day event window, cross-session returns, device switches


The result: drop-off signal emerges from behaviour rather than being planted in a column — which is exactly what makes the leakage analysis meaningful.
