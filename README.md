# 用 Cloudflare Workers + Grok Bot 零成本搭建「真后台」微信 ClawBot 实战记录

> 本文记录把个人微信（已开通「微信 ClawBot」）接入 **Grok Bot** 的完整实战过程：不需要本地 Mac 长期开机，不需要内网穿透，也不需要在云端定时唤醒轮询消耗额度，而是通过 **Cloudflare Workers + Durable Objects + Grok Bot Webhook** 实现真正的事件驱动后台收发。

---

## 一、为什么选 Cloudflare Workers + Webhook 方案？

在对比了几种常见方案后，最终敲定了这条路线：

| 方案 | 是否需要 Mac 常开 | 空闲时额度消耗 | 适用场景 |
| --- | --- | --- | --- |
| 本地 Mac 跑 `wx-bot` + Tunnel | 需要 Mac 不休眠 | 低 | 仅桌面临时测试 |
| 云盒跑定时轮询（每 5–10 分钟） | 不需要 | 持续消耗大量唤醒额度 | 不推荐 |
| **Cloudflare Workers + Webhook（本文方案）** | **不需要** | **只有微信来消息时才触发** | **主力日常使用** |

核心组件选用开源项目 [`lawrencewzen/wx-bot`](https://github.com/lawrencewzen/wx-bot) 的 `packages/cloudflare`：
- 每个微信绑定对应 1 个 Cloudflare Durable Object（SQLite 存储收件箱与登录态，Free Plan 即可使用 `new_sqlite_classes`）。
- 后台 Alarm 按约 14 分钟一个切片长轮询微信 iLink 接口并自动续约。
- 收到消息后通过 Webhook 实时推送给 Grok Bot 的「微信消息」例行任务，处理完直接调 Worker 接口回传微信。

---

## 二、部署步骤总览

### 1. 准备 Worker 打包与 Cloudflare 部署

1. 克隆 `lawrencewzen/wx-bot` 并安装依赖：
   ```bash
   git clone https://github.com/lawrencewzen/wx-bot.git
   cd wx-bot
   bun install
   ```
2. 如需修复先配置 Webhook 再扫码时阻塞 14 分钟的问题，先应用本仓库中的补丁 [`patches/cloudflare-alarm-wakeup.patch`](./patches/cloudflare-alarm-wakeup.patch)（详见第三节踩坑记录）。
3. 配置随机 `API_TOKEN`（如 `openssl rand -hex 32`）并部署到 Cloudflare Workers（启用 Durable Object `WxBinding` 及 SQLite migration `v1`，并开启 `workers.dev` 路由）。
4. 验证基础健康检查与鉴权：
   - `GET /healthz` 返回 `200 ok`
   - 不带 Bearer Token 请求 `/b/<binding>/v1/status` 返回 `401`
   - 带 `Authorization: Bearer <API_TOKEN>` 请求 `/b/<binding>/v1/status` 返回 `200`

### 2. 在 Grok Bot 创建 Webhook 例行任务

1. 创建一个 **Webhook 触发** 的例行任务（例如命名为「微信消息」），提示词可直接使用 [`templates/grokbot-routine-prompt.md`](./templates/grokbot-routine-prompt.md)。
2. 在 Grok Bot 盒内将 Worker 地址、Token 与绑定名保存到 `~/.config/wx-bot/cf-worker.json`（权限设为 `600`）：
   ```json
   {
     "worker_url": "https://<your-worker>.workers.dev",
     "api_token": "<your-random-api-token>",
     "binding": "main"
   }
   ```
3. 获取该例行任务的 **Webhook URL** 和 **Webhook Key**（注意不要把 Key 粘贴到明文聊天或代码仓库中）。

### 3. 绑定 Webhook 并扫码登录微信

```bash
H="Authorization: Bearer <API_TOKEN>"
W="https://<your-worker>.workers.dev"

# 1. 写入 Webhook 投递配置
curl -X PUT -H "$H" -H "Content-Type: application/json" "$W/b/main/config" \
  -d '{"delivery":"webhook","webhook":{"url":"<WEBHOOK_URL>","key":"<WEBHOOK_KEY>"}}'

# 2. 获取微信登录二维码链接并渲染为 PNG 扫码
curl -X POST -H "$H" "$W/b/main/login"
```

用手机微信扫二维码并确认登录后，在微信里给「微信 ClawBot」发一句 `你好`，观察：
- `GET /b/main/v1/status` 中 `logged_in: true`、`cursor_set: true`、`delivery: "webhook"`、`inbox.pending: 0`；
- Grok Bot 例行任务自动触发并把回复发回微信。

---

## 三、实战中踩过的两个真实深坑（附解法）

### 坑 1：先配 Webhook 再扫码，Durable Object Alarm 被锁死 14 分钟

- **现象**：按照文档顺序先执行 `PUT /b/<binding>/config` 设置 `delivery: "webhook"`，再执行 `POST /b/<binding>/login` 生成二维码。扫码并在手机端确认登录、发消息后，机器人没有任何反应；反复查询 `GET /b/<binding>/login`，`status` 一直停在 `"waiting"` 且 `updated_at` 纹丝不动。
- **根因**：
  1. `PUT /config` 会调用 `setAlarm(Date.now())` 立即启动一次 `alarm()`。此时还没有 `login` 状态，所以 `loginStep()` 直接返回，而 `deliverLoop()` 会进入长达 `ALARM_SLICE_MS`（14 分钟）的信号等待循环。
  2. Cloudflare Durable Objects 同一时间只执行一个 `alarm()`。紧接着调用 `POST /login` 时，`startLogin()` 写入了新的 `K.login` 并调用 `setAlarm(Date.now())`，但并没有中止正在等待的 `this.pollAbort`，导致新的 `alarm()` 必须苦等 14 分钟后才能轮上执行——此时微信二维码早已超过 2 分钟有效期而失效。
  3. 并且如果直接重新部署 Worker，正在跑长等待的旧 DO 实例在未完成或未重启前仍可能继续持有旧版本行为。
- **解法**：
  - 在 `packages/cloudflare/src/index.ts` 的 `startLogin()` 和 `PUT /config` 中，调用 `setAlarm(Date.now())` 之前先执行 `this.pollAbort?.abort()`，立即打断空闲的长等待切片；同时在 `GET /login` 中检测若处于 `"waiting"` 且超过 5 秒未更新，主动唤醒并重新调度 Alarm。
  - 若已有旧绑定实例卡在未打断的长等待中，可先调用 `DELETE /b/<binding>/session` 或切换一个新 `<binding>` 名称重新绑定。
  - 完整最小补丁见 [`patches/cloudflare-alarm-wakeup.patch`](./patches/cloudflare-alarm-wakeup.patch)。

### 坑 2：例行任务回传微信时，Python `urllib` 默认 UA 被 Cloudflare WAF 403 拦截

- **现象**：微信发来消息后，Webhook 成功唤醒了 Grok Bot 例行任务，但任务内部用 Python `urllib.request` 请求 `https://<your-worker>.workers.dev/b/<binding>/v1/send` 时被 Cloudflare 边缘以 403 拦截。
- **根因**：Cloudflare 默认安全规则对 `Python-urllib/*` 这类默认 User-Agent 较敏感。
- **解法**：在例行任务提示词第 5 步明确要求使用 `curl`（或在请求头里显式指定自定义 `User-Agent`）。修改后首轮即稳定返回 `200` 并成功把回复发回微信。

---

## 四、已知风险与使用边界

1. **Cloudflare 免费额度余量**：按 `wx-bot` 文档估算，单个绑定 24 小时持续长轮询 Durable Object 约消耗 **10,800 GB-s/天**，Free Plan 额度为 **13,000 GB-s/天**（占约 83%）。单号运行一般够用，但同一账号下再增加 Durable Object 或第二个微信号容易触顶，届时需升级 Cloudflare Workers 付费计划。
2. **微信 ClawBot 24 小时回复窗口**：必须先由用户在微信端主动发消息，机器人才能在 24 小时内回复，无法由机器人主动定时向微信发起早报或提醒。
3. **腾讯出口 IP 风控与登录态过期**：Cloudflare 出口属于海外数据中心共享 IP，存在被腾讯限制或微信登录态周期性过期的可能；建议配合例行任务把 `alert` 异常转发到主对话，失效时及时重扫，或日后迁移至国内轻量服务器。
4. **单点轮询与纯文本限制**：同一绑定只能由一处轮询（不可与本机 CLI 同时跑同一会话）；若未挂载 R2 存储桶，图片和文件附件无法落地，建议在提示词中引导改用文字描述。
5. **与主窗口的上下文隔离**：微信消息由独立例行任务处理，默认通过长期记忆与本地历史文件（`chat_history.jsonl`）保持连贯，不会自动把主对话窗口的临时上下文同步到微信；如需互通，可在提示词中按需补充检索或摘要规则。
