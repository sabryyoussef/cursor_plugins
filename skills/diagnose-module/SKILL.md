---
name: diagnose-module
description: Diagnose a Freezoner custom Odoo 18 module for errors, missing security entries, XPath issues, manifest dependency gaps, field-level logic errors, and common Odoo 18 API problems. Checks live logs and known error tickets.
---

# Diagnose Module

## Trigger

Use when you need to find what is wrong with a Freezoner custom Odoo 18 module — errors, install failures, upgrade failures, view rendering issues, access errors, computed field bugs, or unexpected behavior.

## Required Inputs

- Module name (e.g. `freezoner_custom`, `compliance_cycle`, `partner_custom`)

## Key Paths

| Resource | Path |
|---|---|
| Module root | `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/` |
| Odoo 18 log | `/opt/odoo/logs/odoo18.log` |
| Known error tickets | `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/error_fixes/` |
| Odoo 18 config | `/opt/odoo/conf/odoo18.conf` |

---

## Workflow

### Step 1 — Locate the Module

Use the level map to find the module path:

| Level | Path |
|---|---|
| 0 | `_beshoy/level_0_independent/<module>/` |
| 1 | `_beshoy/level_1_base_custom/<module>/` |
| 2 | `_beshoy/level_2_simple_deps/<module>/` |
| 3 | `_beshoy/level_3_complex_deps/<module>/` |
| 4 | `_beshoy/level_4_high_level/<module>/` |
| 5 | `_beshoy/level_5_top_level/<module>/` |
| Free | `_free_installed_modules/<module>/` |
| Paid | `_paid_modules/<module>/` |
| Fix | `_fix_modules/<module>/` |
| Other devs | `_sabry_youssef/`, `_youssef/`, `_ziad/` |

### Step 2 — Read All Source Files

Read every file in this order:
1. `__manifest__.py` — version, dependencies, data file list
2. `models/__init__.py` — which model files are imported
3. Every `.py` file in `models/` and `wizard/` — capture all classes, `_name`, `_inherit`, all field definitions, `@api.depends`, `@api.onchange`, `@api.constrains`, method signatures
4. Every `.xml` file in `views/` and `wizard/` — all `<field>`, `<xpath>`, `<button>`, `domain=` attributes
5. `security/ir.model.access.csv`
6. `migrations/` folder

### Step 3 — Deep Static Analysis

#### A. Security Access Completeness
- List every `_name` (new model) from model files
- Cross-check against `ir.model.access.csv`
- Every `_name` must have at least one row in the CSV
- Flag: **CRITICAL — MISSING ACCESS RULE: model `x.y` has no entry in ir.model.access.csv**

#### B. Manifest Dependency Completeness
- Collect all `_inherit` values from model files
- Map each to its source module:

```
res.partner              → base
sale.order               → sale
sale.order.line          → sale
project.project          → project
project.task             → project
project.task.type        → project
project.milestone        → project
crm.lead / crm.stage     → crm
hr.employee              → hr
hr.employee.base         → hr
hr.payslip               → hr_payroll
hr.attendance            → hr_attendance
hr.leave                 → hr_holidays
account.move             → account
documents.document       → documents
mail.thread              → mail
mail.activity.mixin      → mail
approval.request         → approvals
approval.category        → approvals
utm.source               → utm
analytic.line            → analytic
report.report_xlsx.*     → report_xlsx
crm.lead (compliance)    → crm + freezoner_custom (for compliance_cycle)
initial.client.onboarding → compliance_cycle
sale.sov                 → freezoner_custom
partner.stage            → partner_custom
shareholder.data         → partner_custom
legal.type               → base_legal_types
```

- Flag any source module missing from `depends` → **WARNING — MISSING DEPENDENCY**

#### C. Field-Level Validation (deep check)

For every `@api.depends(...)` decorator:
- Parse the field path list
- Verify every field name in the decorator exists on the model (or its `_inherit` chain)
- Flag missing fields → **CRITICAL — @api.depends references non-existent field `x`**

For every `@api.onchange(...)`:
- Verify every listed field exists on the model
- Flag: **WARNING — @api.onchange references non-existent field `x`**

For every `@api.constrains(...)`:
- Verify every listed field exists
- Flag: **WARNING — @api.constrains references non-existent field `x`**

For every `fields.Related(...)` or `related='x.y'`:
- Trace the relation chain: `x` must be a relational field, `y` must exist on the target model
- Flag broken chains → **WARNING — broken related field chain `x.y`**

For computed fields with `store=True`:
- Check they have either an inverse or are clearly read-only
- `store=True` computed fields without `readonly=True` or an `inverse=` can cause unexpected write behavior
- Flag: **INFO — computed field `x` is store=True without inverse — confirm this is intentional**

#### D. View Field Name Validation (cross-reference with models)

For every `<field name="...">` in XML views:
- Collect the model from the `<record model="ir.ui.view">` `model` field attribute
- Verify the field name exists on that model (using the model files you already read)
- Flag: **CRITICAL — view `x` references field `y` which does not exist on model `z`**

For every `<button name="method_name">`:
- Verify a Python method with that name exists in the model class
- Flag: **WARNING — button references method `x` not found on model `y`**

For every `domain="[...]"` in views:
- Parse domain tuples: `[('field_name', 'operator', value)]`
- Verify `field_name` exists on the referenced model
- Flag: **WARNING — domain references non-existent field `x`**

#### E. XPath Validation (most common Odoo 18 issue — ERROR_163)

For every `<xpath expr="...">` in inherited views:
- Extract the field or element name from the expr
- Known safe base fields for common inherited models:

```
sale.order form:      partner_id, date_order, validity_date, state, name, order_line
project.project form: name, partner_id, user_id, date, stage_id, tag_ids
project.task form:    name, user_ids, project_id, stage_id, date_deadline, description
crm.lead form:        partner_id, stage_id, user_id, team_id, priority, email_from, phone
res.partner form:     name, email, phone, mobile, street, city, country_id, vat
hr.employee form:     name, job_id, department_id, coach_id, parent_id, user_id
```

- If the XPath targets a field NOT in the known-safe list, flag: **WARNING — XPath `expr` targets field `x` — verify it exists in Odoo 18 base view**
- Specifically watch for: `is_hide_quotation_button` (exists, added by `crm_log`), `approval_request_ids` (added by `freezoner_sale_approval`)

#### F. Odoo 18 API Deprecations

Scan all `.py` files for:

| Pattern | Severity | Fix |
|---|---|---|
| `@api.multi` | CRITICAL | Remove decorator |
| `@api.one` | CRITICAL | Remove decorator, use `self.ensure_one()` |
| `from openerp import` | CRITICAL | Replace with `from odoo import` |
| `self.pool.get(` | CRITICAL | Replace with `self.env['model']` |
| `fields.Many2many(` with 4 positional args | WARNING | Use named kwargs |
| `<workflow>` XML tag | CRITICAL | Remove, use Python state machines |
| `cr.execute(` outside model | WARNING | Wrap in `self.env.cr.execute(` |
| `_columns = {` dict syntax | CRITICAL | Migrate to field declarations |
| `.search([])` with no `limit=` in crons | WARNING | Add `limit=` to prevent full-table scans |

#### G. SOV / Financial Model Checks (specific to freezoner_custom)

If diagnosing `freezoner_custom` or `sales_commission`, check the `sale.sov` model:
- `revenue`, `profit`, `tax`, `net` are all `store=True` computed fields
- Verify `sale_id` Many2one exists and `sale.order` is in `depends`
- Check `commission_attribute` Selection field has all expected values
- Verify `sale_commission_user_ids` Many2many uses named relation table argument

#### H. External Python Dependencies

Check `__manifest__.py` for `external_dependencies`:
- `phonenumbers` (used in `crm_log`) → verify: `sudo /opt/odoo/venvs/odoo18/bin/pip show phonenumbers`
- `xlsxwriter` (used by `report_xlsx`) → verify similarly
- `pytz` → included with Odoo, always available
- `dateutil` → included with Odoo

Flag missing packages → **CRITICAL — Python package `x` not installed in venv**

#### I. Migration Script Check

- Every module should have `migrations/18.0/post-migration.py`
- If the module was migrated from v16/v17 AND has new fields or model changes → missing script is **WARNING**
- If it's a pure new module (never existed before Odoo 18) → missing script is **INFO** only

### Step 4 — Check Live Logs

```bash
grep -i "<module_name>" /opt/odoo/logs/odoo18.log | grep -iE "ERROR|WARNING|CRITICAL" | tail -30
```

Also check module loading:
```bash
grep "odoo.modules.loading" /opt/odoo/logs/odoo18.log | grep -i "<module_name>" | tail -10
```

Look for patterns:
- `invalid field` → field name typo or missing field definition
- `KeyError` → XML references non-existent record or field
- `cannot be located` → broken XPath
- `psycopg2` → database constraint or migration issue
- `AccessError` → missing access rule in CSV

### Step 5 — Cross-Check Known Error Tickets

List the `error_fixes/` folder and check for related tickets:

| Ticket | Module(s) | Issue |
|---|---|---|
| ERROR_163_PARTNER_CUSTOM_XPATH | `partner_custom`, `compliance_cycle` | XPath expressions targeting removed/renamed Odoo 18 fields |
| ERROR_162_COMMISSION_SUBMISSION | `sales_commission` | User view SOV chatter missing — mail.thread inheritance |
| ERROR_161_CORPORATE_TAX_AUTOPOP | `partner_custom` | `@api.onchange` on corporate tax not firing — field name mismatch |
| ERROR_150_INVISIBLE_FIELDS | `partner_custom`, `compliance_cycle` | Odoo 18 changed `attrs` → `invisible` domain syntax |
| ERROR_137_BACKUP_EMPLOYEE | `hr_employee_custom` | `hr.employee.base` vs `hr.employee` inheritance mismatch |
| ERROR_030_PROJECT_TAGS_VALIDATION | `project_custom` | M2M tag field domain constraint raising on unrelated records |

If a related ticket exists, read its analysis file and surface findings in the report.

### Step 6 — Produce Diagnostic Report

```
## Diagnostic Report: <module_name>
**Path:** <full path>
**Version:** <from manifest>
**Author:** <from manifest>
**Level:** <0-5 or other>

---
### Issues Found

[CRITICAL] SECURITY — Model `x.y` has no entry in ir.model.access.csv
  Fix: Add row: access_x_y_user,x.y user,model_x_y,base.group_user,1,1,1,0

[CRITICAL] VIEW — view `view_x_form` references field `some_field` not found on model `x.y`
  File: views/x.xml line 42
  Fix: Remove or rename field reference; check model definition

[WARNING] XPATH — expr `//field[@name='old_field']` — verify field exists in Odoo 18 base view
  File: views/inherited.xml line 17

[WARNING] DEPENDENCY — _inherit `approval.request` used but `approvals` not in manifest depends
  Fix: Add 'approvals' to depends list in __manifest__.py

[INFO] COMPUTED — field `revenue` is store=True without inverse
  File: models/sov.py line 20
  Confirm this is intentional (read-only stored compute)

---
### Summary
- X CRITICAL issues (must fix before upgrade)
- X WARNING issues (should fix)
- X INFO items (review)

**Recommended next step:** <fix the top CRITICAL issue first>
```

---

## Common Fixes Quick Reference

| Issue | Fix |
|---|---|
| Missing ir.model.access entry | `access_<model>_user,<model> user,model_<model_underscores>,base.group_user,1,1,1,0` |
| Missing manifest dependency | Add module to `depends` in `__manifest__.py` |
| Broken XPath | Read the Odoo 18 base view to confirm field path |
| Deprecated `@api.multi` | Remove the decorator (Odoo 18 default) |
| Invisible fields not working | Change `attrs="{'invisible': [...]}"` → `invisible="[...]"` (Odoo 17+ syntax) |
| Missing migration script | `migrations/18.0/post-migration.py` with `def migrate(cr, version): pass` |
| Missing Python package | `sudo /opt/odoo/venvs/odoo18/bin/pip install <package>` |
| `store=True` compute no inverse | Add `readonly=True` to confirm intent, or add `inverse=` method |

## Guardrails

- Never modify files in `/opt/odoo/src/odoo18/` (core Odoo source)
- Always work on files under `custom_addons_odoo18_freezoner/odoo-sh/`
- Do not commit from `/opt/odoo` — sync to `/opt/Freezoner` first
- After fixing, run `upgrade-module` to apply changes
