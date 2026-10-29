---
name: runapps
description: "Build and publish AI web apps on RunApps (runapps.ai) through its MCP tools, and write the app code with the RunApps JavaScript SDK (@runappsai/sdk). Covers getting authorized and connected (device-code flow, agent key, Bearer header, scopes), the platform model (apps, slugs, bundles, visibility and review, creator fee), every MCP tool, the SDK (sign-in, chat, agents with tools, skills and a JS sandbox, images, audio, video, per-user files, IndexedDB sync, private prompts), the bundle rules (zip layout, dot files, entry page) and the safety rules. Use whenever the runapps MCP server is connected, or when a task mentions RunApps, runapps.dev, publishing an app there, the RunApps SDK, or an rk_ key."
license: MIT
---
# RunApps via MCP

RunApps (https://www.runapps.ai) publishes **AI web apps**: a creator uploads a static frontend bundle (zip), it is served on its own `https://<slug>.runapps.dev`, users sign in with one RunApps account and pay for their own model usage, and the creator earns a fee on every call. Everything below is done through the `runapps` MCP server and the SDK. Not connected yet? See "Connecting".

This file is the entry point: connecting, the platform model, the tool catalogue, the workflow and the rules. The SDK (how the app code talks to models and files, and `client.agent()` — the tool-calling loop with skills, a workspace and a sandbox) is `references/sdk.md`; exact tool parameters are `references/tools.md`. Read only what the task needs.

## Connecting (authorization)

The authoritative, always-current version is `https://www.runapps.ai/agent/setup.md`; this is the short form. The user never types a password for you and you never see one: you ask for a key, the user clicks Approve once in a browser.

1. **Request a key**: `POST https://www.runapps.ai/api/agent/device/code` with JSON `{"name": "<your name> @ <machine>", "scopes": "read,develop,publish,gateway"}`. The response has `device_code`, `user_code`, `verification_uri_complete` (`https://www.runapps.ai/agent/authorize?code=XXXX-XXXX`), `expires_in` (600 s) and `interval` (5 s). Show `verification_uri_complete` to the user verbatim and ask them to open it and approve. `name` is the key's identity: approving a request with the same name revokes the previous key of that name, so re-running setup does not pile keys up.
2. **Poll**: `POST https://www.runapps.ai/api/agent/device/token` with `{"device_code": "…"}` every `interval` seconds. `status` is `pending` (wait), `slow_down` (back off), `denied` / `expired` (stop and tell the user) or `approved`, which returns `api_key` (`rk_…`), `key_prefix`, `scopes` and `mcp_url` — **once**. Write it into your MCP configuration immediately; never into the conversation, logs or a repository.
3. **Connect**: MCP Streamable HTTP at `https://www.runapps.ai/api/mcp` with header `Authorization: Bearer <api_key>`. Claude Code: `claude mcp add --transport http runapps "https://www.runapps.ai/api/mcp" --header "Authorization: Bearer <api_key>"`. Codex: `[mcp_servers.runapps] url = "https://www.runapps.ai/api/mcp"`, `http_headers = { Authorization = "Bearer <api_key>" }`. Other clients: the same URL and header. Then `tools/list`; the `instructions` from `initialize` give the recommended workflow.

Scopes: `read`, `develop`, `publish`, `gateway` are granted by default; `danger` (delete / unpublish) is never included unless you ask for it explicitly and tell the user why. The user can untick scopes on the approval page, and later rename the key, change its scopes or revoke it at `https://www.runapps.ai/dashboard/api` (a scope change applies to your next call, no new key needed). A tool called without its scope answers with an error naming the scope: ask the user to authorize again, do not work around it.

The same key works at the gateway (`https://www.runapps.ai/v1`, OpenAI-compatible, `Authorization: Bearer rk_…`) when it has the `gateway` scope — for testing the app you build; each call is billed to the user. To disconnect for good, call `revoke_my_key` with `confirm: true`.

## The platform model

| Object | What it is | How you reach it |
|---|---|---|
| App (project) | A published bundle with a platform-assigned `slug`, name, one-line description, Markdown long description, category, icon, visibility and creator fee (`rebate_rate`). Created as a **draft** with no bundle, so the slug exists before any code does. | `list_apps`, `get_app`, `create_app`, `update_app` |
| Bundle | The zip of the app's build. Stored whole; files are served out of it on `https://<slug>.runapps.dev`. Dot files (`.private/…`) are never served; `index.html` at the zip root is the default entry. | `upload_bundle` (+ `PUT`), `upload_bundle_base64`, `check_bundle` |
| Visibility | `draft` (only the creator) → `private` (unlisted: anyone with the link, no review) or `public` (in the library; the **first** move to public is reviewed and shows as `pending`, later updates are live at once; a rejection carries a `review_note`). | `update_app` with `visibility`, `get_review` |
| User & sign-in | Users sign in with their RunApps account inside the app (the SDK does it; `new RunApps()` with no key). Each user pays their own model usage plus the creator fee. The app never holds a key. | SDK (`references/sdk.md`) |
| Private prompts | Prompt files the app ships but browsers never see: `.private/prompts/<id>.md` in the zip; the app sends `prompt: { id, vars }` and the gateway fills and prepends it. | `get_app` / `check_bundle` list the ids found |
| Per-user files & sync | Each user has a file space per app (`client.files`), and an IndexedDB sync (`client.sync`) mirrors a local database into it. | SDK |
| Models | A catalogue of chat, vision, image, speech, transcription, video, embedding and realtime models; the request names the model by its catalogue name. | `list_models` |
| Agent key | The `rk_` key you hold: `kind = agent`, scoped; listed under the user's API Keys. | `revoke_my_key` |

Categories: Creative, Content, Productivity, Marketing, Engineering, Design, Research, Data Analysis, Business, Finance.

## Workflow

1. **Create**: `create_app` with `name` (and `description`, `category`). The result has `slug` (`proj-xxxxxxxx`, assigned by the platform and never chosen by you or the user; an admin may later give a listed app a readable one) and `url`. Tell the user the slug.
2. **Develop locally** against the slug (`references/sdk.md`, "Build an app"):
   ```html
   <script src="https://www.runapps.ai/sdk/latest.js"></script>
   <script>
     const client = new RunApps({ project: "<slug>" });
     const r = await client.chat.create({ model: "Claude Sonnet 4.6", messages: [{ role: "user", content: "hi" }] });
   </script>
   ```
   Sign-in from `http://localhost:<port>` works and calls go through the real gateway. Pick models from `list_models` (exact names). Never put an `rk_` key in app code.
3. **Build and zip the contents** of the build folder: `cd dist && zip -r ../app.zip .` so `index.html` is at the zip root (zipping the folder itself is the most common mistake). Absolute asset paths work; dot folders are hidden.
4. **Upload**: `upload_bundle` → `PUT` the zip to `upload_url` (`curl -sS -X PUT -H 'Content-Type: application/zip' --data-binary @app.zip '<upload_url>'`). The PUT response is the bundle report: fix every warning (SDK not loaded, entry not at root, a key in the page). `check_bundle` re-reads it any time.
5. **Describe**: `update_app` with `description` (one line, what it does), `long_description` (Markdown: what the user gets, how to use it, what it costs in practice), `category`, `icon_url` (square PNG/SVG, https or data URL).
6. **Ship**: `update_app` `visibility: "private"` for a link to share, or `"public"` for the library. After `"public"`, `get_review` shows `pending` until a reviewer decides; `rejected` comes with a `review_note` to act on, then set `public` again. Open `url` yourself (or ask the user to) before publishing: a bundle that 404s or shows a blank page will be rejected.

Updates: upload a new bundle or change metadata any time; a public app stays public.

## Rules

- `[danger]` tools (`unpublish_app`, `delete_app`) remove something other people may be using. Ask the user first and pass `confirm: true` only after they agree; without the `danger` scope they are refused anyway.
- Every model call made with the `gateway` scope, and every call the user's app makes, costs real money. Test with short prompts and cheap models; `list_models` gives prices.
- The key is the user's: one per machine (the `name` distinguishes them), never shared, never written anywhere but the MCP config. Revoke with `revoke_my_key` when asked to disconnect.
- Do not invent model names; use `list_models`. An unknown name is a 400, never a silent swap.
- A private prompt's text must stay private: the gateway withholds replies that echo it.

## Where to read next

| Task | Read |
|---|---|
| Writing the app: sign-in, chat, streaming, images, audio, video, files, sync, private prompts, errors | `references/sdk.md` |
| Exact tool parameters and result shapes | `references/tools.md` |
| Connecting (full text, always current) | https://www.runapps.ai/agent/setup.md |
| Building a bundle, publishing, earnings | `references/sdk.md` → "Build an app", "Publish it" |
