# External OAuth Configuration Guide

## Overview

This MCP server can accept Google OAuth tokens from external systems without requiring OAuth 2.1 mode. This is the **simplest** configuration for accepting Bearer tokens from another authentication service.

## Problem with Previous Approach

The `EXTERNAL_OAUTH21_PROVIDER=true` setting required `MCP_ENABLE_OAUTH21=true`, which added unnecessary complexity:
- FastMCP's OAuth 2.1 protocol-level authentication
- Session management overhead
- Complex token flow

## Recommended Configuration

### For External OAuth (Bearer Tokens from Another System)

**Environment Variables:**
```bash
# OAuth credentials (required for API calls, not for OAuth flow)
export GOOGLE_OAUTH_CLIENT_ID="your-client-id"
export GOOGLE_OAUTH_CLIENT_SECRET="your-client-secret"

# Disable OAuth 2.1 mode - use simple Bearer token auth
export MCP_ENABLE_OAUTH21="false"

# Transport mode
export WORKSPACE_MCP_TRANSPORT="streamable-http"
export PORT="8000"

# Optional: Enable stateless mode (no file-based credential storage)
export WORKSPACE_MCP_STATELESS_MODE="true"
```

### How It Works

1. **Your External System** handles the OAuth flow with Google
2. **Your External System** sends requests with Bearer tokens:
   ```
   Authorization: Bearer ya29.a0AfB_byD...
   ```
3. **This Server**:
   - `AuthInfoMiddleware` extracts the token
   - `ExternalOAuthProvider.verify_token()` validates it with Google's userinfo API
   - User email is extracted and stored in FastMCP context
   - Tools can access Google APIs using the validated token

### Example Request from External System

```http
POST /mcp/v1/tools/call HTTP/1.1
Host: your-mcp-server.com
Authorization: Bearer ya29.a0AfB_byD...
Content-Type: application/json

{
  "name": "search_gmail_messages",
  "arguments": {
    "query": "from:example@gmail.com",
    "max_results": 10
  }
}
```

**Important:** The external system should **NOT** include `user_google_email` in the arguments when OAuth 2.1 mode is disabled. Instead, the server will:
1. Validate the Bearer token
2. Extract the user's email from the token
3. Automatically use that email for API calls

## OAuth 2.0 Mode vs OAuth 2.1 Mode

### OAuth 2.0 Mode (Recommended for External Tokens)
- ✅ Simple Bearer token authentication
- ✅ No protocol-level auth overhead
- ✅ Works with tokens from any OAuth provider
- ✅ Stateless operation support
- ✅ `user_google_email` parameter still required in tool calls

### OAuth 2.1 Mode (For FastMCP-Managed OAuth)
- Used when this server manages the OAuth flow
- Requires PKCE, state management
- Protocol-level authentication
- More complex but more secure for multi-user scenarios
- `user_google_email` parameter is automatically determined

## Testing Your Configuration

### 1. Check Server Status
```bash
curl http://localhost:8000/health
```

Expected response:
```json
{
  "status": "healthy",
  "service": "workspace-mcp",
  "version": "...",
  "transport": "streamable-http"
}
```

### 2. Test Tool Call with Bearer Token

```bash
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer ya29.YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "user_google_email": "user@gmail.com",
      "query": "is:unread",
      "max_results": 5
    }
  }'
```

### 3. Check Logs

Look for these log messages:
```
INFO - Detected Google OAuth access token format
INFO - Authenticated via Google OAuth: user@gmail.com
INFO - [search_gmail_messages] Authenticated gmail for user@gmail.com
```

## Troubleshooting

### "No valid 'user_google_email' provided"

**Problem:** The middleware isn't extracting the email from the token.

**Solutions:**
1. Ensure the Bearer token is valid (not expired)
2. Check that `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` are set
3. Verify the token has the required scopes
4. Check logs for token verification errors

### "OAuth credentials not found"

**Problem:** The server can't find OAuth client credentials.

**Solution:** Set `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` environment variables.

### "Access denied: Cannot retrieve credentials"

**Problem:** Security validation is blocking access (OAuth 2.1 mode enabled when it shouldn't be).

**Solution:** Ensure `MCP_ENABLE_OAUTH21=false` (or unset).

## Migration from OAuth 2.1 Mode

If you previously had:
```bash
export MCP_ENABLE_OAUTH21="true"
export EXTERNAL_OAUTH21_PROVIDER="true"
```

Change to:
```bash
export MCP_ENABLE_OAUTH21="false"
# Remove EXTERNAL_OAUTH21_PROVIDER - not needed
```

**No code changes required** - the middleware already handles Bearer tokens correctly in OAuth 2.0 mode.

## Security Considerations

### OAuth 2.0 Mode (Current Setup)
- Bearer tokens are validated against Google's userinfo API
- Each request requires a valid, non-expired Google OAuth token
- User identity is verified on every request
- No session state maintained server-side (optional with stateless mode)

### Best Practices
1. **Always use HTTPS** in production
2. **Validate tokens** on every request (already done by `ExternalOAuthProvider`)
3. **Rotate tokens** regularly in your external system
4. **Use short-lived tokens** (1 hour is Google's default)
5. **Implement rate limiting** to prevent abuse

## Architecture Diagram

```
┌─────────────────────┐
│  External System    │
│  (Your OAuth App)   │
└──────────┬──────────┘
           │ 1. User authenticates
           │ 2. Gets access_token (ya29.*)
           │
           ▼
┌─────────────────────┐
│   HTTP Request      │
│   Authorization:    │
│   Bearer ya29...    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  MCP Server (streamable-http)       │
├─────────────────────────────────────┤
│  1. AuthInfoMiddleware              │
│     - Extract Bearer token          │
│     - Call ExternalOAuthProvider    │
│       .verify_token()               │
│     - Validate with Google API      │
│     - Extract user email            │
│                                     │
│  2. Service Decorator               │
│     - Get email from context        │
│     - Create Google credentials     │
│     - Call Google API               │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│   Google APIs       │
│   (Gmail, Drive,    │
│    Calendar, etc.)  │
└─────────────────────┘
```

## FAQ

### Do I need to store credentials on the server?

No! With `WORKSPACE_MCP_STATELESS_MODE=true`, the server doesn't store any credentials. It validates tokens on each request.

### Can I use JWT tokens instead of Google OAuth tokens?

Yes, but you'll need to modify `AuthInfoMiddleware` to handle your JWT format. Google OAuth tokens (`ya29.*`) are currently recommended.

### What scopes do I need?

The external system's OAuth flow should request all scopes needed by the tools your users will call. See `auth/scopes.py` for the full list.

### Can I use this with Claude Desktop or other MCP clients?

This configuration is designed for HTTP transport. For stdio transport (Claude Desktop), use the standard OAuth 2.0 flow without Bearer tokens.
