# Instacart Analytics — End-to-End Product Analytics Pipeline

A production-grade product analytics project built from scratch. The dataset simulates a grocery delivery platform at the behavioral level — not random rows, but users with propensity scores, sessions with realistic funnel drop-off, and orders driven by the same behavioral engine that determines abandonment.

Built to support rigorous funnel analysis, cohort retention, RFM segmentation, and marketing attribution on data that behaves like a real production database.

---

## Why this project exists

Most publicly available datasets are either too clean or too simple to practice real product analytics. This project builds the data infrastructure and behavioral simulation that a product analyst at a company like Instacart would actually work with — including the messiness, the drop-off, and the signals that are only findable through careful analysis.

---

## Stack

| Layer | Tool |
|---|---|
| Data generation | Python |
| Data warehouse | Google BigQuery |
| Transformation | dbt |
| Analysis | SQL |
| Documentation | Medium |

---

## Architecture

```
python_v2/
    config.py          # constants, seeds, sample mode flag
    context.py         # UserContext, SessionContext, BasketContext
    users.py           # user creation and behavioral parameters
    products.py        # product catalog
    stores.py          # stores and shoppers
    campaigns.py       # campaigns and promotions
    seasonality.py     # calendar and environmental multipliers
    multipliers.py     # 50 behavioral signals, weighted multiplier framework
    sessions.py        # session and event generation (clickstream)
    orders.py          # order, payment, item generation (transactional)
    main.py            # orchestration, CSV output

instacart_dbt/
    models/
        staging/       # one model per raw table, clean and typed
        intermediate/  # joined models with business logic
        marts/         # analysis-ready tables
```

The separation of `sessions.py` and `orders.py` mirrors real production architecture where clickstream and transactional data originate from different upstream systems. A session can terminate at any funnel stage without invoking the order generator.

---

## Dataset

| Table | Rows |
|---|---|
| users | 50,000 |
| sessions | 5,821,452 |
| events | 17,670,086 |
| orders | 412,596 |
| payments | 412,596 |
| order_items | 4,327,579 |
| products | 2,000 |
| stores | 80 |
| shoppers | 500 |
| campaigns | 20 |
| promotions | 30 |

---

## Behavioral simulation

Each user is assigned three latent parameters at creation:

- `order_propensity` — baseline likelihood to place orders over time
- `tenure_decay` — rate at which propensity declines with inactivity
- `basket_sensitivity` — price sensitivity, determines promo response strength

Every session runs through a weighted multiplier framework. Final conversion probability is a product of user, session, basket, and environmental multipliers — each bounded between 0.5x and 1.5x, with a floor of 0.01 and ceiling of 0.95.

Each session is assigned a target depth (0 to 5) before any events are generated. Depth determines where in the funnel the session terminates.

| Depth | Label | Terminal event |
|---|---|---|
| 0 | Bounce | app_open |
| 1 | Window shopper | store_selected |
| 2 | Researcher | product_viewed |
| 3 | Cart abandoner | add_to_cart |
| 4 | Checkout hesitator | checkout_started |
| 5 | Converted | order_placed |

---

## dbt transformation layer

14 models across three layers.

**Staging (7 models)** — one per raw table. Cleans, types, and adds derived flags. Business definitions live here: what counts as a delivered order, what counts as a promo order.

**Intermediate (2 models)** — joins staged tables and applies business logic. `int_order_details` joins orders, payments, and items. `int_user_orders` aggregates user order history.

**Marts (3 models)** — analysis-ready tables. `fct_orders`, `dim_users`, `fct_funnel_events`.

---

## Setup

### Prerequisites

- Python 3.10+
- Google Cloud account with BigQuery enabled
- dbt CLI installed (`pip install dbt-bigquery`)
- `gcloud` CLI authenticated

### Generate data

```bash
cd python_v2

# sample run — 1,000 users
# set SAMPLE_MODE = True in config.py
python main.py

# full run — 50,000 users
# set SAMPLE_MODE = False in config.py
python main.py
```

### Load to BigQuery

```bash
bq load --replace --source_format=CSV --autodetect instacart.events ~/instacart_analytics/data_v2/events.csv
bq load --replace --source_format=CSV --autodetect instacart.sessions ~/instacart_analytics/data_v2/sessions.csv
bq load --replace --source_format=CSV --autodetect instacart.orders ~/instacart_analytics/data_v2/orders.csv
bq load --replace --source_format=CSV --autodetect instacart.payments ~/instacart_analytics/data_v2/payments.csv
bq load --replace --source_format=CSV --autodetect instacart.order_items ~/instacart_analytics/data_v2/order_items.csv
bq load --replace --source_format=CSV --autodetect instacart.users ~/instacart_analytics/data_v2/users.csv
bq load --replace --source_format=CSV --autodetect instacart.products ~/instacart_analytics/data_v2/products.csv
bq load --replace --source_format=CSV --autodetect instacart.stores ~/instacart_analytics/data_v2/stores.csv
bq load --replace --source_format=CSV --autodetect instacart.shoppers ~/instacart_analytics/data_v2/shoppers.csv
bq load --replace --source_format=CSV --autodetect instacart.campaigns ~/instacart_analytics/data_v2/campaigns.csv
bq load --replace --source_format=CSV --autodetect instacart.promotions ~/instacart_analytics/data_v2/promotions.csv
```

### Run dbt

```bash
cd instacart_dbt
dbt run
```

---

## Project phases

| Phase | Focus | Status |
|---|---|---|
| 1 | Schema design | Complete |
| 2 | Data generation | Complete |
| 3 | Data quality audit | Complete |
| 4 | dbt transformation layer | Complete |
| 5 | Funnel analysis | In progress |
| 6 | Cohort retention | In progress |
| 7 | RFM segmentation | Planned |
| 8 | Marketing attribution | Planned |
| 9 | Dashboard | Planned |

---

## Key findings so far

**Funnel analysis (Phase 5)**

Overall session-to-order conversion: 7.1%. The largest absolute drop-off occurs between product_viewed and add_to_cart — 1.36 million sessions browse products without adding anything to cart.

Conversion by device: iOS 8.0%, web 7.3%, Android 5.6%. Android drop-off begins early in the funnel, not just at checkout.

Conversion by acquisition channel: referral 9.3%, organic 7.5%, Instagram 5.4%. Instagram has the lowest overall conversion but the highest checkout completion rate once users reach that stage — suggesting intent mismatch at the top of the funnel rather than friction at checkout.

97.2% of users who placed an order had at least one abandoned session before converting.

---

## Write-up

Full methodology, decision framework, and findings documented on Medium:

- Part 1: How I Built Synthetic Data That Does Not Lie
- Part 2: What Building Synthetic Data Taught Me About Real Data
- Part 3: Data Quality Audit
- Part 4: Building the dbt Transformation Layer
- Part 5: The Pivot — When a Well-Designed Generator Produces the Wrong Data

---

## Known limitations

Session duration does not vary meaningfully across segments — the generator assigns fixed random ranges rather than behavior-driven durations. This limits time-to-convert analysis.

Cohort retention curves are currently affected by a generator design issue where users do not churn. A churn mechanism is being added before Phase 6 findings are finalized.

The dataset does not include the top of the acquisition funnel — impressions, clicks, and installs. Analysis starts at app_open.

---

## License

MIT
