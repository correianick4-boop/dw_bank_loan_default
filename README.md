# Bank Loan Default Prediction & Portfolio Analysis — Data Warehouse

A MySQL data warehouse built from two public loan datasets (Lending Club + German
Credit), with OLAP queries and a Python mining pipeline (classification + clustering)
on top, feeding into a Power BI dashboard.

> **Security note:** every command below uses `YOUR_PASSWORD` as a placeholder.
> Replace it with your own MySQL root password locally — never commit a real
> password into a script, a `.env` file, or this README.

---

## 1. Prerequisites

| Tool | Notes |
|---|---|
| **MySQL Community Server 9.7 (LTS)** | [dev.mysql.com/downloads/mysql](https://dev.mysql.com/downloads/mysql) — **MSI Installer**, not ZIP |
| **VSCode** | any recent version |
| **VSCode extension: "MySQL"** by Weijan Chen (`cweijan.vscode-mysql-client2`) | ⚠️ Not Microsoft's "SQL Server (mssql)" extension — that's for SQL Server, a different database engine, and will never connect here |
| **VSCode extension: "Python"** (`ms-python.python`) | for the mining scripts |
| **Python 3.10+** | [python.org/downloads](https://python.org/downloads) |
| **Git + Git LFS** | datasets are large and tracked via LFS |

---

## 2. Install MySQL Server

1. Run the MySQL Installer → **Server only** (or Developer Default if you also want Workbench).
2. **Type and Networking**: Config Type = `Development Computer`, Port = `3306`, TCP/IP checked, firewall ports open.
3. Set a **root password** — remember it.
4. Finish as a Windows Service. Sakila/World sample DBs optional, not needed here.

---

## 3. Set up the VSCode MySQL extension

1. Install `cweijan.vscode-mysql-client2`.
2. Sidebar database icon → **New Connection**.
3. Host `localhost`, Port `3306`, Username `root`, Password `YOUR_PASSWORD`.
4. Save → Connect.

---

## 4. Clone the repository

```powershell
git clone <YOUR_REPO_URL> DWproject
cd DWproject
git lfs pull
```

---

## 5. Datasets

If `git lfs pull` didn't bring the raw CSVs, download directly:

| Dataset | Source |
|---|---|
| Lending Club Loan Data | [kaggle.com/datasets/wordsforthewise/lending-club](https://www.kaggle.com/datasets/wordsforthewise/lending-club) |
| German Credit Risk | [kaggle.com/datasets/uciml/german-credit](https://www.kaggle.com/datasets/uciml/german-credit) |

⚠️ Extracting the Lending Club zip can create a folder with the same name as the
CSV inside it. Check your actual path with `Get-ChildItem` and adjust
`load_staging_data.sql` if it differs from section 8 below.

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
│   ├── 02_warehouse/
│   │   ├── create_dimensions.sql
│   │   └── create_fact_loan.sql
│   ├── 03_etl/
│   │   ├── load_dim_source_system.sql
│   │   ├── load_dim_time.sql
│   │   ├── load_dim_customer.sql
│   │   ├── load_dim_loan_product.sql
│   │   ├── load_dim_geography.sql
│   │   ├── load_dim_credit_history.sql
│   │   └── load_fact_loan.sql
│   └── 04_olap/
│       ├── rollup_default_by_region.sql
│       ├── drilldown_time.sql
│       ├── slice_grade_c.sql
│       ├── dice_multi_filter.sql
│       └── pivot_purpose_year.sql
│
├── python/
│   ├── requirements.txt
│   ├── .env.example
│   ├── .env                                   (you create this - gitignored)
│   ├── db_connection.py
│   ├── classification_default_prediction.py
│   ├── clustering_portfolio_segmentation.py
│   └── outputs/                               (created automatically when scripts run)
│
├── venv/                                      (created by python -m venv venv - gitignored)
├── .gitignore
└── README.md
```

---

## 7. One-time MySQL terminal setup

```powershell
cd DWproject
$env:Path += ";C:\Program Files\MySQL\MySQL Server 9.7\bin"   # every new terminal session
mysql -u root -pYOUR_PASSWORD -e "CREATE DATABASE bank_loan_dw;"
mysql -u root -pYOUR_PASSWORD -e "SET GLOBAL local_infile = 1;"  # every MySQL service restart
```

---

## 8. Run the SQL scripts, in this exact order

```powershell
# Schema
Get-Content sql/01_staging/create_staging_tables.sql | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/02_warehouse/create_dimensions.sql   | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw
Get-Content sql/02_warehouse/create_fact_loan.sql    | mysql --local-infile=1 -u root -pYOUR_PASSWORD bank_loan_dw

# Extraction (Lending Club is ~2.3M rows - takes a few minutes)
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

OLAP queries (`sql/04_olap/*.sql`) are read-only and can be run anytime after this, in any order, the same way.

---

## 9. Python mining environment

```powershell
cd DWproject
python -m venv venv
venv\Scripts\Activate.ps1
# If blocked: Set-ExecutionPolicy -Scope CurrentUser RemoteSigned, then retry

pip install -r python\requirements.txt
```

In VSCode: `Ctrl+Shift+P` → "Python: Select Interpreter" → pick the `venv` one.

Create `python\.env` (do not hand-type it — a stray space or quote breaks the
connection and is hard to spot; generate it directly instead):

```powershell
@"
DB_USER=root
DB_PASSWORD=YOUR_PASSWORD
DB_HOST=localhost
DB_PORT=3306
DB_NAME=bank_loan_dw
"@ | Out-File -Encoding ascii python\.env
```

Add both generated folders to `.gitignore` so secrets and the environment never get committed:

```powershell
Add-Content .gitignore "`npython/.env`nvenv/"
```

Run the mining scripts (classification first — matches the project order, and
surfaces any connection problem immediately):

```powershell
cd python
..\venv\Scripts\Activate.ps1
python classification_default_prediction.py
python clustering_portfolio_segmentation.py
```

Both print metrics to the terminal, save charts to `python\outputs\`, and write
results back into MySQL as new tables (`mining_default_scores`,
`mining_portfolio_segments`) for Power BI to use later.

---

## 10. Verify

```sql
USE bank_loan_dw;
SELECT src.source_name, COUNT(*) AS loans
FROM fact_loan f JOIN dim_source_system src ON src.source_sk = f.source_sk
GROUP BY src.source_name;
```

Expect ~2.26M Lending Club rows, 1,000 German Credit rows.

---

## 11. Troubleshooting

| Symptom | Fix |
|---|---|
| `mysql` not recognized | PATH reset each new terminal — re-run the `$env:Path +=` line |
| `python` opens the wrong/missing file, or Windows Store stub error | venv isn't activated in this terminal — `cd` to the project, then `venv\Scripts\Activate.ps1` (prompt should show `(venv)`) |
| `ERROR 3948: Loading local data is disabled` | `SET GLOBAL local_infile = 1;` again — resets on every MySQL service restart |
| `Cannot drop table ... referenced by a foreign key` | `DROP TABLE IF EXISTS fact_loan;` first, recreate dimensions, then recreate `fact_loan` |
| `The '<' operator is reserved` in PowerShell | PowerShell doesn't support `<` redirection — use `Get-Content file.sql \| mysql ...` instead |
| `Access denied for user 'root'@'localhost' (using password: YES)` from Python | Usually a bad `.env` value - regenerate it with the `Out-File -Encoding ascii` command above rather than hand-editing. `db_connection.py` already uses SQLAlchemy's `URL.create()` so special characters like `@` in the password are handled safely - if you're still hitting this, the password value itself is wrong, not the encoding |
| Connecting via the mssql extension never works | That extension is for SQL Server, not MySQL - use `cweijan.vscode-mysql-client2` instead |

---

## Known data limitation

The German Credit CSV used here (`uciml/german-credit`) has **no target/Risk
column** and no employment length, income, or DTI fields. German Credit rows
load with `is_default = NULL`, are valid for OLAP and portfolio segmentation,
but are excluded from classifier training - only Lending Club rows carry a
usable default label.
