# Azure OIDC (Microsoft Entra ID) Setup Guide

Use this guide when you want CollabMD users to sign in with their Microsoft (Azure AD / Entra ID) account and have their verified `name` and `email` become the app identity and default git commit author.

## What you need

- A stable public URL for CollabMD, such as `https://notes.example.com`
- An Azure AD (Microsoft Entra ID) tenant you can manage
- A deployment path where `PUBLIC_BASE_URL` always matches the browser-visible app origin

Azure OIDC is not compatible with ephemeral Cloudflare Quick Tunnel URLs because Azure OAuth clients require fixed redirect URIs.

## 1. Decide the public URL first

Pick the exact URL where users will open CollabMD.

Examples:

- Root path: `https://notes.example.com`
- Subpath deployment: `https://docs.example.com/collabmd`

If you deploy under a subpath, CollabMD also needs:

```bash
BASE_PATH=/collabmd
```

## 2. Open Microsoft Entra ID Admin Center

1. Go to [Microsoft Entra ID Admin Center](https://entra.microsoft.com/).
2. Select your tenant from the tenant dropdown if you have access to multiple.

## 3. Register an application

1. In the left navigation, go to **Identity** → **Applications** → **App registrations**.
2. Click **New registration**.
3. Give it a clear name such as `CollabMD Production`.
4. Under **Supported account types**, choose one:
   - **Single-tenant** — only users in your organization can sign in
   - **Multitenant and personal Microsoft accounts** — users from any Azure AD tenant or personal Microsoft accounts can sign in
5. Click **Register**.

## 4. Register the redirect URI

1. In your app registration, go to **Authentication**.
2. Under **Redirect URIs**, click **Add a redirect URI**.
3. Select **Web** as the platform.
4. Enter the redirect URI matching your CollabMD deployment:

Root deployment:

```text
https://notes.example.com/api/auth/oidc/callback
```

Subpath deployment with `BASE_PATH=/collabmd`:

```text
https://docs.example.com/collabmd/api/auth/oidc/callback
```

5. Click **Configure** to save.

## 5. Copy the Application (client) ID and Directory (tenant) ID

After creating the app registration, note the following values from the **Overview** page:

- **Application (client) ID** — use for `AUTH_OIDC_CLIENT_ID`
- **Directory (tenant) ID** — use for `AUTH_OIDC_TENANT_ID`

## 6. Create a client secret

1. In your app registration, go to **Certificates & secrets**.
2. Under **Client secrets**, click **New client secret**.
3. Give it a description such as `CollabMD Production`.
4. Choose an expiration period (e.g., 12 months).
5. Click **Add**.
6. **Immediately copy the secret value** — it will not be shown again. Use this for `AUTH_OIDC_CLIENT_SECRET`.

## 7. Configure API permissions

1. In your app registration, go to **API permissions**.
2. Click **Add a permission** → **Microsoft Graph** → **Delegated permissions**.
3. Search for and add the following permissions:
   - `openid`
   - `profile`
   - `email`
4. Click **Add permissions**.

For single-tenant apps, you do not need to grant admin consent. For multitenant apps, click **Grant admin consent** if required.

## 8. Configure CollabMD

Set these environment variables:

```bash
AUTH_STRATEGY=oidc
AUTH_OIDC_PROVIDER=azure
PUBLIC_BASE_URL=https://notes.example.com
AUTH_OIDC_CLIENT_ID=your-azure-client-id
AUTH_OIDC_CLIENT_SECRET=your-azure-client-secret
AUTH_OIDC_TENANT_ID=your-azure-tenant-id
AUTH_SESSION_MAX_AGE_MS=2592000000
```

If the app is mounted under a subpath:

```bash
BASE_PATH=/collabmd
PUBLIC_BASE_URL=https://docs.example.com
```

Example CLI start:

```bash
PUBLIC_BASE_URL=https://notes.example.com \
AUTH_OIDC_PROVIDER=azure \
AUTH_OIDC_CLIENT_ID=your-azure-client-id \
AUTH_OIDC_CLIENT_SECRET=your-azure-client-secret \
AUTH_OIDC_TENANT_ID=your-azure-tenant-id \
collabmd /path/to/vault --auth oidc --no-tunnel
```

## 9. Optional: Multi-tenant deployment

If you want users from any Azure AD tenant to sign in (not just your organization), use `common` or `organizations` as the tenant ID:

```bash
AUTH_OIDC_TENANT_ID=common
# or
AUTH_OIDC_TENANT_ID=organizations
```

This maps to the issuer URL `https://login.microsoftonline.com/common/v2.0` or `https://login.microsoftonline.com/organizations/v2.0`.

## 10. Optional access restrictions

CollabMD can restrict which accounts may sign in.

Allow exact email addresses:

```bash
AUTH_OIDC_ALLOWED_EMAILS=ceo@example.com,cto@example.com
```

Allow whole email domains:

```bash
AUTH_OIDC_ALLOWED_DOMAINS=example.com,subsidiary.com
```

Behavior:

- If neither is set, any Microsoft account with a verified email can sign in
- If `AUTH_OIDC_ALLOWED_EMAILS` is set, those exact addresses are allowed
- If `AUTH_OIDC_ALLOWED_DOMAINS` is set, any verified email in those domains is allowed
- If both are set, an exact allowed email or an allowed domain is enough to grant access

## 11. Verify startup output

When CollabMD starts successfully with Azure OIDC, startup should show the public URL and callback URL.

Example:

```text
Auth:   oidc (Microsoft)
Public: https://notes.example.com
Callback: https://notes.example.com/api/auth/oidc/callback
Tunnel: disabled (OIDC requires a stable PUBLIC_BASE_URL)
```

## 12. Test the sign-in flow

1. Open the CollabMD URL in a browser.
2. Click `Continue with Microsoft`.
3. Complete the Microsoft sign-in flow.
4. Confirm the app opens normally.
5. Confirm the toolbar shows your Microsoft name.
6. If using git from the UI, make a commit and verify the author:

```bash
git -C /path/to/vault log -1 --pretty='%an <%ae>'
```

## Troubleshooting

- `OIDC auth requires PUBLIC_BASE_URL`: set `PUBLIC_BASE_URL` to the browser-visible origin
- `OIDC auth with Azure requires AUTH_OIDC_TENANT_ID`: set `AUTH_OIDC_TENANT_ID` to your Azure tenant ID
- Redirect URI mismatch in Azure: confirm the registered redirect URI exactly matches `/api/auth/oidc/callback`, including any `BASE_PATH`
- Login keeps returning to the auth screen: verify the reverse proxy preserves HTTPS headers and the browser URL matches `PUBLIC_BASE_URL`
- Session expires too quickly: set `AUTH_SESSION_MAX_AGE_MS` to a longer value such as `2592000000` for 30 days
- Expected company accounts cannot sign in: check `AUTH_OIDC_ALLOWED_EMAILS` and `AUTH_OIDC_ALLOWED_DOMAINS` for typos, whitespace, or missing domains
- Tunnel is disabled unexpectedly: this is intentional for OIDC; use a stable public host instead of Quick Tunnel
- `Account is missing a verified email or name`: ensure the user's Microsoft account has a verified email and display name set

## Custom issuer URL

If you need a custom issuer URL (e.g., for sovereign clouds), override the default:

```bash
AUTH_OIDC_ISSUER_URL=https://login.microsoftonline.de/your-tenant-id/v2.0
```

This bypasses the automatic issuer URL construction from `AUTH_OIDC_TENANT_ID`.
