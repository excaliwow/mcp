# Install the Excaliwow MCP server

Excaliwow lets an MCP-capable coding agent create, read, render, organize, and safely edit persistent Excalidraw diagrams that people can refine in the browser.

## Requirements

- Node.js 22.11.0 or newer
- An Excaliwow account
- A Personal Access Token from <https://excaliwow.com/app/settings> with `read` and `write` capabilities

Add `delete` only if the user explicitly wants the reversible trash and restore tools. Do not request or store a broader token.

## MCP configuration

Ask the user for their token through the client's normal secret-input flow. Never write a real token into a repository or include it in generated instructions.

Configure the server as:

```json
{
  "mcpServers": {
    "excaliwow": {
      "command": "npx",
      "args": ["-y", "@excaliwow/mcp"],
      "env": {
        "EXCALIWOW_TOKEN": "<user-provided-secret>"
      }
    }
  }
}
```

`EXCALIWOW_API_URL` is optional and should normally remain unset so the server uses `https://excaliwow.com`.

## Verify

Confirm the published package and runtime before calling tools:

```sh
npx -y @excaliwow/mcp --version
npx -y @excaliwow/mcp --health
```

The version command should report `0.14.2`. The health command requires the token in `EXCALIWOW_TOKEN` and should report `ok`.

Full documentation: <https://excaliwow.com/docs/mcp>
