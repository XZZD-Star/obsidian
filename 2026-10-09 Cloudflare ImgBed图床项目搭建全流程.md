我们现在正式开始搭建你的个人图床。整个项目以 Cloudflare Pages + Cloudflare ImgBed + Workers KV + Cloudflare R2 为核心，先让上传、存储、访问全部正常，再配置自定义域名、图片链接和备份。

我核对了 ImgBed 官方部署文档，有几个配置细节需要以当前版本为准。下面按照实际操作顺序来做，不需要你在 DediRock 的 Ubuntu 服务器上安装任何东西。官方也将 Cloudflare Pages 列为推荐部署方式。 【turn983652view0】

# 一、最终要搭建成什么样？

你的 Windows 电脑 / 手机 / Obsidian / Hexo 博客

上传图片、管理文件、复制 Markdown 图片链接

HTTPS 请求

Cloudflare Pages

部署 ImgBed 程序，同时运行配套的后端 API

文件索引

Workers KV

文件索引及管理数据

原始文件

R2 存储桶

真正保存图片文件

Cloudflare 网络

通过图片访问域名分发内容，并按适当的缓存规则加速访问。

这里有一个重要区别：KV 保存管理数据，R2 保存图片，ImgBed 负责把两者组织成一套可操作的图床。 不能只创建 R2 就认为图床已经搭好了。

## 开始前先确定域名方案

你现有的域名是 `rcardoshaw.de5.net`。我们暂时规划为：

|用途|规划域名|
|---|---|
|图床管理与上传|`bed.rcardoshaw.de5.net`|
|图片访问链接|`img.rcardoshaw.de5.net`|

这两个域名目前只是规划，还需要验证能否成功绑定。

特别注意：我建议先把两个子域名都绑定到同一个 Pages 项目，而不是一开始就把 `img` 子域名直接绑定到 R2 存储桶。 ImgBed 官方支持通过默认 URL 前缀生成 `/file/` 形式的访问链接，因此这种做法更符合应用自身的访问路径。直接绑定 R2 是另一种独立的配置方式，不能把同一个主机名同时分配给 Pages 和 R2。 【turn624852view1】【turn884375search3】

即使域名暂时不能绑定，也不会影响我们先使用 `pages.dev` 完成部署。

# 二、第一阶段：Fork 官方 ImgBed 项目

目标：把开源项目复制到自己的 GitHub 账号，以便 Cloudflare 直接读取代码并自动部署。

## 第 1 步：打开官方项目

点击下面的链接：

[CloudFlare ImgBed 官方 GitHub 仓库](https://github.com/MarSeventh/CloudFlare-ImgBed)

进入项目主页后：

1. 登录自己的 GitHub 账号。
    
2. 点击右上方的 Fork。
    
3. 选择自己的 GitHub 账号作为目标。
    
4. 如果页面让你设置仓库名称，暂时保留默认的 `CloudFlare-ImgBed` 即可。
    
5. 点击创建，等待 Fork 完成。
    

Fork 完成后，你应该能在自己的 GitHub 账号下看到这个仓库。

这里不需要下载 ZIP，也不需要打开 VS Code，更不需要安装 Node.js。

## 第 2 步：确认仓库

打开你自己账号下的 `CloudFlare-ImgBed` 仓库，确认：

- 仓库能够正常打开。
    
- 分支为 `main`。
    
- 仓库中存在项目源码和 `frontend-dist` 目录。
    

当前官方 Pages 部署文档采用 `npm install` 作为构建命令，并指定 `/frontend-dist` 为构建输出目录。 【turn983652view0】

验收标准：你自己的 GitHub 账号中已经有官方项目的 Fork。

# 三、第二阶段：部署 Cloudflare Pages

目标：让 ImgBed 第一次运行起来，并获得一个临时的 pages.dev 域名。

## 第 3 步：进入 Cloudflare 控制台

打开：

[Cloudflare Dashboard](https://dash.cloudflare.com/)

登录你准备长期使用的 Cloudflare 账号。

然后进入：

`Workers & Pages → Create application`

如果你看到的是中文界面，对应的名称可能是“计算和 AI”“Workers 和 Pages”或“创建应用程序”等，Cloudflare 控制台界面可能存在一些差异。

接下来选择 Pages 相关入口，通常是：

`Get started → Start with an existing Git repository`

也就是“使用现有 Git 仓库部署”。

## 第 4 步：连接 GitHub

第一次使用时，Cloudflare 可能会要求你授权 GitHub。

按照页面提示授权，并选择你刚刚 Fork 的 `CloudFlare-ImgBed` 仓库。

随后点击 `Begin setup`，进入项目配置。

## 第 5 步：填写构建参数

这是首次部署最容易出错的地方，请按下面的值填写。

项目名称（Project name）

rcardoshaw-imgbed

复制

生产分支（Production branch）

main

复制

构建命令（Build command）

npm install

复制

构建输出目录（Build output directory）

/frontend-dist

复制

注意两个细节：

- 不要将构建输出目录误填成 `/`。
    
- 不要把构建命令误填成 `npm run build`。当前官方 Pages 部署指南使用的是 `npm install`。 【turn983652view0】
    

填写完成后点击 Save and Deploy。

第一次部署需要等待构建完成。你可以在部署日志中查看执行情况。

## 第 6 步：测试第一次部署

部署成功后，Cloudflare 会给你一个类似下面的地址：

`https://rcardoshaw-imgbed.pages.dev`

这是示例格式，请以 Cloudflare 实际分配的域名为准。

打开项目首页。如果页面加载正常，说明前端已经部署成功。

但是，此时还不能认为整个图床已经可用了，因为 KV 数据库和 R2 存储桶还没有接入。

如果部署失败，先不要修改代码，进入 Pages 项目的 `Deployments`，点击失败的部署记录，查看构建日志。重点检查仓库、分支、构建命令和输出目录。

# 四、第三阶段：创建并绑定 Workers KV

目标：让 ImgBed 有地方保存文件索引和必要的管理数据。

## 第 7 步：创建 KV 命名空间

进入 Cloudflare 控制台，打开：

`Storage & databases → Workers KV`

然后：

1. 点击创建 KV 命名空间或创建实例。
    
2. 命名空间名称填写 `img_url`。
    
3. 点击创建。
    

官方推荐用 `img_url` 作为命名空间名称。实际对程序最关键的，是后续绑定的变量名称必须为 `img_url`。 【turn983652view0】

## 第 8 步：把 KV 绑定到 Pages 项目

返回你的 Pages 项目：

`rcardoshaw-imgbed → Settings → Bindings`

点击添加绑定，选择 `KV namespace`。

填写：

|字段|应填内容|
|---|---|
|Variable name（变量名称）|`img_url`|
|KV namespace（KV 命名空间）|刚刚创建的 `img_url`|

保存。

这里有一个容易混淆的地方：命名空间名称与绑定变量名称是两个概念。 命名空间可以采用其他名称，但变量名称必须是 `img_url`。否则程序可能无法正常读取管理数据，甚至无法进入管理界面。 【turn983652view0】

## 第 9 步：重新部署

KV 绑定完成后，需要重新部署，让绑定在运行环境中生效。

在 Pages 项目的 `Deployments` 页面找到最近的部署记录，点击右侧的 `...`，选择 `Retry deployment` 或界面中相应的重新部署选项。

等待部署成功后再进行下一步。

阶段验收： Pages 项目已经连接官方仓库，并完成 KV 绑定。

# 五、第四阶段：立即设置管理密码

这一步不要拖到整个项目全部做完。

## 第 10 步：打开管理后台

使用你的实际 `pages.dev` 地址，在末尾加上 `/dashboard`。

例如：

`https://rcardoshaw-imgbed.pages.dev/dashboard`

官方文档说明，ImgBed 管理后台默认没有设置管理密码，因此首次进入后必须尽快配置管理员账号和密码。 【turn624852view1】

进入后台后：

1. 打开系统设置中的安全设置。
    
2. 找到管理员认证设置。
    
3. 设置独立、足够强的管理员用户名和密码。
    
4. 保存，并重新登录验证密码确实生效。
    

建议使用密码管理器保存密码，不要直接将真实密码写进公开的 GitHub 仓库或博客。

如果你只打算自己使用图床，还应检查用户认证设置，避免把上传功能不加限制地开放给任何访问者。管理员认证和普通用户认证并不是同一个设置。

# 六、第五阶段：创建 Cloudflare R2 存储桶

目标：建立图片的实际存储位置。你上传的原始图片将保存在 R2，而不是 DediRock VPS 的硬盘中。

## 第 11 步：进入 R2

在 Cloudflare 控制台打开：

[Cloudflare R2](https://dash.cloudflare.com/?to=/:account/r2/overview)

选择 `Create bucket`，创建存储桶。

建议采用简单的英文小写名称，例如：

`rcardoshaw-imgbed`

如果名称不可用，请换一个名称。

创建时的推荐设置：

|配置项|建议|
|---|---|
|Bucket name|`rcardoshaw-imgbed` 或其他可用名称|
|Storage class|Standard|
|数据位置|使用默认选项即可|
|对象公开访问|初期不必单独开启|

如果 Cloudflare 要求先完成 R2 服务订阅或账单设置，按照控制台提示操作。R2 有按月计算的免费用量，但超出额度仍可能收费。 【turn915399search0】

### 为什么先不公开整个存储桶？

因为我们采用的是 ImgBed 管理图片、通过程序读取图片的模式。

只要 ImgBed 能通过 R2 绑定访问存储桶，正常上传和读取图片就不一定需要直接向互联网公开整个 R2 存储桶。

保持存储桶私有，可以避免绕过图床的管理与访问控制。只有当你明确需要直接访问 R2 对象时，再单独研究公开访问与自定义域名。

## 第 12 步：绑定 R2 到 Pages

返回你的 Pages 项目，找到：

`Settings → Bindings`

点击添加绑定，选择 `R2 bucket`。

按照下面填写：

|字段|应填内容|
|---|---|
|Variable name（变量名称）|`img_r2`|
|R2 bucket（R2 存储桶）|刚创建的 `rcardoshaw-imgbed`|

变量名称必须准确填写 `img_r2`，这是 ImgBed 官方部署说明规定的绑定名称。 【turn624852view1】

保存后，重新部署一次，确保最新绑定已经生效。

到目前为止，你已经完成三个核心部分：

- Pages 负责运行 ImgBed。
    
- KV 负责文件索引与管理数据。
    
- R2 负责保存真正的图片文件。
    

不过，还差一步：在 ImgBed 后台配置 R2 存储渠道。

# 七、第六阶段：在 ImgBed 后台配置 R2

## 第 13 步：进入上传设置

重新打开你的管理后台：

`https://你的实际pages.dev域名/dashboard`

然后进入：

`系统设置 → 上传设置`

按照当前管理界面的选项，找到 R2 渠道配置，新增一个 Cloudflare R2 存储渠道。

建议将渠道名称设为：

`R2主存储`

然后确认渠道已启用，并保存配置。

官方指南的操作顺序就是：先将 R2 桶绑定到项目，再进入管理后台配置 R2 渠道。 【turn624852view1】

这里要注意，绑定 R2 和配置 R2 渠道不是同一步。绑定负责让程序具有访问桶的能力，渠道配置则负责告诉 ImgBed 在实际上传时使用哪个存储后端。

## 第 14 步：设为默认上传渠道

在系统设置或上传页面的相应配置中，找到默认上传渠道类型。

将其设为 Cloudflare R2，并选中刚刚创建的 `R2主存储` 渠道。

如果你暂时不使用 Telegram、S3、WebDAV 等其他后端，就不用提前配置这些存储方式。

部分管理字段可能因 ImgBed 版本而变化，不必照搬其他版本的截图填写不确定的参数。

## 第 15 步：上传第一张测试图片

现在先不要急着绑定自定义域名。

打开图床的上传页面，上传一张较小的 JPG 或 PNG 测试图片。

然后分别检查：

1. 上传界面是否提示成功。
    
2. 管理后台的文件列表中是否出现图片。
    
3. R2 控制台的存储桶中是否出现对应对象。
    
4. 点击或复制图片链接，能否正常读取。
    

如果上传成功且 R2 中出现对象，就说明图床的核心数据流已经打通。

# 八、第七阶段：配置你的正式域名

前面所有步骤都可以先用 `pages.dev` 完成。现在再配置域名，能够把故障范围缩小很多。

## 第 16 步：绑定管理域名 `bed.rcardoshaw.de5.net`

进入：

`Workers & Pages → 你的 Pages 项目 → Custom domains`

点击设置自定义域名，填写：

`bed.rcardoshaw.de5.net`

按照 Cloudflare 的提示完成验证和 DNS 配置。

如果这个子域名的 DNS 就由当前 Cloudflare 账号管理，通常可由 Cloudflare 自动添加所需记录。如果 DNS 在其他服务商处管理，则需要到对应 DNS 服务商那里添加 CNAME 记录，目标指向你的 Pages 项目的 `pages.dev` 域名。

即使需要去外部 DNS 服务商添加 CNAME，也要先在 Pages 项目中完成自定义域名关联。Cloudflare 官方明确提示，仅手动增加一条 CNAME、而没有先在 Pages 中添加自定义域名，可能导致访问时出现 522 错误。 【turn884375search0】

绑定完成后，等待状态变为 Active，再测试：

`https://bed.rcardoshaw.de5.net/dashboard`

### 关于你已有的 SSL 证书

如果你说的“SCL 证书绑定”是指 VPS 上已有的 SSL/TLS 证书，那么它不需要直接迁移到 Pages 项目中。Pages 会为成功绑定的自定义域名处理相应的 HTTPS 证书流程。

但证书存在，并不自动说明域名可以绑定到 Cloudflare。真正需要确认的是：你是否有权配置该子域名的 DNS 记录，以及 Cloudflare Pages 是否能完成域名验证。

## 第 17 步：绑定图片访问域名 `img.rcardoshaw.de5.net`

还是进入同一个 Pages 项目的 `Custom domains`，再添加：

`img.rcardoshaw.de5.net`

将它作为同一个 Pages 项目的第二个自定义域名。

如果 DNS 由外部服务商管理，同样需要配置对应的 CNAME 记录。如果你的现有 DNS 服务无法新增这个子域名，先不要强行修改正在使用的 VPS 域名，保留 `pages.dev` 作为可用入口即可。

绑定成功后，先访问：

`https://img.rcardoshaw.de5.net`

确认这个地址能够访问到 ImgBed 应用。

## 第 18 步：让 ImgBed 生成图片访问链接

进入管理后台：

`系统设置 → 网页设置`

找到 默认 URL 前缀（Default URL Prefix）。

当第二个域名已成功绑定并且访问正常后，可以将它设置为：

```
https://img.rcardoshaw.de5.net/file/
```

官方文档规定，默认 URL 前缀用于生成默认文件访问链接和上传 API 返回的 `publicUrl`；它改变的是生成的链接，并不会改变文件本身的实际存储路径。因此必须先保证该地址真正能够访问图片，不能只修改前缀就认为配置完成。 【turn624852view1】

保存后，重新上传一张新图片，检查生成的链接是否使用 `img.rcardoshaw.de5.net`，再直接打开该链接验证。

这一步暂时不要去 R2 存储桶中额外绑定同一个 `img.rcardoshaw.de5.net`。 如果以后想让某个子域名直接访问 R2 对象，而不是访问 ImgBed 的 `/file/` 路径，那属于另一种独立方案，需要单独规划域名和对象 URL。R2 自定义域名有其额外的 Zone 管理要求。 【turn884375search3】

# 九、第八阶段：验收整个项目

到这里，不要仅凭“首页能打开”就判断项目已经完成。请按下面的清单逐项测试。

### 部署验收清单

0/9 项

页面访问

pages.dev、正式管理域名能够正常打开

管理员登录

设置密码后，退出并重新登录成功

图片上传

通过上传界面成功上传一张测试图片

R2 存储

R2 控制台中能找到上传的图片对象

文件管理

管理后台能够看到该文件，并能复制访问链接

外链访问

在无登录状态的浏览器窗口中打开公开图片链接，图片正常显示

图片域名

新生成的链接使用 img 子域名，且实际访问成功

删除验证

删除测试文件后，确认管理列表和存储对象都符合预期

备份准备

记录 KV/R2 绑定、域名和系统配置，另行安排文件备份

复制验收记录

其中，“删除验证”要稍微谨慎：如果图片链接还受浏览器或 CDN 缓存影响，删除后短时间仍可能看到缓存副本。这不一定表示原始图片仍然存在，应分别检查 R2 原始对象、访问响应和缓存行为。

# 十、费用、备份与安全收尾

## 1. 费用应该怎么理解？

根据 Cloudflare 当前的 R2 价格文档，Standard 存储每月的免费用量包括：

|项目|每月免费用量|
|---|---|
|存储空间|10 GB-month|
|A 类操作|100 万次|
|B 类操作|1000 万次|
|公网出站流量|免费|

超过免费额度后，可能按照存储量和请求量等项目收费。 【turn884375search4】

另外，Pages Functions 的动态请求会计入 Workers 相应计划的使用额度，因此不能把整套图床理解为无限量免费。静态资源请求与执行后端函数的请求，计费规则并不相同。 【turn884375search1】

初期建议：

- 先使用默认功能，不开启暂时用不到的 AI 图片识别等功能。
    
- 定期查看 R2 的存储量、请求量和 Cloudflare 账单。
    
- 你的 VPS 不参与这套图床，因此不需要为图床新增 VPS 端口、Nginx 或 Docker 配置。
    

## 2. 正式投入使用前的安全设置

至少做好以下三件事。

第一，确保管理员账号已经设置强密码。第二，检查普通用户认证与上传权限，不要无意中将上传接口开放给所有人。第三，确认 GitHub 仓库里没有明文密码、API Key 或其他敏感信息。

如果以后要配置 ImgBed 的 AI 功能，官方文档还要求为 `AI_CONFIG_SECRET` 设置至少 32 个字符的随机密钥，用于加密保存的供应商 API Key。未使用 AI 功能时，可以把这项留到以后再配置。 【turn624852view1】

## 3. 备份不能只做一半

你最需要保护的是 R2 中的原始图片，以及 KV 中的文件索引和管理数据。

建议正式使用后，定期将重要图片复制到电脑或外置硬盘，并保留一份必要的程序配置记录。仅仅 Fork 了 GitHub 仓库，不能替代图片备份；仅有 R2 也不能替代独立备份。

还要安排一次真实恢复测试：从备份中取出一份图片，确认文件确实能还原并正常打开。

# 十一、遇到问题时，先看这里

|现象|优先检查|
|---|---|
|Pages 构建失败|`main` 分支、`npm install`、`/frontend-dist`、构建日志|
|首页正常，但管理页面异常|KV 绑定变量名是否为 `img_url`，绑定后是否重新部署|
|上传失败|R2 绑定变量名是否为 `img_r2`，R2 渠道是否启用及设为可用|
|上传成功，R2 没有新对象|上传实际选择的渠道是否为 R2，是否查看了正确的桶|
|图片链接 404|链接路径是否正确，图片域名是否仍然绑定到 Pages 项目|
|自定义域名无法访问|DNS 记录、Pages 域名关联状态、HTTPS 证书签发状态|
|R2 自定义域名无法绑定|域名是否满足同一 Cloudflare 账号下的 Zone 管理要求|
|删除图片后仍能打开|检查 R2 原始对象、浏览器缓存和 CDN 缓存|

最后将官方资料保留下来，后续排错以这些文档为准：

- [ImgBed 官方仓库](https://github.com/MarSeventh/CloudFlare-ImgBed)
    
- [ImgBed：Cloudflare Pages 部署说明](https://cfbed.sanyue.de/deployment/pages.html)
    
- [ImgBed：系统设置与存储渠道配置](https://cfbed.sanyue.de/deployment/configuration.html)
    
- [Cloudflare Pages 自定义域名文档](https://developers.cloudflare.com/pages/configuration/custom-domains/)
    
- [Cloudflare R2 价格说明](https://developers.cloudflare.com/r2/pricing/)
    

建议你现在就从第一步开始：先完成 GitHub Fork。 等进入 Cloudflare 的 Pages 构建配置页面时，再根据实际页面逐项填入上面的参数。先拿到可访问的 `pages.dev` 地址，再绑定 KV、R2 和域名，这样遇到问题时更容易确定具体是哪一层出了错。