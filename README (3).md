# IPS_Survey – Foreign Tourists Experience module (dummy-data mode)

PostgreSQL schema `ips_survey` (the SOW's "IPS_Survey"; Postgres folds unquoted names to lower case).
**Main table: `survey_submission`** – one row per completed interview; every response table hangs off its `submission_id`.
Built from the two files you supplied (questionnaire workbook + `pilot_data_for_delhi.xlsx`) and tested on PostgreSQL 16 with
both the pilot data (23 module rows) and a 1,000-row synthetic set.

## SOW §6 deliverables → files

| SOW deliverable | File |
|---|---|
| Cleaned & populated PostgreSQL schema | `01_ddl_ips_survey.sql`, `02_seed_reference_data.sql`, `03_load_pilot_data.sql` |
| 1,000 synthetic records | `data/dummy_export_clean_1000.csv` (made by `06_generate_dummy_data.py`) |
| ER diagram | `ERD_IPS_Survey.png` / `.svg` (from the live catalog by `07_make_erd.py`), `ERD_IPS_Survey.mmd` (Mermaid) |
| Verified SQL library (DDL, indexes, analytical queries) | `01` (DDL + indexes), `04_views_and_analytics.sql`, `05_data_quality_checks.sql` |
| Analytical summary | KPI views in `04`; findings in the chat summary / section 6 below |

## 1. Layout of the normalised tables

```
survey_submission  ◄── MAIN TABLE   PK submission_id            (raw_payload jsonb kept for audit)
 │
 ├── FK sample_id ───────► survey_sample ──► investigator, supervisor, ref_form_type
 │                              └──< sample_shift                     (morning / afternoon / night)
 ├── FK place_id ────────► tourist_place                              (site master: ASI / NON_ASI)
 │
 ├──  1:1  respondent ─────────► ref_country (passport, residence), ref_occupation, ref_currency
 ├──  1:1  visit_profile ──────► ref_visit_purpose
 │             └──< travel_group_member ─► ref_age_band               (gender × age band head-counts)
 ├──  1:1  visa_experience ────► ref_visa_type
 ├──  1:N  visited_place ──────► tourist_place, ref_visit_reason, ref_information_source
 │             └── 1:1  tourist_place_experience                      (8 site ratings, 1-5)
 ├──  1:1  accommodation_experience ─► ref_accommodation_type, ref_hotel_category, ref_dissatisfaction_reason
 ├──  1:N  tourist_interest_ranking ─► ref_tourist_interest_type      (rank 1-3)
 ├──  1:N  tourism_service_rating ───► ref_tourism_service, ref_affordability   (13 services)
 ├──  1:N  payment_mode_usage ───────► ref_payment_category           (5 spend categories)
 └──  1:N  submission_section_time                                    (seconds per questionnaire section)

etl_rejected_row = ETL quarantine (no FK by design)
```
`──<` = one-to-many, `──►` = many-to-one lookup. 34 tables, 34 foreign keys (see ERD for every column, PK/FK and nullability).

| Table | Grain (one row = …) | PK | Parent(s) | Questionnaire |
|---|---|---|---|---|
| survey_submission | one completed interview | submission_id | survey_sample, tourist_place | Block 1 (100-103) |
| survey_sample | one assignment / quota | sample_id | investigator, supervisor, ref_form_type | – |
| respondent | respondent of a submission | submission_id (=FK) | survey_submission, ref_country ×2, ref_occupation, ref_currency | 104-105, 201-203 |
| visit_profile | trip facts | submission_id (=FK) | survey_submission, ref_visit_purpose | 301-303 |
| travel_group_member | head-count per gender × age band | (submission_id, gender, age_band_id) | visit_profile, ref_age_band | 304-311 |
| visa_experience | visa journey | submission_id (=FK) | survey_submission, ref_visa_type | 401-405 |
| visited_place | place visited ("+ Add place") | visited_place_id | survey_submission, tourist_place, ref_visit_reason, ref_information_source | 510-513 |
| tourist_place_experience | 8 site ratings of a visited place | visited_place_id (=FK) | visited_place | 520-527 |
| accommodation_experience | stay experience | submission_id (=FK) | survey_submission, ref_accommodation_type, ref_hotel_category, ref_dissatisfaction_reason | 601-607 |
| tourist_interest_ranking | rank 1-3 attraction type | (submission_id, interest_rank) | survey_submission, ref_tourist_interest_type | 608 |
| tourism_service_rating | importance / satisfaction / affordability of one service | (submission_id, service_id) | survey_submission, ref_tourism_service, ref_affordability | 710-723 |
| payment_mode_usage | payment modes for one spend category | (submission_id, payment_category_id) | survey_submission, ref_payment_category | 801-805 |

## 2. Relationship diagram (Mermaid – paste into mermaid.live / GitHub / VS Code)

See `ERD_IPS_Survey_relations.mmd` (relationships only) and `ERD_IPS_Survey.mmd` (with columns).

## 3. How to run (pgAdmin 4 Query Tool or psql)

```bash
createdb ips_survey_db                                   # or any empty database
psql -d ips_survey_db -f 01_ddl_ips_survey.sql           # schema, domain, constraints, indexes
psql -d ips_survey_db -f 02_seed_reference_data.sql      # SOW lookups + 16-site master

# choose ONE data source, name it survey_export_clean.csv, and run psql from that folder:
cp data/dummy_export_clean_1000.csv survey_export_clean.csv                         # SOW: 1,000 synthetic rows
#   python 00_prepare_csv.py pilot_data_for_delhi.xlsx survey_export_clean.csv      # or: your real pilot file
psql -d ips_survey_db -f 03_load_pilot_data.sql          # staging -> normalised tables (re-runnable)
psql -d ips_survey_db -f 04_views_and_analytics.sql      # KPI views + 11 analytical queries
psql -d ips_survey_db -f 05_data_quality_checks.sql      # 15 PASS/REVIEW checks
```
pgAdmin: run 01, 02, 04, 05 in the Query Tool. For 03, run STEP 1 (staging table), import the CSV with *Tools → Import/Export Data*
into `ips_survey.stg_pilot_export` (header on), then run the rest of the file from STEP 2 (remove the `\copy` line).

## 4. Design decisions

* **Hub-and-spoke**: `survey_submission` is the only table other than reference data that owns the interview identity; 1:1 child tables
  reuse `submission_id` as PK+FK (shared primary key), 1:N children have composite or surrogate keys.
* **Sample ≠ submission**: the export repeats `sample_id` (e.g. sample 334 has many submissions), so assignment data
  (investigator, supervisor, window, target) lives in `survey_sample` and `survey_submission.sample_id` is a FK. The stored,
  fast-going-stale `submission_received` counter is dropped; `v_sample_progress` computes it.
* **First normal form**: JSON objects/arrays and the `{morning,night}` array are exploded into rows (`travel_group_member`,
  `sample_shift`, `submission_section_time`, …). The untouched JSON stays in `survey_submission.raw_payload` for audit / re-parse.
* **Duplicate export columns collapsed**: `sample_created_datetime` ×3 (timestamp/date/time), `submitted_date/on/at`
  (timestamp/date/time) and `sample_id-2` (copy of `sample_id`) become single columns.
* **Integrity in the database, not in Excel**: FKs everywhere, `rating_1_5` domain, CHECKs on age, gender, income currency,
  windows and coordinates. Negative tests (rating 6, orphan FK, income without currency, delete of a referenced sample) are rejected.
* **Rule-breaking rows are quarantined** (`etl_rejected_row`) instead of silently dropped or crashing the load.

## 5. Things to confirm with the survey team (assumptions I had to make)

1. **`service_id` 1-13 = questionnaire items 710-718 then 720-723**, by payload order (seeded in `ref_tourism_service`).
2. **Labels for country, currency, occupation, visa type, purpose, source, accommodation, hotel category, interest type are not in the files**, so they load as `Unmapped … #id`. Overwrite from the app master (UPDATE template at the end of `03`).
3. `ref_affordability.cost_group`: older build says `Cheaper`, new build `Economical` – I grouped both as LOW_COST (assumption).
4. `survey_submission.place_id` = first place in `visited_place_details` (the payload has no separate "site surveyed" field); the single
   `tourist_place_experience` object is attached to that first place.
5. Only form type 1 (Foreign Tourists Experience) is loaded, as the SOW scopes. The other three forms (expenditure, outbound, NRI)
   reuse `survey_sample` / `survey_submission` / `payment_mode_usage`; they need extra child tables (trip cost, package, gifts, visa list) and, for airport forms, `port` / `airport_terminal` keys.

## 6. What the pilot data revealed (run `05_data_quality_checks.sql`)

* **17 of 23 experience submissions have `submitted_at` earlier than the sample's creation time** (29 of 64 in the whole export) – device-clock or time-zone issue.
* **4 submissions (ids 313-316) come from an older app build** (`app_version` NULL): no age/gender, no `tourist_interest_ranking`,
  `cost_reasonableness` instead of `affordability`, `online_booking` but no `net_banking`. The ETL handles both shapes.
* **Submission 313 ranks the same attraction type twice** (rank 1 and 2) → quarantined.
* **`matched_expectation` is stored as `false` (not NULL) when Q603 = No** (7 rows) → the ETL applies the questionnaire skip logic (Q604 only if Q603 = Yes).
* **Income outliers**: 1.5 and 100,000,000 in a stated currency; `visa_process_rating` present with no `processing_days` (4 rows).
* The app records importance + satisfaction + affordability for **all** 13 services, although the questionnaire says some items are "satisfaction only" / "affordability only".
* **The SOW asks for NPS and expenditure distributions, but this module has no 0-10 recommend question and no spend amounts.**
  `v_site_kpis.nps_proxy` is derived from the 1-5 "Overall" rating (5 = promoter, 4 = passive, ≤3 = detractor) and must be labelled a proxy.
  Likewise "length of stay" is only approximated as *days between arrival and survey date* (no departure date is captured).
