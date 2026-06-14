# Exercise 2 — dbt in Snowflake

**Time: ~90 minutes**

## Goal
Run dbt models natively inside Snowflake using the **dbt Snowflake Native App** — without dbt Cloud, without a local install, and without leaving the Snowflake UI. This is the newest way to run dbt and represents the direction of tight warehouse-native analytics tooling.

---

## Part A — What is the dbt Snowflake Native App?

dbt Labs and Snowflake have built a native integration that runs dbt Core directly inside your Snowflake account. It lives in Snowflake's **Native Apps** ecosystem and uses **Snowpark Container Services** to execute dbt in a managed environment.

Key differences vs dbt Cloud and dbt Core:

| | dbt Core (local) | dbt Cloud | dbt Native App (Snowflake) |
|---|---|---|---|
| Where it runs | Your laptop | dbt Cloud servers | Inside your Snowflake account |
| Auth | profiles.yml | UI connection | Snowflake role (already authenticated) |
| Git integration | CLI | GitHub/GitLab | GitHub or inline editor |
| Compute | Your machine | Cloud runner | Snowpark Container Services |
| Best for | Local dev | Team collaboration | Snowflake-native orgs |

---

## Part B — Install the Native App

1. In Snowflake, open **Data Products → Marketplace**.
2. Search for **dbt**.
3. Find **dbt Snowflake Native App** (by dbt Labs) and click **Get**.
4. Choose the `ANALYTICS` database and the `TRANSFORMER` role as the installation target.
5. Click **Get** to install. Installation takes 2–5 minutes.

Once installed, navigate to **Data Products → Apps** — you should see **dbt** listed.

---

## Part C — Create a dbt Project in the Native App

1. Open the dbt Native App.
2. Click **New Project** and name it `dbt_academy_native`.
3. Connect it to your GitHub repository (`dbt-academy`).
4. Configure the connection:
   - **Database**: `ANALYTICS`
   - **Schema** (dev): `dbt_native_<your_name>`
   - **Warehouse**: `DBT_WH`
   - **Role**: `TRANSFORMER`

The app reads your `dbt_project.yml` and `packages.yml` directly from GitHub — no separate configuration needed.

---

## Part D — Run Models from Snowflake

Inside the dbt Native App, open the **IDE** tab (similar to dbt Cloud IDE).

Run the full project:

```bash
dbt deps
dbt seed
dbt build
```

Check the output: models should appear in `ANALYTICS.DBT_NATIVE_<YOUR_NAME>_BRONZE`, etc.

**Question:** The dbt Native App ran inside Snowflake — no data left your Snowflake account. Why does this matter for organisations with strict data residency requirements?

---

## Part E — Compare the Three Environments

You've now run the same project in three different ways. Fill in this comparison from your own experience today:

| | dbt Cloud IDE | dbt Core (local) | dbt Native App |
|---|---|---|---|
| Setup time | | | |
| Where credentials live | | | |
| Where compiled SQL is visible | | | |
| How you update packages | | | |
| What happens when Snowflake is down | | | |

---

## Part F — Snowflake Notebooks + dbt (Advanced)

Snowflake Notebooks let you mix Python, SQL, and dbt in a single interactive environment. This is experimental but worth knowing about.

In Snowflake, open **Projects → Notebooks** and create a new notebook. In a Python cell:

```python
# This is experimental — available in Snowflake previews
# Check if your account has Notebooks enabled first
import subprocess
result = subprocess.run(
    ["dbt", "run", "--select", "bronze_tpch_customers", "--project-dir", "/path/to/project"],
    capture_output=True, text=True
)
print(result.stdout)
```

The practical use: combine a dbt run with downstream Python analysis (Pandas, plotting) in a single shareable notebook.

**Question:** When would a Notebook workflow be better than the Native App IDE? When worse?

---

## Part G — When to Use Each Environment

Based on everything you've seen today, answer these questions for a hypothetical team:

1. A small startup (2 engineers, Snowflake only, no other tools): which environment?
2. A large enterprise with strict data residency rules: which environment?
3. A team that wants analysts (non-engineers) to run models with a single click: which environment?
4. A team that needs to run dbt on a schedule every hour with alerting: which environment?

---

## ✅ Success Criteria

- [ ] dbt Native App is installed in your Snowflake account
- [ ] `dbt build` runs successfully inside the Native App
- [ ] Models appear in a `dbt_native_<your_name>` schema — separate from Cloud and local runs
- [ ] You completed the comparison table in Part E
- [ ] You can explain the data residency advantage of the Native App
