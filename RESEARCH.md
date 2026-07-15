# 企业微信 AI Bot — Research Report

*Researched 2026-07-15, porting the architecture of [discord-ai-2](https://github.com/DE0CH/discord-ai-2) (Firebase Functions + slash command → fetch channel history → LLM → relay reply) to 企业微信 (WeCom).*

*Citations: every externally sourced claim carries a footnote with a verbatim quote from the source, verified by fetching each source on 2026-07-15. Two sources consulted in the original research (知乎, CSDN articles) now block automated access (HTTP 403/521) and have been replaced with accessible equivalents. Claims marked **(inference)** are the author's analysis and carry no external source.*

## TL;DR

企业微信 launched an official **智能机器人** (intelligent robot) API around 2025 that maps surprisingly well onto the Discord interactions model — including a `response_url` mechanism almost identical to Discord's deferred interaction webhook[^resp-url] and native streaming replies.[^stream-id] It can be built in TypeScript/Node on a Singapore/HK server with **no domain and no ICP 备案 at all** if using the WebSocket "长连接" (long-connection) mode, because that mode needs no inbound URL and no payload crypto.[^ws-endpoint][^no-crypto]

Two hard platform constraints to design around:

1. **There is no API to fetch chat history.** Unlike Discord's `GET /channels/{id}/messages`, the bot only *receives* messages pushed to it — group messages that @mention it and direct single-chat messages[^receive-trigger] (plus the quoted message when someone quote-replies[^quote-field]). No history-read endpoint appears anywhere in the 智能机器人 docs — the 接收消息 documentation defines push callbacks only (verified by absence, 2026-07-15). History must be accumulated in our own database, and it will only ever contain what the bot was sent.
2. **The bot only works with internal enterprise members.** Everyone who wants to talk to it must install the 企业微信 app and join the 企业 as a member. It cannot be added to 外部群, and personal WeChat users get no response[^internal-only] — a hard official restriction whose only known workarounds (iPad-protocol clients, PC hooks) are non-official and risk account bans.[^workarounds]

## How the pieces map from Discord

*(The mapping itself is the author's analysis; each 企业微信-side fact is sourced.)*

| Discord concept | 企业微信 equivalent |
|---|---|
| Discord application/bot | 智能机器人, created in 管理后台 (admin console) → 安全与管理 → 管理工具 → 智能机器人[^create-path] |
| Slash command `/ai` | @mentioning the bot in a group, or messaging it in 单聊 (no slash-command registry; the @mention *is* the summon)[^receive-trigger] |
| Interactions endpoint (webhook) | Either a callback URL mode[^callback-mode], or WebSocket long-connection mode[^ws-endpoint] |
| Ed25519 signature verification | Callback mode: encrypted payloads (AES-256-CBC, same AESKey scheme as media)[^callback-crypto][^media-aeskey]; long-connection mode: no message crypto (auth is bot_id + secret at subscribe time)[^no-crypto][^subscribe] |
| Deferred response + interaction webhook (15 min) | `response_url` per incoming message — valid **1 hour**, single use[^resp-url]; or reply frames over the WebSocket[^respond-frames] |
| Fetch channel history via REST | **Does not exist** — see below |
| Message content | Replies as text, markdown, 模板卡片 (template cards), 图文混排 (currently image)[^msg-types]; **streaming supported** (send incremental frames with a `stream.id`, `finish=true` to end[^stream-id]; a stream times out **6 minutes** after its first frame[^stream-timeout]) |

Rate limits: 30 messages/minute and 1000/hour (documented for proactive push in the long-connection docs).[^rate-limit]

## The two API modes, and which to pick

**Callback URL mode** is the Discord architecture almost verbatim: 企业微信 POSTs encrypted payloads to your HTTPS URL[^callback-mode][^callback-crypto], you verify/decrypt, reply inline or later via `response_url`.[^resp-url] It would slot into the Firebase Functions setup directly. **But** **(inference)**: the callbacks originate from Tencent servers in mainland China, and a `*.cloudfunctions.net` / `*.run.app` (Google) URL is blocked or unreliable across the GFW. You'd need a custom domain on infrastructure reachable from the mainland, which pulls the domain/ICP question back in.

**长连接 (long-connection) mode — recommended for the MVP.** The server opens an outbound WebSocket to `wss://openws.work.weixin.qq.com`[^ws-endpoint], authenticates with an `aibot_subscribe` frame carrying the bot's `bot_id` + `secret`[^subscribe], sends a heartbeat every ~30s[^heartbeat], and receives `aibot_msg_callback` frames when someone messages the bot.[^msg-callback] Replies go back as `aibot_respond_msg` frames over the same socket (streaming/card updates via `aibot_respond_update_msg`).[^respond-frames] Because the connection is outbound:

- No public URL, no domain, no ICP 备案, no TLS cert to manage, no payload encryption to implement[^no-crypto] (only media attachments come with a per-resource `aeskey`[^media-aeskey]).
- Works fine from Singapore or Hong Kong **(inference:** outbound connections *to* mainland services are generally not blocked**)**.
- One active connection per bot — a new subscribe kicks the old connection, so exactly one instance.[^single-conn]

Architectural change vs the Discord bot **(inference)**: a persistent WebSocket means an **always-on process, not serverless functions**. Firebase Functions won't work here. A tiny VPS or container host in Singapore/HK (Fly.io `sin`/`hkg`, Railway Singapore, or a small Lightsail/EC2/Tencent Cloud HK box) running a single Node.js process is the right shape. Persist chat logs somewhere simple — SQLite on a volume, or Firestore (the *server's* outbound calls to Google APIs from SG are fine; only inbound-from-mainland is the problem).

There's also `aibot_send_msg` for proactive/unprompted messages (e.g., scheduled digests) — allowed once a user has previously messaged the bot in that conversation.[^proactive]

Duplicate delivery is expected — deduplicate by the callback's unique id.[^msgid]

## The chat-history problem (biggest design difference)

Discord's core trick — pull the last 100 channel messages on demand — has no equivalent. Options, in order of practicality:

1. **Self-accumulate (recommended for MVP).** Store every `aibot_msg_callback` received (message text, sender, chatid, timestamp, quoted content) plus every reply the bot sends. Per-`chatid` this builds a transcript of *bot interactions* — not the full group chatter, but often enough, since people summon the bot when they want it involved. Quote-reply helps: when a user quote-replies while mentioning the bot, the callback includes the quoted message[^quote-field], so users can manually pull specific context in.
2. **会话内容存档 (Conversation Content Archive).** The only official way to get *full* conversation history: internal conversations are retrievable via API after employees are notified (they see a mandatory notice page at login), and external-contact conversations additionally require the contact's consent.[^archive-consent] It is a paid per-account add-on — tiered at roughly 办公版/服务版/企业版 price points (sources list 200/450/900 元/账号/年 and older 100/300/600 figures; confirm current pricing in the console)[^archive-price] — and multiple guides report it requires a 认证 (verified) enterprise (the official prerequisite page is JS-gated and could not be re-verified; treat as likely but unconfirmed). The SDK is a C-style library shipped for Linux (x86/ARM) and Windows that you poll with a `seq` cursor.[^archive-sdk] Significant integration and cost — skip for v1; revisit only if "bot sees everything" turns out to matter.
3. Anything based on iPad-protocol clients or PC hooks to read messages as a fake user — non-official, and gets accounts banned; not a path to build on.[^workarounds]

Design consequence **(inference)**: prompt users to @bot *with their question in the same message*, and treat the stored per-chat log as the "transcript" fed to the model — the same tail-truncation logic from the Discord bot (`buildDiscordTranscript`) transfers directly.

## Model backend

From an SG/HK server, the major Chinese model APIs are reachable and OpenAI-compatible:

- **DeepSeek** — base URL `https://api.deepseek.com`, API format compatible with OpenAI/Anthropic SDKs.[^deepseek-api] ⚠️ The familiar model names are on their way out: `deepseek-chat` and `deepseek-reasoner` are **deprecated on 2026-07-24** and map to the non-thinking/thinking modes of `deepseek-v4-flash` — target the v4-flash/v4-pro names directly.[^deepseek-models] Registration is by phone or email; 实名认证 affects API quota and paid-call permissions, and top-up uses 支付宝/微信 local payment methods[^deepseek-billing] — one of the things to hand to the partner (or do yourself, since you both have mainland credentials).
- **Qwen (阿里云百炼 / Model Studio)** — OpenAI-compatible interface for Qwen models, with a **Singapore endpoint**: `https://{WorkspaceId}.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` (the legacy `dashscope-intl.aliyuncs.com` domain is being discontinued — use the new form).[^qwen-endpoint] Billed through an Alibaba Cloud International account **(inference:** which avoids mainland-payment dependencies**)**.
- Zhipu GLM, Moonshot Kimi — similar OpenAI-compatible setups **(not re-verified)**.

Regulatory angle: China's 《生成式人工智能服务管理暂行办法》 applies to services that offer generated content **to the domestic public** — and it *explicitly exempts* organizations that develop/use GenAI **without** offering it to the public[^genai-scope], which is exactly what an internal-members-only enterprise bot is. Content-compliance obligations (e.g. Article 4) bind public-facing providers[^genai-content]; using a domestic model keeps you aligned with them anyway **(inference)**. Note Anthropic does not list mainland China (nor Hong Kong; Taiwan yes) among supported regions for API access[^anthropic-regions], so routing to Claude from a mainland-user-facing product is off the table regardless.

## What to ask the non-technical partner for (checklist)

**Needed before writing any code:**

1. **注册企业微信** (register a WeCom enterprise) at work.weixin.qq.com — registration asks for the admin's phone number and email[^register]; supplying a business license is an optional "认领" step (raising the member cap to 1000)[^register], not a registration requirement. An unverified (未认证) enterprise is capped at **200 members** and lacks external-contact features — fine for the MVP.[^unverified-cap]
2. **Admin access to the 管理后台**: 管理后台 → 安全与管理 → 管理工具 → 智能机器人 → 创建机器人[^create-path] — set name, avatar, 可见范围 (visibility scope)[^visibility], enable **API模式**[^api-mode], choose **长连接** (the two API sub-modes are mutually exclusive; switching to callback mode kills existing long connections)[^mode-exclusive], and copy the **`Bot ID` + `Secret`** from the bot's detail page.[^credentials] (Two-person 企业: just get full admin — see CLAUDE.md.)
3. **Invite users as enterprise members.** Everyone must install the 企业微信 app (not personal WeChat) and join — external/personal-WeChat users are blocked at the platform level.[^internal-only] Then anyone creates an internal group and adds the bot.
4. **LLM API account + billing** — DeepSeek (mainland payments)[^deepseek-billing] or Qwen via Alibaba Cloud International.[^qwen-endpoint]

**Not needed now — but know *when* it becomes needed:**

5. **企业认证 (enterprise verification)** — audit fee for small enterprises (≤1000 people) is 300元/年[^verify-fee]; guides consistently list 营业执照 + 法人身份证 as the required materials, plus an authorization letter if the admin isn't the legal rep **(reported by multiple guides; not re-verified verbatim)**. One business license can verify at most 5 WeCom enterprises, and one national ID can be the creator of at most 5.[^license-limit] Triggered when you need: >200 members, external-contact/customer features, or 会话存档.
6. **域名 + ICP备案 (domain + ICP filing)** — **not needed at all for the long-connection MVP.** Relevant only when (a) hosting moves into mainland China — services resolving to mainland servers must complete ICP 备案 before serving traffic, and the filing goes through the hosting provider (un-filed domains pointing at mainland servers get blocked)[^icp] — or (b) building web pages inside 企业微信 later (JS-SDK 可信域名 requires an ICP-filed domain — **not re-verified**). The dependency chain **(inference)** is *company → domain at a Chinese registrar → ICP via the cloud provider*, so the company registration is the long pole. Don't spend money on this yet.
7. **Server**: nothing to ask — spin up a Singapore/HK box under your own account **(inference)**.

## Suggested v1 architecture

*(Author's design, following from the sourced constraints above.)*

```
Node.js (TS) on Fly.io/Railway/VPS in SIN or HKG
 ├─ WebSocket client → wss://openws.work.weixin.qq.com
 │    aibot_subscribe(bot_id, secret) · heartbeat every 30s · auto-reconnect
 ├─ on aibot_msg_callback (group @mention or 单聊):
 │    1. dedupe by callback id, store message in DB (SQLite/Firestore)
 │    2. load recent transcript for this chatid, tail-truncate
 │    3. call DeepSeek/Qwen (OpenAI-compatible chat completions, stream: true)
 │    4. stream reply back via aibot_respond_msg (stream.id, finish=true within 6 min)
 │       (or one-shot markdown reply to keep v1 simple)
 └─ store the bot's own replies in the same DB
```

The transcript-formatting and truncation code from discord-ai-2 (`formatDiscordMessage` / `buildDiscordTranscript` in `functions/src/index.ts`) ports over almost unchanged — fed from our own DB instead of Discord's REST API.

## Open question

Will the intended users accept installing the 企业微信 app? If the real audience is on personal WeChat, this approach doesn't reach them — the alternative would be a 微信公众号 (Official Account) bot, a different (also viable) architecture.

## References

[^create-path]: [LangBot 文档 — 企业微信智能机器人](https://docs.langbot.app/zh/usage/platforms/wecom/wecombot): 「点击左边栏的`安全与管理`，点击`管理工具`里面的`智能机器人`」

[^api-mode]: [LangBot 文档 — 企业微信智能机器人](https://docs.langbot.app/zh/usage/platforms/wecom/wecombot): 「在企业微信管理后台，进入智能机器人的配置页面，开启「API 模式」」

[^mode-exclusive]: [LangBot 文档 — 企业微信智能机器人](https://docs.langbot.app/zh/usage/platforms/wecom/wecombot): 「API 模式只能选择一种方式（长连接或设置接收消息回调地址），切换到「设置接收消息回调地址」模式会导致现有长连接失效」

[^credentials]: [LangBot 文档 — 企业微信智能机器人](https://docs.langbot.app/zh/usage/platforms/wecom/wecombot): 「开启长连接 API 模式后，需要获取以下凭证用于建立连接：BotID、Secret」；「复制企业微信机器人详情页的`Bot ID`」。另见 [CodeBuddy — 企业微信智能机器人接入指南](https://www.codebuddy.ai/docs/zh/cli/wecom-bot-setup): 「将连接方式选择为「使用长连接」」…「Bot ID：机器人的唯一标识…Secret：点击「获取」」

[^visibility]: [LangBot 文档 — 企业微信智能机器人](https://docs.langbot.app/zh/usage/platforms/wecom/wecombot): 「注意设置可见范围」

[^callback-mode]: [智能机器人概述 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101039): 「若智能机器人开启API模式，当用户跟智能机器人交互时，企业微信会向智能机器人API设置中的URL的回调地址推送相关消息跟事件。」

[^callback-crypto]: [接收消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/100719): 「接收消息与被动回复消息都是加密的，加密方式参考[回调和回复的加解密方案]」

[^receive-trigger]: [接收消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/100719): 「用户群里@智能机器人或者单聊中向智能机器人发送文本消息」

[^quote-field]: [接收消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/100719): 「若用户引用了其他消息则有该字段，可参考 [引用] 结构体说明」

[^media-aeskey]: [接收消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/100719): 「注意获取到的文件是已加密的，不能直接打开。加密AESKey与[回调加解密的AESKey]相同。加密方式：AES-256-CBC」

[^msgid]: [接收消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/100719): 「本次回调的唯一性标志，开发者需据此进行事件排重（可能因为网络等原因重复回调）」

[^resp-url]: [主动回复消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101138): 「该 response_url 有效期为 1 个小时，超过有效期将无法使用」；「每个 response_url 用户可以调用接口一次」。字段定义见[接收消息](https://developer.work.weixin.qq.com/document/path/100719): 「response_url — 支持主动回复消息的临时url」

[^stream-id]: [被动回复消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101031): stream id — 「自定义的唯一id，某次回调的首次回复第一次设置，后续回调会根据这个id来获取最新的流式消息」；finish — 「流式消息是否结束」

[^stream-timeout]: [CodeBuddy — 企业微信智能机器人接入指南](https://www.codebuddy.ai/docs/zh/cli/wecom-bot-setup): 「流式消息超时：6 分钟（从首次发送 stream 开始计时）」

[^msg-types]: [被动回复消息 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101031): 「流式消息内容，最长不超过20480个字节，必须是utf8编码」；「图文混排消息类型，目前支持：image」；template_card — 「msgtype此时固定为template_card」。markdown/模板卡片回复类型见[主动回复消息](https://developer.work.weixin.qq.com/document/path/101138)消息类型表（`"msgtype": "markdown"`、`"msgtype": "template_card"`）

[^rate-limit]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 主动推送频率限制 「30 条/分钟，1000 条/小时」

[^ws-endpoint]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 长连接地址 `wss://openws.work.weixin.qq.com`

[^subscribe]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 订阅帧 `"cmd": "aibot_subscribe"`，`"body": {"bot_id": "BOTID", "secret": "SECRET"}`

[^heartbeat]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 「建议每 30 秒 发送一次心跳」

[^msg-callback]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): `"cmd": "aibot_msg_callback"` — 用户向智能机器人发送消息时推送

[^respond-frames]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 回复用 `"cmd": "aibot_respond_msg"`（流式消息带 `"stream": {"id": "STREAMID", "finish": false}`）；更新用 `"cmd": "aibot_respond_update_msg"`

[^single-conn]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 「每个智能机器人同一时间只能保持一个有效的长连接。当同一个机器人发起新的连接请求并完成订阅时，新连接会踢掉旧连接」

[^no-crypto]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): 长连接模式特性对比 「无需加解密」；多媒体资源「会额外返回解密密钥 `aeskey`」

[^proactive]: [智能机器人长连接 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/101463): `"cmd": "aibot_send_msg"`；条件 「需要用户在会话中给机器人发消息，后续机器人才能主动推送消息给对应会话中」

[^internal-only]: [腾讯云开发者社区 — 官方机器人限制说明](https://cloud.tencent.com/developer/article/2684021): 「完全不支持外部场景：不能进外部群、不能私聊微信个人、外部用户@机器人无任何回调响应」；「仅支持企业员工私聊、内部群@对话」；「识别逻辑只匹配企业通讯录成员，外部微信用户直接被官方拦截」

[^workarounds]: [腾讯云开发者社区 — 官方机器人限制说明](https://cloud.tencent.com/developer/article/2684021): 「目前只有iPad协议、PC Hook注入两种可行方案」— 文中明确二者为非官方原生能力，需遵守风控规则以避免封号。

[^archive-consent]: [会话内容存档·使用前帮助 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/91361): 「开启存档的员工登录客户端，进入企业后会经过告知页面，获知后可继续使用」；「员工在企业内的会话内容，经告知员工后，企业可通过API获取」；「员工与外部联系人的会话内容，经外部联系人同意后，企业可通过API获取」

[^archive-sdk]: [会话内容存档·获取会话内容 — 企业微信开发者中心](https://developer.work.weixin.qq.com/document/path/91774): SDK 下载提供 linux 环境（x86/arm 服务器）与 windows 环境版本；C 风格接口（NewSdk/Init/GetChatData/DecryptData）；「从指定的seq开始拉取消息，注意的是返回的消息从seq+1开始返回…首次使用请使用seq:0」；「一次拉取调用上限1000条会话记录」

[^archive-price]: [群应用SCRM — 企业微信会话存档收费标准](https://www.wescrm.com/siyuzhishiku/siyugongju/4592.html): 办公版「200元/账号/年」（仅限金融行业）、服务版「450元/账号/年」、企业版「900元/账号/年」；「账号单价×数量×使用时间（年）=总价格」。另一来源（[企业微信指南](https://weibanzhushou.com/blog/8406)）列出 100/300/600 元档位并载明开通路径：「管理员进入企业微信管理后台，在【管理工具】-【会话内容存档】中选择需要开通的类型，并缴纳费用」— 两处价目不一致，以管理后台实际报价为准。

[^register]: [企业微信注册/验证手册](https://open.work.weixin.qq.com/wwopen/manual/detail?t=register): 注册流程「输入管理员的手机号码及企业邮箱账号」；补充营业执照为「认领企业」的可选步骤，「认领后可获得1000人企业人数上限」。

[^unverified-cap]: [未认证企业的功能限制 — 群应用SCRM](https://www.wescrm.com/siyuzhishiku/qiweiyunying/40104.html): 「未认证企业最多只能加 200 名成员，超过就得删老员工才能加新人」；外部功能限制：「未认证企业最多只能加 200 个外部客户，超过就加不上；不能创建 '客户群'（或群人数上限仅 20 人）」

[^verify-fee]: [简鹿办公 — 微信/企业微信认证费用汇总](https://www.jianlu365.com/info/4305.html): 「小型企业 ≤1,000人 300元/年」

[^license-limit]: [群应用SCRM — 一张营业执照能注册多少个企业微信账号](https://www.wescrm.com/siyuzhishiku/siyugongju/6267.html): 「一张营业执照（同一主体），只能认证5个以内的企业微信账号」；「一张身份证仅支持作为5个企业微信账号的创建人。包括认证版和免认证版。」

[^deepseek-api]: [DeepSeek API Docs](https://api-docs.deepseek.com/): "The DeepSeek API uses an API format compatible with OpenAI/Anthropic."; "you can use the OpenAI/Anthropic SDK or softwares compatible with the OpenAI/Anthropic API"; base_url (OpenAI): `https://api.deepseek.com`

[^deepseek-models]: [DeepSeek API Docs](https://api-docs.deepseek.com/): "`deepseek-chat` (to be deprecated on 2026/07/24)"; "`deepseek-reasoner` (to be deprecated on 2026/07/24)"; "they correspond to the non-thinking mode and thinking mode of `deepseek-v4-flash`, respectively."

[^deepseek-billing]: [API知识站 — DeepSeek 购买教程](https://www.apiuspro.cn/tutorial/deepseek): 充值「可使用支付宝、微信等本地支付方式」；实名认证「认证会影响 API 配额和后续付费调用权限」；注册「用手机号或邮箱完成账号注册和登录」

[^qwen-endpoint]: [Alibaba Cloud Model Studio — Call Qwen models via OpenAI API](https://www.alibabacloud.com/help/en/model-studio/compatibility-of-openai-with-dashscope): "Model Studio provides an OpenAI-compatible interface for Qwen models."; "Singapore: https://{WorkspaceId}.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1"; migration notice: "Singapore: from `https://dashscope-intl.aliyuncs.com` to `https://{WorkspaceId}.ap-southeast-1.maas.aliyuncs.com`"

[^genai-scope]: [《生成式人工智能服务管理暂行办法》 — 国家网信办](https://www.cac.gov.cn/2023-07/13/c_1690898327029107.htm) 第二条: 适用于「利用生成式人工智能技术向中华人民共和国境内公众提供生成文本、图片、音频、视频等内容的服务」；豁免条款：「行业组织、企业、教育和科研机构、公共文化机构、有关专业机构等研发、应用生成式人工智能技术，未向境内公众提供生成式人工智能服务的，不适用本办法的规定」

[^genai-content]: [《生成式人工智能服务管理暂行办法》 — 国家网信办](https://www.cac.gov.cn/2023-07/13/c_1690898327029107.htm) 第四条: 「坚持社会主义核心价值观，不得生成煽动颠覆国家政权、推翻社会主义制度」等禁止性内容要求。

[^anthropic-regions]: [Anthropic — Supported Countries and Regions](https://www.anthropic.com/supported-countries): "Countries, regions, and territories where we currently offer commercial API access" — mainland China does not appear on the API or Claude.ai lists (Hong Kong also absent; Taiwan listed). Checked 2026-07-15.

[^icp]: [阿里云帮助中心 — 什么是ICP备案](https://help.aliyun.com/zh/icp-filing/basic-icp-service/product-overview/what-is-an-icp-filing): 「解析至中国内地服务器的网站、App等服务，必须完成ICP备案才可对外提供服务。」；「如果您未在阿里云提交过ICP备案，直接将域名解析至阿里云中国内地服务器上，将被阿里云监测系统识别并阻断」
