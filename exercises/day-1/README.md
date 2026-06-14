# Day 1 — Setup & First Models

**Estimated time: ~4 hours (afternoon exercises) · Full day including morning theory: ~7 hours**

Welcome to the dbt course. Today you will set up your entire environment from scratch and build a complete bronze layer — no data loading required.

## Learning objectives
- Create and configure a Snowflake environment for dbt
- Connect dbt Cloud to Snowflake and GitHub
- Understand the dbt project structure and configuration files
- Write and run your first SQL models against TPC-H data
- Declare sources and replace hardcoded table paths with `{{ source() }}`

## Exercises

| Exercise | Topic | Time |
|---|---|---|
| [01 — Snowflake Setup](exercise-01-snowflake-setup.md) | Warehouse, database, schemas, user, roles | ~15 min |
| [02 — GitHub Setup](exercise-02-github-setup.md) | Create GitHub account and course repository | ~15 min |
| [03 — dbt Cloud Setup](exercise-03-dbt-cloud-setup.md) | Create project, connect Snowflake, link GitHub | ~45 min |
| [04 — First Models](exercise-04-first-model.md) | Bronze models over TPC-H, project config, schema macro | ~90 min |
| [05 — Sources](exercise-05-sources.md) | Declare TPC-H as a source, complete the bronze layer | ~60 min |

## Datasets

| Dataset | Where | How it loads |
|---------|-------|-------------|
| **TPC-H** | `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1` | Already in Snowflake — no loading needed |
| **customers_seed** | `ANALYTICS.RAW` | Introduced in Day 3 via `dbt seed` |

---

## The TPC-H Dataset

TPC-H is a standard benchmark dataset that simulates a wholesale supplier business. It comes built into every Snowflake account — no setup needed. The scenario: a company buys parts from suppliers and sells orders to customers across different countries.

### Tables

**ORDERS** — 1,500,000 rows

The order header. One row per order placed by a customer.

| Column | Description |
|--------|-------------|
| `O_ORDERKEY` | Primary key |
| `O_CUSTKEY` | Foreign key to CUSTOMER |
| `O_ORDERSTATUS` | `O` = open, `F` = fulfilled, `P` = processing |
| `O_TOTALPRICE` | Total order value in USD |
| `O_ORDERDATE` | Date the order was placed |
| `O_ORDERPRIORITY` | `1-URGENT`, `2-HIGH`, `3-MEDIUM`, `4-NOT SPECIFIED`, `5-LOW` |
| `O_SHIPPRIORITY` | Always 0 in SF1 |

---

**LINEITEM** — 6,001,215 rows

The order lines. Each order has between 1 and 7 line items — one per part ordered. Think of it like the individual products inside a shopping cart: the order is the cart, each line item is one product.

| Column | Description |
|--------|-------------|
| `L_ORDERKEY` | Foreign key to ORDERS |
| `L_PARTKEY` | Foreign key to PART |
| `L_SUPPKEY` | Foreign key to SUPPLIER |
| `L_LINENUMBER` | Line number within the order (1–7) |
| `L_QUANTITY` | Units ordered |
| `L_EXTENDEDPRICE` | Gross line value (`quantity × unit_price`) |
| `L_DISCOUNT` | Discount applied (0.00–0.10) |
| `L_TAX` | Tax rate (0.00–0.08) |
| `L_RETURNFLAG` | `R` = returned, `A` = accepted, `N` = none |
| `L_LINESTATUS` | `O` = open, `F` = fulfilled |
| `L_SHIPDATE` | Date shipped |
| `L_COMMITDATE` | Committed delivery date |
| `L_RECEIPTDATE` | Date received by customer |
| `L_SHIPMODE` | `TRUCK`, `AIR`, `RAIL`, `SHIP`, `REG AIR`, `FOB`, `MAIL` |

Net revenue per line = `L_EXTENDEDPRICE * (1 - L_DISCOUNT)`

---

**CUSTOMER** — 150,000 rows

Customer dimension. One row per customer account.

| Column | Description |
|--------|-------------|
| `C_CUSTKEY` | Primary key |
| `C_NAME` | Customer name (e.g. `Customer#000000001`) |
| `C_ADDRESS` | Street address |
| `C_NATIONKEY` | Foreign key to NATION |
| `C_PHONE` | Phone number |
| `C_ACCTBAL` | Account balance in USD |
| `C_MKTSEGMENT` | `AUTOMOBILE`, `BUILDING`, `FURNITURE`, `MACHINERY`, `HOUSEHOLD` |

---

**NATION** — 25 rows

Country reference table.

| Column | Description |
|--------|-------------|
| `N_NATIONKEY` | Primary key |
| `N_NAME` | Country name (e.g. `ARGENTINA`, `GERMANY`) |
| `N_REGIONKEY` | Foreign key to REGION |

---

**SUPPLIER** — 10,000 rows

Supplier dimension. Used in Day 4 capstone models.

| Column | Description |
|--------|-------------|
| `S_SUPPKEY` | Primary key |
| `S_NAME` | Supplier name |
| `S_NATIONKEY` | Foreign key to NATION |
| `S_ACCTBAL` | Account balance |
| `S_PHONE` | Phone number |

---

**PART** — 200,000 rows

Parts catalog. Used in Day 4 capstone models.

| Column | Description |
|--------|-------------|
| `P_PARTKEY` | Primary key |
| `P_NAME` | Part name (e.g. `cornflower chocolate smoke ghost thistle`) |
| `P_MFGR` | Manufacturer (`Manufacturer#1` through `#5`) |
| `P_BRAND` | Brand (`Brand#11` through `#55`) |
| `P_TYPE` | Material and finish (e.g. `STANDARD ANODIZED BRASS`) |
| `P_SIZE` | Size (1–50) |
| `P_RETAILPRICE` | List price |

---

### How the tables relate

```
NATION (25)
  ├── CUSTOMER (150K) ──── ORDERS (1.5M) ──── LINEITEM (6M)
  └── SUPPLIER (10K)  ──────────────────────────────┘
                                  PART (200K) ─────────┘
```

In this course you'll mainly work with ORDERS, LINEITEM, CUSTOMER, and NATION. SUPPLIER and PART appear in the Day 4 capstone exercises.
