# 🚀 Quick Fix: Bearer Token Not Working

## TL;DR

You **DON'T need OAuth 2.1 mode**. Simply use OAuth 2.0 mode with Bearer tokens.

## Immediate Fix

### 1. Update Your `.env` File

```bash
# Set to false or remove entirely (false is the default)
MCP_ENABLE_OAUTH21=false

# Remove this line if you have it
# EXTERNAL_OAUTH21_PROVIDER=true

# Keep these (required)
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# Add this (recommended)
WORKSPACE_MCP_STATELESS_MODE=true
```

### 2. Restart Your Server

```bash
python main.py --transport streamable-http --port 8000
```

### 3. Test It

```bash
# Replace YOUR_TOKEN with a valid Google OAuth access token
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "query": "is:unread",
      "max_results": 5
    }
  }'
```

**Note:** You don't need to include `user_google_email` anymore!

## What Changed

### Code Changes (Already Applied)

1. **`auth/service_decorator.py`:**
   - Made `user_google_email` optional when Bearer token is present
   - Falls back to authenticated user from token

2. **`auth/google_auth.py`:**
   - Checks OAuth21SessionStore for bearer token credentials
   - Works in OAuth 2.0 mode now

### What You Need to Do

1. ✅ Set `MCP_ENABLE_OAUTH21=false` (or remove it)
2. ✅ Remove `EXTERNAL_OAUTH21_PROVIDER` if you have it
3. ✅ Restart server
4. ✅ Test with Bearer token

## How Your SubAgent Should Call the Server

### Before (Required user_google_email)

```json
{
  "name": "search_gmail_messages",
  "arguments": {
    "user_google_email": "user@gmail.com",  // ❌ Had to include this
    "query": "is:unread"
  }
}
```

### After (user_google_email is optional)

```json
{
  "name": "search_gmail_messages",
  "arguments": {
    "query": "is:unread"  // ✅ user_google_email is automatic!
  }
}
```

The server automatically extracts the user email from the Bearer token.

## Expected Log Output

When working correctly, you should see:

```
INFO - Detected Google OAuth access token format
INFO - Authenticated via Google OAuth: user@gmail.com
INFO - [search_gmail_messages] Using authenticated user from bearer token: user@gmail.com
INFO - [search_gmail_messages] Authenticated gmail for user@gmail.com
```

## Troubleshooting

### Still Getting "user_google_email is required"?

**Check:**
1. Is `MCP_ENABLE_OAUTH21=false`?
2. Did you restart the server?
3. Is the Bearer token in the Authorization header?

### Token Validation Failing?

**Check:**
1. Token hasn't expired (Google tokens expire after 1 hour)
2. `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` are set
3. Credentials match the ones that issued the token

### Still Not Working?

**Check logs for:**
```
ERROR - Failed to verify Google OAuth token
```

**Possible causes:**
- Invalid token
- Wrong OAuth credentials
- Network issues connecting to Google

## Docker Quick Start

If using Docker:

```bash
docker run -p 8000:8000 \
  -e GOOGLE_OAUTH_CLIENT_ID=your-id \
  -e GOOGLE_OAUTH_CLIENT_SECRET=your-secret \
  -e MCP_ENABLE_OAUTH21=false \
  -e WORKSPACE_MCP_STATELESS_MODE=true \
  your-mcp-image
```

## Summary

✅ **Simple:** Just use OAuth 2.0 mode (the default)  
✅ **Works:** Bearer tokens are automatically validated  
✅ **Clean:** No need to pass `user_google_email` in every request  
✅ **Secure:** Token validated on every request  

## Next Steps

1. ✅ Update `.env` to disable OAuth 2.1
2. ✅ Restart server
3. ✅ Test with Bearer token
4. ✅ Update your SubAgent to omit `user_google_email` (optional)

## Need More Help?

See these docs:
- **Quick Start:** `BEARER_TOKEN_QUICKSTART.md`
- **Detailed Guide:** `EXTERNAL_OAUTH_SETUP.md`
- **What Changed:** `BEARER_TOKEN_AUTH_CHANGES.md`
