---
name: diagnose-module
description: Diagnose a Freezoner custom Odoo 18 module for errors, missing security entries, XPath issues, manifest dependency gaps, and common Odoo 18 API problems. Checks live logs and known error tickets.
---

# Diagnose Module

## Trigger

Use when you need to find what is wrong with a Freezoner custom Odoo 18 module — errors, install failures, upgrade failures, view rendering issues, access errors, or unexpected behavior.

## Required Inputs

- Module name (e.g. `freezoner_custom`, `compliance_cycle`, `partner_custom`)

## Key Paths

| Resource | Path |
|---|---|
| Module root | `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/` |
| Odoo 18 log | `/opt/odoo/logs/odoo18.log` |
| Known error tickets | `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/error_fixes/` |
| Odoo 18 config | `/opt/odoo/conf/odoo18.conf` |

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

### Step 2 — Read Core Files

Read in this order:
1. `__manifest__.py` — version, dependencies, data file list
2. `models/__init__.py` — which model files are imported
3. Each model file in `models/` — capture all `_name` and `_inherit` values
4. `security/ir.model.access.csv` — which models have access rules
5. All view XML files in `views/` and `wizard/` — scan for XPath and `inherit_id`
6. `migrations/` folder — check for `post-migration.py` presence

### Step 3 — Run Static Checks

#### A. Security Access Completeness
- List every `_name` (new model) found in `models/`
- Cross-check against `ir.model.access.csv`
- Flag any `_name` that has no row in the CSV → **MISSING ACCESS RULE**

#### B. Manifest Dependency Completeness
- List all `_inherit` values from model files (e.g. `res.partner`, `sale.order`)
- Map inherits to their source modules (see mapping below)
- Check that the source module is in `__manifest__.py` `depends`
- Flag any missing dependency → **MISSING DEPENDENCY**

Common inherit → module mapping:
```
res.partner          → base
sale.order           → sale
project.project      → project
project.task         → project
crm.lead             → crm
hr.employee          → hr
hr.payslip           → hr_payroll
account.move         → account
documents.document   → documents
```

#### C. XPath Validation (most common Odoo 18 issue)
- For every `<xpath>` in view XML files, check that the `expr` attribute uses valid Odoo 18 syntax
- Common broken patterns from ERROR_163:
  - Old: `//field[@name='partner_id']` → sometimes needs `//div[hasclass('...')]`
  - Old `position="after"` on non-existent fields
  - References to fields removed in Odoo 18 (e.g. `journal_id` moved, `user_signature` deprecated)
- Flag any xpath that references a field not confirmed to exist in the base model → **XPATH RISK**

#### D. Odoo 18 API Deprecations
Scan all `.py` files for:
- `@api.multi` → deprecated, use no decorator or `@api.model`
- `@api.one` → deprecated
- `self.env['model'].search([])` without limit in crons → performance risk
- Old-style workflow XML (`<workflow>`, `<act_window type="workflow">`) → not supported in Odoo 18
- `fields.Many2many` with positional args (4-arg form) → must use named args in Odoo 18
- `openerp` import paths → use `odoo`

#### E. External Python Dependencies
- Check `__manifest__.py` for `external_dependencies` key
- If `phonenumbers` used (like `crm_log`): verify `phonenumbers` is installed in `/opt/odoo/venvs/odoo18/`
- If `xlsxwriter` used (like `crm_report`): verify similarly
- Flag missing packages → **MISSING PYTHON DEPENDENCY**

#### F. Migration Script Check
- Every module should have `migrations/18.0/post-migration.py`
- If missing and module was migrated from Odoo 16/17 → **NO MIGRATION SCRIPT** (may indicate data issues)

### Step 4 — Check Live Logs

Run:
```bash
grep -i "<module_name>" /opt/odoo/logs/odoo18.log | grep -E "ERROR|WARNING|CRITICAL" | tail -30
```

Also check for:
```bash
grep -i "odoo.modules.loading" /opt/odoo/logs/odoo18.log | grep -i "<module_name>" | tail -10
```

### Step 5 — Cross-Check Known Error Tickets

List the `error_fixes/` folder and check if any ticket relates to the module being diagnosed:
- `ERROR_163_PARTNER_CUSTOM_XPATH` → XPath issues in `partner_custom` and `compliance_cycle`
- `ERROR_162_COMMISSION_SUBMISSION` → `sales_commission` user view SOV chatter
- `ERROR_161_CORPORATE_TAX_AUTOPOP` → `partner_custom` corporate tax auto-population
- `ERROR_150_INVISIBLE_FIELDS` → `partner_custom`, `compliance_cycle` invisible field bugs
- `ERROR_137_BACKUP_EMPLOYEE` → `hr_employee_custom` backup employee handling
- `ERROR_030_PROJECT_TAGS_VALIDATION` → `project_custom` tags validation

If a related ticket exists, read the analysis file and surface its findings.

### Step 6 — Produce Diagnostic Report

Output a structured report with these sections:

```
## Diagnostic Report: <module_name>
### Module Info
- Path, version, author, level

### Issues Found
Each issue in format:
[SEVERITY] CATEGORY — Description
  File: path/to/file.py line N
  Fix: suggested fix

Severities: CRITICAL | WARNING | INFO

### Summary
- N CRITICAL issues
- N WARNING issues
- N INFO items
- Recommended next step
```

## Common Fixes by Error Type

| Issue | Fix |
|---|---|
| Missing ir.model.access entry | Add row: `access_<model>_user,<model> user,model_<model>,base.group_user,1,0,0,0` |
| Missing manifest dependency | Add module to `depends` list in `__manifest__.py` |
| Broken XPath | Read the base Odoo 18 view XML to confirm correct field name/path |
| Deprecated `@api.multi` | Remove decorator (default behavior in Odoo 13+) |
| Missing migration script | Create `migrations/18.0/post-migration.py` with `def migrate(cr, version): pass` |
| Missing Python package | Run: `sudo /opt/odoo/venvs/odoo18/bin/pip install <package>` |

## Guardrails

- Never modify files in `/opt/odoo/src/odoo18/` (core Odoo source)
- Always work on files under `custom_addons_odoo18_freezoner/odoo-sh/`
- Do not commit from `/opt/odoo` — sync to `/opt/Freezoner` first, then commit/push
- After fixing files, always run `upgrade-module` skill to apply changes
