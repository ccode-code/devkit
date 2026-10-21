# DevKit 桌面开发者工具箱 — 实施计划

## Context

用户需要一个**桌面工具合集**：打开软件后，通过开关即可监听手机和电脑发出的 HTTPS 请求。抓包只是第一个工具，后续还要放开发者日常小工具（JSON 格式化、编解码、时间戳、正则测试等）、网络调试类工具，以及预留更多扩展位。

因此不能写成单一抓包器，而要做成**可插拔的工具平台**：常驻左侧功能栏（图标 + 文字）+ 右侧内容区，首页内容区用宫格卡片展示全部功能；新增工具只需要注册一个前端模块和后端能力包，不改动框架代码。

命名：**DevKit**。项目目录 `devkit/`，Go module `devkit`，应用标识 `app.devkit.desktop`，内核二进制 `devkit-core`。

## 技术选型

- **桌面壳**：Tauri 2（Rust）
- **后端**：Go 独立进程 `devkit-core`（sidecar），承载全部业务能力
- **前端**：React 19 + TypeScript + Vite
- **第一个工具**：HTTPS 抓包代理（MITM）

### 为什么是这个组合

跨端成熟度优先，业务开发语言选 Go：

| 方案 | 结论 |
| --- | --- |
| Tauri 2 + Go sidecar | **选它**。Tauri 的跨平台打包、签名、公证、托盘体系成熟；业务全部用 Go 写 |
| Wails v2 | 否决。Go 单进程更简单，但跨端打包与生态成熟度不如 Tauri |
| Electron + Go sidecar | 否决。双层运行时，需要 Node 管理子进程，安装包 100MB+ |
| 纯 Tauri（Rust 写业务） | 否决。Rust 只保留在壳层，不承载业务逻辑 |

### Rust 的边界（硬约束）

Tauri 的壳由 Rust 编写，**无法做到零 Rust**。为把 Rust 的影响降到最低，规定：

- `apps/desktop/src-tauri/src/lib.rs` 只允许做四件事：启动 sidecar、读取握手行、通过 `get_core_info` 把端口与 token 交给前端、退出时回收子进程；外加注册单实例插件
- 预计 50–80 行，不写任何抓包、代理、证书、系统配置、数据存储逻辑
- 任何新需求默认落在 Go 侧；确需改动 Rust 时必须在 PR 里说明为什么 Go 做不到

### 已知代价

1. **两套工具链**：构建机需要 Rust 与 Go。Rust 只在壳层，Go 二进制由脚本预编译后作为 externalBin 打包。
2. **两套构建矩阵**：`devkit-core` 需按 target triple 产出（macOS arm64/x86_64、Windows x64/arm64）。Go 侧保持 `CGO_ENABLED=0`，可以在任意平台交叉编译。
3. **双进程生命周期**：Rust 崩溃或被强杀时，Go 可能成为孤儿进程。靠 stdin EOF + 快照落盘 + 启动检测三层兜底。

## 目标能力（第一版）

抓包工具：

1. 抓取并展示请求/响应（方法、URL、状态码、头、Body、耗时、大小），支持筛选与搜索
2. 根证书一键生成与下载（含手机安装引导、二维码）
3. 断点修改请求/响应
4. 请求重放
5. 流量导出 HAR
6. 一键开关：自动设置/还原系统代理，并在异常退出后兜底还原

平台层：

- 宫格首页、工具路由、前端工具注册表、后端工具注册表
- 预留纯前端工具（无需 Go）与需后端工具的两种接入形态
- macOS 与 Windows 双端；系统代理的 Windows 完整自动化排在后续阶段

## 架构总览

```text
┌──────────────────────────────────────────────┐
│ Tauri 2 壳 (Rust，只做进程管理)               │
│  ├─ 拉起 / 回收 Go sidecar 进程               │
│  ├─ 读取握手行，把 API 端口 + token 暴露给前端 │
│  ├─ 单实例插件                                │
│  └─ 系统托盘 / 窗口                           │
│      ┌────────────────────────────────────┐  │
│      │ React 前端 (Vite)                   │  │
│      │  ├─ 左侧功能栏（图标+文字，常驻）     │  │
│      │  ├─ 首页：宫格卡片展示全部功能        │  │
│      │  └─ /tool/:id → 工具页面             │  │
│      └───────────┬────────────────────────┘  │
└──────────────────┼───────────────────────────┘
                   │ HTTP + WebSocket (127.0.0.1:随机端口, Bearer token)
┌──────────────────▼───────────────────────────┐
│ Go 后台 devkit-core                          │
│  ├─ internal/server      API / WS / 鉴权      │
│  ├─ internal/tool        工具注册接口          │
│  ├─ internal/tools/proxy 抓包工具             │
│  │    ├─ ca        根证书与动态签发            │
│  │    ├─ proxy     代理与 MITM                │
│  │    ├─ store     流量环形缓冲                │
│  │    ├─ intercept 断点                       │
│  │    ├─ replay    重放                       │
│  │    ├─ har       HAR 导出                   │
│  │    └─ certpage  证书下载页                 │
│  ├─ internal/sysproxy   系统代理快照 / 还原    │
│  └─ internal/config     配置与数据目录          │
└──────────────────────────────────────────────┘
```

通信方式：Go 绑定 `127.0.0.1:0` 随机端口启动，stdout 输出一行握手 JSON（`{"api_port":..., "token":"..."}`），Rust 读取后用 `get_core_info` command 交给前端；前端所有请求带 `Authorization: Bearer <token>`。流量数据走 WebSocket 增量推送。

## 目录结构

```text
/Users/liutao/github/devkit/
├── core/                              # Go 后端
│   ├── go.mod
│   ├── cmd/devkit-core/main.go
│   └── internal/
│       ├── server/                    # HTTP API、WS hub、token 鉴权
│       ├── tool/                      # Tool 接口 + 注册表
│       ├── tools/proxy/               # 抓包工具
│       ├── sysproxy/                  # 系统代理快照 / 设置 / 还原
│       └── config/                    # 应用数据目录、config.json
├── apps/desktop/
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── app/                       # App、路由、宫格首页
│   │   ├── components/                # 通用 UI（Tailwind + Radix）
│   │   ├── lib/ipc.ts                 # core 连接信息与 fetch/ws 封装
│   │   └── tools/
│   │       ├── registry.ts            # 前端工具注册表
│   │       └── proxy/                 # 抓包工具页面与组件
│   └── src-tauri/
│       ├── src/lib.rs                 # sidecar 生命周期、握手（唯一允许的 Rust 代码）
│       ├── tauri.conf.json
│       └── binaries/                  # 预编译的 Go 二进制（不入库）
├── scripts/
│   ├── build-core.sh                  # macOS / Linux 上交叉编译
│   └── build-core.ps1                 # Windows 上编译
└── docs/
    ├── PLAN.md
    └── SPIKE.md                       # 阶段 0 验证结论
```

## 关键设计

### 进程握手与鉴权

- Go 启动后绑定 `127.0.0.1:0`，生成 32 字节随机 token，向 stdout 打印单行 JSON 后继续服务
- Rust 用 `tauri-plugin-shell` 启动 sidecar，读取 stdout 首行解析，存入应用状态
- 前端启动时调用 `get_core_info` 拿到 `{api_port, token}`，之后所有 HTTP 与 WS 请求携带 Bearer token
- `/api/health` 免鉴权，但只监听回环地址
- token 每次启动重新生成，不落盘

### 进程生命周期与崩溃兜底

- 正常退出：Rust 在 `RunEvent::ExitRequested` / `Exit` 中 kill sidecar；Go 捕获 `SIGTERM` 后还原系统代理再退出
- Rust 异常退出：Go 监听 **stdin EOF**，管道关闭即尝试还原并退出。shell plugin 的 stdin 是否保持管道在阶段 0 实测，不可用则退化为下一条
- Rust 被 `SIGKILL`：无法通知。Go 检测到父进程消失（`os.Getppid()` 变化）后自行退出
- 最终兜底：系统代理快照**先落盘并 fsync**，再修改系统配置；应用每次启动检测快照文件，存在即提示「上次未正常还原」并支持一键还原
- **单实例**：Rust 注册 `tauri-plugin-single-instance`；Go 在数据目录额外持有文件锁，防止 sidecar 被重复拉起

### 工具注册机制（编译期注册，不做动态加载）

Go 侧：

```go
type Tool interface {
    Meta() Meta
    Routes() []Route          // 挂在 /api/tools/{id}/...
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Status() Status
}
```

`main.go` 中显式注册；`/api/tools` 返回全部工具元数据与状态，前端据此渲染。

前端侧：

```ts
interface ToolMeta {
  id: string
  name: string
  description: string
  icon: ReactNode
  category: 'dev' | 'network' | 'ops'
  backend: boolean                                     // 是否需要 Go 侧能力
  component: () => Promise<{ default: ComponentType }>  // 懒加载
}
```

元数据归属：前端 registry 是 UI 展示的单一来源（名称、图标、描述、分类、懒加载组件）；后端 `/api/tools` 只返回运行状态（`running` / `stopped`）与统计，前端按 id 合并。纯前端工具（如 JSON 格式化）`backend: false`，不参与后端状态匹配。

**校验方式为运行时**：前端拿到 `/api/tools` 后，把 `backend: true` 的工具与返回集合比对，缺失的标记为「不可用」并在 UI 提示，不阻塞启动。不再维护两份 ID 常量和构建期校验脚本——那样每加一个示例工具都要改构建流程。

### IPC 与事件

**事件命名统一为 `域:动作`**，不带点号：

| 事件 | 负载 | 说明 |
| --- | --- | --- |
| `tool.status` | `Status` | 工具启停 |
| `flows:batch` | `FlowSummary[]` | 批量流量摘要，100ms 聚合一次，单批最多 200 条 |
| `breakpoint:hit` | `Breakpoint` | 断点命中 |
| `proxy:error` | `{message}` | 代理运行错误 |
| `sysproxy:changed` | `{detail}` | 检测到系统代理被外部改动 |

流量**必须批量推送**。逐条发 WS 消息在每秒数百请求时会打满前端事件循环；列表只推摘要（不含 body），详情用 `GET /flows/{id}` 按需拉取。

### 抓包核心

- MITM 底座：`github.com/elazarl/goproxy`（版本以阶段 0 spike 实测为准，本文档不锁定未核实的版本号）。它原生提供：
  - `TLSConfigFromCA(&caCert)` 生成按域名动态签发的 `*tls.Config`
  - `ConnectAction{Action: ConnectMitm, TLSConfig: ...}` 注入自建 CA
  - `ProxyHttpServer.CertStore`（`CertStorage` 接口）做证书缓存
  - `AllowHTTP2 = true` 开启 HTTP/2 MITM（默认关闭），需同时开启上游 `ForceAttemptHTTP2`
- CA：RSA 2048（兼容性优先），私钥 `0600` 存于应用数据目录；叶子证书按域名生成并做 LRU 缓存
- 叶子证书要带 SAN、支持 wildcard 兜底
- 流量模型 `Flow`：见下文
- 存储：内存环形缓冲（默认 5000 条），body 默认上限 2MB，超限截断并标记；第一版不落盘

### 断点

- 规则：host 包含 / method / URL 正则，作用阶段（请求、响应、两者）
- goproxy 回调内阻塞等待 channel，前端收到 `breakpoint:hit` 后编辑并 POST 决策（放行 / 修改 / 丢弃）
- 默认超时 5 分钟自动放行；客户端提前断开时立即释放；工具停止时统一释放所有挂起请求，避免 goroutine 泄漏
- 重放使用独立的 `http.Client` 直连目标，**不经过本代理**，避免递归抓包与再次命中断点；重放结果可选写入历史列表
- WebSocket 101 响应不能包装 body，断点仅支持修改其请求头与握手 URL

### 证书下载与手机安装

- 独立轻量端口（默认 8897）提供证书与带二维码的安装引导页，手机无需先配代理即可访问
- iOS 下发 `.mobileconfig`（内含 CA）并说明**必须**到「设置 → 通用 → 关于本机 → 证书信任设置」手动开启完全信任；同时提供 `.pem`
- Android 提供 `.der`/`.crt`，并明确提示：Android 7+ 用户证书默认不被大多数 App 信任，浏览器通常可用
- 桌面端：macOS 引导双击安装并在钥匙串中设为信任；Windows 引导双击安装到「受信任的根证书颁发机构」，不自动调用 `certutil` 修改系统存储
- 前端展示本机局域网 IP 与二维码，页面内附分平台步骤
- 页面使用一次性 token（二维码 URL 携带），后续版本可加配对 IP 白名单

### 局域网暴露（安全约束，不是可选项）

代理的可用前提是手机能连上，但**一个无认证的 MITM 代理绑定 `0.0.0.0` 等于向同一网段的所有设备开放**。因此：

- 默认只监听 `127.0.0.1`，本机可用
- 手机接入时，用户显式打开「局域网模式」，此时**绑定选定的局域网 IP**（不是 `0.0.0.0`），并启用客户端 IP 白名单（首次配对时记录手机 IP）
- 可选启用代理 Basic Auth（密码为一次性 token），作为白名单之外的补充；这会让手机端配置多填两项，默认关闭
- 关闭抓包或退出应用时自动关闭局域网监听
- 证书页端口遵循同一规则

### 系统代理开关

跨平台差异较大，分两期：

#### macOS（第一版完整）

- 开启时先**快照**当前所有网络服务的 HTTP/HTTPS/PAC/bypass/SOCKS 配置
- 快照先写盘并 fsync，再修改配置；正常关闭时先还原再删文件
- 崩溃、`SIGKILL`、断电无法执行还原，靠启动检测兜底
- **检测到已有代理时（如本机正在使用的 8118/8119）不静默覆盖**，明确提示并让用户选择「替换」或「仅手动模式」
- 只设置 HTTP/HTTPS 代理；**绝不能把系统 SOCKS 指向 goproxy 的 HTTP 端口**（协议语义不匹配）
- 是否需要管理员授权在阶段 0 实测；第一版优先保证「可检测、可手动还原」，特权 helper 排后续
- 企业 MDM / 安全软件可能把配置改回去，检测到变更时通过 `sysproxy:changed` 在 UI 告警

#### Windows（第一版提供检测与引导，自动化排后续）

- 需要区分 WinINET（大多数桌面应用与浏览器）与 WinHTTP（服务/命令行）
- WinINET 写 `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings` 的 `ProxyEnable` / `ProxyServer` / `ProxyOverride`，改完必须调 `InternetSetOption` 刷新，否则部分应用不生效
- WinHTTP 用 `netsh winhttp set proxy`
- 第一版只做读取与展示当前代理状态 + 一键跳转系统设置，避免在未充分测试前改注册表

### 跨平台数据目录

- Rust 用 `app.path().app_data_dir()` 取路径，通过命令行参数 `--data-dir` 传给 Go，保证两边一致
- macOS：`~/Library/Application Support/DevKit/`
- Windows：`%APPDATA%\DevKit\`

### 前端

- React 19 + TS + Vite，路由用 react-router 的 **HashRouter**（Tauri 打包后是自定义协议，BrowserRouter 刷新会 404）
- 状态用 Zustand，列表虚拟滚动用 @tanstack/react-virtual
- 样式 Tailwind CSS v4，图标 lucide-react，复杂交互（弹窗、下拉）用 Radix primitives
- 二维码用 `qrcode.react`

整体布局（所有页面一致）：

```text
┌────────────┬─────────────────────────────────────┐
│  Logo      │  内容区                             │
│ ────────   │                                     │
│  首页      │  首页：宫格卡片，展示全部可用功能    │
│  抓包      │  工具页：对应功能界面               │
│  JSON 工具 │                                     │
│  ...       │                                     │
│ ────────   │                                     │
│  设置      │                                     │
└────────────┴─────────────────────────────────────┘
```

- 左侧栏常驻，条目 = 图标 + 文字，当前工具高亮；数据来自同一份 registry，后续新增工具自动出现
- 首页宫格卡片：图标、名称、一句话描述、分类标签；抓包卡片额外显示运行状态徽标（运行中 / 已停止）和快捷开关
- 点击卡片或左侧栏条目都进入 `/tool/:id`
- 抓包页局部布局：工具头（开关/清空/筛选/导出）+ 流量列表（左）+ 详情 Tab（右：概览 / 请求 / 响应 / 原始）

## 接口与数据设计

### 端口分配

| 用途 | 默认 | 绑定 | 说明 |
| --- | --- | --- | --- |
| API / WebSocket | 自动分配 | 127.0.0.1 | 由内核启动后通过 stdout 告知壳 |
| HTTP/HTTPS 代理 | 8899 | 127.0.0.1，局域网模式下绑定指定网卡 IP | 手机与电脑共用，可配置 |
| 证书下载页 | 8897 | 同上 | 仅提供 `/ca.crt` 与说明页，可单独关闭 |

端口冲突时自动顺延并在 UI 提示实际端口。

### HTTP API

```text
GET    /api/health                     # 健康检查（无需 token）
GET    /api/tools                      # 工具列表与运行状态
GET    /api/tools/proxy/config         # 代理配置（端口、系统代理开关等）
PUT    /api/tools/proxy/config
POST   /api/tools/proxy/start          # 开启代理
POST   /api/tools/proxy/stop           # 关闭代理
GET    /api/tools/proxy/flows?query=&method=&status=&limit=&cursor=
GET    /api/tools/proxy/flows/{id}     # 详情（含 body）
DELETE /api/tools/proxy/flows          # 清空
GET    /api/tools/proxy/flows/{id}/export
POST   /api/tools/proxy/flows/export   # body: {ids: []} → HAR 下载
GET    /api/tools/proxy/intercept      # 断点规则
PUT    /api/tools/proxy/intercept
POST   /api/tools/proxy/breakpoints/{id}/decision   # continue / modify / drop
POST   /api/tools/proxy/replay         # 重放，body 可覆盖 method/url/headers/body
GET    /api/tools/proxy/ca             # CA 元信息（指纹、有效期）
GET    /api/tools/proxy/sysproxy       # 系统代理状态
POST   /api/tools/proxy/sysproxy/restore
GET    /ca.crt                         # 公钥证书下载（证书页端口）
```

### WebSocket

`/ws?token=...`（或首帧鉴权），消息见「IPC 与事件」一节。

### Flow 模型

```go
type Flow struct {
    ID            string
    StartedAt     time.Time
    Duration      time.Duration
    ClientIP      string
    Method        string
    URL           string
    Host          string
    Scheme        string
    HTTPVersion   string
    Status        int
    ReqHeaders    http.Header
    ReqBody       []byte
    RespHeaders   http.Header
    RespBody      []byte
    ReqSize       int64
    RespSize      int64
    ReqTruncated  bool
    RespTruncated bool
    ContentType   string
    TLS           *TLSInfo
    Error         string
}
```

`FlowSummary` 用于列表推送（不含 body）。body 读取时按 `Content-Encoding` 解压，二进制内容标记 `isBinary`，前端按需做 base64 或下载。

### 配置持久化

数据目录由 Rust 通过 `--data-dir` 传入：

- `config.json`：代理端口、证书端口、body 上限、缓存条数、断点规则、局域网模式开关
- `ca/`：CA 私钥与证书（私钥 `0600`）
- `proxy-snapshot.json`：系统代理快照（存在即表示上次未正常还原）
- `instance.lock`：Go 侧单实例文件锁

## 分阶段实施

### 阶段 0 — 脚手架与技术验证

1. `core/`：`go mod init devkit`、main 启动本地 HTTP，打印握手 JSON，提供 `/api/health`
2. `apps/desktop`：用 `pnpm create tauri-app` 生成 React + TS + Vite 模板
3. `scripts/build-core.sh` / `.ps1`：按 target triple 编译到 `src-tauri/binaries/devkit-core-<triple>`，脚本内设置 `GOPROXY=https://goproxy.cn,direct`
4. Rust 壳：启动 sidecar、读取握手行、`get_core_info`、退出时 kill、注册单实例插件
5. 前端：拿到端口与 token，显示内核连接状态
6. **goproxy spike**（独立 `spike/` 小程序，不接 UI）：
   - curl HTTP / curl HTTPS / Chrome 访问真实站点
   - HTTP/2 站点、WebSocket、SSE、分块传输
   - 100MB 大文件下载（内存与截断表现）
   - 上游 `ForceAttemptHTTP2` 开关对比
7. **系统代理 spike**：macOS `networksetup` 快照 / 设置 / 还原，确认是否需要提权；Windows 读取 WinINET 状态
8. **sidecar 生命周期 spike**：stdin 是否保持管道、父进程被杀后的行为、单实例插件的实际效果

产出 `docs/SPIKE.md`，每项写结论（通过 / 降级 / 换方案）。**goproxy 若撑不住 HTTP/2 或长连接，在此阶段就决定是否换 `lqqyt2423/go-mitmproxy`，不拖到阶段 2。**

验证：`pnpm tauri dev` 启动后界面显示「内核已连接」；关窗后 Go 进程被回收；`pnpm tauri build` 产物能拉起 sidecar。

### 阶段 1 — 工具平台

1. 全局布局：左侧栏（图标 + 文字）＋ 右侧内容区，HashRouter 路由 `/` 与 `/tool/:id`
2. 前端 registry 与卡片/侧栏共用的元数据；首页宫格卡片
3. Go `Tool` 接口、注册表、`/api/tools`
4. 落一个纯前端示例工具（JSON 格式化）验证「无后端也能接入」
5. 启动时运行时校验（缺失的后端工具标记不可用）

验证：首页卡片与左侧栏自动一致；新增示例工具只改 registry；停用后端工具时 UI 显示不可用。

### 阶段 2 — 抓包 MVP

1. CA 生成与持久化，`TLSConfigFromCA` 注入，`CertStorage` 证书缓存
2. 代理启停、HTTP 与 HTTPS MITM、`AllowHTTP2` 开关
3. Flow 采集（头、body、耗时、大小、错误）、环形缓冲存储
4. `flows:batch` 批量推送，详情按需拉取
5. 前端流量列表（虚拟滚动）、筛选、详情四个 Tab

验证：本地 echo 服务 + curl + 浏览器全通过；每秒 200 请求时列表不卡顿、事件不堆积。

### 阶段 3 — 证书分发与手机接入

1. 证书下载端口、安装引导页、局域网 IP 探测
2. 局域网模式：绑定指定 IP、IP 白名单、退出自动关闭
3. 前端二维码与分平台步骤（iOS 两步信任、Android 用户证书限制、macOS 钥匙串、Windows 根证书）
4. 证书指纹与有效期展示

验证：手机在同一 WiFi 下访问下载页并安装证书，浏览器与 App 的 HTTPS 均能解密；未配对的设备无法使用代理。

### 阶段 4 — 断点与重放

1. 断点规则模型与匹配
2. 请求/响应挂起、超时自动放行、停止时统一释放
3. 前端断点编辑面板与决策提交
4. 重放：可改 method/url/headers/body，记录重放结果

验证：命中规则后修改请求头放行，服务端收到修改后的值；重放得到一致响应；WebSocket 握手可改。

### 阶段 5 — 导出、系统代理、打包

1. HAR 1.2 导出（单条与批量）
2. macOS 系统代理完整开关：快照落盘、设置、还原、启动残留检测、单实例
3. Windows：系统代理状态检测与手动引导（自动化排后续）
4. `pnpm tauri build` 打包 macOS 与 Windows，确认 sidecar 被正确带入
5. README：开发命令、工具接入指南、使用边界说明

验证：HAR 可被 Chrome DevTools 导入；开关前后 `networksetup -getwebproxy` 状态正确；强制杀掉进程后重启能检测并还原；打包产物可独立运行。

### 后续增强

- Windows 系统代理自动设置（WinINET + `InternetSetOption` 刷新）
- 特权 helper，保证崩溃后自动还原
- Linux 支持

## 验证方式

- **本地 echo 服务**：Go 写一个测试服务（HTTP/HTTPS/WS/SSE/分块传输/慢响应/大文件），作为固定回归目标，**不依赖 httpbin 等外网服务**
- Go 单测：CA 生成与证书缓存、断点规则匹配、HAR 序列化、系统代理快照往返
- 端到端抓包：curl（HTTP/HTTPS）、浏览器访问真实站点、WebSocket、SSE、大文件下载
- 手机端：同一 WiFi 下配置代理 + 安装证书，访问 HTTPS 页面验证解密；未配对设备访问被拒
- 重放/断点：手动构造请求验证修改生效
- 打包：`pnpm tauri build` 后运行产物，确认 WebView、WS 推送、sidecar、系统代理在打包环境中都正常（dev 正常不代表打包正常）

## 安全与合规

- 仅用于监听**本人设备**与**已获授权**的流量；UI 内明确提示这一使用边界
- CA 私钥仅保存在本机应用数据目录（权限 `0600`），不随应用分发
- API 绑定 `127.0.0.1` 并用每次启动随机生成的 token 鉴权；证书下载页只暴露静态说明与公钥证书
- 代理与证书页默认只监听回环地址；局域网模式必须显式开启，绑定指定网卡 IP，并启用 IP 白名单
- 支持一键清除已安装的根证书指引

## 风险与待实测项

### 阶段 0 必须验证

1. **goproxy 兼容矩阵**：目标站点、HTTP/2、WebSocket、SSE、分块传输、大文件；若问题严重，回退 `AllowHTTP2=false`，仍不行则换 `lqqyt2423/go-mitmproxy`（MIT）。注意 `AdguardTeam/gomitmproxy` 是 GPL-3.0，未来若闭源分发不要选
2. **goproxy 版本与维护状态**：本文档不锁定版本号，spike 时用实际拉到的版本记录结论
3. **sidecar stdin 行为**：Tauri shell plugin 下 stdin 是否保持管道，决定能否用 EOF 触发还原
4. **`networksetup` 是否需要提权**：决定 macOS 系统代理功能的产品形态
5. **Rust 壳的代码量**：如果为了打通进程管理就超过 100 行，说明方案偏离了「Rust 只写壳」的约束，需要复查

### 已知高风险

1. **系统代理崩溃残留**：Rust 与 Go 都可能被强杀，最终只能靠快照落盘 + 启动检测。第一版以「快照 + 检测 + 提示 + 手动还原」兜底
2. **与现有代理冲突**：本机当前存在 8118/8119 代理，绝不静默覆盖；替换前必须提示并保留快照
3. **Windows 代理机制差异**：WinINET 与 WinHTTP 是两套配置，只改一处会出现「浏览器能抓、命令行抓不到」。第一版不自动改，等实测后再做
4. **手机端信任限制**：iOS 必须手动开启完全信任；Android 7+ 大多数 App 不信任用户证书，需在 UI 提前说明，避免「装了却没抓到」的误解
5. **局域网开放代理**：默认回环 + 显式开启 + IP 白名单，三层缺一不可
6. **Tauri 打包签名/公证**：sidecar 需在打包内保留可执行权限；本地自用先做 ad-hoc 签名，正式分发再处理 Developer ID / Windows 代码签名与公证
7. **固定端口 8899/8897 冲突**：实现自动顺延并在 UI 显示实际端口
8. **两个实例同时运行**：单实例插件与 Go 文件锁必须都生效，否则系统代理快照会被互相覆盖
