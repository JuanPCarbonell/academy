# Exercise 2 — dbt Projects on Snowflake

**Time: ~45 minutes**

## Goal
Connect your GitHub repository to Snowflake Workspaces and run your dbt models directly from the Snowsight IDE — without dbt Cloud and without leaving Snowflake.

---

## Part A — Prerequisites (SQL Setup)

Run as `ACCOUNTADMIN` to allow Snowflake to read your GitHub repository:

```sql
USE ROLE ACCOUNTADMIN;

-- Secret: stores your GitHub Personal Access Token
-- Generate at: github.com → Settings → Developer settings → Personal access tokens
-- Scopes needed: repo (read)
CREATE SECRET ANALYTICS.RAW.GITHUB_SECRET
    TYPE     = PASSWORD
    USERNAME = '<your_github_username>'
    PASSWORD = '<your_personal_access_token>';

-- API Integration: tells Snowflake which GitHub account to trust
CREATE OR REPLACE API INTEGRATION GITHUB_API_INTEGRATION
    API_PROVIDER                   = git_https_api
    API_ALLOWED_PREFIXES           = ('https://github.com/<your_github_username>')
    ALLOWED_AUTHENTICATION_SECRETS = (ANALYTICS.RAW.GITHUB_SECRET)
    ENABLED                        = TRUE;

-- Grant to TRANSFORMER role
GRANT USAGE ON INTEGRATION GITHUB_API_INTEGRATION TO ROLE TRANSFORMER;
GRANT READ  ON SECRET ANALYTICS.RAW.GITHUB_SECRET TO ROLE TRANSFORMER;
```

---


## Part B — Create a Workspace

1. In Snowsight, go to **Projects → Workspaces**.
2. Look for **From Git repository** in the left panel — if not visible, click **+ Add New** and select **From Git repository**.
3. Paste your repository URL: `https://github.com/<your_username>/dbt-academy`
4. Optionally rename the workspace to `dbt_academy_workspace`.
5. In the **API Integration** menu, select `GITHUB_API_INTEGRATION`.
6. Authentication method: select **Personal access token**.
   - Select the database and schema where the secret is stored: `ANALYTICS` → `RAW`
   - Select the secret: `GITHUB_SECRET`
7. Click **Create**.

Your project files (`dbt_project.yml`, `models/`, etc.) appear in the left panel.



## ✅ Success Criteria

- [ ] API Integration and Secret created and granted to `TRANSFORMER`
- [ ] `dbt_packages/` committed and pushed before connecting
- [ ] Workspace connected to GitHub — project files visible in Snowsight
- [ ] `dbt run --select gold_orders` executes successfully inside the workspace
