---
name: integrate-cloud-services-setup
description: >
  Guide for creating the cloud_service_config Odoo 18 module — the single shared module that centralizes
  configuration for Azure Key Vault, Azure Service Bus, and Google Cloud Storage in one Odoo settings screen.
  Use this skill whenever: starting a new project that needs cloud services, the user wants to set up
  the base config module before building business features, or any of the three service-specific skills
  (integrate-azure-key-vault, integrate-azure-service-bus, integrate-google-cloud-storage) reference this
  module and it doesn't exist yet. Also trigger when the user mentions "cloud config module",
  "cloud_service_config", "third-party services setup", or "centralize cloud settings".
---

# cloud_service_config — Unified Cloud Services Module for Odoo 18

## Purpose

This module is the **foundation** that must exist before any business module uses Azure Key Vault, Azure Service Bus, or Google Cloud Storage. It provides:

1. **One settings screen** in Odoo where admins configure all cloud services
2. **One mixin model** (`cloud.service.mixin`) that business modules inherit to get ready-made methods for all three services
3. **One place** for dependencies, imports, and error handling

Business modules never touch cloud SDKs directly — they inherit the mixin and call its methods.

## Module Structure

```
cloud_service_config/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── cloud_service_mixin.py      # Shared mixin with all cloud methods
│   └── res_config_settings.py      # Settings UI fields
├── views/
│   └── res_config_settings_views.xml  # Settings form
├── security/
│   └── ir.model.access.csv
└── data/
    └── ir_config_parameter.xml     # Default values (optional)
```

## __manifest__.py

```python
{
    'name': 'Cloud Service Config',
    'version': '18.0.1.0.0',
    'category': 'Technical',
    'summary': 'Centralized configuration for Azure Key Vault, Azure Service Bus, and Google Cloud Storage',
    'description': 'Provides a single settings screen and shared mixin for all third-party cloud service integrations.',
    'depends': ['base'],
    'data': [
        'security/ir.model.access.csv',
        'views/res_config_settings_views.xml',
    ],
    'external_dependencies': {
        'python': [
            'azure.identity',
            'azure.keyvault.secrets',
            'azure.servicebus',
            'google.cloud.storage',
            'google.oauth2',
        ],
    },
    'installable': True,
    'application': False,
    'license': 'LGPL-3',
}
```

> **Note on `external_dependencies`**: Listing them here means Odoo will refuse to install the module if the libraries are missing. If you want the module to install anyway and degrade gracefully, remove this block and rely on the try/except imports in the mixin.

## models/__init__.py

```python
from . import cloud_service_mixin
from . import res_config_settings
```

## models/cloud_service_mixin.py — The Shared Mixin

This is the core of the module. Every business model that needs cloud services inherits this mixin via `_inherit = ['cloud.service.mixin']`.

```python
import base64
import json
import logging
import os
from datetime import datetime

from odoo import models, api

_logger = logging.getLogger(__name__)

# --- Azure Key Vault ---
try:
    from azure.identity import DefaultAzureCredential
    from azure.keyvault.secrets import SecretClient
    KEYVAULT_AVAILABLE = True
except ImportError:
    _logger.warning("azure-keyvault-secrets not installed. Key Vault disabled.")
    KEYVAULT_AVAILABLE = False

# --- Azure Service Bus ---
try:
    from azure.servicebus import ServiceBusClient, ServiceBusMessage
    SERVICEBUS_AVAILABLE = True
except ImportError:
    _logger.warning("azure-servicebus not installed. Service Bus disabled.")
    SERVICEBUS_AVAILABLE = False

# --- Google Cloud Storage ---
try:
    from google.cloud import storage as gcs_storage
    from google.oauth2 import service_account as gcs_service_account
    GCS_AVAILABLE = True
except ImportError:
    _logger.warning("google-cloud-storage not installed. GCS disabled.")
    GCS_AVAILABLE = False
    gcs_storage = None
    gcs_service_account = None


class CloudServiceMixin(models.AbstractModel):
    _name = 'cloud.service.mixin'
    _description = 'Shared methods for Azure Key Vault, Service Bus, and GCS'

    # ──────────────────────────────────────────
    # Config helpers (read from ir.config_parameter)
    # ──────────────────────────────────────────

    def _get_cloud_param(self, key, default=None):
        """Read a cloud_services.* system parameter."""
        return self.env['ir.config_parameter'].sudo().get_param(
            f'cloud_services.{key}', default
        )

    # ──────────────────────────────────────────
    # Azure Key Vault
    # ──────────────────────────────────────────

    def _get_keyvault_secret(self, secret_name):
        """Retrieve a secret from Azure Key Vault.

        The vault URL is read from cloud_services.keyvault_url system parameter.
        Returns the secret value as string, or None on failure.
        """
        if not KEYVAULT_AVAILABLE:
            _logger.warning("Key Vault SDK not available")
            return None
        try:
            vault_url = self._get_cloud_param('keyvault_url')
            if not vault_url:
                _logger.error("Azure Key Vault URL not configured (Settings > Cloud Services)")
                return None

            credential = DefaultAzureCredential()
            client = SecretClient(vault_url=vault_url, credential=credential)
            return client.get_secret(secret_name).value

        except Exception as e:
            _logger.error(f"Key Vault error for '{secret_name}': {e}")
            return None

    # ──────────────────────────────────────────
    # Azure Service Bus
    # ──────────────────────────────────────────

    def _get_service_bus_client(self):
        """Create a ServiceBusClient using the configured connection string.

        Connection string source depends on config:
        - If keyvault_url is set AND sb_connection_secret is set: reads from Key Vault
        - Otherwise: reads sb_connection_string directly from system parameter
        """
        if not SERVICEBUS_AVAILABLE:
            _logger.warning("Service Bus SDK not available")
            return None
        try:
            # Try Key Vault first if configured
            secret_name = self._get_cloud_param('sb_connection_secret')
            if secret_name:
                connection_string = self._get_keyvault_secret(secret_name)
            else:
                connection_string = self._get_cloud_param('sb_connection_string')

            if not connection_string:
                _logger.error("Service Bus connection string not configured (Settings > Cloud Services)")
                return None

            return ServiceBusClient.from_connection_string(connection_string)

        except Exception as e:
            _logger.error(f"Service Bus client error: {e}")
            return None

    def _publish_event(self, event_type, data, queue_name=None, topic_name=None, session_id=None):
        """Publish a JSON event to Service Bus (queue or topic).

        Args:
            event_type: Event identifier (e.g. "order_created")
            data: Dict payload
            queue_name: Target queue (uses default from config if None)
            topic_name: Target topic (mutually exclusive with queue_name)
            session_id: For ordered delivery on session-enabled queues/topics
        """
        if not SERVICEBUS_AVAILABLE:
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
            message.application_properties = {
                "event_type": event_type,
                "source_system": "odoo"
            }

            if topic_name:
                with client.get_topic_sender(topic_name=topic_name) as sender:
                    sender.send_messages(message)
            else:
                target_queue = queue_name or self._get_cloud_param('sb_default_queue')
                if not target_queue:
                    _logger.error("No queue name provided and no default configured")
                    return False
                with client.get_queue_sender(queue_name=target_queue) as sender:
                    sender.send_messages(message)

            _logger.info(f"Published {event_type} to Service Bus")
            return True

        except Exception as e:
            _logger.error(f"Service Bus publish error: {e}", exc_info=True)
            return False

    # ──────────────────────────────────────────
    # Google Cloud Storage
    # ──────────────────────────────────────────

    def _get_gcs_credentials(self):
        """Load GCS service account credentials.

        Credentials path source depends on config:
        - If keyvault_url is set AND gcs_credentials_secret is set: reads path from Key Vault
        - Otherwise: reads gcs_credentials_path directly from system parameter
        """
        if not GCS_AVAILABLE:
            return None
        try:
            # Try Key Vault first if configured
            secret_name = self._get_cloud_param('gcs_credentials_secret')
            if secret_name:
                credentials_path = self._get_keyvault_secret(secret_name)
            else:
                credentials_path = self._get_cloud_param('gcs_credentials_path')

            if not credentials_path or not os.path.exists(credentials_path):
                _logger.warning(f"GCS credentials file not found: {credentials_path}")
                return None

            return gcs_service_account.Credentials.from_service_account_file(
                credentials_path,
                scopes=['https://www.googleapis.com/auth/cloud-platform']
            )
        except Exception as e:
            _logger.error(f"GCS credentials error: {e}", exc_info=True)
            return None

    def _get_gcs_bucket(self):
        """Return a GCS bucket object ready for upload/delete operations."""
        if not GCS_AVAILABLE:
            return None
        try:
            credentials = self._get_gcs_credentials()
            if credentials:
                client = gcs_storage.Client(credentials=credentials, project=credentials.project_id)
            else:
                client = gcs_storage.Client()  # ADC fallback

            bucket_name = self._get_cloud_param('gcs_bucket_name')
            if not bucket_name:
                _logger.error("GCS bucket name not configured (Settings > Cloud Services)")
                return None

            return client.bucket(bucket_name)

        except Exception as e:
            _logger.error(f"GCS bucket error: {e}", exc_info=True)
            return None

    def _upload_to_gcs(self, file_data_b64, folder, filename, content_type="image/png"):
        """Upload a base64-encoded file to GCS, return the public URL or False."""
        if not GCS_AVAILABLE or not file_data_b64:
            return False
        try:
            bucket = self._get_gcs_bucket()
            if not bucket:
                return False

            blob_path = f"{folder}/{filename}"
            blob = bucket.blob(blob_path)
            blob.upload_from_string(base64.b64decode(file_data_b64), content_type=content_type)

            public_url = f"https://storage.googleapis.com/{bucket.name}/{blob_path}"
            _logger.info(f"Uploaded to GCS: {public_url}")
            return public_url

        except Exception as e:
            _logger.error(f"GCS upload error: {e}", exc_info=True)
            return False

    def _delete_from_gcs(self, file_url):
        """Delete a blob from GCS by its public URL. Fails silently."""
        if not GCS_AVAILABLE or not file_url:
            return
        try:
            bucket = self._get_gcs_bucket()
            if not bucket or "storage.googleapis.com" not in file_url:
                return

            blob_path = file_url.split(f"{bucket.name}/", 1)[-1]
            blob = bucket.blob(blob_path)
            if blob.exists():
                blob.delete()
                _logger.info(f"Deleted from GCS: {blob_path}")

        except Exception as e:
            _logger.error(f"GCS delete error: {e}", exc_info=True)
```

## models/res_config_settings.py

```python
from odoo import models, fields


class ResConfigSettings(models.TransientModel):
    _inherit = 'res.config.settings'

    # ── Azure Key Vault ──
    cloud_keyvault_url = fields.Char(
        string="Key Vault URL",
        config_parameter='cloud_services.keyvault_url',
        help="e.g. https://my-vault.vault.azure.net/"
    )

    # ── Azure Service Bus ──
    cloud_sb_connection_secret = fields.Char(
        string="Connection String (Key Vault Secret Name)",
        config_parameter='cloud_services.sb_connection_secret',
        help="Name of the Key Vault secret containing the connection string. "
             "Leave empty to use the direct connection string below."
    )
    cloud_sb_connection_string = fields.Char(
        string="Connection String (Direct)",
        config_parameter='cloud_services.sb_connection_string',
        help="Only used if Key Vault secret name above is empty."
    )
    cloud_sb_default_queue = fields.Char(
        string="Default Queue Name",
        config_parameter='cloud_services.sb_default_queue',
    )
    cloud_sb_default_topic = fields.Char(
        string="Default Topic Name",
        config_parameter='cloud_services.sb_default_topic',
    )
    cloud_sb_max_retries = fields.Integer(
        string="Max Retries",
        config_parameter='cloud_services.sb_max_retries',
        default=3,
    )
    cloud_sb_batch_size = fields.Integer(
        string="Batch Size",
        config_parameter='cloud_services.sb_batch_size',
        default=50,
    )

    # ── Google Cloud Storage ──
    cloud_gcs_bucket_name = fields.Char(
        string="GCS Bucket Name",
        config_parameter='cloud_services.gcs_bucket_name',
    )
    cloud_gcs_credentials_secret = fields.Char(
        string="Credentials Path (Key Vault Secret Name)",
        config_parameter='cloud_services.gcs_credentials_secret',
        help="Name of the Key Vault secret containing the path to the GCS service account JSON. "
             "Leave empty to use the direct path below."
    )
    cloud_gcs_credentials_path = fields.Char(
        string="Credentials File Path (Direct)",
        config_parameter='cloud_services.gcs_credentials_path',
        help="Absolute path to the service account JSON on the server. "
             "Only used if Key Vault secret name above is empty."
    )
```

## views/res_config_settings_views.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="res_config_settings_view_form" model="ir.ui.view">
        <field name="name">res.config.settings.view.form.inherit.cloud_service_config</field>
        <field name="model">res.config.settings</field>
        <field name="inherit_id" ref="base.res_config_settings_view_form"/>
        <field name="arch" type="xml">
            <xpath expr="//div[hasclass('settings')]" position="inside">
                <div class="app_settings_block"
                     data-string="Cloud Services"
                     data-key="cloud_service_config"
                     groups="base.group_system">

                    <!-- Azure Key Vault -->
                    <h2>Azure Key Vault</h2>
                    <div class="row mt16 o_settings_container">
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_keyvault_url"/>
                                <div class="text-muted">
                                    Azure Key Vault URL for centralized secret management
                                </div>
                                <field name="cloud_keyvault_url"
                                       placeholder="https://my-vault.vault.azure.net/"/>
                            </div>
                        </div>
                    </div>

                    <!-- Azure Service Bus -->
                    <h2 class="mt32">Azure Service Bus</h2>
                    <div class="row mt16 o_settings_container">
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_sb_connection_secret"/>
                                <div class="text-muted">
                                    If using Key Vault, enter the secret name here
                                </div>
                                <field name="cloud_sb_connection_secret"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_sb_connection_string"/>
                                <div class="text-muted">
                                    Direct connection string (only if Key Vault secret is empty)
                                </div>
                                <field name="cloud_sb_connection_string" password="True"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_sb_default_queue"/>
                                <field name="cloud_sb_default_queue"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_sb_default_topic"/>
                                <field name="cloud_sb_default_topic"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_sb_max_retries"/>
                                <field name="cloud_sb_max_retries"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_sb_batch_size"/>
                                <field name="cloud_sb_batch_size"/>
                            </div>
                        </div>
                    </div>

                    <!-- Google Cloud Storage -->
                    <h2 class="mt32">Google Cloud Storage</h2>
                    <div class="row mt16 o_settings_container">
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_gcs_bucket_name"/>
                                <field name="cloud_gcs_bucket_name"
                                       placeholder="my-bucket-name"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_gcs_credentials_secret"/>
                                <div class="text-muted">
                                    If using Key Vault, enter the secret name for the credentials path
                                </div>
                                <field name="cloud_gcs_credentials_secret"/>
                            </div>
                        </div>
                        <div class="col-12 col-lg-6 o_setting_box">
                            <div class="o_setting_left_pane"/>
                            <div class="o_setting_right_pane">
                                <label for="cloud_gcs_credentials_path"/>
                                <div class="text-muted">
                                    Direct path to service account JSON (only if Key Vault secret is empty)
                                </div>
                                <field name="cloud_gcs_credentials_path"
                                       placeholder="/opt/credentials/gcs-service-account.json"/>
                            </div>
                        </div>
                    </div>

                </div>
            </xpath>
        </field>
    </record>
</odoo>
```

## How Business Modules Use It

Any business module that needs cloud services:

### 1. Depend on it in `__manifest__.py`

```python
'depends': ['cloud_service_config', 'sale', ...],
```

### 2. Inherit the mixin

```python
class ProductTemplate(models.Model):
    _inherit = ['product.template', 'cloud.service.mixin']
    # The mixin methods are now available: _publish_event(), _upload_to_gcs(), etc.
```

For models that already inherit another model (most Odoo models), just add the mixin to the list:

```python
class SaleOrder(models.Model):
    _inherit = ['sale.order', 'cloud.service.mixin']
```

### 3. Call the methods

```python
# Upload an image to GCS
url = self._upload_to_gcs(self.image_1920, "products", f"{self.id}.png")

# Publish an event to Service Bus
self._publish_event("product_updated", {"id": self.id, "name": self.name})

# Get a secret from Key Vault
api_key = self._get_keyvault_secret("external-api-key")
```

No imports, no connection logic, no config lookups — the mixin handles everything.

## Config Parameter Reference

All parameters use the `cloud_services.` prefix:

| Parameter | Purpose |
|-----------|---------|
| `cloud_services.keyvault_url` | Azure Key Vault URL |
| `cloud_services.sb_connection_secret` | Key Vault secret name for Service Bus connection string |
| `cloud_services.sb_connection_string` | Direct Service Bus connection string (fallback) |
| `cloud_services.sb_default_queue` | Default queue name |
| `cloud_services.sb_default_topic` | Default topic name |
| `cloud_services.sb_max_retries` | Max retries for failed messages |
| `cloud_services.sb_batch_size` | Batch size for cron processing |
| `cloud_services.gcs_bucket_name` | GCS bucket name |
| `cloud_services.gcs_credentials_secret` | Key Vault secret name for GCS credentials path |
| `cloud_services.gcs_credentials_path` | Direct path to GCS service account JSON (fallback) |

## Design Decisions

**Why one module, not three?**
Admins configure once, in one place. Business developers add one dependency. Connection logic (especially the Key Vault → Service Bus chain) lives in one file.

**Why a mixin, not a regular model?**
`AbstractModel` mixins add methods to existing models without creating a database table. Business models inherit the mixin and get all methods — no delegation, no extra records.

**Why both Key Vault secret name AND direct value?**
Not every project uses Key Vault. The "direct" fields let teams start fast without Azure infrastructure, then switch to Key Vault later by just filling in the secret name.

**Why `ir.config_parameter`?**
It's Odoo's built-in key-value store. Values persist across module updates, are accessible from any model via `self.env['ir.config_parameter'].sudo().get_param()`, and the `config_parameter` attribute on settings fields handles read/write automatically.
