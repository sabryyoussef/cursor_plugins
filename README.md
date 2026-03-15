# Freezoners Plugin

A Cursor plugin for diagnosing, upgrading, exploring, and scaffolding custom Odoo 18 modules for the Freezoner installation.

## Overview

This plugin provides skills and an auto-applied rule to help develop, debug, upgrade, and maintain the 59+ Odoo 18 customizations in the Freezoner codebase — located at `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/`.

## Skills

Invoke skills with `/` in Cursor, e.g. `/diagnose-module`.

| Skill | Invoke | Description |
|---|---|---|
| `diagnose-module` | `/diagnose-module` | Deep scan for errors: field-level validation, XPath, missing security entries, manifest dependency gaps, deprecated API usage, and live log analysis |
| `upgrade-module` | `/upgrade-module` | Step-by-step guide to safely upgrade a module on the running Odoo 18 instance — with level-ordered dependency awareness |
| `workflow-map` | `/workflow-map` | Map and explain all business workflows: CRM pipeline, compliance onboarding risk, project/task lifecycle, sale approval, SOV financials. Includes stage-by-stage validations |
| `explore-modules` | `/explore-modules` | Look up any of the 59+ modules, find reverse dependencies, locate models and fields, browse all developer namespaces |
| `scaffold-freezoner-module` | `/scaffold-freezoner-module` | Create a new Freezoner module following team conventions: correct level folder, Odoo 18 API patterns, security, views, and migration scripts |
| `manage-odoo-module` | `/manage-odoo-module` | General Odoo module scaffold and update helper |

## Rule (Auto-Applied)

**`freezoner-context`** — Automatically activates when you open any `.py`, `.xml`, or `.csv` file under `custom_addons_odoo18_freezoner/`. Provides:
- Project structure and level architecture
- Odoo 18 API conventions and forbidden patterns
- Known error patterns from documented bug tickets (ERROR_030 through ERROR_163)
- Service management commands with correct paths
- Git workflow reminder

## Architecture

The codebase uses a **5-level dependency architecture**:

```
Level 0 (Independent)  →  Level 1  →  Level 2
                                           ↓
                                       Level 3 (freezoner_custom)
                                           ↓
                                       Level 4 (compliance_cycle, sale_approval, commissions)
                                           ↓
                                       Level 5 (project_custom)
```

## Key Paths

| Resource | Path |
|---|---|
| Custom modules | `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/` |
| Odoo 18 config | `/opt/odoo/conf/odoo18.conf` |
| Odoo 18 log | `/opt/odoo/logs/odoo18.log` |
| Odoo binary | `/opt/odoo/src/odoo18/odoo-bin` |
| Virtualenv | `/opt/odoo/venvs/odoo18/bin/python` |
| Error tickets | `odoo-sh/error_fixes/ERROR_XXX_*/` |

## Requirements

- Cursor with plugin support
- Odoo 18 running at port 8018

## Version

0.3.0
