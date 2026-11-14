# 《掘地生财》官网 · 内测购买与兑换码系统开发计划

> 目标：玩家在官网点击购买 → 前往爱发电付款 → 回到官网领取**由本站系统分配的专属兑换码**。
> 爱发电只负责收款；订单核验、兑换码生成/分配/发放/验证全部由官网后端负责。

---

## 0. 当前状态（2026-11-14）

**已完成：第一阶段「基础官网」**

| 页面 | 文件 | 状态 |
| --- | --- | --- |
| 首页 | `index.html` | 已上线，含图标、特色、进度、公告、内测入口、开发日志 |
| 游戏介绍 | `game.html` | 已上线（背景 / 核心玩法 / 特色 / 进度 / 后续计划） |
| 公告 | `news.html` | 已上线（更新 / 内测通知 / 版本记录 / 已知问题 / 维护） |
| 内测资格 | `beta.html` | 已上线，**购买与自动发码未开放** |
| 玩家中心 | `account.html` | 已上线，**占位说明，无登录无查询** |
| 创作者认证 | `support.html` | 保留，`rel="me"` 与爱发电完整地址不变 |

- 全站纯 HTML/CSS、**零 JavaScript**，托管于 GitHub Pages：`https://ccief.github.io/DigForWealth/`
- 游戏图标：`assets/icon.png`、`assets/icon-original.jpg`、`favicon.ico`、`assets/og-image.png`
- 主题：天空蓝 / 金币黄 / 泥土棕 / 草地绿 / 奶油白，响应式，支持 `prefers-reduced-motion`
- 仓库不含游戏源码、不含任何密钥、不含真实兑换码

**尚未实现（不要误认为已可用）**：订单创建、支付核验、自动发码、兑换码验证、玩家登录、玩家中心查询、管理后台、数据库、Worker。
内测资格页与玩家中心页已明确标注这一点，避免把赞助误认为购买资格。

---

## 1. 爱发电官方能力查证结论

以下结论来自爱发电**官方文档**（2022-11-11 发布，2025 年有更新），不是推测：

- 《爱发电 Webhook 与 API 文档》 <https://afdian.com/p/9c65d9cc617011ed81c352540025c377>
- 官方指南《开发者 API 和 Webhook》 <https://guide.afdian.com/creator/developer>
- 官方汇总《开发者功能汇总：API 与 Webhook、OAuth2、网页嵌入》 <https://afdian.com/p/010ff078177211eca44f52540025c377>
- 开发者后台 <https://ifdian.net/dashboard/dev>

### 1.1 Webhook（支付结果通知）

- 在开发者后台配置通知地址；**每当有订单时**平台会请求该 URL。官方明确说明：「如果服务器异常，可能不保证能及时推送，因此建议结合 API 一起使用」，且「不排除以后在异常时会重复请求，因此建议做幂等逻辑」。
- 请求体结构（官方示例，`data.type` 目前仅为 `order`）：

```json
{
  "ec": 200, "em": "ok",
  "data": {
    "type": "order",
    "order": {
      "out_trade_no": "2021062321XXX1083454010626",
      "custom_order_id": "Steam12345",
      "user_id": "adf397fe83748XXXcee52540025c377",
      "user_private_id": "fdf981fu8f7g891euaceXXX430321c377",
      "plan_id": "a45353328af91XXX052540025c377",
      "month": 1,
      "total_amount": "5.00",
      "show_amount": "5.00",
      "status": 2,
      "remark": "",
      "redeem_id": "",
      "product_type": 0,
      "discount": "0.00",
      "sku_detail": [],
      "address_person": "", "address_phone": "", "address_address": ""
    },
    "sign": "xxxxxxxx"
  }
}
```

- 字段含义（官方字段说明）：`out_trade_no` 订单号、`custom_order_id` 自定义信息、`user_id` 下单用户 ID、`plan_id` 方案 ID（自选时为空）、`month` 赞助月份、`total_amount` 真实付款金额、`show_amount` 显示金额、`status` **2 为交易成功，目前仅推送此类型**、`remark` 留言、`redeem_id` 兑换码 ID、`product_type` 0 常规方案 / 1 售卖方案、`discount` 折扣、`sku_detail` 售卖型号数组。
- **签名（官方 2025-07-01 更新「webhook 增加签名」）**：`sign_str` 由 `order` 中的 `out_trade_no`、`user_id`、`plan_id`、`total_amount` **依次拼接**而成；`sign` 在 `data.sign`，是 base64 编码的 RSA-SHA256 签名，用官方公布的 RSA 公钥验签（`openssl_verify($sign_str, base64_decode($sign), $key, 'SHA256')`）。Cloudflare Workers 可用 WebCrypto 的 `RSASSA-PKCS1-v1_5` + SHA-256 验证。公钥是公开的，可以入库，但要允许轮换。
- **成功响应必须是** `{"ec":200,"em":""}`；不返回 `ec:200` 平台视为回调失败。

> 注意：`sign` 只覆盖四个字段。金额与商品的核验除了验签，还要在业务层比对 `plan_id`、`total_amount`、`status`，并用主动 API 复核，不能只信回调体。

### 1.2 主动查询 API（用于复核与补单）

- 接口：POST `https://ifdian.net/api/open/query-order`
- 请求体：`user_id`、`params`（**JSON 字符串**）、`ts`（秒级时间戳）、`sign`
- 签名：`sign = md5(token + 按 key 排序拼接的 key+value)`，本项目固定四个字段，可简化为
  `sign = md5(token + "params" + params + "ts" + ts + "user_id" + user_id)`（token 不发送，仅参与计算）
- `params` 支持 `page`（按订单创建时间倒序）、`out_trade_no`（指定订单号，多个用英文逗号分隔）、`per_page`（1–100，默认 50）
- 返回 `data.list[]`（**层级与 Webhook 不同**，订单在 `list` 里）、`total_count`、`total_page`
- `ts` 允许 3600 秒偏差；错误码 `400001` params 不全、`400002` 时间过期、`400003` params 不是合法 JSON、`400004` 无有效 token、`400005` 签名失败
- 签名自检接口：POST `https://ifdian.net/api/open/ping`，返回 `ec:200` 即签名通过
- 其它官方接口：`query-sponsor`（赞助者）、`query-plan`（方案）、`query-random-reply`（随机自动回复）、`update-plan-reply`（补货/发码）、`send-msg`（私信，限 10/s 与 1000/h）

> `update-plan-reply` 属于**爱发电自动回复发码**，本项目明确**不使用**：兑换码由本站 D1 生成与分配。

### 1.3 下单页 URL 参数（官方汇总文章）

`https://ifdian.net/order/create?plan_id=...&product_type=0` 支持 `remark`（留言）、`month`（月数）、`custom_order_id`（**自定义订单号**）、`custom_price`（自选金额）。
`custom_order_id` 是把本站订单号带进爱发电订单、再经 Webhook / `query-order` 回传的关键字段 —— 这是订单关联设计的基础。

### 1.4 OAuth2（尚未申请）

- 官方提供 `authorization_code` 模式，需**私信官方申请**，提交：应用名称、应用用途、可信域名、`clientSecret`（不提供则官方随机生成）；官方回给 `clientID` 与 `clientSecret`。申请前建议先完成创作者认证（本项目已完成）。
- 授权：`GET https://ifdian.net/oauth2/authorize?response_type=code&scope=basic&client_id=...&redirect_uri=...&state=...`
- 换 token：`POST https://ifdian.net/api/oauth2/access_token`，form 提交 `grant_type=authorization_code`、`client_id`、`client_secret`、`code`、`redirect_uri`；官方示例返回 `data.user_id`、`data.user_private_id`、`data.name`、`data.avatar`。
- **本项目尚未申请，未拿到 clientID/clientSecret，也没实测过返回字段**。因此不假设它一定可用、不假设一定能拿到稳定玩家标识。

---

## 2. 关键设计决策

| 决策 | 结论 |
| --- | --- |
| 谁发码 | 官网后端（Cloudflare Worker + D1）。爱发电只收款，不用自动回复发码 |
| 订单关联 | 后端生成唯一 `order_no` → 作为 `custom_order_id` 放进爱发电下单 URL → Webhook / `query-order` 回传时按 `custom_order_id` 找回本站订单 |
| 付款认定 | **只认** Webhook RSA 验签通过 + `status=2` + `plan_id`/`total_amount` 匹配 + `query-order` 主动复核。返回官网、手输订单号、前端 `?paid=1`、付款截图一律不算 |
| 可靠性 | Webhook 可能延迟或重复 → 事件表按 `out_trade_no` 去重（幂等）+ 定时/手动按 `out_trade_no` 调 `query-order` 补单 |
| 一单一码 | `orders.id` 与 `redeem_codes.order_id` 唯一约束；发码在事务中用条件更新抢占状态 |
| 兑换码存储 | 后端用 CSPRNG 生成足够长随机串，D1 只存 **SHA-256 哈希 + 盐**，明文只在创建时返回一次 |
| 并发兑换 | `UPDATE ... WHERE status='assigned'` 条件更新 + 检查受影响行数，等价于原子操作 |
| 权限 | 前端不含 Token、不改支付状态；管理员接口独立鉴权；兑换接口限频；关键操作写日志 |
| 密钥 | 全部放 Cloudflare Worker Secrets / `.dev.vars`（本地），仓库 `wrangler.toml` 只放非敏感配置 |

---

## 3. OAuth2 未申请时的账号方案

数据库 `users.afdian_user_id` **允许为空**，先不绑定登录方式。两个候选：

| 方案 | 玩家标识 | 优点 | 缺点 | 前提 |
| --- | --- | --- | --- | --- |
| A. 爱发电 OAuth2 登录 | `user_private_id` / `user_id` | 玩家可随时回来查订单，体验最好 | 需官方审批，返回字段需实测 | 私信申请通过 + 拿到 clientID/clientSecret |
| B. 订单号 + 取货凭证（不依赖 OAuth2） | 本站生成的随机取货凭证 | 无需审批、可立即上线、不依赖第三方 | 玩家需自行保管凭证，换设备易丢 | 后端上线即可 |

**当前选择：先不定，保持字段可空。** 阶段二先把订单与兑换码链路打通，阶段五再决定登录方式；若 A 未获批就用 B，并在公告里说明。无论哪种，玩家只能查自己的数据，管理员账号与玩家权限分离。

---

## 4. D1 数据库结构草案（阶段二落地）

> 下面是**设计草案**，尚未创建迁移文件、尚未建库。阶段二会写成 `worker/migrations/0001_init.sql` 并用 `wrangler d1 migrations apply` 执行。
> 字段按需求文档 + 爱发电实际字段调整；`user_private_id`、`custom_order_id`、`plan_id` 等均为官方支持的字段。

```sql
CREATE TABLE users (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  afdian_user_id  TEXT UNIQUE,          -- 可为空，OAuth2 未申请
  credential_hash TEXT,                -- 方案 B 的取货凭证哈希
  username        TEXT,
  is_admin        INTEGER NOT NULL DEFAULT 0,
  created_at      INTEGER NOT NULL
);

CREATE TABLE orders (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  order_no         TEXT NOT NULL UNIQUE,   -- 本站唯一订单号，作为 custom_order_id
  user_id          INTEGER,               -- 允许为空，待账号方案确定
  product_id       TEXT NOT NULL,
  amount           TEXT NOT NULL,         -- 与 total_amount 比对，字符串避免浮点误差
  payment_status   TEXT NOT NULL DEFAULT 'pending',  -- pending / paid / failed / refunded
  afdian_order_id  TEXT UNIQUE,           -- out_trade_no
  afdian_user_id   TEXT,
  plan_id          TEXT,
  created_at       INTEGER NOT NULL,
  paid_at          INTEGER
);
CREATE UNIQUE INDEX idx_orders_paid_once ON orders(order_no) WHERE payment_status='paid';

CREATE TABLE redeem_codes (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  code_hash    TEXT NOT NULL UNIQUE,     -- 不明文存兑换码
  order_id     INTEGER UNIQUE,           -- 一单一码
  user_id      INTEGER,
  status       TEXT NOT NULL DEFAULT 'unassigned', -- unassigned/assigned/redeemed/disabled/expired
  created_at   INTEGER NOT NULL,
  assigned_at  INTEGER,
  redeemed_at  INTEGER,
  expires_at   INTEGER
);

CREATE TABLE redemption_logs (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  redeem_code_id  INTEGER NOT NULL,
  user_id         INTEGER,
  redeemed_at     INTEGER NOT NULL,
  result          TEXT NOT NULL          -- success / invalid / used / disabled / expired
);

CREATE TABLE webhook_events (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  event_id          TEXT NOT NULL UNIQUE,  -- 用 out_trade_no + 状态做去重键
  order_id          INTEGER,
  payload_hash      TEXT NOT NULL,
  processing_status TEXT NOT NULL,         -- received / verified / processed / rejected / failed
  error_note        TEXT,
  created_at        INTEGER NOT NULL
);

CREATE TABLE admin_logs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  actor       TEXT NOT NULL,
  action      TEXT NOT NULL,
  target      TEXT,
  detail      TEXT,
  created_at  INTEGER NOT NULL
);
```

---

## 5. 需要你提供的配置（现在还没有，不要伪造）

以下值**都不在仓库里**。Cloudflare Worker 用 `wrangler secret put <NAME>` 写入，本地开发放 `worker/.dev.vars`（已加入 `.gitignore`）。

| 名称 | 用途 | 去哪里拿 |
| --- | --- | --- |
| `AFDIAN_USER_ID` | 调用 Open API 的身份 | 爱发电头像菜单 → 开发者，或 <https://ifdian.net/dashboard/dev> |
| `AFDIAN_API_TOKEN` | 计算 API 签名 | 同上，开发者后台生成 **API Token** |
| `AFDIAN_WEBHOOK_PUBLIC_KEY` | 验证 Webhook 的 RSA 签名 | 官方《Webhook 与 API 文档》公布的公钥（公开信息，允许入库，但建议可配置以便轮换） |
| `AFDIAN_PLAN_ID` | 内测资格商品/方案 ID | 爱发电创作者后台 → 方案/商品管理，创建「内测资格 ¥1」后可见；也出现在下单 URL 的 `plan_id` |
| `AFDIAN_SKU_ID` | 售卖型商品的型号 ID（如有） | 同上，`query-plan` 返回的 `skus[].sku_id` |
| `PRODUCT_PRICE` | 商品价格（当前计划 ¥1） | 你自己定；后端配置为准，页面文案同步 |
| `ADMIN_TOKEN` | 管理员后台鉴权 | 自己生成，例如 `openssl rand -hex 32`，**不要写进仓库** |
| `CODE_PEPPER` | 兑换码哈希盐 | 自己生成，同上；一旦上线不要随意更换 |
| `SESSION_SECRET` | 会话/取货凭证签名 | 自己生成，同上 |
| `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` | 爱发电 OAuth2 登录 | **尚未申请**：私信官方，提供应用名称、用途、可信域名、`clientSecret`（可选），官方回给；申请入口见官方文档 |
| `API_BASE_URL` | 官网前端指向的 Worker 地址 | 部署 Worker 后得到（`https://<name>.<subdomain>.workers.dev` 或自定义域名） |
| `D1_DATABASE_ID` | 绑定数据库 | `wrangler d1 create digforwealth` 后写进 `wrangler.toml`（非密钥） |
| 自定义域名（可选） | 官网/API 用自有域名 | 域名注册商购买，在 Cloudflare 与 GitHub Pages 两侧配置 |

**爱发电 Webhook 地址配置**：在 <https://ifdian.net/dashboard/dev> 把通知地址填成 Worker 的 `/api/webhook/afdian`，并用官方 `ping` 接口先验证 API 签名是否正确。

---

## 6. 分阶段计划与验收标准

| 阶段 | 内容 | 验收标准 | 状态 |
| --- | --- | --- | --- |
| 一 · 基础官网 | 首页 / 介绍 / 公告 / 内测购买页 + Pages 部署 | 四页线上 200、图标可见、手机布局正常、认证页 `rel="me"` 保留 | ✅ 已完成 |
| 二 · 后端与数据库 | Worker + D1 + 迁移 + 管理员鉴权 | `ping` 验签通过、迁移执行成功、未授权访问管理接口 401/403 | ⬜ 待开始 |
| 三 · 兑换码系统 | CSPRNG 生成、分配、验证、绑定、状态机 | 哈希存储、唯一约束、并发兑换只成功一次、过期/禁用正确拒绝 | ⬜ 待开始 |
| 四 · 爱发电支付 | 订单创建、`custom_order_id` 关联、Webhook 验签、`query-order` 复核、自动发码 | 重复通知幂等、伪造签名拒绝、金额/商品不符拒绝、未付款拿不到码 | ⬜ 待开始 |
| 五 · 玩家中心 / 后台 | 订单查询、兑换码查看、兑换页、订单/码/日志管理 | 玩家隔离、限频生效、管理员操作留日志 | ⬜ 待开始（依赖账号方案） |
| 六 · 上线测试 | 完整购买、失败、重复通知、重复领取、并发兑换、未付款取码、密钥泄露检查、移动端 | 逐项测试报告，全通过才公告「自动发码已上线」 | ⬜ 待开始 |

每阶段完成后更新 `CHANGELOG.md`、首页开发日志静态副本与 `news.html`，并写测试说明。**未通过验收前，内测资格页保持「购买未开放」状态。**

---

## 7. 安全边界（任何阶段都不得突破）

1. 前端不得保存 `AFDIAN_API_TOKEN`、`OAUTH_CLIENT_SECRET`、`CODE_PEPPER`、`ADMIN_TOKEN`。
2. 前端不得直接写订单状态或兑换码状态；所有写操作走后端。
3. 不能通过改 URL、改请求参数、手输订单号、上传截图领取兑换码。
4. 付款成功只以「Webhook 验签 + 业务字段比对 + `query-order` 复核」为准。
5. 同一 `out_trade_no` 重复通知必须幂等；同一订单只能发一次码。
6. 兑换用条件更新保证原子性，兑换接口限频。
7. 管理接口独立鉴权，玩家接口绝不能返回他人订单或兑换码。
8. 数据库不存兑换码明文；真实兑换码、密钥、`.dev.vars` 永不进入公开仓库。
9. 关键操作（发码、禁用、手动分配、退款、验签失败）写日志。
10. 若爱发电回调不可靠，用 `query-order` 补单，**不要**改成「玩家自己点一下就算付款」。

---

## 8. 部署说明（现状与后续）

**现在（GitHub Pages）**：仓库 `main` 分支根目录即为站点源码，Pages 从 `main` + `/` 发布，`.nojekyll` 保留。推送 `main` 后自动更新 <https://ccief.github.io/DigForWealth/>。

**后续（阶段二起，尚未执行）**：

1. `npm create cloudflare@latest worker` 或手写 `worker/wrangler.toml` + `worker/src/index.ts`
2. `wrangler d1 create digforwealth` → 把 `database_id` 填进 `wrangler.toml`（可入库）
3. `wrangler d1 migrations apply digforwealth --local` 先本地跑通，再 `--remote`
4. `wrangler secret put AFDIAN_API_TOKEN` 等逐项写入；本地用 `worker/.dev.vars`
5. `wrangler deploy` → 得到 Worker 地址 → 填入前端 `API_BASE_URL`
6. 在爱发电开发者后台把 Webhook 地址设为 `https://<worker>/api/webhook/afdian`
7. 如需自定义域名：Cloudflare 侧绑 Worker 路由，GitHub Pages 侧设置 CNAME 与 `CNAME` 文件（域名尚未提供，暂不配置）

> 完整的部署教程、环境变量表和测试报告会在对应阶段落地时补进仓库，不在本阶段提前写「已部署」。
