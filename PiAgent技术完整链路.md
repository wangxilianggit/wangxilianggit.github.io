# Pi Agent 技术完整链路

本文整理 Pi agent 从用户输入到最终输出的完整工作流，包括启动、资源加载、会话管理、上下文构建、模型调用、工具循环、压缩、队列和 context-mode 的作用。

---

## 1. 总览

Pi agent 的完整链路可以概括为：

```text
Pi 启动
  ↓
加载设置 / 模型 / packages / skills / extensions / context files
  ↓
打开或创建 session
  ↓
用户输入 prompt
  ↓
预处理：slash command / prompt template / skill / @file / extension input hook
  ↓
写入 session
  ↓
构建 LLM 上下文
  ↓
必要时 compaction 压缩旧上下文
  ↓
调用模型 provider
  ↓
模型输出：文本 或 tool call
  ↓
如果有 tool call：执行工具，把结果回填给模型
  ↓
重复 tool loop，直到模型最终回答
  ↓
最终 assistant 输出写入 session 并显示给用户
```

---

## 2. Pi 启动阶段

当你启动 Pi，例如：

```bash
pi
pi -c
pi -r
pi --session <id>
pi -p "some task"
```

Pi 会先进行运行时初始化。

### 2.1 加载 settings

Pi 会读取配置文件：

```text
~/.pi/agent/settings.json
<project>/.pi/settings.json
```

项目配置会覆盖全局配置。

配置内容可能包括：

- 默认模型
- thinking level
- compaction 配置
- enabled packages
- enabled tools
- sessionDir
- retry 策略
- theme
- extensions

---

### 2.2 加载认证和模型

Pi 的 `ModelRuntime` 会加载：

- 内置 provider / model
- 自定义 `models.json`
- API key / OAuth 凭据

认证优先级大致是：

```text
runtime override
→ ~/.pi/agent/auth.json
→ environment variables
→ custom resolver
```

然后确定当前会话用哪个模型：

```text
恢复 session 中的模型
→ settings 默认模型
→ 第一个可用模型
```

---

### 2.3 加载 packages / resources

Pi 会加载 package 提供的资源：

- extensions
- skills
- prompt templates
- themes

资源来源包括：

```text
~/.pi/agent/
<project>/.pi/
installed pi packages
CLI -e / --extension
```

如果项目里有 `.pi/settings.json`、`.pi/extensions`、`.agents/skills` 等，Pi 会先处理 **project trust**，防止不可信项目自动执行本地扩展。

---

### 2.4 加载 context files

Pi 会加载上下文文件：

```text
~/.pi/agent/AGENTS.md
父目录 AGENTS.md / CLAUDE.md
当前项目 AGENTS.md / CLAUDE.md
.pi/SYSTEM.md
APPEND_SYSTEM.md
```

这些文件会影响 system prompt，也就是 agent 的基础行为规则。

---

### 2.5 打开或创建 session

Pi 的 session 默认保存在：

```text
~/.pi/agent/sessions/--<cwd路径>--/<timestamp>_<uuid>.jsonl
```

如果使用：

```bash
pi -c
```

Pi 会继续最近 session。

如果使用：

```bash
pi -r
```

会打开 session picker。

如果使用：

```bash
pi --no-session
```

就是临时会话，不落盘。

---

## 3. 用户输入阶段

在 interactive mode 里，用户在 editor 输入内容。

用户输入可以是几种形式。

### 3.1 普通 prompt

```text
帮我修这个 bug
```

普通 prompt 会作为用户消息进入 agent。

---

### 3.2 Slash command

例如：

```text
/model
/session
/tree
/compact
/skill:context-mode
```

Pi 会先判断输入是不是命令。

如果是内置命令或 extension command，可能不会进入 LLM，而是由 Pi 本地处理。

例如：

```text
/session
```

会直接显示 session 信息，不需要模型参与。

---

### 3.3 Prompt template

如果输入匹配 prompt template，Pi 会把模板展开成实际 prompt。

---

### 3.4 Skill

skills 是 progressive disclosure：

- skill 名称和 description 常驻上下文
- 完整 `SKILL.md` 只有需要时才读入

例如用户问 context-mode 相关问题，Pi / agent 会读取对应 skill 文件，然后按里面的规则工作。

---

### 3.5 `@file`

可以输入：

```text
@src/main.ts 帮我 review
```

Pi 会把文件作为输入的一部分传给 agent。

---

### 3.6 `!command` 和 `!!command`

```text
!npm test
```

运行命令，并把输出发给模型。

```text
!!npm test
```

运行命令，但不把输出加入模型上下文。

这对 token 控制很重要。

---

## 4. 输入预处理阶段

用户输入后，Pi 会经过一系列 extension / event hook。

官方生命周期大致是：

```text
user sends prompt
  ↓
extension commands checked first
  ↓
input event
  ↓
skill/template expansion
  ↓
before_agent_start
  ↓
agent_start
```

extensions 可以在这里：

- 拦截输入
- 修改输入
- 直接处理输入
- 注入上下文
- 修改 system prompt
- 阻止某些操作

例如某个 extension 可以实现：

> 如果用户要执行危险命令，先弹确认框。

---

## 5. 写入 session

如果输入被接受，Pi 会把用户消息写入 session JSONL。

session entry 结构类似：

```json
{
  "type": "message",
  "id": "a1b2c3d4",
  "parentId": "prev1234",
  "timestamp": "...",
  "message": {
    "role": "user",
    "content": "帮我修这个 bug"
  }
}
```

Pi 的 session 是树结构，不是纯线性历史：

```text
user A
  └─ assistant B
      ├─ user C route 1
      └─ user D route 2
```

所以 `/tree` 可以从旧节点继续，形成新分支。

---

## 6. 构建 LLM 上下文

这是 Pi agent 链路里非常关键的一步。

Pi 不会简单把整个 session 文件丢给模型，而是从当前 active leaf 构建当前分支上下文。

大概流程：

```text
SessionManager.buildContextEntries()
  ↓
SessionManager.buildSessionContext()
  ↓
Agent state.messages
  ↓
convertToLlm()
  ↓
provider request messages
```

模型看到的通常包括：

```text
system prompt
+ 当前工具 schema
+ skills 描述
+ context files
+ 当前分支消息
+ compaction summary
+ branch summary
+ custom_message
+ tool results
```

但不是所有 session entry 都进模型。

| Entry 类型 | 是否进 LLM 上下文 |
|---|---|
| user message | 是 |
| assistant message | 是 |
| tool result | 是，除非被排除 |
| compaction | 变成摘要进入 |
| branch_summary | 变成摘要进入 |
| custom_message | 是 |
| custom entry | 否，只是 extension 状态 |
| `!!command` output | 通常不进上下文 |

---

## 7. 上下文过长时：compaction

如果上下文太长，Pi 会自动压缩旧内容。

触发条件大致是：

```text
contextTokens > contextWindow - reserveTokens
```

默认配置类似：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

压缩流程：

```text
找到 cut point
  ↓
保留最近 keepRecentTokens
  ↓
总结更早的消息
  ↓
写入 compaction entry
  ↓
重新构建上下文
```

压缩后模型看到的是：

```text
system prompt
+ compaction summary
+ 最近真实消息
```

旧消息仍在 session 文件里，但不再全量进入模型上下文。

---

## 8. 调用模型前的 extension hooks

进入模型前，还有一些 hooks：

```text
turn_start
  ↓
context hook
  ↓
before_provider_headers
  ↓
before_provider_request
```

extensions 可以：

- 修改即将发给模型的 messages
- 注入额外上下文
- 修改 provider headers
- 替换 provider request
- 做审计、日志、策略控制

然后 Pi 通过 `ModelRuntime` 找到 provider，并调用对应 API，例如：

- Anthropic
- OpenAI
- Google
- DeepSeek-compatible OpenAI API
- local llama.cpp
- custom provider

---

## 9. 模型响应阶段

模型开始 streaming 输出。

Pi 会收到事件：

```text
message_start
message_update
message_end
```

`message_update` 里可能包含：

- text delta
- thinking delta
- tool call delta

在 TUI 中看到的 assistant 输出，就是这些 streaming delta 被实时渲染出来。

---

## 10. Tool call 阶段

如果模型决定调用工具，例如：

```json
{
  "name": "read",
  "arguments": {
    "path": "src/main.ts"
  }
}
```

Pi 会进入 tool execution 流程：

```text
tool_execution_start
  ↓
tool_call hook
  ↓
执行工具
  ↓
tool_execution_update
  ↓
tool_result hook
  ↓
tool_execution_end
  ↓
tool result 写入 session
```

工具可以是：

- 内置工具
  - `read`
  - `bash`
  - `edit`
  - `write`
  - `grep`
  - `find`
  - `ls`
- extension 注册的工具
- package 提供的工具
- 当前环境提供的 custom tools，例如：
  - `ctx_execute`
  - `ctx_search`
  - `subagent`

---

## 11. Tool result 回填给模型

工具执行完后，结果会作为 `toolResult` 消息加入上下文。

然后模型会继续下一轮 turn：

```text
LLM response asks for tool
  ↓
tool executes
  ↓
tool result appended
  ↓
LLM sees tool result
  ↓
LLM continues
```

这个循环会重复，直到模型不再调用工具，输出最终文本。

所以一个用户请求内部可能有多轮：

```text
用户：修 bug
  ↓
模型：我要读文件
  ↓
read
  ↓
模型：我要搜索引用
  ↓
grep/find
  ↓
模型：我要改文件
  ↓
edit
  ↓
模型：我要跑测试
  ↓
bash / ctx_execute
  ↓
模型：总结修复结果
```

---

## 12. 最终输出阶段

当模型最终不再调用工具，Pi 会触发：

```text
turn_end
  ↓
agent_end
  ↓
agent_settled
```

然后：

- assistant 最终消息写入 session
- TUI 显示最终文本
- footer 更新 token / cost / cache / context usage
- follow-up queue 如有内容，继续处理下一条

---

## 13. 队列机制：steering / follow-up

Pi 支持 agent 正在工作时继续输入。

### 13.1 Steering message

```text
Enter
```

在 agent 当前 assistant turn 结束工具调用后插入。

适合：

> 等一下，别这样，改用方案 B。

### 13.2 Follow-up message

```text
Alt+Enter
```

等当前任务完全结束后再执行。

适合：

> 做完之后顺便跑一下 lint。

---

## 14. 错误、重试和恢复

如果调用模型或工具失败，Pi 可能：

- 显示错误
- 根据 retry 配置自动重试
- 如果是上下文溢出，触发 compaction 后重试
- 记录错误 message
- 允许用户继续输入修正指令

相关事件包括：

```text
auto_retry_start
auto_retry_end
compaction_start
compaction_end
summarization_retry_*
```

---

## 15. context-mode 在这个流程里的位置

`context-mode` 是 package / extension / tool 层增强。

它主要影响 **工具输出进入上下文的方式**。

不用 context-mode：

```text
bash npm test
  ↓
完整测试输出进入 session/tool result
  ↓
后续上下文膨胀
```

用 context-mode：

```text
ctx_execute npm test
  ↓
沙箱里处理完整输出
  ↓
只把失败摘要返回给模型
  ↓
session 更小，cache 更稳定
```

所以 context-mode 不改变 Pi 的主流程，而是优化：

- 大输出
- 日志
- 测试结果
- API JSON
- 文档检索
- 浏览器快照
- git diff / log

进入上下文的方式。

---

## 16. 一轮完整请求的简化伪代码

可以这样理解：

```ts
async function handleUserInput(input) {
  await loadRuntimeIfNeeded();

  const commandResult = await tryHandleSlashCommand(input);
  if (commandResult.handled) return commandResult.output;

  let prompt = await runInputHooks(input);
  prompt = await expandTemplatesAndSkills(prompt);

  sessionManager.appendMessage({
    role: "user",
    content: prompt,
  });

  await runBeforeAgentStartHooks();

  while (true) {
    let context = sessionManager.buildSessionContext();

    if (shouldCompact(context)) {
      const summary = await compactOldMessages(context);
      sessionManager.appendCompaction(summary);
      context = sessionManager.buildSessionContext();
    }

    context = await runContextHooks(context);

    const assistantMessage = await model.stream({
      systemPrompt,
      messages: context.messages,
      tools,
      thinkingLevel,
    });

    sessionManager.appendMessage(assistantMessage);

    if (!assistantMessage.toolCalls.length) {
      renderFinalAnswer(assistantMessage);
      break;
    }

    for (const toolCall of assistantMessage.toolCalls) {
      const allowed = await runToolCallHooks(toolCall);
      if (!allowed) {
        sessionManager.appendToolResult(blockedResult);
        continue;
      }

      const result = await executeTool(toolCall);
      const modifiedResult = await runToolResultHooks(result);

      sessionManager.appendMessage({
        role: "toolResult",
        content: modifiedResult,
      });
    }
  }

  await runAgentSettledHooks();
}
```

---

## 17. 最核心的几个对象

如果从 SDK 角度看，主要组件是：

| 组件 | 作用 |
|---|---|
| `AgentSession` | 管理一场 agent 会话 |
| `SessionManager` | 保存 / 读取 session JSONL，处理树和分支 |
| `Agent` | 核心 LLM / tool loop |
| `ModelRuntime` | 模型 / provider / 认证管理 |
| `ResourceLoader` | 加载 extensions、skills、prompts、themes、context files |
| `SettingsManager` | 加载和合并 settings |
| Extensions | 拦截事件、注册工具、修改流程 |
| Tools | 文件、shell、编辑、外部能力 |
| Compaction | 上下文过长时总结旧内容 |

---

## 18. 总结

Pi agent 的本质是：

> 一个带持久 session 树、资源加载系统、extension 事件总线、工具调用循环、自动上下文压缩、模型 provider 抽象的 coding agent runtime。

从用户输入到最终输出，核心流程是：

```text
输入
→ 命令/模板/skill/extension 预处理
→ 写入 session
→ 构建当前分支上下文
→ 必要时压缩
→ 调用模型
→ 模型流式输出
→ 如果调用工具，则执行工具并把结果回填模型
→ 循环直到最终回答
→ 写入 session 并显示给用户
```
