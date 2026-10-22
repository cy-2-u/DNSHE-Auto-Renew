# DNSHE 域名自动续期助手

本项目是一个基于 GitHub Actions 的自动化脚本，旨在利用 **DNSHE 免费域名 API** 实现子域名的自动续期，并通过 **MeoW / PushPlus / Telegram Bot** 推送执行结果，确保您的免费域名永不过期。

## 功能特性

- **全自动续期**：每月 1 日自动执行续期操作。

- **多域名支持**：自动遍历账户下所有子域名进行批量续期。

- **即时通知**：默认推荐 MeoW，也可并用 PushPlus / Telegram Bot；结果逐行显示。

- **外部触发**：仓库自带 schedule 已关闭，由 cron-job.org 每月调用 `workflow_dispatch`。

- **安全合规**：采用 GitHub Secrets 管理密钥，不在代码中硬编码敏感信息。

***

## 更新说明

（2026-10-22）

- **推送精简**：推送只保留摘要（成功 / 失败 / 状态异常 / 跳过计数），逐域名明细留在 Actions 日志。

- **MeoW 推送**：新增 MeoW 作为推荐通知通道，配置 `MEOW_NICKNAME` 即可；PushPlus / Telegram 仍可并存。
- **推送级别**：`PUSH_LEVEL=fail_only` 时仅在有失败项时推送，默认 `all`。
- **外部触发**：已关闭仓库自带 schedule，仅保留 `workflow_dispatch`，由 cron-job.org 每月调用一次。

（2026-10-21）

- **修复静默误判**：`never_expires` 为字符串 `"0"` 时曾被当成「永不过期」而永久跳过续期，现按正确语义解析。

- **错误信息可读**：保留官方 `error_code` 并给出处理建议（此前 401 只显示一句 `Unauthorized`）；未到续期窗口时会提示还需等待多少天。

- **通知不再短路**：此前 PushPlus 失败会导致 Telegram 完全不被调用，现已修复。

- **分页不再静默截断**：服务端分页异常时改为显式报错，并用接口返回的总数校验是否漏收域名。

- **域名状态可见**：被暂停 / 已过期的域名会在报告中标注。

- **阈值边界修正**：剩余正好 180 天时不再被误判为「跳过」。

- **日志脱敏**：推送失败的日志不再泄漏 Telegram Bot Token。

- **运行频率**：cron 错开到非整点执行，避免 GitHub Actions 高负载时被延迟或丢弃。

（2026-08-21）

- **通知渠道扩展**：新增 Telegram Bot 通知方式，可与 PushPlus 并存，未配置则自动跳过。

- **长报告适配**：超过 Telegram 单条消息长度上限时自动分段发送，内容不丢失。

（2026-07-18）

- **智能续期**：先检查域名到期时间，仅对剩余天数不足 180 天的域名执行续期。

- **永不过期识别**：识别已设置为永不过期的域名，自动跳过续期。

- **通知优化**：推送消息分为两段——第一段展示本次续期结果，第二段汇总所有域名到期时间。

***

## 🚀 快速上手

### 第一步：获取 API 密钥

1. 登录 [DNSHE](https://my.dnshe.com/) 。

2. 进入 **“免费域名”** 页面 。

3. 在底部的 **“API 管理”** 卡片中点击 **“创建 API 密钥”** 。

4. 妥善保存获取到的 `API Key` 和 `API Secret` 。



### 第二步：配置通知（任选，可并存）

推荐使用 **MeoW**，配置最简单。PushPlus / Telegram 作为可选备份。

#### MeoW（推荐）

1. 访问 [MeoW API 文档](https://www.chuckfang.com/MeoW/api_doc.html)，按文档注册并获取**用户昵称**。
2. 将昵称填入 Secrets 的 `MEOW_NICKNAME`。

#### PushPlus（可选）

1. 访问 [PushPlus 官网](https://www.pushplus.plus/)。
2. 登录并获取您的 **Token**。
3. 若需推送到群组，请创建一个群组并记录 **群组编码 (Topic)**。

#### Telegram Bot（可选）

1. 在 Telegram 中找到 [@BotFather](https://t.me/BotFather)，发送 `/newbot` 创建机器人，保存获得的 **Bot Token**。
2. 向您的机器人发送任意一条消息（必须先发起过对话，机器人才能向您推送）。
3. 获取 **Chat ID**：浏览器访问 `https://api.telegram.org/bot<你的Token>/getUpdates`，在返回的 JSON 中找到 `chat.id`；也可使用 [@userinfobot](https://t.me/userinfobot) 查询。
4. 将两个值分别填入 Secrets 的 `TELEGRAM_BOT_TOKEN` 与 `TELEGRAM_CHAT_ID`。

> 通道可任选或并存。都未配置时脚本仅跳过推送，不影响续期执行。只要有一个通道送达即视为成功。

### 第三步：配置 GitHub 仓库

1. **Fork 本仓库** 或将脚本及工作流文件上传至您的私有仓库。
2. 进入仓库设置：**Settings** -> **Secrets and variables** -> **Actions**。
3. 点击 **New repository secret**，依次添加以下变量：

| 变量名称                 | 说明                      | 示例                  |
| -------------------- | ----------------------- | ------------------- |
| `DNSHE_API_KEY`      | DNSHE 的 API Key         | `cfsd_xxxxxxxxxx`   |
| `DNSHE_API_SECRET`   | DNSHE 的 API Secret      | `yyyyyyyyyyyy`      |
| `MEOW_NICKNAME`      | (推荐) MeoW 推送昵称         | `your_nickname`     |
| `PUSHPLUS_TOKEN`     | (可选) PushPlus 的用户令牌     | `9a8b...`           |
| `PUSHPLUS_TOPIC`     | (可选) PushPlus 群组编码      | `123`               |
| `TELEGRAM_BOT_TOKEN` | (可选) Telegram Bot Token | `123456:ABC-DEF...` |
| `TELEGRAM_CHAT_ID`   | (可选) Telegram 目标聊天 ID   | `123456789`         |

> 只要有一个通知通道送达即视为成功；所有通道都失败才会让工作流标红。

#### 可选配置（Repository variables，非敏感）

在 **Settings** -> **Secrets and variables** -> **Actions** -> **Variables** 中添加，不配则用默认值：

| 变量名称                         | 默认值                                               | 说明                                     |
| ---------------------------- | ------------------------------------------------- | -------------------------------------- |
| `PUSH_LEVEL`                 | `all`                                             | `all` 每次推送；`fail_only` 仅有失败项时推送        |
| `DNSHE_RENEW_THRESHOLD_DAYS` | `180`                                             | 剩余天数不超过该值时才续期（官方窗口为到期前 180 天）          |
| `DNSHE_MIN_INTERVAL`         | `2`                                               | 相邻两次 API 调用的最小间隔（秒）                    |
| `DNSHE_TZ_OFFSET`            | `8`                                               | 解析到期时间用的时区偏移。**非中国时区用户务必修改**，否则剩余天数会算错 |
| `DNSHE_API_BASE_URL`         | `https://api005.dnshe.com/index.php?m=domain_hub` | 接口地址，一般无需修改                            |

### 第四步：启用自动化

> ⚠️ **Fork 之后定时任务默认是关闭的**，必须手动启用，否则它永远不会自己运行。
> 这是「配置完以为在跑、其实一次都没跑过」最常见的原因。

1. 点击仓库顶部的 **Actions** 选项卡。
2. 若看到提示，点击 **I understand my workflows, go ahead and enable them**。
3. 在左侧选择 **"DNSHE Domain Auto Renew"** 工作流，点击 **Enable workflow**（若已启用则跳过）。
4. 点击 **Run workflow** 手动触发一次，验证配置是否正确。

### 第五步：用 cron-job.org 外部触发

仓库自带的 `schedule` 已关闭。续期只由 cron-job.org 每月调用一次 `workflow_dispatch`。

用免费的 [cron-job.org](https://cron-job.org) 配置：

1. 打开 [GitHub Tokens](https://github.com/settings/tokens) → **Generate new token (classic)**。
   - Expiration：90 天或更久
   - 勾选 **workflow**（在 repo 下方）
   - 生成后立刻复制，格式类似 `ghp_...`，只显示一次
2. 注册 [cron-job.org](https://cron-job.org)，创建 Cronjob：
   - **Title**：`DNSHE 域名续期`
   - **URL**：`https://api.github.com/repos/你的用户名/DNSHE-Auto-Renew/actions/workflows/renew.yml/dispatches`
   - **Request method**：POST
   - **Time zone**：Asia/Shanghai
   - **执行时间**：每月 1 日 08:23
   - Headers：

     | Key | Value |
     | --- | --- |
     | `Accept` | `application/vnd.github.v3+json` |
     | `Authorization` | `token 你的GitHub_Token` |
     | `Content-Type` | `application/json` |

     `Authorization` 必须是 `token` + 空格 + Token，例如 `token ghp_xxx`。
   - Body（Raw）：`{"ref": "main"}`
3. Save。可先点一次 Execute 验证，GitHub Actions 里应立刻出现一次运行。

常见错误：401 多半是 Authorization 少了 `token ` 前缀；422 多半是 Body 没带 `{"ref": "main"}`；403 多半是 Token 没勾 workflow 权限。

需要临时恢复仓库自带定时时，取消 `.github/workflows/renew.yml` 里 `schedule` 的注释即可。

***

## 运行计划

- **执行频率**：每月 1 日北京时间 08:23，由 cron-job.org 触发。仓库自带 schedule 已关闭。

- **速率限制**：官方文档对默认额度有两处不同表述（30 次/分与 60 次/分），脚本按 2 秒间隔保守节流，两种口径下都安全。如需调整请改 `DNSHE_MIN_INTERVAL`。

- **错误处理**：续期失败时，推送消息会带上官方 `error_code` 与可读原因（如认证失败、额度不足、尚未进入续期窗口还需等待多少天）。

- **仓库不活跃不影响外部触发**：cron-job.org 调用 `workflow_dispatch` 不受 GitHub 对闲置仓库禁用 schedule 的限制。工作流本身仍需保持 Enable。

- **报告分级**：推送只发摘要——续期成功的域名、需要处理的失败项、状态异常项，以及跳过项的计数；完整的逐域名明细仍输出到 Actions 日志。这样平时一次推送只有几行，不会把几十行重复的「未到窗口」灌进手机。

- **时区**：脚本按 `DNSHE_TZ_OFFSET`（默认 UTC+8）解读接口返回的到期时间。接口返回的是不带时区的本地时间字符串，**如果你的账号不在 UTC+8，必须改这个值**，否则「剩余天数」会整体偏移。

## ⚠️ 安全建议

- **密钥保护**：切勿将 `API Secret` 上传至公开代码库 。

- **定期轮换**：建议定期在 DNSHE 后台使用 `regenerate` 操作更新密钥以增强安全性 。

- **最小权限**：建议仅为该脚本配置必要的 API 访问权限 。

- **日志脱敏**：推送失败时打印的异常信息会自动抹掉 Bot Token，可安全地贴到 issue 里。

***

## 🧪 本地运行

```bash
pip install -r requirements.txt

export DNSHE_API_KEY=...
export DNSHE_API_SECRET=...
export MEOW_NICKNAME=...                             # 推荐
export PUSHPLUS_TOKEN=...                            # 可选
export TELEGRAM_BOT_TOKEN=... TELEGRAM_CHAT_ID=...   # 可选
export PUSH_LEVEL=all                                # 可选：all / fail_only

python renew_domains.py
```

> 注意：`key.txt`、`.env` 等本地凭据文件已在 `.gitignore` 中排除，请勿提交。

***

## 🙏 致谢

本项目得以实现，特别感谢以下平台与技术的支持：

- [**DNSHE**](https://www.dnshe.com/)

- [**DeepSeek**](https://platform.deepseek.com/usage)
