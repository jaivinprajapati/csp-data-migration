# Session Notes — Mistakes & Corrections

## Mistake (2026-06-13): Used the Excel "Master tables' list" as the column source instead of code

**What happened.** When building the Netbox/Devices workflow, I took the column lists for `nbrec_execution_candidates` and `netbox_custody` from the **"Master tables' list" sheet** in `CSP Migration - 10 June.xlsx` and treated them as the complete schema. They were not.

- `nbrec_execution_candidates`: sheet had **32 columns**; the real table (csp-tas-service Flyway migrations) has **49**. I missed ~17 columns added in later migrations — customer enrichment (`customer_name`, `google_address_id`, `customer_status`, `is_migrated_customer` — V047/V048), photo/OCR proof (`front_photo_url`, `back_photo_url`, `photo_capture_*` — V013), `cancellation_reason`/`failure_reason`/`cancellation_note` (V013/V038), `recovery_method` (V036), and the 5 task-assignment columns (`assignee_type`, `assigned_member_id`, `assigned_member_name`, `assigned_at`, `reassignment_count` — V054).
- `netbox_custody`: sheet had **26 columns**; the real table has **30**. Missed `was_ever_deployed`, `device_commissioned_at` (V14), `dispatched_at` (V17), `last_accrual_date` (V18).

**Why it happened (root cause).** For the ISP workflow I read the schema from code (base `CREATE TABLE` + every `ALTER`). For Netbox I shortcut to the spreadsheet because it already had recommended values — a **secondary, point-in-time artifact** built from an older migration snapshot. I also ran a `find` filtered by filename (`*nbrec*`/`*netbox*`/`*custody*`), which missed ALTER migrations whose filenames didn't contain those words (e.g. `V11__v1_10_v1_11_device_binding_and_recovery.sql`, `V14__sd_architecture_acs_v1_12.sql`, `V8__alter_csp_id_to_varchar.sql`).

**Fix applied this session.** Re-derived both tables' complete column sets from code: base `CREATE TABLE` + `grep -rl "<table_name>"` across the **entire** migration directory (not filename-filtered) + reconciled. Rebuilt both tables in the view with every column.

## Rule I will follow without fail, every table, every workflow

1. **Code is the only schema source.** Never take a column list from a spreadsheet, doc, or memory as complete. Spreadsheets/specs are inputs for *recommended values*, never for the *column set*.
2. **Enumerate columns mechanically:** base `CREATE TABLE` **plus** `grep -rl "<table>"` over the full migration dir (unfiltered by filename) to catch every `ALTER TABLE … ADD/DROP/ALTER COLUMN`, and cross-check the JPA entity. Account for DROP COLUMN too.
3. **Reconcile a column count** against code before mapping, and state it ("table = N columns, verified vs migrations").
4. **Never filter discovery by filename.** Grep by the entity/table name itself.

(Mirrors the CSP-migration-writeset checklist: "the migration IS the complete column set.")
