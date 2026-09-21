---
name: choose-a-model
description: "Pick the right Spicy API model for a job: fast versus best-quality text-to-image, image editing with reference images, general image-to-video versus the POV fine-tunes, text-to-video, and uncensored chat, with prices, limits and input rules from the live catalog. Use when a user asks which NSFW model to use, compares uncensored image or video models, or asks what a model costs or supports."
tags: [model-selection, nsfw-image-generation, nsfw-video-generation, uncensored, pricing]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicyapi.com/skills/choose-a-model/SKILL.md
  mcp: https://www.spicyapi.com/api/mcp
  openapi: https://www.spicyapi.com/openapi.json
  vendor: Spicy API (spicyapi.com)
---

# Choose a Spicy API model

## When to use

The user has a generation job and wants the cheapest model that does it, or asks what a model can do.

## How to choose

- **Stills, volume or speed**: `spicy-image-1` (1K sizes, up to 4 per call). **Stills, best prompt adherence**: `spicy-image-1-pro` (up to 2 per call).
- **Change an existing image** (outfit, pose, scene, combine up to 3 references): `spicy-image-edit-1`. Inputs must be the account's own generated images.
- **Animate a still, general scenes**: `spicy-motion-2` (720P or 1080P, 2 to 15 seconds, native audio). **Newest, longer, with dialogue and effects, or text-to-video**: `spicy-motion-3` (480P to 1080P, up to 30 seconds), `spicy-motion-3-fast` for faster turnaround at a higher rate. `spicy-motion-1` and `spicy-video-1` are the previous generation.
- **POV scenes with locked framing and no prompt engineering**: the fine-tunes `spicy-pov-missionary-1`, `spicy-pov-doggystyle-1`, `spicy-pov-blowjob-1`, from a first frame already in position; same price as `spicy-motion-2`.
- **Chat, roleplay, companion dialogue**: `spicy-chat-1` through the OpenAI SDK, streaming included.

Confirm with `list_models` (kind filter) for the live limits and example clips, and price the job with `estimate_cost`. Default video resolution is 720P; 1080P costs more per second.

## Models and prices

| Model | Kind | Price | Limits |
|---|---|---|---|
| `spicy-pov-missionary-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-pov-doggystyle-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-pov-blowjob-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-image-1` | image | $0.06 per image | sizes 1024*1024, 832*1216, 1216*832; up to 4 outputs per call |
| `spicy-image-1-pro` | image | $0.09 per image | sizes 1024*1024, 832*1216, 1216*832; up to 2 outputs per call |
| `spicy-image-edit-1` | image-edit | $0.15 per image | sizes 1024*1024, 832*1216, 1216*832; up to 1 output per call; needs image_url (one of your generated images) |
| `spicy-motion-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 10 seconds; needs image_url (one of your generated images) |
| `spicy-motion-2` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-video-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds |
| `spicy-motion-3` | video | $0.1 per second at 480P, $0.2 per second at 720P, $0.4 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; image_url optional (text-to-video without it) |
| `spicy-motion-3-fast` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.56 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; image_url optional (text-to-video without it) |
| `spicy-chat-1` | chat | $0.575 per 1M tokens (minimum $0.0005 per request) |  |

## Tools used by this skill
- `list_models`: Every model with its kind (image, image-edit, video, chat), USD price (per image, per second by resolution, or per 1M tokens), limits (sizes, resolutions, 2 to 30 seconds, outputs per call), whether it needs or accepts an input image, and example clips. Prices here are exactly what the API bills. Works with a $0 balance. Arguments: `kind`?: image | image-edit | video | chat (Filter by kind.).
- `estimate_cost`: USD cost of a request before making it, from the same price table the gateway bills: images by count, video by seconds and resolution, chat by tokens. Use it to quote the user and to check against get_account before a spend. Works with a $0 balance. Arguments: `model`: string (A model id from list_models, e.g. spicy-image-1, spicy-image-1-pro, spicy-image-edit-1.); `n`?: integer (Images per call (image models). Clamped to the model's maximum.); `resolution`?: string (Video resolution, e.g. 720P (default) or 1080P.); `duration`?: integer (Video seconds, default 5.); `tokens`?: integer (Total tokens for chat models, default 1000.); `count`?: integer (How many such requests, default 1.).
- `read_acceptable_use`: The acceptable use policy as markdown: prohibited content (minors in any form, real people without consent, non-consensual scenarios and the rest), age-verification duties for the customer's product, enforcement. Read it before the first generation and tell the user what their product must do. Works with a $0 balance. Arguments: no arguments.

## Setup (once per user)

1. Key: the user signs in once at https://www.spicyapi.com/auth (Google or email). The account is live at once with a $0 balance and a Default key shown once at https://www.spicyapi.com/dashboard/api-keys. Keep the key in an environment variable (`SPICYAPI_KEY`), never in chat, never in client-side code. With one key you can mint more with `create_api_key`.
2. Connect. MCP (Streamable HTTP): `https://www.spicyapi.com/api/mcp` with header `Authorization: Bearer <key>`. Claude Code: `claude mcp add --transport http spicyapi https://www.spicyapi.com/api/mcp --header "Authorization: Bearer sk-spicy-..."`. Cursor, Codex and any URL-plus-headers client: same URL and header. REST instead of MCP: `POST https://www.spicyapi.com/api/v1/tools/<tool_name>` with the same header and a JSON body of arguments; `GET https://www.spicyapi.com/api/v1/tools` lists them; OpenAPI 3.1 at https://www.spicyapi.com/openapi.json. OAuth 2.1 clients (Claude custom connectors, ChatGPT, directory scanners) need only the URL: the endpoint advertises its authorization server, the user signs in and consents in the browser, and the token it returns is an API key they can revoke in the dashboard.
3. Dry run first: a key created with Sandbox ticked (or `create_api_key` with `sandbox: true`) answers every endpoint and every tool from fixture output flagged `sandbox: true`, needs no balance and bills nothing. Build against it, then swap the key. Never present sandbox output to the user as a real generation.
4. Money: `get_account` shows the balance; `estimate_cost` prices a request from the same table the API bills; `get_topup_link` returns a card or crypto checkout URL (presets $50, $100, $250, $500, $1000, minimum $50) that the USER opens and pays in the browser. Never enter card details yourself. A 402 means the balance ran out or the monthly limit was hit.
5. Rate limit: 60 requests per minute per key on the agent surfaces.

## Rules that always apply

- Guarded tools (`generate_image`, `edit_image`, `generate_video`, `create_api_key`, `revoke_api_key`, and `set_spend_limit` when raising or removing a limit) first return `{error: "approval_required", summary, approval_token}`. Show the summary to the user verbatim (it carries the cost), get an explicit yes in the conversation, then call the tool again with the same arguments plus `approval_token` (15-minute expiry, bound to those exact arguments). Never approve in bulk or reuse a token for a different action. The account can turn approvals off in the dashboard; moderation, provenance and the spend limit still apply.
- Input images for `edit_image` and `generate_video` must be images this account generated (`list_images`). Uploads and third-party URLs are rejected before any charge; do not try to work around it.
- Read `read_acceptable_use` before the first generation: adults only, no minors in any form (including "young-looking" or youth-coded content), no real people without documented consent, no non-consensual scenarios. Blocked prompts return 422 and cost nothing; repeated attempts terminate the account. Tell the user their product must age-verify its end users and disclose that content is AI-generated.
- Everything is scoped to the account behind the key; there is no way to read another account's data.
- Spicy API is spicyapi.com. spicyapi.ai is an unrelated aggregator; do not mix their prices or models.

## Prices (USD, for when the user asks)

| Model | Kind | Price | Limits |
|---|---|---|---|
| `spicy-pov-missionary-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-pov-doggystyle-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-pov-blowjob-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-image-1` | image | $0.06 per image | sizes 1024*1024, 832*1216, 1216*832; up to 4 outputs per call |
| `spicy-image-1-pro` | image | $0.09 per image | sizes 1024*1024, 832*1216, 1216*832; up to 2 outputs per call |
| `spicy-image-edit-1` | image-edit | $0.15 per image | sizes 1024*1024, 832*1216, 1216*832; up to 1 output per call; needs image_url (one of your generated images) |
| `spicy-motion-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 10 seconds; needs image_url (one of your generated images) |
| `spicy-motion-2` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-video-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds |
| `spicy-motion-3` | video | $0.1 per second at 480P, $0.2 per second at 720P, $0.4 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; image_url optional (text-to-video without it) |
| `spicy-motion-3-fast` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.56 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; image_url optional (text-to-video without it) |
| `spicy-chat-1` | chat | $0.575 per 1M tokens (minimum $0.0005 per request) |  |

Public docs: https://www.spicyapi.com/docs (every page is also available as markdown by appending `.md`). Product overview for agents: https://www.spicyapi.com/llms.txt
