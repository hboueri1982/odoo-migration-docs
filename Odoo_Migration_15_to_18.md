# Odoo Data Migration (Odoo 15 → Odoo 18) Using OpenUpgrade

---

## 1. Overview

This document outlines the step-by-step procedure for migrating an Odoo database from **Odoo 15** to **Odoo 18** using the **OpenUpgrade** framework.

Since OpenUpgrade performs migrations on a per-major-version basis, the upgrade must be carried out sequentially across each intermediate version as follows:

> **Odoo 15 → Odoo 16 → Odoo 17 → Odoo 18**

**Notes:**
- OpenUpgrade script packages for versions **16.0**, **17.0**, and **18.0** have already been downloaded and placed in the `openUpgrade` folder within our repository.
- All steps and path examples in this document are based on a **Windows local development environment**.

---

## 2. Prerequisites

Ensure the following are in place before starting the migration:

- **Local Odoo source projects** available for all required versions:
  - a. Odoo 15 — *(specify local path)*
  - b. Odoo 16 — *(specify local path)*
  - c. Odoo 17 — *(specify local path)*
  - d. Odoo 18 — *(specify local path)*

- **OpenUpgrade packages** available in the repository:
  - `OpenUpgrade-16.0`
  - `OpenUpgrade-17.0`
  - `OpenUpgrade-18.0`

- **Odoo 15 database backup** including:
  - PostgreSQL database dump
  - Filestore (must correspond to the same database)

- **Python environment** correctly configured in VS Code *(see Section 7)*

---

## 3. Download and Prepare OpenUpgrade

1. Navigate to the official OCA OpenUpgrade repository:
   `https://github.com/OCA/OpenUpgrade`

2. Download the required version branches: **16.0**, **17.0**, and **18.0**.
   *(Already completed — packages are available in the repository.)*

3. Extract each downloaded zip file.
   *(Already completed.)*

4. Copy the extracted folders into the `openUpgrade` folder in the repository.
   *(Already completed.)*

5. Install the required Python library (`openupgradelib`) by running the following command in your terminal:

```bash
py -m pip install --ignore-installed git+https://github.com/OCA/openupgradelib.git@master
```

---

## 4. Configure the Odoo Addons Path

In your **Odoo configuration file**, add the OpenUpgrade folder path to the `addons_path` setting.

The path must point to the OpenUpgrade version corresponding to the current migration step. Adjust the path to match your local clone location.

**Example — Odoo 16:**

```
C:\workarea\SABIS059 - ODOO ERP\ODOO_ERP\openUpgrade\OpenUpgrade-16.0
```

> ⚠️ This path must be updated for each migration step (16, 17, and 18).

---

## 5. Backup, Restore, and Validate (Odoo 15)

1. Take a **full backup** of the Odoo 15 production database, including the **filestore**.

2. **Restore** the database and its filestore to your local Odoo 15 environment.

3. Start Odoo 15 and attempt to **log in** to verify that the restored database is functioning correctly.

4. Since the production environment may use identity-based (OAuth) login, which does not work in a local environment, you must **enable standard username/password authentication**. Run the following SQL query on the restored database:

```sql
UPDATE auth_oauth_provider SET enabled = true WHERE name = 'Odoo.com Accounts';
```

> ⚠️ **Important:** This setting must be **reverted** (`enabled = false`) after the migration to Odoo 18 is complete.

5. Log in using your locally registered email and password. If the password has been lost or was never set, reset it by updating the corresponding record in the `res_users` table using the following pre-encrypted password hash:

| Field | Value |
|---|---|
| **Plain-text password** | `admin` |
| **Encrypted hash** | `$pbkdf2-sha512$25000$HQPgHGMMAYAQAqDUmtN6bw$OKcjrNGSYEX/kZZdwTh./3IsNVXTEnq.yXLDJDNxodT4rPu9552r4zbQXH/1C6hPv11IwxZg1B0zT0vOHa9N5Q` |

```sql
UPDATE res_users 
SET password = '$pbkdf2-sha512$25000$HQPgHGMMAYAQAqDUmtN6bw$OKcjrNGSYEX/kZZdwTh./3IsNVXTEnq.yXLDJDNxodT4rPu9552r4zbQXH/1C6hPv11IwxZg1B0zT0vOHa9N5Q'
WHERE login = 'your_email@domain.com';
```

> ⚠️ **Important:** Once the migration is complete and all validations are done, reset this to a strong, secure password.

6. Once you have confirmed that the restored Odoo 15 database is working correctly, **stop Odoo 15** and switch to the **Odoo 16** workspace/project.

---

## 6. Pre-Migration Cleanup (SQL)

Before running any OpenUpgrade migration command, execute the following SQL script against the database to remove outdated XML view definitions that may cause conflicts during migration.

> ℹ️ This cleanup script must be re-applied **before each migration step** (15→16, 16→17, and 17→18).

```sql
DELETE FROM ir_ui_view WHERE model = 'account.account';
DELETE FROM ir_ui_view WHERE model = 'account.invoice.send';
DELETE FROM ir_ui_view WHERE name  = 'hr.payslip.form.inherited';
DELETE FROM ir_ui_view WHERE model = 'res.users';
DELETE FROM ir_ui_view WHERE model = 'hr.contract';
DELETE FROM ir_ui_view WHERE model = 'account.journal';
DELETE FROM ir_ui_view WHERE model = 'res.partner';
DELETE FROM ir_ui_view WHERE model = 'hr.employee';
DELETE FROM ir_ui_view WHERE name  = 'hr_payroll_payslip_employees';
DELETE FROM ir_ui_view WHERE model = 'hr.indemnity.calculation';
DELETE FROM ir_ui_view WHERE model = 'indemnity.calculation.employees.views';
DELETE FROM ir_ui_view WHERE model = 'hr.loan';
DELETE FROM ir_ui_view WHERE model = 'account.payment';
DELETE FROM ir_ui_view WHERE model = 'stock.quant';
DELETE FROM ir_ui_view WHERE model = 'account.payment.register';
DELETE FROM ir_ui_view WHERE model = 'account.move';
DELETE FROM ir_ui_view WHERE model = 'account.invoice.report';
DELETE FROM ir_ui_view WHERE model = 'budget.budget';
DELETE FROM ir_ui_view WHERE name  = 'transfer_transaction_status';
DELETE FROM ir_ui_view WHERE model = 'product.category';
DELETE FROM ir_ui_view WHERE model = 'product.product';
DELETE FROM ir_ui_view WHERE model = 'hr.payslip.run';

DELETE FROM ir_model_data
WHERE module = 'sabis_accounting'
  AND name   = 'sabis_accounting_stock_quant_tree_inventory_editable_inherit'
  AND model  = 'ir.ui.view';

DELETE FROM ir_ui_view
WHERE (arch_db::TEXT LIKE '%accounting_date%'
    OR arch_db::TEXT LIKE '%asset_category_id%');
```

---

## 7. Validate Your Python Environment

Before running any migration command, confirm that the terminal is using the correct Python interpreter.

1. Open the terminal using the correct workspace settings file:
   `workspace-hboueri.code-workspace`

2. Run the following commands to verify the active Python environment:

```bash
python -c "import sys,site; print(sys.executable); print(site.getusersitepackages())"
pip -V
```

3. Verify that:
   - `sys.executable` points to the expected Python installation
   - `pip -V` references the same Python environment

**If the wrong Python environment is selected**, add or update the following `settings` block in your workspace file, adjusting the path to match your Python installation:

```json
"settings": {
    "python.defaultInterpreterPath": "C:/Program Files/Python310/python.exe",
    "python.terminal.activateEnvironment": true,
    "terminal.integrated.env.windows": {
        "Path": "C:/Program Files/Python310;C:/Program Files/Python310/Scripts;${env:Path}"
    },
    "python.languageServer": "None",
    "xml.symbols.enabled": false
}
```

---

## 8. Migration Step 1: Odoo 15 → Odoo 16

### 8.1 Run the Migration Command

Open a terminal in the **Odoo 16** project folder and execute the following command.
Update the `--database` and `--upgrade-path` parameters to match your environment before running:

```bash
python odoo-bin -c odoo-hboueri.user.conf \
  --database=erp_538_v15_2026-03-31_09-21-46 \
  --upgrade-path="C:\workarea\SABIS059 - ODOO ERP\ODOO_ERP\openUpgrade\OpenUpgrade-16.0\openupgrade_scripts\scripts" \
  --load=base,web,openupgrade_framework \
  --update all \
  --stop-after-init
```

### 8.2 Confirm Successful Migration

The migration has completed successfully when you see the following messages at the end of the terminal output:

```
INFO ... odoo.modules.loading: Modules loaded.
INFO ... odoo.modules.registry: Registry loaded in X.XXXs
```

### 8.3 Post-Migration Validation

Configure the following modules in the **Odoo 16** workspace and start the project. This step regenerates all views, applies newly added fields, and confirms that no errors are present:

```
base, web, mail, account, stock_account, sale, purchase, point_of_sale
```

The terminal should display output similar to:

```
2026-04-15 11:17:34,248 INFO erp_538_v15_2026-03-31_09-21-46 odoo.modules.loading: Modules loaded.
2026-04-15 11:17:34,291 INFO erp_538_v15_2026-03-31_09-21-46 odoo.modules.registry: Registry loaded in 310.332s
```

---

## 9. Migration Step 2: Odoo 16 → Odoo 17

### 9.1 Pre-Migration Checklist

Before running the migration:

1. Re-apply the **Pre-Migration Cleanup SQL** script from **Section 6** against the database.
2. Update `addons_path` in the Odoo configuration file to point to `OpenUpgrade-17.0`, as described in **Section 4**.

### 9.2 Run the Migration Command

Switch to the **Odoo 17** project workspace and run the following command:

```bash
python odoo-bin -c odoo-hboueri.user.conf \
  --database=erp_538_v15_2026-03-31_09-21-46 \
  --upgrade-path="C:\workarea\SABIS059 - ODOO ERP\ODOO_ERP\openUpgrade\OpenUpgrade-17.0\openupgrade_scripts\scripts" \
  --load=base,web,openupgrade_framework \
  --update all \
  --stop-after-init
```

### 9.3 Confirm Successful Migration

The migration has completed successfully when you see the following messages:

```
INFO ... odoo.modules.loading: Modules loaded.
INFO ... odoo.modules.registry: Registry loaded in X.XXXs
```

### 9.4 Post-Migration Validation

Configure the following modules in the **Odoo 17** workspace and start the project:

```
base, web, mail, account, stock_account, sale, purchase, point_of_sale
```

The terminal should display output similar to:

```
2026-04-15 11:17:34,248 INFO erp_538_v15_2026-03-31_09-21-46 odoo.modules.loading: Modules loaded.
2026-04-15 11:17:34,291 INFO erp_538_v15_2026-03-31_09-21-46 odoo.modules.registry: Registry loaded in 310.332s
```

---

## 10. Final Migration Step: Odoo 17 → Odoo 18

### 10.1 Pre-Migration Checklist

1. Re-apply the **Pre-Migration Cleanup SQL** script from **Section 6** against the database.
2. Update `addons_path` in the Odoo configuration file to point to `OpenUpgrade-18.0`, as described in **Section 4**.

### 10.2 Run the Migration Command

Switch to the **Odoo 18** project workspace and run the following command:

```bash
python odoo-bin -c odoo-hboueri.conf \
  --database=erp_538_v15_2026-03-31_09-21-46 \
  --upgrade-path="C:\workarea\SABIS059 - ODOO ERP\ODOO_ERP\openUpgrade\OpenUpgrade-18.0\openupgrade_scripts\scripts" \
  --load=base,web,openupgrade_framework \
  --update all \
  --stop-after-init
```

### 10.3 Confirm Successful Migration

The migration has completed successfully when you see the following messages:

```
INFO ... odoo.modules.loading: Modules loaded.
INFO ... odoo.modules.registry: Registry loaded in X.XXXs
```

### 10.4 Post-Migration Validation

Configure the following modules in the **Odoo 18** workspace and start the project:

```
base, web, mail, account, stock_account, sale, purchase, point_of_sale
```

The terminal should display output similar to:

```
2026-04-15 11:17:34,248 INFO erp_538_v15_2026-03-31_09-21-46 odoo.modules.loading: Modules loaded.
2026-04-15 11:17:34,291 INFO erp_538_v15_2026-03-31_09-21-46 odoo.modules.registry: Registry loaded in 310.332s
```

### 10.5 Re-enable Custom SABIS Modules

Once the core migration is validated, proceed to update and re-enable all custom SABIS modules, for example:

```
sabis_core, sabis_accounting, sabis_pos, sabis_leave_management, ...
```

> *(Add the complete module list here once confirmed.)*

---

## 11. Post-Migration Security Checklist

After completing the full migration to Odoo 18, perform the following security-related cleanup:

| # | Action | Details |
|---|---|---|
| 1 | Disable OAuth password login | Set `enabled = false` for `Odoo.com Accounts` in `auth_oauth_provider` |
| 2 | Reset the `admin` password | Replace the temporary admin password with a strong, secure password |
| 3 | Review user access rights | Confirm all user roles and permissions are correctly configured in Odoo 18 |

---

*Document Version: 1.0 — Last Updated: April 2026*