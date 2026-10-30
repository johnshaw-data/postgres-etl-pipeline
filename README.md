# Multi-Site Sales ETL Pipeline (SQL Server → PostgreSQL)

An incremental ETL pipeline that pulls **new sales orders** from three store-level transactional databases (SQL Server), reconciles customers against a **central PostgreSQL repository**, loads new orders, inserts or updates customer records, and verifies the load with SQL reporting queries.

> **Note:** The source and target databases were hosted for a university data-engineering course and are no longer accessible, so the notebook is shown **with the saved outputs from the original run** — every step, extract through verification. Credentials have been replaced with environment variables.

## 🏗️ Architecture

```
 SQL Server (transactional)          PostgreSQL (central repository)
 ┌──────────┐ ┌──────────┐ ┌──────────┐        ┌──────────────────────┐
 │ pup_sl2  │ │ pup_bo2  │ │ pup_ph2  │        │ Site · SalesPerson   │
 │  (SL)    │ │  (BO)    │ │  (PH)    │        │ SalesOrder · OLine   │
 └────┬─────┘ └────┬─────┘ └────┬─────┘        │ Customer             │
      └────────────┼────────────┘              └──────────┬───────────┘
                   │  ① high-water mark per site (MAX TxID) │
                   │◄───────────────────────────────────────┘
                   ▼
        ② EXTRACT only orders with OrderID > MAX TxID
                   ▼
        ③ TRANSFORM: consolidate sites, split names, match customers
                   ▼
        ④ LOAD: insert orders + order lines; insert new / update existing customers
                   ▼
        ⑤ VERIFY: customers touched today · revenue & order count per site
```

## ⚙️ How it works
1. **High-water mark** — For each site, join `Site → SalesPerson → SalesOrder → OLine` to find the last transaction already loaded (`MAX(TxID)`).
2. **Incremental extract** — Connect to each site's SQL Server database and pull only orders newer than that mark, so nothing is loaded twice.
3. **Transform** — Combine all three sites into one DataFrame, tag each row with its site, split customer names, and match against the central `Customer` table on name plus phone/email.
4. **Load** — Insert new sales orders and their order lines; **update** matching customers and **insert** new ones, stamping `addedstamp` / `updatedstamp` / `TxIngestTimestamp`.
5. **Verify** — SQL reports on the central repository:
   - every customer added or updated in today's load, with their new central order ID
   - total revenue and order count per site for today's load, highest revenue first

## ✅ Results
| Stage | Result |
|---|---|
| High-water marks | SL = 47, BO = 25, PH = 28 |
| Orders extracted | **11** across 3 sites |
| Order lines loaded | **25** |
| Customers | **5 updated**, **6 inserted** |
| New central order IDs | 31109 – 31119 |

**Verification — revenue loaded per site:**

| Site | Revenue | Orders |
|---|---|---|
| Salt Lake City (SL) | $81,800.15 | 5 |
| Phoenix (PH) | $63,384.80 | 4 |
| Boise (BO) | $26,097.00 | 2 |
| **Total** | **$171,281.95** | **11** |

## 🐛 The bug worth talking about: two clocks
After the first load, the verification queries came back **empty** even though the data was clearly there.

**Diagnosis:** a debug query grouping customers by load date showed the rows stamped *2025-11-14*, while the server's `CURRENT_DATE` was already *2025-11-15*. The load step stamped rows with Python's `datetime.now()` — **local Mountain time** — but PostgreSQL runs in **UTC**. An evening load landed on "yesterday" from the server's point of view.

**Fix:** take the timestamp from the database (`SELECT CURRENT_TIMESTAMP`) so loading and querying share one clock. Hardcoding the date would have made the query pass once and break the next day.

**Cleanup:** before reloading, the bad run was removed in foreign-key order — order lines → sales orders → new customers — and the row counts (25 / 11 / 6) were checked against the extract to confirm nothing extra was deleted.

## 🐛 Other bugs fixed along the way
- **`KeyError: 'CustName'`** — the DataFrame keys didn't match the column casing SQL Server actually returned; the load uses the real (lowercase) column names.
- **NumPy types rejected by psycopg2** — pandas yields `numpy.int64` / `float64`, which psycopg2 can't insert by default; registered type adapters fix it.
- **`column so.siteid does not exist`** — orders don't store a site directly; the site is reached through the salesperson (`site → salesperson → salesorder`).
- **Case-sensitive identifiers** — mixed-case PostgreSQL columns like `"TxIngestTimestamp"` must be double-quoted.

## 🛠️ Improvements I'd make today
- Store timestamps as `timestamptz` and use the server clock everywhere
- Wrap the whole batch in **one transaction** (currently commits per order) so a failure can't leave a half-loaded run
- Make the load **idempotent** (upserts / `ON CONFLICT`) so a rerun can't create duplicates and needs no manual cleanup
- **SQLAlchemy** engines and **parameterized queries** instead of f-strings
- One set-based `GROUP BY` instead of a per-site query loop
- Schedule it with an orchestrator (e.g. Airflow) and alert when verification counts don't match the extract

## 🔐 Configuration
Copy `.env.example` to `.env` and fill in connection details. `.env` is git-ignored.

## 🧰 Stack
Python · pandas · psycopg2 (PostgreSQL) · pymssql (SQL Server) · SQL
