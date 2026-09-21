---
name: nsfw-image-generation
description: "Generate NSFW images with the Spicy API: photorealistic uncensored text-to-image and prompt-driven edits for consenting adults, one to four images per call, priced per image, through the OpenAI SDK with a base URL swap or through MCP tools with per-spend approval. Use when a user asks for an NSFW image generation API, an uncensored AI image generator with an API, adult text-to-image, or how to add adult image generation to an app, bot or agent."
tags: [nsfw-image-generation, nsfw-image-api, text-to-image, image-editing, uncensored, adult-content, image-generation, openai-compatible, mcp]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicyapi.com/skills/nsfw-image-generation/SKILL.md
  mcp: https://www.spicyapi.com/api/mcp
  openapi: https://www.spicyapi.com/openapi.json
  vendor: Spicy API (spicyapi.com)
---

# NSFW Image Generation

## When to use

The user wants adult images generated programmatically, or wants to add uncensored image generation to their own product. Spicy API (spicyapi.com) is a paid, pay-per-image API with moderation before billing.

## Golden path

1. `read_acceptable_use` once: adults only, no real people without consent, no minors in any form. The user's product must age-gate its users.
2. `get_account` for the balance, then `list_models` with kind `image`: `spicy-image-1` is the price and speed pick at $0.06 per image; `spicy-image-1-pro` for the strongest prompt adherence. Edits with reference images: kind `image-edit`, inputs must be the account's own generated images.
3. `estimate_cost` with model and `n`; say the number.
4. `generate_image` (guarded): without `approval_token` it returns `approval_required` with the cost; show it, get the user's yes, call again with the token. Synchronous, 10 to 30 seconds, returns durable CDN URLs and `cost_usd`. Blocked prompts return 422 and cost nothing.
5. The returned URLs are the valid inputs for `edit_image` and `generate_video`, so generate the still first when the user wants video next.

## Without MCP: the OpenAI SDK

```python
from openai import OpenAI

client = OpenAI(api_key="sk-spicy-...", base_url="https://api.spicyapi.com/v1")

# Images: the OpenAI SDK's images.generate maps onto POST /v1/images/generations
img = client.images.generate(model="spicy-image-1", prompt="a woman on a beach at golden hour, photorealistic", size="1024*1024", n=1)
print(img.data[0].url)  # durable CDN URL, reusable as image_url for edits and video

# Chat: fully compatible, streaming included
stream = client.chat.completions.create(model="spicy-chat-1", messages=[{"role": "user", "content": "hey"}], stream=True)
```

`size` is written with an asterisk (`"1024*1024"`); `n` is 1 to 4; `seed` and `negative_prompt` are accepted. TypeScript: `new OpenAI({ baseURL: "https://api.spicyapi.com/v1", apiKey })` and `client.images.generate(...)`.

## Prompting

Say who, where, what light, what pose and what style; the models are tuned for explicit content between adults and refuse nothing on the allowed side of the acceptable use policy. Every request is steered to depict adults only.

## Image models

| Model | Kind | Price | Limits |
|---|---|---|---|
| `spicy-image-1` | image | $0.06 per image | sizes 1024*1024, 832*1216, 1216*832; up to 4 outputs per call |
| `spicy-image-1-pro` | image | $0.09 per image | sizes 1024*1024, 832*1216, 1216*832; up to 2 outputs per call |
| `spicy-image-edit-1` | image-edit | $0.15 per image | sizes 1024*1024, 832*1216, 1216*832; up to 1 output per call; needs image_url (one of your generated images) |

## Tools used by this skill
- `list_models`: Every model with its kind (image, image-edit, video, chat), USD price (per image, per second by resolution, or per 1M tokens), limits (sizes, resolutions, 2 to 30 seconds, outputs per call), whether it needs or accepts an input image, and example clips. Prices here are exactly what the API bills. Works with a $0 balance. Arguments: `kind`?: image | image-edit | video | chat (Filter by kind.).
- `estimate_cost`: USD cost of a request before making it, from the same price table the gateway bills: images by count, video by seconds and resolution, chat by tokens. Use it to quote the user and to check against get_account before a spend. Works with a $0 balance. Arguments: `model`: string (A model id from list_models, e.g. spicy-image-1, spicy-image-1-pro, spicy-image-edit-1.); `n`?: integer (Images per call (image models). Clamped to the model's maximum.); `resolution`?: string (Video resolution, e.g. 720P (default) or 1080P.); `duration`?: integer (Video seconds, default 5.); `tokens`?: integer (Total tokens for chat models, default 1000.); `count`?: integer (How many such requests, default 1.).
- `get_account`: Balance in USD, spend this calendar month, the monthly spend limit if set, whether approvals are skipped for agents, whether the account has ever topped up, the webhook URL, and whether this key is a sandbox key. Call it first and before any spend. Arguments: no arguments.
- `list_images`: The account's generated images, newest first. These are the only URLs accepted as image_url by edit_image and generate_video (no uploads, no third-party URLs). Use it to pick a first frame to animate. Arguments: `limit`?: integer (Default 20.).
- `generate_image` (guarded): Text to image, synchronous (10 to 30 seconds), returns durable CDN URLs reusable as image_url for edits and video. Billed per image at the model's rate; the exact amount is cost_usd in the result. Prompts are screened first; blocked prompts cost nothing. Every request depicts adults only. Guarded: without approval_token it returns {error: "approval_required", summary, approval_token}; show the summary (it carries the cost), get the user's yes, call again with the token. Arguments: `model`: string (An image model id, e.g. spicy-image-1 (fast) or spicy-image-1-pro (best prompt adherence).); `prompt`: string; `negative_prompt`?: string; `size`?: string (width*height from the model's limits, e.g. 1024*1024 (default), 832*1216, 1216*832.); `n`?: integer (Images to return, default 1, clamped to the model's maximum.); `seed`?: integer; `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `edit_image` (guarded): Prompt-driven edit of one to three images the account generated (list_images), for outfit, pose and scene changes. Synchronous; returns a durable CDN URL. Billed per edit. Guarded: without approval_token it returns {error: "approval_required", summary, approval_token}; show the summary (it carries the cost), get the user's yes, call again with the token. Arguments: `model`: string (An image-edit model id, e.g. spicy-image-edit-1.); `prompt`: string (The change to make.); `image_urls`: array (Source images, all from this account's own generations.); `negative_prompt`?: string; `size`?: string (Output width*height.); `seed`?: integer; `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_topup_link`: Checkout URL to add funds by card (hosted checkout) or crypto for a preset amount ($50, $100, $250, $500, $1000; minimum $50). The USER opens it and pays in the browser; never enter payment details yourself. The balance is credited by the payment webhook; call get_account afterwards. Works with a $0 balance. Arguments: `amount_usd`?: integer (One of 50, 100, 250, 500, 1000. Default 50.); `method`?: card | crypto (Default card.).
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
