
# 一、先理解：我们最终要搭建的是什么？

首先，你要搭建的并不只是一个上传图片的网站，而是一套由几个组件共同组成的个人图床系统。

## 你的电脑 / 手机 / Obsidian / 博客

上传图片、管理文件、复制图片外链、展示博客图片

## CloudFlare ImgBed

图床管理界面：负责上传接口、文件列表、目录、认证和管理操作

### KV 或 D1

保存文件索引、管理元数据等信息

### Cloudflare R2

真正保存图片、原始文件的内容

## Cloudflare 网络与缓存

通过图片专用域名访问文件，并缓存适合缓存的内容

这张图是整套系统的核心。ImgBed 的官方架构说明也明确区分了管理程序、数据存储和文件存储。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

+1

你可以把它理解为：

- ImgBed 是图床软件。 决定怎么上传、怎么管理、怎么生成外链。
    
- KV / D1 是管理数据存储。 负责保存程序需要的文件信息，并非用来存放图片原始内容。
    
- R2 是存放图片的仓库。 不管管理程序以后在哪里运行，图片都可以保留在 R2。
    
- Cloudflare CDN 是图片分发和缓存层。 用来改善重复访问的速度，减少对原始存储的请求。
    

这四者不要混为一谈，后面绝大多数配置问题都可以根据这个关系来定位。

# 二、我们为什么选择 Cloudflare Pages，而不是直接用 VPS 部署？

ImgBed 支持 Docker 和 Serverless 等部署方式。目前项目官方文档将 Cloudflare Pages 列为推荐部署方式。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

+1

对你来说，有两条路线。

路线 A：Cloudflare Pages + R2

建议采用

ImgBed 运行在 Cloudflare 的托管环境中，图片保存至 R2。通常不需要自己管理 VPS 的操作系统、Docker、Nginx 或服务器证书。

主要工作是连接 GitHub、部署项目、创建数据存储和绑定域名。

路线 B：VPS + Docker + R2

ImgBed 程序运行在你的洛杉矶 VPS 上，图片仍然存放在 R2。

你还需要维护服务器、容器、端口、反向代理、HTTPS 和程序更新。R2 仍能让图片存储独立于 VPS，但管理程序本身依然依赖服务器。

我们正式搭建时采用路线 A。 你的 DediRock VPS 可以继续用于其他项目，不必强行拿来运行图床。这样可以减少需要掌握的技术点，也符合你希望便于迁移的要求。

注意，这不意味着 Pages 完全不需要配置。它只是把服务器运维工作减少了，存储绑定、域名、认证等仍然需要正确设置。

# 三、正式搭建涉及的 8 个核心知识点

## 1. GitHub Fork：取得并维护项目源码

![First Timers Guide to Contributing to an Open Source Project](https://images.openai.com/static-rsc-4/iD6tlObR9MihEuRkz0JR34iK7x_-7G1xOGT1M8GKaNlr4YICfCV7KVyIhZwB7UtpS6Xzjmk3YdrglnNGw_C8xitJzrccE5WH8KqTxv1CxPCEQBjN2U0KH-S6_0yqlH6sAe1wN8AG31iOl3IWiD-VYfOSJgaiF_oMBa6Vb2Vpt-A?purpose=inline)

ImgBed 的源代码托管在 GitHub 上。我们会把官方项目 Fork 到你自己的 GitHub 账户，再让 Cloudflare 从你的仓库部署。

ImgBed 官方仓库

这里涉及三个概念：

Repository（仓库）：保存项目的代码、配置、版本历史。

Fork（派生仓库）：把开源项目复制到自己账户下，保留与上游项目的关联。

部署与更新：我们连接自己的仓库后，Cloudflare 可以根据仓库的提交自动构建和部署。将来更新程序时，需要先检查官方的更新公告，避免不兼容更新破坏现有配置。

你不需要先在 Windows 上安装 Node.js，也不必把整个项目下载到电脑上才能部署。通过网页连接 GitHub 即可完成主要流程。

## 2. Cloudflare Pages：程序到底运行在哪里？

![How I Deploy NextJS Apps to Cloudflare Pages in Just 4 Steps | by Milind Kumar Sahu | Medium](https://images.openai.com/static-rsc-4/rhrgj-I9vX49qhpeOPt3yEstAsypzHM0PcZgQ4pJmJgTTDEcwUntudvphLSey6fDHAjiiO9BGZ6jUeAmm0hiiAMJq7Py6ZTKqSAbzA0j03o6-stI13LrtJCh6BalJHBaakdSGcg1dJq675ajVPo8tVStpg5aVf2vpT3UXV8q89Y?purpose=inline)

[medium.com](https://medium.com/%40milindkusahu/how-i-deploy-nextjs-apps-to-cloudflare-pages-in-just-4-steps-33e6d9b43fe5)

Cloudflare Pages 是我们用来部署 ImgBed 的平台。

你可以把它理解成一个托管网站和执行配套后端代码的环境。部署后会得到一个类似下面的默认域名：

`https://你的项目名.pages.dev`

正式配置完成前，我们先用这个默认域名进行测试，不必急着绑定自己的域名。

官方目前的 Pages 部署文档介绍了通过 GitHub 导入项目、设置构建命令、输出目录和部署的流程。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

几个概念要区分清楚：

|名词|含义|
|---|---|
|构建命令|告诉 Cloudflare 怎样准备项目|
|输出目录|告诉 Pages 去哪里寻找构建后的网页文件|
|部署|将代码发布成能够访问的服务|
|Pages Functions|执行项目后端 API 的 Serverless 代码|
|自动部署|代码发生符合条件的更新后，触发新的构建和发布|

Serverless 不是说没有服务器，而是说服务器的运行和维护主要由平台负责。

## 3. KV 和 D1：为什么还需要数据库？

很多人第一次搭建图床时会有一个疑问：

“图片已经放在 R2 了，为什么还要数据库？”

原因是，图片文件本身并不等于完整的管理系统。

例如，你上传了一张名为 `stm32.jpg` 的图片，图床可能还需要记录：

- 图片所在的存储渠道和对象路径；
    
- 对应的访问地址；
    
- 文件名、目录、标签等信息；
    
- 用于管理界面展示的其他元数据。
    

这时就需要 KV 或 D1。ImgBed 的 Pages 部署指南明确要求二选一，不需要同时创建两种数据库。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

KV：键值存储

类似一个通过名称查找数据的简易存储柜。配置步骤相对直观，适合我们优先考虑简单部署的场景。

D1：SQL 数据库

可以理解为托管的关系型数据库，使用 SQL 管理表和数据。项目需要初始化数据库结构，操作相对多一些。

我们的初步选择：KV。 主要是减少初次搭建的步骤。之后真有复杂查询、数据统计之类的需求，再结合项目支持情况考虑 D1。

有一个很重要的细节：项目在 Cloudflare 中创建 KV 后，还要把它绑定到 Pages，并使用官方指定的变量名 `img_url`。绑定之后通常还需要重新部署才会生效。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

+1

如果绑定名称填错，即使 KV 已经创建成功，程序也可能无法正确访问数据。

## 4. Cloudflare R2：图片真正保存在哪里？

![Cloudflare R2 Hands-On Guide: Set Up Free 10GB Storage, Zero-Egress Object Storage, and S3-Compatible API Keys - DEV Community](https://images.openai.com/static-rsc-4/tNcME-nN0OlHnY5EYTun_aWnVJcWtjGCPWKm6-xWhHkqYBn41myepO_J9mFaS6M-NwzNVbTP4I5iJ4KGSWsRRt6rCZGsk7jG0ZQYWmLlCWyOW4mC4TdRKbgm0QCkrDlykfPAmyPpkIhUv1c5W0sp6IPG0Gq2aNav0LbIrDTmems?purpose=inline)

[dev.to](https://dev.to/yeagoo/cloudflare-r2-hands-on-guide-set-up-free-10gb-storage-zero-egress-object-storage-and-325n)

这是整套系统最重要的存储组件。

R2 属于对象存储，适合保存大量独立的图片、文档、压缩包等文件。每个文件都会有一个对象标识或对象路径，你可以把它理解为存储桶中的文件位置。

例如，逻辑上的文件可能是：

```
images/
├── blog/
│   ├── stm32.jpg
│   └── stm32-pinout.png
├── avatars/
│   └── avatar.webp
└── screenshots/
    └── cloudflare-dashboard.png
```

这些名称只是帮助理解对象的组织方式，实际对象路径要以上传时的命名和 ImgBed 的配置为准。

R2 与 VPS 硬盘最大的区别是：图片不必存在你的 Ubuntu 文件系统中。因此，哪怕以后更换服务器，也不一定需要迁移图片文件。

根据 ImgBed 当前的官方配置说明，Cloudflare 部署时，需要把 R2 存储桶绑定到项目，并使用变量名 `img_r2`；绑定后，还要在 ImgBed 的管理后台添加和配置 R2 存储渠道。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

这里还有一个重要区别：

- R2 存储桶：真正保存图片的地方。
    
- R2 绑定：允许 Pages Functions 通过平台提供的接口访问这个存储桶。
    
- ImgBed 存储渠道：告诉图床程序使用哪一个存储后端进行上传和读取。
    

创建了 R2 存储桶，并不代表图床已经可以正常上传图片。这三步都要完成。

## 5. DNS、域名和 HTTPS：为什么建议分开管理域名？

我们计划将来使用两个不同的子域名。下面只是示例：

|用途|示例域名|作用|
|---|---|---|
|图床管理界面|`bed.rcardoshaw.de5.net`|打开 ImgBed、登录和上传图片|
|图片访问地址|`img.rcardoshaw.de5.net`|在博客或其他网页中展示图片|

这两个域名只是建议的命名方案，实际能否使用，要先检查你的域名是否已经在自己的 Cloudflare 账户中正确管理。

为什么要分开？

因为管理网站和图片文件的使用方式不同。一个负责交互、登录和上传；另一个负责直接返回图片内容，并通过 Cloudflare 缓存进行分发。

这里涉及三个概念：

- DNS 解析：将域名指向相应服务。
    
- HTTPS / TLS 证书：保障浏览器与服务之间的 HTTPS 通信。
    
- 自定义域名绑定：把你选定的域名连接到 Pages 项目或 R2 存储桶。
    

尤其要注意：R2 通过自定义域名公开访问时，域名需要属于同一 Cloudflare 账户下管理的 Zone。官方文档也说明，公开访问的 R2 自定义域名可以使用 Cloudflare Cache，而开发用途的 `r2.dev` 地址不支持这些缓存、安全防护功能。

![](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com&sz=32)

Cloudflare R2 docs

+1

在正式搭建前，我们需要先确认 `rcardoshaw.de5.net` 的域名管理条件是否满足。 如果符合要求，就可以沿用它来设计图床的域名。如果不符合，我们要先确定一个能够被你正常绑定到 Cloudflare 的域名，而不是等搭建完才发现无法绑定。

## 6. CDN 缓存：为什么第二次下载可能更快？

这是整个方案提升下载体验的关键。

假设你把一张 `stm32.jpg` 插入自己的博客，访问者通过图片域名请求它。

第一次访问

访问者请求图片 → 缓存未命中 → 从 R2 获取原图

Cloudflare 缓存适用的图片内容

将可缓存的内容存放在合适的边缘节点

后续访问

若缓存命中，可直接由边缘节点返回，不必每次都读取 R2

Cloudflare 的官方说明指出，R2 自定义域名可以接入 Cloudflare Cache；不同边缘节点各自可能发生缓存未命中，Tiered Cache 可以进一步减少回源请求。

![](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com&sz=32)

Cloudflare Cache (CDN) docs

不过，有三点必须理解：

第一，CDN 不会让所有地区的第一次访问都变得更快。实际表现取决于访问线路、文件大小和缓存命中情况。

第二，CDN 缓存不是永久备份。即使缓存中还有图片，也不能拿它代替 R2 中的原始文件。

第三，如果你反复用同一个 URL 替换图片内容，浏览器和 CDN 可能继续显示旧版本。因此，博客配图建议尽量使用不同的文件名或对象路径，而不是频繁覆盖已有图片。

对于个人图床，我们先使用正确的自定义域名和默认缓存行为，再根据实际测试考虑是否需要额外的缓存规则。

## 7. 费用：哪些东西免费，哪些东西可能收费？

Cloudflare 并不是所有服务都完全免费。我们要分别看 Pages、KV 和 R2。

以 R2 标准存储为例，Cloudflare 在 2026 年 10 月的价格文档列出的每月免费额度包括 10GB 存储、100 万次 A 类操作、1000 万次 B 类操作；公网出站流量不收费。超出免费额度后，会根据存储量和请求操作量等计费。

![](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com&sz=32)

Cloudflare R2 docs

这里的 A 类操作主要涉及写入、修改等操作，B 类操作主要涉及读取现有对象。

这意味着：

- 上传大量图片会涉及写入请求；
    
- 访问图片会涉及读取请求；
    
- 长期保留图片会占用存储空间；
    
- 图片经 CDN 缓存命中后，可以减少对 R2 的回源读取，但仍应以实际账单和统计数据为准。
    

此外，KV、Pages 和其他 Cloudflare 功能也各自有相应的使用限制或计费规则。部署前应查看对应服务的当前计划，不要直接把整个系统理解为“永久无限免费”。

对于个人博客配图、截图和项目文档，如果容量与访问量不大，这套架构值得优先考虑成本控制。

## 8. 备份：如何尽可能避免图片丢失？

你特别在意文件保存稳定性，所以这一部分不能忽略。

R2 是主存储，但它不能代替独立备份。 例如，误删图片、误操作批量清理，或者管理元数据出了问题，仍然可能导致损失。

我建议把需要保护的数据分成三类：

|数据类型|保存位置|保护方式|
|---|---|---|
|图片原文件|R2 存储桶|定期复制到电脑、外置硬盘或另一个独立存储|
|文件索引和管理元数据|KV 或 D1|按所选数据库的能力制定导出或备份方法|
|程序与配置|GitHub 仓库、Cloudflare 配置|保留版本记录，记录绑定名称、域名与必要配置|

另外，密码、API Token、Secret 等敏感信息不能直接提交进公开 GitHub 仓库，也不要在截图或交接文档中公开完整密钥。

我们后续会把备份流程放在图床正常运行之后完成，并实际验证至少一份备份能否恢复。只生成了一个备份文件、却没有验证恢复成功，还不能证明方案真的可靠。

# 四、从现在到图床正式可用，完整流程是怎样的？

前面是知识点。真正动手时，我们会按照下面的顺序推进，而不是一开始就同时配置所有东西。

## 01

准备阶段：确认账户与域名

确认 GitHub 仓库访问正常、Cloudflare 账户可以管理相关资源，并检查 `rcardoshaw.de5.net` 是否满足绑定 Pages 和 R2 自定义域名的条件。

验收目标：确认部署入口和域名方案可行。

## 02

Fork 并部署 ImgBed

将官方项目 Fork 到自己的 GitHub 账户，连接 Cloudflare Pages，按当前官方文档设置构建参数。

验收目标：先通过 `pages.dev` 默认域名打开网页，确认部署成功。

## 03

创建并绑定 KV

创建 KV 命名空间，将它绑定到项目要求的变量名 `img_url`，再重新部署。

验收目标：图床管理界面能正常加载，相关数据存储绑定成功。

## 04

创建并绑定 R2

创建 R2 存储桶，将它绑定到变量名 `img_r2`，然后在 ImgBed 的存储设置中添加 R2 渠道。

![](https://www.google.com/s2/favicons?domain=https://cfbed.sanyue.de&sz=32)

CloudFlare ImgBed

验收目标：图床可以向 R2 正常写入和读取测试图片。

## 05

配置认证与安全

检查管理后台和上传功能的认证设置，立即设置强管理员密码，不公开不必要的上传接口。

验收目标：只有授权用户能够执行管理操作，公开图片则可以通过外链访问。

## 06

绑定两个域名

为管理界面绑定 Pages 自定义域名，为图片存储桶绑定 R2 自定义域名。后者需要满足 Cloudflare 的域名管理要求，并按需要使用缓存。

![](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com&sz=32)

Cloudflare R2 docs

+1

验收目标：管理网站与图片链接均通过 HTTPS 正常访问。

## 07

完整测试与备份

测试上传、复制外链、刷新访问、在无登录状态下打开公开图片、删除测试图片、检查缓存，并建立独立备份。

验收目标：整个流程不仅能够运行，还可以检查故障并恢复数据。

# 五、正式搭建前，你最需要记住什么？

我把整套架构再压缩成一个简单的排错对照表，后续实际操作时可以直接拿来用。

|出现的问题|优先检查哪里|
|---|---|
|Pages 部署失败|GitHub 仓库、构建命令、输出目录、构建日志|
|网页能打开，但管理界面异常|KV 绑定名称、数据库绑定是否生效、是否重新部署|
|图片无法上传|R2 绑定 `img_r2`、存储渠道配置、权限和请求限制|
|上传成功，但图片打不开|图片公开访问域名、对象路径、外链配置、访问权限|
|图片下载速度不稳定|CDN 缓存命中情况、访问网络、文件大小|
|删除或替换图片后仍看到旧内容|浏览器缓存、CDN 缓存和对象 URL 是否变化|
|更换部署环境后文件列表异常|R2 对象是否完整、KV/D1 中的元数据是否保留|
|服务器故障后担心数据丢失|原始文件备份、数据库备份和恢复验证|

最后，我希望你先建立一个正确的预期：我们不是要在 VPS 上安装一个软件，而是要搭建一套以 Cloudflare 为运行平台、以 R2 为图片主存储、以 ImgBed 为管理入口的个人文件服务。

这样设计的好处是，后面你即使更换 VPS、升级电脑，甚至重新部署图床管理程序，图片数据也不必跟着迁移。

下一步正式开始时，建议先完成域名条件检查和 GitHub 仓库准备，再部署 Pages，最后才连接 KV、R2 与图片域名。我们按步骤验证结果，可以把问题限定在当前环节内，排错会更直接。