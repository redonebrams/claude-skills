---
name: integrate-sage-x3
description: >
  Guide for integrating Sage X3 V12 with Odoo 18 using the sage_x3_connector module architecture.
  Use this skill whenever the user wants to: sync data between Odoo and Sage X3, add a new entity to the
  Sage connector (new model sync), build Sage X3 API calls, handle OAuth authentication with X3,
  create field mappings between Odoo and X3, implement sync workflows, add conflict resolution,
  or extend the existing connector with new capabilities. Also trigger when the user mentions
  "Sage", "X3", "SageX3", "sage connector", "ERP sync", "bidirectional sync", or "SOH/SIH/POH"
  in the context of Odoo development.
---

# Sage X3 V12 Integration in Odoo 18

## Architecture Overview

The Sage X3 connector follows a layered architecture:

```
Business Module (sale.order, res.partner, etc.)
    ↓ write() / state change triggers _sage_enqueue()
Sync Queue (sage.sync.queue) — async, cron-processed
    ↓ process_queue() every 5 min, max 50 items
Workflow (orchestration: validate → dependencies → sync → log)
    ↓
Sync Service (BaseSkill: read → validate → transform → API call)
    ↓
SageX3Client (OAuth, rate limiting, retries, HTTP)
    ↓
Sage X3 REST API (V12)
```

Each layer has a single responsibility. Business models never call the API directly — they enqueue, and the queue processor runs the workflow.

## The sage_x3_connector Module

This is a single Odoo module (`sage_x3_connector`) that provides everything:

```
sage_x3_connector/
├── models/
│   ├── sage_sync_mixin.py       # AbstractModel mixin (x3_id, sync status)
│   ├── sage_sync_queue.py       # Queue model with retry logic
│   ├── sage_sync_log.py         # Audit trail
│   ├── sage_field_mapping.py    # Configurable field mappings
│   ├── res_config_settings.py   # Settings UI
│   ├── res_partner.py           # Partner sync hooks
│   ├── product_template.py      # Product sync hooks
│   ├── sale_order.py            # SO sync hooks
│   ├── purchase_order.py        # PO sync hooks
│   ├── account_move.py          # Invoice sync hooks
│   └── account_payment.py       # Payment sync hooks
├── services/
│   ├── sage_client.py           # HTTP client with OAuth + retries
│   ├── base_skill.py            # Abstract base for sync services
│   ├── data_transformer.py      # Field mapping + data transforms
│   ├── conflict_handler.py      # Conflict resolution strategies
│   ├── customer_sync.py         # CustomerSync(BaseSkill)
│   ├── product_sync.py          # ProductSync(BaseSkill)
│   ├── sales_order_sync.py      # SalesOrderSync(BaseSkill)
│   ├── purchase_order_sync.py   # PurchaseOrderSync(BaseSkill)
│   ├── invoice_sync.py          # InvoiceSync(BaseSkill)
│   ├── payment_sync.py          # PaymentSync(BaseSkill)
│   └── inventory_sync.py        # InventorySync(BaseSkill)
├── workflows/
│   ├── customer_workflow.py     # Orchestrates customer sync
│   ├── sales_order_workflow.py  # + auto-syncs dependencies
│   ├── invoice_workflow.py      # + auto-posts in X3
│   ├── payment_workflow.py      # + allocates to invoices
│   ├── reconciliation_workflow.py  # Daily divergence check
│   └── bulk_import_workflow.py  # Batch import
├── controllers/
│   └── sage_api.py              # REST endpoints + webhooks
├── views/                       # Dashboard, queue, logs, mappings, settings
├── data/                        # Field mappings, status mappings, cron jobs
└── security/                    # Access control
```

## Step 1: The Sage X3 API Client

The `SageX3Client` handles all HTTP communication with X3:

- **OAuth**: Client credentials flow (`/auth/token`), auto-refreshes 60s before expiry
- **Rate limiting**: Configurable req/s (default 10)
- **Retries**: Exponential backoff `(2^n) * 2` seconds, max 3 attempts
- **Auto re-auth**: Retries with fresh token on 401
- **API path**: `/api/1/{pool}/{endpoint}` (e.g., `/api/1/SEED/CUSTOMER`)

Key principle: the client is a plain Python class (not an Odoo model) — it takes `env` to read config params and is instantiated per-request.

```python
class SageX3Client:
    def __init__(self, env):
        params = env['ir.config_parameter'].sudo()
        self.base_url = params.get_param('sage.x3.base_url')
        self.client_id = params.get_param('sage.x3.client_id')
        self.client_secret = params.get_param('sage.x3.client_secret')
        self.pool = params.get_param('sage.x3.pool') or 'SEED'
        # ...
```

### X3 Entity Endpoints

| Entity | X3 Endpoint | Key Field |
|--------|-------------|-----------|
| Customer | `CUSTOMER` | `BPCNUM_0` |
| Product | `PRODUCT` | `ITMREF_0` |
| Sales Order | `SOH` | `SOHNUM_0` |
| Purchase Order | `POH` | `POHNUM_0` |
| Invoice | `SIH` | `NUM_0` / `SIVNUM_0` |
| Payment | `PAYMENT` | `NUM_0` |
| Inventory/Stock | `STOCK` | `STOFCY_0` |

## Step 2: The Sync Mixin

Every Odoo model that syncs with X3 inherits this AbstractModel mixin:

```python
class SageSyncMixin(models.AbstractModel):
    _name = 'sage.sync.mixin'

    x3_id = fields.Char("Sage X3 ID", copy=False, index=True)
    x3_last_sync = fields.Datetime("Last X3 Sync", copy=False, readonly=True)
    x3_sync_status = fields.Selection([
        ('synced', 'Synced'),
        ('pending', 'Pending'),
        ('error', 'Error'),
        ('conflict', 'Conflict'),
    ], copy=False, readonly=True)
```

Business models inherit it:

```python
class ResPartner(models.Model):
    _inherit = ['res.partner', 'sage.sync.mixin']
```

## Step 3: The BaseSkill Pattern

Every sync service extends `BaseSkill` and implements three required methods:

```python
class BaseSkill(ABC):
    entity_type = None   # 'customer', 'product', etc.
    odoo_model = None    # 'res.partner', 'sale.order', etc.
    x3_endpoint = None   # 'CUSTOMER', 'SOH', etc.

    @abstractmethod
    def _read_odoo_record(self, record):
        """Extract relevant fields from an Odoo record into a flat dict."""
        pass

    @abstractmethod
    def validate(self, data, direction):
        """Return a list of error strings. Empty = valid."""
        pass

    @abstractmethod
    def transform(self, data, direction):
        """Transform data between Odoo and X3 formats.
        direction: 'to_x3' or 'from_x3'"""
        pass

    def detect_duplicate(self, payload):
        """Optional: check if record already exists in X3. Return X3 record or None."""
        return None
```

The base class provides `sync_to_x3(record)` and `sync_from_x3(x3_id)` which call these methods in sequence.

### Example: Adding a new entity sync

To sync a new entity (e.g., `stock.warehouse`):

```python
# services/warehouse_sync.py
class WarehouseSync(BaseSkill):
    entity_type = 'warehouse'
    odoo_model = 'stock.warehouse'
    x3_endpoint = 'FACILITY'  # X3 endpoint for sites/facilities

    def _read_odoo_record(self, record):
        return {
            'id': record.id,
            'name': record.name,
            'code': record.code,
            'city': record.partner_id.city or '',
        }

    def validate(self, data, direction):
        errors = []
        if direction == 'to_x3' and not data.get('code'):
            errors.append('Warehouse code is required')
        return errors

    def transform(self, data, direction):
        if direction == 'to_x3':
            return {
                'FCY_0': data.get('code', ''),
                'FCYNAM_0': data.get('name', ''),
                'CTY_0': data.get('city', ''),
            }
        else:
            return {
                'name': data.get('FCYNAM_0', ''),
                'code': data.get('FCY_0', ''),
            }
```

## Step 4: The DataTransformer

Handles field-level transformations using configurable mappings stored in `sage.field.mapping`:

**Transform types:**
- `direct` — copy as-is
- `format` — date/float/string formatting (JSON rule: `{"format": "date", "pattern": "%Y-%m-%d"}`)
- `lookup` — status/UOM conversion tables (JSON rule: `{"lookup": "uom_mapping"}`)
- `compute` — phone normalization, boolean conversion (JSON rule: `{"compute": "phone_normalize"}`)

**Built-in lookups:**

| Lookup | Odoo → X3 | X3 → Odoo |
|--------|-----------|-----------|
| UOM | Units→UN, kg→KG, L→L, Dozen(s)→DZ | Reverse |
| Order status | draft→1, sale→2, done→3, cancel→4 | Reverse |
| Invoice status | draft→1, posted→2, cancel→3 | Reverse |
| Phone (Morocco) | 0XXXXXXXXX→+212XXXXXXXXX | Reverse |

## Step 5: Workflows (Orchestration)

Workflows sit between the queue processor and the sync services. They handle dependency resolution, error routing, and logging:

```python
class SalesOrderWorkflow:
    def sync_to_x3(self, record):
        # 1. Ensure customer is synced first
        if not record.partner_id.x3_id:
            CustomerWorkflow(self.env, self.client).sync_to_x3(record.partner_id)

        # 2. Ensure products are synced
        for line in record.order_line:
            if not line.product_id.product_tmpl_id.x3_id:
                ProductWorkflow(self.env, self.client).sync_to_x3(line.product_id.product_tmpl_id)

        # 3. Now sync the order itself
        self.skill.sync_to_x3(record)
```

This automatic dependency resolution means you can sync a sales order and the workflow will auto-sync the customer and products if they haven't been synced yet.

## Step 6: The Sync Queue

The queue decouples business operations from API calls:

```
States: pending → processing → done
                     ↓
                  failed (after max_retries=3)
                     ↓
                  conflict (if divergence detected)
```

**Retry logic**: Exponential backoff — `(2^retry_count) * 2` seconds between attempts.

**Cron**: Runs every 5 minutes, processes up to 50 items per run, ordered by priority (0=highest) then create_date.

**Enqueue from business models** (in `write()` override):

```python
def write(self, vals):
    res = super().write(vals)
    # Critical: check sync is enabled and prevent duplicates
    sync_enabled = self.env['ir.config_parameter'].sudo().get_param('sage.x3.sync_enabled')
    if sync_enabled and self._should_sync(vals):
        for record in self:
            existing = self.env['sage.sync.queue'].search([
                ('model_name', '=', self._name),
                ('record_id', '=', record.id),
                ('state', 'in', ('pending', 'processing')),
            ], limit=1)
            if not existing:
                self.env['sage.sync.queue'].create({
                    'model_name': self._name,
                    'record_id': record.id,
                    'direction': 'to_x3',
                })
    return res
```

**Important**: Never call `self.write()` inside a `write()` override to update sync status — use `super(ClassName, self).write()` or `record.sudo().write()` to avoid recursion.

## Step 7: Conflict Resolution

The `ConflictHandler` supports four strategies (configurable per entity or globally via `sage.x3.conflict_strategy`):

| Strategy | Behavior |
|----------|----------|
| `last_write_wins` | Compare `write_date` vs `UPDDAT_0`, latest wins |
| `odoo_wins` | Always keep Odoo data |
| `x3_wins` | Always keep X3 data |
| `manual` | Queue as 'conflict' state, notify admin for review |

The **daily reconciliation** workflow (cron at 2:00 AM) compares Odoo vs X3 for all synced records and uses the conflict handler to resolve divergences.

## Step 8: Webhooks (X3 → Odoo)

The connector exposes a webhook endpoint for X3 to push changes:

```
POST /api/sage/webhook
{
    "event": "updated",
    "entity": "customer",
    "id": "CUST00042"
}
```

This creates a high-priority queue item with `direction='from_x3'`, triggering a sync from X3 to Odoo on the next cron run.

## Step 9: Configuration (Settings UI)

All config lives in `ir.config_parameter` with prefix `sage.x3.`:

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `sage.x3.base_url` | X3 REST API base URL | — |
| `sage.x3.client_id` | OAuth client ID | — |
| `sage.x3.client_secret` | OAuth client secret | — |
| `sage.x3.pool` | X3 folder/pool name | `SEED` |
| `sage.x3.language` | API language code | `FRA` |
| `sage.x3.rate_limit` | Requests per second | `10` |
| `sage.x3.max_retries` | Max retry attempts | `3` |
| `sage.x3.sync_enabled` | Master toggle | `False` |
| `sage.x3.conflict_strategy` | Conflict resolution | `last_write_wins` |

These are exposed in Settings > Sage X3 Integration via `res.config.settings`.

## Adding a New Entity to the Connector — Checklist

1. **Sync Service**: Create `services/my_entity_sync.py` extending `BaseSkill` with `_read_odoo_record()`, `validate()`, `transform()`, and optionally `detect_duplicate()`
2. **Workflow**: Create `workflows/my_entity_workflow.py` with dependency resolution and error handling
3. **Model inheritance**: In `models/my_model.py`, inherit both the Odoo model and `sage.sync.mixin`, override `write()` to enqueue
4. **Register in queue processor**: Add the workflow to the `workflow_map` in `sage_sync_queue.py:_process_single_item()`
5. **Field mappings**: Add entries in `data/sage_field_mappings.xml` for configurable field-level mappings
6. **Views**: Add `x3_id`, `x3_sync_status`, `x3_last_sync` to the model's form view
7. **Security**: Add access rights for the new model in `security/ir.model.access.csv`
8. **Entity map**: Add the X3 endpoint name to `SageX3Client.get_entity_url()` entity_map

## Common Sage X3 V12 API Patterns

### Reading a record
```python
client.get("CUSTOMER('CUST001')")
```

### Creating a record
```python
client.post("CUSTOMER", {"BPCNAM_0": "John", "CRY_0": "MA", ...})
```

### Updating a record
```python
client.put("CUSTOMER('CUST001')", {"BPCNAM_0": "John Updated"})
```

### Querying with filter
```python
client.get("CUSTOMER", params={"where": "CRN_0 eq '12345'"})
# Response: {"$resources": [...]} or {"value": [...]}
```

### Response structure
X3 V12 returns records in `$resources` or `value` arrays. The key field varies by entity (see entity table above).
