# agent-browser（仓库用途总结）

## 它是什么

`agent-browser` 是一个面向 AI Agent 的浏览器自动化工具：

- **高速 CLI**：`cli/` 下的 Rust 原生 CLI，用于低延迟地解析命令并分发到后台服务。
- **常驻守护进程**：`src/` 下的 Node.js + Playwright daemon，让浏览器在多条命令之间保持运行（更快的后续操作）。
- **Agent 友好的交互模型**：`snapshot` 输出可访问性树（A11y tree）并生成稳定的元素引用（如 `@e1`），Agent 可以用 `click` / `fill` / `get` 等命令对引用进行确定性操作，减少脆弱 CSS 选择器依赖。

## 最适合的使用场景

当你希望 Agent 做这些事情时，这个仓库/工具非常合适：

- 打开网站并完成多步 UI 流程（包括 SPA）
- 表单填写、登录、下单、CRUD 等交互式流程自动化
- 从页面提取小到中等规模的数据（基于 ref/role，更结构化）
- 快速 UI 冒烟验证，并产出调试/复现工件（截图、录屏、trace、profile）

## 优势 / 差异点

- **Snapshot → Ref 工作流**：`snapshot -i`（仅交互元素）+ `@e*` refs 对 LLM 工具链非常友好。
- **机器可读输出**：`--json` 便于与 Agent 框架/编排器集成。
- **会话与状态管理**：隔离 session、持久化 profile、自动保存/恢复状态，且支持可选的落盘加密。
- **网络与调试能力**：请求拦截/Mock、控制台与错误收集、trace/profiler、视频录制。
- **预览/协同**：可选通过 WebSocket 进行视口流式预览，并注入输入事件（人类旁路协作）。
- **多运行目标**：本地 Chromium（Playwright）、通过 CDP 连接到已有 Chrome，以及多种云端 provider；另有 iOS provider（Appium）用于控制 Mobile Safari。

## 不太适合的场景

- 高吞吐大规模爬虫/抓取管线
- 断言密集、规模很大的 E2E 测试体系（此时直接写 Playwright 测试通常更顺手）

## 关键入口（便于定位代码）

- Rust CLI：`cli/src/main.rs`；命令解析/帮助：`cli/src/commands.rs`、`cli/src/output.rs`
- Node daemon + 浏览器管理：`src/daemon.ts`、`src/browser.ts`；协议：`src/protocol.ts`
- Snapshot/Ref 系统：`src/snapshot.ts`
- iOS provider：`src/ios-manager.ts`、`src/ios-actions.ts`
- Streaming：`src/stream-server.ts`
- JS 包装器（调用原生二进制）：`bin/agent-browser.js`
- Agent 技能说明：`skills/agent-browser/SKILL.md`
- 文档站点：`docs/src/app/`

## 关于源码本地编译（实用提示）

源码仓库通常不会提交预编译的原生二进制文件；当 `bin/agent-browser-<platform>` 不存在时，直接运行包装器（例如 `node bin/agent-browser.js --help`）会提示找不到 binary。

在本地从源码编译并把产物复制到 `bin/` 的方式：

```bash
# 1) 安装 Rust（推荐 rustup）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

# 2) 编译 Rust CLI，并复制到 bin/（会生成 bin/agent-browser-<platform>）
npm run build:native

# 3) 验证（通过 JS 包装器调用原生二进制）
node bin/agent-browser.js --help
```

在本机（macOS arm64）执行后，会生成 `bin/agent-browser-darwin-arm64`，之后 `node bin/agent-browser.js ...` 就会直接调用该原生二进制。

注意：原生 CLI 只是“前端”。要在这个源码目录里实际执行 `open` / `snapshot` 等命令，还需要先构建 Node daemon（确保存在 `dist/daemon.js`）并安装依赖，例如：

```bash
npm install
npm run build
```
