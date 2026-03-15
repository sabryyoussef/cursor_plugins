---
name: upgrade-module
description: Step-by-step guide to safely upgrade one or more Freezoner custom Odoo 18 modules. Handles pre-checks, level-ordered upgrades, log monitoring, and post-upgrade validation.
---

# Upgrade Module

## Trigger

Use when you need to apply code changes to a running Odoo 18 module — after editing models, views, security rules, or data files — or when upgrading from one module version to another.

## Required Inputs

- Module name(s) to upgrade (e.g. `freezoner_custom`, or `compliance_cycle,project_custom`)

## Key Paths & Commands

| Resource | Value |
|---|---|
| Odoo 18 binary | `/opt/odoo/src/odoo18/odoo-bin` |
| Virtualenv | `/opt/odoo/venvs/odoo18/bin/python` |
| Config file | `/opt/odoo/conf/odoo18.conf` |
| Log file | `/opt/odoo/logs/odoo18.log` |
| Service name | `odoo18` |
| Database name | `odoo18` |

> **Note:** The `UPDATE_MODULES.sh` script has a bug — it checks `/var/log/odoo18/odoo.log` which does not exist. The correct log path is `/opt/odoo/logs/odoo18.log`.

## Dependency Level Order

Always upgrade dependencies before dependents. Use this order:

```
Level 0  →  Level 1  →  Level 2  →  Level 3  →  Level 4  →  Level 5
```

**Quick dependency reference for the core chain:**
```
base_legal_types (L0)
partner_organization (L0)
hr_employee_custom (L0)
    ↓
partner_custom (L1)
client_documents (L1)
    ↓
partner_custom_fields (L2)
cabinet_directory (L2)
crm_report (L2)
    ↓
freezoner_custom (L3)
crm_log (L3)
    ↓
compliance_cycle (L4)
freezoner_sale_approval (L4)
sales_commission (L4)
    ↓
project_custom (L5)
```

**Rule:** If you are upgrading `project_custom`, you may need to upgrade `compliance_cycle` and `freezoner_custom` first if those were also changed.

## Workflow

### Step 1 — Pre-Upgrade Checks

Before touching the service:

1. Run `diagnose-module` on the target module(s) — fix any CRITICAL issues first.
2. Check if there are new migration scripts needed:
   - If model fields were added/removed/renamed → ensure `migrations/18.0/post-migration.py` handles it
   - If adding a new `_name` model → ensure `ir.model.access.csv` has the new row
3. Check the current module version in `__manifest__.py` — increment it if making a versioned change (e.g. `18.0.1.0.2` → `18.0.1.0.3`)
4. Verify the module's folder is in `addons_path` in `/opt/odoo/conf/odoo18.conf`

### Step 2 — Stop the Service

```bash
sudo systemctl stop odoo18
```

Confirm it stopped:
```bash
sudo systemctl status odoo18
```

### Step 3 — Run the Upgrade

#### Single module:
```bash
sudo -u odoo /opt/odoo/venvs/odoo18/bin/python /opt/odoo/src/odoo18/odoo-bin \
  -c /opt/odoo/conf/odoo18.conf \
  -u <module_name> \
  --stop-after-init
```

#### Multiple modules (comma-separated):
```bash
sudo -u odoo /opt/odoo/venvs/odoo18/bin/python /opt/odoo/src/odoo18/odoo-bin \
  -c /opt/odoo/conf/odoo18.conf \
  -u module_a,module_b,module_c \
  --stop-after-init
```

#### Full dependency chain upgrade (when core modules changed):
```bash
sudo -u odoo /opt/odoo/venvs/odoo18/bin/python /opt/odoo/src/odoo18/odoo-bin \
  -c /opt/odoo/conf/odoo18.conf \
  -u freezoner_custom,compliance_cycle,freezoner_sale_approval,sales_commission,project_custom \
  --stop-after-init
```

> Use `--stop-after-init` always — it runs the upgrade then exits cleanly, allowing you to check logs before restarting.

### Step 4 — Check the Upgrade Log

```bash
tail -100 /opt/odoo/logs/odoo18.log
```

Look for:
- `INFO odoo18 odoo.modules.loading: Upgrading module <name>` → upgrade started
- `INFO odoo18 odoo.modules.loading: Module <name> loaded` → success
- `ERROR` lines → must fix before restarting
- `WARNING` lines → review, may be non-critical

**Common upgrade errors and fixes:**

| Error Pattern | Cause | Fix |
|---|---|---|
| `KeyError: 'field_name'` | Field referenced in view doesn't exist in model | Add field to model or remove from view |
| `ValueError: External ID not found` | Data XML references missing xmlid | Add the record or fix the ref |
| `psycopg2.IntegrityError: NOT NULL` | New required field has no default | Add `default=` or write migration script |
| `Invalid field ... on model ...` | Field name typo or wrong model | Check `_inherit` / `_name` and field spelling |
| `Element '<xpath>' cannot be located` | XPath expression doesn't match base view | Run `diagnose-module` → XPath check |
| `Table ... does not exist` | Model registered but DB table not created | Check `_name` is unique, re-run upgrade |
| `ir.ui.view ... already exists` | Duplicate XML ID | Prefix view IDs with module name |

### Step 5 — Restart the Service

Only restart after the log shows a clean upgrade with no errors:

```bash
sudo systemctl start odoo18
```

Confirm it's running:
```bash
sudo systemctl status odoo18
```

### Step 6 — Post-Upgrade Validation

1. Open the Odoo 18 backend in browser (port 8018)
2. Navigate to the module's main menu items
3. Verify:
   - All views render without error
   - List/form/kanban views load
   - New fields are visible and editable
   - Computed fields calculate correctly
   - Access rights work (test with a non-admin user if possible)
4. If something looks wrong, check the log again:
   ```bash
   tail -50 /opt/odoo/logs/odoo18.log
   ```

## Installing a New Module (first time)

For a brand new module that has never been installed:

```bash
sudo systemctl stop odoo18

sudo -u odoo /opt/odoo/venvs/odoo18/bin/python /opt/odoo/src/odoo18/odoo-bin \
  -c /opt/odoo/conf/odoo18.conf \
  -i <new_module_name> \
  --stop-after-init

sudo systemctl start odoo18
```

> Use `-i` (install) for first-time installs, `-u` (upgrade) for updates to already-installed modules.

## Adding a New Module Path to Config

If a new module is placed in a folder not yet in `addons_path`, edit `/opt/odoo/conf/odoo18.conf`:

```ini
addons_path = ...,/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/_beshoy/level_X_<name>
```

Then restart the service (no `-u` needed just to register the path).

## Git Sync Reminder

**Never commit from `/opt/odoo/`.** After a successful upgrade and verification:

1. Sync changes to the Git repo: `/opt/Freezoner/`
2. Commit and push from there
3. The branch convention from `README.md`: use the `staging` branch

## Guardrails

- Always use `--stop-after-init` — never restart the service while an upgrade is mid-run
- Upgrade lower-level dependencies first
- Never run `-u all` in production — always specify module names explicitly
- Back up the database before upgrading modules with migration scripts
