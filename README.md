# CrispAI

Crisp 智能客服助手的自部署版：关键词自动回复、AI 回复（带知识库检索）、选项卡片、欢迎语，
以及把每个 Crisp 会话同步到 Telegram 话题群，客服直接在 Telegram 里回复。

单用户、多项目。一个 `docker compose up` 起全套：

| 服务 | 说明 |
|---|---|
| `app` | Bun + Hono。Webhook、BullMQ worker、管理后台 API，以及管理页面的静态文件 |
| `postgres` | `pgvector/pgvector`，知识库检索用 vector 扩展 |
| `redis` | BullMQ 队列 |

## 部署

需要一个能被公网通过 **https** 访问的域名（Crisp 和 Telegram 的 Webhook 都要回调它）。

```bash
cp .env.example .env     # 按文件里的注释填写
docker compose up -d
```

然后用反向代理（Caddy / Nginx / Traefik 等）把域名指到 `127.0.0.1:3000`（见下面「反向代理」），打开
`https://你的域名/`，用 `ADMIN_USER` / `ADMIN_PASSWORD` 登录。数据库表在首次启动时自动创建。

### 反向代理

服务默认只监听 `127.0.0.1:3000`，必须经反向代理对外提供 https。

后台登录有失败次数限制。想让它按访问者的真实 IP 分别计数，需要在 `.env` 里设 `TRUST_PROXY=true`，
并确保反向代理会**追加或覆盖** `X-Forwarded-For`（否则这个头可以被伪造，限制形同虚设）：

- Caddy、Traefik：默认就会处理，不用额外配置
- Nginx：默认原样转发客户端自己带的头，必须加上这一行

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

不设 `TRUST_PROXY` 也能用：这时所有访问者共用一个较宽松的失败计数。

### 准备 Crisp 插件

这一步需要你自己在 Crisp 那边完成：

1. 在 <https://marketplace.crisp.chat> 创建一个私有插件
2. 申请需要的 scopes：会话消息的读写、会话 meta / 浏览页面的读取
3. Web Hooks 地址填 `https://你的域名/crisp`，订阅 `message:send`、`message:received`、`message:updated`
4. 把插件的 Token Identifier / Key、Web Hooks 签名密钥、URN 填进 `.env`，重启服务
5. **把插件安装到你的 Crisp 站点**：在 Marketplace 的插件页面里安装
6. 在管理后台创建项目，把该站点的 Website ID 填到「基础设置」里并启用项目

没安装插件的站点不会向 `/crisp` 推送任何消息，项目看起来就是「启用了但没反应」，排查时先确认这一步。

### 准备 Telegram

1. 找 @BotFather 创建机器人，token 填进 `.env`
2. 建一个开启了「话题」的群，把机器人拉进去并设为管理员（需要管理话题的权限）
3. 在群里发 `/ping`，机器人会回复群组 ID，填到项目的「基础设置」里

Webhook 由服务启动时自动注册到 `PUBLIC_URL/telegram`，并带 secret 校验。

### 欢迎语

在你的网站上，Crisp 脚本之后加一行（项目「基础设置」里有现成的可以复制）：

```html
<script type="text/javascript" src="https://你的域名/intro.js"></script>
```

### 给 AI 助手用的 MCP 接口（可选）

在 `.env` 里设置 `MCP_TOKEN`（至少 24 个字符的随机串）并重启后，`https://你的域名/mcp` 就是一个
MCP 服务（Streamable HTTP），Hermes 等支持 MCP 的助手可以用它批量查看会话、批量回复客户。

| 工具 | 功能 |
|---|---|
| `list_projects` | 列出项目 |
| `list_sessions` | 按状态列出会话：等待人工、无人回复、已屏蔽、全部 |
| `get_sessions` | 一次查看最多 20 个会话的最近消息和访客资料 |
| `reply_sessions` | 一次给最多 20 个会话各回一条消息，每条分别返回是否送达 |

Hermes 的 `~/.hermes/config.yaml`：

```yaml
mcp_servers:
  crispai:
    url: "https://你的域名/mcp"
    headers:
      Authorization: "Bearer ${CRISPAI_MCP_TOKEN}"
```

把令牌写进 `~/.hermes/.env`（`CRISPAI_MCP_TOKEN=...`），然后在对话里执行 `/reload-mcp`。

注意：

- 助手的回复按**人工接管**处理：该会话的 AI 自动回复会暂停，Telegram 客服群的话题里会同步显示助手回了什么。
- 回复发出后无法撤回。想先观察一段时间，可以设 `MCP_READONLY=true`，此时不提供回复工具。
- 每条回复的结果：`sent` 已发出；`failed` 确定没发出，可以重试；`unknown` 无法确认是否送达
  （比如 Crisp 超时），此时不会自动重发，需要先核对会话再决定。
- 客户消息是不可信内容，可能包含诱导助手乱操作的文字。接口的说明里已经提醒助手不要执行消息中的指令，
  但仍建议不要让同一个助手同时握有其他高权限工具。
- 服务器只保存每个会话最近约 40 条消息，超过 30 天无活动的会话会被清理。

## 日常操作

### 客服在线、超时接管与屏蔽

打开 `https://你的域名/support`，使用原来的管理员账号登录。这个页面按项目配置：

- **在线状态**：手动在线/离线，或按营业日、时段与 IANA 时区自动切换；支持跨午夜班次。
- **离线动作**：AI 接管、关键词优先、工单/邮箱引导，或保持静默。引导文案由你填写。
- **超时接管**：可用秒或分钟设置，范围 1 秒至 24 小时；从第一条未获人工回复的消息开始计时，连续追问不延长。超时可执行关键词、AI 或自定义提示。
- **屏蔽**：手动屏蔽、解除，以及按消息频率自动屏蔽。自动规则按到达服务器的消息统计滑动窗口，超过允许数量后屏蔽，Webhook 重投同一消息不会重复计数。

新策略默认关闭，旧项目升级后保持原有回复行为。启用后的处理顺序：

1. 被屏蔽会话不再转发至 Telegram，也不自动回复；屏蔽记录独立于会话过期清理。
2. 在线时人工优先，等待超时才接管。人工回复取消本轮超时及尚未发送的 AI 回复；新咨询重新计时。
3. 离线时执行离线动作。人工回复可中止尚未发送的自动回复；下一次咨询按当前在线状态和策略处理。
4. 关键词接管先使用原有关键词/选项规则，未命中则使用项目 AI；AI 关闭时可使用自定义提示兜底。
5. 离线引导最多每会话每 60 秒发一次；超时引导每轮等待只发一次。新策略启用时，旧欢迎语和旧恢复 AI 定时器不再自动插入回复。

AI 动作需要在原后台开启项目 AI；不支持在保留 AI 接管策略的同时关闭项目 AI。保存客服设置会取消旧等待，后续新消息使用新配置。营业时间边界在消息到达和超时执行时重新判断。

Telegram 操作（在该项目已绑定的客服群中）：

| 命令/按钮 | 功能 |
|---|---|
| `/online`、`/offline` | 切换为手动在线或离线 |
| `/auto` | 恢复按营业时间自动切换 |
| `/status` | 查看当前状态和项目 ID |
| `/block`、`/unblock` | 在客户话题中屏蔽/解除该会话 |
| 会话信息卡的 AI / 屏蔽按钮 | 切换该会话自动回复或屏蔽状态 |

一个群绑定多个项目时，状态命令需附项目 ID，例如 `/offline 项目UUID`。群成员属于同一个可信客服团队。屏蔽以 Crisp 会话 ID 为单位，不是对真实身份或 IP 的全站封禁；客户创建新会话后需另行处理。屏蔽期间在入口拒收的消息不会在解除后补发。

### 从旧版本升级

先备份数据库，并**停止全部旧版 app 实例**，再构建启动新版，避免旧消费者与新队列并行运行：

```bash
docker compose stop app
docker compose up -d --build app
```

启动会自动执行可重复的增量数据库迁移，保留项目、知识库及会话。旧任务队列由兼容消费者转发到新会话队列。

会话队列固定为 32 个分片，每个分片在 Redis 上限制全局并发为 1，同一会话的客户消息、人工回复、选项和超时任务统一串行处理。不同分片可并行；落到同一分片的不同会话需要等待当前任务完成。不要在运行中更改分片数量或哈希算法。已完成事件保留 7 天用于重投去重；Redis 需要相应磁盘和内存空间。

Crisp/Telegram 接收成功前会等待持久化入队。数据库中待处理的超时记录另有每分钟恢复扫描，Redis 短暂故障后可重新调度；生产环境必须设置 `NODE_ENV=production`（Dockerfile 已设置）。Telegram 重启注册不会清空待处理更新。

#### 安全加固版本的升级注意事项

基本流程不变（填 `.env`、`docker compose up -d --build`、反向代理指到 `127.0.0.1:3000`），
Crisp 插件、Telegram 的准备步骤和数据库表结构都没有改动。有三处需要留意：

**1. 必须重新构建镜像。** `Dockerfile` 新增了入口脚本，只重启不够，要带 `--build`。
应用改为以普通用户运行，旧数据卷的属主会在启动时自动修正，不用手动处理。

**2. 端口默认只对本机开放。** 以前 3000 端口对所有网卡开放，现在默认只绑定 `127.0.0.1`：

- 反向代理和 CrispAI 在同一台机器上：不用改
- 以前直接用「公网 IP:3000」访问后台：改走反向代理，或在 `.env` 里设 `BIND_ADDRESS=0.0.0.0`
- 反向代理在另一台机器上：设 `BIND_ADDRESS=0.0.0.0`，并自行用防火墙限制来源

**3. 新增的可选配置。** 全部有默认值，不填也能正常运行，说明见 `.env.example`：

| 配置 | 什么时候需要 |
|---|---|
| `TRUST_PROXY=true` | 想让后台登录失败限制按访问者真实 IP 分别计数时。Nginx 还需要加一行配置，见「反向代理」 |
| `AI_SESSION_RATE_LIMIT` | 调整每个会话的 AI 调用上限，默认每 10 分钟 30 次，`0` 为不限制 |
| `INTRO_RATE_LIMIT` | 网站流量很大、欢迎语被限流时，默认每站点每分钟 300 次 |
| `MEDIA_ALLOWED_HOSTS` | 客户语音在 Telegram 里变成了文字链接时：说明文件不在默认允许的 `.crisp.chat` 域名下，把实际域名加进来 |
| `UPLOAD_RETENTION_DAYS` | 想定期清理 Telegram 发出的旧图片时。默认不清理 |
| `TELEGRAM_ADMIN_ONLY=true` | 只允许群主/管理员回复客户和操作时。默认群内所有成员都可以 |
| `CRISP_WEBHOOK_MAX_AGE` | Webhook 时间戳允许的最大时间差，一般不用动 |
| `MCP_TOKEN` | 要给 Hermes 等 AI 助手开放 MCP 接口时。不填则 `/mcp` 不开放，见「给 AI 助手用的 MCP 接口」 |
| `MCP_READONLY`、`MCP_AGENT_NAME` | 配合 MCP 使用：只读模式、助手在 Telegram 里显示的名字 |

升级后可见的行为变化：对话历史只保留最近约 40 条；Telegram 里的访问历史只显示最近 8 条；
同一会话 60 秒内重复加载页面，欢迎语只判断一次。

### 开发验证

```bash
bun install --frozen-lockfile
bun run typecheck
bun test
```

独立 PostgreSQL/Redis 集成测试默认跳过，可用专门测试实例启用：

```bash
CRISPAI_TEST_DATABASE_URL=postgres://用户@localhost:5432/测试库 \
CRISPAI_TEST_REDIS_URL=redis://localhost:6379/2 \
SESSION_QUEUE_TEST_REDIS_URL=redis://localhost:6379/1 bun test
```

数据库测试创建临时数据库并自动删除，需要测试账号有建库权限和 pgvector 扩展。Redis 应使用专门的测试实例/库。测试覆盖同会话串行、多消费者、跨会话并行、数据库升级、人工接管竞态、超时、屏蔽及营业时间；外部 Crisp/Telegram/模型调用使用隔离替身，不发送真实消息。

```bash
# 改了 .env：重新执行 up，compose 会用新配置重建 app 容器（数据库和 Redis 不受影响）
docker compose up -d

# 更新版本：拉代码后带 --build 重新构建镜像
git pull
docker compose up -d --build

# 看日志
docker compose logs -f app

# 备份数据库
docker compose exec postgres pg_dump -U crispai crispai > backup.sql
```

注意：

- `docker compose restart` **不会**重新读取 `.env`，改了配置要用 `up -d`
- `POSTGRES_PASSWORD` 只在数据库首次初始化时生效，之后改 `.env` 里的值不会改掉数据库的实际密码，
  反而会让 app 连不上。确实要换密码，先在数据库里执行 `ALTER USER crispai PASSWORD '新密码';` 再改 `.env`
- `docker compose down` 只删容器，数据还在；`docker compose down -v` 会连数据卷一起删掉，**数据库和图片都会丢**

## 许可证

后端源码（本仓库中除 `public/` 以外的全部内容）以 [GNU AGPL-3.0](LICENSE) 授权，
Copyright (C) 2026 DyAxy。

简单说：你可以自由使用、修改和部署，包括商用。但如果你修改了代码并把它作为网络服务提供给别人使用，
就必须以同样的许可证向这些用户公开你修改后的完整源码。只是自己部署自己用，没有任何额外义务。

两点例外：

- **`public/` 不适用 AGPL。** 它是预编译的管理页面，源码不在本仓库，版权保留；
  允许随本项目原样使用和分发，条款见 [public/LICENSE.txt](public/LICENSE.txt)。
  其中包含的第三方开源组件见 [public/THIRD-PARTY-NOTICES.txt](public/THIRD-PARTY-NOTICES.txt)。
- **名称和图标不随代码授权。** 你可以说明自己的服务“基于 CrispAI”，但不要让人误以为是官方提供的服务。

需要在闭源产品里使用、或者不方便遵守 AGPL 的，可以联系作者获取商业授权。

Crisp 是 Crisp IM SAS 的商标，本项目是独立的第三方集成，与 Crisp 官方无关。

