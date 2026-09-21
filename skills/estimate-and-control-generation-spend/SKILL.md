---
name: estimate-and-control-generation-spend
description: "Quote what a batch of Spicy API generations will cost before running it, watch the balance and month-to-date spend, set a monthly cap, hand the user a top-up link when the balance runs out, and read the request log. Use when a user asks how much NSFW image or video generation will cost, wants a budget or spending limit for an agent, or asks what they spent."
tags: [cost-estimation, spend-limit, budget, nsfw-api, mcp]
license: MIT
metadata:
  version: "1.1.0"
  homepage: https://www.spicyapi.com/skills/estimate-and-control-generation-spend/SKILL.md
  mcp: https://www.spicyapi.com/api/mcp
  openapi: https://www.spicyapi.com/openapi.json
  vendor: Spicy API (spicyapi.com)
---

# Estimate and control generation spend

## When to use

Anything about money on a Spicy API account: quotes, budgets, caps, top-ups, usage. Also the guard rail to run before any batch job an agent starts on its own.

## Golden path

1. `get_account`: balance, spent this month, the monthly limit and what is left under it, whether approvals are skipped.
2. `estimate_cost` per request type with `count` for batches (images by `n`, video by `resolution` and `duration`, chat by `tokens`). Sum and present a table; the numbers are the ones the API bills, and `cost_usd` in every response confirms them afterwards.
3. Budget: `set_spend_limit` with the monthly cap the user names. Lowering or setting a cap needs no approval; raising or removing one returns `approval_required`. Past the cap every request returns 402 `spend_limit_reached` and nothing is charged.
4. Funding: when the balance is short, `get_topup_link` with an amount from the presets ($50, $100, $250, $500, $1000) and `method` card or crypto. Hand the URL to the user; never enter payment details. The balance updates once the payment webhook fires; confirm with `get_account`.
5. Review: `get_usage` (days) for spend by model, type and source, `list_jobs` for the recent requests with status and cost, `get_job` for one of them.
6. Before an unattended batch: confirm the total against the balance and the remaining monthly allowance, and either keep approvals on (one yes per spend) or have the user turn them off in the dashboard with a cap in place.

## What to tell the user up front

- Failed generations are refunded and blocked prompts are never charged, so the estimate is an upper bound.
- Video is debited when the task is accepted, images when the request succeeds; both show as `cost_usd`.

## Prices (USD)

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
- `get_account`: Balance in USD, spend this calendar month, the monthly spend limit if set, whether approvals are skipped for agents, whether the account has ever topped up, the webhook URL, and whether this key is a sandbox key. Call it first and before any spend. Arguments: no arguments.
- `get_job`: Status of a video task (queued, processing, finalizing, succeeded with output.video_url, or failed and refunded) or of a logged image or chat request by id. Arguments: `id`: string (Task or request id, e.g. sj_...).
- `list_jobs`: Recent requests on the account (images, edits, video tasks, chat), newest first: id, type, model, status, cost_usd, output URL when finished, error when failed, and whether it came from the playground, the API or an agent. Arguments: `limit`?: integer (Default 20.); `type`?: image | image-edit | video | chat.
- `get_usage`: Spend and request counts over the last N days, totalled and broken down by model, type and source (playground, api, agent), with the error count. Reads the account's request log. Arguments: `days`?: integer (Default 7.).
- `get_topup_link`: Checkout URL to add funds by card (hosted checkout) or crypto for a preset amount ($50, $100, $250, $500, $1000; minimum $50). The USER opens it and pays in the browser; never enter payment details yourself. The balance is credited by the payment webhook; call get_account afterwards. Works with a $0 balance. Arguments: `amount_usd`?: integer (One of 50, 100, 250, 500, 1000. Default 50.); `method`?: card | crypto (Default card.).
- `set_spend_limit` (guarded): Cap what this account can spend per calendar month (UTC), enforced at debit time on every surface: requests past the cap return 402 without charging. Lowering or setting a limit never needs approval; raising or removing one does. null removes the limit. Arguments: `monthly_usd`: any (Monthly cap in USD, or null to remove it.); `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).

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
