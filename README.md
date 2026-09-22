# Lark / Feishu extension for Claude Desktop

Connects Claude to LarkSuite — messaging, docs, Bitable/Base, calendar, and tasks — by bundling the **official** `@larksuiteoapi/lark-mcp` MCP server ([larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp)) as an [MCPB](https://github.com/anthropics/mcpb) Desktop Extension. No fork, no reimplementation — `server/node_modules/@larksuiteoapi/lark-mcp` is the vendored upstream package, unmodified.

Tested against `@larksuiteoapi/lark-mcp@0.5.1` (latest on npm at time of writing — re-check before bumping, see "Updating the vendored version" below).

## What this does

- Domain: `https://open.larksuite.com` (Lark international, not Feishu China).
- Tools enabled: `preset.default,preset.calendar.default,preset.task.default` — covers IM/messaging, Docs, Bitable/Base, Contacts, Calendar, and Tasks. (Note: `preset.default` alone does **not** include Calendar/Task — both are appended explicitly.)
- Auth: tenant-level App ID + App Secret (not user OAuth). Your Lark custom app must have permission scopes enabled for each capability group you want to use (Messaging, Docs, Bitable, Calendar, Task) on the [Lark Open Platform console](https://open.larksuite.com/).

## Install

1. Build the bundle (see below) to get `lark.mcpb`, or download a prebuilt release.
2. Drag `lark.mcpb` onto Claude Desktop, or Settings → Extensions → Install from file.
3. When prompted, enter your Lark **App ID** and **App Secret**. These are stored encrypted by Claude Desktop (OS keychain/credential manager) — they are never written to this repo, never appear in chat, and never pass through any script here.
4. Confirm the "lark" MCP server shows as connected.

## Verify the tool list (important)

Unknown/misspelled tool or preset names fail **silently** in `lark-mcp` (no error — the tool just doesn't appear). After installing, ask Claude what Lark tools are available and confirm Calendar and Task tools are present, not just Messaging/Docs/Base. If they're missing, the App's permission scopes on the Lark console are the first thing to check.

## Building from source

```bash
npm install --prefix server @larksuiteoapi/lark-mcp@0.5.1
npx @anthropic-ai/mcpb validate manifest.json
npx @anthropic-ai/mcpb pack
```

Produces `lark.mcpb` in the project root. `server/node_modules` is gitignored — always re-run `npm install --prefix server` after a fresh clone before packing.

### Updating the vendored version

`@larksuiteoapi/lark-mcp` is Beta upstream and gets correctness fixes across point releases. Before bumping the pinned version in this README and in `npm install --prefix server @larksuiteoapi/lark-mcp@<new-version>`:

1. Check `npm view @larksuiteoapi/lark-mcp version` for the current latest.
2. Re-run the "Verify the tool list" smoke test above against the new version before shipping.
3. Update this README's pinned version note.

## Known upstream limitations (as of `0.5.1`)

From `larksuite/lark-openapi-mcp`'s open issues — not something this bundle can work around:

- `docx_builtin_search` rejects `tenant_access_token` even though the underlying API accepts it (#108).
- `im.v1.message.list`/`get` unreachable under user identity (#101).
- `wiki.v1.node.search` points at a stale/deprecated endpoint (#104/#74).
- No dedicated `preset.bitable.default` — Bitable tools live under `preset.base.*` (#77).

File upload/download and direct Lark Docs editing support were not confirmed either way from source/issues — verify empirically against the live tool list if you need them.

## Known fixed bug

`v0.1.0` shipped `display_name: "Lark / Feishu"`. Claude Desktop uses the display name to build an internal log file path, and the `/` character tripped its path-escape safety check (`Could not launch Lark / Feishu: path escape: "Lark / Feishu"` in `~/Library/Logs/Claude/main.log`). It happened to recover via a fallback launch path in testing, but don't rely on that — `v0.1.1` renames it to `Lark (Feishu)` (no `/`). If you installed `v0.1.0`, remove it (Settings → Extensions → Lark → Remove) before installing the new build rather than installing over it.

## Cross-platform note

`compatibility.platforms` declares `darwin`, `win32`, and `linux`, but this bundle has only been built and tested on macOS so far. `@larksuiteoapi/lark-mcp` depends on `keytar` (a native module, used for its own `login`/`user_access_token` OAuth flow — not needed for the App ID/Secret flow this bundle uses). A `.mcpb` packed on macOS bundles keytar's macOS-only prebuilt binary; a genuinely cross-platform release would need a build per target platform (or per-platform CI) before Windows/Linux users can rely on it. Not yet done — macOS is the primary target for now.

## Using this in Cowork / Claude Code

Confirmed working with no extra setup: installing the `.mcpb` once (Chat tab) makes the same "Lark (Feishu)" MCP server's tools available inside the Cowork/Code tab too — same Claude Desktop app, shared extension. No separate Cowork plugin was needed.
