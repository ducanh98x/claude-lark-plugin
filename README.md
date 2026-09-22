# Lark / Feishu integration for Claude

Connects Claude to LarkSuite — messaging, docs, Bitable/Base, calendar, and tasks — by wrapping the **official** `@larksuiteoapi/lark-mcp` MCP server ([larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp)). No fork, no reimplementation — `server/node_modules/@larksuiteoapi/lark-mcp` is the vendored upstream package, unmodified, shared by both install paths below.

Tested against `@larksuiteoapi/lark-mcp@0.5.1` (latest on npm at time of writing — re-check before bumping, see "Updating the vendored version" below).

This repo ships **two independent ways to install it** — pick one:

| | Install path | Where it works | Credential entry |
|---|---|---|---|
| A | [`.mcpb` Desktop Extension](#a-install-as-a-desktop-extension-mcpb) | Claude Desktop (Chat tab *and* Cowork/Code tab — same app, shared extension) | Native form in Claude Desktop, stored in OS keychain |
| B | [Plugin marketplace](#b-install-as-a-claude-codecowork-plugin) | Claude Code / Cowork specifically (CLI or any host with the plugin system) | Environment variables you set yourself |

**Don't install both at once** — they'd register two separate `lark` MCP servers and collide. (A) already works inside Cowork with zero extra setup, so it's the simpler default; use (B) only if you specifically want the marketplace-based plugin workflow (e.g. `claude plugin install`, sharing via a marketplace listing) instead of the Desktop Extension UI.

## What this does

- Domain: `https://open.larksuite.com` (Lark international, not Feishu China).
- Tools enabled: `preset.default,preset.calendar.default,preset.task.default` — covers IM/messaging, Docs, Bitable/Base, Contacts, Calendar, and Tasks. (Note: `preset.default` alone does **not** include Calendar/Task — both are appended explicitly.)
- Auth: tenant-level App ID + App Secret (not user OAuth). Your Lark custom app must have permission scopes enabled for each capability group you want to use (Messaging, Docs, Bitable, Calendar, Task) on the [Lark Open Platform console](https://open.larksuite.com/).

## A. Install as a Desktop Extension (MCPB)

1. Build the bundle (see below) to get `lark.mcpb`, or download a prebuilt release.
2. Drag `lark.mcpb` onto Claude Desktop, or Settings → Extensions → Install from file.
3. When prompted, enter your Lark **App ID** and **App Secret**. These are stored encrypted by Claude Desktop (OS keychain/credential manager) — they are never written to this repo, never appear in chat, and never pass through any script here.
4. Confirm the "Lark (Feishu)" MCP server shows as connected. Its tools are available in both the Chat tab and the Cowork/Code tab automatically.

## B. Install as a Claude Code/Cowork plugin

Uses the standard plugin marketplace flow instead of dragging a file:

```bash
claude plugin marketplace add ducanh98x/claude-lark-plugin
claude plugin install lark@claude-lark-plugin
```

Then set credentials as environment variables **before launching Claude** (the plugin's `.mcp.json` reads `LARK_APP_ID`/`LARK_APP_SECRET` — there's no credential-entry UI in the plugin system, unlike the MCPB path):

```bash
export LARK_APP_ID="cli_xxxxxxxxxxxx"
export LARK_APP_SECRET="your_app_secret"
```

Add those two lines to your shell profile (`~/.zshrc`/`~/.zprofile`) for them to persist. **Caveat:** if you launch Claude Desktop from the Dock/Spotlight (not from a terminal), it may not inherit shell-profile exports on macOS — on macOS you can instead set them process-wide with `launchctl setenv LARK_APP_ID ...` / `launchctl setenv LARK_APP_SECRET ...` before starting the app. This path only registers the MCP server for Claude Code/Cowork, not for the plain Claude Desktop chat UI — use path (A) for that.

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

## Repo layout

```
claude-lark-plugin/
├── manifest.json              # MCPB manifest (path A)
├── icon.png
├── .claude-plugin/
│   ├── marketplace.json       # marketplace catalog (path B) — the file `claude plugin marketplace add` looks for
│   └── plugin.json            # plugin manifest (path B)
├── .mcp.json                  # MCP server config for path B, reads LARK_APP_ID/LARK_APP_SECRET env vars
└── server/node_modules/       # vendored @larksuiteoapi/lark-mcp, shared by both A and B
```

Both `manifest.json` (MCPB) and `.mcp.json` (plugin) point at the same `server/node_modules/@larksuiteoapi/lark-mcp/dist/cli.js` — one vendored copy, two ways to launch it.
