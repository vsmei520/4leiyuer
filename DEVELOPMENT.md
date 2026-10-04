# 曾美团队-育儿科普 · 开发文档

> 本文面向**维护者**。使用说明见 [说明文档](README-用户说明.md)，后续计划见 [计划文档](PLAN.md)。

## 一、架构总览

项目采用**壳核分离**设计，公开仓库只提供客户端外壳，创作规则与授权逻辑全部在服务端。

```text
┌─────────────────────────────────────────────┐
│  客户端（Codex 插件 / ChatGPT Action）        │
│  只含外壳：插件清单、MCP 地址、连接层 Skill    │
└──────────────────┬──────────────────────────┘
                   │ HTTPS + OAuth 2.1
                   ▼
┌─────────────────────────────────────────────┐
│  服务端（Node 原生 HTTP，默认端口 28787）      │
│  ├─ OAuth 授权服务                            │
│  ├─ 授权码 / 设备绑定管理                      │
│  ├─ MCP 端点（工具 yuer_run）                  │
│  └─ 管理后台 /admin                            │
│                   │                           │
│                   ▼ 读取                        │
│  私有核心（3 文件，不入公开仓库）               │
│  ├─ SKILL.md                                  │
│  ├─ source-long-term-memory.txt               │
│  └─ source-four-types.txt                     │
└─────────────────────────────────────────────┘
```

### 核心机制：客户端模型生成

**服务端不调用任何大模型。** 它只负责读取私有文件、拼接上下文，然后把字符串返回给客户端，由客户端的模型完成生成。

这样做的收益：
- 服务端零模型成本
- 私有规则不出服务器（只作为上下文下发，用完即弃）
- 换模型无需改服务端

`runWorkflow` 返回体中 `mode: 'client-model'` 标记的正是这个模式。

## 二、两种运行形态

| | 线上版 | 本地版 |
| --- | --- | --- |
| 形态 | 远程 MCP + OAuth | 纯 Skill |
| 规则来源 | 服务端 `private-core/` | 插件内 `references/` |
| 授权 | OAuth 2.1 + 激活码 + 设备绑定 | 无 |
| 网络 | 需要 | 无 |
| 构建 | `release.yml` 打插件包 | `local/build-local.mjs` |

**两者的规则内容同源**——都来自 `private-core/` 三文件。

## 三、目录结构

```text
.
├── .codex-plugin/plugin.json     # Codex 插件清单
├── .agents/plugins/marketplace.json  # 市场入口
├── .mcp.json                     # 远程 MCP 连接配置
├── skills/yuer-film/             # 线上版连接层 Skill（不含规则）
├── chatgpt/openapi.yaml          # ChatGPT Actions 定义
│
├── private-core/                 # 【私有】创作规则三文件
│   ├── SKILL.md
│   ├── source-long-term-memory.txt
│   └── source-four-types.txt
├── server/                       # 【私有】服务端
│   ├── index.mjs                 # HTTP 入口与路由
│   ├── store.mjs                 # 授权码与令牌持久化
│   └── core-engine.mjs           # 私有核心加载与上下文组装
├── admin/                        # 【私有】管理后台前端
├── deploy/                       # 【私有】部署文档与配置
└── local/                        # 【私有】本地版构建
    ├── build-local.mjs
    └── dist/
```

> 标【私有】的目录均在 `.gitignore` 中，**不会进入公开仓库**。

## 四、服务端要点

### 路由一览

| 路径 | 方法 | 用途 |
| --- | --- | --- |
| `/healthz` | GET | 健康检查 |
| `/.well-known/oauth-*` | GET | OAuth 元数据发现 |
| `/oauth/authorize` | GET | 授权页 |
| `/oauth/authorize/approve` | POST | 提交手机号 + 激活码 |
| `/oauth/token` | POST | 换取访问令牌（PKCE 校验） |
| `/oauth/register` | POST | 动态客户端注册 |
| `/mcp` | POST | MCP 端点 |
| `/chatgpt/run` | POST | ChatGPT Actions 入口 |
| `/api/v1/workflow` | POST | 通用工作流入口 |
| `/admin` | GET | 管理后台 |
| `/api/v1/admin/*` | — | 后台接口（需管理会话） |

### 关键常量

| 项 | 值 |
| --- | --- |
| 访问令牌有效期 | 30 天 |
| 管理会话有效期 | 12 小时 |
| 授权请求有效期 | 10 分钟 |
| 授权码有效期 | 60 秒 |
| 时长档位 | `month` / `year` / `lifetime` |
| 激活码格式 | `YUER-XXXXXX-XXXXXX-XXXXXX` |

### 授权状态机

```text
UNACTIVATED ──激活──> ACTIVE ──到期──> EXPIRED
                 │         │
                 │         ├──管理员封禁──> BANNED
                 │         └──管理员撤销──> REVOKED
                 └──管理员解绑/解封──> UNACTIVATED
```

### 私有核心加载

`core-engine.mjs` 的 `loadPrivateCore`：

1. 读取 `PRIVATE_CORE_DIR` 下三个文件
2. 用三者 **mtime 拼接**作为缓存签名
3. 签名未变则复用缓存，变了才重新读取

**含义**：覆盖文件后服务端会**自动重载**，无需重启。

> ⚠️ 若部署环境使用保留原 mtime 的同步工具（部分 Docker 卷、rsync 配置），改了内容但 mtime 不变，缓存不会失效。此时需重启进程。

### 类型预判与截断

`classify()` 用关键词正则做**粗略预判**，结果作为提示传给客户端模型，最终判类仍以 `source-long-term-memory.txt` 的精细规则为准。

`extractModule()` 从 `source-four-types.txt` 按章节标题截取对应类型，上限 **24000 字符**。

> ⚠️ 当前四个章节最长约 14021 字符，尚未触发截断。但截断是**静默的**——若某章节继续扩写超过上限，内容会被无声削掉且无任何报错。

## 五、环境变量

| 变量 | 说明 | 默认 |
| --- | --- | --- |
| `NODE_ENV` | 设为 `production` 时强制要求管理密码 | — |
| `PORT` | 监听端口 | `28787` |
| `PUBLIC_BASE_URL` | 对外基础地址（OAuth 回调与元数据发现均基于此值） | 部署时按实际域名设置 |
| `ADMIN_USERNAME` | 后台账号 | `admin` |
| `ADMIN_PASSWORD` | 后台密码（**生产必填**） | 空 |
| `PRIVATE_CORE_DIR` | 私有核心目录 | `../private-core` |
| `DATA_FILE` | 状态持久化文件 | `./data/state.json` |

## 六、本地版构建

```bash
node local/build-local.mjs
```

脚本流程：

1. 清空并重建 `local/dist/yuer-4skill-local/`
2. 复制插件清单，**移除** `mcpServers` / `homepage` / `repository`
3. 复制两份 TXT 到 `skills/yuer-4skill-local/references/`
4. 读取 `private-core/SKILL.md`，**剥离原 frontmatter**后重新拼装
5. 生成 README，压缩为 zip

> ⚠️ 第 4 步的 frontmatter 剥离是必要的。私有 SKILL.md 自带 frontmatter，若不剥离会产生**两个 YAML 块叠加**，后者覆盖前者的 `name` 字段。

## 七、发布流程

### 公开外壳（GitHub）

推送 `v*` 标签时由 `.github/workflows/release.yml` 自动发布，流程为：

1. 校验三个 JSON 配置（`plugin.json` / `marketplace.json` / `.mcp.json`）
2. **检查仓库中是否混入私有路径**，若发现则直接失败
3. 打包公开外壳为 `yuer-4skill-plugin.zip`
4. 输出包内容清单，便于在日志中核对
5. 上传为 Release 资产

打包清单仅含 9 个文件：

```text
.codex-plugin/plugin.json
.agents/plugins/marketplace.json
.mcp.json
skills/yuer-film/SKILL.md
skills/yuer-film/agents/openai.yaml
chatgpt/openapi.yaml
chatgpt/README.md
README.md
.gitignore
```

> 第 2 步是**安全闸门**——即便误将私有目录加入版本控制，发布也会中止而非泄露。

### 私有更新（服务器）

**改动私有核心** → 上传覆盖 `private-core/` 对应文件 → **无需重启**（mtime 自动重载）

**改动服务端代码** → 上传 `server/` → **需重启 PM2 进程**

## 八、安全注意事项

- 私有核心三文件**永远不提交公开仓库**
- `ADMIN_PASSWORD` 生产环境必须设置，否则启动即报错
- 设备绑定：一个激活码绑定一台设备，换机需先解绑
- 访问令牌 30 天有效，被解绑/封禁立即失效
- 提交前用 `git status` 确认没有私有文件被意外纳入

## 九、已知问题清单

| # | 问题 | 影响 | 状态 |
| --- | --- | --- | --- |
| 1 | `extractModule` 24000 字符静默截断 | 章节超限时内容无声丢失 | 待观察 |
| 2 | mtime 缓存依赖 | 特定同步方式下热更新失效 | 待观察 |
| 3 | `classify()` 关键词预判粗糙 | 与精细判类规则能力不对等 | 待优化 |
