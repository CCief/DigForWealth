# 掘地生财 · DigForWealth

> 从挖地开始，以富足为目标。

《掘地生财》是 CCief 正在开发的一款**挖矿 + 生活经营**独立游戏：挖矿获取金钱，用金钱购买家具，打造和装饰自己的家园。

## 这个仓库里没有游戏源码

这是本仓库最重要的一条约定：**这里不存放游戏源码，也不发布可下载的游戏本体。**

仓库承担四件事：

| 用途 | 位置 |
| --- | --- |
| 🌐 官方网站 | <https://ccief.github.io/DigForWealth/> · `index.html` 等静态页 |
| 📒 更新日志 | [`CHANGELOG.md`](CHANGELOG.md) · 官网「开发日志」静态副本 |
| ✅ 爱发电认证主页 | [`support.html`](support.html) · <https://afdian.com/a/CCief> |
| 🗺️ 内测购买与发码系统计划 | [`ROADMAP.md`](ROADMAP.md) |

游戏本体处于**企划阶段**，尚无版本、无构建产物、无下载链接。

## 官方网站

**<https://ccief.github.io/DigForWealth/>**（GitHub Pages，`main` 分支根目录直接发布，`.nojekyll` 保留）

| 页面 | 文件 | 内容 |
| --- | --- | --- |
| 首页 | `index.html` | 游戏图标、简介、核心循环、进度与最新公告、内测入口、开发日志 |
| 游戏介绍 | `game.html` | 背景、核心玩法、特色、开发进度、画面说明、后续计划 |
| 公告 | `news.html` | 更新公告、内测通知、版本记录、已知问题、维护通知 |
| 内测资格 | `beta.html` | 商品说明（计划价 ¥1）、购买须知、目标支付流程、当前状态 |
| 玩家中心 | `account.html` | 功能说明占位页，**未开放**，不收集任何信息 |
| 创作者认证 | `support.html` | 提交给爱发电审核的认证专页 |

资源：`assets/site.css`（主题样式）、`assets/icon.png`（游戏图标）、`assets/icon-original.jpg`（图标原图）、`assets/og-image.png`（分享图）、`favicon.ico`。

主题取自游戏图标：天空蓝 `#4fa3e3`、金币黄 `#f5b731`、泥土棕 `#6b4024`、草地绿 `#84b84c`、奶油白 `#fffaf0`。响应式布局，支持 `prefers-reduced-motion`。

**站点为纯 HTML/CSS、零 JavaScript**，内容无需执行脚本即可完整阅读 —— 这对爱发电人工审核与搜索引擎抓取都很重要。后续阶段五的玩家中心/兑换/管理后台会按需引入必要脚本，但内容页与认证专页继续保持零脚本。

## 当前状态（2026-11-14）

**第一阶段「基础官网」已完成**：首页、介绍、公告、内测资格页面上线，图标与新主题落地，多页结构统一，已部署 GitHub Pages。

**尚未上线**：订单创建、支付核验、自动发码、兑换码验证、玩家登录、玩家中心查询、管理后台、Cloudflare Worker 与 D1 数据库。爱发电 OAuth2 也**尚未申请**。

内测资格页与玩家中心页都明确标注了这一点：**现在通过爱发电赞助不会自动获得内测兑换码**。分阶段计划、爱发电官方 API 查证结论、数据库草案、需要申请的凭据与获取位置，全部写在 [`ROADMAP.md`](ROADMAP.md)。

> 原则：爱发电只负责收款，兑换码的生成、分配、发放与验证全部由本站后端负责；付款成功只认官方 Webhook 验签 + 订单查询 API 复核，不认返回参数、手输订单号或付款截图。

## 关于「爱发电认证网站」

爱发电的「认证」不是域名所有权验证，不需要 DNS TXT 记录、meta 验证码或上传验证文件；它是平台工作人员**人工确认「这个主页确实是创作者本人」**的审核。创作者认证已通过。

官方创作者 FAQ 的三条标准与本站落实方式：

1. **完整、有效的主页网址** → <https://ccief.github.io/DigForWealth/> 或 <https://ccief.github.io/DigForWealth/support.html>
2. **主页中公开宣传爱发电** → 顶部公告条、按钮、赞助区、页脚与认证专页均以纯文本写出完整地址 `https://afdian.com/a/CCief`，链接使用真实 `<a href>` 且带 `rel="me"`
3. **主页中有创作内容** → 游戏图标、介绍、玩法草案、公告与持续维护的更新日志

申请入口：<https://afdian.com/apply/certify>。详见 [`support.html`](support.html)。

## 部署与维护

**现在**：编辑根目录静态文件 → 提交并推送 `main` → GitHub Pages 自动更新。无需构建步骤。

**后续（阶段二起，尚未执行）**：Cloudflare Worker + D1，密钥通过 `wrangler secret put` 写入，本地开发用 `worker/.dev.vars`（已在 `.gitignore` 中排除）。完整步骤、环境变量表与凭据获取位置见 [`ROADMAP.md`](ROADMAP.md) 第 5、8 节。

**仓库安全约定**：不提交 API Token、Webhook 私钥配置、`clientSecret`、管理员口令、`.env`/`.dev.vars`、真实兑换码或含兑换码的导出文件；`.gitignore` 已覆盖常见形式。

**维护更新日志**：

1. 编辑根目录 [`CHANGELOG.md`](CHANGELOG.md)，按日期倒序新增条目
2. 同步更新 `index.html` 中 `<!-- CHANGELOG:START -->` 与 `<!-- CHANGELOG:END -->` 之间的静态日志区
3. 有对外影响的变更同步发到 `news.html`

日志区刻意写成静态 HTML 而非 JS 动态渲染，是为了让审核人员与爬虫在不执行脚本的情况下也能读到创作内容。

## 署名与版权

作者：**CCief**。站点文案、更新日志与图标 © 2026 CCief。本仓库未声明任何开源许可，且不包含游戏源码。
