<!-- markdownlint-disable MD036 -->
# CORS Setup for Adv.Attachments

This document describes CORS configuration for two components:

1. **YouTrack REST API** — for the plugin to work inside an iframe
2. **S3/MinIO Storage** — for uploading and downloading files from the browser

---

## Part 1: CORS for YouTrack REST API

### The Problem

YouTrack plugins run inside an iframe that may be loaded via `srcdoc` or a special URL. In this case the browser sends `Origin: null` instead of the actual domain (e.g. `youtrack.office.lan`).

This is standard browser behavior for:

- iframe with `srcdoc` attribute
- iframe with `data:` URL
- iframe loaded via `file://`
- Some sandboxed iframe cases

### Solution 1: Configure CORS in YouTrack (Recommended)

#### Via REST API

Add **two** origins to the allowed list:

1. **Your YouTrack domain** (e.g. `https://youtrack.office.lan`) — for regular browser requests
2. **`null`** — for requests from plugin iframes

```bash
# 1. Get current CORS settings
curl -X GET "https://youtrack.office.lan/api/admin/globalSettings?fields=restSettings(allowAllOrigins,allowedOrigins)" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Accept: application/json"

# 2. Update CORS settings (add both origins)
curl -X POST "https://youtrack.office.lan/api/admin/globalSettings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "restSettings": {
      "allowedOrigins": ["https://youtrack.office.lan", "null"],
      "allowAllOrigins": false
    }
  }'
```

**Important:**

- Replace `https://youtrack.office.lan` with your actual YouTrack domain
- If you already have other origins in the list, include them too — otherwise they will be overwritten
- Both values (`https://youtrack.office.lan` and `null`) are required for correct operation

#### Via YouTrack Admin UI (Recommended)

1. Go to **Administration** → **Global Settings** → **REST API**
2. In the **CORS (Cross-Origin Resource Sharing)** / **Resource Sharing** section:
   - Uncheck **Allow requests from all origins** (if checked)
   - In the **Allowed Origins** field add:
     - `https://youtrack.office.lan` (your main domain)
     - `null` (for plugin iframes)
3. Save changes

📖 **Official documentation**: [YouTrack Server Configuration - Resource Sharing Settings](https://www.jetbrains.com/help/youtrack/server/2025.3/server-configuration-settings.html#resource-sharing-settings)

### Solution 2: allowAllOrigins (Not recommended for production)

For testing environments you can temporarily allow all origins:

```bash
curl -X POST "https://youtrack.office.lan/api/admin/globalSettings/restSettings" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "allowAllOrigins": true
  }'
```

⚠️ **Warning:** This is insecure for production environments!

### Solution 3: Web Server Level (Alternative)

If you have access to the YouTrack server configuration, you can configure CORS at the web server level (nginx/Apache).

#### nginx Example

```nginx
location /api/ {
    # Allow requests from null origin (for plugin iframes)
    if ($http_origin = "null") {
        add_header 'Access-Control-Allow-Origin' 'null' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
    }
    
    # Allow requests from the main domain
    if ($http_origin ~* "^https://youtrack\.office\.lan$") {
        add_header 'Access-Control-Allow-Origin' "$http_origin" always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
    }
    
    # Handle preflight requests
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    
    proxy_pass http://youtrack-backend;
}
```

### Verification

After configuring CORS, verify that requests work:

```bash
# Check preflight request
curl -X OPTIONS "https://youtrack.office.lan/api/admin/projects" \
  -H "Origin: null" \
  -H "Access-Control-Request-Method: GET" \
  -v

# Check regular request
curl -X GET "https://youtrack.office.lan/api/admin/projects" \
  -H "Origin: null" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -v
```

Response should include these headers:

- `Access-Control-Allow-Origin: null`
- `Access-Control-Allow-Methods: GET, POST, ...`
- `Access-Control-Allow-Headers: ...`

### Security

#### Why `null` origin is safe in this case

1. **Limited context** — `null` origin only appears for iframes already loaded within YouTrack
2. **Authentication** — all YouTrack API requests require an auth token
3. **Isolation** — plugins run in an isolated YouTrack context

#### Recommendations

- ✅ Use `allowedOrigins: ["https://youtrack.office.lan", "null"]` instead of `allowAllOrigins: true`
- ✅ Always require authentication for API requests
- ✅ Regularly check logs for suspicious activity
- ❌ Do not use `allowAllOrigins: true` in production

### Alternative (if CORS cannot be configured)

If you cannot change YouTrack CORS settings:

1. **Proxy via plugin backend** — route all YouTrack API calls through the backend HTTP handlers
2. **Use host.fetchYouTrack** — this method bypasses CORS since requests go through YouTrack, not directly from the iframe

### References

- [YouTrack REST API CORS Documentation](https://www.jetbrains.com/help/youtrack/devportal/operations-api-admin-globalSettings-restSettings.html)
- [MDN: Origin header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin)
- [CORS and null origin](https://stackoverflow.com/questions/8456538/origin-null-is-not-allowed-by-access-control-allow-origin)

---

## Part 2: CORS for S3/MinIO Storage

### Why S3/MinIO needs CORS

The app uploads and downloads files directly from the browser using presigned URLs. CORS must be configured on your S3/MinIO bucket to allow the browser to make these requests.

### AWS S3: CORS Setup

#### Via AWS Console

1. Open [AWS S3 Console](https://console.aws.amazon.com/s3/)
2. Select your bucket
3. Go to the **"Permissions"** tab
4. Scroll to **"Cross-origin resource sharing (CORS)"**
5. Click **"Edit"**
6. Paste the following configuration:

```json
[
    {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
        "AllowedOrigins": ["https://youtrack.yourcompany.com", "null"],
        "ExposeHeaders": [
            "Date",
            "ETag",
            "Content-Length",
            "Last-Modified",
            "x-amz-server-side-encryption",
            "x-amz-request-id",
            "x-amz-id-2"
        ],
        "MaxAgeSeconds": 3000
    }
]
```

**Important:**

- Replace `https://youtrack.yourcompany.com` with your YouTrack server URL
- `"null"` origin is required because the plugin runs inside a `srcdoc` iframe (browser sends `Origin: null`)
- `Date` in `ExposeHeaders` is required for automatic clock skew correction
- For testing you can use `"*"` in `AllowedOrigins`, but use a specific domain in production

#### Via AWS CLI

```bash
# Create cors.json file
cat > cors.json << 'EOF'
{
    "CORSRules": [
        {
            "AllowedHeaders": ["*"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedOrigins": ["https://youtrack.yourcompany.com", "null"],
            "ExposeHeaders": [
                "Date",
                "ETag",
                "Content-Length",
                "Last-Modified"
            ],
            "MaxAgeSeconds": 3000
        }
    ]
}
EOF

# Apply configuration
aws s3api put-bucket-cors \
    --bucket your-bucket-name \
    --cors-configuration file://cors.json
```

### MinIO: CORS Setup

#### Via MinIO Console

1. Open MinIO Console (usually `http://localhost:9001` or your MinIO URL)
2. Go to **"Buckets"** → select your bucket
3. Go to the **"Anonymous"** tab
4. Click **"Edit"** in the CORS section
5. Add CORS rules

#### Via MinIO Client (mc)

**Install MinIO Client:**

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```

**Configure alias:**

```bash
mc alias set myminio http://localhost:9000 minioadmin minioadmin123
```

**Create cors.json file:**

```json
{
    "CORSRules": [
        {
            "AllowedOrigins": ["https://youtrack.yourcompany.com", "null"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedHeaders": ["*"],
            "ExposeHeaders": ["Date", "ETag", "Content-Length", "Last-Modified"],
            "MaxAgeSeconds": 3000
        }
    ]
}
```

**Apply configuration:**

```bash
mc anonymous set-json cors.json myminio/your-bucket-name
```

**For testing (allow all origins):**

```json
{
    "CORSRules": [
        {
            "AllowedOrigins": ["*"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedHeaders": ["*"],
            "ExposeHeaders": ["Date", "ETag"],
            "MaxAgeSeconds": 3000
        }
    ]
}
```

### Verifying CORS Configuration

#### Via curl

```bash
# Replace URL with your S3/MinIO endpoint
curl -X OPTIONS "https://your-bucket.s3.amazonaws.com/" \
  -H "Origin: https://youtrack.yourcompany.com" \
  -H "Access-Control-Request-Method: GET" \
  -v
```

Response should include these headers:

- `Access-Control-Allow-Origin: https://youtrack.yourcompany.com`
- `Access-Control-Allow-Methods: GET, PUT, POST, DELETE, HEAD`
- `Access-Control-Expose-Headers: Date, ETag, ...`

#### Via browser

1. Open DevTools (F12)
2. Go to the **Network** tab
3. Try uploading a file through the app
4. Check requests to S3/MinIO — they should complete without CORS errors

### Common Issues

#### "CORS policy: No 'Access-Control-Allow-Origin' header"

**Cause:** CORS is not configured or misconfigured.

**Solution:**

1. Verify CORS configuration is applied to the bucket
2. Ensure `AllowedOrigins` contains your YouTrack URL
3. For MinIO, check that the correct bucket name is used

#### "CORS policy: Response to preflight request doesn't pass"

**Cause:** Required methods or headers are missing from the CORS configuration.

**Solution:**

1. Add all required methods: `GET`, `PUT`, `POST`, `DELETE`, `HEAD`
2. Add `AllowedHeaders: ["*"]`
3. Ensure the `OPTIONS` method is allowed (usually enabled automatically)

#### Clock skew detection not working

**Cause:** The `Date` header is not included in `ExposeHeaders`.

**Solution:**
Add `"Date"` to the `ExposeHeaders` list in the CORS configuration.

### S3/MinIO CORS Security

#### Production recommendations

- ✅ Specify your YouTrack domain in `AllowedOrigins` instead of `"*"`
- ✅ Limit `AllowedMethods` to only the required methods
- ✅ Use HTTPS for YouTrack and S3/MinIO endpoints
- ✅ Regularly review your CORS configuration
- ❌ Do not use `"AllowedOrigins": ["*"]` in production

#### For testing

You can temporarily use `"*"` in `AllowedOrigins`, but replace it with a specific domain before deploying to production.

### References (S3/MinIO)

- [AWS S3 CORS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html)
- [MinIO CORS Documentation](https://min.io/docs/minio/linux/administration/object-management.html#minio-bucket-cors)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
