# Bank Loan Default Prediction & Portfolio Analysis — Data Warehouse

A MySQL data warehouse built from two public loan datasets (Lending Club + German
Credit), designed for OLAP analysis, data mining, and a Power BI dashboard.

> **Security note:** every command below uses `YOUR_PASSWORD` as a placeholder.
> Replace it with your own MySQL root password when you run it — never commit
> your real password into a script or into this file.

---

## 1. Prerequisites

| Tool | Notes |
|---|---|
| **MySQL Community Server 9.7 (LTS)** | [dev.mysql.com/downloads/mysql](https://dev.mysql.com/downloads/mysql) — download the **MSI Installer**, not the ZIP |
| **VSCode** | any recent version |
| **VSCode extension: "MySQL"** by Weijan Chen (`cweijan.vscode-mysql-client2`) | ⚠️ Do **not** use Microsoft's "SQL Server (mssql)" extension — that connects to SQL Server, not MySQL, and will not work here |
| **Git + Git LFS** | the datasets are large and tracked via LFS in this repo |

---

## 2. Install MySQL Server

1. Run the MySQL Installer. Choose **Server only** (or Developer Default if you also want Workbench — it's useful for visually inspecting the schema later).
2. On **Type and Networking**: keep Config Type = `Development Computer`, Port = `3306`, TCP/IP checked, "Open Windows Firewall ports" checked.
3. Set a **root password** when prompted — remember it.
4. Let it install as a Windows Service and finish the wizard. Optional: skip the Sakila/World sample databases, not needed for this project.

---

## 3. Set up the VSCode MySQL extension

1. Install `cweijan.vscode-mysql-client2` from the Extensions marketplace.
2. Open its database icon in the VSCode sidebar → **New Connection**.
3. Fill in: Host `localhost`, Port `3306`, Username `root`, Password `YOUR_PASSWORD`.
4. Save and Connect.

---

## 4. Clone the repository

```powershell
git clone <YOUR_REPO_URL> DWproject
cd DWproject
git lfs pull
```

`git lfs pull` is important — without it, the large CSVs in `datasets/` will be empty pointer files, not the actual data.

---

## 5. Datasets

If `git lfs pull` didn't bring in the raw CSVs (e.g. cloning to a machine without LFS fully configured), download them directly:

| Dataset | Source |
|---|---|
| Lending Club Loan Data | [kaggle.com/datasets/wordsforthewise/lending-club](https://www.kaggle.com/datasets/wordsforthewise/lending-club) |
| German Credit Risk | [kaggle.com/datasets/uciml/german-credit](https://www.kaggle.com/datasets/uciml/german-credit) |

⚠️ **Known quirk:** extracting the Lending Club zip can create a folder with the same name as the CSV inside it (e.g. `accepted_2007_to_2018q4.csv/` containing `accepted_2007_to_2018Q4.csv`). Check your actual extracted structure with `Get-ChildItem` and adjust the path in `load_staging_data.sql` if it differs from what's documented below.

---

## 6. Folder structure

```
DWproject/
├── datasets/
│   └── raw/
│       ├── dataset1/
│       │   └── accepted_2007_to_2018q4.csv/
│       │       └── accepted_2007_to_2018Q4.csv
│       └── dataset2/
│           └── german_credit_data.csv
│
├── sql/
│   ├── 01_staging/
│   │   ├── create_staging_tables.sql
│   │   └── load_staging_data.sql
│   │
│   ├── 02_warehouse/
│   │   ├── create_dimensions.sql
│   │   └── create_fact_loan.sql
│   │
│   └── 03_etl/
│       ├── load_dim_source_system.sql
│       ├── load_dim_time.sql
│       ├── load_dim_customer.sql
│       ├── load_dim_loan_product.sql
│       ├── load_dim_geography.sql
│       ├── load_dim_credit_history.sql
│       └── load_fact_loan.sql
│
└── README.md
```

---

## 7. One-time terminal setup

Open a terminal in VSCode at the repo root (`DWproject/`).

**Add MySQL to PATH for this session** (needed every time you open a fresh terminal — this doesn't persist):

```powershell
$env:Path += ";C:\Program Files\MySQL\MySQL Server 9.7\bin"
```

**Create the database:**

```powershell
mysql -u root -pYOUR_PASSWORD -e "CREATE DATABASE bank_loan_dw;"
```

**Enable local file loading** (needed once per server restart — required for the extraction step):

```powershell
mysql -u root -pYOUR_PASSWORD -e "SET GLOBAL local_infile = 1;"
```

---

## 8. Run the scripts, in this exact order

Schema first, then extraction, then transform/load. Run each line from the repo root.

```powershell
# Schema
Get-Content sql/01_staging/create_staging_tables.sql | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/02_warehouse/create_dimensions.sql   | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/02_warehouse/create_fact_loan.sql    | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw

# Extraction (loads the two CSVs into staging - the Lending Club file is
# ~2.3 million rows, this step takes a few minutes)
Get-Content sql/01_staging/load_staging_data.sql | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw

# Transform + load, dimensions before the fact table
Get-Content sql/03_etl/load_dim_source_system.sql  | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/03_etl/load_dim_time.sql            | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/03_etl/load_dim_customer.sql        | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/03_etl/load_dim_loan_product.sql    | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/03_etl/load_dim_geography.sql       | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/03_etl/load_dim_credit_history.sql  | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/03_etl/load_fact_loan.sql           | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
```

The last script prints a summary table (loans/defaults/unlabeled per source) — that's your confirmation everything loaded correctly.

---

## 9. Verify

```sql
USE bank_loan_dw;

SELECT * FROM fact_loan LIMIT 20;

SELECT
    src.source_name, dt.year, dp.loan_purpose, dg.country,
    f.loan_amount, f.is_default
FROM fact_loan f
JOIN dim_source_system src ON src.source_sk = f.source_sk
JOIN dim_time dt           ON dt.time_sk = f.time_sk
JOIN dim_loan_product dp   ON dp.product_sk = f.product_sk
JOIN dim_geography dg      ON dg.geography_sk = f.geography_sk
LIMIT 20;
```

Expect roughly 2.26M Lending Club rows (~268K marked as default) and 1,000 German
Credit rows (all `is_default = NULL` — that source has no target label, see note below).

---

## 10. Troubleshooting

| Symptom | Fix |
|---|---|
| `mysql` not recognized | PATH reset — re-run the `$env:Path +=` command from step 7, every new terminal session |
| `ERROR 3948: Loading local data is disabled` | Run `SET GLOBAL local_infile = 1;` again (step 7) — resets on every MySQL service restart |
| `Cannot drop table ... referenced by a foreign key` | You're dropping a dimension table while `fact_loan` still points to it. Always `DROP TABLE IF EXISTS fact_loan;` first, recreate dimensions, then recreate `fact_loan` |
| `Data too long for column` | A source value is longer than the column allows. Current schema is already patched for the known cases (e.g. `job_skill_band VARCHAR(30)`) |
| `LOAD DATA LOCAL INFILE` path error | Check your actual extracted CSV path with `Get-ChildItem` — the Lending Club zip's nested-folder quirk (see Datasets section) varies by extraction tool, and your drive letter may differ from `D:\` |

---

## Known data limitation

The German Credit CSV used here (`uciml/german-credit`) has **no target/Risk
column** and no employment length, income, or DTI fields. German Credit rows are
loaded with `is_default = NULL` and are valid for OLAP queries and portfolio
segmentation, but excluded from classifier training — only Lending Club rows
carry a usable default label.
