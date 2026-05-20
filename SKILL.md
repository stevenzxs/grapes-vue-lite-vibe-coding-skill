---
name: grapes-vue-lite-vibe-coding-skill
description: Use when controlling an already-open Grapes Vue Lite / craft.js-vue editor through the local bridge. Prefer the MCP tools exposed by grapes-vue-lite-vibe-coding-mcp (editor_list_sessions, editor_get_snapshot, editor_apply_a2ui, editor_create_page, etc.) over raw HTTP calls. Triggered by mentions of localhost/127.0.0.1:3003, bridge port 3456, opencodeBridge, localBridgeToken, or A2UI apply operations.
---

# Grapes Vue Lite Local Bridge

Use this skill when controlling an already-open Grapes Vue Lite page through the local command bridge.

## Trigger Cues

Use this skill for requests that mention any of:

- `Grapes Vue Lite`, `grapes-vue-lite`, or `craft.js-vue`
- `127.0.0.1:3003`, `localhost:3003`, `opencodeBridge`, or `localBridgeToken`
- local bridge, bridge port `3456`, session, command, snapshot, apply, ingest, preview
- `a2ui.ingestApply`, A2UI JSONL, applying a generated design to the current open page
- instructions such as "do not probe unknown endpoints", "use the confirmed bridge flow", or "avoid /token"

Do not use this skill for generic web browsing, unrelated localhost apps, or editing source code unless the task also needs this bridge command flow.

## Available MCP Tools

The `grapes-vue-lite-vibe-coding-mcp` server exposes these tools for bridge interaction:

### Session Discovery
- **`editor_list_sessions`** — List active Grapes Vue Lite bridge sessions with capabilities.

### Snapshot
- **`editor_get_snapshot`** — Get the current editor state (pages, components, tree).

### Page Operations
- **`editor_create_page`** — Create a new page. Args: `name`, `path`, `session_id` (optional).
- **`editor_duplicate_page`** — Duplicate an existing page. Args: `page_id`, `session_id` (optional).

### Asset Operations
- **`editor_browse_asset_library`** — Browse configured image libraries. Args: `source_id`, `page`, `page_size`, `session_id`.
- **`editor_select_local_image`** — Add/select a local image. Args: `src` (or `local_path`), `name`, `alt`, `apply_to_selected`, `session_id`.
- **`editor_delete_asset`** — Delete a project image asset. Args: `asset_id`, `session_id`.

### Block / Component Operations
- **`editor_list_blocks`** — List available editor blocks. Args: `category`, `search`, `include_content`, `session_id`.
- **`editor_list_components`** — List available component types. Args: `search`, `include_traits`, `session_id`.
- **`editor_browse_component_library`** — Browse component-library sources. Args: `source_id`, `page`, `page_size`, `search`, `category`, `include_block`, `session_id`.
- **`editor_update_node_style`** — Update a canvas node's style. Args: `node_id`, `style`, `mode`, `session_id`.

### Apply / Preview
- **`editor_apply_a2ui`** — Apply an A2UI JSONL stream to the editor. Args: `input`, `session_id`.

## Required Workflow

1. Call **`editor_list_sessions`** to get active sessions.
2. Pick a session where `capabilities.snapshot === true` (for reads) and/or `capabilities.apply === true` (for mutations).
3. Pass the chosen `session_id` to the appropriate MCP tool above.
4. Read the tool result directly. Do not query separate result endpoints.

All tools handle bridge token authentication internally via the MCP server configuration.

## Capability Boundary

Only use operations that are explicitly exposed by the currently available MCP tools, this skill, or the bridge session capabilities returned by `/sessions`.

If the user asks for an editor operation that is not explicitly supported, reply that the current bridge/MCP tooling does not support that operation yet. Do not search local source code to discover hidden command names, internal editor APIs, or undocumented behavior.

Examples of unsupported-operation handling:

- If there is no explicit page creation tool or documented bridge command, do not guess `editor.addPage`; say page creation is not currently supported by the bridge/MCP tools.
- If there is no explicit page path update tool or documented bridge command, do not infer one from editor source; say path update is not currently supported.
- If the required operation could be done only by editing source code, browser console injection, or undocumented command types, stop and report the missing capability unless the user explicitly asks to implement that capability.

Allowed discovery:

- Calling `editor_list_sessions` or `GET /sessions` to inspect sanitized runtime session capabilities.
- Calling documented MCP tools such as `editor_get_snapshot` and `editor_apply_a2ui`.
- Reading this skill and user-provided docs.

Disallowed discovery:

- Searching `craft.js-vue` or other local source trees to find command names.
- Probing unknown bridge endpoints or guessed command types.
- Using browser-console/editor internals to bypass the bridge contract.

## Do Not Probe

Do not call these unless the user is explicitly debugging bridge routes:

- `/token`
- `/command`
- `/commands`
- `/results`
- `/api/command`
- `/sessions/{sessionId}`
- guessed result or wait endpoints

Known behavior:

- `404` means the path does not exist.
- `403` means the auth shape is wrong; the command endpoint requires query `token=<token>`.
- `409` means the command id was reused; send a new unique `id`.

These errors do not imply that the frontend page or bridge stopped.

## Applying A2UI

Use the **`editor_apply_a2ui`** MCP tool. The tool takes:
- `input` — the A2UI JSONL string
- `session_id` (optional)

When applying a fixed-size Pencil/Figma prototype, preserve the source top-level frame dimensions:

- Keep `beginRendering.root` as a protocol/editor-safe wrapper such as `root`.
- Do not put fixed prototype dimensions or visual frame styling directly on `beginRendering.root`.
- Keep the first visible design container at the source frame `width` and `height`.
- Make the first visible design container the only child of `root` when applying a whole fixed prototype screen.
- Do not add outer root padding that changes the visible prototype size.
- Do not turn a fixed prototype frame into a fluid semantic layout unless the user asks for responsive reinterpretation.
- Use fixed `height`, not only `minHeight`, for fixed prototype frames.
- Keep `root` free of visual padding/background/border radius unless the user explicitly wants page chrome outside the prototype.
- If dimensions still do not match after apply, first inspect the generated A2UI stream before changing `craft.js-vue` source.

For `editor_apply_a2ui`, the `input` must be the raw A2UI JSONL string:

```
{"surfaceUpdate":{...}}
{"beginRendering":{...}}
```

Do **not** wrap it in an object like:

```json
{
  "surfaceId": "pencil-lamp",
  "jsonl": "...",
  "pageName": "lamp-from-pencil",
  "mode": "replacePage"
}
```

This returns `Missing payload.input` because the bridge expects `input` to be the JSONL stream directly.

Use bridge-compatible A2UI JSONL when the target bridge has shown timeout sensitivity to an empty `dataModelUpdate` after `beginRendering`.
