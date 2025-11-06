# Fork Modifications

This document tracks all modifications made to the upstream [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) repository for use in the **AI Receptionist** project.

## Purpose of This Fork

This fork is maintained by **0x44dd22** to support **Bearer token authentication** in **stateless mode** for the AI Receptionist project. Our architecture differs from upstream:

- **Upstream**: Server manages OAuth flows itself (OAuth 2.1 with PKCE)
- **Our Architecture**: Java backend manages OAuth, stores encrypted tokens in Firestore, passes Bearer tokens to MCP server

## Active Modifications

### 1. Remove OAuth 2.1 Requirement for Stateless Mode

**File**: `auth/oauth_config.py`  
**Lines**: 44-48  
**Date Modified**: 2025-11-06

**Original Code**:
```python
# Stateless mode configuration
self.stateless_mode = os.getenv("WORKSPACE_MCP_STATELESS_MODE", "false").lower() == "true"
if self.stateless_mode and not self.oauth21_enabled:
    raise ValueError("WORKSPACE_MCP_STATELESS_MODE requires MCP_ENABLE_OAUTH21=true")
```

**Modified Code**:
```python
# Stateless mode configuration
self.stateless_mode = os.getenv("WORKSPACE_MCP_STATELESS_MODE", "false").lower() == "true"
# Note: Stateless mode can work WITHOUT OAuth 2.1 when using Bearer token authentication
# OAuth 2.1 is only required if the server manages OAuth flows itself
```

**Reason**: The upstream validation assumes stateless mode requires OAuth 2.1 session management. However, our architecture uses Bearer token authentication where:
- Java backend manages OAuth flows
- Tokens are encrypted and stored in Firestore
- MCP server receives tokens via `Authorization: Bearer <token>` header
- No OAuth session management needed in MCP server

**Impact**: Without this change, server crashes on startup with:
```
ValueError: WORKSPACE_MCP_STATELESS_MODE requires MCP_ENABLE_OAUTH21=true
```

---

## Maintenance Workflow

### Syncing with Upstream

```bash
# Fetch latest changes from upstream
git fetch upstream

# Merge upstream changes into our branch
git checkout ai-receptionist-bearer-auth
git merge upstream/main

# Check if our modification is still present
git diff HEAD auth/oauth_config.py

# If modification was overwritten, reapply it
# (see modification details above)

# Push to our fork
git push origin ai-receptionist-bearer-auth
```

### Testing After Sync

```bash
# Verify the server starts without ValueError
python3 -c "
import os
os.environ['WORKSPACE_MCP_STATELESS_MODE'] = 'true'
from auth.oauth_config import OAuthConfig
config = OAuthConfig()
print('✓ Server configuration successful')
print(f'Stateless mode: {config.stateless_mode}')
print(f'OAuth 2.1 enabled: {config.oauth21_enabled}')
"
```

Expected output:
```
✓ Server configuration successful
Stateless mode: True
OAuth 2.1 enabled: False
```

---

## Deployment to Google Cloud Run

### Prerequisites

1. **Google Cloud CLI installed and authenticated**
   ```bash
   # Check authentication
   gcloud auth list
   
   # Check project is set
   gcloud config get-value project
   # Should show: ai-receptionist-a412e
   ```

2. **Required APIs enabled** (already enabled for ai-receptionist-a412e)
   - Cloud Run API
   - Cloud Build API

3. **Source code on the correct branch**
   ```bash
   git checkout ai-receptionist-bearer-auth
   git pull origin ai-receptionist-bearer-auth
   ```

### Deployment Command

**Full deployment with all options:**
```bash
gcloud run deploy workspace-mcp \
  --source . \
  --platform managed \
  --region us-east1 \
  --allow-unauthenticated \
  --set-env-vars "WORKSPACE_MCP_STATELESS_MODE=true" \
  --memory 1Gi \
  --cpu 1 \
  --cpu-boost \
  --timeout 300 \
  --max-instances 10 \
  --min-instances 0 \
  --project ai-receptionist-a412e
```

**Minimal deployment (recommended for first deploy):**
```bash
gcloud run deploy workspace-mcp \
  --source . \
  --region us-east1 \
  --allow-unauthenticated \
  --set-env-vars "WORKSPACE_MCP_STATELESS_MODE=true" \
  --project ai-receptionist-a412e
```

### Deployment Process

1. **Build Phase** (~3-5 minutes)
   - Cloud Build creates Docker image from source
   - Uses Dockerfile in repository root
   - Installs dependencies via `uv sync`

2. **Deploy Phase** (~1-2 minutes)
   - Deploys container to Cloud Run
   - Configures networking and scaling
   - Makes service available at URL

3. **Output**
   ```
   Building using Dockerfile and deploying container to Cloud Run service...
   ✓ Building and deploying new service... Done.
     ✓ Creating Revision...
     ✓ Routing traffic...
   Done.
   Service [workspace-mcp] revision [workspace-mcp-00001-abc] has been deployed.
   Service URL: https://workspace-mcp-623592001761.us-east1.run.app
   ```

### Configuration Options

**Environment Variables:**
```bash
# Minimal (Bearer token auth only)
WORKSPACE_MCP_STATELESS_MODE=true

# With OAuth credentials (for token refresh)
WORKSPACE_MCP_STATELESS_MODE=true
GOOGLE_OAUTH_CLIENT_ID=<your-client-id>
GOOGLE_OAUTH_CLIENT_SECRET=<your-client-secret>
```

**Resource Configuration:**
- `--memory 1Gi` - Memory allocation (default: 512Mi)
- `--cpu 1` - CPU allocation (default: 1)
- `--cpu-boost` - Extra CPU during startup
- `--timeout 300` - Request timeout in seconds (default: 300)

**Scaling Configuration:**
- `--min-instances 0` - Scale to zero when idle (saves money)
- `--max-instances 10` - Maximum concurrent instances
- `--concurrency 80` - Requests per instance (default: 80)

### Verifying Deployment

**1. Check deployment status:**
```bash
gcloud run services describe workspace-mcp \
  --region us-east1 \
  --project ai-receptionist-a412e
```

**2. Test health endpoint:**
```bash
curl https://workspace-mcp-623592001761.us-east1.run.app/health
```

Expected response:
```json
{"status": "healthy"}
```

**3. View logs:**
```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=workspace-mcp" \
  --limit=50 \
  --project=ai-receptionist-a412e \
  --format="table(timestamp,textPayload)"
```

**4. Test with Bearer token:**
```bash
# Get a Google access token (replace with actual token)
export GOOGLE_ACCESS_TOKEN="ya29.a0AfH6SMB..."

# List available tools
curl -X POST https://workspace-mcp-623592001761.us-east1.run.app/mcp \
  -H "Authorization: Bearer $GOOGLE_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "params": {},
    "id": 1
  }'
```

### Redeployment

**To deploy updates after code changes:**
```bash
# Ensure you're on the correct branch
git checkout ai-receptionist-bearer-auth
git pull origin ai-receptionist-bearer-auth

# Deploy (same command as initial deployment)
gcloud run deploy workspace-mcp \
  --source . \
  --region us-east1 \
  --set-env-vars "WORKSPACE_MCP_STATELESS_MODE=true" \
  --project ai-receptionist-a412e
```

### Updating Environment Variables Only

```bash
gcloud run services update workspace-mcp \
  --region us-east1 \
  --update-env-vars "WORKSPACE_MCP_STATELESS_MODE=true,GOOGLE_OAUTH_CLIENT_ID=new-value" \
  --project ai-receptionist-a412e
```

### Troubleshooting

**Build fails:**
- Check Dockerfile syntax
- Verify pyproject.toml dependencies
- View build logs: `gcloud builds list --project ai-receptionist-a412e`

**Service fails to start:**
- Check logs: `gcloud logging read ...` (see command above)
- Verify environment variables are set correctly
- Test locally first: `docker build -t workspace-mcp . && docker run -p 8000:8000 workspace-mcp`

**503 Service Unavailable:**
- Cold start delay (first request after idle)
- Check if service is deployed: `gcloud run services list --project ai-receptionist-a412e`
- Increase timeout if needed: `--timeout 600`

**401 Unauthorized:**
- Bearer token missing or invalid
- Token expired (Java backend needs to refresh)
- Verify token has correct Google OAuth scopes

### Cost Estimation

**With default settings (min-instances=0):**
- **Idle**: $0/month (scales to zero)
- **Active**: ~$1-5/month for typical usage
- **Heavy load**: ~$10-50/month (depends on request volume)

**Pricing factors:**
- CPU/memory allocation
- Request count
- Request duration
- Network egress

### Service URL

After deployment, the service is available at:
```
https://workspace-mcp-623592001761.us-east1.run.app
```

**Endpoints:**
- Health check: `GET /health`
- MCP requests: `POST /mcp`

---

## Post-Deployment Integration

### Java Backend Configuration

Update your Java backend to use the deployed service URL:

```java
// application.yml or environment variables
mcp:
  workspace:
    url: https://workspace-mcp-623592001761.us-east1.run.app/mcp
```

### Testing Integration

```bash
# From Java backend, make a test request
curl -X POST https://workspace-mcp-623592001761.us-east1.run.app/mcp \
  -H "Authorization: Bearer ${USER_GOOGLE_ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "calendar_list_events",
      "arguments": {
        "max_results": 5
      }
    },
    "id": 1
  }'
```

---

## Version History

| Date | Upstream Version | Our Version | Changes |
|------|------------------|-------------|---------|
| 2025-11-06 | main @ latest | ai-receptionist-bearer-auth | Initial fork setup, applied OAuth 2.1 modification |

---

## Architecture Context

### How This Fork Fits In

```
Phone Call (Twilio)
    ↓
Java Backend (AWS Fargate)
    ↓ (fetch encrypted tokens)
Firestore
    ↓ (decrypt tokens)
AWS KMS
    ↓ (Bearer token auth)
Google Workspace MCP Server (Google Cloud Run) ← THIS FORK
    ↓ (Google API calls)
Google Calendar/Gmail/Drive APIs
```

### Environment Variables

```bash
WORKSPACE_MCP_STATELESS_MODE=true        # No disk storage
GOOGLE_OAUTH_CLIENT_ID=<client-id>       # For token refresh only
GOOGLE_OAUTH_CLIENT_SECRET=<secret>      # For token refresh only
```

**Note**: OAuth credentials are optional and only used if the server needs to refresh tokens. Primary authentication is via Bearer tokens from the Java backend.

---

## Support

- **Fork Owner**: 0x44dd22 organization
- **Main Project**: ai-receptionist-be
- **Upstream Issues**: https://github.com/taylorwilsdon/google_workspace_mcp/issues
- **Our Branch**: `ai-receptionist-bearer-auth`
- **Deployment**: Google Cloud Run (`workspace-mcp` service)

---

## Summary

✅ **Single file modified**: `auth/oauth_config.py` (3 lines changed)  
✅ **Purpose**: Support Bearer token auth in stateless mode  
✅ **Impact**: Removes ValueError that blocks server startup  
✅ **Maintenance**: Reapply after upstream merges  
