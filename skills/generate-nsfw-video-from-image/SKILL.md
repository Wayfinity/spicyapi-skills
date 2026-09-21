---
name: generate-nsfw-video-from-image
description: "Turn a generated still into an uncensored short clip with Spicy API: generate or pick the first frame, choose between the general image-to-video models and the POV fine-tunes, estimate the per-second cost, get the user's approval, start the task and poll it to a durable video URL. Use when a user asks to animate an image, make an adult video from a picture, create image-to-video or text-to-video NSFW content, or which uncensored image-to-video API to use."
tags: [nsfw-video-generation, image-to-video, uncensored, adult-content, video-generation, mcp]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicyapi.com/skills/generate-nsfw-video-from-image/SKILL.md
  mcp: https://www.spicyapi.com/api/mcp
  openapi: https://www.spicyapi.com/openapi.json
  vendor: Spicy API (spicyapi.com)
---

# Generate NSFW video from an image

## When to use

The user wants a video from an image (or from text alone) with adult content. This skill takes it from first frame to finished clip on the user's own account with every spend approved.

## Golden path

1. `get_account`: confirm the balance. A five-second 720P clip on `spicy-motion-2` costs $0.875; say the number before starting.
2. First frame: `list_images` to reuse one of the account's generated images, or `generate_image` (guarded) to make one. Uploads and third-party URLs are rejected; for the POV fine-tunes the frame should already show the position.
3. Pick the model with `list_models` (kind video): `spicy-motion-2` for general scenes up to 15 seconds at 720P or 1080P; `spicy-motion-3` and `spicy-motion-3-fast` for 480P to 1080P up to 30 seconds with dialogue and sound, and text-to-video when no image is given; `spicy-pov-missionary-1`, `spicy-pov-doggystyle-1`, `spicy-pov-blowjob-1` for locked POV framing with no prompt engineering (the activation token is injected for you).
4. `estimate_cost` with the model, resolution and duration; show the user the total.
5. `generate_video` (guarded): it returns `approval_required` with the cost; show it, get a yes, call again with `approval_token`. The price is debited when the task is accepted and refunded if it fails.
6. `get_job` every 10 to 15 seconds until `status` is `succeeded` (`output.video_url`, a durable CDN URL) or `failed` (`error`, already refunded). Tell the user the clip is ready and where it is; do not embed it in chat unless asked.
7. Keep the prompt inside the acceptable use policy; a 422 means it was blocked and nothing was charged.

## What to tell the user up front

- Video is priced per output second by resolution; longer or 1080P costs more, and the price is fixed once the task is accepted.
- The first frame decides most of the result; generate a frame that already shows the scene rather than asking the video model to invent it.

## Video models

| Model | Kind | Price | Limits |
|---|---|---|---|
| `spicy-pov-missionary-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-pov-doggystyle-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-pov-blowjob-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-motion-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 10 seconds; needs image_url (one of your generated images) |
| `spicy-motion-2` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; needs image_url (one of your generated images) |
| `spicy-video-1` | video | $0.175 per second at 720P, $0.29 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds |
| `spicy-motion-3` | video | $0.1 per second at 480P, $0.2 per second at 720P, $0.4 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; image_url optional (text-to-video without it) |
| `spicy-motion-3-fast` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.56 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; image_url optional (text-to-video without it) |

## Tools used by this skill
- `list_models`: Every model with its kind (image, image-edit, video, chat), USD price (per image, per second by resolution, or per 1M tokens), limits (sizes, resolutions, 2 to 30 seconds, outputs per call), whether it needs or accepts an input image, and example clips. Prices here are exactly what the API bills. Works with a $0 balance. Arguments: `kind`?: image | image-edit | video | chat (Filter by kind.).
- `estimate_cost`: USD cost of a request before making it, from the same price table the gateway bills: images by count, video by seconds and resolution, chat by tokens. Use it to quote the user and to check against get_account before a spend. Works with a $0 balance. Arguments: `model`: string (A model id from list_models, e.g. spicy-image-1, spicy-image-1-pro, spicy-image-edit-1.); `n`?: integer (Images per call (image models). Clamped to the model's maximum.); `resolution`?: string (Video resolution, e.g. 720P (default) or 1080P.); `duration`?: integer (Video seconds, default 5.); `tokens`?: integer (Total tokens for chat models, default 1000.); `count`?: integer (How many such requests, default 1.).
- `get_account`: Balance in USD, spend this calendar month, the monthly spend limit if set, whether approvals are skipped for agents, whether the account has ever topped up, the webhook URL, and whether this key is a sandbox key. Call it first and before any spend. Arguments: no arguments.
- `list_images`: The account's generated images, newest first. These are the only URLs accepted as image_url by edit_image and generate_video (no uploads, no third-party URLs). Use it to pick a first frame to animate. Arguments: `limit`?: integer (Default 20.).
- `generate_image` (guarded): Text to image, synchronous (10 to 30 seconds), returns durable CDN URLs reusable as image_url for edits and video. Billed per image at the model's rate; the exact amount is cost_usd in the result. Prompts are screened first; blocked prompts cost nothing. Every request depicts adults only. Guarded: without approval_token it returns {error: "approval_required", summary, approval_token}; show the summary (it carries the cost), get the user's yes, call again with the token. Arguments: `model`: string (An image model id, e.g. spicy-image-1 (fast) or spicy-image-1-pro (best prompt adherence).); `prompt`: string; `negative_prompt`?: string; `size`?: string (width*height from the model's limits, e.g. 1024*1024 (default), 832*1216, 1216*832.); `n`?: integer (Images to return, default 1, clamped to the model's maximum.); `seed`?: integer; `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `generate_video` (guarded): Image to video (or text to video on spicy-motion-3 and spicy-motion-3-fast). Asynchronous: returns a task id at once with the price already debited; poll get_job every 10 to 15 seconds until succeeded (output.video_url) or failed (refunded). image_url must be one of the account's own generated images. Price = seconds x the per-second rate for the resolution. Our POV fine-tunes (spicy-pov-*) need a first frame already in position. Guarded: without approval_token it returns {error: "approval_required", summary, approval_token}; show the summary (it carries the cost), get the user's yes, call again with the token. Arguments: `model`: string (A video model id, e.g. spicy-motion-2, spicy-motion-3, spicy-pov-missionary-1.); `prompt`: string (What should happen in the clip.); `image_url`?: string (First frame, from list_images or a generate_image result. Required unless the model is text-to-video capable.); `resolution`?: string (480P, 720P (default) or 1080P, within the model's limits.); `duration`?: integer (Seconds, default 5, within the model's limits.); `negative_prompt`?: string; `audio`?: boolean (Generate audio where the model supports it.); `seed`?: integer; `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `get_job`: Status of a video task (queued, processing, finalizing, succeeded with output.video_url, or failed and refunded) or of a logged image or chat request by id. Arguments: `id`: string (Task or request id, e.g. sj_...).
- `list_jobs`: Recent requests on the account (images, edits, video tasks, chat), newest first: id, type, model, status, cost_usd, output URL when finished, error when failed, and whether it came from the playground, the API or an agent. Arguments: `limit`?: integer (Default 20.); `type`?: image | image-edit | video | chat.
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
