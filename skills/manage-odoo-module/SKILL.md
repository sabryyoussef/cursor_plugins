---
name: manage-odoo-module
description: Create, scaffold, or update a custom Odoo module for Freezoners. Handles module structure, manifest generation, models, views, and workflow definitions.
---

# Manage Odoo Module

## Trigger

Use when you need to create a new Odoo module, update an existing one, or scaffold any part of a Freezoners custom Odoo module (models, views, wizards, workflows, security rules, etc.).

## Required Inputs

- Module name (lowercase with underscores, e.g. `freezone_company`)
- Module purpose / description
- Components to include (models, views, wizards, reports, workflows, security)

## Workflow

1. Validate module name: lowercase, underscores only, no spaces.
2. Determine target path inside the Odoo addons directory.
3. Scaffold the module structure:
   - `__manifest__.py` with name, version, summary, depends, data files
   - `__init__.py`
   - `models/` with `__init__.py` and model files
   - `views/` with XML view and menu definitions
   - `security/ir.model.access.csv`
   - Optional: `wizards/`, `reports/`, `data/`, `static/`
4. Populate `__manifest__.py` with correct dependencies based on module purpose.
5. Create model stubs with appropriate fields and `_name`, `_description`.
6. Create view XML with form, tree, and action definitions.
7. Wire security access rules for all new models.
8. If workflows are needed, define them using Odoo's built-in state field + button patterns.

## Guardrails

- Always use Odoo 16/17 API conventions (`models.Model`, `fields.*`, `@api.*`).
- Never use deprecated v7/v8 workflow XML — use Python state machines instead.
- Keep `__manifest__.py` `depends` list minimal and accurate.
- All view IDs must be globally unique (prefix with module name).
- Security CSV must have an entry for every new model.

## Output

- Full scaffolded module file tree
- Summary of created models and their key fields
- List of menu items and actions created
- Any dependencies added to `__manifest__.py`
