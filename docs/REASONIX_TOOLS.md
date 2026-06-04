# Reasonix 工具集文档

> 🏷️ **AI 标识**
> - **由谁生成**：Reasonix Agent（编码智能体框架）
> - **底层模型**：通过 OpenAI 兼容 API 调用（provider 配置）
> - **目标项目**：Reasonix（DeepSeek-Reasonix）
> - **目标版本**：1.0.0（Go 重写版，`main` 分支，未正式发布）
> - **生成时间**：2025-06-02
> - **生成方式**：直接从源码 `internal/` 目录提取，遍历每个 `tool.RegisterBuiltin()` 注册点及各工具实现文件

> 本文档枚举 Reasonix 项目中所有暴露给 AI 模型（agent）调用的工具。
> **重点**：本地工具集（17 个 Built-in Tools）逐条详细说明；其他工具简略列出。

---

## 目录

1. [本地工具集（核心）—— 17 个](#一本地工具集核心--17-个)
2. [其他工具（概览）](#二其他工具概览)
3. [注册流程总览](#三注册流程总览)
4. [关键结论](#四关键结论)

---

## 一、本地工具集（核心）—— 17 个

> **定义**：通过 `tool.RegisterBuiltin()` 编译进 Reasonix 二进制的内置工具，不依赖外部进程或 agent 框架上下文。
> 全部在 `internal/tool/builtin/` 下，纯 Go 代码实现。
>
> **实现形式标记说明**：
> - **🔧 Go编译代码** —— 工具逻辑编译进 Reasonix 二进制，不依赖外部文件或进程
> - **📡 调远程接口** —— 工具内部发 HTTP/RPC 请求到外部服务
> - **🖥️ 调系统进程** —— 工具通过 `os/exec` 启动外部子进程

### 实现框架

每个工具实现 `tool.Tool` 接口：

```go
type Tool interface {
    Name() string
    Description() string
    Schema() json.RawMessage       // JSON Schema 参数定义
    Execute(ctx, args) (string, error)
    ReadOnly() bool                 // 是否无副作用的只读操作
}
```

通过 `init()` → `tool.RegisterBuiltin()` 自注册，`internal/boot/boot.go:457-479` 中统一加载。
`ReadOnly=true` 的工具 agent 可**并行调度**；`ReadOnly=false` 的工具按序执行。

---

### 1.1 文件读写类（7 个）

---
#### `read_file`

| 属性 | 值 |
|------|-----|
| **功能** | 读取文本文件，支持 `offset`/`limit` 分页。自动检测二进制（NUL 字节阻断），支持 UTF-8/UTF-16 BOM 解码。输出带行号前缀，尾部显示剩余行数 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/readfile.go:20`（`init` 注册），`readFile` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ✅ 可以 — 纯 `os.Open` + `bufio.Scanner` |
| **参数** | `path`（必填），`offset`（可选，默认 0），`limit`（可选，默认 2000） |
| **关键实现** | 扫描前 8KB 检测 NUL 字节判断二进制；BOM 检测支持 UTF-8/UTF-16LE/UTF-16BE 自动解码 |

---

#### `write_file`

| 属性 | 值 |
|------|-----|
| **功能** | 写文件（覆盖已有内容），自动创建父目录 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/writefile.go:13`（`init` 注册），`writeFile` struct |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ✅ 可以 — `os.WriteFile` + `os.MkdirAll` 封装 |
| **参数** | `path`（必填），`content`（必填） |
| **安全限制** | 通过 `ConfineWriters`（`confine.go:25`）限定只能写到工作区目录 |

---

#### `edit_file`

| 属性 | 值 |
|------|-----|
| **功能** | 精确字符串替换编辑文件。要求 `old_string` **在文件中唯一**，否则报错。比全量重写更安全 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/editfile.go:13`（`init` 注册），`editFile` struct |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ✅ 可以 |
| **参数** | `path`（必填），`old_string`（必填），`new_string`（必填） |
| **关键实现** | `os.ReadFile` → `strings.Count` 检查唯一性 → `strings.Replace` 替换 → `os.WriteFile` 写回 |

---

#### `multi_edit`

| 属性 | 值 |
|------|-----|
| **功能** | 对一个文件进行批量编辑（多个 `{old_string, new_string}`），**原子化执行**——中间任一步失败文件不写回，避免半修改状态。支持 `replace_all` 批量替换 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/multiedit.go:13`（`init` 注册），`multiEdit` struct |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ✅ 可以 |
| **参数** | `path`（必填），`edits`（数组，每个含 `old_string` + `new_string` + 可选 `replace_all`） |

---

#### `delete_range`

| 属性 | 值 |
|------|-----|
| **功能** | 用起始/结束行锚点删除文件中的连续文本范围，返回 unified diff。锚点文本必须在各自行上唯一 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/delete_range.go:14`（`init` 注册），`deleteRange` struct |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ✅ 可以 |
| **参数** | `path`（必填），`start_anchor`（必填），`end_anchor`（必填），`inclusive`（可选，默认 true） |
| **关键实现** | 先在内存中计算 diff（`preview` 方法），校验通过后才 `os.WriteFile`；也实现了 `Previewer` 接口 |

---

#### `delete_symbol`

| 属性 | 值 |
|------|-----|
| **功能** | 使用 Go AST 解析删除 Go 源文件中的命名符号（函数、方法、类型、接口、常量、变量） |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/delete_symbol.go:18`（`init` 注册），`deleteSymbol` struct |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ⚠️ **仅限 `.go` 文件**，非 Go 文件请用 `delete_range` |
| **参数** | `path`（必填），`name`（必填），`kind`（可选：func/method/type/interface/const/var），`parent`（可选：方法消歧） |
| **关键实现** | `go/parser` 解析 → `go/ast` 遍历定位符号 → 计算行范围 → 删除并保留文档注释 |

---

#### `notebook_edit`

| 属性 | 值 |
|------|-----|
| **功能** | 编辑 Jupyter `.ipynb` 笔记本单元格。支持 `replace` / `insert` / `delete` 三种模式，保持 JSON 合法性 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/notebookedit.go:14`（`init` 注册），`notebookEdit` struct |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ✅ 可以 |
| **参数** | `path`（必填），`cell_number` / `cell_id`（目标），`new_source`，`cell_type`（code/markdown），`edit_mode`（replace/insert/delete） |
| **关键实现** | `json.RawMessage` 收尾保留未识别的元数据；`json.RawMessage` 重新序列化保证输出合法 |

---

### 1.2 搜索查询类（3 个）

---
#### `grep`

| 属性 | 值 |
|------|-----|
| **功能** | 在文件或目录中递归搜索正则表达式（RE2 语法）。结果格式 `path:line:text`，上限 200 条。自动跳过 `.git`、`node_modules`、`vendor` 等目录 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/grep.go:19`（`init` 注册），`grepTool` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ✅ 可以 — `regexp` + `filepath.WalkDir` 实现 |
| **参数** | `pattern`（必填），`path`（可选，默认 `.`） |
| **关键实现** | `filepath.WalkDir` 遍历目录 → `bufio.Scanner` 逐行匹配 → 达到 200 条返回 `io.EOF` 终止遍历 |

---

#### `glob`

| 属性 | 值 |
|------|-----|
| **功能** | 按通配符模式查找文件，支持 `*`、`?`、`[]`、`**`（递归）。当简单文件名在当前目录匹配不到时，自动进行递归搜索 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/glob.go:15`（`init` 注册），`globTool` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ✅ 可以 — `filepath.Glob` + `filepath.WalkDir` |
| **参数** | `pattern`（必填） |
| **关键实现** | 不含 `**` 先用 `filepath.Glob`；无匹配且文件名无路径分隔符时自动回退到 `**/<name>` 递归搜索 |

---

#### `ls`

| 属性 | 值 |
|------|-----|
| **功能** | 列出目录内容。目录带 `/` 后缀，文件显示字节大小 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，无外部依赖） |
| **源码** | `internal/tool/builtin/ls.go:13`（`init` 注册），`listDir` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ✅ 可以 — `os.ReadDir` 封装 |
| **参数** | `path`（可选，默认 `.`） |

---

### 1.3 命令/网络执行类（5 个）

---
#### `bash`

| 属性 | 值 |
|------|-----|
| **功能** | 执行 shell 命令。支持超时（120s）和后台运行模式。自动检测 Windows PowerShell 并适配（阻止 `&&`/`||`，提示用 `;` 或 `if ($?)`）。支持 OS 沙箱隔离 |
| **实现形式** | 🔧 Go编译代码 + 🖥️ 调系统进程（通过 `os/exec` 启动系统 shell） |
| **源码** | `internal/tool/builtin/bash.go:20`（`init` 注册），`bash` struct |
| **ReadOnly** | ❌ 否 — 副作用无法静态推断 |
| **可独立运行** | ✅ 可以 — `os/exec` 封装 |
| **参数** | `command`（必填），`run_in_background`（可选，布尔值） |
| **关键实现** | 前台模式：`exec.CommandContext` + 120s timeout；后台模式：通过 `jobs.Manager.Start()` 注册 session 级后台任务。沙箱通过 `sandbox.Command()` 封装 |

---

#### `bash_output`

| 属性 | 值 |
|------|-----|
| **功能** | 读取后台 bash 任务的新输出（非阻塞）。可选正则过滤 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，依赖 jobs.Manager 运行时） |
| **源码** | `internal/tool/builtin/bgjobs.go:21-24`（三个工具同一文件注册） |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ⚠️ 依赖 `jobs.Manager`（session 级别） |

---

#### `kill_shell`

| 属性 | 值 |
|------|-----|
| **功能** | 终止运行中的后台任务 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，依赖 jobs.Manager 运行时） |
| **源码** | `internal/tool/builtin/bgjobs.go:21-24` |
| **ReadOnly** | ❌ 否 |
| **可独立运行** | ⚠️ 依赖 `jobs.Manager` |

---

#### `wait`

| 属性 | 值 |
|------|-----|
| **功能** | 阻塞等待后台任务完成，返回结果。可指定等待特定 job 或全部 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，依赖 jobs.Manager 运行时） |
| **源码** | `internal/tool/builtin/bgjobs.go:21-24` |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ⚠️ 依赖 `jobs.Manager` |
| **参数** | `job_ids`（可选，数组），`timeout_seconds`（可选） |

---

#### `web_fetch`

| 属性 | 值 |
|------|-----|
| **功能** | HTTP(S) 获取 URL 内容。HTML 自动提取纯文本（去 script/style/标签）。**内建 SSRF 防护** |
| **实现形式** | 🔧 Go编译代码 + 📡 调远程接口（通过 `net/http` 发起 HTTP 请求） |
| **源码** | `internal/tool/builtin/webfetch.go:18`（`init` 注册），`webFetch` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ✅ 可以 — 独立 HTTP 客户端 |
| **参数** | `url`（必填，http/https 绝对地址） |
| **关键实现** | 自定义 `net.DialContext` 在 IP 层检查：拒绝私有 IP（RFC1918）、链路本地（169.254.x.x）、CGNAT（100.64.0.0/10）、未指定地址（0.0.0.0），并固定解析后的 IP 防止 DNS rebinding。超时 15s，最大 1 MiB |

---

### 1.4 Agent 内务类（2 个）

---
#### `todo_write`

| 属性 | 值 |
|------|-----|
| **功能** | 记录和更新当前任务的规划列表。支持两层级（phase + sub-step），维护一个 `in_progress` 状态。**无文件副作用** |
| **实现形式** | 🔧 Go编译代码（编译进二进制，依赖 agent 运行时上下文） |
| **源码** | `internal/tool/builtin/todo.go:12`（`init` 注册），`todoWrite` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ⚠️ 依赖 agent 上下文校验 |
| **参数** | `todos` 数组（每个含 `content`、`status`、`activeForm`、`level`） |
| **关键实现** | 不读写文件，只校验 JSON 格式和状态转换合法性。配合 `complete_step` 使用 |

---

#### `complete_step`

| 属性 | 值 |
|------|-----|
| **功能** | 记录带证据的步骤完成签收——证明某一步已经完成（运行了什么命令、改了哪些文件、做了什么手动检查）。没有证据的完成会被拒绝 |
| **实现形式** | 🔧 Go编译代码（编译进二进制，依赖 agent 运行时上下文） |
| **源码** | `internal/tool/builtin/completestep.go:13`（`init` 注册），`completeStep` struct |
| **ReadOnly** | ✅ 是 |
| **可独立运行** | ⚠️ 依赖 agent 上下文校验 |
| **参数** | `step`、`result`、`evidence`（含 `kind` = verification/diff/files/manual）、`notes` |
| **关键实现** | 调取 `evidence.FromContext` 对比哪些步骤是"新完成但无对应 complete_step 调用"的，拒绝不合规的完成声明 |

---

### 本地工具集一览表

| # | 工具名 | 类别 | 实现形式 | ReadOnly | 可独立运行 | 核心实现方式 | 源码文件 |
|---|--------|------|---------|:--------:|:--------:|------------|---------|
| 1 | `read_file` | 文件读写 | 🔧 Go编译代码 | ✅ | ✅ | `os.Open` + `bufio.Scanner` | `internal/tool/builtin/readfile.go` |
| 2 | `write_file` | 文件读写 | 🔧 Go编译代码 | ❌ | ✅ | `os.WriteFile` + `os.MkdirAll` | `internal/tool/builtin/writefile.go` |
| 3 | `edit_file` | 文件读写 | 🔧 Go编译代码 | ❌ | ✅ | `strings.Replace` 唯一匹配 | `internal/tool/builtin/editfile.go` |
| 4 | `multi_edit` | 文件读写 | 🔧 Go编译代码 | ❌ | ✅ | 内存中批量操作 + 原子写回 | `internal/tool/builtin/multiedit.go` |
| 5 | `delete_range` | 文件读写 | 🔧 Go编译代码 | ❌ | ✅ | 锚点行定位 + 裁剪 + diff | `internal/tool/builtin/delete_range.go` |
| 6 | `delete_symbol` | 文件读写 | 🔧 Go编译代码 | ❌ | ⚠️ 仅 Go | `go/ast` + `go/parser` | `internal/tool/builtin/delete_symbol.go` |
| 7 | `notebook_edit` | 文件读写 | 🔧 Go编译代码 | ❌ | ✅ | `json.RawMessage` 收尾操作 | `internal/tool/builtin/notebookedit.go` |
| 8 | `grep` | 搜索查询 | 🔧 Go编译代码 | ✅ | ✅ | `regexp` + `filepath.WalkDir` | `internal/tool/builtin/grep.go` |
| 9 | `glob` | 搜索查询 | 🔧 Go编译代码 | ✅ | ✅ | `filepath.Glob` + `filepath.WalkDir` | `internal/tool/builtin/glob.go` |
| 10 | `ls` | 搜索查询 | 🔧 Go编译代码 | ✅ | ✅ | `os.ReadDir` | `internal/tool/builtin/ls.go` |
| 11 | `bash` | 命令/网络 | 🔧 Go代码 + 🖥️ 调系统进程 | ❌ | ✅ | `os/exec` + `sandbox` | `internal/tool/builtin/bash.go` |
| 12 | `bash_output` | 命令/网络 | 🔧 Go编译代码 | ✅ | ⚠️ | `jobs.Manager.Output()` | `internal/tool/builtin/bgjobs.go` |
| 13 | `kill_shell` | 命令/网络 | 🔧 Go编译代码 | ❌ | ⚠️ | `jobs.Manager.Kill()` | `internal/tool/builtin/bgjobs.go` |
| 14 | `wait` | 命令/网络 | 🔧 Go编译代码 | ✅ | ⚠️ | `jobs.Manager.Wait()` | `internal/tool/builtin/bgjobs.go` |
| 15 | `web_fetch` | 命令/网络 | 🔧 Go代码 + 📡 调远程接口 | ✅ | ✅ | `net/http` + SSRF 防护 | `internal/tool/builtin/webfetch.go` |
| 16 | `todo_write` | agent 内务 | 🔧 Go编译代码 | ✅ | ⚠️ | 纯参数校验，无文件操作 | `internal/tool/builtin/todo.go` |
| 17 | `complete_step` | agent 内务 | 🔧 Go编译代码 | ✅ | ⚠️ | evidence 一致性校验 | `internal/tool/builtin/completestep.go` |

---

## 二、其他工具（概览）

以下工具虽然也注册在 `tool.Registry` 中，但不属于"本地工具集"（依赖外部进程或 agent 框架），此处仅简要列出。

### 2.1 LSP 工具（4 个）

通过 `internal/lsp/tool.go` 的 `Tools(*Manager)` 构造，连接本机安装的语言服务器（`gopls`、`rust-analyzer` 等）。

| 工具名 | 功能 | 实现形式 | ReadOnly |
|--------|------|---------|:--------:|
| `lsp_definition` | 跳转到符号定义位置 | 🔧 Go代码 + 🖥️ 调系统进程（JSON-RPC 连接外部 LSP server） | ✅ |
| `lsp_references` | 列出工作区中符号的所有引用 | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |
| `lsp_hover` | 显示符号类型签名和文档 | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |
| `lsp_diagnostics` | 报告文件编译/语法错误 | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |

**实现方式**：Go 语言作为 JSON-RPC 2.0 客户端，连接外部 LSP server 子进程（`gopls`、`rust-analyzer` 等）。

### 2.2 元工具（6 个）

| 工具名 | 功能 | 实现形式 | ReadOnly |
|--------|------|---------|:--------:|
| `task` | 生成子 agent 执行独立子任务 | 🔧 Go编译代码（依赖 agent 框架调度） | ❌ |
| `ask` | 向用户提多项选择题 | 🔧 Go编译代码（依赖前端交互通道） | ✅ |
| `remember` | 持久化保存事实到项目记忆 | 🔧 Go编译代码（读写内存 Markdown 文件） | ✅ |
| `forget` | 删除已保存的记忆 | 🔧 Go编译代码（删除内存文件） | ✅ |
| `run_skill` | 按名称调用已安装技能 | 🔧 Go编译代码（依赖 skill store + agent） | ❌ |
| `install_skill` | 编写并保存新技能 | 🔧 Go编译代码（写文件到 `.reasonix/skills/`） | ❌ |

### 2.3 子 Agent 包装工具（4 个）

这些是 `run_skill` 的特化版本，将内置 4 个技能包装为独立工具：

| 工具名 | 对应的技能名 | 功能 | 实现形式 | ReadOnly |
|--------|------------|------|---------|:--------:|
| `explore` | `explore` | 隔离子 agent 只读探索代码库 | 🔧 Go编译代码（子 agent 调度壳） | ✅ |
| `research` | `research` | 结合代码 + web_fetch 的综合调研 | 🔧 Go编译代码（子 agent 调度壳） | ✅ |
| `review` | `review` | 审查当前 git 分支的代码变更 | 🔧 Go编译代码（子 agent 调度壳） | ✅ |
| `security_review` | `security-review` | 安全视角代码审查 | 🔧 Go编译代码（子 agent 调度壳） | ✅ |

**源码**：`internal/skill/tools.go:140-172`（`BuiltinSubagentTools` 函数）
**技能 body**：`internal/skill/builtins.go`

### 2.4 MCP / CodeGraph 工具（N 个）

**CodeGraph** 是基于 tree-sitter + SQLite 的代码索引引擎，作为 MCP stdio 插件自动注入。

| 工具名 | 功能 | 实现形式 | ReadOnly |
|--------|------|---------|:--------:|
| `codegraph_context` | 获取代码上下文（入口点 + 相关符号） | 🔧 Go代码 + 🖥️ 调系统进程（MCP stdio 连接独立 Node.js 进程） | ✅ |
| `codegraph_explore` | 按符号/文件名批量探索，返回源码 | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |
| `codegraph_node` | 符号的详细信息（位置、签名、调用链） | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |
| `codegraph_search` | 按名称快速搜索符号 | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |
| `codegraph_trace` | 两个符号之间的调用路径追踪 | 🔧 Go代码 + 🖥️ 调系统进程 | ✅ |

**实现方式**：Reasonix 作为 JSON-RPC 2.0 客户端（MCP），通过 stdio 连接独立 Node.js 二进制 `codegraph serve --mcp`。
首次使用时下载（~45MB），缓存在版本目录下。

**用户自定义 MCP 插件**：可在 `reasonix.toml` 配置任意 MCP 服务器（stdio/http/sse），自动注册为 `mcp__<server>__<tool>` 名称。

### 2.5 Slash 命令工具（1 个）

| 工具名 | 功能 | ReadOnly |
|--------|------|:--------:|
| 工具名 | 功能 | 实现形式 | ReadOnly |
|--------|------|---------|:--------:|
| `slash_command` | 按名称调用项目 slash 命令或技能 | 🔧 Go编译代码（索引 `.reasonix/commands/*.md` 模板） | ✅ |

**源码**：`internal/command/slashtool.go:30`
**实现方式**：将 `.reasonix/commands/` 下的命令文件和已安装技能统一为索引，调用时返回展开后的 prompt 文本。

---

## 三、注册流程总览

所有工具在 `internal/boot/boot.go:124-325` 中组装：

```
1. tool.NewRegistry()                    ← 创建空注册表 (boot.go:124)
2. addBuiltins(reg, ...)                 ← 注册 17 个本地工具 (boot.go:132 → 457)
   ├─ 遍历 tool.Builtins() 逐个 reg.Add(t)
   └─ ConfineWriters(writeRoots)         ← 替换写工具为工作区限定版 (boot.go:474)
   └─ ConfineBash(bashSpec)              ← 替换 bash 为沙箱版 (boot.go:474)
3. plugin.StartAvailable(specs)          ← 启动 MCP 插件（含 CodeGraph）(boot.go:176)
4. lsp.Tools(lspMgr)                     ← 注册 4 个 LSP 工具 (boot.go:193)
5-8. 元工具注册                          ← task / remember / forget / ask (行 235-248)
9-10. 技能工具                           ← run_skill / install_skill (行 281-282)
11. skill.BuiltinSubagentTools(...)      ← explore/research/review/security_review (行 283)
12. reg.Add(NewSlashCommandTool(...))    ← slash_command (行 325)
```

**最终注册工具数**：约 **37+** 个（不含用户自定义 MCP 插件）

工具组成：

```
                    ┌────────────────────────────────────┐
                    │        工具注册表 (37+ 个)          │
                    ├────────────────────────────────────┤
                    │  [本地工具集] 17 个                  │
                    │  [LSP 工具]    4 个                  │
                    │  [元工具]      6 个                  │
                    │  [子 Agent 工具] 4 个                │
                    │  [MCP 插件]    N 个 (含 CodeGraph 5) │
                    │  [Slash 命令]  1 个                  │
                    └────────────────────────────────────┘
```

---

## 四、关键结论

### 1. 本地工具集的实现形式

所有 17 个本地工具都是 **🔧 Go编译代码**（编译进 Reasonix 二进制），但根据与外部世界的交互方式可分为三档：

| 类型 | 数量 | 工具 | 说明 |
|------|:----:|------|------|
| 🔧 纯 Go 编译代码 | **14** | `read_file`、`write_file`、`edit_file`、`multi_edit`、`delete_range`、`delete_symbol`、`notebook_edit`、`grep`、`glob`、`ls`、`bash_output`、`kill_shell`、`wait`、`todo_write`、`complete_step` | 仅使用 Go 标准库，不调用外部进程或网络 |
| 🔧 + 🖥️ 调系统进程 | **1** | `bash` | 通过 `os/exec` 启动系统 shell 执行命令 |
| 🔧 + 📡 调远程接口 | **1** | `web_fetch` | 通过 `net/http` 发起 HTTP 请求 |

> 注意：没有工具是用脚本（比如 shell 脚本、Python 脚本）实现的，全部是编译进二进制的 Go 代码。

### 2. 独立运行能力
- **纯 Go 代码**：17 个工具全部是 Go 标准库封装，无外部脚本依赖
- **编译进二进制**：通过 `init()` 自注册，不依赖文件系统上的外部文件
- **13 个可独立运行**：`read_file`、`write_file`、`edit_file`、`multi_edit`、`delete_range`、`notebook_edit`、`grep`、`glob`、`ls`、`bash`、`web_fetch` 可以脱离 Reasonix 框架单独使用
- **4 个不能独立运行**：`bash_output`、`kill_shell`、`wait`（依赖 jobs manager）、`todo_write`、`complete_step`（依赖 agent context）

### 2. ReadOnly 决定调度方式
✅ **只读（可并行）**：`read_file`、`grep`、`glob`、`ls`、`web_fetch`、`todo_write`、`complete_step`、`bash_output`、`wait`
❌ **非只读（串行）**：`write_file`、`edit_file`、`multi_edit`、`delete_range`、`delete_symbol`、`notebook_edit`、`bash`、`kill_shell`

### 3. 安全边界
- **bash 沙箱**：可通过 `sandbox.Spec` 配置 OS 级沙箱（`bash.go` 第 84 行）
- **写限定**：所有写操作通过 `ConfineWriters` 限定到工作区（`confine.go:25`）
- **SSRF 防护**：`web_fetch` 拒绝私有 IP、链路本地、CGNAT 地址，防 DNS rebinding（`webfetch.go:51-98`）
- **权限门**：每个工具调用经过 `permission.Gate` 审核（`boot.go:211`）

### 4. 其他工具的关键区别
- **LSP 工具**：依赖本机安装的语言服务器进程
- **元工具**：依赖 agent 框架上下文
- **MCP 插件**：通过 MCP 协议连接外部进程（CodeGraph 等）
- **Slash 命令**：指向 `.reasonix/commands/*.md` 模板的索引器

---

> **生成时间**：从 Reasonix 主分支源码提取
> **核心源码目录**：
> - 本地工具：`internal/tool/builtin/`
> - 工具接口：`internal/tool/tool.go`
> - LSP 工具：`internal/lsp/tool.go` + `internal/lsp/manager.go`
> - 元工具：`internal/agent/` + `internal/memory/` + `internal/skill/`
> - MCP 插件：`internal/plugin/`
> - CodeGraph：`internal/codegraph/`
> - 启动装配：`internal/boot/boot.go`
