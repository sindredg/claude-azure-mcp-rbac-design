# Troubleshooting and lessons learned

How I got the Azure MCP Server to authenticate as a service principal (Entra ID app registration) instead of an interactive Azure user login.
Written down because the failure was not obvious and cost some time.

Terminology follows the [README](README.md): the model is Claude running remotely, the MCP host is Claude Desktop, the MCP server is `azmcp.exe` running locally as a child process of the host.

## The problem

The Azure MCP Server extension kept falling back to interactive browser login, even though `AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET` were set correctly as Windows user-level environment variables. The credentials themselves were fine; signing in with them through `az login --service-principal` worked.

## Root cause

Not traced all the way down, but the evidence points one way: the extension-managed server process never saw the variables. They were
provably set correctly (a fresh PowerShell saw them, and az login with the same values succeeded), yet the server reported no environment credentials
found. The extension log also shows the host constructing its own PATH for the server process, which suggests it builds a controlled environment rather than passing the user's through.

The effect: `EnvironmentCredential`, first in the server's credential chain, finds nothing, and the chain falls through credential by credential until it ends at interactive browser login. The error message lists every failed step, but it is easy to misread the symptom as a credential problem when the actual problem was that the process never sees them.

## The fix

The extension is just packaging around one program, `azmcp.exe`. Instead of letting the extension launch it, add your own MCP server entry in `claude_desktop_config.json` that points at the same exe, with the credentials in an explicit `env` block. Same program, different launch path, and this one passes the credentials through to the child process.

Claude Desktop > Settings > Developer > Edit config:
Choose `claude_desktop_config.json` and add the below:

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

After saving, the connector overview in the chat shows `azure-sp` as its own connector alongside the extension's Azure MCP Server entry. They are two separate MCP server processes; disable the extension's toggle so only `azure-sp` runs and every call goes through the service principal.

Verified working: a subscription listing requested through the model comes back via the service principal, with no user login involved.

## Lessons learned

- `env` is defined per server entry. There seem to be no global env section in `claude_desktop_config.json`.
- Claude Desktop does not expand `${VAR}` placeholders in the config (only Claude Code). 
  Using `${AZURE_TENANT_ID}` breaks auth.
- If auth fails, test the credentials outside the MCP host first: `az login --service-principal -u <client-id> -p "<secret>" --tenant <tenant-id>`.
  If that fails, the problem is the app registration, not the MCP setup.
  Run `az logout` afterwards so no CLI credential stays cached.

## Security notes

The client secret sits in plain text in `claude_desktop_config.json`.

Better options:

1. **Certificate auth:** switch the app registration to a certificate and set `AZURE_CLIENT_CERTIFICATE_PATH`. No secret string exists anywhere.
2. **Wrapper script:** point `command` at a script that reads the secret from Windows Credential Manager into the environment and then launches `azmcp.exe`.
3. **Rotation:** short secret expiry, and rotate immediately if the config file is ever exposed.
4. **Least privilege:** does not protect the secret, but caps what a leaked one is worth. Reader on one resource group rather than Contributor on the subscription.

See the [README](README.md) for the full least-privilege design this setup
is part of.
