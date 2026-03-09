---
name: integrate-azure-service-bus
description: >
  Guide for integrating Azure Service Bus into any Odoo 18 module for async event-driven messaging.
  Use this skill whenever the user wants to: publish events from Odoo to Azure Service Bus, implement async
  messaging between Odoo and external systems, sync data to a mobile app or microservice, add queue-based
  processing with retry logic, or set up topic/subscription patterns. Also trigger when the user mentions
  "Service Bus", "Azure queue", "Azure topic", "event publishing", "message queue", "async sync",
  or "event-driven" in the context of Odoo development.
---

# Azure Service Bus Integration in Odoo 18

## First-Time Setup: The `cloud_service_config` Module

**Before using Service Bus in any business module**, you need the **`cloud_service_config`** module installed and configured. This is a single shared module that centralizes the configuration for **all** third-party cloud services (Azure Key Vault, Azure Service Bus, Google Cloud Storage) in one Odoo settings screen.

If the module doesn't exist yet in the project, **create it first** — see the `integrate-cloud-services-setup` skill for the full module structure. The Service Bus-related parts it provides:
- Settings fields for: connection string (or Key Vault secret name), default queue name, default topic name, batch size, max retries
- A shared mixin (`cloud.service.mixin`) with `_get_service_bus_client()` and `_publish_event()` methods
- System parameters: `cloud_services.sb_connection_string`, `cloud_services.sb_default_queue`, `cloud_services.sb_default_topic`, `cloud_services.sb_max_retries`

All business modules that publish events should **depend on `cloud_service_config`** and inherit its mixin — never hardcode connection strings, queue names, or duplicate messaging logic.

## When to Use

Use Azure Service Bus when your Odoo module needs to communicate asynchronously with external systems — mobile apps, microservices, data pipelines. It decouples Odoo from consumers: Odoo publishes events, consumers process them at their own pace.

**Queue vs Topic**:
- **Queue**: Point-to-point. One sender, one receiver. Good for tasks (process this order, send this email).
- **Topic + Subscriptions**: Pub/sub. One sender, many receivers. Good for events (product updated — notify mobile app AND analytics AND search index).

## Dependencies

```
azure-servicebus>=7.8.0
azure-identity>=1.12.0      # if using Key Vault for connection strings
azure-keyvault-secrets       # if using Key Vault for connection strings
```

## Step 1: Imports with Graceful Fallback

```python
import json
import logging
from datetime import datetime

_logger = logging.getLogger(__name__)

try:
    from azure.servicebus import ServiceBusClient, ServiceBusMessage
    AZURE_SB_AVAILABLE = True
except ImportError:
    _logger.warning("azure-servicebus not installed. Service Bus features disabled.")
    AZURE_SB_AVAILABLE = False
```

## Step 2: Connection String

The connection string is the key to accessing Service Bus. Never hardcode it — retrieve from your secrets manager or configuration.

```python
def _get_service_bus_client(self):
    """Create a ServiceBusClient from a securely stored connection string."""
    if not AZURE_SB_AVAILABLE:
        return None
    try:
        # Adapt this to your config strategy:
        #   self._get_keyvault_secret("my-sb-connection")
        #   self.env['ir.config_parameter'].sudo().get_param('azure.servicebus.connection')
        #   os.environ.get('AZURE_SERVICEBUS_CONNECTION')
        connection_string = self._get_connection_string()

        if not connection_string:
            _logger.error("Service Bus connection string not configured")
            return None

        return ServiceBusClient.from_connection_string(connection_string)

    except Exception as e:
        _logger.error(f"Error creating Service Bus client: {e}")
        return None
```

## Step 3: Publishing Messages to a Queue

The basic pattern for sending a single event:

```python
def _publish_event(self, queue_name, event_type, data, session_id=None):
    """Publish a JSON event to an Azure Service Bus queue.

    Args:
        queue_name: Target queue name
        event_type: Event identifier (e.g. "order_created")
        data: Dict payload
        session_id: Optional session ID for ordered delivery
    Returns:
        True on success, False on failure
    """
    if not AZURE_SB_AVAILABLE:
        _logger.warning("Service Bus not available - event not published")
        return False

    try:
        client = self._get_service_bus_client()
        if not client:
            return False

        envelope = {
            "event_type": event_type,
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "source": "odoo",
            "data": data
        }

        message = ServiceBusMessage(
            body=json.dumps(envelope),
            content_type="application/json"
        )

        # Session ID: ensures ordered delivery for messages about the same entity.
        # Required if the queue is session-enabled.
        if session_id:
            message.session_id = session_id

        # Application properties: metadata for server-side filtering/routing
        message.application_properties = {
            "event_type": event_type,
            "source_system": "odoo"
        }

        # Subject: enables subscription rules to filter by event
        message.subject = event_type

        with client.get_queue_sender(queue_name=queue_name) as sender:
            sender.send_messages(message)

        _logger.info(f"Published {event_type} to queue '{queue_name}'")
        return True

    except Exception as e:
        _logger.error(f"Failed to publish {event_type}: {e}", exc_info=True)
        return False
```

## Step 4: Publishing Messages to a Topic

Same idea, but use `get_topic_sender` instead of `get_queue_sender`. Topics support multiple subscriptions, each with optional filter rules:

```python
def _publish_to_topic(self, topic_name, event_type, data, session_id=None, properties=None):
    """Publish to an Azure Service Bus topic."""
    if not AZURE_SB_AVAILABLE:
        return False

    try:
        client = self._get_service_bus_client()
        if not client:
            return False

        envelope = {
            "event_type": event_type,
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "source": "odoo",
            "data": data
        }

        message = ServiceBusMessage(
            body=json.dumps(envelope),
            content_type="application/json"
        )
        if session_id:
            message.session_id = session_id
        message.subject = event_type
        message.application_properties = properties or {"source_system": "odoo"}

        with client.get_topic_sender(topic_name=topic_name) as sender:
            sender.send_messages(message)

        return True

    except Exception as e:
        _logger.error(f"Failed to publish to topic '{topic_name}': {e}", exc_info=True)
        return False
```

## Step 5: Batch Sending

When publishing many messages at once (bulk import, cron job), batch them for efficiency:

```python
def _publish_batch(self, queue_name, messages_data):
    """Send multiple messages in a single batch.

    Args:
        messages_data: List of dicts with keys: event_type, data, session_id
    """
    if not AZURE_SB_AVAILABLE or not messages_data:
        return False

    try:
        client = self._get_service_bus_client()
        if not client:
            return False

        messages = []
        for item in messages_data:
            envelope = {
                "event_type": item["event_type"],
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "source": "odoo",
                "data": item["data"]
            }
            msg = ServiceBusMessage(
                body=json.dumps(envelope),
                content_type="application/json",
                session_id=item.get("session_id")
            )
            msg.subject = item["event_type"]
            messages.append(msg)

        with client.get_queue_sender(queue_name=queue_name) as sender:
            sender.send_messages(messages)

        _logger.info(f"Batch sent {len(messages)} messages to '{queue_name}'")
        return True

    except Exception as e:
        _logger.error(f"Batch send failed: {e}", exc_info=True)
        return False
```

## Step 6: Session IDs (Ordered Delivery)

If your queue is **session-enabled**, every message must have a `session_id`. Messages with the same session ID are delivered in FIFO order. This is critical when you have create → update → delete sequences for the same entity.

Convention: use `{entity_type}-{entity_id}` as the session ID:

```python
session_id = f"order-{order.id}"       # All events for order 42 processed in order
session_id = f"product-{product.id}"   # All events for product 7 processed in order
session_id = f"settings"               # Singleton session for global config
```

## Step 7: Hook into Odoo Model Lifecycle

Publish events automatically when records change:

```python
from odoo import models, fields, api

class MyModel(models.Model):
    _name = 'my.model'

    name = fields.Char()

    def _build_event_payload(self):
        """Build the data dict for Service Bus events."""
        return {
            "id": self.id,
            "name": self.name,
            # ... add fields the consumer needs
        }

    @api.model_create_multi
    def create(self, vals_list):
        records = super().create(vals_list)
        for record in records:
            record._publish_event(
                "my-queue", "my_model_created",
                record._build_event_payload(),
                session_id=f"my_model-{record.id}"
            )
        return records

    def write(self, vals):
        res = super().write(vals)
        for record in self:
            record._publish_event(
                "my-queue", "my_model_updated",
                record._build_event_payload(),
                session_id=f"my_model-{record.id}"
            )
        return res

    def unlink(self):
        for record in self:
            record._publish_event(
                "my-queue", "my_model_deleted",
                {"id": record.id},
                session_id=f"my_model-{record.id}"
            )
        return super().unlink()
```

## Step 8: Async Queue Pattern (For High-Volume Events)

For entities with frequent changes (products, stock), sending a message on every `write()` can be too heavy. Instead, queue the sync intent in Odoo and process in batches via cron.

### Queue Model

```python
class SyncQueue(models.Model):
    _name = 'my.sync.queue'
    _description = 'Async Service Bus Sync Queue'

    record_id = fields.Integer(required=True, index=True)
    model_name = fields.Char(required=True)
    action = fields.Selection([
        ('create', 'Create'), ('update', 'Update'), ('delete', 'Delete')
    ], required=True)
    state = fields.Selection([
        ('pending', 'Pending'), ('processing', 'Processing'),
        ('done', 'Done'), ('failed', 'Failed')
    ], default='pending', index=True)
    retry_count = fields.Integer(default=0)
    max_retries = fields.Integer(default=3)
    error_message = fields.Text()
    priority = fields.Integer(default=5)
    created_date = fields.Datetime(default=fields.Datetime.now)
```

### Cron Processor

```python
@api.model
def _cron_process_queue(self):
    """Process pending sync items in batches."""
    pending = self.sudo().search([
        ('state', '=', 'pending'),
        ('retry_count', '<', 3)
    ], order='priority asc, created_date asc', limit=50)

    if not pending:
        return

    for item in pending:
        item.state = 'processing'

    try:
        # Build and send messages in batch
        # ... on success: item.state = 'done'
        # ... on failure: increment retry_count, reset to 'pending' or 'failed'
        pass
    except Exception as e:
        self._handle_failure(pending, str(e))
```

### Retry Logic

```python
def _handle_failure(self, items, error_message):
    for item in items:
        item.retry_count += 1
        if item.retry_count >= item.max_retries:
            item.write({'state': 'failed', 'error_message': error_message})
        else:
            item.write({'state': 'pending', 'error_message': error_message})
```

### Cron XML

```xml
<record id="ir_cron_process_sync_queue" model="ir.cron">
    <field name="name">Process Service Bus Sync Queue</field>
    <field name="model_id" ref="model_my_sync_queue"/>
    <field name="state">code</field>
    <field name="code">model._cron_process_queue()</field>
    <field name="interval_number">2</field>
    <field name="interval_type">minutes</field>
    <field name="active">True</field>
</record>
```

### Direct + Fallback Pattern

For the best of both worlds — fast sync for single changes, queued for bulk or failures:

```python
def _sync_smart(self, action):
    """Direct sync for single records, queue for bulk or on failure."""
    is_bulk = self.env.context.get('import_file') or len(self) > 5

    if is_bulk:
        self._enqueue_sync(action)
    else:
        try:
            self._publish_event("my-queue", f"entity_{action}", self._build_event_payload())
        except Exception:
            _logger.warning("Direct sync failed, falling back to queue")
            self._enqueue_sync(action, priority=1)
```

## Step 9: Message Envelope Design

Keep a consistent envelope across all your events. This makes the consumer's job easier:

```python
{
    "event_type": "order_created",           # What happened
    "timestamp": "2026-03-06T14:30:22Z",     # When (always UTC)
    "source": "odoo",                         # Who sent it
    "data": {                                 # Entity-specific payload
        "id": 42,
        "name": "SO/2026/001",
        ...
    }
}
```

The `event_type` + `source` + `timestamp` pattern lets consumers handle idempotency, ordering, and routing without parsing the payload.

## Configuration Checklist for a New Module

1. **Dependencies**: Add `azure-servicebus` to requirements
2. **Imports**: Use try/except with `AZURE_SB_AVAILABLE` flag
3. **Connection string**: Store securely (Key Vault, env var, or `ir.config_parameter`) — never hardcode
4. **Queue/Topic names**: Store in configuration, not code
5. **`_publish_event()` method**: Add to your model
6. **Payload builder**: Implement `_build_event_payload()` with only the fields consumers need
7. **Session IDs**: Use `{entity_type}-{id}` if the queue is session-enabled
8. **Lifecycle hooks**: Override `create()`/`write()`/`unlink()`
9. **High-volume**: Add a sync queue model + cron for batch processing
10. **Security file**: Add `ir.model.access.csv` entry for the sync queue model

## Common Pitfalls

- **Session-enabled queues require session_id**: If the queue has sessions enabled, every message **must** have a `session_id` or sending will fail.
- **Connection string in code**: Never. Use Key Vault, env vars, or Odoo system parameters.
- **Blocking on send**: `sender.send_messages()` is synchronous. For high-frequency writes, use the async queue pattern instead of blocking Odoo requests.
- **Forgetting `super()`**: When overriding `create()`/`write()`/`unlink()`, always call `super()` — Service Bus publishing is a side effect, not a replacement for Odoo logic.
- **Missing `content_type`**: Set `content_type="application/json"` so consumers know how to parse the body.
- **No error handling on `unlink()`**: If Service Bus is down, `unlink()` should still succeed. Publish in a try/except and let the record be deleted regardless.
