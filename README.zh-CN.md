<h1 align="center">SideX</h1>

<p align="center">
  <strong>VSCode 的工作台，不依赖 Electron。</strong>
</p>

<p align="center">
  <a href="https://discord.gg/8CUCnEAC4J"><img src="https://img.shields.io/badge/Discord-加入-5865F2?style=for-the-badge&logo=discord&logoColor=white" alt="Discord"></a>
  <a href="https://github.com/Sidenai/sidex/issues"><img src="https://img.shields.io/badge/贡献-欢迎-brightgreen?style=for-the-badge" alt="Contributing"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/许可证-MIT-blue?style=for-the-badge" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/构建于-Tauri_2-FFC131?style=for-the-badge&logo=tauri&logoColor=white" alt="Built with Tauri">
</p>

<br>

<p align="center">
  <img src="./docs/assets/preview.jpg" alt="SideX — 运行于 Tauri 上的 VSCode 工作台" width="900">
</p>

<br>

<p align="center">
  <a href="#为什么">为什么</a> · <a href="#功能现状">功能现状</a> · <a href="#快速开始">快速开始</a> · <a href="#架构简介">架构简介</a> · <a href="#安全风险分析">安全风险分析</a> · <a href="#贡献">贡献</a>
</p>

---

SideX 是 Visual Studio Code 的移植版本，将 Electron 替换为 [Tauri](https://tauri.app/)——一个 Rust 后端加上操作系统原生 WebView。同样的 TypeScript 工作台、同样的编辑器、终端和 Git 集成，无需捆绑浏览器即可运行。

> **早期发布版本。** 核心编辑功能和终端已较稳定。扩展宿主和调试器仍在开发中。详见[功能现状](#功能现状)。

---

## 为什么

VSCode 的内存占用几乎完全来自其捆绑的 Chromium，而非编辑器本身。Tauri 用系统内置的 WebView 取而代之——macOS 上是 WKWebView，Windows 上是 WebView2——在多个应用间共享，几乎不产生额外开销。

<p align="center">
  <img src="./docs/assets/compare.jpg" alt="SideX 16.4 MB vs Visual Studio Code 797.8 MB" width="760">
</p>

内存节省效果在 macOS 上测试最为充分（WKWebView 与 Safari 共享）。Windows 上情况更复杂——根据测量方式不同，WebView2 内存用量有时看起来更高，这是 [Tauri 生态系统正在积极解决的问题](https://github.com/tauri-apps/tauri/issues/5889)。目标是在 macOS 上**空闲时内存低于 200 MB**。等应用足够稳定后，我们会发布真实的基准测试数据。

---

## 功能现状

**已稳定：**

- 带语法高亮和基础智能感知的 Monaco 编辑器
- 文件资源管理器——打开文件夹、创建、重命名、删除
- 集成终端——基于 Rust 的完整 PTY、Shell 检测、调整大小、信号处理
- Git——状态、差异、日志、暂存、提交、分支、推送/拉取/获取、储藏、重置
- 主题——VSCode 目录中的多个内置主题
- 原生操作系统菜单（macOS、Windows、Linux）
- 从 [Open VSX](https://open-vsx.org/) 安装扩展
- 文件监视、文件搜索、全文搜索、Rust 搜索索引
- SQLite 存储、文档管理（自动保存、撤销/重做、编码）

---

## 快速开始

### 系统要求

| 系统 | 要求 |
|------|------|
| **macOS** | 10.15 (Catalina) 或更高版本（建议使用 12 Monterey 及以上，10.15 已停止安全更新） |
| **Windows** | Windows 10（已安装 WebView2）或更高版本 |
| **Linux** | libwebkit2gtk-4.1、libgtk-3 |
| **Node.js** | 建议 20 LTS 或更高版本 |
| **Rust** | 稳定版工具链（通过 `rustup` 安装） |

### 开发模式运行

```bash
git clone https://github.com/Sidenai/sidex.git
cd sidex
npm install
npm run tauri dev
```

### 从源码构建

```bash
npm install

# macOS / Linux
NODE_OPTIONS="--max-old-space-size=12288" npm run build

# Windows (PowerShell)
$env:NODE_OPTIONS="--max-old-space-size=12288"
npm run build

npx tauri build
```

首次构建需要 5–10 分钟（Rust 编译时间）。目前尚未分发预构建的二进制文件。

### 安装 Rust 工具链

如果尚未安装 Rust：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

---

## 架构简介

SideX 将 VSCode 的 Electron 架构逐层映射到 Tauri 上：

| VSCode（Electron） | SideX（Tauri） |
|---|---|
| Electron 主进程 | Tauri Rust 后端 |
| `BrowserWindow` | `WebviewWindow` |
| `ipcMain` / `ipcRenderer` | `invoke()` + Tauri 事件 |
| Node.js `fs`、`pty` 等 | Rust 命令（`std::fs`、`portable-pty`） |
| 菜单 / 对话框 / 剪贴板 | Tauri 插件 |
| 渲染进程（DOM + TypeScript） | 相同——运行于原生 WebView 中 |
| 扩展宿主 | Sidecar 进程（开发中） |

TypeScript 前端是 VSCode 工作台的直接移植。Rust 后端位于 `src-tauri/src/commands/`，处理所有原本属于 Node.js 原生模块的工作：文件 I/O、终端 PTY、Git、文件监视、搜索索引、SQLite 和进程管理。

### 项目结构

```
sidex/
├── src/                    # TypeScript 工作台（从 VSCode 移植）
│   └── vs/
│       ├── base/           # 核心工具
│       ├── platform/       # 平台服务和依赖注入
│       ├── editor/         # Monaco 编辑器
│       └── workbench/      # IDE 外壳、面板、功能、贡献
├── src-tauri/              # Rust 后端
│   └── src/
│       ├── commands/       # fs、终端、git、搜索、调试等
│       ├── lib.rs          # 应用初始化和命令注册
│       └── main.rs         # 入口点
├── index.html
├── vite.config.ts
└── package.json
```

### 技术栈

| 层级 | 技术 |
|---|---|
| 前端 | TypeScript、Vite 6、Monaco 编辑器 |
| 终端 UI | xterm.js + WebGL 渲染器 |
| 语法 / 主题 | vscode-textmate、vscode-oniguruma（WASM） |
| 后端 | Rust、Tauri 2 |
| 终端 | portable-pty（Rust） |
| 文件监视 | notify crate（macOS 上使用 FSEvents） |
| 搜索 | dashmap + rayon + regex（并行、Rust） |
| 存储 | 通过 rusqlite 使用 SQLite |
| 扩展 | Open VSX 注册表 |

---

## 安全风险分析

以下是对项目当前代码库的安全风险评估，旨在帮助用户和贡献者了解潜在的安全注意事项。

### ✅ 已有的安全措施

**1. 路径遍历防护（`validation.rs`）**

所有文件系统命令在执行前都会调用 `validate_path()`，该函数会：

- 拒绝空路径
- 拒绝包含 NUL 字节（`\0`）的路径
- 拒绝包含父目录引用（`..`）的路径，防止目录遍历攻击

**2. 代理请求主机白名单（`proxy.rs`）**

`proxy_request` 和 `proxy_request_full` 命令会对目标主机进行白名单校验，只允许请求以下类型的域名：

- SideX 扩展市场（`marketplace.siden.ai` 等）
- Open VSX（`open-vsx.org` 等）
- Microsoft 扩展市场（`marketplace.visualstudio.com` 等）
- GitHub（`github.com`、`api.github.com` 等）

此外，代理验证函数还会屏蔽：
- 环回地址（`127.x.x.x`、`::1`）
- 私有/链路本地地址（`192.168.x.x`、`10.x.x.x` 等）
- 云元数据端点（`169.254.169.254`）

**3. 更新签名验证**

自动更新使用 Minisign 公钥对更新包进行签名验证（见 `tauri.conf.json` 中的 `pubkey` 字段），防止供应链攻击和中间人篡改。

**4. 敏感环境变量过滤（`os.rs`）**

`get_all_env` 命令会过滤掉包含以下关键词的环境变量，防止敏感凭证泄露：

`SECRET`、`TOKEN`、`PASSWORD`、`API_KEY`、`AWS_`、`AZURE_`、`GCP_` 等。

**5. 扩展 ID 清理**

安装扩展时，会对扩展 ID 进行清理（`sanitize_ext_id`），防止通过恶意扩展 ID 进行路径遍历。

**6. git clone URL 方案验证**

`git_clone` 命令会校验 URL 方案，只允许 `https`、`http`、`ssh` 和 `git`，拒绝 `file://` 等危险方案。

---

### ⚠️ 已识别的安全风险

**风险 1：CSP（内容安全策略）为空**

**位置：** `src-tauri/tauri.conf.json`

```json
"security": {
  "csp": null
}
```

**说明：** WebView 没有配置内容安全策略（CSP），这意味着如果前端存在 XSS 漏洞，攻击者可以注入任意脚本，并通过 Tauri IPC 与 Rust 后端交互，执行文件系统操作、终端命令等高权限操作。

**建议：** 配置严格的 CSP，限制脚本来源、禁止内联脚本执行。

---

**风险 2：`fetch_url` / `fetch_url_text` 不受主机白名单限制**

**位置：** `src-tauri/src/commands/proxy.rs`

```rust
pub async fn fetch_url(url: String) -> Result<Vec<u8>, String> {
    let parsed = validate_url(&url)?;  // 仅验证格式和 IP，不检查白名单
    ...
}
```

**说明：** `fetch_url` 和 `fetch_url_text` 命令只进行基本的 URL 格式和 IP 地址验证，但**不检查主机白名单**。相比之下，`proxy_request` 会进行白名单校验。这意味着前端代码（或被注入的脚本）可以通过这两个命令向任意互联网地址发起请求，可能导致信息泄露或被用于 SSRF 攻击。

**建议：** 对 `fetch_url` / `fetch_url_text` 也启用主机白名单，或限制其只能在特定场景下使用。

---

**风险 3：扩展从任意 URL 安装**

**位置：** `src-tauri/src/commands/extensions.rs`

```rust
pub async fn install_extension_from_url(url: String) -> Result<InstalledExtension, String> {
    let resp = reqwest::get(&url).await...  // 无主机白名单校验
}
```

**说明：** `install_extension_from_url` 可以从任意 HTTP/HTTPS URL 下载并安装扩展包（VSIX）。虽然安装前会对 VSIX 包进行格式校验（`validate_vsix`），但不对下载来源进行白名单限制。若前端存在漏洞或攻击者控制了前端，可能安装来自恶意源的扩展。

**建议：** 对扩展下载 URL 进行白名单校验（限制在 Open VSX、VS Code 市场等可信域名），或在安装前对扩展签名进行校验。

---

**风险 4：调试适配器可启动任意可执行文件**

**位置：** `src-tauri/src/commands/debug.rs`

```rust
pub fn debug_spawn_adapter(
    executable: String,
    args: Option<Vec<String>>,
    ...
) -> Result<u32, String> {
    let mut cmd = Command::new(&executable);  // 无路径限制
}
```

**说明：** `debug_spawn_adapter` 命令接受任意可执行文件路径，可以启动系统中的任何程序。虽然这是调试器正常工作所必需的，但若该命令被滥用，存在任意代码执行的风险。

**建议：** 对 `executable` 参数进行校验，只允许已知调试适配器的路径，或限制在应用数据目录内。

---

**风险 5：远程 SSH 命令执行无内容过滤**

**位置：** `src-tauri/src/commands/remote.rs`

```rust
pub async fn remote_exec_ssh(
    connection_id: u64,
    command: String,  // 任意命令字符串
    ...
) -> Result<RemoteExecResult, String>
```

**说明：** `remote_exec_ssh` 允许在已建立的 SSH 连接上执行任意命令字符串。虽然 SSH 连接本身是授权的（用户主动建立），但该接口没有对命令内容进行任何过滤或限制，若前端被注入恶意内容，可能导致远程命令注入。

**建议：** 在文档中明确说明该命令的风险，并在前端层面限制其调用场景。

---

**风险 6：`get_env` 可读取任意环境变量**

**位置：** `src-tauri/src/commands/os.rs`

```rust
pub fn get_env(key: String) -> Option<String> {
    env::var(&key).ok()  // 可读取任意 key，包括敏感变量
}
```

**说明：** `get_all_env` 已对敏感变量进行过滤，但 `get_env` 可以按名称读取**任意**环境变量，包括 `AWS_SECRET_ACCESS_KEY`、`GITHUB_TOKEN` 等敏感凭证。

**建议：** 为 `get_env` 添加与 `get_all_env` 相同的敏感变量过滤逻辑。

---

**风险 7：资产协议访问范围过广**

**位置：** `src-tauri/tauri.conf.json`

```json
"assetProtocol": {
  "enable": true,
  "scope": ["$HOME/**", "/tmp/**"]
}
```

**说明：** 资产协议允许 WebView 访问用户主目录下的**所有文件**（`$HOME/**`），范围较广。若 WebView 存在 XSS 漏洞，攻击者可能利用此协议读取用户的私钥、凭证文件等敏感数据。

**建议：** 将资产协议的访问范围收窄至应用真正需要访问的目录（如应用数据目录、工作区目录等），而非整个主目录。

---

### 🔒 安全使用建议

1. **仅从可信来源安装扩展**：优先通过内置的 Open VSX 扩展市场安装扩展，避免通过 URL 直接安装来源不明的 VSIX 文件。

2. **谨慎打开不受信任的代码仓库**：恶意的 `.vscode/launch.json` 可能配置危险的调试适配器命令。在打开来源不明的项目时请注意相关风险。

3. **保持应用及时更新**：SideX 使用 Minisign 签名的自动更新机制，请及时应用安全更新。

4. **注意 SSH 远程连接**：通过 SideX 建立的 SSH 远程连接赋予编辑器在远程主机上执行命令的能力，请只连接受信任的主机。

5. **在生产/敏感环境中谨慎使用**：SideX 目前处于早期版本，上述安全风险尚未全部修复，建议在生产环境或处理高度敏感数据时评估相关风险。

---

## 贡献

项目早期发布正是为了吸引外部贡献者参与。

### 如何贡献

1. Fork 仓库并创建分支
2. 选择一个任务——查看 [Issues](https://github.com/Sidenai/sidex/issues) 或从已知缺陷列表中选取
3. 提交 PR——贡献者会被记入致谢

### 开发说明

- 遵循 VSCode 的代码模式——如果读过 VSCode 源码会很熟悉
- TypeScript 导入使用 `.js` 扩展名（ES 模块规范）
- 服务使用 VSCode 的 `@inject` 依赖注入装饰器
- 新的 Rust 命令放在 `src-tauri/src/commands/` 并在 `lib.rs` 中注册

---

## 社区

- **Discord：** [加入 SideX 服务器](https://discord.gg/8CUCnEAC4J)
- **X / Twitter：** [@ImRazshy](https://x.com/ImRazshy)
- **邮箱：** kendall@siden.ai

---

## 许可证

MIT——SideX 是 [Visual Studio Code（Code - OSS）](https://github.com/microsoft/vscode) 的移植版，后者同样采用 MIT 许可证。详见 [LICENSE](./LICENSE)。
