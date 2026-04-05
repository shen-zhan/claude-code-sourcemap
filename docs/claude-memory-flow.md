# Claude Code 记忆流程说明

> 说明：下面收录的是和“记忆管理”最相关的原文片段，以及它们在代码里的位置。
> 由于提示词本身很长，这里保留的是关键原文摘录和精确源码引用；完整上下文请直接查看对应源码文件。

## 1. 记忆是怎么分层的

这个项目没有把“记忆”做成一个单独的模型内存模块，而是拆成了几层：

1. `CLAUDE.md` 作为长期、可版本控制的项目记忆。
2. `ProjectConfig.context` 作为用户显式写入的键值记忆。
3. `history` 作为命令历史记忆。
4. `messages` 作为当前会话短期记忆。
5. `/compact` 把长对话压缩成摘要后继续。

对应代码：
- [src/utils/style.ts](/home/zhan/claude-code-sourcemap/src/utils/style.ts#L9)
- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L157)
- [src/utils/config.ts](/home/zhan/claude-code-sourcemap/src/utils/config.ts#L28)
- [src/history.ts](/home/zhan/claude-code-sourcemap/src/history.ts#L8)
- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L12)

## 2. `CLAUDE.md` 的默认工作流

### 2.1 启动时自动读取

`getCodeStyle()` 会沿着当前目录一路向上找 `CLAUDE.md`，把每一份都读出来，拼成一个“代码风格 + 项目规则”的上下文块。

原文片段：

> `The codebase follows strict style guidelines shown below. All code changes must strictly adhere to these guidelines to maintain consistency and quality.`

源码：
- [src/utils/style.ts](/home/zhan/claude-code-sourcemap/src/utils/style.ts#L6)

### 2.2 额外的 `CLAUDE.md`

`getClaudeFiles()` 会递归搜索工作区里的额外 `CLAUDE.md`，并告诉模型这些文件在对应目录下也要遵守。

原文片段：

> `If the current working directory contains a file called CLAUDE.md, it will be automatically added to your context.`

源码：
- [src/constants/prompts.ts](/home/zhan/claude-code-sourcemap/src/constants/prompts.ts#L29)
- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L24)

### 2.3 写入入口：`/init`

`/init` 的任务不是“随便总结”，而是生成一个适合作为长期记忆的 `CLAUDE.md`，重点写 build/lint/test 命令和代码风格规范。

原文片段：

> `Please analyze this codebase and create a CLAUDE.md file containing:`

其余要求在源码里继续列出，重点是 build/lint/test 命令和代码风格规范。

源码：
- [src/commands/init.ts](/home/zhan/claude-code-sourcemap/src/commands/init.ts#L14)

### 2.4 onboarding 里的引导

项目首次进入时，UI 会提示如果仓库没有 `CLAUDE.md`，就运行 `/init`。

原文片段：

> `Run /init to create a CLAUDE.md file with instructions for Claude.`

源码：
- [src/ProjectOnboarding.tsx](/home/zhan/claude-code-sourcemap/src/ProjectOnboarding.tsx#L102)

## 3. `CLAUDE.md` 是怎么进入 prompt 的

`getContext()` 会把多个来源拼成一个对象，最后交给 `formatSystemPromptWithContext()`。

流程是：

1. `getCodeStyle()` 读取当前目录链上的 `CLAUDE.md`。
2. `getClaudeFiles()` 搜索工作区里额外的 `CLAUDE.md`。
3. `getContext()` 把这些内容合并成 `context`。
4. `formatSystemPromptWithContext()` 把 `context` 包装成 `<context name="...">...</context>`。
5. `query()` 把这组 system prompt 送给模型。

对应代码：
- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L157)
- [src/services/claude.ts](/home/zhan/claude-code-sourcemap/src/services/claude.ts#L426)
- [src/query.ts](/home/zhan/claude-code-sourcemap/src/query.ts#L124)

原文片段：

> `As you answer the user's questions, you can use the following context:`

> `<context name="...">...</context>`

源码：
- [src/services/claude.ts](/home/zhan/claude-code-sourcemap/src/services/claude.ts#L436)

## 4. 其他记忆模块

### 4.1 `context set/get/remove`

这是显式的结构化记忆，不是自动抽取的文本。

原文片段：

> `Set a value in context`
>
> `Get a value from context`
>
> `Remove a value from context`

源码：
- [src/entrypoints/cli.tsx](/home/zhan/claude-code-sourcemap/src/entrypoints/cli.tsx#L953)

### 4.2 `history`

命令历史只记录最近 100 条，用于输入体验，不进入模型 prompt。

源码：
- [src/history.ts](/home/zhan/claude-code-sourcemap/src/history.ts#L6)

### 4.3 会话消息

当前对话存在 `messages` 里，属于短期记忆。它会被保存在 transcript/log 中，也可以恢复。

源码：
- [src/screens/REPL.tsx](/home/zhan/claude-code-sourcemap/src/screens/REPL.tsx#L127)
- [src/screens/ResumeConversation.tsx](/home/zhan/claude-code-sourcemap/src/screens/ResumeConversation.tsx#L1)

## 5. 记忆是怎么压缩的

### 5.1 `/compact`

`/compact` 会把当前所有消息拿出来，让模型先生成一段摘要，再清空旧消息，把“摘要 + 说明”作为新会话起点。

原文片段：

> `Clear conversation history but keep a summary in context`

> `Provide a detailed but concise summary of our conversation above.`

源码：
- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L12)

### 5.2 压缩后保留什么

压缩后会保留：

1. 一个新 user message，提示“用 compact 清理历史并带着摘要继续”。
2. 模型生成的 summary response。
3. 新的 messages 列表。

同时会清掉：

1. `getContext.cache`
2. `getCodeStyle.cache`

这样新一轮会重新读取最新 `CLAUDE.md` 和上下文。

源码：
- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L75)

### 5.3 `/clear`

`/clear` 比 `/compact` 更彻底，直接把会话消息、上下文缓存和 cwd 重置。

原文片段：

> `Clear conversation history and free up context`

源码：
- [src/commands/clear.ts](/home/zhan/claude-code-sourcemap/src/commands/clear.ts#L22)

## 6. 这个设计为什么合理

1. `CLAUDE.md` 是文件级、可审计、可提交的长期记忆。
2. `context` 是用户显式写入的项目事实，适合少量强结构信息。
3. `compact` 解决上下文窗口有限的问题。
4. `clear` 解决“我要完全重新开始”的需求。
5. 这些内容都以文本和配置文件为载体，便于团队协作和调试。

## 7. 一句话流程图

`/init` 生成 `CLAUDE.md` -> 启动时自动读入 `CLAUDE.md` 和项目上下文 -> 模型运行时不断累积消息 -> `/compact` 把对话压成摘要 -> `/clear` 彻底清空并重来。
