# @excaliwow/mcp

The Excaliwow **Model Context Protocol** server. It lets an AI agent create,
read, render, manage, and edit diagrams in your [Excaliwow](https://excaliwow.com)
account through the public REST API (`/api/v1`), the same one the CLI uses. It
runs over **stdio**, so any MCP client (Claude Code, Claude Desktop, and others)
can launch it with `npx`.

[![npm](https://img.shields.io/npm/v/@excaliwow/mcp)](https://www.npmjs.com/package/@excaliwow/mcp)
[![license](https://img.shields.io/npm/l/@excaliwow/mcp)](./LICENSE)

## Tools

Seven tools, scoped to what your Personal Access Token allows:

| Tool | What it does | Capability |
|------|--------------|------------|
| `generate_diagram` | Create a new diagram from the Excaliwow DSL | write |
| `read_diagram` | Read a diagram's content and metadata | read |
| `list_diagrams` | List diagrams in your account | read |
| `move_diagram` | Move a diagram into or out of a folder | write |
| `edit_diagram` | Edit a diagram with an additive merge | write |
| `trash_diagram` | Soft-delete a diagram | delete |
| `restore_diagram` | Restore a diagram from trash | delete |

`read` + `write` cover five of the seven. Add `delete` only if you want the agent
to trash and restore diagrams.

## Setup

Mint a Personal Access Token at [excaliwow.com](https://excaliwow.com)
(Settings → Developer / API tokens) with the capabilities you want, then pass it
as `EXCALIWOW_TOKEN`.

### Claude Code

```sh
claude mcp add excaliwow --scope local \
  --env EXCALIWOW_TOKEN=excw_pat_… \
  -- npx -y @excaliwow/mcp
```

`--scope local` keeps the token in your own settings, out of any file you might
commit.

### Claude Desktop and other JSON-config clients

Pin the version. A process that holds a token to your account should not
silently auto-update.

```json
{
  "mcpServers": {
    "excaliwow": {
      "command": "npx",
      "args": ["-y", "@excaliwow/mcp@0.3.0"],
      "env": { "EXCALIWOW_TOKEN": "excw_pat_…" }
    }
  }
}
```

The server reads `EXCALIWOW_TOKEN` per call from the environment (or, if you also
use [`@excaliwow/cli`](https://github.com/excaliwow/cli) and have run
`excaliwow auth login`, that stored login). It never writes the token to disk
itself.

## Troubleshooting

**`npx -y @excaliwow/mcp@0.3.0` fails with `ENOENT … package.json`.** On some
npm/Node versions, `npx` misreads a scoped package plus `@version` as a local
directory. It is an upstream npm bug, not an Excaliwow one. Either drop the
version (`npx -y @excaliwow/mcp`) or install once and point the client at the
binary:

```sh
npm i -g @excaliwow/mcp@0.3.0
```

```json
{
  "mcpServers": {
    "excaliwow": {
      "command": "excaliwow-mcp",
      "env": { "EXCALIWOW_TOKEN": "excw_pat_…" }
    }
  }
}
```

## Links

- Website: [excaliwow.com](https://excaliwow.com)
- Questions and issues: [excaliwow/community](https://github.com/excaliwow/community)
- Discord: [discord.gg/z2fFgThAu6](https://discord.gg/z2fFgThAu6)

## License

Apache-2.0. See [LICENSE](./LICENSE) and [NOTICE](./NOTICE).
