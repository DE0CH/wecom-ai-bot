# wecom-ai-bot

An AI chat bot for 企业微信 (WeCom), modeled on [discord-ai-2](https://github.com/DE0CH/discord-ai-2): users summon the bot in a group chat, the backend builds a transcript from stored chat context, sends it to an LLM (DeepSeek/Qwen), and relays the reply back into the group.

## Planned architecture (v1)

- WeCom **智能机器人** (intelligent robot) in **长连接 (long-connection) mode**: the service opens an outbound WebSocket to `wss://openws.work.weixin.qq.com` — no public callback URL, no domain, no ICP 备案 needed.
- Single always-on Node.js/TypeScript process (Singapore/HK hosting) that:
  1. Subscribes with `bot_id` + `secret`, pings every ~30s, auto-reconnects.
  2. On `aibot_msg_callback` (group @mention): dedupes by `msgid`, stores the message, builds a tail-truncated transcript from its own DB (WeCom has no chat-history API).
  3. Calls an OpenAI-compatible Chinese model API and streams the reply back via `aibot_respond_msg` / `aibot_respond_update_msg`.

## Key platform constraints

- The bot only receives messages that @mention it (plus quoted messages) — context must be self-accumulated.
- Internal-only: all users must join the 企业微信 enterprise as members (200-member cap until 企业认证).
- Rate limits: 30 msgs/min, 1000/hour per conversation.

## memory/

Claude Code project memory, tracked in git. `~/.claude/projects/*/memory` for this project symlinks here.
