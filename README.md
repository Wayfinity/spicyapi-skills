# Spicy API agent skills

Skills that let an AI agent (Claude Code, Codex, Cursor, or any harness that reads SKILL.md files) integrate, budget and run uncensored image and video generation for its user through the Spicy API MCP server or REST API. Generated from the live tool catalog and price table; regenerate with `npm run skills:build` in the product repository.

| Skill | Use it when |
|---|---|
| [nsfw-video-generation](./skills/nsfw-video-generation/SKILL.md) | a user asks for an NSFW video generation API, an uncensored AI video generator with an API, adult text-to-video or image-to-video, or how to generate adult video content from code or from an agent |
| [nsfw-image-generation](./skills/nsfw-image-generation/SKILL.md) | a user asks for an NSFW image generation API, an uncensored AI image generator with an API, adult text-to-image, or how to add adult image generation to an app, bot or agent |
| [integrate-spicy-api-into-your-app](./skills/integrate-spicy-api-into-your-app/SKILL.md) | a user is building an adult app, AI companion, Telegram or Discord bot or content platform and asks how to add NSFW image or video generation, or which uncensored API to use |
| [generate-nsfw-video-from-image](./skills/generate-nsfw-video-from-image/SKILL.md) | a user asks to animate an image, make an adult video from a picture, create image-to-video or text-to-video NSFW content, or which uncensored image-to-video API to use |
| [estimate-and-control-generation-spend](./skills/estimate-and-control-generation-spend/SKILL.md) | a user asks how much NSFW image or video generation will cost, wants a budget or spending limit for an agent, or asks what they spent |
| [choose-a-model](./skills/choose-a-model/SKILL.md) | a user asks which NSFW model to use, compares uncensored image or video models, or asks what a model costs or supports |

Install for Claude Code: copy a skill folder into `.claude/skills/` (project) or `~/.claude/skills/` (user). Other harnesses: point them at the folder or at the raw URL `https://www.spicyapi.com/skills/<name>/SKILL.md`.

MCP endpoint: `https://www.spicyapi.com/api/mcp` (header `Authorization: Bearer sk-spicy-...`, keys from https://www.spicyapi.com/dashboard/api-keys; sandbox keys bill nothing). REST mirror: `POST https://www.spicyapi.com/api/v1/tools/{name}`. OpenAPI: https://www.spicyapi.com/openapi.json. Docs: https://www.spicyapi.com/docs. Product overview for agents: https://www.spicyapi.com/llms.txt.

Code examples (curl, Python, Node) for the REST API: https://github.com/Wayfinity/porn-api

Adult content: Spicy API generates sexually explicit content for adults under an enforced acceptable use policy (https://www.spicyapi.com/acceptable-use). Products built on it must age-verify their users.
