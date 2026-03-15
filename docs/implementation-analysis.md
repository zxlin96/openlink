# OpenLink 功能实现原理分析

本文档详细分析 OpenLink 各核心模块的实现原理，帮助开发者深入理解系统架构和代码设计。

---

## 目录

1. [整体架构](#整体架构)
2. [Chrome 扩展实现原理](#chrome-扩展实现原理)
3. [Go 服务端实现原理](#go-服务端实现原理)
4. [安全沙箱机制](#安全沙箱机制)
5. [工具系统设计](#工具系统设计)
6. [Skills 扩展系统](#skills-扩展系统)
7. [完整调用链路](#完整调用链路)
8. [系统提示词设计](#系统提示词设计)

---

## 整体架构

OpenLink 由两个独立组件构成：

```
┌─────────────────────────────────┐
│        AI 网页（浏览器）          │
│  Gemini / ChatGPT / DeepSeek …  │
└──────────────┬──────────────────┘
               │ 输出 <tool> XML 或 JSON
               ↓
┌─────────────────────────────────┐
│       Chrome 扩展（内容脚本）      │
│  extension/src/content/index.ts  │
│  • DOM 变化监听 / 输入事件监听     │
│  • 解析工具调用、渲染执行卡片       │
│  • 结果注入编辑器并自动发送        │
└──────────────┬──────────────────┘
               │ HTTP POST Bearer Token
               ↓
┌─────────────────────────────────┐
│   本地 Go HTTP 服务（127.0.0.1）  │
│  internal/server/server.go       │
│  • Gin 路由 + CORS + Token 鉴权   │
│  • /exec → Executor → Tool       │
└──────────────┬──────────────────┘
               │
       ┌───────┴────────┐
       ↓                ↓
┌─────────────┐  ┌──────────────┐
│ 文件系统操作  │  │  Shell 命令   │
│ (沙箱保护)   │  │ (危险命令过滤) │
└─────────────┘  └──────────────┘
```

**核心设计思路**：让网页端 AI 通过"工具调用 XML → 扩展拦截 → 本地服务执行"这条链路获得本地文件系统和命令执行能力，同时通过沙箱和 Token 鉴权保证安全性。

---

## Chrome 扩展实现原理

源文件：`extension/src/content/index.ts`

### 2.1 平台自适应配置

扩展通过 `getSiteConfig()` 函数根据当前域名返回不同平台的 UI 选择器和行为配置：

```typescript
function getSiteConfig(): SiteConfig {
  const h = location.hostname;
  if (h.includes('aistudio.google.com'))
    return {
      editor: 'ms-prompt-input-wrapper textarea',
      sendBtn: 'run-button button',
      fillMethod: 'value',
      useObserver: true,
      responseSelector: '.response-container',
    };
  if (h.includes('chat.openai.com') || h.includes('chatgpt.com'))
    return {
      editor: '.ProseMirror[contenteditable="true"]',
      sendBtn: 'button[data-testid="send-button"]',
      fillMethod: 'prosemirror',
      useObserver: true,
      responseSelector: '[data-message-author-role="assistant"]',
    };
  // ...更多平台
}
```

`fillMethod` 决定如何向编辑器写入文本，有四种策略：

| fillMethod | 适用场景 | 原理 |
|---|---|---|
| `value` | 普通 `<textarea>` | 直接修改 `.value` 属性并触发 React 合成事件 |
| `execCommand` | ContentEditable 富文本 | 调用已废弃但仍可用的 `document.execCommand('insertText')`（注：该 API 已被 W3C 列为废弃，未来浏览器版本可能移除，需关注兼容性变化） |
| `prosemirror` | ProseMirror 编辑器 | 模拟键盘 Ctrl+A 全选后粘贴 |
| `paste` | 特殊注入页面 | 通过 injected.js 消息通信写入 |

### 2.2 工具调用检测：两种方式

**方式 1：DOM 变化监听（useObserver=true）**

对支持的平台（AI Studio、Gemini、ChatGPT 等）使用 `MutationObserver`：

```typescript
function startDOMObserver(config: SiteConfig) {
  const observer = new MutationObserver(
    debounce(handleMutations, 800, { maxWait: 3000 })
  );
  observer.observe(document.body, { childList: true, subtree: true });
}
```

- **防抖**：800ms 内不再有 DOM 变化才触发，最多等待 3000ms
- **目的**：AI 流式输出时 DOM 频繁更新，防抖可避免重复处理半成品工具调用

**方式 2：输入事件监听（useObserver=false）**

对不支持 DOM Observer 的平台（DeepSeek、Kimi 等），监听用户将工具结果粘贴/填写到输入框时的事件。

### 2.3 工具调用解析

从 AI 响应中提取工具调用，支持两种格式：

**XML 格式**（主格式）：
```xml
<tool name="read_file" call_id="a3f9k">
  <parameter name="path">src/main.go</parameter>
  <parameter name="limit">50</parameter>
</tool>
```

解析函数 `parseXmlToolCall(raw)` 用正则提取 `name`、`call_id` 和所有 `<parameter>` 标签。

**JSON 格式**（备用）：
```json
{"name": "read_file", "args": {"path": "src/main.go", "limit": 50}}
```

`tryParseToolJSON(raw)` 支持宽松解析，处理 AI 输出的格式不规范情况。

### 2.4 防重复执行

每次执行工具调用后，将 `call_id` 记录到 `localStorage`（TTL 7 天）：

```typescript
function markExecuted(callId: string) {
  const key = `openlink_executed_${callId}`;
  localStorage.setItem(key, String(Date.now() + 7 * 86400 * 1000));
}

function isExecuted(callId: string): boolean {
  const val = localStorage.getItem(`openlink_executed_${callId}`);
  return val != null && Number(val) > Date.now();
}
```

这防止了页面刷新或 MutationObserver 重新触发导致的重复执行。

### 2.5 执行卡片 UI

每发现一个工具调用，扩展在 AI 响应上方渲染一张可视化卡片：

- 显示工具名称、参数列表
- 提供手动「执行」按钮（用户确认后才真正发起请求）
- 执行成功后显示结果摘要和状态图标

### 2.6 输入框快捷补全

在任意 AI 平台输入框中，扩展监听 `input` 事件：

- **输入 `/`**：请求 `GET /skills`，弹出 Skill 选择列表，选中后插入工具调用 XML
- **输入 `@`**：请求 `GET /files?q=<输入>`，弹出文件路径补全，选中后插入路径

弹出框支持键盘导航（↑↓ + Enter），结果有缓存（Skills 30s，文件 5s），并通过版本计数器防止竞态条件。

---

## Go 服务端实现原理

### 3.1 HTTP 路由与中间件

服务端使用 Gin 框架（`internal/server/server.go`），在 `setupRoutes()` 中按顺序注册两个全局中间件后挂载路由：

**中间件 1：CORS**

```go
c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
c.Writer.Header().Set("Access-Control-Allow-Methods", "POST, GET, OPTIONS")
c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
```

必须允许所有来源，因为浏览器扩展的内容脚本以网页域名发起请求，而服务运行在 `127.0.0.1`，属于跨域请求。

**中间件 2：Token 鉴权**

```go
s.router.Use(security.AuthMiddleware(s.config.Token))
```

除 `/health`（健康检查）外，所有路由都需要验证 `Authorization: Bearer <token>` 请求头。

**路由列表**：

| 路由 | 方法 | 用途 |
|---|---|---|
| `/health` | GET | 健康检查（无需鉴权） |
| `/auth` | POST | 验证 Token 有效性 |
| `/config` | GET | 返回工作目录和超时配置 |
| `/tools` | GET | 列出所有可用工具及元数据 |
| `/exec` | POST | **核心接口**：执行工具调用 |
| `/prompt` | GET | 获取动态注入系统信息后的初始化提示词 |
| `/skills` | GET | 列出所有可用 Skills |
| `/files` | GET | 搜索工作目录下的文件 |

### 3.2 Token 生成与存储

Token 在首次启动时自动生成，持久化到用户主目录（`internal/security/auth.go`）：

```go
func LoadOrCreateToken() (string, error) {
    path := filepath.Join(home, ".openlink", "settings.json")
    // 1. 尝试读取已有 Token
    if settings.Token != "" {
        return settings.Token, nil
    }
    // 2. 生成 32 字节随机数，hex 编码为 64 字符字符串
    b := make([]byte, 32)
    rand.Read(b)
    token := hex.EncodeToString(b)
    // 3. 以 0600 权限写入（仅当前用户可读）
    os.WriteFile(path, data, 0600)
    return token, nil
}
```

**鉴权验证**使用恒定时间比较，防止计时攻击：

```go
subtle.ConstantTimeCompare([]byte(auth), []byte("Bearer "+token)) == 1
```

### 3.3 /exec 请求处理

`handleExec` 是系统核心，处理所有工具调用请求：

```
收到 POST /exec
    ↓
JSON 反序列化 → ToolRequest
    ↓
AI 兼容性修复（edit 工具的 \t→\n 转换）
    ↓
context.WithTimeout（默认 60 秒）
    ↓
executor.Execute(ctx, req)
    ↓
返回 JSON ToolResponse
```

**AI 兼容性修复**：部分 AI 模型会将 `edit` 工具的换行符误写为 `\t`，服务端在执行前自动修正：

```go
func fixTabNewlines(s string) string {
    if strings.Contains(s, "\n") { return s }  // 已有真实换行，不处理
    if !strings.Contains(s, "\t") { return s }  // 没有 \t，不处理
    // 在每个 \t 前插入 \n，保留 \t 作为缩进："\t\tfoo" → "\n\t\tfoo"
    return strings.ReplaceAll(s, "\t", "\n\t")
}
```

### 3.4 ToolRequest 兼容性反序列化

不同 AI 模型的 JSON 输出字段名可能不同（`args` vs `arguments`），`ToolRequest` 用自定义 `UnmarshalJSON` 兼容两种格式：

```go
func (r *ToolRequest) UnmarshalJSON(data []byte) error {
    type raw struct {
        Name      string                 `json:"name"`
        Args      map[string]interface{} `json:"args"`
        Arguments map[string]interface{} `json:"arguments"`
    }
    var v raw
    json.Unmarshal(data, &v)
    r.Name = v.Name
    if v.Args != nil {
        r.Args = v.Args
    } else {
        r.Args = v.Arguments  // 兼容 "arguments" 字段名
    }
    return nil
}
```

### 3.5 动态提示词注入（/prompt）

`handlePrompt` 读取 `prompts/init_prompt.txt` 后做三步处理：

1. **注入系统信息**：将 `{{SYSTEM_INFO}}` 占位符替换为实时信息（OS、工作目录、主机名、当前时间）
2. **注入 Skills 列表**：扫描所有 Skills 目录，将可用 Skills 追加到提示词末尾
3. **追加初始化回复**：固定追加 `你好，我是 openlink，请问有什么可以帮你？`，确保 AI 首次响应友好

### 3.6 /files 文件列表

`handleListFiles` 通过 `filepath.WalkDir` 递归遍历工作目录：

- 跳过 `.git`、`node_modules`、`dist`、`build`、`vendor`、`.next` 等目录
- 验证每个文件不是指向工作目录外的符号链接（防止信息泄露）
- 支持关键词过滤（`?q=xxx`，不区分大小写）
- 最多返回 50 条结果

---

## 安全沙箱机制

源文件：`internal/security/sandbox.go`

### 4.1 文件路径沙箱（SafePath）

所有文件操作工具在执行前必须调用 `SafePath()` 验证路径合法性：

```go
func SafePath(rootDir, targetPath string) (string, error) {
    // 1. 解析 rootDir 的真实路径（跟随符号链接）
    absRoot, _ := filepath.EvalSymlinks(rootDir)
    
    // 2. 拼接目标路径
    joined := filepath.Join(absRoot, targetPath)
    
    // 3. 解析目标路径的真实路径（文件不存在时 fallback 到 Abs）
    absTarget, _ := filepath.EvalSymlinks(joined)
    // fallback: absTarget, _ = filepath.Abs(joined)
    
    // 4. 验证目标必须在 rootDir 内
    if !strings.HasPrefix(absTarget, absRoot+"/") && absTarget != absRoot {
        return "", errors.New("path outside sandbox")
    }
    return absTarget, nil
}
```

关键点：
- **先解析符号链接，再验证路径**：防止 `../../../etc/passwd` 或指向外部目录的符号链接绕过沙箱
- **新建文件时的 fallback**：文件尚不存在时 `EvalSymlinks` 会失败，改用 `filepath.Abs` 验证路径是否合规

对于需要访问工作目录外特定位置的工具（如 Skills 文件），使用 `SafeAbsPath()` 并传入白名单根目录。

### 4.2 危险命令过滤（IsDangerousCommand）

`exec_cmd` 工具在执行前调用此函数过滤危险命令：

```go
// 多词模式：子串匹配（含空格，不会误匹配普通路径）
var dangerousPatterns = []string{
    "rm -rf", "rm -fr", "> /dev/", "chmod 777", "kill -9",
}

// 单词命令：单词边界匹配（避免把文件路径中的 "nc" 误判为 netcat）
var dangerousCommands = []string{
    "mkfs", "format", "nc", "netcat", "sudo", "reboot", "shutdown",
}
```

**单词边界匹配算法**：

```go
before := abs == 0 || isCmdSeparator(lower[abs-1])
after := abs+len(word) >= len(lower) || isCmdSeparator(lower[abs+len(word)])
if before && after { return true }
```

分隔符包括：空格、制表符、换行、分号、管道、`&`、括号等 Shell 特殊字符。这样 `/path/to/nc_tool` 不会被误判，而 `nc localhost 4444` 会被正确拦截。

---

## 工具系统设计

### 5.1 Tool 接口

所有工具实现统一接口（`internal/tool/tool.go`）：

```go
type Tool interface {
    Name() string
    Description() string
    Parameters() interface{}
    Validate(args map[string]interface{}) error
    Execute(ctx *Context) *Result
}
```

`Context` 携带执行上下文：
```go
type Context struct {
    Args   map[string]interface{}
    Config *types.Config
    Ctx    context.Context  // 携带超时控制
}
```

### 5.2 工具注册与分发

`Registry`（`internal/tool/registry.go`）是一个简单的名称→工具 map：

```go
type Registry struct {
    tools map[string]Tool
}

func (r *Registry) Register(t Tool) {
    r.tools[t.Name()] = t
}

func (r *Registry) Get(name string) (Tool, bool) {
    t, ok := r.tools[name]
    return t, ok
}
```

`Executor.New()` 在初始化时注册所有 11 个内置工具，并在 `Execute()` 中支持大小写不敏感查找（先精确匹配，后 `strings.ToLower` 匹配）。

### 5.3 各工具实现要点

| 工具 | 文件 | 关键实现 |
|---|---|---|
| `exec_cmd` | exec_cmd.go | `sh -c` 执行命令，Windows 改用 `cmd.exe /C`；输出截断（2000 行/50KB） |
| `list_dir` | list_dir.go | 目录名加 `/` 后缀；沙箱验证 |
| `read_file` | read_file.go | 支持 `offset`/`limit` 分页（最大 2000 行/50KB）；行号前缀 |
| `write_file` | write_file.go | `append` 模式用 `O_APPEND`；覆写模式先创建父目录 |
| `glob` | glob.go | 结果按修改时间倒序；最多 100 条 |
| `grep` | grep.go | 优先调用系统 `rg`（ripgrep），不可用时降级为 Go 原生正则；最多 100 条 |
| `edit` | edit.go | **10 种替换策略级联**（详见 5.4） |
| `web_fetch` | web_fetch.go | 屏蔽私有 IP（SSRF 防护）；HTML 转纯文本；1MB 大小限制 |
| `question` | question.go | 阻塞等待用户输入；可选多选题模式 |
| `skill` | skill.go | 加载 SKILL.md；列出同目录关联文件（最多 10 个） |
| `todo_write` | todo_write.go | 写入 `.todos.json` 到工作目录 |

### 5.4 edit 工具的 10 种替换策略

`edit` 工具是最复杂的工具，因为 AI 生成的代码字符串与文件实际内容经常存在细微差异（空格、缩进、换行符等）。工具实现了一个**级联替换策略**，依次尝试从严格到宽松的匹配方式：

```
1. SimpleReplacer           — 精确字符串匹配
2. LineTrimmedReplacer      — 每行 trim 后匹配（处理行首尾空格差异）
3. BlockAnchorReplacer      — 用首末行作为锚点 + Levenshtein 距离评分
4. WhitespaceNormalizedReplacer — 压缩所有空白字符后匹配
5. IndentationFlexibleReplacer  — 移除公共缩进后匹配
6. EscapeNormalizedReplacer     — 处理转义字符（\n、\t 等）
7. TrimmedBoundaryReplacer      — 匹配 trim 后的整块内容
8. TabNewlineReplacer           — \t 视为换行符处理
9. ContextAwareReplacer         — 利用上下文行精确定位
10. MultiOccurrenceReplacer     — 允许多处匹配时的最佳匹配选择
```

只要有一种策略找到**唯一**匹配，立即执行替换并返回，不继续尝试后续策略。若找到多个匹配（歧义）或零匹配，继续尝试下一种策略。

### 5.5 身份提醒机制

`Executor.Execute()` 在每次工具调用完成后自动追加身份提醒：

```go
n := e.callCount.Add(1)
const reinjectEvery = 20

if n%reinjectEvery == 0 {
    // 每 20 次调用，重新注入完整提示词
    if data, err := os.ReadFile(...); err == nil {
        resp.Output += "\n\n[系统重新注入提示词]\n" + string(data)
    }
} else {
    // 其余次数追加简短提醒
    resp.Output += "\n\n[系统提示] 请记住你是 openlink，严格遵循工具调用规范，不要忘记自己的身份和指令。"
}
```

这防止了在长对话中 AI 模型因上下文漂移而忘记工具调用规范（"目标漂移"问题）。

---

## Skills 扩展系统

源文件：`internal/skill/loader.go`

### 6.1 目录扫描与优先级

`SkillDirs()` 返回按优先级排序的搜索目录列表：

```go
func SkillDirs(rootDir string) []string {
    return []string{
        filepath.Join(rootDir, ".skills"),               // 项目本地（最高优先级）
        filepath.Join(rootDir, ".openlink", "skills"),
        filepath.Join(rootDir, ".agent", "skills"),
        filepath.Join(rootDir, ".claude", "skills"),
        filepath.Join(home, ".openlink", "skills"),
        filepath.Join(home, ".agent", "skills"),
        filepath.Join(home, ".claude", "skills"),        // 用户主目录（最低优先级）
    }
}
```

`LoadInfos()` 用 `seen` map 记录已加载的 Skill 名称，**同名 Skill 以先找到的为准**（即优先级越高的目录优先）。

### 6.2 Skill 文件格式解析

每个 Skill 是一个子目录，其中包含 `SKILL.md`（大小写不敏感）：

```
.skills/
└── deploy/
    └── SKILL.md
```

`SKILL.md` 使用 YAML frontmatter 声明元数据：

```markdown
---
name: deploy
description: 项目部署流程
---

## 部署步骤

1. 构建镜像...
```

`parse()` 函数手动解析 frontmatter（不依赖 YAML 库，仅需标准库）：

```go
end := strings.Index(content[3:], "---")  // 找结束 "---"
front := content[3 : end+3]               // 提取 frontmatter 内容
for _, line := range strings.Split(front, "\n") {
    if k, v, ok := strings.Cut(line, ":"); ok {
        switch strings.TrimSpace(k) {
        case "name":        name = v
        case "description": description = v
        }
    }
}
```

### 6.3 Skill 工具执行流程

当 AI 调用 `skill` 工具时（`internal/tool/skill.go`）：

1. 验证 skill 名称（禁止 `..`、`/`、`\` 字符，防路径遍历）
2. 调用 `skill.FindSkill()` 搜索 SKILL.md
3. 返回内容包括：
   - SKILL.md 全文
   - Skill 目录下的关联文件列表（最多 10 个）及其绝对路径

---

## 完整调用链路

以 AI 请求读取文件 `src/main.go` 为例：

```
① AI 输出工具调用 XML：
   <tool name="read_file" call_id="x7k2p">
     <parameter name="path">src/main.go</parameter>
     <parameter name="limit">50</parameter>
   </tool>

② Chrome 扩展内容脚本（MutationObserver 检测到新 DOM）：
   • parseXmlToolCall() 提取 name="read_file", args={path, limit}
   • isExecuted("x7k2p") → false（未执行过）
   • 渲染执行卡片，等待用户点击「执行」

③ 用户点击执行，扩展发起 HTTP 请求：
   POST http://127.0.0.1:39527/exec
   Authorization: Bearer <64-char-hex-token>
   Content-Type: application/json
   {"name":"read_file","args":{"path":"src/main.go","limit":50}}

④ Go 服务端 CORS 中间件：
   • 设置 Access-Control-Allow-Origin: *

⑤ Token 鉴权中间件：
   • 提取 Authorization 头
   • subtle.ConstantTimeCompare() 验证 Token
   • 验证失败 → 返回 401

⑥ handleExec()：
   • JSON 反序列化为 ToolRequest
   • 不是 edit 工具，跳过 fixTabNewlines
   • context.WithTimeout(60s)
   • executor.Execute(ctx, req)

⑦ Executor.Execute()：
   • registry.Get("read_file") → ReadFileTool
   • t.Validate({path, limit}) → 验证 path 非空
   • t.Execute(context)

⑧ ReadFileTool.Execute()：
   • security.SafePath(rootDir, "src/main.go")
     - EvalSymlinks(rootDir) → /workspace
     - Abs(/workspace/src/main.go) → /workspace/src/main.go
     - 验证以 /workspace/ 开头 → 通过
   • 读取文件内容
   • 按 limit=50 截取前 50 行
   • 每行加行号前缀："1. fn main() {..."
   • 返回 Result{Status:"success", Output:"..."}

⑨ Executor 后处理：
   • callCount += 1（假设第 5 次调用）
   • 追加身份提醒字符串
   • 返回 ToolResponse

⑩ handleExec 返回：
   HTTP 200
   {"status":"success","output":"1. fn main() {\n2.     ..."}

⑪ Chrome 扩展接收响应：
   • 更新执行卡片显示成功状态
   • markExecuted("x7k2p") 写入 localStorage
   • fillEditor(result.output) 将结果填入 AI 输入框
   • clickSendButton() 自动发送给 AI

⑫ AI 收到工具执行结果，继续生成响应
```

---

## 系统提示词设计

源文件：`prompts/init_prompt.txt`

### 8.1 提示词结构

初始化提示词是 AI 行为规范的核心，分为以下几个部分：

| 段落 | 内容 | 作用 |
|---|---|---|
| 身份声明 | 定义 AI 是 openlink CLI 工具 | 确立角色 |
| `{{SYSTEM_INFO}}` | 运行时注入 OS/目录/时间信息 | 上下文感知 |
| 安全准则 | 拒绝恶意代码、SSRF 等 | 安全边界 |
| 回复风格 | 简洁、直接、Markdown 格式 | 输出质量 |
| 工具调用规范 | XML 格式定义、call_id 要求 | 协议一致性 |
| 工具文档 | 11 个工具的参数和示例 | 工具使用指南 |
| 安全限制 | 沙箱范围、危险命令、超时 | 与服务端对齐 |
| Skills 说明 | 如何加载和使用 Skill | 扩展能力引导 |

### 8.2 动态注入流程

```
init_prompt.txt（静态模板）
    ↓
替换 {{SYSTEM_INFO}} → 实时系统信息
    ↓
追加可用 Skills 列表（从磁盘实时扫描）
    ↓
追加初始化回复模板
    ↓
通过 GET /prompt 返回给 Chrome 扩展
    ↓
扩展将完整内容填入 AI 系统提示词（System Instructions）
    ↓
AI 加载规范，具备工具调用能力
```

### 8.3 工具调用格式规范

提示词中定义的标准工具调用格式：

```xml
<tool name="工具名" call_id="唯一ID">
  <parameter name="参数名">参数值</parameter>
</tool>
```

- `call_id` 由 AI 生成，用于唯一标识一次调用（防重复执行）
- 参数值为纯文本，多行内容直接换行，无需转义

---

## 关键设计决策总结

| 设计决策 | 原因 |
|---|---|
| 使用 XML 而非 JSON 作为工具调用格式 | AI 模型在文本中嵌入 XML 更自然，且浏览器可用正则轻松提取 |
| Token 认证而非 IP 白名单 | 同一机器上的恶意网页也在 127.0.0.1，Token 是更有效的访问控制 |
| 恒定时间 Token 比较 | 防止通过响应时间差异推断 Token 内容（计时攻击） |
| edit 工具 10 级替换策略 | AI 生成代码时常有格式细微差异，必须有容错能力 |
| 每次工具响应追加身份提醒 | 长对话中防止 AI 上下文漂移，忘记工具调用规范 |
| 符号链接解析后再校验路径 | 直接字符串比较无法防止通过符号链接逃离沙箱 |
| grep 优先使用系统 rg | ripgrep 搜索速度远超 Go 原生正则，大型代码库体验更好 |
| 防抖 DOM Observer（800ms/3000ms） | AI 流式输出会频繁触发 DOM 变化，直接处理会消费未完成的工具调用 |
