# Claude Code 记忆 Prompt 总览

这份文档只做一件事: 把这个项目里所有和“记忆”有关的 prompt 原文整理出来，并说明它们如何协作。

这里的“记忆”分成四类:

1. `CLAUDE.md` 作为长期项目记忆
2. `context` 作为显式键值记忆
3. 会话消息 `messages` 作为短期记忆
4. `/compact` 作为记忆压缩机制

## 1. 记忆流转总图

默认流程是:

1. 启动时读取 `CLAUDE.md` 和上下文。
2. 把这些内容注入 system prompt。
3. 会话过程中不断累积 messages。
4. 当上下文接近上限时，用 `/compact` 生成摘要。
5. 摘要作为新会话起点，旧消息清空。
6. 下一轮会重新读取 `CLAUDE.md` 和上下文。

对应源码:

- [src/constants/prompts.ts](/home/zhan/claude-code-sourcemap/src/constants/prompts.ts#L16)
- [src/utils/style.ts](/home/zhan/claude-code-sourcemap/src/utils/style.ts#L9)
- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L157)
- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L12)
- [src/screens/REPL.tsx](/home/zhan/claude-code-sourcemap/src/screens/REPL.tsx#L203)

## 2. 全量提示词原文

下面按“记忆层级”列出相关 prompt 原文。为了便于对照，我保留了源码里的完整文本块，而不是只摘一句话。

### 2.1 主系统提示词里的 Memory 块

这个块在 `getSystemPrompt()` 里，属于主 system prompt 的一部分。

源码:

- [src/constants/prompts.ts](/home/zhan/claude-code-sourcemap/src/constants/prompts.ts#L16)

原文:

```text
# Memory
If the current working directory contains a file called CLAUDE.md, it will be automatically added to your context. This file serves multiple purposes:
1. Storing frequently used bash commands (build, test, lint, etc.) so you can use them without searching each time
2. Recording the user's code style preferences (naming conventions, preferred libraries, etc.)
3. Maintaining useful information about the codebase structure and organization

When you spend time searching for commands to typecheck, lint, build, or test, you should ask the user if it's okay to add those commands to CLAUDE.md. Similarly, when learning about code style preferences or important codebase information, ask if it's okay to add that to CLAUDE.md so you can remember it for next time.
```

这段是整个记忆设计的核心说明:

- `CLAUDE.md` 会自动进入上下文
- 它保存 build/test/lint 命令
- 它保存代码风格偏好
- 它保存仓库结构知识
- 如果模型花了时间去找这些信息，就应该建议写回 `CLAUDE.md`

### 2.2 `CLAUDE.md` 的风格注入提示词

这个 prompt 不是模型生成的，而是系统在读取 `CLAUDE.md` 时加上的包装语。

源码:

- [src/utils/style.ts](/home/zhan/claude-code-sourcemap/src/utils/style.ts#L6)

原文:

```text
The codebase follows strict style guidelines shown below. All code changes must strictly adhere to these guidelines to maintain consistency and quality.
```

这句的作用是告诉模型: 接下来看到的 `CLAUDE.md` 内容是“必须遵守的风格和规则”，不是普通参考资料。

### 2.3 额外 `CLAUDE.md` 文件的提示词

当工作区里还有其他 `CLAUDE.md` 时，系统会把它们显式注入上下文。

源码:

- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L24)

原文模板:

```text
NOTE: Additional CLAUDE.md files were found. When working in these directories, make sure to read and follow the instructions in the corresponding CLAUDE.md file:
- <absolute-path-1>
- <absolute-path-2>
- <absolute-path-N>
```

说明:

- 这个文本是动态生成的
- 末尾的文件列表由实际扫描结果填充
- 它的作用是把子目录规则也纳入同一个上下文体系

### 2.4 `formatSystemPromptWithContext()` 的包装提示词

当 `context` 里有显式键值时，会被包装成 `<context>` 块注入。

源码:

- [src/services/claude.ts](/home/zhan/claude-code-sourcemap/src/services/claude.ts#L426)

原文:

```text
As you answer the user's questions, you can use the following context:
```

随后会追加这样的结构:

```text
<context name="key">value</context>
```

这层不是专门针对 `CLAUDE.md`，但它是同一套记忆系统的一部分，因为它会把用户手动设置的上下文写进每轮对话。

### 2.5 `/init` 的完整 prompt

这个命令负责生成或改善 `CLAUDE.md`。

源码:

- [src/commands/init.ts](/home/zhan/claude-code-sourcemap/src/commands/init.ts#L14)

原文:

```text
Please analyze this codebase and create a CLAUDE.md file containing:
1. Build/lint/test commands - especially for running a single test
2. Code style guidelines including imports, formatting, types, naming conventions, error handling, etc.

The file you create will be given to agentic coding agents (such as yourself) that operate in this repository. Make it about 20 lines long.
If there's already a CLAUDE.md, improve it.
If there are Cursor rules (in .cursor/rules/ or .cursorrules) or Copilot rules (in .github/copilot-instructions.md), make sure to include them.
```

这段 prompt 的重点:

- 让模型生成一个适合长期记忆的文件
- 重点写命令和风格
- 合并已有规则来源
- 让 `CLAUDE.md` 成为后续 agent 的可读知识库

### 2.6 `/compact` 的完整 prompt

这个命令负责把现有会话压缩成摘要。

源码:

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L18)

原文里的摘要请求:

```text
Provide a detailed but concise summary of our conversation above. Focus on information that would be helpful for continuing the conversation, including what we did, what we're doing, which files we're working on, and what we're going to do next.
```

原文里的摘要 system prompt:

```text
You are a helpful AI assistant tasked with summarizing conversations.
```

原文里的压缩后桥接消息:

```text
Use the /compact command to clear the conversation history, and start a new conversation with the summary in context.
```

这三段合在一起，构成了压缩动作的完整语义:

- 先总结
- 再把摘要作为新会话起点
- 再告诉后续会话如何继续

### 2.7 onboarding 中的记忆引导

这不是模型 prompt，而是 UI 引导文本，但它直接影响用户会不会建立 `CLAUDE.md`。

源码:

- [src/ProjectOnboarding.tsx](/home/zhan/claude-code-sourcemap/src/ProjectOnboarding.tsx#L73)

原文:

```text
Run /init to create a CLAUDE.md file with instructions for Claude.
```

### 2.8 `context` CLI 的描述文本

这组命令让用户显式写入/读取/删除上下文，也就是另一层记忆。

源码:

- [src/entrypoints/cli.tsx](/home/zhan/claude-code-sourcemap/src/entrypoints/cli.tsx#L953)

原文:

```text
Set static context (eg. claude context add-file ./src/*.py)
Get a value from context
Set a value in context
List all context values
Remove a value from context
```

这组命令不是自动学习，而是用户手工把知识写进项目状态。

## 3. 它们怎么配合

### 3.1 `CLAUDE.md` 负责长期知识

它保存的是:

- 构建、测试、lint 命令
- 代码风格规范
- 仓库结构和组织信息

它会被 `getCodeStyle()` 和 `getClaudeFiles()` 在会话开始时重新读入。

源码:

- [src/utils/style.ts](/home/zhan/claude-code-sourcemap/src/utils/style.ts#L9)
- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L24)

### 3.2 `context` 负责用户显式知识

它保存的是用户通过命令手工写进去的键值信息，适合放少量、明确、稳定的事实。

### 3.3 `messages` 负责短期对话

它保存当前会话中的实际消息流，随着模型调用不断增长。

源码:

- [src/screens/REPL.tsx](/home/zhan/claude-code-sourcemap/src/screens/REPL.tsx#L127)

### 3.4 `/compact` 负责压缩短期对话

它不会修改 `CLAUDE.md` 的内容，而是把当前对话压缩成摘要，让后续会话继续使用。

源码:

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L34)

### 3.5 下一轮会重新加载长期记忆

压缩以后会清掉 `getContext.cache` 和 `getCodeStyle.cache`，这样新的会话会重新读取最新的 `CLAUDE.md` 和上下文。

源码:

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L75)

## 4. 压缩到底做了什么

`/compact` 的实现可以理解成:

1. 取出当前所有 messages
2. 添加一条“总结对话”的用户消息
3. 用单独的摘要 system prompt 调用模型
4. 取回摘要文本
5. 清空旧 messages
6. 把“摘要 + 继续指令”作为新会话起点
7. 重新让后续对话从压缩后的状态继续

源码:

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L26)
- [src/screens/REPL.tsx](/home/zhan/claude-code-sourcemap/src/screens/REPL.tsx#L264)

## 5. 为什么这样设计

这个设计的目的不是“让模型自己记住一切”，而是把记忆拆开:

- 可版本控制的长期记忆: `CLAUDE.md`
- 用户显式设置的上下文: `context`
- 临时对话: `messages`
- 压缩后的摘要: `/compact`

这样做的好处是:

1. 长期知识可审计、可修改、可提交到 Git。
2. 临时对话不会无限膨胀。
3. 压缩后的摘要仍然能继续工作。
4. 新的 `CLAUDE.md` 修改会在下一轮自动生效。

## 6. 一句话总结

这个项目的记忆机制不是单点黑盒，而是:

`CLAUDE.md` 负责长期知识，`context` 负责显式知识，`messages` 负责短期对话，`/compact` 负责把短期对话压缩成可继续使用的摘要。

