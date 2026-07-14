# @excaliwow/mcp

The Excaliwow **Model Context Protocol** server — lets an AI agent create, read,
render, manage, and edit diagrams in your Excaliwow account through the same
public REST API (`/api/v1`) the CLI uses. It runs over **stdio**, so any MCP
client (Claude Desktop, Claude Code, etc.) can launch it with `npx`.

The public [`excaliwow/mcp`](https://github.com/excaliwow/mcp) repository contains
distribution metadata and documentation for this package; the hosted application
source is not published there.

## Install

First mint a Personal Access Token at https://excaliwow.com/app/settings
(Settings → Developer / API tokens) with **`read` + `write`** capabilities —
enough for sixteen of the eighteen tools. Add **`delete`** only if you want the agent
to trash and restore diagrams (see [Security notes](#security-notes)). Pass it as
`EXCALIWOW_TOKEN`.

### Claude Code (CLI)

One command. `--scope local` stores the server in your own settings, so the
token never lands in a file you might commit. Export the token first so the
literal PAT never lands in your shell history — `--env NAME="$NAME"` passes
the value through without typing it a second time on the command line:

```sh
export EXCALIWOW_TOKEN=excw_pat_…
claude mcp add excaliwow --scope local \
  --env EXCALIWOW_TOKEN="$EXCALIWOW_TOKEN" \
  -- npx -y @excaliwow/mcp
```

### Claude Desktop (and other JSON-config clients)

Add it to your client's config file (e.g. Claude Desktop's
`claude_desktop_config.json`, or your client's MCP settings — see your client's
MCP setup docs). The spec is unversioned, so `npx -y` always launches the latest
release — you pick up new tools and fixes automatically, with nothing to bump:

```json
{
  "mcpServers": {
    "excaliwow": {
      "command": "npx",
      "args": ["-y", "@excaliwow/mcp"],
      "env": {
        "EXCALIWOW_TOKEN": "excw_pat_…"
      }
    }
  }
}
```

The server reads `EXCALIWOW_TOKEN` per call from the environment (or, if you also
use `@excaliwow/cli` and have run `excaliwow auth login`, that stored login) and
never writes the token to disk itself.

## Troubleshooting

**"Not authenticated" / 401 / the agent's tool calls fail.** The server starts
even without a token (so it can list its tools), so a missing or invalid
`EXCALIWOW_TOKEN` only surfaces when the agent first calls a tool. Starting with
no token prints a one-line `EXCALIWOW_TOKEN is not set` warning to **stderr**. To
check a token directly, run the health probe — it makes one authenticated read
and prints a clear verdict (`ok`, `401 — token is invalid or expired`, or
`could not reach <url>` when the API itself is unreachable) and exits with a
matching code (0 ok, 1 missing/invalid token, 2 unreachable):

```sh
npx -y @excaliwow/mcp --health
```

### CLI flags

| Flag        | Effect                                                                                        |
| ----------- | --------------------------------------------------------------------------------------------- |
| `--health`  | Check the token + API reachability, then exit (0 ok, 1 missing/invalid token, 2 unreachable). |
| `--version` | Print the installed version and exit.                                                         |
| `--help`    | Print usage (flags + env vars) and exit.                                                      |

## Tools

Eighteen tools, scoped to safe agent use:

| Tool                     | Capability | What it does                                                                                                           |
| ------------------------ | ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| `generate_diagram`       | `write`    | Create a diagram from the high-level node/edge DSL (auto-laid-out); returns the editor URL. Best for quick flowcharts. |
| `create_scene`           | `write`    | Create a **rich, hand-authored** diagram from a raw Excalidraw scene — full control of layout, style, and typography.  |
| `read_diagram`           | `read`     | Compact summary (title + per-type counts) **plus** a rendered PNG; opt into `includeGeometry` for a bounds list.       |
| `get_scene`              | `read`     | Return a diagram's full raw scene (`{ elements, appState }`) for editing — read → mutate → `regenerate_diagram`.       |
| `export_diagram`         | `read`     | Render to full-fidelity bytes to **save** (png base64, or svg as raw text) — the bytes to keep, not a vision image.    |
| `list_diagrams`          | `read`     | Page through your diagrams (`filter: active \| trash`).                                                                |
| `move_diagram`           | `write`    | Move a diagram to a folder (or to root).                                                                               |
| `edit_diagram`           | `write`    | Additively merge a DSL fragment (add nodes/edges, update node style/label).                                            |
| `regenerate_diagram`     | `write`    | Replace a diagram's contents in place — from a fresh `spec` (re-layout) **or** a raw `scene`, same id.                 |
| `trash_diagram`          | `delete`   | Soft-delete a diagram to trash. **Reversible** (see `restore_diagram`).                                                |
| `restore_diagram`        | `delete`   | Restore a trashed diagram, reopening it at its original id and URL.                                                    |
| `lint_diagram`           | `read`     | Check a diagram for overlaps, clipped labels, out-of-frame elements, and low-contrast text.                            |
| `patch_elements`         | `write`    | Apply element-addressable deltas (add/update/move/resize/restyle/delete) in place — the cheapest edit path.            |
| `generate_from_template` | `write`    | Create a diagram from a named layout template (`swimlane`, `layered-stack`, `matrix`, `container-with-children`).      |
| `list_icons`             | `read`     | List the curated icons you can embed via an `image` element + `icon-<name>` fileId (id + label + category).            |
| `list_folders`           | `read`     | List the caller's folders (id + name + parentId) to resolve or discover a `folderId`.                                  |
| `create_folder`          | `write`    | Create a folder (optionally nested under a `parentId`) to organize diagrams.                                           |
| `rename_diagram`         | `write`    | Change only a diagram's title, leaving its contents and id/url untouched.                                              |

`read_diagram` returns a summary + image, **never** the raw scene JSON, to keep
context small. Pass `includeGeometry: true` to additionally get a compact,
bounded `{ id, type, label, x, y, w, h }` list (top-left x/y) so the agent can
**detect** label/box collisions or misplaced nodes programmatically instead of
eyeballing the PNG — it is derived from the scene, so it is present even when the
render fails, and it is a small fixed-field summary, not the raw element dump.
Pass `includeScene: 'compact'` instead for a bounded, element-addressable
projection — a superset of the geometry fields that also carries style, text,
and container/binding refs (≤17 fields/element) — the read half of the
`read_diagram` → `patch_elements` surgical-edit loop.
`export_diagram` returns the rendered **bytes** to save to a file — png as
base64, svg as raw text — distinct from `read_diagram`, which returns an image
block for a vision model to look at. An MCP server runs over stdio and cannot
write to your repo, so a client with filesystem access (e.g. Claude Code) decodes
and saves the bytes itself. Or skip the round-trip through the model and stream
straight to disk with the CLI: `excaliwow diagrams render <id> -o
docs/architecture.png` (or `.svg`) — also the fallback when a render is too large
to return inline.

`trash_diagram` / `restore_diagram` are a **reversible** pair
gated on the `delete` capability — registered always, they return a clean
`insufficient_scope` error (changing nothing) unless the token carries `delete`,
so a `read` + `write` token can't trash anything. Hard-delete/purge and making a
diagram publicly shareable are deliberately **not** agent tools — those are
irreversible, so a misled agent can't destroy or expose your diagram. Purge or
publish from the dashboard or the CLI.

### Two authoring paths

`generate_diagram` takes the high-level node/edge **DSL** and auto-lays it out —
reach for it when you just want a quick flowchart. `create_scene` takes a **raw,
hand-authored Excalidraw scene**, so the agent controls every element's position,
size, color, stroke, fill, typography, and connections — the path for rich,
polished, custom diagrams. You author elements _tersely_ (id, type, geometry,
text, colors) and the Excalidraw boilerplate is filled in for you; a
fully-specified element (or a pasted `.excalidraw` scene) is passed through
unchanged. To iterate on a rich diagram: `get_scene` → edit the elements →
`regenerate_diagram` with the edited `scene`.

### Discovery resources

The DSL grammar + a worked example are embedded in the `generate_diagram` tool
description, with the full reference served as an MCP resource at
**`excaliwow://dsl/reference`**. The rich-scene authoring primer + a worked
example ride in the `create_scene` description, with the full guide (every
element type, all styling props, bindings, groups, frames, palette, and
layout/beauty heuristics) at **`excaliwow://scene/authoring`**.

## Environment variables

| Var                 | Effect                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `EXCALIWOW_TOKEN`   | **Required** (standalone). Bearer PAT; read per call, never written to disk by this server. |
| `EXCALIWOW_API_URL` | API origin. Default `https://excaliwow.com`; normally leave unset.                          |

## Requirements

Node **>= 22.11.0**. This is a support-policy floor, not a technical one —
nothing shipped here uses a Node-22-only API (the code's newest runtime
dependency is global `fetch`, available since Node 18); the floor tracks
"current LTS" so `npx -y @excaliwow/mcp` runs on a Node build we actually test
against, and matches `@excaliwow/cli`'s floor. If your MCP client is stuck on
an older Node and `npx` refuses to run this server, that refusal is
intentional — upgrade Node rather than treat it as a bug.

## Security notes

- **Keep the token out of anything you commit.** A project-scoped config that
  lives in the repo (a committed `.mcp.json`, or `claude mcp add --scope
project`) puts the token into git history. Use a user- or local-scoped config
  (`--scope local`), or reference an environment variable instead of pasting the
  literal token.
- **Typing the literal token on a command line puts it in your shell history**
  (`~/.zsh_history`, `~/.bash_history`, …) and briefly in the process list.
  `export EXCALIWOW_TOKEN=…` once, then pass it through with `--env
EXCALIWOW_TOKEN="$EXCALIWOW_TOKEN"` (or pull it from a secrets manager) instead
  of pasting the PAT into the `claude mcp add` command itself.
- **Mint with `read` + `write` (add `delete` only if you want trash/restore).**
  Sixteen of the eighteen tools need just `read` + `write`; a `read` + `write` PAT can
  neither expose your diagrams publicly nor delete them, even if the agent is
  misled (`trash_diagram` / `restore_diagram` simply return `insufficient_scope`
  and do nothing). Add the `delete` capability only if you want the agent to be
  able to soft-delete and restore — it's reversible, but it's still a capability
  to grant deliberately. Pick capabilities specifically rather than the coarse
  `read-write` preset, which additionally grants `publish` (and `delete`).
- The token is a credential to your account. It lives only in your MCP client
  config / environment; this server does not persist it. Treat that config the
  way you'd treat any secrets file.

## Support

Found a bug or have a question? See [excaliwow.com/docs/mcp](https://excaliwow.com/docs/mcp)
or reach us at [excaliwow.com/contact](https://excaliwow.com/contact).

## License

Apache-2.0 — see [LICENSE](./LICENSE) and [NOTICE](./NOTICE).
