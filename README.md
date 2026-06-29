# 📚 Public Library Consortium — Database Management System

A 3-member backend database project built entirely in on **Microsoft SQL Server Management Studio (SSMS)** — no application or GUI layer. The system models a small library consortium's day-to-day operations: cataloguing books, tracking physical copies, registering members, and recording every checkout and return.

---

## 👥 Team & Work Division

| Member | Owns | Focus Area |
|---|---|---|
| **[M.Waqar Altaf** | `Books`, `Book_Copies` | Schema design, constraints, CRUD, JOINs |
| **Moarij Irfan** | `Members` | Stored procedures, transactions |
| **Muhammad Sami Ullah** | `Book_Loans` | Views, subqueries, aggregates, final integration |

> Work was split by table ownership, not arbitrary tasks — Members 1 & 2 had zero dependencies on each other and worked in parallel; Member 3's table depends on both, so that work started last and Member 3 owned the final merge.

---

## 🗂️ Database Schema

```
Books (1) ──▶ Book_Copies (Many) ──▶ Book_Loans (Many) ◀── Members (1)
```

| Table | Purpose |
|---|---|
| `Books` | Catalogue of titles — ISBN, title, author, genre, year |
| `Book_Copies` | Each physical copy of a book, with condition & availability |
| `Members` | Registered library members and their status |
| `Book_Loans` | Checkout / return transaction records |

---

## ⚙️ Tech Stack

- **Database Engine:** Microsoft SQL Server (Express)
- **Client Tool:** SQL Server Management Studio (SSMS)
- **Language:** Pure T-SQL — no front-end/application layer
- **Database Name:** `PublicLibraryConsortium`

---

## ✅ Features Implemented

- 🔐 **Constraints** — PRIMARY KEY, FOREIGN KEY (`ON DELETE NO ACTION`), UNIQUE, CHECK
- 📝 **CRUD Operations** — INSERT, SELECT, UPDATE, DELETE on all tables
- 🔗 **JOINs** — INNER JOIN and LEFT JOIN across all 4 tables
- 🔍 **Subqueries** — including one correlated subquery
- 📊 **Aggregates** — GROUP BY / HAVING for genre and circulation stats
- 🔄 **Stored Procedures** — `sp_CheckoutBookCopy`, `sp_ReturnBookCopy`, wrapped in transactions with COMMIT/ROLLBACK
- 👁️ **Views** — `vw_OverdueLoanFulfillment`, `vw_CatalogCirculationSummary`
- 🧪 **Constraint & Logic Testing** — duplicate ISBN, FK-blocked delete, suspended-member checkout, already-returned loan

---


## 🚀 How to Run This Project

1. Open **SSMS** and connect to your SQL Server instance.
2. Open `sql/04_master_script.sql`.
3. Execute the entire script — it creates the database, all 4 tables with constraints, 120 rows of seed data, both stored procedures, and both views, in the correct dependency order.
4. Explore individual features using `01_`, `02_`, and `03_` files, or run:
   ```sql
   EXEC sp_CheckoutBookCopy @MemberID = 1, @CopyID = 5;
   SELECT * FROM vw_OverdueLoanFulfillment;
   ```

---

## 🧱 Normalization

The schema is normalized to **3NF**:
- **1NF** — atomic columns, surrogate primary keys on every table
- **2NF** — single-column keys mean no partial dependencies are possible
- **3NF** — book/member attributes are stored once and referenced by foreign key elsewhere, preventing update anomalies

---

## 🧩 Problems Faced & Resolved

| Issue | Fix |
|---|---|
| ISBN string truncation on insert | Used properly-sized, real-format ISBN values |
| FK violation when inserting Book_Copies | Reset IDENTITY counters with `DBCC CHECKIDENT` |
| Loan count showed 31 instead of 30 | Confirmed it was a genuine successful checkout test, not a bug |

*(Full write-up in the project report under `docs/`.)*

---

## 📄 Full Report & Presentation

- 📘 [Project Report (Word)] [PublicLibraryConsortium_Report(119)(190)(1130).docx](https://github.com/user-attachments/files/29463406/PublicLibraryConsortium_Report.119.190.1130.docx)

- 📊 [Presentation Slides (PowerPoint)]

---[PublicLibraryConsortium_Presentation(119)(190)(1130).pptx](https://github.com/user-attachments/files/29463428/PublicLibraryConsortium_Presentation.119.190.1130.pptx)


## 🧪 Testing Summary

All 16 required test cases — constraint violations, JOIN correctness, procedure success/failure paths, view output, subqueries, and the data-consistency check — were executed and **passed**. See the full results table in the project report.
