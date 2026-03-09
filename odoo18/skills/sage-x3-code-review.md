---
name: sage-x3-code-review
description: >
  Code review checklist specific to the sage_x3_connector module and any code that interacts with Sage X3.
  Use this skill whenever reviewing, modifying, or creating code in the sage_x3_connector module — sync services,
  workflows, models inheriting sage.sync.mixin, field mappings, queue processing, or X3 API interactions.
  Also trigger when the user asks to review Sage-related code, debug sync issues, or validate connector changes
  before deployment.
---

# Sage X3 Connector — Code Review Checklist

## Critical Issues (Blocking — must fix before merge)

### Recursive write() trap
**The #1 bug source in this connector.** If a model inherits `sage.sync.mixin` and overrides `write()` to enqueue a sync, calling `self.write({'x3_sync_status': 'pending'})` inside that override creates an infinite loop.

```python
# WRONG — infinite recursion
def write(self, vals):
    res = super().write(vals)
    self.write({'x3_sync_status': 'pending'})  # calls write() again!
    return res

# CORRECT — bypass the override
def write(self, vals):
    res = super().write(vals)
    super(MyModel, self).write({'x3_sync_status': 'pending'})
    return res

# ALSO CORRECT — in services/workflows, use sudo().write()
record.sudo().write({'x3_sync_status': 'synced', 'x3_last_sync': ...})
```

### Duplicate queue items
Every `_sage_enqueue()` call must check for existing pending/processing items to prevent duplicates:

```python
# WRONG — creates duplicates on rapid writes
self.env['sage.sync.queue'].create({...})

# CORRECT — check first
existing = self.env['sage.sync.queue'].search([
    ('model_name', '=', self._name),
    ('record_id', '=', record.id),
    ('state', 'in', ('pending', 'processing')),
], limit=1)
if not existing:
    self.env['sage.sync.queue'].create({...})
```

### Sync enabled check
Always check the master toggle before enqueueing. Without this, sync items pile up even when the admin has disabled sync:

```python
sync_enabled = self.env['ir.config_parameter'].sudo().get_param('sage.x3.sync_enabled')
if not sync_enabled:
    return
```

### Hardcoded credentials or URLs
No X3 base URL, client_id, client_secret, pool name, or IP address should appear in Python code or XML data files. Everything must come from `ir.config_parameter` (configured in Settings > Sage X3).

## BaseSkill Subclass Rules

Every sync service (`services/*.py`) extending `BaseSkill` must:

| Method | Required | What it does |
|--------|----------|--------------|
| `_read_odoo_record(record)` | Yes | Extract Odoo fields into a flat dict |
| `validate(data, direction)` | Yes | Return list of error strings (empty = valid) |
| `transform(data, direction)` | Yes | Convert between Odoo and X3 field formats |
| `detect_duplicate(payload)` | No | Check if record exists in X3, return X3 record or None |

- `direction` is always `'to_x3'` or `'from_x3'`
- `_read_odoo_record` must handle missing/optional fields with `or ''` / `or 0` — never return `None` values that break JSON serialization
- `validate` for `to_x3` should check Odoo field requirements; for `from_x3` should check X3 field requirements
- `transform` for `to_x3` outputs X3 field names (e.g., `BPCNAM_0`); for `from_x3` outputs Odoo field names (e.g., `name`)
- Do not reimplement OAuth, rate limiting, or retries — `SageX3Client` handles all of that

## Workflow Rules

Workflows (`workflows/*.py`) orchestrate the sync process:

- **Always resolve dependencies first**: Before syncing a sales order, ensure customer and products have `x3_id`. If not, sync them first.
- **Log every outcome**: Success, error, and conflict must all be logged to `sage.sync.log` with duration_ms and workflow_name.
- **Queue errors, don't swallow them**: On failure, update `x3_sync_status` to `'error'` and create/update a queue item with the error message and failed step.
- **Workflow classes are plain Python** (not Odoo models) — they take `env` and `client` in `__init__`.

## Queue & Cron Rules

- Queue items use priority 0-10 (0 = highest priority)
- `process_queue()` processes **max 50 items** per cron run — do not increase without load testing
- Retry uses **exponential backoff**: `(2^retry_count) * 2` seconds
- Max retries default is 3 — configurable per queue item
- After max retries, state goes to `'failed'` (not deleted)
- Cron jobs must NOT use `numbercall` or `doall` — deprecated in Odoo 18. Use `interval_number` + `interval_type` + `active` only
- `self.env.cr.commit()` is used in queue processing to prevent a single failure from rolling back the entire batch — this is intentional, don't remove it

## Field Mapping Rules

- Selection fields must NOT have `default=False` — use a valid selection value like `default='pending'` or omit the default entirely
- Transform rules are JSON — validate they're parseable
- `direction` on mappings must be `'bidirectional'`, `'to_x3'`, or `'from_x3'`
- Mapping `sequence` determines processing order — lower = first
- Hardcoded values in XML mapping data files (IPs, URLs, environment names) are forbidden — defaults must be empty, configured by admin in Settings

## Data Transformation Rules

- **Phone numbers**: Use `DataTransformer.normalize_phone()` for Morocco format (+212 prefix). Don't write custom phone normalization.
- **UOM mapping**: Use the built-in `UOM_ODOO_TO_X3` / `UOM_X3_TO_ODOO` dicts. Add new entries there, don't hardcode in sync services.
- **Status mapping**: Use the built-in `STATUS_ODOO_TO_X3` / `STATUS_X3_TO_ODOO` dicts. Same for invoice statuses.
- **Dates**: X3 returns ISO format with optional `Z` suffix. Always handle both `2026-03-06` and `2026-03-06T14:30:00Z`.

## API Controller Rules

- All endpoints in `controllers/sage_api.py` must validate authentication (JWT Bearer token)
- JSON response format must be consistent: `{"status": "success|error", "message": "...", "data": ...}`
- Input validation on all parameters — never trust incoming JSON
- Webhook endpoint must validate the event/entity/id structure before creating queue items

## Security Rules

- `sage.sync.queue`, `sage.sync.log`, `sage.field.mapping` must have access rights in `ir.model.access.csv`
- Admin: full CRUD. Regular users: read-only on queue and logs
- No `sudo()` without clear justification (accessing config params is OK, bulk-writing records during sync is OK with comment)
- Never log the OAuth `client_secret` or full connection strings — only log safe identifiers

## Odoo 18 Compliance

These apply to all files in the connector:

| Rule | Check |
|------|-------|
| XML views | `<list>` not `<tree>`, `view_mode='list,form'` not `'tree,form'` |
| Cron definitions | No `numbercall`, no `doall` |
| `invisible`/`readonly` | Use simplified expression syntax: `invisible="state != 'draft'"` |
| Imports | `from odoo import api, fields, models, _` |
| Logging | `import logging` + `_logger = logging.getLogger(__name__)` at module level |
| `_description` | Every model class must have `_description` defined |

## Red Flags Table

| Issue | Severity | Action |
|-------|----------|--------|
| `self.write()` inside `write()` override | CRITICAL | Use `super().write()` instead |
| Duplicate queue items (no dedup check) | CRITICAL | Add `search()` before `create()` |
| Missing `sync_enabled` check | CRITICAL | Add before any enqueue |
| Hardcoded X3 URL/credentials | CRITICAL | Move to `ir.config_parameter` |
| `default=False` on Selection field | MAJOR | Use valid value or omit |
| BaseSkill missing required method | MAJOR | Implement all abstract methods |
| No error logging in workflow | MAJOR | Add `sage.sync.log` call |
| `sudo()` without justification | MAJOR | Add comment explaining why |
| Missing input validation on API | MAJOR | Add parameter checks |
| `<tree>` in XML | CRITICAL | Replace with `<list>` |
| `numbercall`/`doall` in cron | CRITICAL | Remove — Odoo 18 |
| No `_description` on model | MINOR | Add description string |
