---
name: agent-browser
description: 面向 AI Agent 的浏览器自动化 CLI。适用于打开网页、填写表单、点击按钮、截图、提取数据、测试 Web 应用、登录流程、抓取页面等需要“可编程浏览器交互”的任务。
allowed-tools: Bash(npx agent-browser:*), Bash(agent-browser:*)
---

# 使用 agent-browser 做浏览器自动化（面向 AI Agent）

> English version: `skills/agent-browser/SKILL.en.md`

## 核心工作流

所有浏览器自动化都建议遵循这个循环：

1. **打开页面**：`agent-browser open <url>`
2. **抓快照**：`agent-browser snapshot -i`（拿到元素 ref：`@e1`, `@e2`）
3. **交互操作**：用 ref 执行 `click` / `fill` / `select` / `get`
4. **重新快照**：页面跳转或 DOM 变化后必须重新 `snapshot`

```bash
agent-browser open https://example.com/form
agent-browser snapshot -i
# 输出示例：@e1 [input type="email"], @e2 [input type="password"], @e3 [button] "Submit"

agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "password123"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i  # 检查结果
```

## 命令链式执行（推荐）

同一次 shell 调用里可以用 `&&` 串联多条命令。由于 agent-browser 使用常驻 daemon 复用浏览器实例，链式执行通常更快、更稳定。

```bash
# 一次性串联：打开 + 等待 + 快照
agent-browser open https://example.com && agent-browser wait --load networkidle && agent-browser snapshot -i

# 串联多步交互
agent-browser fill @e1 "user@example.com" && agent-browser fill @e2 "password123" && agent-browser click @e3

# 打开并截图
agent-browser open https://example.com && agent-browser wait --load networkidle && agent-browser screenshot page.png
```

**何时适合链式：**中间步骤不需要“读输出再决定下一步”时（例如 open + wait + screenshot）。如果需要先解析 `snapshot` 输出再决定点击哪个 ref，就分开运行。

## 常用命令速查

```bash
# 导航
agent-browser open <url>              # 打开 URL（别名：goto、navigate）
agent-browser close                   # 关闭浏览器

# 快照
agent-browser snapshot -i             # 仅交互元素（推荐，带 refs）
agent-browser snapshot -i -C          # 额外包含 cursor-interactive 元素（onclick、cursor:pointer 等）
agent-browser snapshot -s "#selector" # 限定到某个 CSS selector 范围

# 交互（优先使用 snapshot 输出的 @ref）
agent-browser click @e1               # 点击
agent-browser click @e1 --new-tab     # 新标签页打开
agent-browser fill @e2 "text"         # 清空并填入
agent-browser type @e2 "text"         # 追加输入（不清空）
agent-browser select @e1 "option"     # 选择下拉选项
agent-browser check @e1               # 勾选
agent-browser press Enter             # 按键（Enter、Tab、Control+a 等）
agent-browser scroll down 500         # 页面滚动

# 获取信息
agent-browser get text @e1            # 取文本
agent-browser get url                 # 当前 URL
agent-browser get title               # 页面标题

# 等待
agent-browser wait @e1                # 等元素出现/可见
agent-browser wait --load networkidle # 等网络空闲
agent-browser wait --url "**/page"    # 等 URL 匹配
agent-browser wait 2000               # 等待毫秒

# 截图/导出
agent-browser screenshot              # 截图到临时目录（会输出路径）
agent-browser screenshot --full       # 全页截图
agent-browser screenshot --annotate   # 带数字标签的截图（标注交互元素）
agent-browser pdf output.pdf          # 导出 PDF
```

## 常见模式

### 表单提交

```bash
agent-browser open https://example.com/signup
agent-browser snapshot -i
agent-browser fill @e1 "Jane Doe"
agent-browser fill @e2 "jane@example.com"
agent-browser select @e3 "California"
agent-browser check @e4
agent-browser click @e5
agent-browser wait --load networkidle
```

### 登录与状态复用（state）

```bash
# 登录一次并保存 state
agent-browser open https://app.example.com/login
agent-browser snapshot -i
agent-browser fill @e1 "$USERNAME"
agent-browser fill @e2 "$PASSWORD"
agent-browser click @e3
agent-browser wait --url "**/dashboard"
agent-browser state save auth.json

# 后续复用
agent-browser state load auth.json
agent-browser open https://app.example.com/dashboard
```

### 自动持久化（session-name）

```bash
# 自动保存/恢复 cookies 与 localStorage（跨浏览器重启）
agent-browser --session-name myapp open https://app.example.com/login
# ... login flow ...
agent-browser close  # state 自动保存到 ~/.agent-browser/sessions/

# 下次自动加载
agent-browser --session-name myapp open https://app.example.com/dashboard

# 落盘加密（AES-256-GCM）
export AGENT_BROWSER_ENCRYPTION_KEY=$(openssl rand -hex 32)
agent-browser --session-name secure open https://app.example.com

# 管理已保存的 state
agent-browser state list
agent-browser state show myapp-default.json
agent-browser state clear myapp
agent-browser state clean --older-than 7
```

### 数据抽取

```bash
agent-browser open https://example.com/products
agent-browser snapshot -i
agent-browser get text @e5              # 取某个元素文本
agent-browser get text body > page.txt  # 取整页文本（重定向到文件）

# JSON 输出（便于程序解析）
agent-browser snapshot -i --json
agent-browser get text @e1 --json
```

### 并行会话（多 Agent 隔离）

```bash
agent-browser --session site1 open https://site-a.com
agent-browser --session site2 open https://site-b.com

agent-browser --session site1 snapshot -i
agent-browser --session site2 snapshot -i

agent-browser session list
```

### 连接现有 Chrome（CDP）

```bash
# 自动发现本机 Chrome 的 remote debugging 并连接
agent-browser --auto-connect open https://example.com
agent-browser --auto-connect snapshot

# 或者显式指定端口
agent-browser --cdp 9222 snapshot
```

### 可视化与调试

```bash
agent-browser --headed open https://example.com
agent-browser highlight @e1            # 高亮元素
agent-browser record start demo.webm   # 录屏（WebM）
agent-browser profiler start           # 开始性能采样
agent-browser profiler stop trace.json # 停止并保存（path 可选）
```

### 本地文件（PDF/HTML）

```bash
# 使用 file:// 打开本地文件
agent-browser --allow-file-access open file:///path/to/document.pdf
agent-browser --allow-file-access open file:///path/to/page.html
agent-browser screenshot output.png
```

### iOS Simulator（Mobile Safari）

```bash
# 列出可用的 iOS 模拟器
agent-browser device list

# 在指定设备上启动 Safari
agent-browser -p ios --device "iPhone 16 Pro" open https://example.com

# 与桌面端同样流程：snapshot → 交互 → 重新 snapshot
agent-browser -p ios snapshot -i
agent-browser -p ios tap @e1          # 点击（tap 是 click 的别名）
agent-browser -p ios fill @e2 "text"
agent-browser -p ios swipe up         # 移动端手势

# 截图
agent-browser -p ios screenshot mobile.png

# 关闭（会关闭模拟器）
agent-browser -p ios close
```

**要求：**macOS + Xcode；并安装 Appium：`npm install -g appium && appium driver install xcuitest`

**真机：**如果已完成 WebDriverAgent 等配置，也可用真机。`--device "<UDID>"`，UDID 可从 `xcrun xctrace list devices` 获取。

## Grokipedia（grokipedia.com）实战要点

在 Grokipedia 上常见的三个“坑点”，对应三个解决套路：

### 1) 搜索结果很多不是 link/button：用 `snapshot -i -C`

很多条目列表是自定义可点击元素（`div` 等），A11y role 不一定是 `link/button`。用 `-C` 才能把 `cursor:pointer` / `onclick` 的条目抓出来并生成 `clickable "..."`

```bash
agent-browser --session grok open https://grokipedia.com
agent-browser --session grok snapshot -i

# 输入关键词
agent-browser --session grok fill @e3 "OpenAI"

# 关键：带 -C 才能看到可点击的条目 refs
agent-browser --session grok snapshot -i -C
# 输出包含：
#   - clickable "OpenAI" [ref=e8] [cursor:pointer]

agent-browser --session grok click @e8
agent-browser --session grok wait --url "**/page/**"
agent-browser --session grok get url
```

### 2) Command Menu（⌘K）：用 `press "Meta+k"` + 填 combobox

```bash
# macOS: ⌘K
agent-browser press "Meta+k"
# Windows/Linux: 可能是 Ctrl+K
# agent-browser press "Control+k"

agent-browser wait 200
agent-browser snapshot -c -d 2
# 会出现：
# - combobox "Command Menu" [ref=...]

agent-browser fill @e8 "ChatGPT"
agent-browser press Enter
agent-browser wait --url "**/search**"
```

如果快捷键不生效，直接点击页面上的 `Search ⌘ K` 按钮再走同样流程。

### 3) Edits History 侧栏：用 selector scope + 在侧栏内滚动加载更多

侧栏内容很长，且会在滚动时加载更多编辑记录。建议只对侧栏做 `snapshot`，避免全页输出巨大。

```bash
# 打开侧栏（先 snapshot 拿到 Edits History 的 ref）
agent-browser snapshot -c -d 2
agent-browser click @e5
agent-browser wait 500

# 只抓侧栏（减少输出 + 避免重复元素的 strict-mode 误匹配）
agent-browser snapshot -c -d 5 -s 'aside:not(.hidden):has(button[aria-label="Close edits sidebar"])'
```

如果需要“加载更多 edits”，要滚动侧栏内部的滚动容器（不是页面本身）。可以用 `eval --stdin` 找到可滚动容器并拉到底：

```bash
agent-browser eval --stdin <<'EVALEOF'
(() => {
  const aside = document.querySelector(
    'aside:not(.hidden):has(button[aria-label="Close edits sidebar"])'
  );
  if (!aside) return { error: "no aside" };
  const els = [aside, ...Array.from(aside.querySelectorAll("*"))];
  const scroller = els.find((el) => {
    const s = getComputedStyle(el);
    return (s.overflowY === "auto" || s.overflowY === "scroll") &&
      el.scrollHeight > el.clientHeight + 10;
  });
  if (!scroller) return { error: "no scroller" };
  scroller.scrollTop = scroller.scrollHeight;
  return { scrollTop: scroller.scrollTop, scrollHeight: scroller.scrollHeight };
})();
EVALEOF
agent-browser wait 800
```

关闭侧栏推荐用：`agent-browser press Escape`（“Close edits sidebar” 在 DOM 中可能有重复元素，直接点有时会触发 strict-mode 冲突）。

### 4) 结构化抽取词条信息框（`dt`/`dd` → JSON）

Grokipedia 词条页通常有信息框（例如 Type、Founded、Key People 等）。可以用 `eval` 把 `dt`/`dd` 抽成 JSON 结构，方便下游程序处理：

```bash
agent-browser eval --stdin <<'EVALEOF'
(() => {
  const out = {};
  for (const dt of document.querySelectorAll("dt")) {
    const key = (dt.innerText || dt.textContent || "").trim();
    const dd = dt.nextElementSibling;
    if (!key || !dd || dd.tagName.toLowerCase() !== "dd") continue;
    out[key] = (dd.innerText || dd.textContent || "")
      .replace(/\s+/g, " ")
      .trim();
  }
  return out;
})();
EVALEOF
```

## 超时、慢页面与 SPA 跳转

本地浏览器默认 Playwright 超时为 60 秒。慢站点/大页面建议用显式等待：

```bash
# 等网络空闲（慢页面优先）
agent-browser wait --load networkidle

# 等某个元素出现
agent-browser wait "#content"
agent-browser wait @e1

# 等 URL 变化（SPA/重定向很实用）
agent-browser wait --url "**/dashboard"

# 等 JS 条件
agent-browser wait --fn "document.readyState === 'complete'"

# 最后手段：固定等待毫秒
agent-browser wait 5000
```

经验法则：`open` 之后先 `wait --load networkidle` 再 `snapshot`；如果是 SPA，则 `wait --url` 往往更可靠。

## 会话管理与清理

多 Agent 并发时务必用命名 session 做隔离：

```bash
# 每个 Agent 一个 session
agent-browser --session agent1 open site-a.com
agent-browser --session agent2 open site-b.com

# 查看活跃会话
agent-browser session list
```

任务结束务必关闭，避免残留进程：

```bash
agent-browser close                    # Close default session
agent-browser --session agent1 close   # Close specific session
```

如果出现 daemon 忙/卡住（例如报错包含 `os error 35`），优先尝试：

1. 换一个 `--session <name>` 重新开始
2. 对卡住的 session 执行 `agent-browser --session <name> close`
3. 最后执行 `agent-browser close` 清理默认 session

另外：启动参数（如 `--executable-path`、`--proxy`、`--headed`）只在 daemon 启动时生效；如果提示参数被忽略，请先 `close` 再重新 `open`。

## Ref 生命周期（重要）

Refs（`@e1`、`@e2`）在页面变化后会失效。以下情况必须重新 `snapshot`：

- 点击导致跳转的链接/按钮
- 表单提交
- 动态加载（下拉框、弹窗、侧栏内容变化等）

```bash
agent-browser click @e5              # 触发跳转/更新
agent-browser snapshot -i            # 必须重新 snapshot
agent-browser click @e1              # 使用新的 refs
```

## 带标注的截图（Vision Mode）

使用 `--annotate` 在截图上叠加带数字的标签。每个标签 `[N]` 对应 ref `@eN`。这同时也会缓存 refs，无需额外 snapshot 即可直接交互。

```bash
agent-browser screenshot --annotate
# 输出包含图片路径和图例：
#   [1] @e1 button "Submit"
#   [2] @e2 link "Home"
#   [3] @e3 textbox "Email"
agent-browser click @e2              # 使用截图中的 ref 进行点击
```

适用场景：
- 页面有未标注的图标按钮或纯视觉元素
- 需要验证视觉布局或样式
- 页面包含 Canvas 或图表（文字 snapshot 不可见）
- 需要对元素位置进行空间推理

## 语义定位（refs 不好用时的替代方案）

当 refs 不可用或不稳定时，用 `find` 系列：

```bash
agent-browser find text "Sign In" click
agent-browser find label "Email" fill "user@test.com"
agent-browser find role button click --name "Submit"
agent-browser find placeholder "Search" type "query"
agent-browser find testid "submit-btn" click
```

## 执行 JavaScript（eval）

`eval` 在浏览器上下文执行 JS。注意：复杂表达式容易被 shell 转义破坏，推荐 `--stdin` 或 `-b`。

```bash
# 简单表达式
agent-browser eval 'document.title'
agent-browser eval 'document.querySelectorAll("img").length'

# 复杂 JS：推荐 --stdin
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(
  Array.from(document.querySelectorAll("img"))
    .filter(i => !i.alt)
    .map(i => ({ src: i.src.split("/").pop(), width: i.width }))
)
EVALEOF

# 另一种：base64（彻底避免 shell 转义问题）
agent-browser eval -b "$(echo -n 'Array.from(document.querySelectorAll("a")).map(a => a.href)' | base64)"
```

为什么要这样：shell 会处理双引号、`!`（history expansion）、反引号、`$()` 等，可能在 JS 到达 agent-browser 之前就被改写。`--stdin`/`-b` 能绕过这些问题。

经验法则：
- 单行、无嵌套引号：`eval 'expr'`
- 有嵌套引号/箭头函数/模板字符串/多行：`eval --stdin <<'EVALEOF'`
- 程序生成脚本：`eval -b "$(echo -n ... | base64)"`

## 配置文件

在项目根目录创建 `agent-browser.json` 作为持久化默认值：

```json
{
  "headed": true,
  "proxy": "http://localhost:8080",
  "profile": "./browser-data"
}
```

优先级（低到高）：`~/.agent-browser/config.json` < `./agent-browser.json` < 环境变量 < CLI flags。可用 `--config <path>` 或 `AGENT_BROWSER_CONFIG` 指定配置文件（文件缺失/无效会报错）。CLI 参数在配置里用 camelCase（例如 `--executable-path` -> `"executablePath"`）。布尔参数支持显式 `true/false`（例如 `--headed false` 覆盖配置）。extensions 会做合并（拼接），不是覆盖。

## 深入参考文档

| 参考 | 适用场景 |
|-----------|-------------|
| [references/commands.md](references/commands.md) | 全量命令与参数说明 |
| [references/snapshot-refs.md](references/snapshot-refs.md) | ref 生命周期、失效规则、排障技巧 |
| [references/session-management.md](references/session-management.md) | 并行 session、状态持久化、并发自动化 |
| [references/authentication.md](references/authentication.md) | 登录/OAuth/2FA、复用 state 的套路 |
| [references/video-recording.md](references/video-recording.md) | 录屏的推荐工作流（调试与文档） |
| [references/profiling.md](references/profiling.md) | 性能分析（Chrome DevTools profiling） |
| [references/proxy-support.md](references/proxy-support.md) | 代理配置、地域测试、轮换代理 |

## 可直接复用的模板

| 模板 | 说明 |
|----------|-------------|
| [templates/form-automation.sh](templates/form-automation.sh) | 表单填写 + 基础校验 |
| [templates/authenticated-session.sh](templates/authenticated-session.sh) | 登录一次，复用状态 |
| [templates/capture-workflow.sh](templates/capture-workflow.sh) | 抽取内容 + 截图产物 |

```bash
./templates/form-automation.sh https://example.com/form
./templates/authenticated-session.sh https://app.example.com/login
./templates/capture-workflow.sh https://example.com ./output
```
