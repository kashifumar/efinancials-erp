# EFinancials — Enterprise Financial Management & HRM System

> A production-grade, multi-tenant ERP platform built with Laravel 8, serving multiple companies and regional offices with full financial accounting, human resource management, and CRM capabilities.

---

## Table of Contents

- [Business Problem](#business-problem)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Modules & Features](#modules--features)
- [Database Schema](#database-schema)
- [API Reference](#api-reference)
- [Key Engineering Challenges](#key-engineering-challenges)
- [Project Stats](#project-stats)
- [Local Setup](#local-setup)
- [Author](#author)

---

## Business Problem

Mid-to-large enterprises operating across multiple countries and office locations face a common set of problems:

- **Fragmented data** — finance, HR, and CRM data live in separate tools with no shared context.
- **No multi-tenancy** — off-the-shelf software cannot isolate data cleanly per company, per office, and per financial year.
- **Compliance risk** — manual voucher management, no audit trail, and inconsistent currency handling create regulatory exposure.
- **Reporting gaps** — executives cannot get consolidated reports across offices or drill down into per-office financials without exporting spreadsheets.

**EFinancials** solves all of this in a single, self-hosted platform:

- All transactions (25+ voucher types) are double-entry and audit-logged
- Every user session is scoped to a specific company → office → financial year
- A built-in REST API powers companion mobile apps used in the field
- Consolidated multi-office reports give management a single source of truth

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | PHP 8 / Laravel 8.12 |
| **Auth** | Laravel Jetstream + Fortify (2FA, session-based) |
| **API Auth** | Laravel Sanctum (token-based) |
| **Database** | MySQL / MariaDB |
| **ORM** | Eloquent (soft deletes throughout) |
| **Queue / Events** | Laravel Queue (database driver) |
| **Realtime** | Laravel Broadcasting + Pusher |
| **Frontend** | CoreUI, Tailwind CSS v2, Alpine.js v2, Livewire v2 |
| **UI Components** | jQuery, Select2, DataTables, Chart.js, Zebra DatePicker |
| **PDF Generation** | mPDF v8 |
| **Excel Import/Export** | PhpSpreadsheet v1.16 + Maatwebsite Excel |
| **Word Documents** | PhpWord v0.18 |
| **Precise Decimals** | brick/math v0.9 (eliminates float rounding on financial figures) |
| **File Storage** | Local (`storage/`, `uploads/`) + AWS S3 (per-file `is_s3` flag) |
| **Monitoring** | Laravel Telescope v4.4 |
| **HTTP Client** | Guzzle v7 |
| **Email Integration** | Webklex Laravel IMAP |

---

## Architecture

### Multi-Tenancy Model

```
Company
  └─ Office (branch / country office)
       └─ Financial Year
            └─ User Session
```

Each web request passes through the `SetDatabaseConnection` middleware, which resolves the correct DB connection from the session. The `ValidateOffice` middleware ensures a valid office context is set before any controller logic runs. All repository queries are scoped by `officeId` — cross-tenant data leakage is impossible at the query level.

### Repository Pattern

Controllers are intentionally thin. All business logic and database interaction lives in **130+ Repository classes** under `app/Repository/`. This means:

- Controllers hold zero raw queries
- The same repository method can be called from both a web controller and an API controller
- Business rules (voucher posting conditions, salary calculations, leave balance checks) are tested and changed in one place

```
app/
  Http/
    Controllers/Finance/Vouchers/Sales.php   ← thin: validate, call repo, return view
  Repository/Finance/Vouchers/
    SalesRepository.php                      ← all business logic lives here
```

### Middleware-Driven RBAC

Access control is implemented as **120+ granular middleware classes** (e.g. `SalesAuth`, `BankReceiptAuth`, `EmployeesAuth`). Each middleware checks a specific permission bit stored in the DB before allowing the request through. Permission changes are DB updates — no redeployment needed.

### Voucher Lifecycle (State Machine)

Every financial transaction follows a strict state machine:

```
Draft  →  Posted  →  (Revoke)  →  Draft
```

Posting locks the voucher and writes to the General Ledger. Revoking requires a separate permission and produces an audit entry. `LedgerTrack` and `InvoiceAdjustmentTrack` models provide a full immutable change history per voucher.

---

## Modules & Features

### Finance Module

The core of the system — a complete double-entry accounting engine.

**Chart of Accounts (COA)**
- Hierarchical tree structure with parent/child account codes
- Supports analysis codes for granular expense classification
- Each office maintains its own isolated COA

**25+ Voucher Types**

| Voucher | Code | Description |
|---|---|---|
| Sales Invoice | SI | Customer sales with line items |
| Sales Return | SR | Reversal of a sales invoice |
| Purchase Invoice | PI | Supplier purchases |
| Cash Receipt | CR | Cash received from customers |
| Batch Cash Receipt | BCR | Bulk cash receipt entry |
| Cash Payment | CP | Cash paid to suppliers / expenses |
| Bank Receipt | BR | Bank-based incoming payments |
| Bank Payment | BP | Bank-based outgoing payments |
| Journal Voucher | JV | Manual double-entry journals |
| Credit Note | CN | Credit adjustments |
| Debit Note | DN | Debit adjustments |
| Tele Transfer | TT | Inter-bank wire transfers |
| Inter-Currency | IC | Multi-currency revaluation |
| Party Inter-Currency | PIC | Customer / supplier FX adjustments |
| Other Sales | OS | Secondary sales channel invoices |
| Other Purchases | OP | Non-inventory purchase invoices |
| Electronics Sales | ES | Electronics product invoices |
| Attendance-Based Sales | ATS | Billable attendance invoices |
| Tyre Sales | TS | Tyre product invoices |
| Tyre Quotation | TQ | Tyre sales quotations |
| Costing Voucher | CV | Cost allocation entries |
| Day-Book Expenses | DBE | Daily petty-cash expense tracking |

**Financial Reports (15+)**

- Day Book (summary / detail / daily drill-down)
- General Ledger
- Trial Balance
- Subsidiary Ledgers (per COA account, date range)
- Bank Statement (per bank account)
- Sales Report (by customer, product, date range)
- Sales Aging / Ageing Detail (30/60/90-day buckets)
- Customer Statement
- Other Sales Reports (fitting, general)
- Code Analysis Report
- Day-Book Expense Report
- Consolidated Report (multi-office roll-up)
- Profit & Loss

**Other Finance Capabilities**
- Multi-currency with historical exchange rate tracking
- Bank reconciliation
- Receivables / Payables tracking
- File attachments on vouchers (local + S3)
- Batch post / revoke
- ZIP bundle download of voucher sets
- Office-specific branded PDF print templates

---

### HRM Module

A full HR lifecycle system — from onboarding to payroll.

**Employee Master**
- Personal, contact, and emergency contact information
- Document management (national ID, visa, contracts) with S3 support
- Assets, licenses, skills, and language proficiencies
- Digital signature storage
- Activation / de-activation workflow with supervisor approval

**Attendance**
- Daily attendance marking (present / absent / half-day / leave)
- Project-based attendance for billable time tracking
- Biometric ID linking
- Shift management with configurable duty start/end times

**Leave Management**
- Leave request → supervisor approval workflow
- Leave balance tracking per type (annual, sick, emergency)
- Gratuity conversion for accumulated leave

**Payroll**
- Allowances and deductions configuration per employee
- Monthly payroll register generation
- Salary payment vouchers integrated directly with the Finance module
- Provident Fund (PF) opening balance and ongoing tracking

**Appraisals** — Performance review cycle management

**HR Master Data (35+ types)**
Departments, Designations, Grades, Shifts, Sponsors, Visa Types, Relationships, Skills, Languages, Banks, Employee Types, and more.

---

### CRM Module

- **Projects** — active project tracking with location and status
- **Sales Proposals / Quotations** — versioned proposals with remarks, status tracking, and terms & conditions
- **Prospects** — lead and prospecting pipeline management
- **Sales Teams** — team structure and assignment

---

### Product Module

- Product categories and variants
- Store-level inventory with stock tracking
- Unit types and conversion factors
- Product assembly / bill of materials (BOM)
- Product purchase management

---

### Admin Module

- User management (create, assign roles, assign offices)
- Role and permission configuration
- API Manager — mobile app authentication and device registration
- User device tracking (mobile apps)

---

### SuperAdmin Module

- Company and Office setup
- Financial Year configuration
- Global RBAC permission definitions
- DB-driven navigation menus (configurable without code changes)
- Voucher type management
- Analysis code management
- Error / Query / Request log viewer (Telescope + custom audit models)

---

### Closing Module

- Period-end expense head configuration
- Closing period financial reports

---

## Database Schema

The schema spans **55 migrations** covering the full business domain. Key design decisions:

**Tenant isolation**: Every major table carries an `officeId` foreign key indexed for fast per-office queries. No cross-office data is returned without explicitly requesting it.

**Soft deletes everywhere**: All entities use Laravel's `SoftDeletes` — nothing is ever hard-deleted, preserving a full historical record.

**Audit columns**: Every table carries `createdBy` and `deletedBy` integer columns alongside `created_at` / `updated_at` timestamps.

**Financial precision**: Monetary values are stored as `bigInteger` in the smallest currency unit. Formatting and conversion happen only at the presentation layer.

**Reporting indexes**: Heavy-query tables (ledger, sale, employees) have composite indexes on `(officeId, status)`, `(officeId, voucherId)`, etc.

### Core Table Groups

```
── Tenancy
   companies, offices, financial_years

── Users & Access
   users, roles, role_permissions, user_offices

── Chart of Accounts
   chart_of_accounts (code, title, parentId, isPostable, officeId)

── Vouchers & Ledger
   ledger             -- voucher header (type, date, officeId, status)
   ledger_details     -- line items (debit/credit per COA account)
   ledger_files       -- attachments (local / S3)
   ledger_track       -- immutable change audit trail

── Sales & Purchases
   sale, sale_details
   purchase, purchase_details
   other_sale, other_sale_details
   payment, receivable, payable

── Multi-Currency
   currencies (code, exchange_rate, date)

── HR
   employees          -- 40+ columns: personal, payroll, biometric, status
   designations, departments, shifts, grades
   employee_attendance
   payroll_register, allowances, deductions

── Products
   products, product_images
   product_unit_types, product_unit_types_conversions
   product_sale, product_purchase
   product_sale_details, product_purchase_details

── CRM
   projects, quotations, terms_and_conditions
   prospects, prospecting, sales_teams

── System Audit
   error_logs, query_logs, request_logs
```

---

## API Reference

The REST API is consumed by companion mobile applications. All endpoints except `/api/login/` require a Bearer token issued by Sanctum.

### Authentication

```http
POST   /api/login/
```

### Session / Context

```http
POST   /api/set-cco                   Set active company / office for session
POST   /api/get-cco                   Get current company / office context
POST   /api/get-menus                 Retrieve navigation menu for current user
POST   /api/user-companies-offices    List all accessible companies & offices
GET    /api/get-dashboard-report      Dashboard KPI summary
```

### Finance — Vouchers

```http
POST   /api/get-vouchers              List vouchers (paginated + filtered)
GET    /api/voucher/{id}              Single voucher detail
POST   /api/post-voucher              Post or revoke a voucher
POST   /api/save-cash-receipt         Create cash receipt
POST   /api/update-cash-receipt       Update cash receipt
POST   /api/save-purchase             Create purchase invoice
POST   /api/update-purchase           Update purchase invoice
POST   /api/save-sales                Create sales invoice
POST   /api/update-sales              Update sales invoice
POST   /api/save-journal-voucher      Create journal entry
POST   /api/save-bank-payment         Create bank payment
POST   /api/save-tele-transfer        Create tele-transfer
POST   /api/save-cash-payment         Create cash payment
POST   /api/delete-voucher            Soft-delete a voucher
```

### Finance — Reports

```http
POST   /api/get-day-book              Day book report
POST   /api/get-trial-balance         Trial balance
POST   /api/get-general-ledgers       General ledger
POST   /api/get-subsidiary-ledgers    Subsidiary ledger per account
POST   /api/get-bank-statement        Bank statement
POST   /api/get-sales-report          Sales report (filterable)
```

### HRM — Employees

```http
GET    /api/employees-list            List all employees
GET    /api/employee/{id}             Employee full profile
POST   /api/save-employee             Create employee record
POST   /api/update-employee           Update employee record
POST   /api/delete-employee           Soft-delete employee
POST   /api/approve-employee          Approve employee record
POST   /api/save-employee-salary      Update salary information
POST   /api/save-employee-placement   Update placement details
POST   /api/save-employee-personal    Update personal info
POST   /api/save-employee-contact     Update contact info
POST   /api/save-employee-asset       Add / update asset record
```

### HRM — Attendance

```http
POST   /api/save-attendance           Mark daily attendance
POST   /api/update-attendance         Update attendance record
POST   /api/delete-attendance         Delete attendance record
POST   /api/get-attendance            List attendance records
POST   /api/save-leave                Submit leave request
POST   /api/approve-leave             Approve leave request
```

---

## Key Engineering Challenges

### 1. Precise Financial Arithmetic

**Problem**: PHP's native `float` type produces compounding rounding errors that cause trial balances to drift from zero, especially in multi-currency environments.

**Solution**: All monetary calculations use [brick/math](https://github.com/brick/math) — an arbitrary-precision decimal library. Values are stored as `bigInteger` in the smallest currency unit and only formatted at the presentation layer via `FinanceHelper`. This guarantees that `Σ debits = Σ credits` holds exactly, not approximately.

---

### 2. Multi-Currency Historical Reconciliation

**Problem**: Transactions are recorded in any supported currency. Year-end and period reports must reconcile everything to the base currency using the exchange rate *at the time of the transaction*, not the current spot rate.

**Solution**: The `currencies` table stores dated exchange rate records. Every voucher captures its `exchangeRate` at entry time. Reports apply the stored per-voucher rate via `brick/math` before aggregation — so closing a financial year produces accurate, auditable figures regardless of subsequent FX movements.

---

### 3. Row-Level Tenant Isolation Without Separate Databases

**Problem**: Dozens of offices needed strict data isolation without the operational overhead of per-tenant database instances.

**Solution**: `SetDatabaseConnection` and `ValidateOffice` middleware enforce session context on every request. All repository methods require an explicit `officeId` parameter — there is no way to call a data-access method without scoping it. MySQL composite indexes on `officeId` ensure per-office queries remain fast even on shared, high-row-count tables.

---

### 4. Voucher Posting Integrity Under Concurrent Requests

**Problem**: A user double-clicking the "Post" button — or two concurrent sessions — could post the same voucher twice, corrupting the General Ledger.

**Solution**: The post/revoke endpoint wraps the status check and update in a DB transaction with a row-level `SELECT ... FOR UPDATE` lock. If the status has already changed by the time the lock is acquired, the second request is rejected with a conflict error. The status write is atomic within the same transaction.

---

### 5. Regional Theming and Report Variants

**Problem**: Offices in different regions had different regulatory requirements for printed reports and different corporate branding guidelines.

**Solution**: The layout system checks the session's `officeId` against a configured list of regional offices at render time and switches CSS variables (gold/yellow palette vs. default blue) and Blade template paths accordingly. Report controllers resolve the correct template path from `officeId`, keeping regional variants isolated without duplicating any controller logic.

---

### 6. Shared Business Logic Across Web and Mobile API

**Problem**: The same business rules needed to power both a Blade/session web application and a stateless mobile REST API without duplication.

**Solution**: Repository classes are completely HTTP-agnostic. Web controllers and API controllers call the same repository methods. The only difference is the outermost layer: web controllers return `view()` responses; API controllers return `response()->json()`. Authentication is handled by separate middleware stacks (`web` vs. `my_api` / `sanctum`) without touching shared logic.

---

## Project Stats

| Metric | Count |
|---|---|
| Laravel Controllers | 108 |
| Repository Classes | 130+ |
| Eloquent Models | 170+ |
| Middleware Classes | 120+ |
| Database Migrations | 55 |
| Blade Templates | 450+ |
| Web Routes | 350+ |
| API Routes | 80+ |
| Voucher Types Supported | 25+ |
| Financial Report Types | 15+ |
| HR Master Data Types | 35+ |

---

## Local Setup

```bash
# 1. Clone the repository
git clone https://github.com/your-username/EFinancials.git
cd EFinancials

# 2. Install PHP dependencies
composer install

# 3. Install Node dependencies and compile assets
npm install && npm run dev

# 4. Configure environment
cp .env.example .env
php artisan key:generate
# Edit .env — configure DB_*, MAIL_*, and AWS_* values

# 5. Run migrations
php artisan migrate

# 6. Clear caches
php artisan config:clear && php artisan cache:clear && php artisan view:clear

# 7. Start development server
php artisan serve
```

**Requirements**: PHP 8.0+, MySQL 5.7+ / MariaDB 10.3+, Node.js 14+, Composer 2

---

## Author

**Kashif Umar** — Full-Stack & Backend Engineer

Specializing in enterprise Laravel applications, multi-tenant SaaS architecture, REST API design, and financial systems.

---

> *This repository is a portfolio showcase. Sensitive configuration, credentials, and proprietary business data have been removed. The codebase demonstrates architecture patterns, module design, and engineering decisions made on a real production system.*
