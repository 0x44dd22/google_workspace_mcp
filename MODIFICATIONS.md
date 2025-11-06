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

### Deployment After Sync

See deployment instructions in the AI Receptionist project:
- `ai-receptionist-be/mcp-servers/google-workspace/README.md`
- `ai-receptionist-be/mcp-servers/google-workspace/DEPLOYMENT.md`

Quick deploy:
```bash
gcloud run deploy workspace-mcp \
  --source . \
  --platform managed \
  --region us-east1 \
  --allow-unauthenticated \
  --set-env-vars "WORKSPACE_MCP_STATELESS_MODE=true" \
  --project ai-receptionist-a412e
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
