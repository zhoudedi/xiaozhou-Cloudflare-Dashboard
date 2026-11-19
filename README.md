# xiaozhou-Cloudflare-Dashboard

> 一个跑在 **Cloudflare Workers** 上的自托管 Cloudflare 管理面板，二开自 [`fangovo/cloudflare-Management`](https://github.com/fangovo/cloudflare-Management) v2.9.0（MIT）。
>
> 整个后端 + 前端只有**一个 `.js` 文件**，部署后即可管理自己账号下的 Workers / KV / D1 / R2 / Pages / DNS / 隧道，并带用量统计。所有 Cloudflare API 请求都由 Worker 后端代理，浏览器不直连 `api.cloudflare.com`，无 CORS 问题。

**本实例访问地址**：<https://cfkb.fuxicun.top>（访问密码由部署者设置为 Secret `ACCESS_PASSWORD`）

---

## 目录

- [与上游的差异](#与上游的差异)
- [核心功能](#核心功能)
- [部署架构](#部署架构)
- [部署前须知](#部署前须知)
- [控制台部署（推荐，全程网页操作）](#控制台部署推荐全程网页操作)
- [配置项详解](#配置项详解)
- [wrangler.toml 配置说明](#wranglertoml-配置说明)
- [GitHub 自动构建说明](#github-自动构建说明)
- [OAuth 客户端配置](#oauth-客户端配置)
- [命令行部署（进阶）](#命令行部署进阶)
- [安全模型](#安全模型)
- [常见问题](#常见问题)
- [已知限制](#已知限制)
- [测试](#测试)
- [版权与致谢](#版权与致谢)

---

## 与上游的差异

本仓库是 `fangovo/cloudflare-Management` v2.9.0 的个人二开版，主要变化：

| 项目 | 上游 | 本仓库 |
| --- | --- | --- |
| 仓库地址 | `fangovo/cloudflare-Management` | `zhoudedi/xiaozhou-Cloudflare-Dashboard` |
| Worker 名称 | `cf-panel` | `xiaozhou-cloudflare-dashboard` |
| 访问域名 | 随机 `workers.dev` 子域 | 自定义域 `cfkb.fuxicun.top` |
| 构建方式 | 手动 `wrangler deploy` | Cloudflare 后台连接 GitHub `main` 分支自动构建部署 |
| `OAUTH_CLIENT_ID` | 需自行注册填写 | 已内置预置 ID（32 位 hex），开箱即用 |
| `CF_ACCOUNTS_KV` | 可选 | 已创建并绑定，命名空间名 `xiaozhou-Cloudflare-Dashboard-kv` |
| 密码 | 自行设置 | 通过加密 Secret `ACCESS_PASSWORD` 设置，**不入库** |

> 核心逻辑（`cf-panel-worker-v2.9.0.js`）保持与上游一致，仅通过配置层完成个性化，便于跟随上游升级。

---

## 核心功能

### Workers
- 列表、部署 / 编辑 / 删除，经典脚本与 ES Module 都支持
- workers.dev 子域名开关、账号级自定义域名（绑定 / 解绑 / 列表）
- 环境变量（文本 / 密钥 / JSON）、Secrets 批量增删改
- KV / D1 / R2 绑定，兼容性日期与兼容性标志
- Cron 定时触发器、Worker 路由（Route）管理
- 版本列表与回滚（Versions API）

### Pages
- 项目管理：创建 / 列表 / 删除，同名项目自动转为更新
- 部署记录：列表 / 触发 / 重试 / 回滚 / 删除
- 代码来源：文件上传 / 整文件夹上传 / ZIP 直链 / GitHub 仓库 / GitHub 单文件链接
- 绑定：环境变量 / KV / D1，生产 + 预览分别生效
- 自定义域名：Cloudflare 托管域名自动补 CNAME，外部域名给出配置提示

### 隧道（Cloudflare Tunnel）
- 新建 / 列表 / 详情 / 删除
- 建隧道时自动写 ingress 规则 + 自动创建 CNAME 解析，失败自动回滚

### 批量创建
- 多账号批量部署 Worker / Pages
- 内置模板存于 KV（`CF_ACCOUNTS_KV`），也支持直链 / ZIP / 粘贴代码
- 自动查找或创建 KV / D1 并绑定

### KV / D1 / R2 / DNS
- **KV**：命名空间 CRUD、键分页与前缀搜索、单键读写删（TTL）、批量写删
- **D1**：创建 / 列表 / 删除、参数化查询、Raw 模式、导出备份
- **R2**：存储桶管理、对象列表与上传删除、预签名下载链接、临时 S3 凭证
- **DNS**：Zone 管理、记录 CRUD 与批量导入、Zone 分析

### 用量与鉴权
- KV / D1 / R2 用量卡、Workers 今日请求 / 排行 / 24 小时时间序列
- Token 权限自检（7 项探针）
- 鉴权三选一：OAuth 2.0 + PKCE（推荐）、API Token、Global API Key

---

## 部署架构

```
浏览器 ──HTTPS(cfkb.fuxicun.top)──> Worker（本面板）──HTTPS──> api.cloudflare.com
                                      │
                                      ├─ 校验访问密码 / 会话
                                      ├─ 注入调用方的 API Token / Global Key / OAuth token
                                      └─ 返回 JSON 给内嵌前端

GitHub main 分支 push ──> Cloudflare Builds（连接 Git）──> 自动 npx wrangler deploy ──> 更新 Worker
```

要点：
- **前端、后端同文件**：无需单独托管前端，无需构建前端产物。
- **凭据只在浏览器**：API Token / OAuth token 只存浏览器 `localStorage`，Worker 不落库、不写日志。
- **CI 自动部署**：GitHub 仓库 `main` 分支任何 push，都会触发 Cloudflare 后台自动执行 `wrangler deploy`。

---

## 部署前须知

- 需要：一个 **Cloudflare 账号**（本项目使用 `Zdd@fuxicun.top`，Account ID `75eb60de6e66a294d20e38e000c986a7`）+ 一个 **GitHub 账号**（`zhoudedi`）。
- 域名 `fuxicun.top` 已托管在 Cloudflare（Zone ID `39cb8a67b43e8b8eb488b1dd1ee48353`），自定义域名 `cfkb.fuxicun.top` 生效后由 Cloudflare 自动签发 SSL 证书。
- **密钥不入仓库**：本仓库为公开仓库，访问密码 `ACCESS_PASSWORD` **绝不写进任何代码 / 配置 / 文档**，一律在 Cloudflare 后台以「加密 Secret」设置。仓库里只有「怎么设」的说明，没有值本身。
- 预置的 `OAUTH_CLIENT_ID`（`d2a54c9a8c235bdd733c4d65671a3904`）是 32 位公开标识，不是密钥，可入库；如需换成你自己的 OAuth 应用，见 [OAuth 客户端配置](#oauth-客户端配置)。

---

## 控制台部署（推荐，全程网页操作）

适合从零复刻本实例。已部署好的实例可跳过，直接看[配置项详解](#配置项详解)。

### 第 1 步：创建 KV 命名空间（批量创建模板存储）

1. Cloudflare Dashboard → 左侧 **Workers 和 Pages → KV**（顶部搜索框输入 `KV` 也能找到）。
2. **Create Namespace**，名称填：`xiaozhou-Cloudflare-Dashboard-kv`。
3. 创建后复制 **Namespace ID**（形如 `1319d93193094c5c9b8fd42027b7b999`），下一步要填进 `wrangler.toml`。
4. 本仓库已预填该 ID；若你在别的账号复刻，需把 `wrangler.toml` 里 `[[kv_namespaces]].id` 换成你自己的。

> 这个 KV 只用于「批量创建」页存放内置模板，不绑也能用，但绑上功能更完整。

### 第 2 步：fork / 克隆仓库并确认配置

1. 在 GitHub 打开 [`zhoudedi/xiaozhou-Cloudflare-Dashboard`](https://github.com/zhoudedi/xiaozhou-Cloudflare-Dashboard)，点右上角 **Fork** 到你自己的账号（或按[命令行部署](#命令行部署进阶)克隆）。
2. 打开 `wrangler.toml`，核对以下三项（本仓库已填好，复刻到自己账号时才需改）：
   - `name = "xiaozhou-cloudflare-dashboard"` —— Worker 名，需与下一步创建的 Worker 名一致。
   - `[vars].OAUTH_CLIENT_ID` —— 预置 OAuth 客户端 ID。
   - `[[kv_namespaces]].id` —— 第 1 步复制的 KV Namespace ID。
3. 提交改动。**必须先提交，再做第 4 步连接 Git 自动部署。**

### 第 3 步：创建 Worker

1. Dashboard → **Workers 和 Pages → 创建 → Worker**。
2. 名称填 `xiaozhou-cloudflare-dashboard`（与 `wrangler.toml` 的 `name` 完全一致）。
3. 创建时若让选模板 / 写初始代码，**随便填或留空即可** —— 第 4 步接 Git 后，仓库代码会部署上去覆盖掉，初始模板不影响最终结果。
4. 创建后先不做别的，下一步接 Git。

### 第 4 步：连接 GitHub 自动构建（核心）

进入刚创建的 Worker → **Settings → Builds → Connect to Git**，按顺序填：

**① 连接到仓库**

| 界面项 | 填什么 |
| --- | --- |
| Git 帐户 | 选你授权 GitHub 时登录的账号（`zhoudedi`） |
| 存储库 | `xiaozhou-Cloudflare-Dashboard` |
| 分支 | `main` |

**② 构建配置**

| 界面项 | Cloudflare 默认值 | 本项目怎么处理 | 说明 |
| --- | --- | --- | --- |
| 构建命令（Build command） | `npm run build` | 保持默认即可，也可留空 | 单文件 Worker 无需真正构建；本仓库 `package.json` 已内置空的 `build` 脚本，即使保留默认也会正常通过 |
| 部署命令（Deploy command） | `npx wrangler deploy` | **保持不变** | 这就是部署 Worker 的正确命令 |
| 根目录（Root directory） | `/` | 保持 `/` | `wrangler.toml` 在仓库根目录 |

> ⚠️ 「启用预览部署（Enable preview deployments）」默认开启，会用 `npx wrangler preview` 为 PR 生成预览。本项目不需要预览环境，建议**关掉**避免预览任务报错；开着也不影响 `main` 分支的正式部署。

**③ 保存并验证**

点保存后 Cloudflare 会立即触发第一次构建部署（约 1–3 分钟）。以后往 `main` push 任何改动，都会自动重新部署。

### 第 5 步：设置访问密码（加密 Secret）

进入 Worker → **Settings → Variables and Secrets**：

- 点 **Add → Secret**（注意选 Secret，不是 Variable）。
- 变量名填 `ACCESS_PASSWORD`，值填你设定的访问密码。
- 点 Save/Deploy。

> ⚠️ **改完 Secret 必须触发一次部署才生效**：往 GitHub `main` push 任意改动（CI 自动部署），或本地 `npx wrangler deploy`。单纯在后台改值不部署，运行中的 Worker 仍是旧值。
>
> 为什么用 Secret 而不是明文 Variable：本仓库已接入 CI，每次 `wrangler deploy` 会按 `wrangler.toml` 全量同步、清掉未写进 `[vars]` 的明文 Variable；而加密 Secret 不受影响、不进 git，公开仓库下最安全。

### 第 6 步：绑定自定义域名

进入 Worker → **Settings → Domains & Routes → Add → Custom Domain**：

1. 填入 `cfkb.fuxicun.top`。
2. 因 `fuxicun.top` 已托管在本 Cloudflare 账号，Cloudflare 会自动配置 DNS（CNAME）并签发 SSL 证书。
3. 等待证书签发（通常几分钟）。

> 若域名不在当前 Cloudflare 账号，需按提示在你的 DNS 服务商手动添加 `CNAME` 记录指向 Cloudflare 提供的地址。

### 第 7 步：验证

1. 浏览器打开 <https://cfkb.fuxicun.top>，应看到登录页。
2. 输入第 5 步设置的 `ACCESS_PASSWORD` 登录。
3. 登录后即可看到 Workers / KV / D1 / R2 / Pages / DNS / 隧道等管理页面。

---

## 配置项详解

| 名称 | 类型 | 必填 | 本实例值 | 说明 |
| --- | --- | --- | --- | --- |
| `ACCESS_PASSWORD` | Secret | **建议必设** | （部署者自设，不入库） | 面板访问密码。**不设置则 `/api` 完全无鉴权**，任何人都可管理你的 Cloudflare 账号 |
| `OAUTH_CLIENT_ID` | Variable（`[vars]`） | 否 | `d2a54c9a8c235bdd733c4d65671a3904` | 预置 OAuth 客户端 ID（32 位 hex），免去每个浏览器手动填写。公开标识，非密钥 |
| `CF_ACCOUNTS_KV` | KV 绑定 | 否 | 命名空间 `xiaozhou-Cloudflare-Dashboard-kv`，ID `1319d93193094c5c9b8fd42027b7b999` | 「批量创建」页的内置模板存储 |
| `SESSIONS` | KV 绑定 | 否 | 未启用 | 会话落库，可精确吊销；不绑也能用（无状态签名 + 7 天自动过期） |

**鉴权说明**：登录面板后，还需要在面板内选择一种方式管理 Cloudflare 资源：

- **OAuth 2.0 + PKCE（推荐）**：用预置的 `OAUTH_CLIENT_ID`，点击登录授权一次即可。应用注册与域名验证见 [OAUTH.md](./OAUTH.md)。
- **API Token**：在面板内填入你的 Cloudflare API Token（需具备 Workers/KV/D1/R2/Pages/DNS 编辑权限，详见上游 README「需要哪些权限」）。Token 只存浏览器 `localStorage`。
- **Global API Key**：旧版兼容，账号全权限，不建议长期使用。

---

## wrangler.toml 配置说明

```toml
# ① Worker 名称（全局唯一，也是 CF 控制台项目名）
name = "xiaozhou-cloudflare-dashboard"

# ② 入口文件：前端 + 后端全在这一个文件
main = "cf-panel-worker-v2.9.0.js"

# ③ 运行时行为版本（像 Node LTS），保持即可，别随意升 "latest"
compatibility_date = "2026-09-01"
compatibility_flags = ["nodejs_compat"]

# ④ 预置 OAuth Client ID（非敏感，可入库）
[vars]
OAUTH_CLIENT_ID = "d2a54c9a8c235bdd733c4d65671a3904"

# ⑤ KV 绑定：代码里用 env.CF_ACCOUNTS_KV 访问
[[kv_namespaces]]
binding = "CF_ACCOUNTS_KV"
id = "1319d93193094c5c9b8fd42027b7b999"
```

| 字段 | 作用 | 注意事项 |
| --- | --- | --- |
| `name` | Worker 标识 | 全局唯一；改它要同步改 CF 控制台项目名 |
| `main` | HTTP 请求进来执行的文件 | 本项目的全部前端 + 后端逻辑 |
| `compatibility_date` | 锁定运行时行为版本 | 升级 Cloudflare 新特性时才改 |
| `compatibility_flags` | 兼容性标志 | `nodejs_compat` 必须保留，代码依赖 Node 兼容层 |
| `[vars].OAUTH_CLIENT_ID` | 预置 OAuth 客户端 ID | 非敏感；若删掉这段，浏览器需手动填 Client ID |
| `[[kv_namespaces]].binding` | 代码里的 KV 句柄名 | 与代码引用一致，**不要改** |
| `[[kv_namespaces]].id` | 连到哪个 KV 命名空间 | fork 到别的账号后**必须改**成自己的 Namespace ID |

> ⚠️ 敏感值（`ACCESS_PASSWORD`）**绝不写进本文件**。仓库公开，写进去即泄露。它在 CF 后台 → Worker → Settings → Encrypted Secrets 设置，不进 git、部署不会被清掉。

---

## GitHub 自动构建说明

本实例采用 **Cloudflare Builds 连接 Git**（而非 GitHub Actions），与参考项目 `xiaozhou-ledger` 一致：

- **触发条件**：`zhoudedi/xiaozhou-Cloudflare-Dashboard` 的 `main` 分支发生 push。
- **执行流程**：Cloudflare 拉取仓库 → 执行构建命令（本项目为空操作）→ 执行 `npx wrangler deploy` → 按 `wrangler.toml` 更新 Worker。
- **配置位置**：Cloudflare Dashboard → Worker → Settings → Builds（不在仓库的 `.github/workflows` 里）。

因此：
1. 修改 `wrangler.toml` / worker 代码后，**push 到 `main` 即自动上线**，无需手动部署。
2. 在后台改了 Secret / Variable 后，**需要 push 一次或手动部署才生效**。
3. 明文 Variable 必须写进 `wrangler.toml [vars]`，否则每次 CI 部署会被清掉；Secret 不受影响。

> 若你更习惯 GitHub Actions，也可在 `.github/workflows/deploy.yml` 里用 `cloudflare/wrangler-action` + `CLOUDFLARE_API_TOKEN` Secret 自行实现；本仓库默认不采用，以避免两套 CI 同时部署。

---

## OAuth 客户端配置

本仓库已内置 `OAUTH_CLIENT_ID = d2a54c9a8c235bdd733c4d65671a3904`（32 位 hex 公开标识），配合 OAuth 2.0 + PKCE 使用，开箱即用。

如果要换成你自己的 OAuth 应用（例如改了域名），完整步骤见 **[OAUTH.md](./OAUTH.md)**，摘要：

1. Cloudflare Dashboard → **My Profile → API Tokens → OAuth Applications → 创建**。
2. 应用名随意（如 `cf-manager-v2`），类型选 **Public**，回调 URL 填 `https://cfkb.fuxicun.top/oauth/callback`（改成你实际域名）。
3. Scope 全选（32/32，或按需选 Workers / D1 / DNS / Analytics 等）。
4. 拿到 Client ID 后，替换 `wrangler.toml` 的 `[vars].OAUTH_CLIENT_ID`，push 触发自动部署。
5. 按 OAuth.md 完成域名验证（TXT 记录 `cloudflare_oauth_client_publisher`）。

> OAuth 登录较繁琐，日常自用也可以直接在面板里用 **API Token** 方式，一个 Token 即可，无需注册 OAuth 应用。

---

## 命令行部署（进阶）

习惯 CLI 的话，可跳过控制台的连接 Git 步骤：

```bash
# 1. 克隆仓库
git clone https://github.com/zhoudedi/xiaozhou-Cloudflare-Dashboard.git
cd xiaozhou-cloudflare-dashboard

# 2. 安装依赖（可选，wrangler 用 npx 拉取即可）
npm install

# 3. 登录 Cloudflare（浏览器授权）
npx wrangler login
# 或用 API Token：export CLOUDFLARE_API_TOKEN=<你的 token>

# 4. 设置访问密码（交互式输入，密文不回显）
npx wrangler secret put ACCESS_PASSWORD

# 5. 部署（KV 绑定、OAuth ID 均从 wrangler.toml 读取）
npx wrangler deploy
```

部署后绑定自定义域名仍需到 Dashboard → Worker → Settings → Domains & Routes 添加（wrangler 不直接管理自定义域名）。

验证：

```bash
curl https://cfkb.fuxicun.top/    # 应返回登录页 HTML
```

---

## 安全模型

面板管理的是**别人账号的全权限凭据**，安全设计要点：

- **凭据存放**：API Token / Global Key / OAuth token 只存在浏览器 `localStorage`，每次请求发给同源 Worker，Worker 不落库、不写日志。但**任何一个 XSS 都可能偷走凭据**，故加了 CSP（禁外部脚本、禁 iframe 嵌套）。
- **访问密码**：常量时间比较；同一 IP 5 分钟内失败 10 次临时拒绝；会话令牌 `v1.<过期时间>.<随机ID>.<HMAC>` 每次登录不同、7 天过期，绑定 `SESSIONS` 后可吊销；改密码使全部会话失效。
- **响应头**：`Content-Security-Policy`、`X-Content-Type-Options`、`X-Frame-Options`、`Referrer-Policy`、`Permissions-Policy`；API 响应统一 `Cache-Control: no-store`。
- **建议再加一层**：Cloudflare Access（Zero Trust）限制来源 IP / 邮箱，或至少把面板挂在自己域名下、不公开传播地址。

---

## 常见问题

**Q：打开 cfkb.fuxicun.top 提示 521 / 无法访问？**
A：确认 Worker 已部署成功（Builds 页面看最近一次构建状态）、自定义域名已绑定且 SSL 证书已签发。改了后台配置后需 push 或手动部署一次。

**Q：登录页能打开，但填了 API Token 后功能报错？**
A：检查 Token 权限是否齐全（Workers/KV/D1/R2/Pages/DNS 编辑 + Zone/Analytics 读取），可用面板内「Token 权限自检」探针排查。

**Q：改了访问密码不生效？**
A：Secret 改动必须触发一次部署。往 `main` push 任意改动，或本地 `npx wrangler deploy`。

**Q：`OAUTH_CLIENT_ID` 需要保密吗？**
A：不需要。它是 OAuth 应用的公开标识（类似用户名），不是密钥，可安全写入仓库。

**Q：为什么后台设的明文 Variable 部署后不见了？**
A：CI 的 `wrangler deploy` 会按 `wrangler.toml` 全量同步，清掉未写进 `[vars]` 的明文 Variable。非敏感变量请写进 `[vars]`；敏感值一律用 Secret。

**Q：免费版够用吗？**
A：个人自用基本够，但 `list-workers` 需要 `1 + 3N` 次子请求，Worker 超过约 16 个会触发免费版单请求 50 次子请求上限；Pages 大批量上传也可能超限。重度使用建议 Workers Paid 计划。

---

## 已知限制

- **子请求数**：免费版单请求最多 50 次子请求，Worker 数量多时列表接口会失败，建议 Paid 计划。
- **CPU 时间**：免费版 10 ms 偏紧（Pages 上传要算哈希 / base64），真实使用建议 Paid。
- **多模块 Worker**：面板无法安全回传 ESM 多模块代码，保存变量时会明确拒绝。
- Versions API 为 Beta，权限不足时会降级提示。
- R2 对象操作走 S3 兼容接口，需 S3 凭证（可在面板内一键换取临时凭证）。
- DNS 记录不能修改「类型」（Cloudflare 2026-06-30 起的行为），需删除重建。
- 单文件发布意味着改前端也要改这个文件；跟随上游升级时替换 `cf-panel-worker-v2.9.0.js` 即可，配置层无需改动。

---

## 测试

仓库自带离线测试，**不需要 Token、不连网、不会碰 Cloudflare 账号**：

```bash
node tests/run.mjs
```

覆盖 50+ 个 action：部署 / 变量 / 密钥 / 版本 / 路由 / 域名、KV、D1、R2、DNS / Zone、Pages、隧道、用量统计、鉴权与安全头、SSRF 防护、ZIP 路径穿越等。

---

## 版权与致谢

- 上游项目：[`fangovo/cloudflare-Management`](https://github.com/fangovo/cloudflare-Management)，MIT License，感谢作者开源。
- 隧道功能思路参考 [`eooce/Cloudflare-Tunnel-Panel`](https://github.com/eooce/Cloudflare-Tunnel-Panel)。
- 本二开版仅做配置层个性化与文档完善，核心逻辑版权归上游作者所有，遵循 MIT 协议。
