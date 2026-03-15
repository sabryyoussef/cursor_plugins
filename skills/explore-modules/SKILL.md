---
name: explore-modules
description: Explore the Freezoner Odoo 18 custom module registry. Answer questions about module dependencies, model locations, which modules a developer owns, and what the full dependency chain is for any module. Also cross-references workflow documentation.
---

# Explore Modules

## Trigger

Use when you need to:
- Find which module contains a specific model or feature
- See what depends on a given module
- List all modules by a developer or category
- Understand the dependency chain before upgrading
- Check if a module is in quarantine
- Find integration and bridge modules

## Key Paths

| Resource | Path |
|---|---|
| Module root | `/opt/odoo/custom_addons_odoo18_freezoner/odoo-sh/` |
| Config (addons_path) | `/opt/odoo/conf/odoo18.conf` |
| Enhancement plans | `odoo-sh/module_enhancement_plans/` |
| Error tickets | `odoo-sh/error_fixes/` |

---

## Full Module Registry

### Beshoy Wageh — `_beshoy/` (Core Custom Modules)

#### Level 0 — Independent (no internal custom deps)

| Module | Key Purpose | Key `_inherit` |
|---|---|---|
| `account_invoice_report` | Custom invoice report | `account.move` |
| `base_legal_types` | Legal entity type lookup | new model `legal.type` |
| `bwa_email_conf` | Email configuration | `ir.mail_server` |
| `bwa_f360_commission` | F360 commission logic | `sale.order` |
| `employee_salesperson_task` | Link employee → salesperson | `hr.employee`, `project.task` |
| `hr_attendance_ip_mac` | Attendance by IP/MAC | `hr.attendance` |
| `hr_employee_custom` | Employee + payslip extensions | `hr.employee.base`, `hr.payslip` |
| `hr_expense_custom` | Expense customizations | `hr.expense` |
| `hr_leave_custom` | Leave customizations | `hr.leave`, `hr.leave.type` |
| `leaves_check` | Leave balance checks | `hr.leave` |
| `partner_fname_lname` | Split first/last name on partner | `res.partner` |
| `partner_organization` | Partner parent organization chart | `res.partner` |
| `partner_risk_assessment` | Risk scoring on partners | `res.partner` |
| `product_restriction` | Product availability rules | `product.template` |
| `project_partner_fields` | Extra fields on project linked to partner | `project.project` |
| `sales_person_customer_access` | Limit salesperson to own customers | `res.partner`, `sale.order` |
| `task_update` | Task update notifications | `project.task` |

#### Level 1 — Base Custom

| Module | Key Purpose | Depends on (custom) |
|---|---|---|
| `client_documents` | Document management per client | `partner_organization` |
| `partner_custom` | Full partner lifecycle: stages, shareholders, trade license, VAT, documents | `base_legal_types`, `partner_fname_lname` |

#### Level 2 — Simple Deps

| Module | Key Purpose | Depends on (custom) |
|---|---|---|
| `cabinet_directory` | Document cabinet/directory structure | `partner_custom` |
| `crm_report` | CRM XLSX reports via wizard | `report_xlsx` (free) |
| `partner_custom_fields` | Extra partner fields extension | `partner_custom` |

#### Level 3 — Complex Deps

| Module | Key Purpose | Depends on (custom) |
|---|---|---|
| `crm_log` | CRM stage transitions, call logging, phone validation (`phonenumbers`) | `hr_employee_custom`, `client_documents` |
| `freezoner_custom` | **Core module**: projects, tasks, documents, SOV, wizards, sales pipeline | `cabinet_directory`, `base_legal_types` |

#### Level 4 — High Level

| Module | Key Purpose | Depends on (custom) |
|---|---|---|
| `bwa_survey` | Custom survey workflows | `freezoner_custom` |
| `compliance_cycle` | Compliance tracking: business structures, onboarding, required documents | `freezoner_custom`, `partner_organization`, `crm_log` |
| `freezoner_password` | Password policy customization | `freezoner_custom` |
| `freezoner_sale_approval` | Sale order approval workflow with portal | `freezoner_custom`, `approvals` |
| `sales_commission` | Commission calculation integrated with payroll | `freezoner_custom`, `hr_payroll`, `account_budget` |

#### Level 5 — Top Level

| Module | Key Purpose | Depends on (custom) |
|---|---|---|
| `project_custom` | Full project customization: shareholders, milestones, documents, partner fields | `freezoner_custom`, `compliance_cycle`, `partner_custom`, `client_documents`, `partner_custom_fields` |

---

### Other Developers

#### Sabry Youssef — `_sabry_youssef/`

| Module | Version | Key Deps | Purpose |
|---|---|---|---|
| `employee_accountability` | — | `hr` | Employee accountability tracking |
| `error_ai_assistant` | — | `mail` | AI-assisted error analysis |
| `error_reporter_16` | — | `mail` | Error reporting (migrated from v16) |
| `error_workflow_manager` | 18.0.1.8.0 | `error_reporter_16`, `odoo_whatsapp_integration` | **Automated error fixing workflow** with API integration, history tracking, WhatsApp notifications, AI assistance. Has views: config, attempts, fetch wizard, local wizard, git commit/push wizards, sessions |
| `fsm_workflow_quote_v2` | — | `project`, `sale` | FSM workflow quoting v2 |
| `module_user_guide` | — | `base` | In-app user guide system |
| `openclaw_gateway` | — | `base`, `mail` | OpenClaw AI gateway integration (connects to OpenClaw MCP) |
| `pinecone_connector` | — | `base` | Pinecone vector DB connector (semantic search) |
| `product_category_search` | — | `product` | Product category search enhancement |
| `project_checkpoints_basic` | — | `project` | Project checkpoint/milestone tracking |
| `project_compliance` | 18.0.1.0.0 | `project`, `unified_documents`, `project_handover_notes`, `project_templates_basic`, `project_checkpoints_basic` | Compliance functionality for projects with shareholder management. Has: business_shareholder, project_compliance, partner_compliance views + return_compliance_wizard |
| `project_handover_notes` | — | `project` | Project handover documentation |
| `project_templates_basic` | — | `project` | Project template management |
| `shared_assets` | — | `base` | Shared CSS/JS assets used by other modules |
| `smart_templates` | — | `base`, `mail` | Smart document template generation |
| `unified_documents` | — | `documents` | Unified document management across modules |

> **Note on `error_workflow_manager`:** This is Sabry's most advanced module (v1.8.0). It integrates with `error_reporter_16` and WhatsApp to create automated error fixing sessions with git commit/push capabilities directly from Odoo.

#### Youssef — `_youssef/`

| Module | Key Deps | Purpose |
|---|---|---|
| `attendance_detection` | `hr_attendance` | Automated attendance detection |
| `crm_lead_heat` | `crm` | CRM lead heat scoring (extends `crm_log`'s priority field) |
| `discipline_system` | `hr` | HR discipline case management |
| `hr_attendance_location` | `hr_attendance` | Attendance with GPS location tracking |

#### Ziad — `_ziad/`

| Module | Version | Key Deps | Purpose |
|---|---|---|---|
| `client_birthday` | — | `res.partner` | Client birthday tracking and automated notifications |
| `client_categorisation` | — | `res.partner` | Client categorization by industry/type |
| `crm_assignation` | — | `crm` | CRM lead automatic assignment rules |
| `crm_controller` | — | `crm` | CRM controller/route extensions |
| `multiproject_saleorder` | 18.0.1.0.0 | `sale`, `project`, `analytic`, `sale_project` | Link multiple projects to one sale order. Has `data/data.xml` and views |
| `payment_validation` | 18.0.1.0.0 | `account`, `project`, `sale` | Payment validation workflow with project/sale linking. Has config views + mail templates |
| `project_by_client` | — | `project`, `res.partner` | Group and filter projects by client |

> **Note on `_sabry_youssef/`, `_youssef/`, `_ziad/` permissions:** These directories may have restricted read permissions. Use `sudo -u odoo` when reading files in these namespaces if permission errors occur.

---

### Free/Community Modules — `_free_installed_modules/`

| Module | Purpose |
|---|---|
| `activity_dashboard_mngmnt` | Activity dashboard management |
| `database_cleanup` | Database cleanup tools |
| `hide_any_menu` | Hide menus per user/group |
| `kw_project_assign_wizard` | Project assignment wizard |
| `ms_query` | Raw SQL query interface |
| `odoo_whatsapp_integration` | WhatsApp messaging integration |
| `payment_status_in_sale` | Show payment status on sale orders |
| `prt_email_from` | Customize email from address |
| `query_deluxe` | Advanced query builder |
| `remove_studio_field` | Remove Studio-created fields |
| `report_xlsx` | Excel (XLSX) report base |

---

### Paid Modules — `_paid_modules/`

| Module | Purpose |
|---|---|
| `bi_hr_equipment_asset_management` | Asset and equipment management |
| `bi_user_audit_management` | User audit log |
| `hr_salary_certificate` | Salary certificate generation |
| `partner_statement_knk` | Partner account statements |
| `stripe_fee_extension` | Stripe payment fee handling |

---

### Fix Modules — `_fix_modules/`

| Module | Purpose |
|---|---|
| `documents_search_panel_fix` | Fix documents search panel Odoo 18 issue |
| `pypdf2_fix` | Fix PyPDF2 compatibility |
| `sale_crm_fix` | Fix sale/CRM integration |
| `sale_subscription_column_fix` | Fix subscription column display |

---

### Integration & Bridge Modules (root level)

| Module | Purpose |
|---|---|
| `integration_bridge_core` | Core webhook/API bridge (n8n, external systems) |
| `chatwoot_evolution_error_bridge` | Chatwoot + Evolution WhatsApp error bridge |
| `openclaw_gateway` | OpenClaw AI gateway |
| `pinecone_connector` | Pinecone vector database connector |
| `odoo_twilio_sms` | Twilio SMS integration |

---

### Quarantine — `_quarantine/` (disabled/broken modules)

| Module | Reason |
|---|---|
| `base_address_extended_fix` | Address fix incompatible with current setup |
| `freezoner_field_migration` | Field migration — handled separately |
| `hr_attendance_geofence` | Geofence attendance — conflicts |
| `hr_attendance_photo_geolocation` | Photo geolocation — not deployed |
| `ks_curved_backend_theme_enter` | Theme — removed |
| `odoo_attendance_user_location` | Duplicate of hr_attendance_location |
| `payment_stripe_checkout` | Replaced by stripe_fee_extension |

---

### Odoo 18 Compatibility Fixes — `odoo18_fixes/`

| Item | Purpose |
|---|---|
| `hr_us_payroll_stub` | Stub to satisfy US payroll dependency |
| `stock_warehouse_xmlid_fix` | Fix stock warehouse XML IDs |
| `kanban_card_migration` | Kanban card view migration for Odoo 18 |
| `not_null_fixes/` | SQL scripts for NOT NULL constraint issues |
| `2026-01-03_compatibility_fixes/` | Jan 2026 compatibility batch (deadlock cron, missing attributes) |

---

## Dependency Query Guide

When asked "what depends on module X?", check these reverse dependencies:

| Module | Depended on by |
|---|---|
| `freezoner_custom` | `compliance_cycle`, `freezoner_sale_approval`, `freezoner_password`, `bwa_survey`, `sales_commission`, `project_custom` |
| `compliance_cycle` | `project_custom`, `partner_custom` |
| `partner_custom` | `partner_custom_fields`, `project_custom` |
| `crm_log` | `compliance_cycle` |
| `hr_employee_custom` | `crm_log` |
| `base_legal_types` | `freezoner_custom`, `partner_custom`, `project_custom` |
| `partner_organization` | `compliance_cycle`, `client_documents` |
| `cabinet_directory` | `freezoner_custom` |
| `client_documents` | `crm_log`, `project_custom` |
| `report_xlsx` | `crm_report` |

## Model Location Guide

When asked "which module has model X?":

| Model `_name` | Module |
|---|---|
| `legal.type` | `base_legal_types` |
| `partner.stage` | `partner_custom` |
| `shareholder.data` | `partner_custom` |
| `shareholder.data.position` | `partner_custom` |
| `license.activity` | `partner_custom` |
| `compliance.record` (and variants) | `compliance_cycle` |
| `business.structure` | `compliance_cycle` |
| `crm.wizard` | `crm_log` |
| `crm.call.wizard` | `crm_log` |

## Workflow Cross-Reference

For questions about **how a business process works** (stages, transitions, validations), use the `workflow-map` skill instead of or in addition to this skill:

| Process | Skill to use |
|---|---|
| CRM lead stages and what's required to advance | `/workflow-map` |
| Compliance onboarding risk scoring | `/workflow-map` |
| Project/task lifecycle | `/workflow-map` |
| Sale order approval flow | `/workflow-map` |
| SOV financial computation chain | `/workflow-map` |
| Which module owns a model | `/explore-modules` (this skill) |
| What depends on module X | `/explore-modules` (this skill) |

## Key Field Lookup

When asked "where is field X defined?":

| Field | Model | Module |
|---|---|---|
| `lead_ref` | `crm.lead` | `crm_log` |
| `service` | `crm.lead` | `crm_log` |
| `customer_status` | `crm.lead` | `crm_log` |
| `business_proposal` | `crm.lead` | `crm_log` |
| `stage_id` (partner) | `res.partner` | `partner_custom` |
| `shareholder_ids` | `res.partner` | `partner_custom` |
| `trade_license_*` | `res.partner` | `partner_custom` |
| `corporate_tax` | `res.partner` | `partner_custom` |
| `approval_request_ids` | `sale.order` | `freezoner_sale_approval` |
| `payment_state` (task) | `project.task` | `freezoner_custom` |
| `all_milestone_id` | `project.task` | `project_custom` |
| `checkpoint_ids` | `project.task` | `project_custom` |
| `compliance_state` | `crm.lead` | `compliance_cycle` |
| `initial_risk_scoring` | `initial.client.onboarding` | `compliance_cycle` |
| `commission_attribute` | `sale.sov` | `freezoner_custom` |
| `joining_date` | `hr.employee.base` | `hr_employee_custom` |
| `parent_partner_ids` | `res.partner` | `partner_organization` |

## Workflow

1. Read the question and identify the module or model name
2. Look it up in the registry above
3. If more detail is needed (e.g. exact field names, full model code), read the source file directly
4. For workflow/stage questions, use `/workflow-map`
5. For questions about enhancement plans, read `module_enhancement_plans/`
6. For questions about past errors, read `error_fixes/ERROR_XXX_*/`
