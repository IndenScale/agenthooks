<!-- markdownlint-disable MD060 -->

# Agent Hooks 规范定义

本文档定义 Agent Hooks 格式的完整规范，包括事件类型、执行模式、匹配器和推荐实践。

---

## 1. 事件类型 (Event Types)

Agent Hooks 支持 12 种事件类型，分为 5 个类别：

### 1.1 Session 生命周期

| 事件 | 触发时机 | 可阻断 | 推荐模式 |
|------|----------|--------|----------|
| `pre-session` | 代理会话开始时 | ✅ 可以 | 同步 |
| `post-session` | 代理会话结束时 | ✅ 可以 | 同步 |

### 1.2 Agent 循环

| 事件 | 触发时机 | 可阻断 | 推荐模式 |
|------|----------|--------|----------|
| `pre-agent-turn` | 代理处理用户输入前 | ✅ 可以 | 同步 |
| `post-agent-turn` | 代理完成处理后 | ✅ 可以 | 同步 |
| `pre-agent-turn-stop` | 代理停止响应前 | ✅ **质量门禁** | **同步** |
| `post-agent-turn-stop` | 代理停止响应后 | ✅ 可以 | 同步 |

### 1.3 工具拦截（核心）

| 事件 | 触发时机 | 可阻断 | 推荐模式 |
|------|----------|--------|----------|
| `pre-tool-call` | 工具执行前 | ✅ **推荐** | **同步** |
| `post-tool-call` | 工具成功执行后 | ✅ 可以 | 同步 |
| `post-tool-call-failure` | 工具执行失败后 | ✅ 可以 | 同步 |

### 1.4 Subagent 生命周期

| 事件 | 触发时机 | 可阻断 | 推荐模式 |
|------|----------|--------|----------|
| `pre-subagent` | Subagent 启动时 | ✅ 可以 | 同步 |
| `post-subagent` | Subagent 结束时 | ✅ 可以 | 同步 |

### 1.5 上下文管理

| 事件 | 触发时机 | 可阻断 | 推荐模式 |
|------|----------|--------|----------|
| `pre-context-compact` | 上下文压缩前 | ✅ 可以 | 同步 |
| `post-context-compact` | 上下文压缩后 | ✅ 可以 | 同步 |

> **命名规范**: 所有事件统一使用 `pre-*` 和 `post-*` 前缀，保持语义清晰和对称性。

---

## 2. 输出协议

### 2.1 输出流

Hook 脚本通过退出码和输出流与 Agent 通信：

| 输出流 | 描述 |
|--------|------|
| **退出码** | 执行结果信号 |
| **stdout** | 机器可解析的 JSON，用于控制与通信 |
| **stderr** | 人类可读文本，用于错误与反馈 |

### 2.2 退出码

| 退出码 | 含义 |
|--------|------|
| `0` | 执行成功，操作继续 |
| `2` | 执行完成，操作阻断 |
| 其他 | 执行异常，操作继续 |

### 2.3 stdout (控制与通信)

**触发条件：** 仅在 Exit Code 为 `0`（成功）时生效。

**主要用途：** 传输 JSON 配置对象，告诉 Agent 允许、拒绝、修改输入或添加上下文。

**解析方式：** Agent 会尝试将 stdout 解析为 JSON。

示例：
```bash
# 通过 stdout JSON 返回决策
echo '{"decision": "allow", "log": "Command validated"}'
exit 0
```

### 2.4 stderr (错误与反馈)

**触发条件：**
- Exit Code `2` (阻断)：stderr 内容作为阻断理由展示给用户
- 其他非 0 退出码：stderr 仅作为调试/日志文本

**主要用途：** 传输错误信息、拒绝理由或调试日志。

**解析方式：** Agent 将 stderr 视为纯文本字符串。

示例：
```bash
echo "Dangerous command blocked: rm -rf / would destroy the system" >&2
exit 2
```

---

## 3. 执行模式分类

**重要：所有 Hook 默认均为同步模式（`async = false`）。如需异步执行，必须显式设置 `async: true`。**

### 3.1 同步模式 (Sync) - 默认

```yaml
---
name: security-check
trigger: pre-tool-call
async: false # 默认，可省略
---
```

**特性：**

- 等待 hook 完成后再继续执行
- 可以阻断操作（通过 exit code 2，stderr 作为理由）
- 可以修改输入参数（通过 stdout JSON，仅在 exit code 为 0 时）
- 适用于安全检查、权限验证、输入校验

**适用事件：** 所有事件（默认）

### 3.2 异步模式 (Async)

```yaml
---
name: auto-format
trigger: post-tool-call
async: true
---
```

**特性：**

- 立即返回，不等待 hook 完成
- 无法阻断操作（退出码不用于阻断判断）
- 无法修改输入参数
- stdout 和 stderr 仅用于日志/调试
- 适用于格式化、通知、日志、分析

**适用事件：** 所有事件（如需异步执行，可在任何事件上显式设置 `async: true`）

### 3.3 模式选择决策树

```text
是否需要阻断操作？
├── 是 → 同步模式
│       └── 输出到 stderr 并使用 exit code 2
└── 否 → 异步模式
        └── 是否需要等待结果？
            ├── 是 → 同步模式（谨慎使用）
            └── 否 → 异步模式（推荐）
```

---

## 4. 匹配器 (Matcher)

匹配器用于过滤 hook 的触发条件，仅适用于工具相关的事件。

### 4.1 匹配器配置

```yaml
---
name: block-dangerous-commands
trigger: pre-tool-call
matcher:
  tool: "Shell" # 工具名正则匹配
  pattern: "rm -rf /|mkfs|>:/dev/sda" # 参数内容正则匹配
---
```

### 4.2 匹配器字段

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `tool` | string | 否 | 工具名正则表达式（如 `Shell`, `WriteFile`, `ReadFile`） |
| `pattern` | string | 否 | 工具输入参数的正则表达式匹配 |

### 4.3 匹配逻辑

- 如果 `tool` 指定，则只有该工具会触发 hook
- 如果 `pattern` 指定，则只有参数内容匹配时触发
- 两者都指定时，必须**同时满足**
- 都不指定时，hook 对所有工具触发

### 4.4 常见匹配器示例

```yaml
# 仅拦截 Shell 工具
matcher:
  tool: "Shell"

# 拦截特定文件类型的写入
matcher:
  tool: "WriteFile"
  pattern: "\.(py|js|ts)$"

# 拦截包含敏感关键词的命令
matcher:
  tool: "Shell"
  pattern: "(rm -rf|mkfs|dd if=/dev/zero)"

# 拦截特定目录的操作
matcher:
  pattern: "/etc/passwd|/var/www"
```

---

## 5. 事件数据结构 (Event Data)

Hook 脚本通过 **stdin** 接收 JSON 格式的事件数据。

### 5.1 基础事件字段

所有事件都包含以下字段：

```json
{
  "event_type": "pre-tool-call",
  "timestamp": "2024-01-15T10:30:00Z",
  "session_id": "sess-abc123",
  "work_dir": "/home/user/project",
  "context": {}
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `event_type` | string | 事件类型 |
| `timestamp` | string | ISO 8601 格式时间戳 |
| `session_id` | string | 会话唯一标识 |
| `work_dir` | string | 当前工作目录 |
| `context` | object | 额外上下文信息 |

### 5.2 工具事件 (pre-tool-call / post-tool-call / post-tool-call-failure)

```json
{
  "event_type": "pre-tool-call",
  "timestamp": "2024-01-15T10:30:00Z",
  "session_id": "sess-abc123",
  "work_dir": "/home/user/project",
  "tool_name": "Shell",
  "tool_input": {
    "command": "rm -rf /tmp/old-files"
  },
  "tool_use_id": "tool_123"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `tool_name` | string | 工具名称（如 Shell, WriteFile） |
| `tool_input` | object | 工具的输入参数 |
| `tool_use_id` | string | 工具调用唯一标识 |

### 5.3 Subagent 事件 (pre-subagent / post-subagent)

```json
{
  "event_type": "pre-subagent",
  "timestamp": "2024-01-15T10:30:00Z",
  "session_id": "sess-abc123",
  "work_dir": "/home/user/project",
  "subagent_name": "code-reviewer",
  "subagent_type": "coder",
  "task_description": "Review the authentication module"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `subagent_name` | string | Subagent 名称 |
| `subagent_type` | string | Subagent 类型 |
| `task_description` | string | 任务描述 |

### 5.4 Session 事件 (pre-session / post-session)

**pre-session:**

```json
{
  "event_type": "pre-session",
  "timestamp": "2024-01-15T10:30:00Z",
  "session_id": "sess-abc123",
  "work_dir": "/home/user/project",
  "model": "kimi-latest",
  "args": {
    "ui": "shell",
    "agent": "default"
  }
}
```

**post-session:**

```json
{
  "event_type": "post-session",
  "timestamp": "2024-01-15T11:30:00Z",
  "session_id": "sess-abc123",
  "work_dir": "/home/user/project",
  "duration_seconds": 3600,
  "total_steps": 25,
  "exit_reason": "user_exit"
}
```

### 5.5 Agent Turn Stop 事件（质量门禁）

**pre-agent-turn-stop:**

```json
{
  "event_type": "pre-agent-turn-stop",
  "timestamp": "2024-01-15T10:35:00Z",
  "session_id": "sess-abc123",
  "work_dir": "/home/user/project",
  "stop_reason": "no_tool_calls",
  "step_count": 5,
  "final_message": {
    "role": "assistant",
    "content": "任务已完成"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `stop_reason` | string | 停止原因：`no_tool_calls`、`tool_rejected`、`max_steps` |
| `step_count` | integer | 本轮执行的步数 |
| `final_message` | object | 助手的最终消息（可能为 null） |

**使用场景：质量门禁**

`pre-agent-turn-stop` 事件用于在允许代理完成前强制执行质量标准：

```yaml
---
name: enforce-tests
description: 确保测试通过才允许完成
trigger: pre-agent-turn-stop
timeout: 60000
async: false
priority: 999
---
```

当 `pre-agent-turn-stop` hook 阻断时（exit 2 或 `decision: deny`），代理会收到反馈并继续工作而非停止。这创造了一个强大的质量控制机制。

---

## 6. 推荐实践汇总

### 6.1 按事件类型的推荐用法

| 事件类型 | 同步/异步 | 推荐用途 | 示例场景 |
|----------|-----------|----------|----------|
| `pre-session` | 同步 | 初始化、日志记录 | 发送会话开始通知、初始化环境 |
| `post-session` | 同步 | 清理、统计、通知 | 生成会话摘要、发送 Slack 通知 |
| `pre-agent-turn` | 同步 | 输入验证、安全检查 | 敏感词过滤、输入审查 |
| `post-agent-turn` | 同步 | 日志、分析 | 记录响应时间、分析输出质量 |
| `pre-tool-call` | 同步 | 安全检查、拦截 | 阻断危险命令、权限验证 |
| `post-tool-call` | 同步 | 格式化、通知 | 自动格式化代码、发送操作通知 |
| `post-tool-call-failure` | 同步 | 错误处理、重试 | 记录失败日志、发送告警 |
| `pre-subagent` | 同步 | 资源限制、审批 | 检查并发数限制、任务审批 |
| `post-subagent` | 同步 | 结果验证、清理 | 验证输出质量、回收资源 |
| `pre-context-compact` | 同步 | 备份、分析 | 备份上下文、分析压缩效果 |
| `pre-agent-turn-stop` | **同步** | **质量门禁、完成标准** | **强制测试通过、验证所有任务完成** |
| `post-agent-turn-stop` | 同步 | 清理、最终日志 | 记录轮次完成、更新指标 |

### 6.2 常见 Hook 模式

#### 模式 1: 危险操作拦截（同步 + 阻断）

```yaml
---
name: block-dangerous-commands
description: Blocks dangerous system commands
trigger: pre-tool-call
matcher:
  tool: Shell
  pattern: "rm -rf /|mkfs|dd if=/dev/zero"
timeout: 5000
async: false
priority: 999
---
```

**脚本逻辑:**

```bash
# 检查命令内容
echo "Dangerous command blocked: rm -rf / would destroy the system" >&2
exit 2
```

#### 模式 2: 代码自动格式化（异步）

```yaml
---
name: auto-format-python
description: Auto-format Python files after write
trigger: post-tool-call
matcher:
  tool: WriteFile
  pattern: "\.py$"
timeout: 30000
async: true
---
```

#### 模式 3: 敏感操作阻断（同步）

```yaml
---
name: block-prod-deploy
description: Block production deployment operations
trigger: pre-tool-call
matcher:
  tool: Shell
  pattern: "deploy.*prod|kubectl.*production"
timeout: 60000
async: false
---
```

**脚本逻辑:**

```bash
echo "This operation affects production environment and is not allowed" >&2
exit 2
```

#### 模式 4: 会话审计日志（异步）

```yaml
---
name: audit-log
description: Log all session activities
trigger: post-session
async: true
---
```

#### 模式 5: 质量门禁（同步 + pre-agent-turn-stop）

```yaml
---
name: enforce-test-coverage
description: 确保测试通过才允许完成任务
trigger: pre-agent-turn-stop
timeout: 120000
async: false
priority: 999
---
```

**脚本逻辑:**

```bash
#!/bin/bash
# enforce-quality.sh

# 从 stdin 读取事件数据
event_data=$(cat)

# 检查测试是否通过
if ! npm test 2>&1; then
    echo "测试通过前不能完成任务" >&2
    exit 2
fi

# 检查代码格式
if ! black --check . 2>&1; then
    echo "代码未格式化。先运行 'black .'" >&2
    exit 2
fi

# 所有检查通过
exit 0
```

**行为说明:**

当此 hook 以 exit 2 退出时，代理会收到 stderr 消息作为反馈并继续工作而非停止。这创造了强制执行质量标准的强大机制。

#### 模式 6: Memo 捕获提醒（同步 + pre-agent-turn-stop）

```yaml
---
name: memo-capture
description: 提醒捕获工作过程中的 fleeting thoughts
trigger: pre-agent-turn-stop
timeout: 5000
async: false
priority: 10
---
```

**脚本逻辑:**

```bash
#!/bin/bash
# Memo Capture Reminder Hook
# Trigger: pre-agent-turn-stop

PROJECT_ROOT="${MONOCO_PROJECT_ROOT:-$(pwd)}"
MONOCO_DIR="$PROJECT_ROOT/.monoco"
MEMO_PATH="$MONOCO_DIR/MEMO.md"
SENTINEL_PATH="$MONOCO_DIR/.memo_reminded"

# Ensure .monoco directory exists
mkdir -p "$MONOCO_DIR"

# Check if already reminded in this turn
if [ -f "$SENTINEL_PATH" ]; then
    exit 0
fi

# Create sentinel file to mark reminder as shown
touch "$SENTINEL_PATH"

# Count existing memos
MEMO_COUNT=0
if [ -f "$MEMO_PATH" ]; then
    MEMO_COUNT=$(grep -cE '^##\s*' "$MEMO_PATH" 2>/dev/null || echo "0")
fi

cat << GUIDANCE

╔══════════════════════════════════════════════════════════════╗
║  📝 Memo Capture                                              ║
╚══════════════════════════════════════════════════════════════╝

本轮工作即将结束，捕获任何 fleeting thoughts:

  • 遇到的技术障碍？
  • 需要后续完善的想法？
  • 值得为下轮会话保留的上下文？
  • 跳过的 "应该调查" 时刻？

当前待处理 memos: $MEMO_COUNT

GUIDANCE

exit 0
```

---

## 7. 配置优先级与执行顺序

### 7.1 优先级 (Priority)

- 范围: 0 - 1000
- 默认值: 100
- 规则: **数值越高，越早执行**

```yaml
# 安全检查优先执行
priority: 999

# 普通通知后执行
priority: 10
```

### 7.2 多 Hook 执行顺序

1. 按优先级降序排序
2. 同优先级按配置顺序执行
3. 任一 hook 阻断则停止执行后续 hook

### 7.3 异步 Hook 处理

- 异步 hook 之间并行执行
- 不等待完成，不收集结果
- 失败不影响主流程

---

## 8. 超时与错误处理

### 8.1 超时配置

- 默认值: 30000ms (30秒)
- 范围: 100ms - 600000ms (10分钟)

### 8.2 超时行为

- 超时视为 hook 失败
- 采用 **Fail Open** 策略：允许操作继续
- 记录警告日志

### 8.3 错误处理原则

| 情况 | 处理方式 |
|------|----------|
| Hook 执行失败 | 记录警告，允许操作（Fail Open） |
| Hook 返回无效 JSON (exit 0) | 记录错误，允许操作 |
| Hook 超时 | 记录警告，允许操作 |
| Exit code 2 | **阻断操作**，stderr 展示给用户 |
| 其他非零退出码 | 仅作为警告/调试日志，允许操作 |

---

## 9. 渐进式披露设计

Agent Hooks 采用渐进式披露设计，优化上下文使用：

| 层级 | 内容 | 大小 | 加载时机 |
|------|------|------|----------|
| **Metadata** | name, description, trigger | ~100 tokens | 启动时加载所有 hooks |
| **Configuration** | HOOK.md 完整内容 | < 1000 tokens | 事件触发时加载 |
| **Scripts** | 可执行脚本 | 按需 | 匹配后执行 |

---

## 10. 完整示例

### 10.1 目录结构示例

```text
~/.config/agents/             # 用户级 (XDG)
└── hooks/
    ├── security-check/
    │   ├── HOOK.md
    │   └── scripts/
    │       └── run.sh
    └── notify-slack/
        └── HOOK.md

./my-project/                 # 项目级
└── .agents/
    └── hooks/
        └── project-specific/
            └── HOOK.md
```

### 10.2 HOOK.md 示例

````markdown
---
name: block-dangerous-commands
description: Blocks dangerous shell commands that could destroy data
trigger: pre-tool-call
matcher:
  tool: Shell
  pattern: "rm -rf /|mkfs|dd if=/dev/zero|>:/dev/sda"
timeout: 5000
async: false
priority: 999
---

# Block Dangerous Commands

This hook prevents execution of dangerous system commands.

## Blocked Patterns

- `rm -rf /` - Recursive deletion of root
- `mkfs` - Filesystem formatting
- `dd if=/dev/zero` - Zeroing drives
- `>:/dev/sda` - Direct write to disk

## Exit Codes

- `0` - Command is safe, operation continues
- `2` - Command matches dangerous pattern, operation blocked

## Output

When blocking (exit code 2), outputs reason to stderr:

```
Dangerous command blocked: rm -rf / would destroy the system
```text
````

### 10.3 脚本示例 (scripts/run.sh)

```bash
#!/bin/bash
# Block dangerous commands hook

# Read event data from stdin
event_data=$(cat)

# Extract command from event
tool_input=$(echo "$event_data" | grep -o '"command": "[^"]*"' | head -1 | cut -d'"' -f4)

# Dangerous patterns
dangerous_patterns=("rm -rf /" "mkfs" "dd if=/dev/zero")

for pattern in "${dangerous_patterns[@]}"; do
    if echo "$tool_input" | grep -qE "\b${pattern}\b"; then
        echo "Dangerous command blocked: ${pattern} would destroy the system" >&2
        exit 2
    fi
done

# Command is safe
exit 0
```

---

## 附录: 字段参考表

### HOOK.md Frontmatter 字段

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `name` | string | 是 | - | Hook 标识符 (1-64 字符) |
| `description` | string | 是 | - | Hook 描述 (1-1024 字符) |
| `trigger` | string | 是 | - | 触发事件类型 |
| `matcher` | object | 否 | - | 匹配条件 |
| `timeout` | integer | 否 | 30000 | 超时时间 (毫秒) |
| `async` | boolean | 否 | false | 是否异步执行 |
| `priority` | integer | 否 | 100 | 执行优先级 (0-1000) |
| `metadata` | object | 否 | - | 额外元数据 |

### Matcher 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `tool` | string | 工具名正则表达式 |
| `pattern` | string | 参数内容正则表达式 |
