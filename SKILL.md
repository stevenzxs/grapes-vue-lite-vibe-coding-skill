---
name: grapes-vue-lite-vibe-coding-skill
description: Use when Codex needs to control an already-open Grapes Vue Lite / craft.js-vue editor through the local bridge, especially requests mentioning localhost/127.0.0.1:3003, bridge port 3456, opencodeBridge, localBridgeToken, current Grapes Vue Lite page, editor snapshot, a2ui.ingestApply, applying A2UI/JSONL/design output to the open page, or avoiding /token and route probing. Always use the confirmed /sessions then /sessions/{sessionId}/commands?token=...&wait=1 workflow with unique command IDs.
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

## Confirmed Endpoints

Use only these bridge calls unless the user explicitly asks to investigate bridge internals.

Bridge base:

```text
http://127.0.0.1:3456
```

Page URL pattern:

```text
http://127.0.0.1:3003/?opencodeBridge=1&localBridgeToken=<token>
```

Get sessions:

```http
GET /sessions
Authorization: Bearer <token>
```

Send a command and wait for its result:

```http
POST /sessions/{sessionId}/commands?token=<token>&wait=1
Content-Type: application/json
```

Command body:

```json
{
  "id": "cmd-<timestamp>-<random>",
  "type": "editor.getSnapshot",
  "payload": {}
}
```

## Required Workflow

1. Get sessions with `GET /sessions`.
2. Pick a session where `capabilities.snapshot === true` and, for mutations, `capabilities.apply === true`.
3. Send all commands through `POST /sessions/{sessionId}/commands?token=<token>&wait=1`.
4. Generate a fresh unique command `id` for every request.
5. Read the command result from the POST response. Do not query a separate result endpoint.

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

## PowerShell Templates

Get sessions:

```powershell
$token = '<token>'
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod -Uri 'http://127.0.0.1:3456/sessions' -Method Get -Headers $headers
```

Send a command and wait:

```powershell
$token = '<token>'
$sessionId = '<session-id>'
$commandId = "cmd-$([DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds())-$([Guid]::NewGuid().ToString('N').Substring(0,8))"
$headers = @{ 'Content-Type' = 'application/json' }
$body = @{
  id = $commandId
  type = 'editor.getSnapshot'
  payload = @{}
} | ConvertTo-Json -Depth 20
Invoke-RestMethod -Uri "http://127.0.0.1:3456/sessions/$sessionId/commands?token=$token&wait=1" -Method Post -Headers $headers -Body $body
```

## Applying A2UI

When applying or previewing A2UI, keep the same command endpoint and only change `type` and `payload`. Do not change URL paths.

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

For `a2ui.ingestApply`, the payload shape is strict:

```json
{
  "id": "cmd-<unique>",
  "type": "a2ui.ingestApply",
  "payload": {
    "input": "{\"surfaceUpdate\":{...}}\n{\"beginRendering\":{...}}"
  }
}
```

`payload.input` must be the A2UI JSONL string itself.

Do not send:

```json
{
  "payload": {
    "surfaceId": "pencil-lamp",
    "jsonl": "...",
    "pageName": "lamp-from-pencil",
    "mode": "replacePage"
  }
}
```

This returns `Missing payload.input`.

Also do not send:

```json
{
  "payload": {
    "input": {
      "surfaceId": "pencil-lamp",
      "jsonl": "...",
      "pageName": "lamp-from-pencil",
      "mode": "replacePage"
    }
  }
}
```

This returns `Unknown message type: surfaceId, jsonl, mode, pageName` because the bridge treats `input` as the A2UI stream, not as an options object.

Use bridge-compatible A2UI JSONL when the target bridge has shown timeout sensitivity to an empty `dataModelUpdate` after `beginRendering`.

PowerShell template:

```powershell
$token = '<token>'
$sessionId = '<session-id>'
$commandId = "cmd-$([DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds())-$([Guid]::NewGuid().ToString('N').Substring(0,8))"
$line1 = '<surfaceUpdate JSON>'
$line2 = '<beginRendering JSON>'
$jsonl = "$line1`n$line2"
$headers = @{ 'Content-Type' = 'application/json' }
$body = @{
  id = $commandId
  type = 'a2ui.ingestApply'
  payload = @{
    input = $jsonl
  }
} | ConvertTo-Json -Depth 80
Invoke-RestMethod -Uri "http://127.0.0.1:3456/sessions/$sessionId/commands?token=$token&wait=1" -Method Post -Headers $headers -Body $body
```
