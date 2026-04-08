# Deploying Extensions

How to install, configure, enable, and remove Litium extensions — via the backoffice UI and the Extension Management API.

## Install via Settings > Extensions (UI)

1. Log in to the Litium backoffice
2. Click **Settings** in the left navigation
3. Click **Extensions** under the Settings section
4. Click **Install extension** (top right)
5. Either:
   - **Enter a manifest URL** — paste the URL to `extension.manifest.json` (e.g. `https://cdn.example.com/my-extension/extension.manifest.json`) and click **Fetch**
   - **Paste manifest JSON** — paste the full JSON content directly
6. Review the extension details
7. Click **Install**

The extension is enabled by default.

**Local development:** Use `http://localhost:3000/extension.manifest.json` as the manifest URL. The host loads bundles from your Vite dev server with HMR.

---

## Install via Extension Management API

For automated / CI deployments, use the REST API at `/Litium/api/extension-management/extensions`.

### Step 1: Obtain an OAuth Token

```bash
TOKEN=$(curl -s -X POST "{BACKEND_URL}/Litium/OAuth/Token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id={CLIENT_ID}&client_secret={CLIENT_SECRET}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

### Step 2: Check Existing Extensions

```bash
curl -s -X GET "{BACKEND_URL}/Litium/api/extension-management/extensions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" | python3 -m json.tool
```

### Step 3: Install (POST) or Update (PUT)

**Install new:**
```bash
curl -s -X POST "{BACKEND_URL}/Litium/api/extension-management/extensions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @extension.manifest.json | python3 -m json.tool
```

**Update existing:**
```bash
curl -s -X PUT "{BACKEND_URL}/Litium/api/extension-management/extensions/{EXT_ID}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @extension.manifest.json | python3 -m json.tool
```

### Step 4: Enable (if needed)

```bash
curl -s -X PATCH "{BACKEND_URL}/Litium/api/extension-management/extensions/{EXT_ID}/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"enabled": true}'
```

### Step 5: Verify

Check the extension page in the browser:
```
{BACKEND_URL}/Litium/UI/settings/extensions/{EXT_ID}/
```

---

## Enable / Disable (UI)

1. Find the extension in Settings > Extensions list
2. Click the three-dot menu (⋮)
3. Select **Enable** or **Disable**

Disabled extensions are not loaded — users see no UI from them.

---

## Update an Extension (UI)

1. Find the extension → click ⋮ → **Edit**
2. Modify fields or paste a new manifest
3. Click **Save**

Alternatively, re-install with the same `id` — the system updates rather than duplicating.

---

## Remove an Extension

**Via UI:** Find the extension → ⋮ → **Delete** → confirm.

**Via API:**
```bash
curl -s -X DELETE "{BACKEND_URL}/Litium/api/extension-management/extensions/{EXT_ID}" \
  -H "Authorization: Bearer $TOKEN"
```

Returns `204 No Content` on success.

---

## Production Build Workflow

```bash
# 1. Build the IIFE bundle
npm run build

# 2. Output: dist/extension.js + dist/extension.manifest.json

# 3. Upload dist/extension.js to your CDN (HTTPS required for production)

# 4. Update bundleUrl in extension.manifest.json to the CDN URL
#    e.g. "bundleUrl": "https://cdn.example.com/my-extension/1.0.0/extension.js"

# 5. Register or update via Settings > Extensions or the API
```

---

## CI/CD Deployment (GitHub Actions)

```yaml
name: Deploy Extension
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Configure registry
        run: echo "@litiumab:registry=https://registry.npmjs.org/" >> .npmrc

      - run: npm ci
      - run: npm run build
      - run: npm test

      # Upload dist/extension.js to your CDN (varies by provider)
      # ...

      # Register/update the extension via API
      - name: Register extension
        run: |
          TOKEN=$(curl -s -X POST "${{ secrets.LITIUM_URL }}/Litium/OAuth/Token" \
            -H "Content-Type: application/x-www-form-urlencoded" \
            -d "grant_type=client_credentials&client_id=${{ secrets.CLIENT_ID }}&client_secret=${{ secrets.CLIENT_SECRET }}" \
            | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

          curl -s -X PUT "${{ secrets.LITIUM_URL }}/Litium/api/extension-management/extensions" \
            -H "Authorization: Bearer $TOKEN" \
            -H "Content-Type: application/json" \
            -d @dist/extension.manifest.json
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| 401 Unauthorized | Token expired or invalid | Re-request OAuth token |
| 403 Forbidden on GET/PUT/DELETE | Extension owned by different client | Use the same `client_id` that created it |
| Extension page blank | Dev server not running or CORS issue | Ensure `npm run dev` is running; check browser console |
| Extension not in menu | Not installed or disabled | Check via API; enable if disabled |
| `net::ERR_CONNECTION_REFUSED` in console | Dev server port mismatch | Check `.dev.config.json` port matches manifest `bundleUrl` |
