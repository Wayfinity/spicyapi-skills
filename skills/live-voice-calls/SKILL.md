---
name: live-voice-calls
description: "Add live, interruptible voice calls with an adult AI companion to an app: open a session with a persona and voice on the Spicy API, hand the one-time ticket URL to the browser or app, stream microphone audio over a WebSocket and play the replies, billed per turn. Use when a user asks for a realtime voice API, a phone-call style companion, speech-to-speech role-play, or an OpenAI Realtime compatible uncensored voice model."
tags: [voice-call, realtime-voice, ai-companion, websocket, speech-to-speech, uncensored, adult-content]
license: MIT
metadata:
  version: "1.4.0"
  homepage: https://www.spicyapi.com/skills/live-voice-calls/SKILL.md
  mcp: https://www.spicyapi.com/api/mcp
  openapi: https://www.spicyapi.com/openapi.json
  vendor: Spicy API (spicyapi.com)
---

# Live voice calls with an AI companion

## When to use

The user wants their users to talk to a companion out loud, in real time, with interruptions handled. This is a WebSocket API for the user's app; an agent sets it up in code, it cannot hold a call itself. For one-off voiced lines use the nsfw-voice-generation skill instead.

## How it works

1. Server side, with the API key: `POST https://api.spicyapi.com/v1/realtime/sessions` with `{ model: "spicy-live-1", voice, instructions, turn_detection }`. `instructions` is the persona (up to 8,000 characters, screened like a prompt). `voice` is one of 28 adult voices (default `Tina`): `Tina`, `Cindy`, `Liora Mira`, `Serena`, `Maia`, `Mia`, `Katerina`, `Jennifer`, `Sonrisa`, `Hana`, `Griet`, `Sigga`, `Bea`, `Chloe`, `Kiki`, `Sohee`, `Zane`, `Ryan`, `Raymond`, `Theo Calm`, `Aiden`, `Andre`, `Dolce`, `Bodega`, `Jakub`, `Alek`, `Emilien`, `Evan`. `turn_detection`: `{ type: "server_vad" }` (default), `{ type: "semantic_vad" }`, optional `threshold` and `silence_duration_ms`, or `null` for push-to-talk. Opening a session needs a $0.05 balance.
2. The response carries `url` (`wss://api.spicyapi.com/v1/realtime?session=rts_...`), a one-time ticket that works once within 60 seconds. Send only that URL to the browser or app; the API key never leaves the server.
3. The client connects and speaks the OpenAI Realtime event shape. Send `input_audio_buffer.append` with `audio` = base64 PCM16 mono 16 kHz in chunks of about 100 ms (and `input_audio_buffer.commit` / `response.create` in push-to-talk). Play `response.audio.delta` (base64 PCM16 mono 24 kHz). Transcripts arrive as `conversation.item.input_audio_transcription.completed` (the caller) and `response.audio_transcript.delta` / `.done` (the companion). Stop playback on `input_audio_buffer.speech_started` so the caller can interrupt. Text turns: `conversation.item.create` with `input_text` parts. `session.update` may change only `session.turn_detection`.
4. Billing, per 1M tokens: text in $0.46, audio in $1.86, text out $1.4, audio out $3.74, billed per turn (about half a cent per minute of conversation; opening a session needs a $0.05 balance). Audio is about 7 tokens per second in and 12.5 out, and the conversation so far is re-billed as input each turn; `estimate_cost` with `minutes` gives the audio-only floor. After every billed turn the server sends `spicy.usage` `{turn, cost_usd, total_cost_usd}`. A call lasts up to 13 minutes (`spicy.session_expired`), then open a new session.
5. Errors arrive as `error` events: `payment_required` (balance or monthly cap, the call ends), `content_blocked` (a transcript failed moderation, the call ends), `moderation_unavailable`, `upstream_unavailable`, `unsupported_event`, `unsupported_item`, `unsupported_field`, `invalid_json`, `invalid_event`.

No cloned or custom voices, no camera or image input and no tools on live calls. A sandbox key opens a session and returns canned events, billing nothing.

## Browser snippet

```js
// Server: mint a ticket (Node). Never ship the key to the client.
const r = await fetch("https://api.spicyapi.com/v1/realtime/sessions", {
  method: "POST",
  headers: { Authorization: `Bearer ${process.env.SPICYAPI_KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({ model: "spicy-live-1", voice: "Tina", instructions: "You are Mara, a flirty adult companion. Keep replies short." }),
});
const { url } = await r.json(); // hand url to the browser; it works once, within 60 seconds

// Browser: microphone in (PCM16 16 kHz), replies out (PCM16 24 kHz).
const ws = new WebSocket(url);
const mic = await navigator.mediaDevices.getUserMedia({ audio: true });
const inCtx = new AudioContext({ sampleRate: 16000 });
const proc = inCtx.createScriptProcessor(2048, 1, 1); // about 128 ms per chunk
proc.onaudioprocess = (e) => {
  const f = e.inputBuffer.getChannelData(0);
  const pcm = new Int16Array(f.length);
  for (let i = 0; i < f.length; i++) pcm[i] = Math.max(-1, Math.min(1, f[i])) * 0x7fff;
  if (ws.readyState === 1) ws.send(JSON.stringify({ type: "input_audio_buffer.append", audio: btoa(String.fromCharCode(...new Uint8Array(pcm.buffer))) }));
};
inCtx.createMediaStreamSource(mic).connect(proc);
proc.connect(inCtx.destination);

const outCtx = new AudioContext({ sampleRate: 24000 });
let playAt = 0;
let playing = [];
ws.onmessage = ({ data }) => {
  const ev = JSON.parse(data);
  if (ev.type === "response.audio.delta") {
    const bytes = Uint8Array.from(atob(ev.delta), (ch) => ch.charCodeAt(0));
    const s16 = new Int16Array(bytes.buffer);
    const buf = outCtx.createBuffer(1, s16.length, 24000);
    buf.getChannelData(0).set(Float32Array.from(s16, (v) => v / 0x8000));
    const src = outCtx.createBufferSource();
    src.buffer = buf;
    src.connect(outCtx.destination);
    playAt = Math.max(playAt, outCtx.currentTime);
    src.start(playAt);
    playAt += buf.duration;
    playing.push(src);
  } else if (ev.type === "input_audio_buffer.speech_started") {
    playing.forEach((s) => s.stop()); // the caller interrupts
    playing = [];
    playAt = 0;
  } else if (ev.type === "spicy.usage") {
    console.log("call cost so far", ev.total_cost_usd);
  } else if (ev.type === "error") {
    console.warn(ev.error.code, ev.error.message);
  }
};
```

Native apps do the same with their audio APIs: 16 kHz mono PCM16 in, 24 kHz mono PCM16 out.

## What to tell the user up front

- The persona and both sides' transcripts are screened; a blocked line ends the call. Adults only, and their product must age-gate users.
- Calls bill while they run; set a monthly cap (`set_spend_limit`) before launch.

## Tools used by this skill
- `list_models`: Every model with its kind (image, image-edit, video, video-edit, chat, speech, realtime, transcription, embedding), USD price (per image, per second by resolution or quality, per 1M tokens, per 10,000 characters of speech input, or per minute of audio), limits (image sizes, outputs per call, resolutions, clip lengths, aspect ratios, prompt character caps), whether it needs or accepts an input image or references, and example clips. Video rows carry `inputs`: which of first frame, last frame, reference images, clip and audio the model takes, `inputs.audio.mode` (driving: the clip is the soundtrack and the mouth follows it; reference: the model generates sound and uses the clip for voice, tone and beat; none) and `inputs.combinations`, the valid input combinations in plain sentences. Read those before generate_video. `silent: true` marks a video model that renders without sound; `bills_input_video_seconds: true` one that also bills the seconds of an input clip. Video-edit rows carry `video_edit` (instruction or motion mode, clip lengths, reference images, whether input seconds are billed) for edit_video. Speech rows carry `speech`: preset voices, whether `instructions` are accepted, custom voices, languages, inline `tags` and voice creation fees. Realtime rows (spicy-live-1) are live voice calls over a WebSocket: a server or app opens a session with POST /v1/realtime/sessions and connects the returned ticket URL; they cannot be driven from a tool call. Prices here are exactly what the API bills. Works with a $0 balance. Arguments: `kind`?: image | image-edit | video | video-edit | chat | speech | realtime | transcription | embedding (Filter by kind.).
- `estimate_cost`: USD cost of a request before making it, from the same price table the gateway bills: images by count, video by seconds and resolution (plus the input clip or action clip on models that bill it), video edits by clip length (input plus output seconds for instruction edits, output seconds for motion transfer), chat by tokens, speech by characters of input (plus instructions), transcription by seconds of audio, embeddings by tokens, live calls by minutes. Use it to quote the user and to check against get_account before a spend. Works with a $0 balance. Arguments: `model`: string (A model id from list_models, e.g. spicy-image-1, spicy-image-1-pro, spicy-image-photo-1.); `n`?: integer (Images per call (image models), clamped to the model's maximum.); `resolution`?: string (Video or video-edit resolution, e.g. 720P (default) or 1080P.); `quality`?: standard | pro (Motion transfer tier on spicy-animate-1, default standard.); `duration`?: integer (Video seconds, default 5; -1 for a model-chosen length where supported.); `input_seconds`?: number (Length of the input clip in seconds: the clip to edit or motion clip on video-edit models (default 5), or a video_url on video models that bill input seconds.); `motion`?: string (A library motion id (list_motions) on spicy-animate-1; its length sets the output length.); `action`?: string (An action id (list_actions) on spicy-motion-3 and -fast: its reference clip seconds (4 on most) are billed as input, and duration defaults to 8.); `fps`?: any (Video frame rate: 30 (default) or 60. 60 smooths motion with frame interpolation for 20% on top of the video price (input or action clip seconds included); audio is kept. If that step fails the 30 fps clip is delivered and the fee refunded.); `tokens`?: integer (Chat: total tokens, default 1000. Embeddings: text tokens, default 1000.); `image_tokens`?: integer (Embeddings on spicy-embed-vision-1: image tokens, default 0.); `characters`?: integer (Speech models: characters of input plus instructions (CJK ideographs and inline tags count too), default 100.); `audio_seconds`?: number (Transcription: seconds of audio, default 60.); `minutes`?: number (Live calls (spicy-live-1): minutes of conversation, default 1.); `count`?: integer (How many such requests, default 1.).
- `get_account`: Balance in USD, spend this calendar month, the monthly spend limit if set, whether approvals are skipped for agents, whether the account has ever topped up, the webhook URL, and whether this key is a sandbox key. Call it first and before any spend. Arguments: no arguments.
- `get_usage`: Spend and request counts over the last N days, totalled and broken down by model, type and source (playground, api, agent), with the error count. Reads the account's request log. Arguments: `days`?: integer (Default 7.).
- `set_spend_limit` (guarded): Cap what this account can spend per calendar month (UTC), enforced at debit time on every surface: requests past the cap return 402 without charging. Lowering or setting a limit never needs approval; raising or removing one does. null removes the limit. Arguments: `monthly_usd`: any (Monthly cap in USD, or null to remove it.); `approval_token`?: string (Approval token from a previous approval_required response, after the user said yes.).
- `read_acceptable_use`: The acceptable use policy as markdown: prohibited content (minors in any form, images or video of real people with or without consent, non-consensual scenarios and the rest), age-verification duties for the customer's product, enforcement. Read it before the first generation and tell the user what their product must do. Works with a $0 balance. Arguments: no arguments.

## Setup (once per user)

1. Key: the user signs in once at https://www.spicyapi.com/auth (Google or email). The account is live at once with a $0 balance and a Default key shown once at https://www.spicyapi.com/dashboard/api-keys. Keep the key in an environment variable (`SPICYAPI_KEY`), never in chat, never in client-side code. With one key you can mint more with `create_api_key`.
2. Connect. MCP (Streamable HTTP): `https://www.spicyapi.com/api/mcp` with header `Authorization: Bearer <key>`. Claude Code: `claude mcp add --transport http spicyapi https://www.spicyapi.com/api/mcp --header "Authorization: Bearer sk-spicy-..."`. Cursor, Codex and any URL-plus-headers client: same URL and header. REST instead of MCP: `POST https://www.spicyapi.com/api/v1/tools/<tool_name>` with the same header and a JSON body of arguments; `GET https://www.spicyapi.com/api/v1/tools` lists them; OpenAPI 3.1 at https://www.spicyapi.com/openapi.json. OAuth 2.1 clients (Claude custom connectors, ChatGPT, directory scanners) need only the URL: the endpoint advertises its authorization server, the user signs in and consents in the browser, and the token it returns is an API key they can revoke in the dashboard.
3. Dry run first: a key created with Sandbox ticked (or `create_api_key` with `sandbox: true`) answers every endpoint and every tool from fixture output flagged `sandbox: true`, needs no balance and bills nothing. Build against it, then swap the key. Never present sandbox output to the user as a real generation.
4. Money: `get_account` shows the balance; `estimate_cost` prices a request from the same table the API bills; `get_topup_link` returns a card or crypto checkout URL (presets $50, $100, $250, $500, $1000, minimum $50) that the USER opens and pays in the browser. Never enter card details yourself. A 402 means the balance ran out or the monthly limit was hit.
5. Rate limit: 60 requests per minute per key on the agent surfaces.

## Rules that always apply

- Guarded tools (`generate_image`, `edit_image`, `generate_video`, `edit_video`, `transcribe_audio`, `create_embeddings`, `upload_audio`, `delete_audio`, `generate_speech`, `create_voice`, `delete_voice`, `create_character`, `delete_character`, `create_api_key`, `revoke_api_key`, and `set_spend_limit` when raising or removing a limit) first return `{error: "approval_required", summary, approval_token}`. Show the summary to the user verbatim (it carries the cost), get an explicit yes in the conversation, then call the tool again with the same arguments plus `approval_token` (15-minute expiry, bound to those exact arguments). Never approve in bulk or reuse a token for a different action. The account can turn approvals off in the dashboard; moderation, provenance and the spend limit still apply.
- Input images for `edit_image`, `generate_video`, `edit_video` and `create_embeddings` must be images this account generated (`list_images`); input clips (`video_url`) must be the account's own finished tasks. Uploads and third-party URLs are rejected before any charge; do not try to work around it. The external inputs are audio: `upload_audio` takes a WAV or MP3 (2 to 30 seconds) from a public URL, transcribes and screens it for $0.01, and its returned `url`, or the `url` of a `generate_speech` clip, is what `audio_url` accepts; `transcribe_audio` reads any public audio URL into text.
- The same person across many outputs: `create_character` from 1 to 3 generated images of them, then pass the returned id as `character` to `generate_image`, `edit_image` or `generate_video` (`spicy-character-video-1` needs no first frame). This is the supported way to get consistency; never ask the user for a photo.
- Read `read_acceptable_use` before the first generation: adults only, no minors in any form (including "young-looking" or youth-coded content), no images or video of real people (with or without consent; voice clones need the speaker's written consent), no non-consensual scenarios. Blocked prompts return 422 and cost nothing; repeated attempts terminate the account. Tell the user their product must age-verify its end users and disclose that content is AI-generated.
- When the user's product serves other people, pass each person's own stable id as `user` on every call. Screening history, strikes and suspensions then apply to that one person (403 `end_user_suspended`) instead of the whole account, and the id is kept with each generation's record, so a report about one person's output is traced to them and not to everyone on the account.
- Everything is scoped to the account behind the key; there is no way to read another account's data.
- Spicy API is spicyapi.com. spicyapi.ai is an unrelated aggregator; do not mix their prices or models.

## Prices (USD, for when the user asks)

| Model | Kind | Price | Limits | Inputs (audio mode, valid combinations) |
|---|---|---|---|---|
| `spicy-image-1-pro` | image | $0.09 per image | sizes 1024*1024, 832*1216, 1216*832, 1280*1280, 1440*1440, 1024*1536, 1536*1024, 1080*1920, 1920*1080, 1152*2048, 2048*1152; up to 6 outputs per call |  |
| `spicy-image-photo-1` | image | $0.08 per image | sizes 1024*1024, 832*1216, 1216*832, 1280*1280, 1440*1440, 1024*1536, 1536*1024, 1080*1920, 1920*1080, 1152*2048, 2048*1152; up to 6 outputs per call |  |
| `spicy-image-1` | image | $0.06 per image | sizes 1024*1024, 832*1216, 1216*832, 1280*1280, 1440*1440, 1024*1536, 1536*1024, 1080*1920, 1920*1080, 1152*2048, 2048*1152; up to 6 outputs per call |  |
| `spicy-image-action-1` | image | $0.15 per image | up to 4 outputs per call |  |
| `spicy-image-reference-1` | image | $0.24 per image | up to 1 output per call |  |
| `spicy-image-edit-1` | image-edit | $0.15 per image | sizes 1024*1024, 832*1216, 1216*832, 1280*1280, 1440*1440, 1024*1536, 1536*1024, 1080*1920, 1920*1080, 1152*2048, 2048*1152; up to 6 outputs per call; needs image_url (one of your generated images) |  |
| `spicy-motion-3` | video | $0.1 per second at 480P, $0.2 per second at 720P, $0.4 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; text-to-video aspect ratios 16:9, 9:16, 1:1, 4:3, 3:4, 21:9; prompt up to 20,000 characters; reference clip (video_url) up to 15 seconds; the input clip's seconds are billed too, at the same rate; image_url optional (text-to-video without it) | Audio: reference audio (the model generates sound and uses the clip for voice, tone and beat), up to 5 clips and 15s total. Valid combinations: "Text prompt only, with aspect_ratio; dialogue written in the prompt is spoken natively"; "First frame (image_url), optionally a last frame (last_frame_url); no references or audio in the same request"; "Reference images (up to 10, or a character) plus a text prompt: same person, framing chosen by the model"; "Reference images, a reference video (video_url, 15s max) and reference audio (up to 5 clips, 15s total) in any mix, with a text prompt; audio guides voice, tone and beat while the words come from the prompt" |
| `spicy-motion-3-fast` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.56 per second at 1080P | resolutions 480P, 720P, 1080P; 2 to 30 seconds; text-to-video aspect ratios 16:9, 9:16, 1:1, 4:3, 3:4, 21:9; prompt up to 20,000 characters; reference clip (video_url) up to 15 seconds; the input clip's seconds are billed too, at the same rate; image_url optional (text-to-video without it) | Audio: reference audio (the model generates sound and uses the clip for voice, tone and beat), up to 5 clips and 15s total. Valid combinations: "Text prompt only, with aspect_ratio; dialogue written in the prompt is spoken natively"; "First frame (image_url), optionally a last frame (last_frame_url); no references or audio in the same request"; "Reference images (up to 10, or a character) plus a text prompt: same person, framing chosen by the model"; "Reference images, a reference video (video_url, 15s max) and reference audio (up to 5 clips, 15s total) in any mix, with a text prompt; audio guides voice, tone and beat while the words come from the prompt" |
| `spicy-cinema-1-image` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.36 per second at 1080P | resolutions 480P, 720P, 1080P; 3 to 15 seconds; prompt up to 5,000 characters; needs image_url (one of your generated images) | Audio: no audio input. Valid combinations: "First frame (image_url) with a text prompt; the output keeps the image's shape" |
| `spicy-motion-2` | video | $0.2 per second at 720P, $0.3 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; prompt up to 5,000 characters; needs image_url (one of your generated images) | Audio: driving audio (the clip is the soundtrack and the mouth follows it). Valid combinations: "First frame (image_url), optionally a last frame (last_frame_url)"; "First frame plus driving audio (audio_url): the clip becomes the soundtrack and the mouth follows it"; "First frame, last frame and driving audio together"; "A clip to continue (video_url, 2 to 10s) instead of a first frame, optionally with a last frame" |
| `spicy-character-video-1` | video | $0.2 per second at 720P, $0.3 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; prompt up to 5,000 characters; image_url optional (text-to-video without it) | Audio: no audio input. Valid combinations: "A character (or reference images) plus a text prompt"; "A character plus a first frame (image_url) that opens the clip" |
| `spicy-cinema-1-character` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.36 per second at 1080P | resolutions 480P, 720P, 1080P; 3 to 15 seconds; text-to-video aspect ratios 16:9, 9:16, 1:1, 4:3, 3:4, 4:5, 5:4, 9:21, 21:9; prompt up to 5,000 characters | Audio: no audio input. Valid combinations: "A character or reference images (up to 9) plus a text prompt that names them Image 1, Image 2..." |
| `spicy-cinema-1` | video | $0.14 per second at 480P, $0.28 per second at 720P, $0.36 per second at 1080P | resolutions 480P, 720P, 1080P; 3 to 15 seconds; text-to-video aspect ratios 16:9, 9:16, 1:1, 4:3, 3:4, 4:5, 5:4, 9:21, 21:9; prompt up to 5,000 characters | Audio: no audio input. Valid combinations: "Text prompt only, with aspect_ratio; the clip has native audio" |
| `spicy-video-1` | video | $0.2 per second at 720P, $0.3 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; text-to-video aspect ratios 16:9, 9:16, 1:1, 4:3, 3:4; prompt up to 5,000 characters | Audio: driving audio (the clip is the soundtrack and the mouth follows it). Valid combinations: "Text prompt only, with aspect_ratio"; "Text prompt plus driving audio (audio_url): the clip becomes the soundtrack and motion follows it" |
| `spicy-motion-draft-1` | video | $0.05 per second at 720P, $0.075 per second at 1080P | resolutions 720P, 1080P; 2 to 15 seconds; prompt up to 1,500 characters; always silent; needs image_url (one of your generated images) | Audio: no audio input. Valid combinations: "First frame (image_url) with a text prompt; the clip is silent" |
| `spicy-motion-1` | video | $0.2 per second at 720P, $0.3 per second at 1080P | resolutions 720P, 1080P; 2 to 10 seconds; prompt up to 1,500 characters; needs image_url (one of your generated images) | Audio: no audio input. Valid combinations: "First frame (image_url) with a text prompt" |
| `spicy-video-edit-1` | video-edit | $0.2 per second at 720P, $0.3 per second at 1080P | resolutions 720P, 1080P; text-to-video aspect ratios 16:9, 9:16, 1:1, 4:3, 3:4; prompt up to 5,000 characters; the input clip's seconds are billed too, at the same rate; edits your own 2 to 10 second clips (output up to 10 seconds) with up to 4 reference images, aspect_ratio, keep_audio |  |
| `spicy-cinema-1-edit` | video-edit | $0.28 per second at 720P, $0.48 per second at 1080P | resolutions 720P, 1080P; prompt up to 5,000 characters; the input clip's seconds are billed too, at the same rate; edits your own 3 to 30 second clips (output up to 15 seconds) with up to 5 reference images, keep_audio |  |
| `spicy-animate-1` | video-edit | $0.24 per second at standard, $0.36 per second at pro | resolutions standard, pro; motion transfer: image_url plus a motion id (GET /v1/videos/motions) or your own 2 to 30 second clip, quality standard or pro, billed per output second; needs image_url (one of your generated images) |  |
| `spicy-companion-1` | chat | $1 per 1M prompt tokens and $2.8 per 1M completion tokens (minimum $0.001 per request) |  |  |
| `spicy-companion-1-flash` | chat | $0.1 per 1M prompt tokens and $0.8 per 1M completion tokens (minimum $0.0005 per request) |  |  |
| `spicy-chat-1` | chat | $0.8 per 1M prompt tokens ($0.16 when served from cache) and $2.4 per 1M completion tokens (minimum $0.001 per request) |  |  |
| `spicy-voice-2` | speech | $0.40 per 10,000 characters of input text | input up to 5,000 characters; 2 preset voices; `instructions` steer emotion, pace and delivery; inline tags like [whispers], [excited], [giggles] inside `input`; WAV or MP3 URL usable as audio_url on video, or `stream: true` for raw audio as it is synthesized |  |
| `spicy-voice-2-flash` | speech | $0.30 per 10,000 characters of input text | input up to 5,000 characters; 9 preset voices; `instructions` steer emotion, pace and delivery; inline tags like [whispers], [excited], [giggles] inside `input`; WAV or MP3 URL usable as audio_url on video, or `stream: true` for raw audio as it is synthesized |  |
| `spicy-voice-1-expressive` | speech | $0.23 per 10,000 characters of input text | input up to 600 characters; 21 preset voices; `instructions` steer emotion, pace and delivery; WAV output, a URL usable as audio_url on video |  |
| `spicy-voice-1-custom` | speech | $0.23 per 10,000 characters of input text; creating a voice costs $0.40 (designed) or $0.02 (cloned), once | input up to 600 characters; speaks your designed and cloned voices (vc_...); WAV output, a URL usable as audio_url on video |  |
| `spicy-voice-1` | speech | $0.20 per 10,000 characters of input text | input up to 600 characters; 45 preset voices; WAV output, a URL usable as audio_url on video |  |
| `spicy-live-1` | realtime | per 1M tokens: text in $0.46, audio in $1.86, text out $1.4, audio out $3.74, billed per turn (about half a cent per minute of conversation; opening a session needs a $0.05 balance) | 28 voices, up to 13 minutes per session, POST /v1/realtime/sessions then a WebSocket |  |
| `spicy-transcribe-1` | transcription | $0.0042 per minute of audio, billed by the second (minimum $0.0001 per request) | wav, mp3, m4a, ogg, flac, webm and more, up to 10 MB and 5 minutes |  |
| `spicy-embed-1` | embedding | $0.14 per 1M tokens |  |  |
| `spicy-embed-vision-1` | embedding | $0.18 per 1M text tokens and $0.06 per 1M image tokens |  |  |

Public docs: https://www.spicyapi.com/docs (every page is also available as markdown by appending `.md`). Product overview for agents: https://www.spicyapi.com/llms.txt
