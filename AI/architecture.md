# Clyde on Desk — 项目架构文档

> **版本**: v0.1.6 | **协议**: AGPL-3.0 | **仓库**: https://github.com/oahc09/Clyde
>
> 生成时间: 2026-05-16

---

## 1. 项目概述

Clyde on Desk 是一款轻量级跨平台桌面宠物应用，实时镜像 AI 编码代理的工作状态。宠物会根据代理事件呈现 12 种动画状态：思考、打字、杂耍、报错、庆祝、睡眠等。

**支持的 AI 代理**（可同时运行）：

| 代理 | 接入方式 | 权限气泡 | 终端聚焦 |
|------|---------|---------|---------|
| Claude Code | HTTP Hook + JSONL 轮询 | ✅ | ✅ |
| Codex CLI | JSONL 日志轮询 | ❌ | ❌ |
| Copilot CLI | HTTP Hook | ❌ (仅 deny) | ✅ |

**核心指标**：

| 指标 | 数值 |
|------|------|
| 安装包体积 | ~5 MB (Tauri, 非 Electron 的 150+ MB) |
| 冷启动时间 | < 1 s |
| 常驻内存 | < 30 MB |
| Tick 频率 | 50 ms (光标追踪 / 眼睛跟随 / 睡眠检测) |

---

## 2. 技术栈

```
┌────────────────────────────────────────────────────────────┐
│                       Svelte 5 前端                        │
│  (4 个窗口: pet / hit / bubble / menu, 共 < 30KB JS)       │
├────────────────────────────────────────────────────────────┤
│                       Tauri v2 框架                         │
│  (IPC 桥接, 透明窗口, 系统托盘, 原生 API)                    │
├────────────────────────────────────────────────────────────┤
│                       Rust 后端                             │
│  Axum HTTP + Tokio 异步运行时 + 状态机 + 多会话管理          │
├────────────────────────────────────────────────────────────┤
│                       操作系统 API                           │
│  Win32 / macOS NSWorkspace / Linux wmctrl+xdotool          │
└────────────────────────────────────────────────────────────┘
```

| 层级 | 技术 | 选型理由 |
|------|------|---------|
| **桌面框架** | Tauri v2 | ~5 MB 包体; 原生 OS API (透明窗口/托盘/全局快捷键); Rust 零 IPC 开销 |
| **后端** | Rust | 无 GC, 零成本抽象; 单进程 50ms 定时器 + 多会话状态机, 近零 CPU; `Mutex` + `Arc` 线程安全 |
| **前端** | Svelte 5 | 编译时框架, 无虚拟 DOM; `$state` / `$props` 响应式, SVG 渲染极轻 |
| **HTTP** | Axum | Tokio 异步, 类型安全路由; 与 Tauri 共享 Tokio runtime |
| **构建** | Vite 6 | HMR 即时热更新; 生产构建激进 tree-shaking |

---

## 3. 项目目录结构

```
Clyde/
├── src-tauri/                    # Rust 后端 (Tauri 应用)
│   ├── Cargo.toml                # Rust 依赖配置
│   ├── tauri.conf.json           # Tauri 窗口/打包配置
│   └── src/
│       ├── main.rs               # 应用入口, panic hook 设置
│       ├── lib.rs                # 核心主文件 (67KB): Tauri 命令, 拖拽, 事件处理
│       ├── state_machine.rs      # 多会话状态机 + 优先级决议
│       ├── http_server.rs        # Axum HTTP 服务 (POST /state, /permission, /elicitation)
│       ├── hooks.rs              # Hook 脚本部署 + Claude settings.json 注册
│       ├── permission.rs         # 权限气泡窗口管理
│       ├── permission_mode.rs    # Claude 权限模式追踪 (Default/AcceptEdits/Bypass/Plan)
│       ├── session_meta.rs       # 会话元数据解析 (summary, cwd, agent label)
│       ├── claude_monitor.rs     # Claude Code JSONL 会话轮询 (~/.claude/projects/)
│       ├── codex_monitor.rs      # Codex CLI JSONL 日志轮询 (~/.codex/sessions/)
│       ├── tick.rs               # 50ms 光标轮询 (眼睛跟随/睡眠/Mini peek)
│       ├── mini.rs               # 边缘吸附/Mini模式/抛物线跳跃
│       ├── windows.rs            # 窗口坐标/Hit-test/DPI/多显示器
│       ├── hit_regions.rs        # 透明窗口点击区域计算
│       ├── focus.rs              # 按PID聚焦终端 (Win32/macOS/Linux)
│       ├── tray.rs               # 系统托盘菜单
│       ├── prefs.rs              # 偏好设置持久化 (JSON)
│       ├── i18n.rs               # 英文/中文国际化
│       ├── environment.rs        # 环境检测 (全屏/会议自动隐藏, 仅 macOS)
│       ├── update_check.rs       # GitHub Releases 版本检查
│       ├── macos_spaces.rs       # macOS Space 切换处理
│       └── util.rs               # Mutex 恢复扩展 trait
│
├── src/                          # Svelte 5 前端
│   ├── lib/
│   │   └── stores.ts             # Svelte 全局 store (状态/SVG/DND/语言)
│   └── windows/
│       ├── pet/                  # 宠物主窗口: SVG 动画渲染 + 眼睛跟随
│       │   ├── App.svelte
│       │   ├── index.html
│       │   └── main.ts
│       ├── hit/                  # 透明点击层: 拖拽/双击/右键交互
│       │   ├── App.svelte
│       │   ├── index.html
│       │   └── main.ts
│       ├── bubble/               # 权限气泡窗口: Allow/Deny/规则
│       │   ├── App.svelte
│       │   ├── BubbleCard.svelte  # 气泡卡片组件 (33KB, 功能最复杂)
│       │   ├── index.html
│       │   └── main.ts
│       └── menu/                 # 右键上下文菜单
│           ├── App.svelte
│           ├── index.html
│           └── main.ts
│
├── hooks/                        # JS Hook 脚本 (编译时嵌入二进制)
│   ├── clyde-hook.js             # Claude Code 命令 Hook (事件→状态映射)
│   ├── copilot-hook.js           # Copilot CLI 命令 Hook
│   ├── server-config.js          # 端口发现 + HTTP 发送
│   ├── auto-start.js             # SessionStart 自动启动 Clyde
│   ├── install.js                # 手动 Hook 注册 CLI
│   ├── package.json
│   └── tests/                    # Hook 单元测试
│
├── assets/
│   ├── svg/                      # 35+ SVG 动画帧
│   ├── gif/                      # 预览 GIF
│   └── tray-icon.png
│
├── AI/                           # AI 生成文档目录
├── package.json                  # Node.js 依赖 (Tauri CLI, Svelte, Vite)
├── vite.config.ts                # Vite 多入口配置 (pet/hit/bubble/menu)
├── tsconfig.json
└── svelte.config.js
```

---

## 4. 系统架构

### 4.1 整体数据流

```
                    ┌──────────────────────┐
                    │    AI Coding Agent    │
                    │  (Claude/Codex/Copilot)│
                    └──────┬───────┬───────┘
                           │       │
            ┌──────────────┘       └──────────────┐
            ▼ Command Hook                        ▼ JSONL Polling
    ┌───────────────┐                     ┌───────────────────┐
    │ hooks/*.js    │                     │ claude_monitor.rs │
    │ (Node.js)     │                     │ codex_monitor.rs  │
    └───────┬───────┘                     └─────────┬─────────┘
            │ HTTP POST                             │ 文件读取
            ▼                                       ▼
    ┌─────────────────────────────────────────────────────┐
    │              Axum HTTP Server (:23333)               │
    │  POST /state     POST /permission  POST /elicitation│
    └─────────────────────────┬───────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────┐
    │                  State Machine                       │
    │  ┌─────────────────────────────────────────────┐    │
    │  │  sessions: HashMap<SessionId, SessionEntry>  │    │
    │  │  current_state → SVG → emit("state-change")  │    │
    │  │  优先级: error > notification > sweeping >    │    │
    │  │          attention > carrying > juggling >    │    │
    │  │          working > thinking > idle > sleeping  │    │
    │  └─────────────────────────────────────────────┘    │
    └─────────────────────────┬───────────────────────────┘
                              │ Tauri Events
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                   ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
   │  pet Window  │  │  hit Window  │  │  bubble Windows  │
   │  SVG 渲染    │  │  透明交互层  │  │  权限审批卡片    │
   └──────────────┘  └──────────────┘  └──────────────────┘
```

### 4.2 多窗口架构

Tauri 管理的窗口由 `tauri.conf.json` 和运行时动态创建：

| 窗口 | Label | 大小 | 透明 | 置顶 | 用途 |
|------|-------|------|------|------|------|
| 宠物 | `pet` | 200×200 (S) / 280×280 (M) / 360×360 (L) | ✅ | ✅ | SVG 动画渲染 |
| 点击层 | `hit` | 60×60 | ✅ | ✅ | 透明拖拽/点击区域 |
| 权限气泡 | `bubble-{id}` | 340×动态 | ✅ | ✅ | 运行时动态创建, 权限审批 |
| 右键菜单 | `menu` | 动态 | ✅ | ✅ | 运行时动态创建 |

**关键设计**：pet 窗口负责渲染 SVG, hit 窗口覆盖其上负责捕获鼠标事件（拖拽、双击、右键）。两者通过 Rust 后端同步位置和尺寸。

### 4.3 事件映射表

**Hook 事件 → 状态映射**（定义在 3 处，需同步更新）：

| Claude Code Event | Copilot Event | State | SVG |
|-------------------|--------------|-------|-----|
| SessionStart | sessionStart | idle | clyde-idle-follow.svg |
| UserPromptSubmit | userPromptSubmitted | thinking | clyde-working-thinking.svg |
| PreToolUse | preToolUse | working | clyde-working-typing.svg |
| PostToolUse | postToolUse | thinking | clyde-working-thinking.svg |
| PostToolUseFailure | errorOccurred | error | clyde-error.svg |
| Stop | agentStop | attention | clyde-happy.svg |
| SubagentStart | subagentStart | juggling | clyde-working-juggling.svg |
| SubagentStop | subagentStop | idle | clyde-idle-follow.svg |
| Notification | - | notification | clyde-notification.svg |
| PreCompact | preCompact | sweeping | clyde-working-sweeping.svg |
| WorktreeCreate | - | carrying | clyde-working-carrying.svg |
| 3+ sessions | - | working | clyde-working-building.svg |
| 60s 无活动 | - | sleeping | clyde-sleeping.svg |

**一次性状态** (Oneshot)：`attention`, `error`, `notification`, `sweeping`, `carrying` — 播放动画后不持久化为会话状态。

---

## 5. 核心模块详解

### 5.1 状态机 (`state_machine.rs`)

```
StateMachine
├── current_state: String          # 当前显示状态
├── current_svg: String            # 当前 SVG 文件名
├── sessions: HashMap<String, SessionEntry>  # 多会话跟踪
├── manual_dnd: bool               # 手动勿扰
├── auto_dnd: bool                 # 自动勿扰 (会议检测)
├── dnd: bool                      # 最终 DND 状态 = manual || auto
└── auto_hidden: bool              # 全屏自动隐藏

SessionEntry
├── state: String                  # 会话当前状态
├── updated_at: Instant            # 最后更新时间
├── source_pid: Option<u32>        # 终端进程 PID
├── cwd: String                    # 工作目录
├── agent_id: String               # "claude-code" / "codex" / "copilot"
└── summary: String                # 会话摘要
```

**优先级决议**：`resolve_display_state()` 遍历所有活跃会话，取优先级最高的状态。

**过期清理**：
- 工作状态 (working/juggling/thinking) 超过 **5 分钟**降级为 idle
- 整个会话超过 **10 分钟**未更新则移除

**juggling 保护**：一旦进入 juggling 状态，不会被普通的 working 事件降级（除非收到 SubagentStop 事件）。

### 5.2 HTTP 服务 (`http_server.rs`)

```
Axum Router (共享 Tokio runtime)
├── POST /state           # Hook 发送状态更新
├── POST /permission      # 权限审批请求 (阻塞等待用户决策)
├── POST /elicitation     # 交互式请求 (Accept/Decline/Cancel)
├── GET  /health          # 健康检查
├── POST /clear-permissions  # 清除权限请求
└── Request Watchdog      # 超时自动 Deny (默认 300s)
```

**端口**: `127.0.0.1:23333`（默认），仅绑定本地回环。

**审批队列** (`ApprovalQueue`)：多个权限请求按顺序排队，前一个处理完才显示下一个。

**Hook 决策类型**：
- `Permission(Allow | Deny | AllowWithPermissions)` — 权限审批
- `Elicitation(Accept | Decline | Cancel)` — 交互式请求

### 5.3 Hook 系统 (`hooks.rs` + `hooks/`)

**编译时嵌入**：Hook JS 脚本通过 `include_str!()` 嵌入 Rust 二进制，部署时写入 `~/.claude/hooks/`。

**注册流程**：
1. 部署 `clyde-hook.js`, `server-config.js`, `auto-start.js` 到 `~/.claude/hooks/`
2. 修改 `~/.claude/settings.json`，注册命令 Hook 到 12 个事件
3. 注册 HTTP Hook 到 `PermissionRequest` 事件（URL: `http://127.0.0.1:23333/permission`）
4. 清理旧版 "clawd" 命名文件

**进程树遍历**（`clyde-hook.js`）：从 `process.ppid` 向上最多走 8 层，查找已知终端应用（Windows Terminal, iTerm2, Alacritty 等），获得稳定的终端 PID 用于后续聚焦。

### 5.4 50ms Tick 循环 (`tick.rs`)

```
TickState
├── mouse_still_since: Instant     # 光标静止起始时间
├── has_triggered_yawn: bool       # 已触发打哈欠
├── has_triggered_wake: bool       # 已触发唤醒
├── last_eye_dx/dy: f64            # 上次眼睛偏移 (去抖)
└── peek_phase: PeekPhase          # Mini 模式窥视状态机

PeekPhase: Hidden → Peeking → Retracting → Hidden
```

**功能**：
1. **眼睛跟随**：idle 状态下，计算光标相对于宠物中心的偏移，驱动 SVG 中 `#eyes-js` / `#body-js` / `#shadow-js` 的 transform
2. **睡眠触发**：光标静止 ≥ 60s → 打哈欠 → 打盹 → 倒下 → 睡眠（渐进动画序列）
3. **唤醒**：光标移动时从睡眠状态恢复
4. **Mini 窥视**：三阶段状态机防止窥视振荡

### 5.5 Mini 模式 (`mini.rs`)

```
Mini Mode 流程:
1. 拖拽宠物到屏幕左/右边缘 (30px 容差)
2. 触发吸附预览 (缩小到 70%, 透明度 60%)
3. 释放 → 抛物线滑入动画
4. 宠物隐藏在边缘，仅露出一小部分
5. 光标靠近 → peek 窥出动画
6. 光标离开 → 回缩动画
```

**动画世代计数器** (`AnimationGeneration`)：AtomicU64 单调递增，新的动画取消旧动画，防止状态竞争。

### 5.6 权限系统 (`permission.rs` + `permission_mode.rs`)

**气泡窗口**：
- 运行时动态创建 Tauri 窗口（label: `bubble-{id}`）
- 窗口宽度固定 340px，高度自适应内容
- 多个气泡从宠物位置向上堆叠
- 用户在终端先响应则自动消失

**权限模式追踪** (`permission_mode.rs`)：

| 模式 | 含义 |
|------|------|
| Default | 工具调用需审批 |
| AcceptEdits | 编辑自动通过，其他工具仍需审批 |
| BypassPermissions | 不弹审批气泡 |
| Plan | 仅规划，不执行工具 |

模式变更时弹出通知气泡，来源优先级：Hook > Transcript > Settings。

### 5.7 窗口管理 (`windows.rs` + `hit_regions.rs`)

- **多显示器支持**：通过 Tauri Monitor API 获取显示器列表，计算宠物所在显示器
- **DPI 缩放**：`pet_scale_factor()` 获取 DPI 缩放因子，物理像素/逻辑像素转换
- **Hit-test 区域**：在宠物 SVG 上定义可交互区域（身体/头部/脚部），通过透明 hit 窗口捕获
- **启动位置修复**：确保宠物窗口在可见屏幕区域内（`STARTUP_MIN_VISIBLE_LP = 120px`）

### 5.8 终端聚焦 (`focus.rs`)

| 平台 | 实现方式 |
|------|---------|
| Windows | `EnumWindows` 遍历窗口 → 匹配 PID → `SetForegroundWindow` (ALT 键绕过前台锁) |
| macOS | AppleScript: `tell application "System Events" to set frontmost` |
| Linux | `wmctrl -ip` 或 fallback `xdotool search --pid ... windowfocus` |

所有聚焦操作在独立 OS 线程执行，不阻塞 Tokio runtime。

### 5.9 会话监控

**Claude Monitor** (`claude_monitor.rs`)：
- 轮询 `~/.claude/projects/**/*.jsonl`
- 2s 间隔, 首次跳到文件末尾，只处理新增行
- 解析 JSONL 事件映射到状态

**Codex Monitor** (`codex_monitor.rs`)：
- 轮询 `~/.codex/sessions/YYYY/MM/DD/*.jsonl`
- 1.5s 间隔
- 运行在独立 OS 线程（避免同步 I/O 阻塞 Tokio）

### 5.10 偏好设置 (`prefs.rs`)

持久化到 `{app_data_dir}/clyde-prefs.json`，原子写入（先写 `.tmp` 再 rename）。

| 设置项 | 类型 | 默认值 |
|--------|------|--------|
| size | String | "S" (S/M/L → 200/280/360) |
| mini_mode | bool | false |
| lang | String | "en" |
| opacity | f32 | 1.0 (范围 0.4-1.0) |
| lock_position | bool | false |
| click_through | bool | false |
| auto_hide_fullscreen | bool | false |
| auto_dnd_meetings | bool | false |
| auto_start_with_claude | bool | false |
| permission_decision_window_secs | u16 | 12 (范围 8-120) |
| check_for_updates | bool | true |
| monitor_positions | HashMap | {} (多显示器位置记忆) |

---

## 6. 前端架构

### 6.1 技术实现

- **Svelte 5** 编译时框架，使用 `$state` / `$props` rune 响应式
- **SVG 动画**：35+ 个 SVG 文件通过 Vite `import.meta.glob` 预加载，构建时缓存在 `svgCache` 对象中
- **Tauri Event** 驱动：所有 UI 更新通过 `listen()` 监听 Rust 后端事件

### 6.2 Pet 窗口 (`src/windows/pet/App.svelte`)

```
监听事件:
├── state-change     → 切换 SVG 动画 + 翻转方向
├── eye-move         → 瞳孔/身体/阴影 CSS transform
├── dnd-change       → 勿扰模式切换
├── play-click-reaction → 点击反应动画 (临时覆盖)
├── pet-config-changed  → 透明度更新
├── start-drag-reaction → 拖拽表情
├── snap-preview     → Mini 模式吸附预览
├── trigger-yawn/wake   → 睡眠/唤醒序列
├── mini-peek-in/out    → Mini 窥视动画
└── set-size/set-lang   → 大小/语言切换
```

SVG 中的动态元素通过 `id` 标识（`eyes-js`, `body-js`, `shadow-js`），由 JS 直接操作 CSS transform，transition 80ms 平滑过渡。

### 6.3 Store (`src/lib/stores.ts`)

```typescript
PetState = 'idle' | 'yawning' | 'dozing' | 'collapsing' | 'thinking' |
           'working' | 'juggling' | 'sweeping' | 'error' | 'attention' |
           'notification' | 'carrying' | 'sleeping' | 'waking' |
           'mini-idle' | 'mini-alert' | 'mini-happy' | 'mini-peek' |
           'mini-enter' | 'mini-sleep'

writable stores: currentState, currentSvg, dndEnabled, currentLang
```

### 6.4 构建 (`vite.config.ts`)

4 个入口点 (Rollup multi-entry)：
```
pet    → src/windows/pet/index.html
hit    → src/windows/hit/index.html
bubble → src/windows/bubble/index.html
menu   → src/windows/menu/index.html
```

目标平台：Windows → chrome105, macOS → safari13, Linux → es2021+chrome100。

---

## 7. 依赖关系

### 7.1 Rust 依赖 (`Cargo.toml`)

| 依赖 | 版本 | 用途 |
|------|------|------|
| tauri | 2 | 桌面框架核心 |
| tauri-plugin-single-instance | 2 | 单实例锁 |
| axum | 0.7 | HTTP 服务 |
| tokio | 1 | 异步运行时 |
| serde / serde_json | 1 | 序列化 |
| uuid | 1 | 唯一 ID |
| anyhow | 1 | 错误处理 |
| dirs | 5 | 系统目录路径 |
| open | 5 | 打开 URL/文件 |
| reqwest | 0.12 | HTTP 客户端 (版本检查) |

**平台特定**：
- Windows: `windows` crate (Win32 API)
- macOS: `block2`, `objc2-app-kit`, `objc2-foundation`, `core-foundation`, `core-graphics`
- Unix: `nix` (signal)

### 7.2 JS 依赖 (`package.json`)

| 依赖 | 版本 | 用途 |
|------|------|------|
| @tauri-apps/api | ^2 | Tauri 前端 API |
| @tauri-apps/cli | ^2 | Tauri CLI 工具 |
| svelte | ^5.0 | UI 框架 |
| vite | ^6.0 | 构建工具 |
| typescript | ^5.0 | 类型检查 |

---

## 8. 线程模型

```
┌─────────────────────────────────────────────────────┐
│                    Main Thread                       │
│  Tauri 事件循环, 窗口管理, Tauri Commands            │
├─────────────────────────────────────────────────────┤
│                 Tokio Runtime (Tauri 内置)            │
│  ┌──────────────────┐  ┌──────────────────────┐     │
│  │  Axum HTTP Server │  │  50ms Tick Loop      │     │
│  │  (端口 23333)      │  │  光标/眼睛/睡眠      │     │
│  └──────────────────┘  └──────────────────────┘     │
│  ┌──────────────────┐  ┌──────────────────────┐     │
│  │  Claude Monitor   │  │  Environment Loop    │     │
│  │  (2s JSONL 轮询)  │  │  (2s 全屏/会议检测)  │     │
│  └──────────────────┘  └──────────────────────┘     │
│  ┌──────────────────┐  ┌──────────────────────┐     │
│  │  Update Check     │  │  State Stale Cleanup │     │
│  │  (4h GitHub API)  │  │  (30s 间隔)          │     │
│  └──────────────────┘  └──────────────────────┘     │
├─────────────────────────────────────────────────────┤
│                   OS Threads                         │
│  ┌──────────────────┐  ┌──────────────────────┐     │
│  │  Codex Monitor    │  │  focus-window        │     │
│  │  (1.5s 同步 I/O)  │  │  (Win32 EnumWindows) │     │
│  └──────────────────┘  └──────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

**共享状态保护**：所有共享状态通过 `Arc<Mutex<T>>` 保护，使用 `MutexExt::lock_or_recover()` 从 poison 恢复而非 panic 传播。

---

## 9. 关键流程

### 9.1 启动流程

```
main.rs → clyde_lib::run()
  ├── 加载偏好设置 (prefs.rs)
  ├── 初始化 StateMachine (Arc<Mutex<>>)
  ├── 创建 Tauri 窗口 (pet + hit)
  ├── 启动 Axum HTTP Server (端口 23333)
  ├── 部署 Hook 脚本到 ~/.claude/hooks/ (hooks.rs)
  ├── 注册 Hook 到 ~/.claude/settings.json
  ├── 启动 50ms Tick 循环 (tick.rs)
  ├── 启动 Claude JSONL Monitor (claude_monitor.rs)
  ├── 启动 Codex JSONL Monitor (codex_monitor.rs)
  ├── 启动环境检测循环 (environment.rs)
  ├── 启动版本检查 (update_check.rs)
  └── 设置系统托盘 (tray.rs)
```

### 9.2 权限审批流程

```
Claude Code 触发 PermissionRequest
  → HTTP POST /permission (Axum handler)
  → 创建 BubbleData, 推入 ApprovalQueue
  → 动态创建 bubble 窗口
  → 用户点击 Allow/Deny
  → POST /decision
  → oneshot::Sender 发送决策
  → HTTP handler 返回响应给 Claude Code
  → 关闭 bubble 窗口
  → 处理队列中的下一个请求
```

### 9.3 睡眠序列

```
idle + 光标静止 60s
  → emit("trigger-yawn")
  → 打哈欠动画 (3s)
  → 打盹动画 (4s)
  → 倒下动画 (3s)
  → 睡眠状态 (持续)
  
光标移动
  → emit("trigger-wake")
  → 唤醒动画 (1.5s)
  → 恢复到 resolve_display_state()
```

---

## 10. 已知限制

| 限制 | 详情 |
|------|------|
| Codex 无终端聚焦 | JSONL 轮询不含终端 PID |
| Copilot 无权限气泡 | Copilot hook 协议仅支持 deny |
| HTTP 无认证 | 仅绑定 127.0.0.1; token 认证计划中 |
| 无自动更新 | 需从 GitHub Releases 下载 |
| 环境控制仅 macOS | 全屏/会议检测依赖 macOS NSWorkspace |

---

## 11. 事件-状态映射同步点

新增状态时必须同步更新的位置（代码注释中标注）：

1. `hooks/clyde-hook.js` — `EVENT_TO_STATE` (Claude Code 事件 → 状态)
2. `hooks/copilot-hook.js` — `EVENT_TO_STATE` (Copilot 事件 → 状态)
3. `src-tauri/src/state_machine.rs` — `state_priority()` + `svg_for_state()` (状态 → 优先级/SVG)
4. `src-tauri/src/codex_monitor.rs` — `map_codex_event()` (Codex JSONL 事件 → 状态)
5. `src-tauri/src/claude_monitor.rs` — `map_claude_event()` (Claude session 事件 → 状态)
6. `src/lib/stores.ts` — `PetState` 类型联合

---

*文档生成自源代码分析，覆盖 Rust 后端 22 个源文件 + Svelte 前端 4 个窗口 + JS Hook 脚本。*
