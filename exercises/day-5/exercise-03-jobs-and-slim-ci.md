# Exercise 3 — Production Jobs, Slim CI & Exposures

**Time: ~90 minutes**

## Goal
Configure dbt Cloud for production: a scheduled daily job, a Slim CI job that runs only modified models on every pull request, and Exposures to document downstream consumers.

---

## Part A — Configure Environments

In dbt Cloud you need two environments — development (already set up on Day 1) and production.

**Create the Production environment:**

Go to **Deploy → Environments → New Environment** and fill in:

| Field | Value |
|-------|-------|
| Name | `Production` |
| Environment type | `Deployment` |
| dbt version | same as Development |
| Default schema | leave blank (comes from `dbt_project.yml`) |
| Credentials | `TRANSFORMER` role, `DBT_WH` |

---

## Part B — Daily Production Job

Go to **Deploy → Jobs → Create Job**:

**Job name:** `Production — Daily Build`  
**Environment:** Production  
**Commands:**
```
dbt build
dbt source freshness
```

**Schedule:** `0 6 * * *` (every day at 06:00 UTC)  
**Advanced:** tick *Generate docs on run*

Save, then click **Run Now** to test it manually.

After it finishes:
- Click the run to inspect the run log — which models ran, in which order, how long each took
- Open the **Artifacts** tab — download `run_results.json` and find the slowest model
- Open the automatically generated docs site from the **Artifacts** tab

---

## Part C — Slim CI Job

Slim CI runs **only the models that changed** in a pull request instead of rebuilding the whole project. This makes CI fast enough to be useful.

Create a second job:

**Job name:** `CI — Pull Request Check`  
**Environment:** Production  
**Commands:**
```
dbt build --select state:modified+ --defer --state ./target
```
**Triggers:** tick *Run on pull request*

What each flag does:
- `state:modified+` — models that changed in this branch + all their downstream dependents
- `--defer` — for models *not* in scope, borrow results from the last successful production run instead of re-running them
- `--state ./target` — path to the production manifest to diff against

**Test it:**

```bash
# In your local repo or dbt Cloud IDE terminal
git checkout -b feat/add-order-year
```

Open `models/gold/gold_orders.sql` in the IDE, add `YEAR(order_date) AS order_year`, commit, and open a Pull Request on GitHub.

Watch the CI job trigger in **Deploy → Jobs**. Answer:
- Which models ran? Which were deferred?
- How long did it take vs a full `dbt build`?

---

## Part D — Exposures

Exposures document what consumes your dbt models — dashboards, reports, ML models. They close the lineage graph from raw source all the way to end consumers.

Create `models/exposures.yml`:

```yaml
version: 2

exposures:

  - name: orders_dashboard
    label: "Orders & Revenue Dashboard"
    type: dashboard
    maturity: high
    url: https://your-bi-tool.com/dashboards/orders
    description: >
      Main revenue operations dashboard. Shows daily order volume,
      revenue by channel, and fulfilment status. Used by Operations
      and Finance every morning.
    depends_on:
      - ref('gold_orders')
      - ref('gold_customers')
    owner:
      name: "Data Team"
      email: "data@example.com"

  - name: customer_360_report
    label: "Customer 360 Report"
    type: analysis
    maturity: medium
    description: >
      Customer lifetime value and segmentation. Feeds the CRM system
      for marketing automation.
    depends_on:
      - ref('gold_customers')
    owner:
      name: "Marketing Analytics"
      email: "marketing-analytics@example.com"

  - name: procurement_dashboard
    label: "Supplier & Parts Analytics"
    type: dashboard
    maturity: low
    description: >
      Supplier revenue ranking and parts performance. Built in the capstone.
    depends_on:
      - ref('gold_supplier_revenue')
      - ref('gold_part_performance')
      - ref('gold_supplier_monthly_revenue')
    owner:
      name: "Procurement Team"
      email: "procurement@example.com"
```

Commit and run:

```bash
dbt docs generate
```

In the dbt Cloud docs site, verify:
- Exposures appear as terminal nodes in the DAG
- Clicking an exposure shows owner, description, and linked models
- `dbt run --select +exposure:orders_dashboard` runs only the models that feed it — try `+exposure:procurement_dashboard` too

---

## Part E — Notifications

In **Account Settings → Notifications**, configure email notifications for job failures. Set to notify on failure only — success notifications create noise and get ignored.

---

## ✅ Success Criteria

- [ ] Production environment configured in dbt Cloud
- [ ] Daily production job runs successfully (green)
- [ ] CI job triggers on a pull request and runs only modified models + downstream
- [ ] `exposures.yml` with 3 exposures — they appear in the DAG
- [ ] `dbt run --select +exposure:orders_dashboard` runs only the relevant subgraph
- [ ] Email notification configured for failures
