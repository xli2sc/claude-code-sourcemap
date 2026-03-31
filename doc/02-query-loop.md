# 02 - 查询循环（核心引擎）

> 查询循环是 Claude Code 的心脏 — 驱动所有交互，包括主 REPL 和子 Agent。

## 关键文件

| 文件 | 职责 |
|------|------|
| `src/query.ts` | 核心查询生成器，工具编排 (15 KB) |
| `src/screens/REPL.tsx` | 主交互循环，消费查询结果 (22 KB) |
| `src/utils/generators.ts` | `all()` 并发生成器 |

## 查询流程

```mermaid
sequenceDiagram
    participant REPL as REPL.tsx
    participant Query as query()
    participant Claude as Claude API
    participant Tools as 工具系统

    REPL->>Query: query(messages, tools, systemPrompt)
    Query->>Claude: POST /messages（流式）

    loop 流式文本
        Claude-->>Query: text delta
        Query-->>REPL: yield AssistantMessage
    end

    alt 包含 tool_use
        Claude-->>Query: tool_use 块
        Query->>Query: 判断工具类型

        alt 全部只读
            Query->>Tools: runToolsConcurrently()（上限 10）
        else 包含写操作
            Query->>Tools: runToolsSerially()
        end

        Tools-->>Query: tool_results
        Query->>Claude: 携带 tool_results 继续（递归）
    end

    Query-->>REPL: yield 最终 AssistantMessage
```

## 核心代码结构

### query() 生成器

```typescript
async function* query(messages, tools, systemPrompt) {
  // 1. 调用 Claude API
  const response = await querySonnet(messages, tools, systemPrompt)

  // 2. 提取 tool_use 块
  const toolUses = response.content.filter(b => b.type === 'tool_use')

  if (toolUses.length === 0) {
    yield response  // 无工具调用，直接返回
    return
  }

  // 3. 执行工具
  const allReadOnly = toolUses.every(t => findTool(t).isReadOnly())
  const results = allReadOnly
    ? yield* runToolsConcurrently(toolUses, concurrency: 10)
    : yield* runToolsSerially(toolUses)

  // 4. 递归继续对话
  messages.push(response, ...results)
  yield* query(messages, tools, systemPrompt)  // 递归
}
```

### REPL 消费模式

```typescript
// REPL.tsx - 简化版
async function onQuery(userMessage) {
  messages.push(userMessage)

  for await (const message of query(messages, tools, systemPrompt)) {
    if (message.type === 'progress') {
      updateToolUI(message)        // 更新工具执行进度
    } else if (message.type === 'result') {
      messages.push(message)       // 追加到消息历史
      renderMessage(message)       // 渲染到终端
    }
  }
}
```

## 并发策略

```mermaid
flowchart TD
    Start["收到 tool_use 块"] --> Count{"几个工具？"}
    Count -->|1 个| Single["直接执行"]
    Count -->|多个| ReadOnly{"全部只读？"}
    ReadOnly -->|是| Concurrent["并发执行<br/>MAX_CONCURRENCY = 10"]
    ReadOnly -->|否| Serial["串行执行"]

    Concurrent --> AllGen["all() 生成器"]
    AllGen --> Race["Promise.race()<br/>哪个先完成就先 yield"]
    Race --> Collect["按原始顺序收集结果"]

    Serial --> OneByOne["按顺序逐个执行"]
    OneByOne --> Collect

    Single --> Collect
    Collect --> Continue["发送 tool_results → 递归 query()"]
```

## Binary Feedback（A/B 测试）

仅对 Anthropic 内部用户启用：

```mermaid
graph TD
    Input["用户输入"] --> Gate{"Statsig 开关<br/>启用 A/B？"}
    Gate -->|否| Normal["正常查询"]
    Gate -->|是| Fork["同时生成两个响应"]
    Fork --> A["候选 A"]
    Fork --> B["候选 B"]
    A --> UI["显示两个选项"]
    B --> UI
    UI --> Choose["用户选择更好的"]
    Choose --> Continue["使用选中的继续"]
```

## 学习建议

1. **先读** `query.ts` 的 `query()` 函数 — 理解递归流式模式
2. **再读** `runToolsConcurrently()` 和 `runToolsSerially()` — 理解并发策略
3. **然后读** `REPL.tsx` 的 `onQuery` — 理解消费端
