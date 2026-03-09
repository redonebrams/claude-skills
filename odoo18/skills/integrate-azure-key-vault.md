---
name: integrate-azure-key-vault
description: >
  Guide for integrating Azure Key Vault into any Odoo 18 module for secure secrets management.
  Use this skill whenever the user wants to: store or retrieve secrets securely, manage connection strings,
  API keys, or credentials outside of Odoo system parameters, set up Azure Key Vault in an Odoo module,
  or replace hardcoded credentials with vault-backed secrets. Also trigger when the user mentions
  "Key Vault", "secrets management", "DefaultAzureCredential", "vault", "secure credentials",
  or "azure secrets" in the context of Odoo development.
---

# Azure Key Vault Integration in Odoo 18

## First-Time Setup: The `cloud_service_config` Module

**Before using Key Vault in any business module**, you need the **`cloud_service_config`** module installed and configured. This is a single shared module that centralizes the configuration for **all** third-party cloud services (Azure Key Vault, Azure Service Bus, Google Cloud Storage) in one Odoo settings screen.

If the module doesn't exist yet in the project, **create it first** — see the `integrate-cloud-services-setup` skill for the full module structure, or build it with these essentials:
- A `res.config.settings` form with a "Cloud Services" section containing fields for: Key Vault URL, Service Bus connection string (or its Key Vault secret name), GCS bucket name, GCS credentials path, queue/topic names, retry limits
- A shared mixin model (`cloud.service.mixin`) with reusable methods: `_get_keyvault_secret()`, `_get_service_bus_client()`, `_get_gcs_bucket()` — so business modules just inherit the mixin and call the methods
- System parameters (`ir.config_parameter`) to persist all values

All business modules that need cloud services should **depend on `cloud_service_config`** and inherit its mixin — never duplicate connection logic or hardcode config values.

## When to Use

Use Azure Key Vault when your Odoo module needs secrets (API keys, connection strings, credentials) that shouldn't live in `ir.config_parameter`, `.env` files, or source code. Key Vault gives you centralized, audited, rotatable secret storage with fine-grained access control.

Typical secrets: database connection strings, external API keys, service account credentials, OAuth client secrets, encryption keys.

## Dependencies

Add to your module's external dependencies or `requirements.txt`:

```
azure-identity>=1.12.0
azure-keyvault-secrets
```

## Step 1: Imports with Graceful Fallback

```python
import logging

_logger = logging.getLogger(__name__)

try:
    from azure.identity import DefaultAzureCredential
    from azure.keyvault.secrets import SecretClient
    AZURE_KEYVAULT_AVAILABLE = True
except ImportError:
    _logger.warning("azure-keyvault-secrets not installed. Key Vault features disabled.")
    AZURE_KEYVAULT_AVAILABLE = False
```

## Step 2: The Core Secret Retrieval Method

Add this method to any model that needs secrets. The vault URL should come from configuration, not be hardcoded:

```python
def _get_keyvault_secret(self, secret_name):
    """Retrieve a single secret from Azure Key Vault.

    Returns the secret value as a string, or None on failure.
    """
    if not AZURE_KEYVAULT_AVAILABLE:
        _logger.warning("Key Vault SDK not available, cannot retrieve secret")
        return None
    try:
        # Get vault URL from Odoo config — adapt to your config strategy
        vault_url = self._get_vault_url()

        credential = DefaultAzureCredential()
        client = SecretClient(vault_url=vault_url, credential=credential)
        return client.get_secret(secret_name).value

    except Exception as e:
        _logger.error(f"Error retrieving secret '{secret_name}': {e}")
        return None
```

### Where to store the vault URL

The vault URL (`https://your-vault.vault.azure.net/`) needs to be configurable per environment. Options:

```python
# Option A: Odoo system parameter (simplest)
def _get_vault_url(self):
    return self.env['ir.config_parameter'].sudo().get_param('azure.keyvault.url')

# Option B: Environment variable (good for containers)
def _get_vault_url(self):
    return os.environ.get('AZURE_KEYVAULT_URL')

# Option C: res.config.settings field (admin UI)
def _get_vault_url(self):
    return self.env['ir.config_parameter'].sudo().get_param('my_module.keyvault_url')
```

## Step 3: Authentication with DefaultAzureCredential

`DefaultAzureCredential` tries multiple authentication methods automatically, in order:

1. **Environment variables** — `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`
2. **Managed Identity** — when running on Azure (VM, App Service, AKS)
3. **Azure CLI** — uses your `az login` session (local development)
4. **VS Code / Visual Studio** — cached IDE credentials

For **production on a Linux server** (typical Odoo deployment), use either:
- Environment variables (set in systemd service file or `/etc/odoo.conf` environment)
- Managed Identity (if the server is an Azure VM)

For **local development**: just run `az login` once, and it works.

## Step 4: Common Usage Patterns

### Pattern A: Retrieve a connection string

```python
def _get_service_bus_client(self):
    connection_string = self._get_keyvault_secret("my-servicebus-connection")
    if not connection_string:
        return None
    return ServiceBusClient.from_connection_string(connection_string)
```

### Pattern B: Retrieve a file path to credentials

Some services (like GCS) use a JSON credentials file. Store the **file path** in Key Vault:

```python
def _get_gcs_credentials(self):
    credentials_path = self._get_keyvault_secret("gcs-credentials-path")
    if not credentials_path or not os.path.exists(credentials_path):
        return None
    return service_account.Credentials.from_service_account_file(credentials_path)
```

### Pattern C: Validate an API token

Protect your Odoo REST endpoints by comparing incoming tokens against a Key Vault secret:

```python
def _validate_api_token(self, request):
    auth_header = request.httprequest.headers.get('Authorization', '')
    if not auth_header.startswith('Bearer '):
        return False

    token = auth_header[7:]
    expected = self._get_keyvault_secret("my-api-key")
    if not expected:
        _logger.error("Could not retrieve API key from Key Vault")
        return None  # distinguish config error from auth failure

    return token == expected
```

## Step 5: Optional — Caching

Each call to `_get_keyvault_secret()` makes a network round-trip to Azure. For secrets that don't change often (connection strings, API keys), add a simple TTL cache:

```python
import time

_secret_cache = {}
_CACHE_TTL = 300  # 5 minutes

def _get_keyvault_secret(self, secret_name):
    if not AZURE_KEYVAULT_AVAILABLE:
        return None

    # Check cache first
    now = time.time()
    cached = _secret_cache.get(secret_name)
    if cached and (now - cached[1]) < _CACHE_TTL:
        return cached[0]

    try:
        vault_url = self._get_vault_url()
        credential = DefaultAzureCredential()
        client = SecretClient(vault_url=vault_url, credential=credential)
        value = client.get_secret(secret_name).value

        if value:
            _secret_cache[secret_name] = (value, now)
        return value

    except Exception as e:
        _logger.error(f"Error retrieving secret '{secret_name}': {e}")
        return None
```

## Step 6: Optional — Database Gating

If your Odoo instance serves multiple databases (staging + dev on same server), restrict Key Vault access to specific databases to prevent cross-environment leaks:

```python
def _get_keyvault_secret(self, secret_name):
    allowed_db = self.env['ir.config_parameter'].sudo().get_param('azure.keyvault.allowed_db')
    if allowed_db and self.env.cr.dbname != allowed_db:
        _logger.info(f"Skipping Key Vault - database {self.env.cr.dbname} not allowed")
        return None
    # ... rest of the method
```

## Step 7: Admin Configuration UI (Optional)

Let admins configure the vault URL from Odoo Settings:

```python
# models/res_config_settings.py
from odoo import models, fields

class ResConfigSettings(models.TransientModel):
    _inherit = 'res.config.settings'

    keyvault_url = fields.Char(
        string="Azure Key Vault URL",
        config_parameter='my_module.keyvault_url',
        help="e.g. https://my-vault.vault.azure.net/"
    )
```

```xml
<!-- views/res_config_settings_views.xml -->
<record id="res_config_settings_view_form_inherit" model="ir.ui.view">
    <field name="name">res.config.settings.view.form.inherit.my_module</field>
    <field name="model">res.config.settings</field>
    <field name="inherit_id" ref="base.res_config_settings_view_form"/>
    <field name="arch" type="xml">
        <xpath expr="//div[hasclass('settings')]" position="inside">
            <div class="app_settings_block" data-string="My Module" data-key="my_module">
                <h2>Azure Key Vault</h2>
                <div class="row mt16 o_settings_container">
                    <div class="col-12 col-lg-6 o_setting_box">
                        <div class="o_setting_left_pane"/>
                        <div class="o_setting_right_pane">
                            <label for="keyvault_url"/>
                            <field name="keyvault_url"/>
                        </div>
                    </div>
                </div>
            </div>
        </xpath>
    </field>
</record>
```

## Configuration Checklist for a New Module

1. **Dependencies**: Add `azure-identity` and `azure-keyvault-secrets` to requirements
2. **Imports**: Use try/except with `AZURE_KEYVAULT_AVAILABLE` flag
3. **Vault URL**: Store in `ir.config_parameter` or env variable — never hardcode
4. **`_get_keyvault_secret()` method**: Add to your model
5. **Authentication**: Ensure the server has valid Azure credentials (env vars, managed identity, or `az login`)
6. **Error handling**: Always handle `None` return — the method fails silently by design
7. **Caching** (optional): Add TTL cache for frequently-accessed secrets
8. **Database gating** (optional): Restrict to specific databases if multi-db

## Adding a New Secret

1. Create the secret in Azure Portal or CLI: `az keyvault secret set --vault-name my-vault --name my-secret --value "the-value"`
2. Reference by name: `self._get_keyvault_secret("my-secret")`
3. Document the secret name in your module's README

## Common Pitfalls

- **No credential on server**: `DefaultAzureCredential` will throw if no auth method works. Set up env vars in your systemd service file for production.
- **Vault firewall**: If Key Vault has network restrictions, ensure the Odoo server's IP is whitelisted.
- **Secret not found**: `get_secret()` throws `ResourceNotFoundError`. The try/except catches it, but check your secret names for typos.
- **Performance**: Without caching, each secret retrieval is a network call. For cron jobs running every few minutes, this adds up.
