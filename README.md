# Adv.Attachments for YouTrack

Unified attachment manager for YouTrack Server — native YouTrack attachments and S3 files in one widget.

Designed for YouTrack Server.

## Features

- **Native YouTrack attachments** — browse, preview and manage native issue attachments directly in the widget, no download required.
- **S3‑compatible storage** — Amazon S3, MinIO, and any S3-compatible service for external file storage.
- **Unified file view** — both native and S3 files displayed together with consistent UI.
- **Built-in file preview** for images, PDF, Office (XLSX, XLS, XLSB, DOCX, ODT), JSON, Markdown, ZIP and text files:
  - XLSX/XLS/XLSB: up to 1000 rows × 50 columns with multi-sheet tabs and truncation warnings
  - DOCX/ODT: 30-second rendering timeout with size warnings for large files (>5MB)
  - JSON: syntax-highlighted with auto-formatting
  - Markdown: rendered with proper formatting
  - ZIP: browse archive contents without downloading
  - Text/code files: canvas thumbnail previews in grid view (code, configs, logs, CSV, XML, etc.)
  - Selectable preview engine (built‑in or native browser) and open mode (tab or popup)
  - Unified header with file metadata (size, date) auto-updated from response headers
- **Progressive loading** — non-blocking UI shell, inline loaders, parallel file list fetch.
- **Grid/List views** with automatic switching, lazy-loaded image thumbnails, and responsive design.
- **AWS Signature V4** presigned URLs with native runtime-compatible cryptographic implementation.
- **MinIO optimized** with proper path‑style addressing and query parameter handling.
- **Clock skew auto-detection** and compensation for presigned URL signature issues.
- **Admin healthcheck** with real presigned operations (list/put/get/delete) for configuration validation.
- **Safe admin API**: credentials never returned; only configuration flags are exposed.
- **Modular backend** architecture with separated concerns (crypto, handlers, utilities).
- **Optimized thumbnail caching** — thumbnails fetched once and stay in browser cache; expired presigned URLs retried on demand. Settings and presign requests are cached and deduplicated to minimize backend traffic.
- **Sensitive file lock** — lock native attachments to restrict access to the issue reporter, a configurable user field (e.g. Assignee), and selected groups. Configurable per project, disabled by default. Locked files display a lock overlay in grid view and a lock badge in list view. Real-time polling detects field changes (e.g. Assignee) and automatically updates file visibility without page reload.

## Goal & Explanation

Provide a unified attachment management experience for YouTrack: view and manage native attachments alongside external S3 files in a single widget. Upload large files to S3, browse and preview inline without downloading. The app uses the YouTrack REST API for native attachments and the S3 endpoint you configure in Admin Settings; no other external services are required.

### For System Administrators

- **Simplified Backups** — Files stored externally in your S3 bucket, separate from YouTrack database. Back up file attachments independently using your S3 provider's backup tools.
- **Configurable File Size Limits** — Set maximum upload size up to 2 GB via settings (default 50 MB), or set to `0` for unlimited. Upload large files directly to S3 without impacting YouTrack performance.
- **Reduced Database Size** — Keep your YouTrack database lean and fast by offloading file storage to dedicated S3-compatible infrastructure.
- **Scalable Storage** — Use existing S3/MinIO infrastructure with unlimited scaling capacity. Pay only for actual storage used.

## Architecture

The app consists of three integrated components:

- **File Browser Widget** — Displays in issue footer; unified view of native and S3 files with thumbnails, upload/download/delete operations
- **Admin Configuration Panel** — Global settings page for S3 credentials, TTL, preview engine, and system diagnostics
- **File Previewer** — Standalone preview window for Office documents (XLSX, XLS, XLSB, DOCX, ODT), images, PDFs, JSON, Markdown, ZIP and text/code files with built-in or native browser rendering

## Security

- **Obfuscated credentials at rest** — S3 connection string (endpoint, keys, bucket) is obfuscated before storage in YouTrack settings. Each installation derives a unique key, so obfuscated values are not portable between instances. **Note:** this is obfuscation, not encryption — any YouTrack instance admin can read the value via the REST API (YouTrack does not yet support `secret`-format settings in custom apps, see JT-88186). Scope your AWS IAM policy to the specific bucket and prefix used by the app.
- **No plaintext secrets in storage** — raw credentials are never persisted; only the obfuscated blob is stored.
- **Credentials never exposed via API** — admin endpoints return configuration flags only; secret values are redacted from all responses and logs.
- **Presigned URLs** — all S3 operations use short-lived presigned URLs (AWS Signature V4). No long-lived tokens are sent to the browser.
- **Least-privilege permissions** — the widget requires only `READ_ISSUE`, `UPDATE_ISSUE` (for file operations) and `ADMIN` (for configuration). No broader access is needed.
- **No external network requests** — the app contacts only your configured S3 endpoint. Preview libraries are bundled locally; no CDN or analytics services are used.
- **Per-issue file isolation** — files are scoped per issue. Each issue has its own isolated storage folder. If isolation cannot be established, the operation is blocked entirely — no fallback to shared access.
- **Privacy** — no telemetry, analytics, or personal data is collected or transmitted. All processing occurs within your YouTrack instance and your S3 storage.

## Getting Started

### 1. Install the app

Install Adv.Attachments from YouTrack Marketplace or upload manually to your YouTrack Server instance.

### 2. Obtain S3 credentials

Get Access Key ID and Secret Access Key from your S3 provider:

- **Amazon S3**: [AWS IAM Access Keys Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- **MinIO**: [MinIO User Management](https://min.io/docs/minio/linux/administration/identity-access-management/minio-user-management.html)
- **Other S3-compatible**: Check your provider's documentation for API credentials (Wasabi, DigitalOcean Spaces, Backblaze B2, etc.)

### 3. Configure the app

Go to YouTrack Admin → Applications → Adv.Attachments and configure:

- **S3 endpoint URL**: e.g., `https://s3.amazonaws.com` or `http://localhost:9000`
- **Region**: e.g., `us-east-1` (required even for MinIO)
- **Bucket name**: Your S3 bucket name
- **Base directory**: Optional prefix for all files (e.g., `youtrack-attachments/`)
- **Access Key ID and Secret Access Key**: From step 2
- **Path-style addressing**: Enable for MinIO and some S3-compatible services
- **Preview engine**: Choose built-in (bundled renderers) or native browser
- **TTL settings**: Configure presigned URL expiration times

### 4. Test configuration

Click **"Run S3 Healthcheck"** button to verify all operations (list, upload, download, delete) work correctly.

### 5. Start using

Navigate to any issue and use the Adv.Attachments widget in the issue footer to upload and manage files.

## Settings

### S3 Connection

- **S3 Connection String** — Obfuscated S3 connection string (endpoint, region, bucket, keys, options). Generated via the admin panel "Generate Connection String" button. The value is obfuscated, not encrypted — any YouTrack instance admin can read it. See [Security](#security).

### General

- **Clock Skew Adjustment (sec)** — Offset (sec) to compensate clock skew; can be negative. `0` = auto-detect. Default: `0`.
- **Disable caching for presign requests** — Add cache‑busting to presign requests, disable intermediate caching. Default: `true`.
- **Operating mode** — Logging mode: `no_log` | `minimal` | `debug`. Default: `no_log`.
- **Metadata storage** — Where to store per-issue file metadata: `youtrack` (extension properties, recommended) or `s3` (legacy). Default: `youtrack`.

### TTL

- **List URL TTL (sec)** — TTL for presigned List URLs. Default: `120`.
- **Upload URL TTL (sec)** — TTL for presigned Upload (PUT) URLs. Default: `900`.
- **Download URL TTL (sec)** — TTL for presigned Download (GET) URLs. Default: `600`.
- **Delete URL TTL (sec)** — TTL for presigned Delete URLs. Default: `300`.

### Preview

- **Preview engine** — File preview engine: `built_in` (bundled renderers) | `native` (browser). Default: `built_in`.
- **Preview open mode** — Where to open preview: `tab` | `popup`. Default: `popup`.
- **Show Thumbnails** — Thumbnail loading mode: `all` (S3 + native) | `native_only` (skip S3 presign traffic) | `none` (no thumbnails). Default: `all`.

### Native Attachments

- **Native Attachments Refresh Interval (sec)** — Auto-refresh interval for native attachment list polling. Lower values detect changes faster but increase server load. `0` = disable auto-refresh. Default: `60`.

### Upload & Files

- **Minimum File Size (Mb)** — Upload routing threshold (MB). Files below this go to native YouTrack; at or above go to S3. Set to `0` for everything to S3. Default: `10`.
- **Maximum File Size (Mb)** — Maximum file size (MB) for S3 uploads. Set to `0` for unlimited. Configurable up to 2048 MB. Default: `50`.
- **Default List View** — Default list view: `grid` | `list`. Default: `grid`.
- **Files List View Switch Threshold** — Auto‑switch to list view threshold. Default: `20`.

### Sensitive File Lock (per-project)

- **Sensitive Files: Mode** — Controls the locking feature. `enabled` — locking is active. `disabled_clear` *(default)* — feature disabled, all existing locks are removed automatically on next sync. `disabled_leave` — feature disabled, but existing locked files remain locked and can still be unlocked manually by permitted users.
- **Sensitive Files: Access Groups** — User groups that always have access to locked files and can lock/unlock them (e.g. Project Admins, Team Leads). Supports multiple groups.
- **Sensitive Files: Sync Poll Interval (sec)** — How often the widget checks for permission changes on locked files (e.g. after Assignee change). Only active when the feature is enabled. Default: `5`.
- **Sensitive Files: Access Field** — Issue field name(s) whose value (a user) gets read access to locked files. Supports multiple fields separated by comma. Examples: `Assignee` or `Assignee, QA Engineer`. Leave empty if only groups should have access.

**Who can lock/unlock:** Issue reporter, members of Access Groups, and system admins.
**Who can see a locked file:** Issue reporter, the user from Access Field, and members of Access Groups.
**Auto-sync:** The widget polls for permission changes every few seconds (configurable). When a field change is detected (e.g. Assignee updated), file visibility and lock button state update automatically without page reload.

### Comments (per-project)

- **Comment File Operations** — Auto-add comments on upload/delete with embedded preview cards. Default: `true`.
- **Comment Visibility Group** — Restrict file operation comments to a specific user group. Leave empty for public.

### Upload routing

The widget automatically routes uploads based on file size and the `minFileSizeMB` setting:

- **Below threshold** → native YouTrack attachments (via REST API with `base64Content`)
- **At or above threshold** → S3 presigned PUT
- **S3 not configured** → all uploads go to native YouTrack

Native upload uses the YouTrack `base64Content` JSON attribute, which adds ~33% encoding overhead. This works reliably for files up to ~10 MB. For larger files, S3 storage is recommended.

Recommended configurations:

- **Optimal (recommended)** — set `minFileSizeMB` to `1`–`5`. S3 object storage is optimized for larger objects; many tiny files (< 1 MB) increase per-request overhead and listing latency. Routing small files to native YouTrack keeps S3 efficient and reduces API calls.
- **Default** — `minFileSizeMB` at `10`. Conservative threshold; works out of the box.
- **S3 not configured** — all files are uploaded to native YouTrack regardless of this setting.
- **Everything to S3** — set `minFileSizeMB` to `0`. No files go to native storage.

## Usage

- Open an issue to use the attachments widget.
- **Attach files**: Drag & drop or click "Attach" button.
- **Browse files**: Switch between Grid and List views; sort by date, type, or name.
- **Preview files**: Double-click to open preview (supports images, PDF, text, XLSX, XLS, XLSB, DOCX, ODT, JSON, Markdown, ZIP, and code files).
  - Native attachments preview directly without downloading.
  - Built-in preview uses bundled viewer libraries and renders in a popup/tab window.
  - Preview limits: XLSX/XLS/XLSB (1000×50), DOCX/ODT (30s timeout, 5MB warning).
  - Use "Download" button for full file access if preview is truncated or slow.
- **Download/Delete**: Hover over file for action buttons.

---

### Notes

- Settings are managed via the YouTrack App Settings and stored in YouTrack storage in production.
- For MinIO you typically need `s3ForcePathStyle=true` and a valid `region` (commonly `us-east-1`).
- Ensure required permissions are configured on your S3 provider side according to your organization’s policy. Configure only the settings visible in the app Admin page.
- S3 Healthcheck is triggered from the Admin page. It performs real presigned operations server‑side; browser CORS does not apply.
- **Moving issues between projects**: when `metaStorage` is set to `youtrack` (default), file metadata is stored in YouTrack extension properties on the issue. If an issue is moved to a project where Adv.Attachments is **not installed**, the metadata is preserved but inaccessible — the widget won't appear and no data is lost. Once the app is added to that project, the widget picks up existing metadata automatically. S3 files themselves are unaffected by project moves since they are stored by a stable `storageId` independent of the project key.
- **Privacy**: This app does not collect, transmit, or store any telemetry, analytics, or personal data. All processing occurs locally within your YouTrack instance and your configured S3 storage. The only external service contacted is your configured S3 endpoint. No CDN or third-party network requests are made.

#### Bundled preview libraries

- Built‑in preview uses locally bundled viewer libraries (no external CDN requests):
  - `xlsx` (Apache‑2.0) — spreadsheet rendering
  - `jszip` (MIT) — ZIP decompression for DOCX
  - `docx-preview` (Apache‑2.0) — Word document rendering
  - `odf-kit` (Apache‑2.0) — ODT document rendering
  See `THIRD_PARTY_NOTICES.md` → Bundled Preview Libraries.

### Troubleshooting

- **"S3 configuration is incomplete"** — fill all required fields in Admin Settings and click "Save", then run the Healthcheck.
- **403 errors**: verify bucket policy and credentials, check region/endpoint, and ensure MinIO path‑style is enabled when required.
  - The app includes automatic clock skew detection and retry logic for expired signatures.
- **Time skew**: if you see RequestTimeTooSkewed, the app will auto-detect and compensate. You can also manually set `clockSkewAdjustSec` in settings.
- **Preview issues**:
  - "Rendering timeout" for DOCX/ODT: file is too large (>5MB recommended limit). Use Download button.
  - "Showing X of Y rows" for XLSX/XLS/XLSB: spreadsheet exceeds 1000×50 limit. Use Download for full file.
  - Preview libraries fail to load: check browser console for errors. Switch to `native` preview engine in settings.
- **CORS Configuration**: YouTrack plugins run in iframe and send `Origin: null` header. S3/MinIO also requires CORS for browser file access. See complete setup guide: [SETUP_CORS.md](./SETUP_CORS.md).

Preview metadata (size/date) priority used by the app:

- Parameters passed from the file list (sizeBytes/lastModified) — highest priority.
- Response headers (Content-Length/Last-Modified) — used to fill missing fields when available via CORS ExposeHeaders.
- As a fallback, size is derived from the downloaded payload (ArrayBuffer/Blob) when headers/parameters are unavailable.

---

## Support

For questions and support, contact:

- Email: <apps@mag1.cc>

## License

This is proprietary software. See `LICENSE` and `EULA.md`.

- Distribution via JetBrains Marketplace — subscription/activation, entitlement scope (seats/instance) and billing are governed by JetBrains Marketplace. Our EULA also applies: `EULA.md`.
- Internal use by the licensor’s employer — allowed free of charge for the duration of employment and ends automatically upon its termination (see the simplified clause in `EULA.md`, details in `LICENSE`).
- This project is not affiliated with JetBrains. YouTrack is a trademark of JetBrains s.r.o. See `NOTICE`.
- Third‑party open‑source components are used under their respective licenses. See `THIRD_PARTY_NOTICES.md`.

## Vendor

- **Author**: mag1cc
- **Source**: [GitHub](https://github.com/mag1co/yta-adv-attachments-pub)
