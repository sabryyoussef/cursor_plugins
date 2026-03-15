---
name: workflow-map
description: Map, explain, and document all Freezoner Odoo 18 business workflows — CRM stages, compliance onboarding, project/task lifecycle, sale approval, and partner pipeline. Use to understand how a process works, debug stuck records, or document workflows for new team members.
---

# Workflow Map

## Trigger

Use when you need to:
- Understand how a business process flows through the system
- Debug a record stuck in a wrong stage/state
- Find which validations are blocking a stage transition
- Document a workflow for a new developer or user
- Understand what fields are required at each stage

---

## Workflow 1 — CRM Lead Pipeline

**Module:** `crm_log` (Level 3) + `compliance_cycle` (Level 4)
**Model:** `crm.lead` (`_inherit`)
**Location:** `_beshoy/level_3_complex_deps/crm_log/models/crm.py`

### Stages & Transitions

```
New
 ├─→ Proposal Sent      requires: sent mail + call activity + attachments
 ├─→ Negotiation        sets date_conversion = today
 ├─→ Invoice Sent       requires: sent mail + call activity + attachments + ≥1 proforma invoice
 ├─→ Full Payment       requires: ≥1 proforma invoice
 └─→ Partial Payment Collected

Negotiation
 └─→ (any next stage via crm.wizard)

Won / Lost               (native Odoo stages)
```

### Stage Transition Validations

Every stage move goes through `action_stage()` → opens `crm.wizard` → `wizard.submit()`:

| From | To | Required |
|---|---|---|
| New | Proposal Sent | `mail.mail` sent, `mail.activity` of type "Call", `ir.attachment` on record |
| New | Invoice Sent | Same as above + `quotation_count ≥ 1` |
| Any | Negotiation | Sets `date_conversion = today` automatically |
| Any | (any) | `_validate_stage_requirements()` checks: Heat Level, Revenue, Closing Date, Quotation |

### Key Fields Added by crm_log

| Field | Type | Purpose |
|---|---|---|
| `referred_id` | Many2one `res.partner` | Internal referral tracking |
| `service` | Selection | Service the lead is interested in (Business Setup, Freelance, Bank, etc.) |
| `customer_status` | Selection | approved / approved_reservations / deferred / non_responsive |
| `lead_ref` | Char | Auto-generated reference number (sequence) |
| `employee_id` | Many2one `hr.employee` | Assigned salesperson as employee |
| `business_proposal` | Many2one `documents.document` | Attached proposal document |
| `is_quotation_expired` | Computed Boolean | True if `date_closed` > 6 months ago |
| `phone` | Computed Char | Built from `country_code` + `custom_phone` using `phonenumbers` library |
| `priority` | Selection | Heat level: Low / Medium / High / Very High |

### Compliance Lead Extension (compliance_cycle)

CRM leads can also be **compliance records** (`is_compliance = 'compliance'`):

```
draft (Pro-forma Invoice)
  → submit
  → sent (Pro-forma Invoice Sent)
  → confirm (Pro-forma Confirmed)
  → cancel
```

Actions: `action_draft`, `action_submit`, `action_send`, `action_submit`, `action_cancel`, `action_confirm`

---

## Workflow 2 — Initial Client Onboarding (KYC / Risk Assessment)

**Module:** `compliance_cycle` (Level 4)
**Model:** `initial.client.onboarding`
**Location:** `_beshoy/level_4_high_level/compliance_cycle/models/onboarding.py`

### States

```
new
 → submitted     (action_submit)
 → validated     (action_validated)
 → secondary     (action_secondary — secondary review required)
 → approved      (action_approved)
```

### Risk Scoring System

The onboarding record computes a **risk score** from 8 categories:

| Category | Model | Fields |
|---|---|---|
| Service Risk | `onboarding.service.risk` | assessment_id, scoring_id, rating_id |
| Product Risk | `onboarding.product.risk` | same pattern |
| Client Risk | `onboarding.client.risk` | same pattern |
| Geography Risk | `onboarding.geography.risk` | same pattern |
| PEP Risk | `onboarding.pep.risk` | same pattern |
| Adverse Media Risk | `onboarding.adverse.risk` | same pattern |
| Sanction Risk | `onboarding.sanction.risk` | same pattern |
| Interface Risk | `onboarding.interface.risk` | same pattern |

**Risk Rating Thresholds:**

| Score | Rating |
|---|---|
| ≤ 20 | Low Risk |
| 21 – 32 | Medium Risk |
| 33 – 53 | High Risk |
| > 53 | Very High Risk |

### Key Fields

| Field | Type | Purpose |
|---|---|---|
| `state` | Selection | new / submitted / validated / secondary / approved |
| `partner_id` | Many2one `res.partner` | Client being onboarded |
| `initial_risk_scoring` | Float (computed) | Sum of all 8 category scores |
| `initial_risk_rating` | Char (computed) | Derived from threshold table |
| `final_risk_rating_id` | Many2one | Officer's final rating decision |
| `compliance_recommendation` | Text | Compliance officer recommendation |
| `next_risk_assessment_date` | Date | When to re-assess |
| `document_required_type_ids` | One2many | Required documents checklist |
| `document_ids` | One2many | Submitted documents |
| `is_validated` | Boolean | Validated by compliance officer |
| `is_approved` | Boolean | Final approval flag |

---

## Workflow 3 — Partner Lifecycle Pipeline

**Module:** `partner_custom` (Level 1)
**Model:** `res.partner` (inherited) + `partner.stage`
**Location:** `_beshoy/level_1_base_custom/partner_custom/`

### Stage Structure

Stages are defined in `partner.stage` model, ordered by `sequence`. Seed data in `data/partner_stage_data.xml`.

The partner has a `stage_id` Many2one → `partner.stage`. A **cron job** (`data/stage_automation_cron.xml`) automates stage transitions based on rules.

### Key Partner Fields Added

| Field | Type | Purpose |
|---|---|---|
| `stage_id` | Many2one `partner.stage` | Current pipeline stage |
| `vat` | Extended | VAT number |
| `corporate_tax` | Extended | Corporate tax number (ERROR_161: autopop issue) |
| `trade_license_*` | Multiple | Trade license number, expiry, issuing authority |
| `moa_*` | Multiple | Memorandum of Association details |
| `emirates_id_*` | Multiple | Emirates ID number and expiry |
| `visa_*` | Multiple | Visa details |
| `shareholder_ids` | One2many `shareholder.data` | Shareholders linked to this partner |

---

## Workflow 4 — Sale Order Approval

**Module:** `freezoner_sale_approval` (Level 4)
**Model:** `sale.order` (inherited) + `approval.request`
**Location:** `_beshoy/level_4_high_level/freezoner_sale_approval/models/sale.py`

### Flow

```
Sale Order created
  → salesperson clicks "Request Approval" (action_approve_sale)
  → creates approval.request record:
      - category: approval.category where is_sale=True
      - linked fields: sale_id, partner_id, amount, date, reference
      - product_line_ids: copied from sale.order.line
      - request_status: 'new'
  → Approval goes through native Odoo approvals workflow
  → Smart button on sale order shows approval count / status
```

### Key Fields

| Field | Type | Purpose |
|---|---|---|
| `approval_request_ids` | One2many `approval.request` | All approval requests for this order |

### Configuration Required

- An `approval.category` record must exist with `is_sale = True`
- Without this, `action_approve_sale()` will create no approval request (silent fail)
- Debug tip: check `self.env['approval.category'].search([('is_sale', '=', True)])` in shell

---

## Workflow 5 — Project & Task Lifecycle

**Module:** `freezoner_custom` (Level 3) + `project_custom` (Level 5)
**Models:** `project.project`, `project.task` (both inherited)

### Project States

Defined in `freezoner_custom/data/project_stages.xml`:

```
New (sequence=1)
  → (additional stages loaded from data)
```

Projects use `project.task.type` for task stages. The `project.project` model in `freezoner_custom` adds a `state` field and is ordered by state (`_order = 'state'`).

### Task Stage Transition (task.wizard)

**Location:** `freezoner_custom/wizard/task_wizard.py`

```
Task in Stage X
  → salesperson clicks stage-advance button
  → opens task.wizard
  → wizard validates requirements
  → sends email notification on advance
  → moves task to next stage
```

### Key Task Fields (freezoner_custom)

| Field | Type | Purpose |
|---|---|---|
| `payment_state` | Selection | not_paid / in_payment / paid / partial / reversed |
| `document_ids` | One2many | Documents attached to task |
| `sale_id` | Many2one `sale.order` | Linked sale order |

### Key Task Fields (project_custom Level 5)

| Field | Type | Purpose |
|---|---|---|
| `all_milestone_id` | Many2one `project.milestone` | Linked milestone |
| `checkpoint_ids` | One2many `project.task.checkpoint` | Checklist items per task |
| `custom_start_date` | Date | Custom start date (separate from Odoo's date_assign) |
| `reached_checkpoint_all_ids` | Many2many `reached.checkpoint` | Completed checkpoints |

### Key Project Fields (project_custom Level 5)

| Field | Type | Purpose |
|---|---|---|
| `hand_partner_*` | Multiple | Handover partner details |
| `compliance_*` | Multiple | Compliance status sync from compliance_cycle |
| `shareholder_ids` | One2many | Shareholders (synced from partner) |
| `visa_application_*` | Multiple | Visa application status |

---

## Workflow 6 — SOV (Statement of Value)

**Module:** `freezoner_custom` (Level 3)
**Model:** `sale.sov`
**Location:** `_beshoy/level_3_complex_deps/freezoner_custom/models/sov.py`

### Purpose

Tracks financial breakdown per sale order line: revenue, planned expenses, profit, tax, net achievement.

### Computed Fields Chain

```
qty × unit_price  →  revenue (store=True)
qty × unit_cost   →  planned_expenses (store=True)
revenue - planned_expenses  →  profit (store=True)
revenue × tax_rate  →  tax (store=True)
profit - tax  →  net (store=True)
```

### Commission Integration

`commission_attribute` Selection field links SOV lines to `sales_commission` module calculations.

---

## Debugging a Stuck Record

When a record is stuck in a wrong stage or won't advance:

### 1. Identify the workflow
Use this table to find the right model and module:

| Record type | Model | Module |
|---|---|---|
| CRM Lead | `crm.lead` | `crm_log` |
| Onboarding | `initial.client.onboarding` | `compliance_cycle` |
| Compliance Lead | `crm.lead` (with `is_compliance`) | `compliance_cycle` |
| Partner | `res.partner` | `partner_custom` |
| Sale Order | `sale.order` | `freezoner_sale_approval` |
| Project | `project.project` | `freezoner_custom` / `project_custom` |
| Task | `project.task` | `freezoner_custom` / `project_custom` |

### 2. Check what's blocking the transition
Read the relevant model's validation method and check what conditions are failing:
- CRM: `_validate_stage_requirements()` in `crm_log/models/crm.py`
- Onboarding: state action methods in `compliance_cycle/models/onboarding.py`
- Task: `task_wizard.py` validation logic

### 3. Check the log
```bash
grep -i "ValidationError\|UserError" /opt/odoo/logs/odoo18.log | tail -20
```

### 4. Fix the data via Odoo shell (if needed)
```bash
sudo -u odoo /opt/odoo/venvs/odoo18/bin/python /opt/odoo/src/odoo18/odoo-bin \
  -c /opt/odoo/conf/odoo18.conf shell -d odoo18
```

```python
# Example: force a CRM lead to a specific stage
lead = env['crm.lead'].browse(123)
stage = env['crm.stage'].search([('name', '=', 'Negotiation')], limit=1)
lead.write({'stage_id': stage.id})
env.cr.commit()
```

---

## Workflow Dependency Map

```
Partner Stage Pipeline (partner_custom L1)
    ↓ partner created
CRM Lead Pipeline (crm_log L3)
    ↓ lead won / converted
Initial Client Onboarding (compliance_cycle L4)
    ↓ approved
Project Created (freezoner_custom L3 + project_custom L5)
    ├── Sale Order + SOV (freezoner_custom L3)
    │       └── Sale Approval (freezoner_sale_approval L4)
    └── Tasks with Milestones + Checkpoints (project_custom L5)
```
