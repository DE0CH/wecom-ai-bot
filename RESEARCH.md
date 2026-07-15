# 企业微信 AI Bot — Research Report

*Researched 2026-07-15, porting the architecture of [discord-ai-2](https://github.com/DE0CH/discord-ai-2) (Firebase Functions + slash command → fetch channel history → LLM → relay reply) to 企业微信 (WeCom).*

## TL;DR

企业微信 launched an official **智能机器人** (intelligent robot) API around 2025 that maps surprisingly well onto the Discord interactions model — including a `response_url` mechanism almost identical to Discord's deferred interaction webhook, and native streaming replies. It can be built in TypeScript/Node on a Singapore/HK server with **no domain and no ICP 备案 at all** if using the WebSocket "长连接" (long-connection) mode.

Two hard platform constraints to design around:

1. **There is no API to fetch chat history.** Unlike Discord's `GET /channels/{id}/messages`, the bot only *receives* messages that @mention it (plus the quoted message when someone quote-replies). History must be accumulated in our own database, and it will only ever contain what the bot was mentioned in.
2. **The bot only works with internal enterprise members.** Everyone who wants to talk to it must install the 企业微信 app and join the 企业 as a member. It cannot be added to 外部群 (groups containing personal WeChat users), and personal WeChat users get no response — a hard official restriction with no compliant workaround.

## How the pieces map from Discord

| Discord concept | 企业微信 equivalent |
|---|---|
| Discord application/bot | 智能机器人, created in 管理后台 (admin console) → 管理工具 → 智能机器人 |
| Slash command `/ai` | @mentioning the bot in a group (no slash-command registry; the @mention *is* the summon) |
| Interactions endpoint (webhook) | Either a callback URL mode, or WebSocket long-connection mode |
| Ed25519 signature verification | Callback mode: AES-256-CBC encrypted payloads with Token + EncodingAESKey; long-connection mode: no message crypto needed (auth is bot_id + secret at subscribe time) |
| Deferred response + interaction webhook (15 min) | `response_url` per incoming message — valid **1 hour**, single use; or reply frames over the WebSocket |
| Fetch channel history via REST | **Does not exist** — see below |
| Message content | Text, image, mixed (图文混排), voice, file, video; replies as markdown or 模板卡片 (template cards); **streaming supported** (send incremental frames with a `stream.id`, `finish=true` to end, refreshable up to ~6 min) |

Rate limits: 30 messages/minute and 1000/hour per conversation.

## The two API modes, and which to pick

**Callback URL mode** is the Discord architecture almost verbatim: 企业微信 POSTs encrypted payloads to your HTTPS URL, you verify/decrypt, reply inline or later via `response_url`. It would slot into the Firebase Functions setup directly. **But**: the callbacks originate from Tencent servers in mainland China, and a `*.cloudfunctions.net` / `*.run.app` (Google) URL is blocked or unreliable across the GFW. You'd need a custom domain on infrastructure reachable from the mainland, which pulls the domain/ICP question back in.

**长连接 (long-connection) mode — recommended for the MVP.** The server opens an outbound WebSocket to `wss://openws.work.weixin.qq.com`, authenticates with an `aibot_subscribe` frame carrying the bot's `bot_id` + `secret`, sends a `ping` every ~30s, and receives `aibot_msg_callback` frames when someone @mentions the bot. Replies go back as `aibot_respond_msg` frames over the same socket (streaming updates via `aibot_respond_update_msg`). Because the connection is outbound:

- No public URL, no domain, no ICP 备案, no TLS cert to manage.
- Works fine from Singapore or Hong Kong (outbound connections *to* mainland services are generally not blocked).
- No payload encryption to implement (only media attachments come with a per-resource `aeskey`).
- One active connection per bot (a new subscribe kicks the old one — so exactly one instance).

Architectural change vs the Discord bot: a persistent WebSocket means an **always-on process, not serverless functions**. Firebase Functions won't work here. A tiny VPS or container host in Singapore/HK (Fly.io `sin`/`hkg`, Railway Singapore, or a small Lightsail/EC2/Tencent Cloud HK box) running a single Node.js process is the right shape. Persist chat logs somewhere simple — SQLite on a volume, or Firestore (the *server's* outbound calls to Google APIs from SG are fine; only inbound-from-mainland is the problem).

There's also `aibot_send_msg` for proactive/unprompted messages (e.g., scheduled digests) — allowed once a user has previously messaged the bot in that conversation.

## The chat-history problem (biggest design difference)

Discord's core trick — pull the last 100 channel messages on demand — has no equivalent. Options, in order of practicality:

1. **Self-accumulate (recommended for MVP).** Store every `aibot_msg_callback` received (message text, sender, chatid, timestamp, quoted content) plus every reply the bot sends. Per-`chatid` this builds a transcript of *bot interactions* — not the full group chatter, but often enough, since people summon the bot when they want it involved. Quote-reply helps: when a user quote-replies while mentioning the bot, the callback includes the quoted message, so users can manually pull specific context in.
2. **会话内容存档 (Conversation Content Archive).** The only official way to get *full* group history. Paid per-seat add-on requiring a **认证 (verified) enterprise**; employees are notified/must consent; the SDK is a C/C++ Linux library you poll — significant integration and cost. Skip for v1; revisit only if "bot sees everything" turns out to matter.
3. Anything based on iPad-protocol clients or PC hooks to read messages as a fake user — violates Tencent's ToS and gets accounts banned; not a path to build on.

Design consequence: prompt users to @bot *with their question in the same message*, and treat the stored per-chat log as the "transcript" fed to the model — the same tail-truncation logic from the Discord bot (`buildDiscordTranscript`) transfers directly.

## Model backend

From an SG/HK server, all the major Chinese model APIs are reachable and OpenAI-compatible:

- **DeepSeek** — `https://api.deepseek.com` (OpenAI-compatible), models `deepseek-chat` (V3-series) and `deepseek-reasoner` (R1-series). Very cheap. Billing on the DeepSeek platform is oriented around Chinese payment methods (微信支付/支付宝) — one of the things to hand to the non-technical partner.
- **Qwen (阿里云百炼 / Model Studio)** — has an **international Singapore endpoint** through Alibaba Cloud International, which accepts international cards. Lowest-friction option to avoid depending on the partner for billing.
- Zhipu GLM, Moonshot Kimi — also fine, similar OpenAI-compatible setups.

A non-obvious reason to prefer a Chinese model over routing to Anthropic/OpenAI: China's generative-AI regulations (《生成式人工智能服务管理暂行办法》). An internal-only enterprise tool is much lower-risk than a public service, but domestic models come with built-in content compliance — and Anthropic/OpenAI don't permit serving users in mainland China under their ToS anyway.

## What to ask the non-technical partner for (checklist)

**Needed before writing any code:**

1. **注册企业微信** (register a WeCom enterprise) at work.weixin.qq.com — needs a mainland phone number for the admin; a business license is *not* required just to register. An unverified (未认证) enterprise is capped at **200 members** and lacks external-contact features — fine for the MVP.
2. **Admin access to the 管理后台** (or have them do this one thing): 管理后台 → 管理工具 → 智能机器人 → 创建机器人 — set name, avatar, 可见范围 (visibility scope), enable **API模式**, choose **长连接**, and hand over the **`bot_id` + `secret`** (and Token/EncodingAESKey if callback mode is ever tried). Realistically the developer wants admin access to iterate.
3. **Invite users as enterprise members.** Everyone must install the 企业微信 app (not personal WeChat) and join. Then anyone creates an internal group and adds the bot.
4. **DeepSeek API account + billing**, if going with DeepSeek: register at platform.deepseek.com and top up (Chinese payment methods). Skip if using Qwen via Alibaba Cloud International (foreign card works).

**Not needed now — but know *when* it becomes needed:**

5. **企业认证 (enterprise verification)** — requires a 营业执照 (Chinese business license), 法人身份证 (legal representative's ID), plus an authorization letter if the admin isn't the legal rep; typically a small annual audit fee (~¥300). Triggered when you need: >200 members, external-contact/customer features, or 会话存档. One business license can verify up to 5 WeCom enterprises.
6. **域名 + ICP备案 (domain + ICP filing)** — **not needed at all for the long-connection MVP.** Relevant only when (a) hosting moves into mainland China — mainland hosting legally requires ICP filing, which requires the registered company, a domain bought/transferred through a Chinese registrar with real-name verification, and a few weeks of processing through the hosting provider — or (b) building web pages inside 企业微信 later (JS-SDK / 可信域名 requires an ICP-filed, ownership-verified domain). Don't spend money on this yet, but the dependency chain is *company → domain at Chinese registrar → ICP via the cloud provider*, so the company registration is the long pole.
7. **Server**: nothing to ask — spin up a Singapore/HK box under your own account. For eventual mainland hosting, the company + ICP from item 6 apply (mainland cloud accounts also need 实名认证, ideally under the company).

## Suggested v1 architecture

```
Node.js (TS) on Fly.io/Railway/VPS in SIN or HKG
 ├─ WebSocket client → wss://openws.work.weixin.qq.com
 │    aibot_subscribe(bot_id, secret) · ping every 30s · auto-reconnect
 ├─ on aibot_msg_callback (group @mention):
 │    1. dedupe by msgid, store message in DB (SQLite/Firestore)
 │    2. load recent transcript for this chatid, tail-truncate
 │    3. call DeepSeek/Qwen (OpenAI-compatible chat completions, stream: true)
 │    4. stream reply back via aibot_respond_msg / aibot_respond_update_msg
 │       (or one-shot markdown reply to keep v1 simple)
 └─ store the bot's own replies in the same DB
```

The transcript-formatting and truncation code from discord-ai-2 (`formatDiscordMessage` / `buildDiscordTranscript` in `functions/src/index.ts`) ports over almost unchanged — fed from our own DB instead of Discord's REST API.

## Open question

Will the intended users accept installing the 企业微信 app? If the real audience is on personal WeChat, this approach doesn't reach them — the alternative would be a 微信公众号 (Official Account) bot, a different (also viable) architecture.

## Sources

- [智能机器人概述](https://developer.work.weixin.qq.com/document/path/101039)
- [接收消息](https://developer.work.weixin.qq.com/document/path/100719)
- [主动回复消息 (response_url)](https://developer.work.weixin.qq.com/document/path/101138)
- [智能机器人长连接](https://developer.work.weixin.qq.com/document/path/101463)
- [API模式机器人说明](https://developer.work.weixin.qq.com/document/path/101468)
- [智能机器人使用指南 (知乎)](https://zhuanlan.zhihu.com/p/1946175495847781314)
- [群聊智能机器人介绍 (CSDN)](https://blog.csdn.net/zhanyd/article/details/145772183)
- [官方机器人不支持外部群/个微的说明](https://cloud.tencent.com/developer/article/2684021)
- [企业微信注册/认证流程 (知乎)](https://zhuanlan.zhihu.com/p/10584327490)
- [未认证企业的功能限制](https://www.wescrm.com/siyuzhishiku/qiweiyunying/40104.html)
- [注册/认证上限](https://open.work.weixin.qq.com/wwopen/manual/detail?t=register)
