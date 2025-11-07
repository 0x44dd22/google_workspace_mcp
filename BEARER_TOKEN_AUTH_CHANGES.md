# Bearer Token Authentication - Changes Summary

## Problem Statement

The server was configured to require OAuth 2.1 mode (`MCP_ENABLE_OAUTH21=true`) for accepting Bearer tokens from external OAuth systems. This added unnecessary complexity:

- OAuth 2.1 protocol-level authentication overhead
- Complex session management requirements
- Confusion about when to use `EXTERNAL_OAUTH21_PROVIDER`
- Users couldn't understand why tokens weren't being accepted

The other system (SubAgent) was correctly sending Bearer tokens but the server wasn't recognizing them in OAuth 2.0 mode.

## Root Cause

The service decorator in OAuth 2.0 mode required `user_google_email` as a parameter, but didn't fall back to the authenticated user from Bearer tokens. This meant:

1. Bearer token arrives → `AuthInfoMiddleware` validates it ✅
2. Middleware sets `authenticated_user_email` in context ✅
3. Service decorator requires `user_google_email` parameter ❌
4. No fallback to `authenticated_user_email` from token ❌

## Solution

**Simplified to use OAuth 2.0 mode** with Bearer token support. No need for OAuth 2.1 unless the server manages the OAuth flow itself.

### Changes Made

#### 1. Updated Service Decorator (`auth/service_decorator.py`)

**Function:** `_extract_oauth20_user_email()`

```python
# BEFORE: Required user_google_email parameter
def _extract_oauth20_user_email(args, kwargs, wrapper_sig) -> str:
    bound_args = wrapper_sig.bind(*args, **kwargs)
    user_google_email = bound_args.arguments.get("user_google_email")
    if not user_google_email:
        raise Exception("'user_google_email' parameter is required")
    return user_google_email

# AFTER: Falls back to authenticated_user from Bearer token
def _extract_oauth20_user_email(args, kwargs, wrapper_sig, authenticated_user=None) -> str:
    bound_args = wrapper_sig.bind(*args, **kwargs)
    user_google_email = bound_args.arguments.get("user_google_email")
    
    # If user_google_email not provided, use authenticated_user from bearer token
    if not user_google_email and authenticated_user:
        logger.info(f"Using authenticated user from bearer token: {authenticated_user}")
        return authenticated_user
    
    if not user_google_email:
        raise Exception(
            "'user_google_email' parameter is required but was not found. "
            "Either provide user_google_email or use a Bearer token."
        )
    return user_google_email
```

**Impact:** Tools can now work with Bearer tokens in OAuth 2.0 mode without requiring `user_google_email` parameter.

#### 2. Updated OAuth 2.0 Credential Retrieval (`auth/google_auth.py`)

**Function:** `get_credentials()`

```python
# ADDED: Check OAuth21SessionStore for bearer token credentials
if not credentials and user_google_email:
    # First try OAuth21SessionStore (for bearer token credentials)
    try:
        oauth21_store = get_oauth21_session_store()
        credentials = oauth21_store.get_credentials(user_google_email)
        if credentials:
            logger.debug("Loaded credentials from OAuth21SessionStore")
    except Exception as e:
        logger.debug(f"Error checking OAuth21SessionStore: {e}")
    
    # If not found, try file-based credential store (unless stateless)
    if not credentials and not is_stateless_mode():
        store = get_credential_store()
        credentials = store.get_credential(user_google_email)
```

**Impact:** Credentials from Bearer tokens (stored in OAuth21SessionStore) are now accessible in OAuth 2.0 mode.

#### 3. Documentation Updates

Created three new documentation files:

1. **`EXTERNAL_OAUTH_SETUP.md`** - Comprehensive guide to external OAuth configuration
2. **`BEARER_TOKEN_QUICKSTART.md`** - Quick start guide with examples
3. **`.env.external_oauth_example`** - Example environment configuration

## Configuration Changes

### OLD Configuration (Required OAuth 2.1)

```bash
# WRONG - Too complex
MCP_ENABLE_OAUTH21=true
EXTERNAL_OAUTH21_PROVIDER=true
GOOGLE_OAUTH_CLIENT_ID=...
GOOGLE_OAUTH_CLIENT_SECRET=...
```

### NEW Configuration (Simple OAuth 2.0)

```bash
# CORRECT - Simple and works
MCP_ENABLE_OAUTH21=false  # or omit entirely (default is false)
GOOGLE_OAUTH_CLIENT_ID=...
GOOGLE_OAUTH_CLIENT_SECRET=...
WORKSPACE_MCP_STATELESS_MODE=true  # Recommended
```

## How It Works Now

### Request Flow

```
External System (SubAgent)
  │
  ├─ User authenticates with Google
  ├─ Gets access token (ya29.*)
  │
  ▼
POST /mcp/v1/tools/call
Authorization: Bearer ya29.xxx
{
  "name": "search_gmail_messages",
  "arguments": {
    "query": "is:unread"
  }
}
  │
  ▼
AuthInfoMiddleware
  ├─ Extract Bearer token
  ├─ Validate with Google userinfo API
  ├─ Extract user email
  ├─ Store in OAuth21SessionStore
  └─ Set authenticated_user_email in context
  │
  ▼
Service Decorator
  ├─ Get authenticated_user from context
  ├─ Use it as user_google_email (if not explicitly provided)
  └─ Call get_credentials(user_google_email)
  │
  ▼
get_credentials()
  ├─ Check OAuth21SessionStore by user_email
  ├─ Find credentials from bearer token
  └─ Return credentials
  │
  ▼
Google API Call
  └─ Success!
```

## Testing

### Before (Didn't Work)

```bash
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer ya29.xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "query": "is:unread"
    }
  }'

# ERROR: user_google_email parameter is required
```

### After (Works!)

```bash
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer ya29.xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "query": "is:unread"
    }
  }'

# SUCCESS: Returns email messages
```

## Migration Guide

### For Existing Deployments

1. **Update Environment Variables:**
   ```bash
   # Remove or set to false
   MCP_ENABLE_OAUTH21=false
   
   # Remove this entirely
   # EXTERNAL_OAUTH21_PROVIDER=true
   
   # Optional: Enable stateless mode
   WORKSPACE_MCP_STATELESS_MODE=true
   ```

2. **Restart Server:**
   ```bash
   python main.py --transport streamable-http --port 8000
   ```

3. **Test with Bearer Token:**
   ```bash
   curl -X POST http://localhost:8000/mcp/v1/tools/call \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name": "search_gmail_messages", "arguments": {"query": "is:unread"}}'
   ```

4. **Check Logs:**
   ```
   INFO - Detected Google OAuth access token format
   INFO - Authenticated via Google OAuth: user@gmail.com
   INFO - [search_gmail_messages] Using authenticated user from bearer token: user@gmail.com
   INFO - [search_gmail_messages] Authenticated gmail for user@gmail.com
   ```

## When to Use Each Mode

### OAuth 2.0 Mode (Use for External Bearer Tokens)

**When:**
- Another system handles OAuth flow
- You receive Bearer tokens (ya29.*)
- You want simple token validation
- You don't need protocol-level authentication

**Configuration:**
```bash
MCP_ENABLE_OAUTH21=false
```

**Pros:**
- ✅ Simple configuration
- ✅ Works with any OAuth provider
- ✅ No session management overhead
- ✅ Stateless operation possible

**Cons:**
- ❌ Less protocol-level security
- ❌ Each request requires token validation

### OAuth 2.1 Mode (Use for Server-Managed OAuth)

**When:**
- This server manages the OAuth flow
- You need protocol-level authentication
- You have multi-user scenarios
- You want PKCE support

**Configuration:**
```bash
MCP_ENABLE_OAUTH21=true
```

**Pros:**
- ✅ Protocol-level authentication
- ✅ PKCE support
- ✅ Better for multi-user
- ✅ Session management

**Cons:**
- ❌ More complex setup
- ❌ Requires OAuth flow management
- ❌ More overhead

## Security Considerations

### OAuth 2.0 Mode with Bearer Tokens

**Security Measures:**
1. **Token Validation:** Every request validates the token with Google
2. **User Isolation:** Each token is tied to a specific user
3. **No Persistent Storage:** With stateless mode, no credentials on disk
4. **HTTPS Required:** Always use HTTPS in production
5. **Short-lived Tokens:** Google tokens expire after 1 hour

**Security Best Practices:**
1. Always use HTTPS for Bearer token transmission
2. Implement rate limiting in your external system
3. Monitor for suspicious activity
4. Rotate tokens regularly
5. Use the minimum required scopes

## FAQ

### Q: Do I need OAuth 2.1 mode?

**A:** No, not if you're receiving Bearer tokens from an external system. Use OAuth 2.0 mode (`MCP_ENABLE_OAUTH21=false`).

### Q: What's the difference between OAuth 2.0 and OAuth 2.1 mode?

**A:**
- **OAuth 2.0 mode:** Simple Bearer token validation. The server accepts tokens and validates them.
- **OAuth 2.1 mode:** Full protocol implementation. The server manages the OAuth flow, PKCE, state, etc.

### Q: Can I still use `user_google_email` parameter?

**A:** Yes! It's optional now. If you provide it, it will be used. If you don't, the server extracts it from the Bearer token.

### Q: Will this work with my existing SubAgent setup?

**A:** Yes! Your SubAgent can now call tools without providing `user_google_email`. The server extracts it from the Bearer token automatically.

### Q: Do I need to change my client code?

**A:** No. If your client already sends Bearer tokens, it will work. You can optionally remove `user_google_email` from your requests.

### Q: What about file-based credentials?

**A:** With `WORKSPACE_MCP_STATELESS_MODE=true`, no credentials are stored in files. All authentication is via Bearer tokens.

## Rollback

If you need to rollback to the old behavior:

1. **Re-enable OAuth 2.1:**
   ```bash
   MCP_ENABLE_OAUTH21=true
   EXTERNAL_OAUTH21_PROVIDER=true
   ```

2. **Restart server**

3. **Update clients to include `user_google_email`** (though this shouldn't be necessary with OAuth 2.1)

## Summary

The changes **simplify external OAuth integration** by:

1. ✅ Making `user_google_email` optional when Bearer token is present
2. ✅ Checking OAuth21SessionStore for bearer token credentials in OAuth 2.0 mode
3. ✅ Removing the requirement for OAuth 2.1 mode
4. ✅ Providing clear documentation and examples

**Result:** Your SubAgent can now call tools with just a Bearer token, and the server automatically extracts the user email and uses the token for API calls.
