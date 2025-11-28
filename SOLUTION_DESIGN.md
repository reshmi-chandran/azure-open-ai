# Microsoft Graph API Integration - OneDrive for Business Read Access
## Solution Design Document

---

## Overview

This solution enables an external application (ChatGPT) to read files from OneDrive for Business using Microsoft Graph API with OAuth2 client credentials flow. The integration provides read-only access to files, file lists, and metadata.

---

## Architecture

```
External App (ChatGPT)
    ↓
Azure AD App Registration (Client Credentials)
    ↓
Microsoft Graph API
    ↓
OneDrive for Business
```

**Authentication Flow:** OAuth2 Client Credentials (App-only authentication)

---

## Step 1: Azure App Registration Setup

### 1.1 Create App Registration

1. Navigate to [Azure Portal](https://portal.azure.com)
2. Go to **Azure Active Directory** → **App registrations**
3. Click **+ New registration**
4. Configure:
   - **Name:** `OneDrive-Read-Only-Integration` (or your preferred name)
   - **Supported account types:** `Accounts in this organizational directory only`
   - **Redirect URI:** Leave blank (not needed for client credentials flow)
5. Click **Register**

### 1.2 Record Application IDs

After registration, note down:
- **Application (client) ID** - This is your `APP_ID`
- **Directory (tenant) ID** - This is your `TENANT_ID`

---

## Step 2: Configure Application Permissions

### 2.1 Add Microsoft Graph API Permissions

1. In your App Registration, go to **API permissions**
2. Click **+ Add a permission**
3. Select **Microsoft Graph**
4. Choose **Application permissions** (not Delegated)
5. Add the following permissions:
   - `Files.Read.All` - Read all files that the app can access
6. Click **Add permissions**

### 2.2 Grant Admin Consent

⚠️ **Critical Step:** You must grant admin consent for the permissions to work.

1. In the **API permissions** page, click **Grant admin consent for [Your Organization]**
2. Click **Yes** to confirm
3. Verify that the status shows "Granted for [Your Organization]" with a green checkmark

---

## Step 3: Create Client Secret

### 3.1 Generate Secret

1. In your App Registration, go to **Certificates & secrets**
2. Click **+ New client secret**
3. Configure:
   - **Description:** `OneDrive Read Access Secret`
   - **Expires:** Choose your preferred duration (6 months, 12 months, or 24 months)
4. Click **Add**
5. **⚠️ IMPORTANT:** Copy the **Value** immediately (it won't be shown again)
   - This is your `CLIENT_SECRET`

---

## Step 4: Required Credentials Summary

After completing the above steps, you will have:

| Credential | Description | Where to Find |
|------------|-------------|---------------|
| **TENANT_ID** | Your Azure AD tenant ID | App Registration → Overview → Directory (tenant) ID |
| **APP_ID** | Application (client) ID | App Registration → Overview → Application (client) ID |
| **CLIENT_SECRET** | Client secret value | App Registration → Certificates & secrets → Value |

---

## Step 5: Microsoft Graph API Endpoints

### 5.1 Authentication Endpoint

```
POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token
```

### 5.2 OneDrive API Endpoints

#### List Files in OneDrive Root
```
GET https://graph.microsoft.com/v1.0/me/drive/root/children
```

#### List Files in Specific Folder
```
GET https://graph.microsoft.com/v1.0/me/drive/root:/{folder-path}:/children
```

#### Get File Content
```
GET https://graph.microsoft.com/v1.0/me/drive/root:/{file-path}:/content
```

#### Get File Metadata
```
GET https://graph.microsoft.com/v1.0/me/drive/root:/{file-path}
```

#### Get File by ID
```
GET https://graph.microsoft.com/v1.0/me/drive/items/{item-id}/content
```

#### Get File Metadata by ID
```
GET https://graph.microsoft.com/v1.0/me/drive/items/{item-id}
```

---

## Step 6: Authentication Flow

### 6.1 Obtain Access Token

**Request:**
```http
POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id={APP_ID}
&scope=https://graph.microsoft.com/.default
&client_secret={CLIENT_SECRET}
&grant_type=client_credentials
```

**Response:**
```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJ0eXAiOiJKV1QiLCJub..."
}
```

### 6.2 Use Access Token

Include the token in the Authorization header:
```http
Authorization: Bearer {access_token}
```

---

## Step 7: Working Examples

### 7.1 Python Example

```python
import requests
import json
from datetime import datetime, timedelta

# Configuration
TENANT_ID = "your-tenant-id-here"
APP_ID = "your-app-id-here"
CLIENT_SECRET = "your-client-secret-here"

# Token cache
token_cache = {
    "token": None,
    "expires_at": None
}

def get_access_token():
    """Obtain or refresh access token"""
    # Check if token is still valid (with 5 minute buffer)
    if (token_cache["token"] and 
        token_cache["expires_at"] and 
        datetime.now() < token_cache["expires_at"] - timedelta(minutes=5)):
        return token_cache["token"]
    
    # Request new token
    token_url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
    
    token_data = {
        "client_id": APP_ID,
        "scope": "https://graph.microsoft.com/.default",
        "client_secret": CLIENT_SECRET,
        "grant_type": "client_credentials"
    }
    
    response = requests.post(token_url, data=token_data)
    response.raise_for_status()
    
    token_info = response.json()
    token_cache["token"] = token_info["access_token"]
    token_cache["expires_at"] = datetime.now() + timedelta(seconds=token_info["expires_in"])
    
    return token_cache["token"]

def list_files_in_root():
    """List all files in OneDrive root"""
    token = get_access_token()
    
    url = "https://graph.microsoft.com/v1.0/me/drive/root/children"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    }
    
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    
    return response.json()

def get_file_content(file_path):
    """Download file content by path"""
    token = get_access_token()
    
    # URL encode the file path
    encoded_path = file_path.replace("/", ":/").replace(" ", "%20")
    url = f"https://graph.microsoft.com/v1.0/me/drive/root:/{encoded_path}:/content"
    
    headers = {
        "Authorization": f"Bearer {token}"
    }
    
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    
    return response.content

def get_file_metadata(file_path):
    """Get file metadata by path"""
    token = get_access_token()
    
    encoded_path = file_path.replace("/", ":/").replace(" ", "%20")
    url = f"https://graph.microsoft.com/v1.0/me/drive/root:/{encoded_path}"
    
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    }
    
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    
    return response.json()

# Example Usage
if __name__ == "__main__":
    try:
        # List files
        print("Listing files in OneDrive root...")
        files = list_files_in_root()
        
        print(f"\nFound {len(files.get('value', []))} items:")
        for item in files.get('value', []):
            print(f"  - {item['name']} ({item.get('size', 0)} bytes)")
            if 'file' in item:
                print(f"    Modified: {item.get('lastModifiedDateTime', 'N/A')}")
        
        # Get content of first file (if any)
        if files.get('value'):
            first_file = files['value'][0]
            if 'file' in first_file:
                print(f"\nReading content of: {first_file['name']}")
                content = get_file_content(first_file['name'])
                print(f"File size: {len(content)} bytes")
                
                # Get metadata
                metadata = get_file_metadata(first_file['name'])
                print(f"Metadata: {json.dumps(metadata, indent=2)}")
    
    except requests.exceptions.HTTPError as e:
        print(f"Error: {e}")
        print(f"Response: {e.response.text}")
```

### 7.2 cURL Examples

#### Get Access Token
```bash
curl -X POST "https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id={APP_ID}" \
  -d "scope=https://graph.microsoft.com/.default" \
  -d "client_secret={CLIENT_SECRET}" \
  -d "grant_type=client_credentials"
```

#### List Files
```bash
curl -X GET "https://graph.microsoft.com/v1.0/me/drive/root/children" \
  -H "Authorization: Bearer {access_token}" \
  -H "Accept: application/json"
```

#### Get File Content
```bash
curl -X GET "https://graph.microsoft.com/v1.0/me/drive/root:/Documents/example.txt:/content" \
  -H "Authorization: Bearer {access_token}" \
  -o downloaded_file.txt
```

#### Get File Metadata
```bash
curl -X GET "https://graph.microsoft.com/v1.0/me/drive/root:/Documents/example.txt" \
  -H "Authorization: Bearer {access_token}" \
  -H "Accept: application/json"
```

---

## Step 8: Token Renewal

### 8.1 Token Expiration

- Access tokens expire after **3600 seconds (1 hour)**
- Tokens should be refreshed before expiration
- Implement token caching with expiration checking

### 8.2 Renewal Strategy

**Recommended Approach:**
1. Cache the token and expiration time
2. Check token validity before each API call
3. Refresh token if it expires within 5 minutes
4. Use the same token endpoint with same credentials

**Python Implementation:**
```python
def get_access_token():
    """Automatically handles token refresh"""
    if token_is_valid():
        return cached_token
    else:
        return request_new_token()
```

### 8.3 Error Handling

If you receive a `401 Unauthorized` response:
1. Your token has expired
2. Request a new token using the same authentication endpoint
3. Retry the original request with the new token

---

## Step 9: Important Notes

### 9.1 Application vs Delegated Permissions

- **Application permissions** (used here): App-only access, works without user sign-in
- **Delegated permissions**: Requires user sign-in, acts on behalf of the user

For ChatGPT integration, **Application permissions** are required.

### 9.2 OneDrive vs SharePoint Endpoints

- **OneDrive endpoint:** `https://graph.microsoft.com/v1.0/me/drive/...`
- **SharePoint endpoint:** `https://graph.microsoft.com/v1.0/sites/{site-id}/drive/...`

This solution uses OneDrive endpoints as requested.

### 9.3 Security Best Practices

1. **Store secrets securely:**
   - Use environment variables
   - Never commit secrets to version control
   - Use Azure Key Vault for production

2. **Token management:**
   - Cache tokens securely
   - Implement automatic refresh
   - Handle token expiration gracefully

3. **Permissions:**
   - Use least privilege principle
   - Only grant `Files.Read.All` (not write permissions)

---

## Step 10: Testing Checklist

- [ ] App Registration created
- [ ] Application permissions added (`Files.Read.All`)
- [ ] Admin consent granted
- [ ] Client secret created and saved
- [ ] Access token obtained successfully
- [ ] List files endpoint works
- [ ] Get file content endpoint works
- [ ] Get file metadata endpoint works
- [ ] Token refresh mechanism tested

---

## Troubleshooting

### Common Issues

1. **401 Unauthorized**
   - Token expired → Refresh token
   - Invalid credentials → Verify APP_ID, CLIENT_SECRET, TENANT_ID
   - Admin consent not granted → Grant admin consent in Azure Portal

2. **403 Forbidden**
   - Permissions not granted → Grant admin consent
   - Wrong permission type → Use Application permissions, not Delegated

3. **404 Not Found**
   - File path incorrect → Verify file path format
   - File doesn't exist → Check file location in OneDrive

4. **Token endpoint returns error**
   - Verify TENANT_ID, APP_ID, CLIENT_SECRET are correct
   - Check that client secret hasn't expired
   - Ensure grant_type is `client_credentials`

---

## Next Steps

1. Complete Azure App Registration setup
2. Test authentication with provided examples
3. Integrate with your ChatGPT application
4. Implement error handling and token refresh
5. Deploy to your production environment

---

## Support Resources

- [Microsoft Graph API Documentation](https://docs.microsoft.com/en-us/graph/overview)
- [OneDrive API Reference](https://docs.microsoft.com/en-us/graph/api/resources/onedrive)
- [Azure AD Authentication](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow)

---

**Document Version:** 1.0  
**Last Updated:** 2024

