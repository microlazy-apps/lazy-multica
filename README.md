# lazy-multica

懒猫微服 lpk 包装：[multica-ai/multica](https://github.com/multica-ai/multica)

> Multica 是开源的托管式 AI 编码代理平台 — 把 Claude Code、Codex、
> GitHub Copilot CLI、Gemini、Cursor Agent 等编码 agent 当成可派活的
> 远程同事使用。

## 安装

通过懒猫微服「应用商店」搜索 *Multica* 安装。无必填参数 — 应用 URL 自动从盒子的子域名推导（即 `https://multica.<your-box-domain>`）。

可选参数（不填不影响基本登录与使用）：

| 参数 | 用途 |
|------|------|
| `RESEND_API_KEY` + `RESEND_FROM_EMAIL` | 邮箱验证码通过 [Resend](https://resend.com) 发送；不填则验证码打印到 backend 容器日志（私有部署够用） |
| `MULTICA_DEV_VERIFICATION_CODE` | 设为 6 位数字（如 `888888`）→ 自动切到 `APP_ENV=development`，所有账户的邮箱验证码都固定为该值。⚠️ **公网可达的盒子请留空** — 否则知道邮箱的人都能登录 |
| `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` | Google OAuth 登录（在 Google Cloud Console 把回调 URL 设为 `https://multica.<your-box-domain>/auth/callback`） |
| `ALLOW_SIGNUP` | 是否允许公开注册（默认 `true`） |
| `ALLOWED_EMAILS` / `ALLOWED_EMAIL_DOMAINS` | 邮箱白名单（与 `ALLOW_SIGNUP=false` 搭配使用） |

## 首次登录

1. 访问 `https://multica.<your-box-domain>`
2. 输入邮箱 → 点击发送验证码
3. **未配置 Resend 时**：验证码会打印到 `backend` 容器日志
   ```sh
   ssh root@<your-box>
   lpk-manager logs cloud.lazycat.app.multica backend | grep "Verification code"
   ```
4. 输入验证码登录

## 派活给 AI agent

派活逻辑发生在**你自己的本地机器**（不是懒猫盒子）。每个想用 AI agent 接活的成员都要：

1. 安装 `multica` CLI
   ```sh
   brew install multica-ai/tap/multica
   ```
2. 同时安装至少一个支持的 agent CLI（`claude` / `codex` / `copilot` / `gemini` / `cursor-agent` / ...）
3. 把 CLI 指向懒猫上的 multica
   ```sh
   multica setup self-host \
     --server-url https://multica.your-box.heiyu.space \
     --app-url    https://multica.your-box.heiyu.space
   ```
4. daemon 启动后，回到 Web UI 「Settings → Agents」创建 agent
5. 创建 issue 并指派给 agent — agent 会在你本地机器上自动接活

## 数据持久化

| 路径 | 内容 |
|------|------|
| `/lzcapp/var/postgres` | PostgreSQL 17 (pgvector) 数据 |
| `/lzcapp/var/uploads` | 用户上传的文件 |

## 资源占用

- 推荐：≥ 2 vCPU，≥ 2 GB RAM
- 内置依赖：PostgreSQL 17 with pgvector
- 后端是单一 Go 二进制；前端是 Next.js standalone

## License 与重打包说明

- 上游项目：[multica-ai/multica](https://github.com/multica-ai/multica)
  使用**修改版 Apache 2.0** 协议（见仓库内 LICENSE）
- 上游 license 限制：
  1. 不可作为 SaaS 服务托管给第三方（单组织内部使用 OK，含多 workspace）
  2. 不可作为组件嵌入到对外销售/分发的商业产品
  3. **不可移除/修改前端 LOGO 与版权信息**
- 本仓库做的事：仅把上游已发布到 GHCR 的 OCI 镜像（`ghcr.io/multica-ai/multica-web`、`ghcr.io/multica-ai/multica-backend`）按 sha256 锁版本后重打包成 lpk，**不修改源码、镜像、前端 LOGO**

## 升级

镜像在仓库 [README](#) 顶部按 sha256 锁版本。要升级到上游新版本：

1. 重新解析三个镜像的 digest（脚本见 `docker/Dockerfile` 头注释 + `CLAUDE.md`）
2. 用 `lzc-cli appstore copy-image` 重新预镜像 backend / postgres
3. 更新 `docker/Dockerfile` 与 `lazycat/lzc-manifest.template.yml`
4. bump `version` 后打 tag 触发 release.yml
