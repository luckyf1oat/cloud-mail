# Cloud Mail 部署文档

本文档说明如何将 Cloud Mail 部署到 Cloudflare Workers，并完成前端、后端、数据库、对象存储和邮件能力的初始化配置。

## 1. 项目架构概览

Cloud Mail 由两个子项目组成：

- `mail-vue`：前端页面，基于 Vue 3 + Vite
- `mail-worker`：后端 Worker，基于 Hono + Cloudflare Workers

部署时采用一体化发布方式：

1. 构建 `mail-vue`
2. 将前端静态文件输出到 `mail-worker/dist`
3. 使用 `mail-worker` 统一发布 API、静态页面、附件访问与邮件处理逻辑

---

## 2. 部署前准备

部署前请准备以下资源。

### 2.1 Cloudflare 资源

必须准备：

- Cloudflare 账号
- 一个已接入 Cloudflare 的域名
- Workers
- D1 数据库
- KV Namespace
- R2 Bucket

可选但推荐准备：

- Cloudflare AI 绑定
- Email Routing
- 自定义 Worker 域名

### 2.2 第三方服务

如果需要发信功能，还需要：

- [Resend](https://resend.com/) 账号
- 已验证的发信域名
- Resend API Key

---

## 3. 仓库结构

```text
cloud-mail/
├─ mail-vue/         # 前端项目
├─ mail-worker/      # Worker 后端项目
├─ doc/              # 文档目录
├─ README.md
└─ README-en.md
```

---

## 4. 部署方式

支持两种方式：

1. GitHub Actions 自动部署
2. 本地 Wrangler 手动部署

推荐优先使用 **GitHub Actions 自动部署**。

---

## 5. GitHub Actions 自动部署

## 5.1 Fork 仓库

先将仓库 Fork 到你自己的 GitHub 账号，或克隆后推送到自己的仓库。

## 5.2 配置 GitHub Secrets

进入仓库：

- `Settings`
- `Secrets and variables`
- `Actions`
- `New repository secret`

添加以下 Secrets：

| Secret 名称 | 必需 | 说明 |
|---|---:|---|
| `CLOUDFLARE_API_TOKEN` | ✅ | Cloudflare API Token |
| `CLOUDFLARE_ACCOUNT_ID` | ✅ | Cloudflare 账户 ID |
| `D1_DATABASE_ID` | ✅ | D1 数据库 ID |
| `KV_NAMESPACE_ID` | ✅ | KV 命名空间 ID |
| `R2_BUCKET_NAME` | ✅ | R2 Bucket 名称 |
| `DOMAIN` | ✅ | 邮件域名，示例：`["example.com"]` |
| `ADMIN` | ✅ | 管理员邮箱，示例：`admin@example.com` |
| `JWT_SECRET` | ✅ | JWT 密钥，建议使用高强度随机字符串 |
| `INIT_URL` | ❌ | 初始化地址，可选 |

### 5.3 获取 Cloudflare API Token

1. 打开 Cloudflare Dashboard
2. 进入 API Tokens 页面
3. 创建新 Token
4. 赋予 Worker 和相关资源操作权限
5. 将生成的 Token 填入 `CLOUDFLARE_API_TOKEN`

建议至少具备以下资源权限：

- Workers
- D1
- KV
- R2

### 5.4 获取 Cloudflare Account ID

在 Cloudflare 控制台中找到你的账户 ID，并填入 `CLOUDFLARE_ACCOUNT_ID`。

### 5.5 运行 GitHub Action

进入仓库 Actions 页面，手动运行工作流。

部署成功后：

- Worker 会自动发布
- 前端会自动构建并打包进 Worker 静态资源中

### 5.6 初始化系统

如果没有配置 `INIT_URL`，则需要手动执行初始化：

```text
https://你的域名/api/init/你的jwt_secret
```

示例：

```text
https://mail.example.com/api/init/your_jwt_secret
```

初始化完成后，再访问首页：

```text
https://你的域名/
```

---

## 6. 本地 Wrangler 手动部署

## 6.1 安装依赖

在项目根目录执行：

```cmd
pnpm --prefix mail-vue install
pnpm --prefix mail-worker install
```

如果你的环境未安装 `pnpm`，请先安装。

## 6.2 登录 Cloudflare

```cmd
pnpm --prefix mail-worker wrangler login
```

登录成功后，Wrangler 才能进行本地部署。

## 6.3 修改 `mail-worker/wrangler.toml`

需要将注释掉的资源绑定补全。

参考配置：

```toml
name = "cloud-mail"
main = "src/index.js"
compatibility_date = "2025-06-04"
keep_vars = true

[observability]
enabled = true

[[d1_databases]]
binding = "db"
database_name = "你的D1数据库名"
database_id = "你的D1数据库ID"

[[kv_namespaces]]
binding = "kv"
id = "你的KV命名空间ID"

[[r2_buckets]]
binding = "r2"
bucket_name = "你的R2桶名"

[[send_email]]
name = "email"

[ai]
binding = "ai"

[assets]
binding = "assets"
directory = "./dist"
not_found_handling = "single-page-application"
run_worker_first = true

[triggers]
crons = ["0 16 * * *"]

[vars]
domain = ["example.com"]
admin = "admin@example.com"
jwt_secret = "请替换为安全的随机长字符串"

[build]
command = "pnpm --prefix ../mail-vue install && pnpm --prefix ../mail-vue run build"
```

### 重要说明

以下绑定名建议不要修改：

- `db`
- `kv`
- `r2`
- `ai`
- `assets`

因为后端代码默认使用这些名字读取 Cloudflare 资源。

## 6.4 前端构建

前端生产环境配置位于：

- `mail-vue/.env.release`

内容如下：

```env
NODE_ENV = 'release'
VITE_APP_TITLE = '发布环境'
VITE_BASE_URL = '/api'
VITE_PWA_NAME = 'Cloud Mail'
VITE_OUT_DIR = ../mail-worker/dist
```

执行构建：

```cmd
pnpm --prefix mail-vue run build
```

构建结果会输出到：

```text
mail-worker/dist
```

## 6.5 部署 Worker

执行：

```cmd
pnpm --prefix mail-worker run deploy
```

也可以直接依赖 `wrangler.toml` 中的构建命令，让部署时自动构建前端。

## 6.6 初始化系统

部署后访问：

```text
https://你的域名/api/init/你的jwt_secret
```

初始化完成后访问首页：

```text
https://你的域名/
```

---

## 7. Cloudflare 资源创建建议顺序

建议按以下顺序准备资源：

1. 创建 D1 数据库
2. 创建 KV Namespace
3. 创建 R2 Bucket
4. 创建或启用 Worker
5. 将域名接入 Cloudflare
6. 配置自定义域名或路由
7. 配置 Email Routing
8. 配置 Resend 发信域名和 API Key

---

## 8. 邮件能力配置

Cloud Mail 的核心能力包括收信与发信，因此部署成功后还需要额外配置邮件相关能力。

## 8.1 收信配置

项目中 `mail-worker/src/index.js` 暴露了 `email` 处理器，因此如果要支持真正的邮件接收，需要在 Cloudflare 侧完成 Email Routing 或相应邮件事件配置，并将收信事件交给这个 Worker 处理。

通常需要：

- 将域名邮箱解析接入 Cloudflare
- 配置 Email Routing
- 将目标路由绑定到当前 Worker

## 8.2 发信配置

项目 README 说明发信依赖 Resend，因此你需要：

1. 在 Resend 注册账号
2. 验证发信域名
3. 获取 API Key
4. 将相关密钥配置到部署环境中

如果未配置 Resend，则发送邮件相关功能无法正常工作。

---

## 9. 项目访问路径

部署后通常有两类访问地址。

### 9.1 前端页面

```text
https://你的域名/
```

### 9.2 后端 API

```text
https://你的域名/api/
```

例如：

- 登录接口：`/api/...`
- 初始化接口：`/api/init/你的jwt_secret`

---

## 10. 常见问题

## 10.1 页面能打开但接口 404

请检查：

- `mail-vue/.env.release` 中 `VITE_BASE_URL` 是否为 `/api`
- Worker 是否正确处理 `/api/*`
- 自定义域名是否正确绑定到 Worker

## 10.2 前端资源未更新

请检查：

- `mail-vue` 是否已成功构建
- `mail-worker/dist` 是否存在最新构建产物
- 是否重新执行了 Worker 部署

## 10.3 初始化接口访问失败

请检查：

- 路径是否正确：`/api/init/你的jwt_secret`
- `jwt_secret` 是否与 Worker 中配置一致
- Worker 是否已成功部署

## 10.4 邮件发送失败

请检查：

- Resend 是否已正确配置
- 发信域名是否已验证
- API Key 是否注入到运行环境

## 10.5 附件上传或下载失败

请检查：

- R2 Bucket 是否创建成功
- `wrangler.toml` 中 `r2` 绑定是否正确
- 权限是否正常

---

## 11. 建议的生产部署清单

上线前建议确认以下内容：

- [ ] Cloudflare 账号与域名已准备
- [ ] D1 已创建并记录 ID
- [ ] KV 已创建并记录 ID
- [ ] R2 已创建并记录 Bucket 名称
- [ ] Worker 已启用
- [ ] `DOMAIN` 配置正确
- [ ] `ADMIN` 配置正确
- [ ] `JWT_SECRET` 已替换为强随机字符串
- [ ] 前端已成功构建
- [ ] Worker 已成功部署
- [ ] 初始化接口已执行
- [ ] Resend 已配置
- [ ] Email Routing 已配置
- [ ] 首页与 API 均可正常访问

---

## 12. 最简部署流程

如果你只需要一个最短可用流程，可按以下步骤执行：

1. Fork 仓库
2. 在 Cloudflare 创建 D1、KV、R2
3. 配置域名
4. 配置 GitHub Secrets
5. 运行 GitHub Actions
6. 访问初始化接口
7. 访问首页验证系统是否可用

---

## 13. 相关文件参考

部署时可重点查看这些文件：

- `README.md`
- `doc/github-action.md`
- `mail-worker/wrangler.toml`
- `mail-worker/package.json`
- `mail-vue/package.json`
- `mail-vue/.env.release`

---

## 14. 总结

Cloud Mail 的部署核心是：

- 使用 Cloudflare Workers 承载后端与静态资源
- 使用 D1、KV、R2 提供数据库、缓存与对象存储
- 使用 Email Routing 和 Resend 提供邮件收发能力
- 使用初始化接口完成系统首次安装

推荐使用 **GitHub Actions 自动部署**，维护成本最低；需要更高灵活性时，再使用 **Wrangler 本地手动部署**。