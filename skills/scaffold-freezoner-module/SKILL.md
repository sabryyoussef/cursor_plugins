---
name: scaffold-freezoner-module
description: Create a new Freezoner custom Odoo 18 module following team conventions. Determines the correct dependency level folder, generates all required files, and updates the Odoo 18 config if needed.
---

# Scaffold Freezoner Module

## Trigger

Use when you need to create a brand new custom Odoo 18 module for the Freezoner installation, following the team's established conventions for file structure, naming, versioning, and dependency levels.

## Required Inputs

- Module name (lowercase with underscores, e.g. `freezone_kyc`)
- Module purpose / description (1–2 sentences)
- Dependencies: list of Odoo standard modules + any Freezoner custom modules it needs
- Author name (default: `Beshoy Wageh` — change if for another dev)

## Module Root

All Freezoner custom modules live under:
```
/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/
```

## Workflow

### Step 1 — Validate Module Name

- Must be lowercase with underscores only (no hyphens, no spaces, no uppercase)
- Must not conflict with existing module names (check the registry in `explore-modules`)
- Must not start with a number

### Step 2 — Determine Dependency Level

Analyze the declared dependencies and assign the correct level folder:

| Condition | Level | Folder |
|---|---|---|
| Depends only on Odoo standard modules (no custom deps) | 0 | `_beshoy/level_0_independent/` |
| Depends on one Level 0 custom module | 1 | `_beshoy/level_1_base_custom/` |
| Depends on Level 1 modules | 2 | `_beshoy/level_2_simple_deps/` |
| Depends on Level 2 modules (e.g. `freezoner_custom`, `crm_log`) | 3 | `_beshoy/level_3_complex_deps/` |
| Depends on Level 3 modules | 4 | `_beshoy/level_4_high_level/` |
| Depends on Level 4 modules | 5 | `_beshoy/level_5_top_level/` |

Use the deepest level of any custom dependency as the module's level.

**If created by another developer**, use their namespace folder instead:
- `_sabry_youssef/`, `_youssef/`, `_ziad/`

### Step 3 — Create the Directory Structure

```
<level_folder>/<module_name>/
├── __init__.py
├── __manifest__.py
├── models/
│   └── __init__.py
├── views/
├── security/
│   ├── ir.model.access.csv
│   └── security.xml
├── migrations/
│   └── 18.0/
│       └── post-migration.py
└── docs/
```

Add these directories only if needed:
- `wizard/` — if the module has wizard dialogs
- `data/` — if the module has seed data, stages, or cron jobs
- `controller/` — if the module exposes HTTP routes or portal pages
- `static/src/css/` and `static/src/js/` — if the module has frontend assets
- `reports/` — if the module has QWeb PDF reports

### Step 4 — Generate `__manifest__.py`

Follow this exact format:

```python
{
    'name': "<Human Readable Name>",
    'author': "Beshoy Wageh",
    'version': '18.0.1.0.0',
    'license': 'LGPL-3',
    'depends': [
        # Standard Odoo modules first:
        'base',
        'sale',           # add as needed
        # Freezoner custom modules after:
        'freezoner_custom',  # add as needed
    ],
    'data': [
        'security/security.xml',
        'security/ir.model.access.csv',
        # 'data/data.xml',        # uncomment if needed
        # 'views/views.xml',      # add view files
    ],
    # Only add 'assets' block if using static JS/CSS:
    # 'assets': {
    #     'web.assets_backend': [
    #         '<module_name>/static/src/css/style.scss',
    #     ],
    # },
    'auto_install': False,
}
```

**Versioning convention:** `18.0.<major>.<minor>.<patch>` — always start new modules at `18.0.1.0.0`.

### Step 5 — Generate `__init__.py` (root)

```python
from . import models
```

Add more imports if using wizards/controllers:
```python
from . import models, wizard, controller
```

### Step 6 — Generate `models/__init__.py`

```python
from . import models  # rename to your first model file name
```

### Step 7 — Generate a Starter Model File

Create `models/<module_name>.py`:

```python
from odoo import models, fields, api


class <ClassName>(models.Model):
    _name = '<module_name>.record'
    _description = '<Human Readable Description>'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char(string='Name', required=True, tracking=True)
    active = fields.Boolean(default=True)
    # Add fields here

    def action_example(self):
        self.ensure_one()
        # Add logic here
```

**Odoo 18 conventions:**
- Never use `@api.multi` (removed in Odoo 13+)
- Never use `@api.one` (removed)
- Use `self.ensure_one()` instead of `@api.one`
- Use `fields.Many2many` with named arguments: `fields.Many2many('res.partner', 'rel_table', 'col1', 'col2')`
- Use `_inherit = ['mail.thread', 'mail.activity.mixin']` for chatter support
- Use `tracking=True` on fields that should appear in chatter log

### Step 8 — Generate `views/<module_name>.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- Form View -->
    <record id="view_<model_short>_form" model="ir.ui.view">
        <field name="name"><module_name>.record.form</field>
        <field name="model"><module_name>.record</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <group>
                        <field name="name"/>
                    </group>
                </sheet>
                <div class="oe_chatter">
                    <field name="message_follower_ids"/>
                    <field name="activity_ids"/>
                    <field name="message_ids"/>
                </div>
            </form>
        </field>
    </record>

    <!-- Tree View -->
    <record id="view_<model_short>_tree" model="ir.ui.view">
        <field name="name"><module_name>.record.tree</field>
        <field name="model"><module_name>.record</field>
        <field name="arch" type="xml">
            <tree>
                <field name="name"/>
            </tree>
        </field>
    </record>

    <!-- Action -->
    <record id="action_<model_short>" model="ir.actions.act_window">
        <field name="name"><Human Name></field>
        <field name="res_model"><module_name>.record</field>
        <field name="view_mode">tree,form</field>
    </record>

    <!-- Menu -->
    <menuitem id="menu_<module_name>_root"
              name="<Human Name>"
              sequence="10"/>
    <menuitem id="menu_<module_name>"
              name="<Human Name>"
              parent="menu_<module_name>_root"
              action="action_<model_short>"/>

</odoo>
```

**View ID naming rule:** Always prefix view IDs with the module name to ensure uniqueness across all 59+ modules.

### Step 9 — Generate `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_<module_name>_record_user,<module_name>.record user,model_<module_name>_record,base.group_user,1,1,1,0
access_<module_name>_record_manager,<module_name>.record manager,model_<module_name>_record,base.group_system,1,1,1,1
```

**Note:** Add one row per `_name` model defined in the module. The `model_id:id` field uses underscores: `model_` + `_name` with dots replaced by underscores.

### Step 10 — Generate `security/security.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Security groups for <module_name> -->
    <!-- Add custom groups here if needed, or leave minimal -->
</odoo>
```

### Step 11 — Generate `migrations/18.0/post-migration.py`

```python
def migrate(cr, version):
    """Post-migration script for version 18.0."""
    pass
```

### Step 12 — Update `addons_path` in Config

If the new module is in a level folder not already registered, add it to `/opt/odoo/conf/odoo18.conf`:

```ini
addons_path = ...,/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/_beshoy/level_X_<name>
```

Currently registered level paths (already in config):
- `_beshoy/level_0_independent`
- `_beshoy/level_1_base_custom`
- `_beshoy/level_2_simple_deps` (via `_beshoy` root — confirm)
- `_beshoy/level_3_complex_deps` (via `_beshoy` root — confirm)
- `_beshoy/level_4_high_level`
- `_beshoy/level_5_top_level`

Check the current `addons_path` in config before editing.

### Step 13 — Install the New Module

Use `upgrade-module` skill with `-i` flag:

```bash
sudo systemctl stop odoo18
sudo -u odoo /opt/odoo/venvs/odoo18/bin/python /opt/odoo/src/odoo18/odoo-bin \
  -c /opt/odoo/conf/odoo18.conf \
  -i <new_module_name> \
  --stop-after-init
sudo systemctl start odoo18
```

## Output

After scaffolding, provide:
1. Full file tree of the created module
2. List of models created and their `_name` values
3. Any `addons_path` changes made to the config
4. Command to install the module
5. Reminder to sync to `/opt/Freezoner/` before committing

## Git Workflow Reminder

**Never commit directly from `/opt/odoo/`.** The correct flow:
1. Make changes under `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/`
2. Sync to `/opt/Freezoner/` (the Git repo)
3. Commit and push from `/opt/Freezoner/` on the `staging` branch
4. Keep module versions incremented on every committed change

## Guardrails

- Never place new modules inside `/opt/odoo/src/odoo18/` (core source)
- Never name a module the same as an existing Odoo standard module
- Always test installation with `--stop-after-init` before live restart
- Run `diagnose-module` after scaffolding to catch any structural issues before installation
