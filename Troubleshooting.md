# Troubleshooting and lessons learned

Getting the Azure MCP Server to authenticate as a service principal instead of an interactive Azure
user login. Terminology follows the [README](README.md).

## Symptom

The Azure MCP Server extension kept falling back to interactive browser login, even though
`AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET` were set correctly as Windows
user-level environment variables. The credentials themselves were fine: `az login --service-principal`
with the same values worked.

## Cause

Not traced all the way down, but the evidence points one way: the extension-managed server process
never saw the variables. They were provably set (a fresh PowerShell saw them, and `az login` with the
same values succeeded), yet the server reported no environment credentials found. The extension log
also shows the host constructing its own PATH for the server process, which suggests it builds a
controlled environment rather than passing the user's through.

The effect: `EnvironmentCredential`, first in the server's credential chain, finds nothing, and the
chain falls through credential by credential until it ends at interactive browser login. The error
lists every failed step, which reads like a credential problem when the actual problem is that the
process never sees them.

## Fix

The extension is packaging around one program, `azmcp.exe`. Instead of letting the extension launch
it, add a manual MCP server entry in `claude_desktop_config.json` pointing at the same exe, with the
credentials in an explicit `env` block. Same program, different launch path, and this one passes the
credentials through to the child process.

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

After saving, the connector overview shows `azure-sp` as its own connector alongside the extension's
Azure MCP Server entry. They are two separate MCP server processes, so disable the extension's toggle
and every call goes through the service principal.

**Verified working:** a subscription listing requested through the model comes back via the service
principal, with no user login involved.

## Lessons learned

- `env` is defined per server entry. There appears to be no global env section in
  `claude_desktop_config.json`.
- Claude Desktop does not expand `${VAR}` placeholders in the config, only Claude Code does. Using
  `${AZURE_TENANT_ID}` breaks auth.
- If auth fails, test the credentials outside the MCP host first:
  `az login --service-principal -u <client-id> -p "<secret>" --tenant <tenant-id>`. If that fails the
  problem is the app registration, not the MCP setup. Run `az logout` afterwards so no CLI credential
  stays cached.

The client secret sits in plain text in `claude_desktop_config.json`. Better options, and the
data-security posture this setup carries, are in [SecurityNotes.md](SecurityNotes.md). The
least-privilege design it is part of is in the [README](README.md).
