# Bearer Token Authentication Quick Start

This guide shows how to use this MCP server with Bearer tokens from an external OAuth system.

## Overview

Your external system handles the OAuth flow with Google and obtains access tokens. This MCP server accepts those tokens and uses them to call Google APIs on behalf of authenticated users.

## Prerequisites

1. A Google Cloud project with OAuth 2.0 credentials
2. An external system that handles Google OAuth and obtains access tokens
3. Valid Google OAuth access tokens (format: `ya29.*`)

## Configuration

### 1. Create Environment File

```bash
cp .env.external_oauth_example .env
```

### 2. Set Your OAuth Credentials

Edit `.env` and set:

```bash
GOOGLE_OAUTH_CLIENT_ID=your-client-id-here.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret-here
```

**Important:** These credentials should match the ones used by your external OAuth system.

### 3. Configure OAuth Mode

Ensure OAuth 2.1 is **disabled** (this is the default):

```bash
MCP_ENABLE_OAUTH21=false
```

### 4. Enable Stateless Mode (Recommended)

```bash
WORKSPACE_MCP_STATELESS_MODE=true
```

This prevents the server from storing credentials in files.

## How It Works

### Request Flow

```
┌─────────────────────┐
│  Your Application   │
│  (External OAuth)   │
└──────────┬──────────┘
           │
           │ 1. User authenticates
           │ 2. Gets access_token
           │
           ▼
    Authorization: Bearer ya29.xxx
           │
           ▼
┌─────────────────────┐
│   MCP Server        │
│   (This server)     │
└──────────┬──────────┘
           │
           │ 3. Validates token
           │ 4. Extracts user email
           │ 5. Calls Google API
           │
           ▼
┌─────────────────────┐
│   Google APIs       │
└─────────────────────┘
```

### Authentication Process

1. **Your system** obtains a Google OAuth access token (ya29.*)
2. **Your system** sends API requests with the token:
   ```http
   POST /mcp/v1/tools/call
   Authorization: Bearer ya29.a0AfB_byD...
   Content-Type: application/json
   
   {
     "name": "search_gmail_messages",
     "arguments": {
       "query": "from:example@gmail.com"
     }
   }
   ```
3. **This server**:
   - Validates the token with Google's userinfo API
   - Extracts the user's email address
   - Uses the token to call Gmail API
   - Returns results

## API Examples

### Example 1: Search Gmail

```bash
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer ya29.YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "query": "is:unread",
      "max_results": 10
    }
  }'
```

**Note:** You don't need to include `user_google_email` - it's automatically extracted from the Bearer token!

### Example 2: List Google Drive Files

```bash
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer ya29.YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "list_files",
    "arguments": {
      "page_size": 20
    }
  }'
```

### Example 3: Create Calendar Event

```bash
curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer ya29.YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "create_event",
    "arguments": {
      "summary": "Team Meeting",
      "start_time": "2024-12-15T10:00:00-08:00",
      "end_time": "2024-12-15T11:00:00-08:00"
    }
  }'
```

## Running the Server

### Development

```bash
python main.py --transport streamable-http --port 8000
```

### Production (Docker)

```bash
docker build -t workspace-mcp .
docker run -p 8000:8000 \
  -e GOOGLE_OAUTH_CLIENT_ID=your-id \
  -e GOOGLE_OAUTH_CLIENT_SECRET=your-secret \
  -e WORKSPACE_MCP_STATELESS_MODE=true \
  -e MCP_ENABLE_OAUTH21=false \
  workspace-mcp
```

## Troubleshooting

### Issue: "user_google_email parameter is required"

**Cause:** The Bearer token couldn't be validated or doesn't contain an email.

**Solutions:**
1. Verify the token is valid (not expired)
2. Ensure `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` match your external system
3. Check that the token has the `email` scope
4. Look at server logs for token validation errors

### Issue: "Invalid or expired OAuth state parameter"

**Cause:** This error shouldn't occur with external OAuth. You're likely in OAuth 2.1 mode by mistake.

**Solution:** Ensure `MCP_ENABLE_OAUTH21=false` in your `.env` file.

### Issue: Token validation fails

**Cause:** The server can't validate the token with Google.

**Solutions:**
1. Verify OAuth credentials match between your external system and this server
2. Ensure the token hasn't expired (Google tokens expire after 1 hour)
3. Check network connectivity to Google APIs
4. Look for `AuthInfoMiddleware` errors in logs

### Issue: "Access denied: Cannot retrieve credentials"

**Cause:** OAuth 2.1 security mode is enabled.

**Solution:** Set `MCP_ENABLE_OAUTH21=false`

## Required Scopes

Your external OAuth system must request the appropriate scopes for the tools your users will call. Common scopes:

### Gmail
- `https://www.googleapis.com/auth/gmail.readonly` - Read email
- `https://www.googleapis.com/auth/gmail.send` - Send email
- `https://www.googleapis.com/auth/gmail.modify` - Modify email

### Google Drive
- `https://www.googleapis.com/auth/drive.readonly` - Read files
- `https://www.googleapis.com/auth/drive.file` - Manage files

### Google Calendar
- `https://www.googleapis.com/auth/calendar.readonly` - Read calendar
- `https://www.googleapis.com/auth/calendar.events` - Manage events

### Google Docs
- `https://www.googleapis.com/auth/documents.readonly` - Read documents
- `https://www.googleapis.com/auth/documents` - Edit documents

See `auth/scopes.py` for the complete list.

## Security Considerations

### Token Security
- Always use HTTPS in production
- Tokens are validated on every request
- Tokens expire after 1 hour (Google default)
- No credentials stored on disk (with stateless mode)

### User Isolation
- Each token is tied to a specific user
- Users can only access their own data
- Token validation happens on every request

### Best Practices
1. **Use short-lived tokens** - Google's 1-hour default is good
2. **Implement token refresh** in your external system
3. **Use HTTPS** - Never send Bearer tokens over HTTP
4. **Rate limiting** - Implement on your external system
5. **Logging** - Monitor for suspicious activity

## Testing

### 1. Start the Server

```bash
python main.py --transport streamable-http --port 8000
```

### 2. Check Health

```bash
curl http://localhost:8000/health
```

Expected:
```json
{
  "status": "healthy",
  "service": "workspace-mcp",
  "version": "...",
  "transport": "streamable-http"
}
```

### 3. Test with a Valid Token

Get a token from your external OAuth system, then:

```bash
export TOKEN="ya29.YOUR_TOKEN_HERE"

curl -X POST http://localhost:8000/mcp/v1/tools/call \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "query": "is:unread",
      "max_results": 1
    }
  }'
```

### 4. Check Logs

Look for:
```
INFO - Detected Google OAuth access token format
INFO - Authenticated via Google OAuth: user@gmail.com
INFO - [search_gmail_messages] Authenticated gmail for user@gmail.com
```

## Integration Examples

### Python Client

```python
import requests

def call_mcp_tool(tool_name, arguments, access_token):
    """Call an MCP tool with a Google OAuth access token."""
    response = requests.post(
        "http://localhost:8000/mcp/v1/tools/call",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        },
        json={
            "name": tool_name,
            "arguments": arguments
        }
    )
    return response.json()

# Example usage
token = "ya29.a0AfB_byD..."
result = call_mcp_tool(
    "search_gmail_messages",
    {"query": "from:example@gmail.com", "max_results": 10},
    token
)
print(result)
```

### JavaScript/TypeScript Client

```typescript
async function callMCPTool(
  toolName: string,
  arguments: Record<string, any>,
  accessToken: string
): Promise<any> {
  const response = await fetch("http://localhost:8000/mcp/v1/tools/call", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${accessToken}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      name: toolName,
      arguments
    })
  });
  return response.json();
}

// Example usage
const token = "ya29.a0AfB_byD...";
const result = await callMCPTool(
  "search_gmail_messages",
  { query: "from:example@gmail.com", max_results: 10 },
  token
);
console.log(result);
```

## Next Steps

1. **Review available tools** - See `README.md` for the full tool list
2. **Configure tool tiers** - Use `WORKSPACE_MCP_TIER` to limit available tools
3. **Set up monitoring** - Track API usage and errors
4. **Implement caching** - Cache responses in your external system
5. **Add rate limiting** - Protect against abuse

## Support

For issues or questions:
1. Check the [main documentation](EXTERNAL_OAUTH_SETUP.md)
2. Review server logs for detailed error messages
3. Verify OAuth credentials and token validity
4. Open an issue on GitHub with logs and configuration (redact secrets!)
