# Cloudflare ImgBed + Telegram 图床搭建交接文档

> 整理日期：2026-10-10  
> 用途：记录本次个人图床的实际部署过程、最终架构、已验证功能、注意事项和后续待办。

---

## 一、最终方案概览

本项目使用 **Cloudflare Pages + ImgBed + Workers KV + Telegram Bot/频道**。Cloudflare R2 因为银行卡验证未通过，本次没有采用，也不需要为当前方案开通 R2。

### 架构

```text
Windows / 手机 / Obsidian / Hexo 博客
                 │
                 ▼
       Cloudflare Pages（ImgBed）
                 │
          ┌──────┴──────┐
          ▼             ▼
    Workers KV      Telegram Bot API
  文件索引/元数据         │
                         ▼
                Telegram 存储频道
                My ImgBed Storage
```

- **Cloudflare Pages**：托管 ImgBed 应用和相关 API。
- **Workers KV**：保存文件索引、管理元数据等应用数据。
- **Telegram 频道**：保存实际上传的图片、压缩包等文件；大文件可能被 ImgBed 拆分为多个 `.part` 分块消息。
- **DediRock VPS**：本图床方案暂时不使用它存储图片，也不需要在 VPS 上安装 ImgBed、Docker 或 Nginx。
- **Cloudflare R2**：本次未开通、未创建存储桶，当前架构不依赖 R2。

> 重要：Telegram 没有在此方案中提供可视为“无限、永久、带备份保证”的存储承诺。不要把 Telegram 当作唯一的珍贵照片备份。

## 二、本次部署的实际配置

| 项目 | 当前值/状态 |
|---|---|
| GitHub Fork 仓库 | `XZZD-Star/CloudFlare-ImgBed` |
| Cloudflare Pages 项目 | `cloudflare-imgbed` |
| Pages 默认域名 | `https://cloudflare-imgbed-6s2.pages.dev` |
| 正式使用的统一域名 | `https://bed.rcardoshaw.de5.net` |
| 管理后台 | `https://bed.rcardoshaw.de5.net/dashboard` |
| 生产分支 | `main` |
| 构建命令 | `npm install` |
| 构建输出目录 | `/frontend-dist` |
| KV 绑定变量名 | `img_url` |
| KV 命名空间 | 建议/当前使用 `img_url` |
| Telegram Bot 显示名 | `Shaw`（以 Telegram 中实际 Bot 为准） |
| Telegram 存储频道 | `My ImgBed Storage` |
| ImgBed 内的渠道名称 | `Telegram主存储` |
| 默认 URL 前缀 | 留空，使用当前站点域名 |
| R2 | 未开通、未配置 |

**凭据提醒：** Bot Token、管理员密码等没有写入本文档。请将 Token 保存在密码管理器或安全的本地记录中，不要放进公开仓库、博客或截图。

## 三、从零开始的完整部署流程

### 1. Fork 官方项目

1. 打开官方仓库：<https://github.com/MarSeventh/CloudFlare-ImgBed>。
2. 登录 GitHub，点击 `Fork`，将仓库复制到自己的账号。
3. 本次使用的 Fork 为 `XZZD-Star/CloudFlare-ImgBed`。
4. 确认默认分支为 `main`。

本次曾在 Cloudflare Pages 页面见到 GitHub 连接警告，之后前往 GitHub 的 Cloudflare Workers and Pages 应用授权页面检查/补充了授权。后续应再确认 Pages 项目不再显示 Git 断开警告，并检查新提交是否能触发自动部署。

GitHub 应用授权管理：<https://github.com/settings/installations>

### 2. 创建 Cloudflare Pages 项目

1. 登录 <https://dash.cloudflare.com/>。
2. 进入 `Workers & Pages`，点击创建应用。
3. 新界面可能默认进入 Workers 流程；选择页面中的 `Looking to deploy Pages?` → `Continue to Pages` 或 `Get started`。
4. 选择导入现有 Git 仓库。
5. 授权 Cloudflare 访问 GitHub，并选择自己的 `CloudFlare-ImgBed` Fork。
6. 配置项目参数：

| 配置项 | 值 |
|---|---|
| Project name | `cloudflare-imgbed` |
| Production branch | `main` |
| Build command | `npm install` |
| Build output directory | `/frontend-dist` |

7. 点击 `Save and Deploy`，等待部署成功。
8. 先用 `pages.dev` 地址确认网页能正常打开。

> 注意：不要误选普通 Workers 创建流程；ImgBed 本次采用 Pages。当前官方部署说明使用 `npm install` 和 `/frontend-dist`。

### 3. 创建并绑定 Workers KV

1. Cloudflare 控制台进入 `Storage & databases` → `Workers KV`。
2. 创建 KV 命名空间，建议命名为 `img_url`。
3. 返回 Pages 项目 `cloudflare-imgbed`。
4. 进入 `Settings` → `Bindings` → `Add` → `KV namespace`。
5. 填写：
   - Variable name：`img_url`
   - KV namespace：选择创建的 `img_url`
6. 保存后重新部署一次，让绑定在运行环境中生效。

> `img_url` 是绑定变量名，必须准确填写。Cloudflare Pages 部署时 KV 和 D1 选一个即可，本项目采用 KV，不需要同时建 D1。

### 4. 配置 Telegram Bot 和频道

#### 4.1 创建 Bot

1. 在 Telegram 打开官方 `@BotFather`：<https://t.me/BotFather>。
2. 发送 `/newbot`，按提示设置 Bot 名称和用户名。
3. 复制 Bot Token，并安全保存；不要把 Token 放进 GitHub 或发到公开位置。

#### 4.2 创建存储频道

1. 在 Telegram 创建一个专用频道，本次频道名为 `My ImgBed Storage`。
2. 将 Bot 添加为频道管理员。
3. 根据需要授予 Bot 发消息和管理消息的权限，并保存管理员设置。
4. 在频道中发一条测试消息。
5. 按项目文档建议，通过可读取频道信息的 Bot/工具获取频道 Chat ID。Chat ID 如果以负号开头，必须保留完整值。

#### 4.3 在 ImgBed 配置 Telegram 渠道

1. 打开管理后台：<https://bed.rcardoshaw.de5.net/dashboard>。
2. 进入 `系统设置` → `上传设置`。
3. 找到 `Telegram 渠道配置`，点击添加渠道。
4. 填写：
   - 渠道名称：`Telegram主存储`
   - Bot Token：填自己的真实 Token
   - Chat ID：填频道实际 ID（保留负号）
   - 代理 URL：如不需要代理可留空
   - 启用状态：开启
5. 保存配置。
6. 在 `系统设置` → `网页设置` → `客户端设置` 中，将默认渠道类型设为 `Telegram`，默认渠道名称选 `Telegram主存储`。

> 以当前部署版本的管理后台为准。不要照搬旧教程去添加已经废弃的业务环境变量；项目官方文档说明 v2.0 之后这类旧环境变量配置方式已废弃。

### 5. 设置管理员密码与基础安全

1. 进入 `系统设置` → `安全设置` → `认证管理`。
2. 设置并验证管理员用户名与强密码。ImgBed 官方文档提醒，初始管理后台可能无需密码，首次进入后应及时设置认证。
3. 如果只供自己使用，检查普通用户认证、上传 API 访问等设置，避免无意公开上传权限。
4. 不需要的功能先保持关闭，尤其不要把管理员凭据或 Bot Token 写入公开配置。
5. `Secure` Cookie 模式只有在 HTTPS 正常工作的情况下才开启，否则可能出现无法登录的问题。

### 6. 上传与大文件验证

本次已经完成以下测试：

- Telegram 频道中出现上传的文件消息。
- ImgBed 管理界面中能显示文件记录。
- 上传了约 `170.34 MB` 的 ZIP 文件 `nand_buildroot_img_260329.zip`。
- Telegram 中该 ZIP 被拆分成约 11 条 `.part` 分块消息。
- 从 ImgBed 下载该 ZIP 后，用户确认文件下载没有问题，说明当前大文件分块上传/读取/合并链路已通过实际测试。

注意：

- Telegram 中看到多个 `.part` 文件是该分块机制下的预期现象，不应逐个当作独立 ZIP 解压。
- 日常恢复大文件，应从 ImgBed 下载该逻辑文件，而不是在 Telegram 中手工分别下载每个分块。
- 重要文件仍需独立备份到电脑或外置硬盘。

### 7. 设置统一域名和分享链接

本次最终决定只保留一个日常域名：

`https://bed.rcardoshaw.de5.net`

此前尝试过将管理域名和图片域名拆分为 `bed.` 与 `img.` 两个子域名，但最终决定使用同一个域名，减少配置复杂度。

操作要点：

1. 在 Cloudflare Pages 项目 `cloudflare-imgbed` 的 `Custom domains` 中添加并验证 `bed.rcardoshaw.de5.net`。
2. 根据 DNS 实际托管位置，检查或创建所需 DNS 记录。先在 Pages 中添加自定义域名，再处理 DNS；不要把该域名指向 VPS IP。
3. 确认 `https://bed.rcardoshaw.de5.net/dashboard` 可以正常访问。
4. 进入 `系统设置` → `网页设置`，将 `默认 URL 前缀` 留空。
5. 保存。留空后 ImgBed 使用当前站点域名生成链接。
6. 在文件管理页的操作栏点击复制图标，选择 `原始链接` / `Markdown` / `HTML` / `BBCode` 等格式，再点击确认复制。
7. 在记事本中按 `Ctrl+V`，确认剪贴板已拿到链接。
8. 在浏览器无痕窗口打开一条分享链接，确认未登录状态下能显示图片或下载文件。

本次实际检测成功的链接示例：

`https://bed.rcardoshaw.de5.net/file/1791558710780_a0fa8b65c0b388f73add1f5bf8573e8c482619644.jpg`

该链接在当时的测试中能直接显示图片。它只是示例文件链接，实际使用时请复制当前文件真实生成的链接。

> 旧的 `img.rcardoshaw.de5.net` 前缀已经不再作为当前方案使用。如果之前保存过以 `img.` 开头的旧链接，需要重新复制；在清理旧域名绑定之前，先确认没有仍在使用的链接。

### 8. 目录整理（待完成/按需设置）

目录是 ImgBed 的逻辑目录，用于文件分类，不代表 Telegram 频道会创建物理文件夹。

设置入口：

1. `系统设置` → `网页设置`。
2. 找到 `客户端设置`。
3. 开启 `目录候选项`，这样上传时可以通过目录树选择文件夹。
4. 在 `默认上传目录` 中填入以 `/` 开头的路径；留空时使用根目录。

可以按需规划目录，例如：

```text
/phone/photos/2026
/phone/videos/2026
/blog
/projects
/documents
```

例如，要将上传默认位置设为 2026 年手机照片目录，可填：`/phone/photos/2026`。建议先上传少量测试文件，确认目录路径与文件记录符合预期后，再批量整理。

### 9. 检查 GitHub 自动部署与更新

1. 确认 Cloudflare Pages 不再提示 Git 账号断开连接。
2. 在 GitHub 修改非敏感内容或进行一次受控提交后，观察 Pages 是否触发新的部署（无需为了测试而改生产代码）。
3. 检查 Pages `Deployments` 最新生产部署是否成功。
4. 更新 Fork 前先查看官方仓库公告和兼容性说明，再执行 `Sync fork`，不要无条件覆盖自己做过的修改。

官方仓库和公告：

- <https://github.com/MarSeventh/CloudFlare-ImgBed>
- <https://github.com/MarSeventh/CloudFlare-ImgBed/discussions/categories/announcements>

---

## 四、当前已知问题：从 ImgBed 删除文件不会同步删除 Telegram 消息

### 现象

- 在 ImgBed 删除测试图片后，Telegram 的 `My ImgBed Storage` 频道中对应消息仍然保留。
- Telegram Bot（显示名 Shaw）已在频道中设为管理员，并开启了管理消息相关权限；因此目前不应只继续反复调整权限。
- 这不代表 ImgBed 在 Pages 上另存了一份原文件，更可能是删除操作只处理了 ImgBed 管理数据，或者 Telegram 消息删除链路未执行/未完整处理。具体根因尚未通过源码确认。

### 官方反馈链接

ImgBed GitHub 上存在同类讨论：

<https://github.com/MarSeventh/CloudFlare-ImgBed/discussions/588>

### 暂行处理方式

- 不要把 ImgBed 删除按钮视作“Telegram 原文件已经被删除”的保证。
- 确实要清理文件时，在 Telegram 频道手动检查并删除对应消息。
- 如果是大文件，需要留意该逻辑文件对应的所有 `.part` 分块消息。
- 不要在没有备份的情况下对真实频道进行批量删除测试。

### 后续若要自行修复源码

建议使用 Windows + GitHub Desktop + VS Code/Codex：

1. 克隆自己的 Fork 到本地。
2. 新建分支，例如 `fix/telegram-delete-sync`，不要直接在 `main` 上试改。
3. 先让 Codex 只分析源码：追踪 Telegram 上传、消息 ID 保存、文件元数据写入、前端删除请求、后端删除 API 和 Telegram 渠道删除实现。
4. 如果上传时没有保存 Telegram 的 `chat_id` 和每个分块的 `message_id`，就需要在上传成功时记录映射；删除时遍历所有对应消息 ID 调用 Telegram Bot API 的删除接口。
5. 删除失败或部分失败时，不能无条件丢弃唯一的 KV 元数据，否则后续可能无法重试清理。
6. 先用独立 KV 和测试频道验证普通文件、大文件多分块、部分删除失败、重复删除等情况；确认后再合并与部署。

Telegram Bot API 说明：<https://core.telegram.org/bots/api#deletemessage>

---

## 五、费用和限制说明

### 本项目当前费用结构

- 本次没有开通 Cloudflare R2，因此不产生 R2 存储桶的费用。
- Cloudflare Pages、Workers KV 以及 Telegram 的使用仍应遵守对应服务当前的计划、用量限制和政策。
- Telegram 不能被视为承诺无限容量、永久可用或自动备份的专业对象存储。使用前应自行留意服务变更、Bot API 限制与账号/频道安全。
- 如果未来改用 R2 或其他 S3 兼容存储，需重新评估银行卡验证、费用、上传限制和迁移方式。

## 六、日常运维建议

### 日常上传

- 图片、视频、ZIP、文档等先通过 ImgBed 上传。
- 对大文件，优先使用 ImgBed 的完整文件下载链接进行恢复，不要手动逐个处理 Telegram 分块。
- 文件名和目录尽量规范，便于以后检索。

### 备份

- 珍贵的手机照片和视频保留本地或独立存储副本，不要只保存在 Telegram。
- 定期抽查下载文件是否可打开；压缩包可以使用 7-Zip 的“测试”功能检查完整性。
- 保留 GitHub Fork 和必要的配置记录。
- 安全保存 Bot Token、管理员凭据和频道 Chat ID；不要把 Token 贴到公开 Issue、博客、截图或仓库。
- 了解 KV 文件索引的导出/恢复方式。Telegram 里仍有原文件，不代表丢失 KV 后管理列表一定能自动重建。

### 故障排查

| 问题 | 优先检查 |
|---|---|
| Pages 部署失败 | GitHub 仓库授权、生产分支、`npm install`、`/frontend-dist`、构建日志 |
| 管理后台异常 | KV 绑定变量是否准确为 `img_url`，绑定后是否重新部署 |
| 上传失败 | Telegram 渠道是否启用、Bot Token、Chat ID、Bot 频道权限、网络/代理设置 |
| 大文件下载损坏 | 重新从 ImgBed 下载；确认分块是否完整；用 7-Zip 测试 ZIP |
| 分享链接打不开 | 域名绑定状态、默认 URL 前缀、链接路径、文件是否仍存在、缓存 |
| 删除后 Telegram 文件仍在 | 当前已知问题；手动检查频道消息并保留文件映射，后续考虑修改源码 |
| 上传后看不到目录 | `系统设置 → 网页设置 → 客户端设置`，开启 `目录候选项` 并设置默认上传目录 |
| GitHub 更新不触发部署 | 检查 Pages 的 Git 连接警告、授权范围和最新部署日志 |

---

## 七、官方参考资料

1. [ImgBed 官方 GitHub 仓库](https://github.com/MarSeventh/CloudFlare-ImgBed)
2. [Cloudflare Pages 部署说明](https://cfbed.sanyue.de/deployment/pages.html)
3. [ImgBed 配置说明](https://cfbed.sanyue.de/deployment/configuration.html)
4. [ImgBed 官方公告](https://github.com/MarSeventh/CloudFlare-ImgBed/discussions/categories/announcements)
5. [删除不同步的同类反馈](https://github.com/MarSeventh/CloudFlare-ImgBed/discussions/588)
6. [Telegram Bot API：deleteMessage](https://core.telegram.org/bots/api#deletemessage)

---

## 八、项目状态总览

### 已完成并已验证

- [x] Fork ImgBed 到自己的 GitHub 账号。
- [x] 创建 Cloudflare Pages 项目并部署成功。
- [x] 使用 `npm install` 和 `/frontend-dist` 构建。
- [x] 创建并绑定 KV，变量名为 `img_url`。
- [x] 创建 Telegram Bot 和存储频道，并配置 ImgBed Telegram 渠道。
- [x] 上传普通图片，Telegram 频道中能看到对应文件。
- [x] 上传约 170.34 MB ZIP，Telegram 中出现多个分块消息。
- [x] 从 ImgBed 下载该大文件，用户确认下载正常。
- [x] 绑定统一自定义域名 `bed.rcardoshaw.de5.net`。
- [x] 清空默认 URL 前缀后，复制分享链接功能恢复正常。
- [x] 提供的图片链接可正常访问。

### 仍需完成或定期确认

- [ ] 确认 Cloudflare Pages 的 GitHub 连接警告已消失、自动部署正常。
- [ ] 检查管理员密码和访问策略已按个人使用场景配置。
- [ ] 按需开启目录候选项并设置默认上传目录。
- [ ] 处理 Telegram 删除不同步问题，或接受其现状并手动清理。
- [ ] 建立独立备份并实际测试恢复。
- [ ] 在批量上传大量手机照片前，进一步验证删除、长期访问与恢复流程。

> **当前最重要的未解决问题是 Telegram 删除不同步。** 大文件上传/下载和分享链接已经通过实际测试，但在删除逻辑解决、备份就绪前，不建议把这套系统作为重要照片的唯一存储位置。
