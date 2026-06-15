# Exercise 1 — Snowflake Setup

**Time: ~15 minutes**

## Goal
Create all the Snowflake objects dbt will need: a warehouse, a database, the required schemas, a dedicated role, and a service user. You won't touch dbt yet — that's Exercise 2.

---

## Part A — Create Snowflake Objects

Log in to your Snowflake account as `ACCOUNTADMIN` and run the following SQL in a worksheet. Read each block before executing.

```sql
-- Step 1: Virtual warehouse for dbt
CREATE WAREHOUSE IF NOT EXISTS DBT_WH
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'Warehouse used by dbt transformations';

-- Step 2: Main analytics database
CREATE DATABASE IF NOT EXISTS ANALYTICS;

-- Step 3: Schemas
-- Raw schema: where seed data lands
CREATE SCHEMA IF NOT EXISTS ANALYTICS.RAW;
-- Transformed schemas: where dbt models live
CREATE SCHEMA IF NOT EXISTS ANALYTICS.BRONZE;
CREATE SCHEMA IF NOT EXISTS ANALYTICS.SILVER;
CREATE SCHEMA IF NOT EXISTS ANALYTICS.GOLD;
CREATE SCHEMA IF NOT EXISTS ANALYTICS.SNAPSHOTS;

-- Step 4: Role for dbt
CREATE ROLE IF NOT EXISTS TRANSFORMER;

GRANT USAGE ON WAREHOUSE DBT_WH TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON DATABASE ANALYTICS TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON FUTURE SCHEMAS IN DATABASE ANALYTICS TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON ALL TABLES IN DATABASE ANALYTICS TO ROLE TRANSFORMER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN DATABASE ANALYTICS TO ROLE TRANSFORMER;

-- Step 5: Grant TRANSFORMER to your own Snowflake user
-- Replace <YOUR_SNOWFLAKE_USERNAME> with the username you log in with
GRANT ROLE TRANSFORMER TO USER <YOUR_SNOWFLAKE_USERNAME>;

-- Step 6: Service user for dbt
-- Replace <YOUR_PASSWORD> with a strong password
CREATE USER IF NOT EXISTS DBT_USER
    PASSWORD = '<YOUR_PASSWORD>'
    DEFAULT_ROLE = TRANSFORMER
    DEFAULT_WAREHOUSE = DBT_WH
    MUST_CHANGE_PASSWORD = FALSE;

GRANT ROLE TRANSFORMER TO USER DBT_USER
```

> **Note:** In production you would use key-pair authentication instead of a password.

---

## Part B — Verify the Setup

Switch to the `TRANSFORMER` role and confirm everything is in place:

```sql
USE ROLE TRANSFORMER;
USE WAREHOUSE DBT_WH;

SHOW DATABASES;
-- Expected: ANALYTICS (plus Snowflake system databases)

SHOW SCHEMAS IN DATABASE ANALYTICS;
-- Expected: RAW, BRONZE, SILVER, GOLD, SNAPSHOTS

-- Confirm write access
CREATE TABLE ANALYTICS.BRONZE.CONNECTION_TEST (id INT);
DROP TABLE ANALYTICS.BRONZE.CONNECTION_TEST;
-- If both run without error, permissions are correct.
```

---

## Part C — Verify TPC-H Access

TPC-H is a large benchmark dataset built into every Snowflake account. No loading needed.

```sql
-- Confirm the sample data is accessible
SELECT COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;
-- Expected: 1,500,000

SELECT COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM;
-- Expected: 6,001,215
```

If these queries fail, ask your Snowflake admin to share the `SNOWFLAKE_SAMPLE_DATA` database with your account.

---

## Part D — Note Down Your Credentials

Before moving to Exercise 2 you will need:

| Value | Where to find it |
|-------|-----------------|
| **Account identifier** | Admin → Accounts → copy the locator (e.g. `xy12345.eu-west-1`) |
| **DBT_USER password** | The one you set in Step 5 above |
| **Warehouse name** | `DBT_WH` |
| **Database name** | `ANALYTICS` |

Keep these handy — you will enter them into the dbt Cloud connection wizard in Exercise 2.

---

## ✅ Success Criteria

- [ ] All 5 schemas exist in `ANALYTICS` (`RAW`, `BRONZE`, `SILVER`, `GOLD`, `SNAPSHOTS`)
- [ ] The connection test table creates and drops without errors
- [ ] `SELECT COUNT(*) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS` returns 1,500,000
- [ ] You have your Snowflake account identifier and `DBT_USER` password noted down
