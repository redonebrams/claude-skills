---
name: odoo18-developer
description: >
  Expert Odoo 18 development skill for building modules, customizing existing apps, writing Python/XML/JS code, 
  configuring OWL components, setting up CI/CD, and navigating Odoo 18-specific APIs and architecture changes.
  Use this skill whenever the user wants to: create or extend an Odoo module, write models/views/controllers, 
  work with Odoo 18's new spreadsheet API, configure multi-company environments, write QWeb templates, 
  set up automated actions, or debug Odoo server errors. Also trigger for any Odoo upgrade from 16/17 → 18, 
  Studio customizations, or questions about Odoo's new AI/LLM integration hooks.
---

# Odoo 18 Developer Skill

## What's New in Odoo 18 (Key for Developers)

### Architecture & Framework
- **OWL 2 (Odoo Web Library)** — All frontend components now fully migrated to OWL 2. No more legacy Widget system.
- **Python 3.12** support required; Python 3.10 minimum.
- **PostgreSQL 15/16** recommended.
- **New `ir.actions.act_url`** improvements with embedded views.
- **`env.ref()` lazy loading** — performance boost for record references.
- **`BaseModel._auto_init()`** hooks revamped for smoother module initialization.

### Models & ORM
```python
# New computed stored field shorthand (Odoo 18)
field_name = fields.Char(compute='_compute_field', store=True, precompute=True)

# New `_sql_constraints` with conflict resolution
_sql_constraints = [
    ('unique_ref', 'unique(reference)', 'Reference must be unique per company'),
]

# Domain new operators: '=?' for optional filters
domain = [('partner_id', '=?', partner_id)]  # skips if partner_id is False

# New: flush_recordset() for partial flushes
self.env['sale.order'].flush_recordset(['state', 'amount_total'])
```

### Views & XML
```xml
<!-- New list view editable improvements -->
<list editable="bottom" multi_edit="1" optional_columns="1">
  <field name="product_id" column_invisible="not parent.show_products"/>
</list>

<!-- New attrs= replacement: invisible/required/readonly as direct attrs -->
<field name="date_end" invisible="state != 'draft'" required="state == 'confirmed'"/>

<!-- Spreadsheet view (NEW in 18) -->
<record model="ir.ui.view" id="view_spreadsheet_pivot">
  <field name="arch" type="xml">
    <spreadsheet/>
  </field>
</record>
```

### New Odoo 18 Modules & APIs

| Module | What's New |
|---|---|
| `mail` | AI-powered message summarization, Thread reactions, Voice messages |
| `account` | AI invoice OCR v2, SEPA pain.008, dynamic payment terms |
| `sale` | Combo products, margin visibility per line |
| `stock` | Lot/SN traceability rewrite, putaway strategy enhancements |
| `mrp` | Shop Floor v2 (tablet UX), Work Center capacity planning |
| `hr_payroll` | Global payroll structure types, multi-period salary simulation |
| `website` | AI page builder, lazy-load assets, cookie consent v2 |
| `spreadsheet` | PIVOT() formula, workbook-level access control |
| `knowledge` | Blocks API, AI article generation |
| `sign` | Certified e-signature providers (IT, FR, BE) |
| `project` | Burndown chart, task dependency graph |
| `helpdesk` | SLA escalation trees, AI ticket categorization |
| `appointments` | Resource-aware scheduling, Teams integration |
| `fleet` | EV fleet tracking, carbon reporting |

### AI / LLM Integration (Odoo 18 Native)

```python
# Using Odoo 18 AI provider abstraction
from odoo.addons.mail.tools.discuss import AIProvider

class MyModel(models.Model):
    _name = 'my.model'

    def generate_summary(self):
        provider = self.env['mail.ai.provider']._get_active_provider()
        result = provider.complete(
            prompt="Summarize the following: " + self.description,
            max_tokens=500,
        )
        self.summary = result['content']
```

```python
# AI action on any model (new ir.actions.ai_action)
{
    'type': 'ir.actions.ai_action',
    'model': 'crm.lead',
    'operation': 'summarize' | 'translate' | 'custom',
    'custom_prompt': "...",
}
```

### OWL 2 Component Patterns

```javascript
// Odoo 18 OWL component with new hooks
import { Component, useState, useService } from "@odoo/owl";
import { registry } from "@web/core/registry";

class MyWidget extends Component {
    static template = "my_module.MyWidget";
    static props = { record: Object, field: String };

    setup() {
        this.state = useState({ loading: false });
        this.orm = useService("orm");
        this.notification = useService("notification");
    }

    async onClick() {
        this.state.loading = true;
        try {
            await this.orm.call("my.model", "my_method", [[this.props.record.resId]]);
            this.notification.add("Done!", { type: "success" });
        } finally {
            this.state.loading = false;
        }
    }
}

registry.category("fields").add("my_widget", { component: MyWidget });
```

### Spreadsheet Integration (Major New API)

```python
# Embed a spreadsheet in your module
class SaleSpreadsheet(models.Model):
    _name = 'sale.spreadsheet'
    _inherit = 'spreadsheet.mixin'

    name = fields.Char(required=True)
    sale_order_ids = fields.Many2many('sale.order')

    def _get_spreadsheet_snapshot(self):
        # Returns spreadsheet JSON state
        return self._spreadsheet_snapshot
```

```javascript
// Add a PIVOT data source from your module
import { SpreadsheetPivot } from "@spreadsheet/pivot/pivot_data_source";

// Register a custom function in spreadsheet
const { functionRegistry } = spreadsheet;
functionRegistry.add("MY.CUSTOM", {
    description: "My custom Odoo data function",
    compute: async (args) => { ... },
    args: [{ name: "model", description: "Odoo model name" }],
    returns: ["NUMBER"],
});
```

### Security & Multi-Company

```python
# New company-dependent field syntax
class Product(models.Model):
    price = fields.Float(company_dependent=True)  # Per-company value (unchanged)
    
    # New: branch-aware records (Odoo 18 branches feature)
    branch_id = fields.Many2one('res.branch', default=lambda self: self.env.branch)
```

### Testing

```python
# Odoo 18 test helpers
from odoo.tests import tagged, HttpCase

@tagged('post_install', '-at_install', 'odoo18')
class TestMyModule(HttpCase):
    def test_tour_my_feature(self):
        self.start_tour('/web', 'my_module.tour_name', login='admin')
    
    def test_with_ai_mock(self):
        with self.mock_ai_provider(response="Mocked AI response"):
            result = self.env['my.model'].generate_summary()
        self.assertEqual(result, "Mocked AI response")
```

## Module Scaffold (Odoo 18 Standard)

```
my_module/
├── __manifest__.py          # version: '18.0.1.0.0'
├── __init__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   └── my_model_views.xml
├── security/
│   ├── ir.model.access.csv
│   └── my_module_security.xml
├── data/
│   └── my_module_data.xml
├── static/
│   ├── src/
│   │   ├── components/      # OWL components
│   │   ├── views/           # Custom view types
│   │   └── js/              # Legacy JS (avoid if possible)
│   └── description/
│       └── icon.png
├── tests/
│   ├── __init__.py
│   └── test_my_model.py
└── wizard/
    └── my_wizard.py
```

```python
# __manifest__.py template
{
    'name': 'My Module',
    'version': '18.0.1.0.0',
    'category': 'Custom',
    'summary': 'Short description',
    'author': 'Your Company',
    'depends': ['base', 'mail'],
    'data': [
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'my_module/static/src/components/**/*.js',
            'my_module/static/src/components/**/*.xml',
            'my_module/static/src/components/**/*.scss',
        ],
    },
    'installable': True,
    'auto_install': False,
    'license': 'LGPL-3',
}
```

## Common Upgrade Issues (16/17 → 18)

| Issue | Fix |
|---|---|
| `attrs=` deprecated | Use `invisible=`, `required=`, `readonly=` directly |
| `Widget` class | Migrate to OWL `Component` |
| `web.legacy_*` assets | Remove; use `web.assets_backend` |
| `fields_view_get()` override | Use `get_views()` instead |
| `_check_recursion()` | Still available but prefer `_check_company()` |
| `ir.sequence.next_by_code()` | Now requires `sequence_date` param |
| `product.pricelist` arch | Pricelists now compute_method-aware |

## Python Conventions & Header

```python
# -*- coding: utf-8 -*-
import logging

from odoo import api, fields, models, _
from odoo.exceptions import ValidationError, UserError

_logger = logging.getLogger(__name__)
```

- Every model must have `_description` defined (not empty)
- Docstrings on all public methods
- `@api.constrains` for business rules validation
- `ensure_one()` before single-record operations
- No unresolved TODO/FIXME in production code

## Security Best Practices

```python
# BAD - never use sudo() without justification
self.sudo().write({'state': 'done'})

# GOOD - document why sudo is needed
# sudo required: writing cross-company record for inter-company sync
self.sudo().write({'state': 'done'})

# ALWAYS ensure_one() before single-record operations
def action_confirm(self):
    self.ensure_one()
    ...

# NEVER log sensitive data
_logger.info("Processing order %s", order.name)  # GOOD
_logger.info("API key: %s", api_key)              # BAD
```

- No `sudo()` without documented justification
- No sensitive data in logs (API keys, credentials, customer data)
- Permission checks before sensitive operations
- No SQL injection via string formatting in ORM calls — always use ORM methods

## Security CSV (ir.model.access.csv)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_manager,1,1,1,1
```

- **Every** custom model must have access rights defined
- Wizards (`TransientModel`) also need access rights
- `model_id:id` format: `model_` + model name with `_` instead of `.`
- No trailing whitespace or missing commas in CSV

## CRON Definitions (Odoo 18)

```xml
<!-- CORRECT Odoo 18 CRON -->
<record id="ir_cron_my_task" model="ir.cron">
    <field name="name">My Scheduled Task</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="code">model._cron_my_task()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">hours</field>
    <field name="active">True</field>
</record>
```

- **No `numbercall`** — deprecated in Odoo 18
- **No `doall`** — deprecated in Odoo 18
- Use only `interval_number` + `interval_type` + `active`
- CRON methods must have try/except + `_logger.exception()` error handling

## XML View Best Practices

- `groups=` on sensitive fields and action buttons
- `widget="badge"` on Selection status fields (in read-only contexts)
- `decoration-*` on lists for visual indicators (e.g., `decoration-danger="state == 'cancel'"`)
- `readonly=` on computed fields displayed in views
- `invisible=` to hide irrelevant fields by context

## Controller Patterns

```python
from odoo import http
from odoo.http import request

class MyController(http.Controller):

    @http.route('/api/my_endpoint', type='json', auth='public', methods=['POST'])
    def my_endpoint(self, **kwargs):
        # 1. Validate input
        if not kwargs.get('required_field'):
            return {'status': 'error', 'message': 'Missing required_field'}

        # 2. Process with error handling
        try:
            result = request.env['my.model'].sudo().search_read(
                [('active', '=', True)],
                ['name', 'state'],
                limit=50, offset=kwargs.get('offset', 0),
            )
        except Exception as e:
            _logger.exception("API error on /api/my_endpoint")
            return {'status': 'error', 'message': str(e)}

        # 3. Consistent JSON response
        return {'status': 'success', 'data': result}
```

- Input validation on all parameters
- Consistent JSON response format (`status`, `message`, `data`)
- Rate limiting considered for public endpoints
- Large response payloads paginated (use `limit`/`offset`)
- Firebase/token validation on authenticated endpoints when applicable

## Debugging Tips

```bash
# Run Odoo 18 with debug assets
./odoo-bin -d mydb --dev=xml,reload,qweb

# Update specific module with log
./odoo-bin -d mydb -u my_module --log-level=debug

# Run specific test
./odoo-bin -d mydb --test-enable --stop-after-init -i my_module \
  --test-tags='/my_module.TestMyModel'
```

## Reference Files

- `references/owl2-patterns.md` — Advanced OWL 2 component patterns
- `references/accounting-api.md` — Odoo 18 accounting move/journal entry API
- `references/rpc-patterns.md` — JSON-RPC and ORM call patterns from frontend
