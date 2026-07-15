---
name: wecom-bot-project
description: "User is porting their Discord AI bot to 企业微信 (WeCom) using the 智能机器人 API; prefers western tech stack, HK/SG hosting; a partner handles registration/ICP/domain"
metadata: 
  node_type: memory
  type: project
  originSessionId: 5d0fa054-25cd-4da0-8ef6-fdcb7f26da6e
---

As of July 2026, the user wants to build a WeCom (企业微信) equivalent of their Discord AI bot (Firebase Functions + slash command + channel history → LLM → reply). Research done 2026-07-15 concluded:

- The right primitive is WeCom's 智能机器人 (intelligent robot, launched ~2025), created in the admin console (管理后台 → 管理工具 → 智能机器人). Two API modes: HTTPS callback (encrypted XML/JSON, response_url valid 1h) and WebSocket long connection (`wss://openws.work.weixin.qq.com`, bot_id + secret, no public URL/domain/ICP needed). Recommended: long-connection mode to avoid GFW inbound issues with western serverless URLs.
- Biggest gap vs Discord: no API to fetch group chat history; the bot only receives messages that @mention it (plus quoted messages). Context must be self-accumulated in a DB, or use paid 会话存档 (verified enterprise, C/C++ Linux SDK).
- 智能机器人 is internal-only: works only with enterprise members in internal chats/groups; no WeChat personal users or external customer groups (official hard limit).
- Users must install the WeCom app and join the enterprise (unverified enterprise cap: 200 members; verification needs a Chinese business license 营业执照).
- Rate limits: 30 msgs/min, 1000/hour per conversation. Streaming replies supported.
- User is Chinese, fluent in Chinese, but prefers western dev stacks. A non-technical partner handles company registration, 企业认证, ICP filing, domain.
