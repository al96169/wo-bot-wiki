# 用户帐号服务器部署指南

> 子项目：`wo-bot-account` | Logto 官方镜像 + NestJS 设备管理 API + Vue 3 前端
> 最后更新：2026-07-19

## 架构概览

wo-bot-account 由五个服务组成，通过 Caddy 反向代理三域名架构对外提供服务：

| 服务 | 技术栈 | 端口 | 职责 |
|------|--------|------|------|
| Caddy | Go | 80/443 | HTTPS 反向代理，三域名路由，自动 TLS 证书 |
| Logto | 官方镜像 `ghcr.io/logto-io/logto` | 3001(Core) / 3002(Admin) | 用户注册/登录/OAuth2 授权/管理后台 |
| Device API | NestJS | 3002 容器内 | 设备绑定/解绑/重命名/状态查询 |
| PostgreSQL | 16 | 5432 | 数据持久化（Logto + 设备库） |
| Redis | 7 | 6379 | 会话缓存 / 速率限制 |

前端（wo-bot-web）为 Vue 3 SPA，构建后由 Caddy 直接托管静态文件。

### 三域名架构

```
                      ┌─────────── wo-bot.com ───────────┐
                      │                                   │
          user.wo-bot.com    api.wo-bot.com    admin.wo-bot.com
          (用户侧页面)        (管理 API)         (管理控制台)
               │                  │                  │
               ▼                  ▼                  ▼
       ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
       │ wo-bot-web   │  │ Device API   │  │ Logto Admin      │
       │ (静态文件)    │  │ (NestJS)     │  │ Console (3002)   │
       │ Logto 页面    │  │              │  │ + Cloudflare     │
       │ Device API    │  │              │  │   Access + WAF   │
       └──────────────┘  └──────────────┘  │ + 审计日志       │
                                           └──────────────────┘
```

### 请求路由

```
user.{DOMAIN} (用户侧页面)
  ├── /api/devices*, /api/health*, /api/apps*  →  Device API (3002)
  ├── /oidc/*                                  →  Logto Core (3001)
  ├── /api/experience*, /api/.well-known*      →  Logto Core (3001)
  ├── /api/interaction*, /api/session*         →  Logto Core (3001)
  ├── /api/account-center*                     →  Logto Core (3001)
  ├── /login → /sign-in                        →  Logto Core (3001)
  ├── /register, /forgot-password              →  Logto Core (3001)
  ├── /account/*, /sign-in*, /callback*        →  Logto Core (3001)
  ├── /consent*, /assets*                      →  Logto Core (3001)
  └── /* (SPA 兜底)                             →  wo-bot-web 静态文件

api.{DOMAIN} (管理 API)
  └── /api/*                                   →  Device API (3002)

admin.{DOMAIN} (管理控制台，六层纵深防护)
  ├── /oidc/*                                  →  Logto (3002), 清除 Authorization 头
  └── /*                                       →  Logto (3002), 独立审计日志
```

## 前置条件

- Docker 24+ 及 Docker Compose v2
- 根域名 `wo-bot.com`，三个子域名 A 记录指向服务器 IP：
  - `user.wo-bot.com` — 用户侧页面
  - `api.wo-bot.com` — 管理 API
  - `admin.wo-bot.com` — 管理控制台（独立防护）
- Resend 账号及已验证的发件域名
- （推荐）Cloudflare Zero Trust Access + WAF 保护 `admin.{domain}`
- （可选）Cloudflare Turnstile 密钥用于 CAPTCHA

## 部署步骤

### 1. 克隆代码

```bash
git clone <repo-url> wo-bot-account
cd wo-bot-account
```

### 2. 配置环境变量

```bash
cp .env.example .env
vim .env
```

```env
# 根域名（不含协议和子域名）
DOMAIN=wo-bot.com

# 数据库
POSTGRES_USER=wobot
POSTGRES_PASSWORD=<强密码>

# OIDC Cookie 加密密钥（生成: openssl rand -hex 32）
OIDC_COOKIE_KEYS=<hex>

# JWT 签名密钥（Device API 用，与 Logto OIDC 签名密钥一致）
JWT_SECRET=<hex>

# 机器人共享密钥
ROBOT_SECRET=<hex>

# Logto M2M 凭证（Device API 调用 Logto Management API 用）
M2M_CLIENT_ID=<Logto M2M App 的 client id>
M2M_CLIENT_SECRET=<Logto M2M App 的 client secret>

# Resend 邮件
RESEND_API_KEY=re_xxxxxxxxxxxxx
EMAIL_FROM=noreply@yourdomain.com

# OIDC Issuer
OIDC_ISSUER=https://user.wo-bot.com/oidc

# 速率限制
RATE_LIMIT_BIND_DAILY=10
MAX_DEVICES_PER_USER=20
```

### 3. 拉取镜像并启动

```bash
# 拉取 Logto 官方镜像
docker compose pull logto

# 构建自建组件
docker compose build wo-bot-device-api

# 启动所有服务
docker compose up -d

# 查看状态
docker compose ps
```

### 4. 初始化 Logto

首次部署需要安装官方连接器并运行种子配置：

```bash
# 安装官方连接器（SMTP 等）
docker compose run --rm logto npm run cli connector add -- --official

# 数据库种子初始化
docker compose run --rm logto npm run cli db seed -- --swe

# 运行自定义种子配置（创建 OAuth2 客户端 + 品牌定制）
docker compose run --rm logto-seed
```

种子脚本自动完成：
- 创建 Wo-Bot Web SPA OAuth2 客户端（PKCE 强制，第一方应用）
- 配置中文语言、品牌配色
- 配置找回密码（EmailVerificationCode）
- 启用 Account Center（email/phone/username/password/profile 字段可编辑）
- 创建 M2M 应用供 Device API 调用 Management API
- 输出 Client ID（用于前端配置）

### 5. 配置 SMTP 邮件连接器

**Admin Console 方式**（推荐）：
1. 打开 `https://admin.wo-bot.com`（通过 Cloudflare Access 认证 → Logto OIDC 登录）
2. 进入 Connectors → Add → SMTP
3. 填写 Resend 配置：Host: `smtp.resend.com`, Port: `465`, 认证方式: `login`
4. 保存并测试发送

**数据库直配方式**（自动化部署用）：

```sql
INSERT INTO connectors (tenant_id, id, connector_id, config, metadata)
VALUES ('default', 'simple-mail-transfer-protocol', 'simple-mail-transfer-protocol',
  '{
    "host": "smtp.resend.com",
    "port": 465,
    "auth": {"type": "login", "user": "resend", "pass": "re_xxxxxxxx"},
    "fromEmail": "Wo-Bot <noreply@yourdomain.com>",
    "secure": true,
    "templates": [
      {"usageType": "SignIn", "subject": "Wo-Bot 登录验证码", "contentType": "text/plain", "content": "你的登录验证码是 {{code}}，有效期 10 分钟。"},
      {"usageType": "Register", "subject": "Wo-Bot 注册验证码", "contentType": "text/plain", "content": "欢迎注册 Wo-Bot！你的注册验证码是 {{code}}，有效期 10 分钟。"},
      {"usageType": "ForgotPassword", "subject": "Wo-Bot 重置密码验证码", "contentType": "text/plain", "content": "你的重置密码验证码是 {{code}}，有效期 10 分钟。"},
      {"usageType": "Generic", "subject": "Wo-Bot 验证码", "contentType": "text/plain", "content": "你的验证码是 {{code}}，有效期 10 分钟。"}
    ]
  }'::jsonb,
  '{"id": "simple-mail-transfer-protocol", "target": "smtp", "platform": null, "name": {"en": "SMTP", "zh-CN": "SMTP 邮件服务"}, "logo": "./logo.svg", "logoDark": null, "description": {"en": "SMTP email service", "zh-CN": "SMTP 邮件服务"}, "formItems": [], "readme": "", "isToastHidden": false}'::jsonb
)
ON CONFLICT (id) DO UPDATE SET config = EXCLUDED.config;
```

### 6. 配置 CAPTCHA（可选但推荐）

数据库直配 Cloudflare Turnstile：

```sql
INSERT INTO captcha_providers (tenant_id, id, config)
VALUES ('default', 'cloudflare-turnstile',
  '{"type": "Turnstile", "siteKey": "<your-site-key>", "secretKey": "<your-secret-key>"}')
ON CONFLICT (id) DO UPDATE SET config = EXCLUDED.config;

UPDATE sign_in_experiences
SET captcha_policy = '{"enabled": true}'
WHERE tenant_id = 'default';
```

### 7. 配置 Admin Console 独立域名防护（Cloudflare Dashboard）

Admin Console 通过独立子域名 `admin.{DOMAIN}` 访问，叠加多层防护：

**已完成（自动化部署）**：
- ✓ DNS 记录：`admin.wo-bot.com` A 记录添加
- ✓ Logto `ADMIN_ENDPOINT` 更新为 `https://admin.wo-bot.com`
- ✓ Caddyfile：`admin.{DOMAIN}` 独立站点块（反向代理 + JSON 审计日志）
- ✓ `user.{DOMAIN}` 移除 `/admin*` `/console*` 路由
- ✓ 端口隔离：Logto 3002 端口不暴露到宿主机

**待手动配置（Cloudflare Dashboard）**：
- ⏳ Cloudflare Zero Trust Access：创建 Access Application 保护 `admin.wo-bot.com`，配置邮箱白名单
- ⏳ Cloudflare WAF：IP 白名单规则，仅允许指定 IP 访问

### 8. 构建并部署前端

```bash
cd packages/wo-bot-web

# 配置环境变量
cat > .env << 'EOF'
VITE_LOGTO_ENDPOINT=https://user.wo-bot.com
VITE_LOGTO_CLIENT_ID=<seed脚本输出的Client ID>
VITE_DEVICE_API_URL=https://user.wo-bot.com
EOF

# 构建
npm install && npm run build
```

前端产物位于 `packages/wo-bot-web/dist`，由 Caddy 通过 `file_server` 直接托管。

### 9. 验证部署

```bash
# 检查服务状态
docker compose ps

# 测试用户侧页面
curl -sI https://user.wo-bot.com/sign-in

# 测试 Device API
curl -s https://api.wo-bot.com/api/health
curl -s https://user.wo-bot.com/api/health

# 测试 Logto OIDC Discovery
curl -s https://user.wo-bot.com/api/.well-known/sign-in-exp

# 测试 Admin Console（需通过 Cloudflare Access 认证）
curl -sI https://admin.wo-bot.com
```

## 本地构建镜像

授权服务直接使用 Logto 官方镜像，无需构建。两个自建组件分别构建：

Device API 镜像：

```bash
docker compose build wo-bot-device-api
```

前端（标准 Vite 项目，无需打镜像）：

```bash
cd packages/wo-bot-web && npm run build
# 产物位于 packages/wo-bot-web/dist
```

## CI/CD 流程

仓库通过 GitHub Actions 自动构建并推送镜像。触发方式：推送到 `main` 构建快照，打 `v*` tag 构建发布版本。

### 构建矩阵（2 个镜像）

| 镜像 | 构建上下文 | 用途 |
|------|-----------|------|
| `wo-bot-device-api` | `./packages/wo-bot-device-api` | 设备管理 API |
| `wo-bot-web` | `./packages/wo-bot-web` | 前端静态文件镜像 |

> Logto 官方镜像 `ghcr.io/logto-io/logto` 由 Logto 团队维护，不在 CI 中构建。直接引用指定版本 tag。

### 版本管理

- 推送 `main`：镜像带 `main` 和 `latest` 标签
- 打 `v1.2.3` tag：镜像带 `1.2.3` 和 `1.2` 标签
- 生产服务器固定到具体版本号（非 `latest`）

## Logto 版本升级

授权服务使用官方 Docker 镜像，版本升级简化：

```bash
# 1. 修改 docker-compose.yml 中的镜像 tag
#    image: ghcr.io/logto-io/logto:v1.21.0 → ghcr.io/logto-io/logto:v1.22.0

# 2. 拉取新镜像并重启
docker compose pull logto
docker compose up -d logto

# 3. 运行数据库变更（查阅 release notes）
docker compose run --rm logto npm run alteration deploy latest

# 4. 重新执行种子配置（恢复品牌定制）
docker compose run --rm logto-seed
```

与原 Fork 方案相比，升级从"git subtree 合并 → 解决冲突 → 重新构建镜像 → 重启"简化为"修改 tag → 拉取镜像 → 重启"。

## 常见运维操作

### 查看 Logto 日志

```bash
docker compose logs -f logto
```

### 查看 Admin Console 访问日志

```bash
tail -f /data/caddy/logs/admin-access.log
```

### 重启单个服务

```bash
docker compose restart logto
docker compose restart wo-bot-device-api
```

### 更新 SMTP 配置

Admin Console（`https://admin.wo-bot.com`）→ Connectors → SMTP → 编辑配置 → 保存

### 数据库备份

```bash
# 备份
docker compose exec postgres pg_dump -U wobot logto > backup_logto_$(date +%Y%m%d).sql
docker compose exec postgres pg_dump -U wobot wobot_devices > backup_devices_$(date +%Y%m%d).sql

# 恢复
docker compose exec -T postgres psql -U wobot logto < backup_logto_xxx.sql
```

### 更新版本

```bash
git pull
docker compose pull logto       # 拉取新版 Logto 官方镜像
docker compose build            # 重建自建组件
docker compose up -d
```

## 安全注意事项

- `.env` 文件包含所有密钥，不提交到版本控制
- 生产环境必须启用 HTTPS（Caddy 自动处理 TLS 证书）
- `admin.{DOMAIN}` 独立日志记录到 `/data/caddy/logs/admin-access.log`，JSON 格式含真实客户端 IP
- Cloudflare Zero Trust Access + WAF IP 白名单保护管理面
- Logto Admin Console 3002 端口不暴露到宿主机（仅 Docker 内部网络可访问）
- 定期备份数据库（logto 库 + wobot_devices 库）
- Resend API Key 通过数据库配置，不暴露在前端
- JWT_SECRET 和 ROBOT_SECRET 不可泄露，泄露后需轮换
- OIDC Cookie Keys 不可丢失，否则所有用户 session 失效

## 本地开发环境

本地开发使用与生产不同的架构（Vite Proxy 模式）。详见 R00040-2 需求文档。

核心区别：
- 生产用 Caddy 反向代理，本地用 Vite Proxy（端口 9094）
- 生产用 Caddy 自动 TLS，本地用 mkcert 本地证书
- 生产和本地均使用 Logto 官方镜像（仅配置不同）
- 生产用 Resend SMTP，本地用模拟 SMTP 服务器（端口 1025 + Web UI 端口 8025）
- 本地域名：`wo-bot-dev.com`（局域网 DNS 解析）

本地开发启动方式：
```bash
# 启动基础设施
docker compose -f docker-compose.dev.yml up -d

# 启动 Logto
docker compose -f docker-compose.logto-dev.yml up -d

# 启动 Device API
cd packages/wo-bot-device-api && npm run start:dev

# 启动前端
cd packages/wo-bot-web && npm run dev
```

访问地址：`https://user.wo-bot-dev.com:9094`（mkcert HTTPS 证书）
