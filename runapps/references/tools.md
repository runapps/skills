# RunApps MCP tools

Server: `https://www.runapps.ai/api/mcp` (Streamable HTTP, stateless, JSON responses), header `Authorization: Bearer rk_…`. Every tool returns its result as JSON text (also as structured content). A failed call sets `isError` and the text says why; a missing scope reads `this key was not approved for the "<scope>" scope …`.

Apps are addressed by `app`: the id (UUID) or the slug, among the apps **this account created**.

## Read

### `list_apps` — scope `read`
No arguments. Returns an array of
```json
{"id": "uuid", "slug": "proj-1a2b3c4d", "name": "…", "description": "…", "category": "Productivity",
 "visibility": "draft|pending|public|private|rejected", "has_bundle": true,
 "url": "https://proj-1a2b3c4d.runapps.dev", "page_url": "https://www.runapps.ai/projects/proj-1a2b3c4d",
 "review_note": "…", "updated_at": "…"}
```
(`review_note` only when set.)

### `get_app` — scope `read`
`{"app": "<id or slug>"}`. The row above plus `long_description`, `icon_url`, `rebate_rate` (creator fee, 0–0.5), `entry_path`, `private_prompts` (ids found in the bundle).

### `get_review` — scope `read`
`{"app": …}` → `{"app_id", "slug", "visibility", "review_note", "updated_at"}`. `pending` = waiting for a reviewer; `rejected` = read `review_note`, fix, set `public` again.

### `check_bundle` — scope `read`
`{"app": …}` → the bundle report:
```json
{"entry_path": "index.html", "files": 12, "size_bytes": 123456,
 "sdk": "latest.js (platform-served, recommended)" | "bundled or imported",
 "private_prompts": ["system"], "note": "dot folders (.private/…) are never served to browsers",
 "warnings": ["…"], "url": "https://<slug>.runapps.dev"}
```
Warnings worth fixing before publishing: entry not at the zip root, SDK not loaded, an API key in the page.

### `list_models` — scope `read`
`{"capability": "text|vision|image_generation|image_edit|text_to_speech|speech_to_text|video_generation|embedding|realtime"}` (optional). Each row: `name` (use verbatim as `model`), `capability`, `provider`, and either `usd_per_mtok_in` / `usd_per_mtok_out` or `usd_per_call`.

## Develop

### `create_app` — scope `develop`
`{"name": "…", "description": "…", "category": "…"}` (name required). Returns the full app row (`visibility: "draft"`, `has_bundle: false`) plus `next` (what to do). The slug is assigned by the platform here and never changes; there is no way to choose one.

### `update_app` — scope `develop`; `visibility` also needs `publish`
```json
{"app": "…", "name": "…", "description": "one line", "long_description": "Markdown",
 "category": "Creative|Content|Productivity|Marketing|Engineering|Design|Research|Data Analysis|Business|Finance",
 "icon_url": "https://… or data:image/…", "visibility": "draft|private|public"}
```
Only the fields present are changed. `visibility: "public"` requires a bundle; the first time it stores `pending` (reviewed) and the result carries a `note`; an app that is already public stays public. Returns the full app row.

### `upload_bundle` — scope `develop`
`{"app": "…", "entry_path": "index.html"}` (entry optional). Returns
```json
{"upload_url": "https://www.runapps.ai/api/agent/upload/<token>", "method": "PUT",
 "content_type": "application/zip", "max_bytes": 52428800, "expires_in": 600,
 "curl": "curl -sS -X PUT -H 'Content-Type: application/zip' --data-binary @app.zip '<upload_url>'"}
```
The URL is single-use and needs no other authentication. The PUT answers `{"success": true, "data": <bundle report>}` or `{"success": false, "error": "…"}` (not a zip, no HTML file, entry not found, over 50 MB, link expired). On success the new bundle is live immediately and the old one is deleted.

### `upload_bundle_base64` — scope `develop`
`{"app": "…", "data_base64": "<zip as base64>", "entry_path": "…"}`. Same effect and report as the PUT. For small bundles only (the whole zip travels inside the JSON-RPC message).

## Danger

### `unpublish_app` — scope `danger`
`{"app": "…", "confirm": true}`. `public` → `private`: out of the library, still opens by link. Ask the user first.

### `delete_app` — scope `danger`
`{"app": "…", "confirm": true}`. Soft-deletes the app and removes its bundle; the slug stops resolving. Ask the user first.

## Key

### `revoke_my_key`
`{"confirm": true}`. Revokes the key this connection uses; the next call is a 401.

## Errors at the transport level

| HTTP | meaning |
|---|---|
| 401 `invalid or revoked key` | wrong or revoked `rk_` key: run the device-code flow again |
| 401 `this is a personal API key…` | a key made by hand on the API page; the MCP endpoint takes agent keys (device-code flow) |
| 404 on the upload URL | ticket expired (10 min) or already used: call `upload_bundle` again |
