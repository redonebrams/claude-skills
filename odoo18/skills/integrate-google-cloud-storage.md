---
name: integrate-google-cloud-storage
description: >
  Guide for integrating Google Cloud Storage (GCS) into any Odoo 18 module — uploading, downloading, deleting files
  from a GCS bucket. Use this skill whenever the user wants to: add image/file upload to GCS in an Odoo model,
  set up GCS credentials in Odoo, generate public CDN URLs, clean up blobs on record deletion, or build any
  feature that stores files in Google Cloud Storage. Also trigger when the user mentions "CDN", "image upload",
  "cloud storage", "bucket", "blob", "GCS", or "Google Storage" in the context of Odoo development.
---

# Google Cloud Storage Integration in Odoo 18

## First-Time Setup: The `cloud_service_config` Module

**Before using GCS in any business module**, you need the **`cloud_service_config`** module installed and configured. This is a single shared module that centralizes the configuration for **all** third-party cloud services (Azure Key Vault, Azure Service Bus, Google Cloud Storage) in one Odoo settings screen.

If the module doesn't exist yet in the project, **create it first** — see the `integrate-cloud-services-setup` skill for the full module structure. The GCS-related parts it provides:
- Settings fields for: GCS bucket name, credentials file path (or Key Vault secret name for the path), default upload folder prefixes
- A shared mixin (`cloud.service.mixin`) with `_get_gcs_credentials()` and `_get_gcs_bucket()` methods
- System parameters: `cloud_services.gcs_bucket_name`, `cloud_services.gcs_credentials_path`

All business modules that upload to GCS should **depend on `cloud_service_config`** and inherit its mixin — never hardcode bucket names or duplicate credential logic.

## When to Use

Use GCS when your Odoo module needs to store files (images, documents, exports) on a cloud CDN instead of Odoo's database-backed `ir.attachment`. Typical reasons: serving images to a mobile app, reducing database size, public URL access without Odoo auth.

## Dependencies

Add to your module's external dependencies or `requirements.txt`:

```
google-cloud-storage
google-auth
```

## Step 1: Imports with Graceful Fallback

Odoo modules must not crash on startup if an optional library is missing. Wrap GCS imports in a try/except so the module installs cleanly and degrades gracefully:

```python
import base64
import os
import logging
from datetime import datetime

_logger = logging.getLogger(__name__)

try:
    from google.cloud import storage
    from google.oauth2 import service_account
    GCS_AVAILABLE = True
except ImportError:
    _logger.warning("google-cloud-storage not installed. GCS features disabled.")
    GCS_AVAILABLE = False
    storage = None
    service_account = None
```

Check `GCS_AVAILABLE` before any GCS operation.

## Step 2: Credentials

GCS supports multiple credential strategies. Pick the one that fits your infrastructure:

### Option A: Service Account JSON file (most common in on-premise Odoo)

The JSON file lives on the server. Store the **path** in Odoo system parameters, Azure Key Vault, or an env variable — never hardcode it.

```python
def _get_gcs_credentials(self):
    """Load GCS service account credentials."""
    if not GCS_AVAILABLE:
        return None
    try:
        # Get the path from wherever your project stores config
        # Examples:
        #   self.env['ir.config_parameter'].sudo().get_param('gcs.credentials_path')
        #   self._get_keyvault_secret("my-gcs-creds-path")
        #   os.environ.get('GCS_CREDENTIALS_PATH')
        credentials_path = self._get_credentials_path()

        if not credentials_path or not os.path.exists(credentials_path):
            _logger.error(f"GCS credentials file not found: {credentials_path}")
            return None

        return service_account.Credentials.from_service_account_file(
            credentials_path,
            scopes=['https://www.googleapis.com/auth/cloud-platform']
        )
    except Exception as e:
        _logger.error(f"Error loading GCS credentials: {e}", exc_info=True)
        return None
```

### Option B: Application Default Credentials (ADC)

If Odoo runs on GCP (Compute Engine, Cloud Run) or you've run `gcloud auth application-default login` locally, just use:

```python
client = storage.Client()  # auto-discovers credentials
```

### Option C: JSON content from a secret manager

If your secret manager stores the JSON content (not a file path), parse it directly:

```python
import json
from google.oauth2 import service_account

json_content = self._get_secret("gcs-service-account-json")
info = json.loads(json_content)
credentials = service_account.Credentials.from_service_account_info(
    info, scopes=['https://www.googleapis.com/auth/cloud-platform']
)
```

## Step 3: Client and Bucket Initialization

```python
def _get_gcs_bucket(self):
    """Return a GCS bucket object ready for operations."""
    credentials = self._get_gcs_credentials()
    if credentials:
        client = storage.Client(credentials=credentials, project=credentials.project_id)
    else:
        client = storage.Client()  # ADC fallback

    bucket_name = self._get_bucket_name()  # from config, not hardcoded
    return client.bucket(bucket_name)
```

Keep the bucket name configurable — store it in `ir.config_parameter` or a settings model so it can differ between environments (dev/staging/prod).

## Step 4: Upload

Standard pattern for uploading a base64-encoded image (Odoo stores binary fields as base64):

```python
def _upload_to_gcs(self, image_data_b64, folder, filename):
    """Upload base64 image to GCS, return public URL or False."""
    if not GCS_AVAILABLE or not image_data_b64:
        return False
    try:
        bucket = self._get_gcs_bucket()
        blob_path = f"{folder}/{filename}"
        blob = bucket.blob(blob_path)

        image_bytes = base64.b64decode(image_data_b64)
        blob.upload_from_string(image_bytes, content_type="image/png")

        # Public URL (requires bucket to have public access or uniform ACL)
        public_url = f"https://storage.googleapis.com/{bucket.name}/{blob_path}"
        _logger.info(f"Uploaded to GCS: {public_url}")
        return public_url

    except Exception as e:
        _logger.error(f"GCS upload failed: {e}", exc_info=True)
        return False
```

### Filename best practices

Build unique, safe filenames to avoid collisions and URL issues:

```python
now_str = datetime.now().strftime("%Y%m%d%H%M%S")
safe_name = record.name.replace(' ', '_').replace('/', '_')
filename = f"{safe_name}_{now_str}.png"
```

### Uploading from a file on disk

Use `upload_from_filename` when you have a temp file (useful for PDF exports, generated reports):

```python
import tempfile

with tempfile.NamedTemporaryFile(suffix='.pdf', delete=True) as tmp:
    tmp.write(file_bytes)
    tmp.flush()
    blob = bucket.blob(f"reports/{filename}")
    blob.upload_from_filename(tmp.name, content_type="application/pdf")
```

## Step 5: Delete

Clean up GCS when a record is deleted from Odoo:

```python
def unlink(self):
    """Remove GCS files before deleting Odoo records."""
    if GCS_AVAILABLE:
        try:
            bucket = self._get_gcs_bucket()
            for record in self:
                if record.file_url and "storage.googleapis.com" in record.file_url:
                    blob_path = record.file_url.split(f"{bucket.name}/", 1)[-1]
                    blob = bucket.blob(blob_path)
                    if blob.exists():
                        blob.delete()
        except Exception as e:
            _logger.error(f"GCS cleanup failed: {e}", exc_info=True)
    return super().unlink()
```

## Step 6: Hook into Odoo Model Lifecycle

The typical integration point — auto-upload when an image field changes:

```python
from odoo import models, fields, api

class MyModel(models.Model):
    _name = 'my.model'

    image = fields.Binary(string="Image")
    image_url = fields.Char(string="CDN URL", readonly=True)

    @api.model_create_multi
    def create(self, vals_list):
        records = super().create(vals_list)
        for record in records:
            if record.image:
                url = record._upload_to_gcs(record.image, "my_model", f"{record.id}.png")
                if url:
                    record.sudo().write({'image_url': url})
        return records

    def write(self, vals):
        res = super().write(vals)
        if 'image' in vals:
            for record in self:
                if record.image:
                    url = record._upload_to_gcs(record.image, "my_model", f"{record.id}.png")
                    if url:
                        # Use context flag to prevent recursion
                        record.with_context(_skip_gcs=True).sudo().write({'image_url': url})
        return res
```

## Step 7: Add to Views

Show the CDN URL in the form view:

```xml
<field name="image_url" widget="url" readonly="1"/>
<!-- Or display the image directly from URL -->
<img t-att-src="record.image_url.value" style="max-height:200px;"/>
```

## Organizing Your Bucket

Use path prefixes to separate entity types. This keeps things clean and makes IAM rules possible per-prefix:

```
your-bucket/
├── products/          # Product images
├── categories/        # Category icons
├── brands/            # Brand logos
├── banners/           # Marketing banners
├── reports/           # Generated PDFs
└── exports/           # Data exports
```

## Configuration Checklist for a New Module

1. **Dependencies**: Add `google-cloud-storage` and `google-auth` to requirements
2. **Imports**: Use try/except with `GCS_AVAILABLE` flag
3. **Credentials**: Pick a strategy (service account file, ADC, or JSON from secret manager) — keep the source configurable
4. **Bucket name**: Store in `ir.config_parameter` or a settings model, never hardcode
5. **Model fields**: Add a `Char` field for the CDN URL (readonly)
6. **Upload method**: Implement `_upload_to_gcs()` with folder prefix and safe filenames
7. **Lifecycle hooks**: Override `create()`/`write()` for auto-upload, `unlink()` for cleanup
8. **Views**: Add the URL field to form/list views
9. **Bucket permissions**: Ensure the bucket allows public read if you need public URLs, or use signed URLs for private access

## Signed URLs (Private Buckets)

If your bucket isn't public, generate time-limited signed URLs instead:

```python
from datetime import timedelta

blob = bucket.blob(blob_path)
signed_url = blob.generate_signed_url(
    version="v4",
    expiration=timedelta(hours=1),
    method="GET"
)
```

## Common Pitfalls

- **Recursion in `write()`**: Uploading in `write()` then calling `write()` to save the URL can loop. Use a context flag (`_skip_gcs`) to break it.
- **Large files**: `upload_from_string` loads everything in memory. For files >50MB, use `upload_from_filename` with a temp file.
- **Missing credentials at startup**: The try/except import pattern prevents crashes, but make sure to check `GCS_AVAILABLE` before every operation.
- **Bucket doesn't exist**: `client.bucket(name)` doesn't verify the bucket exists. Call `bucket.exists()` during setup if you want to validate early.
