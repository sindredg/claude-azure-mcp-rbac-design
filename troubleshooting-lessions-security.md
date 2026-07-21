# Troubleshooting, lessons learned and security notes

How I got the Azure MCP Server in Claude Desktop to authenticate as an
Entra ID app registration (service principal) instead of an interactive
Azure user login. Written down because the failure mode was not obvious
and cost some time.

## The problem

The Azure MCP Server extension kept falling back to interactive browser
login, even though `AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and
`AZURE_CLIENT_SECRET` were set correctly as Windows user-level environment
variables. The credentials themselves were fine; signing in with them
through `az login --service-principal` worked.

## Root cause

Not verified to the bottom, but the evidence points one way: the
extension-managed server process never saw the variables. They were
provably set correctly (a fresh PowerShell saw them, and az login with the
same values succeeded), yet the server reported no environment credentials
found. The extension log also shows Claude Desktop constructing its own
PATH for the server process, which suggests it builds a controlled
environment rather than passing the user's through.

The effect is what matters: `EnvironmentCredential`, first in the server's
credential chain, finds nothing, and the chain falls through credential by
credential until it ends at interactive browser login. The error message
lists every failed step, but it is easy to misread the symptom as a
credential problem when the actual problem is that the process never sees
them.

## The fix

The extension is just packaging around one program, `azmcp.exe`. Instead
of letting the extension launch it, add your own entry in
`claude_desktop_config.json` that points at the same exe, with the
credentials in an explicit `env` block. Same program, different launch
path, and this one passes the credentials through.

No file hunting needed: Claude Desktop > Settings > Developer > Edit
Config opens `claude_desktop_config.json` directly. It took effect
immediately after saving:

```json
{
  "mcpServers": {
    "azure-sp": {
      "command": "C:\\Users\\<user>\\AppData\\Roaming\\Claude\\Claude Extensions\\local.mcpb.microsoft.azure.mcp.server\\server\\azmcp.exe",
      "args": [
        "server",
        "start",
        "--read-only"
      ],
      "env": {
        "AZURE_TENANT_ID": "<tenant-id>",
        "AZURE_CLIENT_ID": "<app-client-id>",
        "AZURE_CLIENT_SECRET": "<secret-value>"
      }
    }
  }
}
```

After saving, the connector overview in the chat shows `azure-sp` as its
own connector alongside the extension's Azure MCP Server entry. They are
two separate server processes; disable the extension's toggle so only
`azure-sp` runs and every call goes through the service principal.

Verified working: listing subscriptions through Claude returns results via
the service principal, with no user login involved.

## Lessons learned

- `env` is defined per server entry. There is no global env section in
  `claude_desktop_config.json`.
- Claude Desktop does not expand `${VAR}` placeholders in the config (that
  is a Claude Code feature). A literal `${AZURE_TENANT_ID}` is passed to
  the process as-is and breaks auth. Worse, it overrides any correct value
  the process might otherwise have seen.
- The secret must be the client secret value, not the secret ID. Easy to
  copy the wrong one from the portal.
- Optional but useful while debugging: adding
  `"AZURE_TOKEN_CREDENTIALS": "EnvironmentCredential"` to the env block
  pins the chain to the service principal only. Auth failures then produce
  a clear error instead of a silent fallback to browser login, which is
  the confusing behavior this whole document started with.
- When auth fails, test the credentials outside Claude first:
  `az login --service-principal -u <client-id> -p "<secret>" --tenant <tenant-id>`.
  If that fails, the problem is the app registration, not the MCP setup.
  Run `az logout` afterwards so no CLI credential stays cached.

## Security notes

The client secret sits in plain text in `claude_desktop_config.json`,
readable by anything running as the signed-in user. Acceptable for a lab
identity with a narrow Reader scope, but not something to be proud of.
Better options:

1. **Certificate auth:** switch the app registration to a certificate and
   set `AZURE_CLIENT_CERTIFICATE_PATH`. No secret string exists anywhere.
2. **Wrapper script:** point `command` at a script that reads the secret
   from Windows Credential Manager into the environment and then launches
   `azmcp.exe`.
3. **Least privilege:** grant the app registration only the narrowest RBAC
   role and scope it needs, for example Reader on one resource group
   rather than Contributor on the subscription.
4. **Rotation:** short secret expiry, and rotate immediately if the config
   file is ever exposed.

See the [README](README.md) for the full least-privilege design this setup
is part of.