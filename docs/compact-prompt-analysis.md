# Claude Code 压缩 Prompt 研究

这份文档只讲一件事：`/compact` 是怎么把现有信息压缩成一个新会话起点的。

结论先说：

1. 它不是模型内部自动压缩。
2. 它是一次专门的“总结对话”调用。
3. 总结结果会被放回消息流，原始长历史被清掉。
4. 之后新会话继续跑，新的上下文和 `CLAUDE.md` 会重新读取。

## 1. 压缩入口

压缩入口是 `/compact` 命令。

对应源码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L12)

这个命令的说明文本表达的是：清掉完整对话，只保留摘要继续用。

也就是说，它不会保留完整对话，而是把“有用的摘要”留下来。

## 2. 压缩时用的 prompt

`/compact` 里有两段最关键的 prompt。

### 2.1 摘要请求 prompt

这是发给模型的用户消息，用来要求它压缩对话。

代码位置：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L29)

这段 prompt 的实际意图不是“复述全部内容”，而是让模型提炼出继续工作需要的最小充分信息。

### 2.2 摘要专用 system prompt

`/compact` 不是用主对话的 system prompt，而是单独给了一个极短的摘要角色说明：让模型专注做总结。

代码位置：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L34)

这个 system prompt 很重要，因为它把模型的任务空间收窄了:

- 不是继续做代码任务
- 不是继续调用工具
- 不是回答用户新问题
- 而是只做总结

## 3. 它具体怎么压缩

压缩流程在代码里是固定的。

### 3.1 取出当前全部消息

先从内存里拿当前对话：

- `const messages = getMessagesGetter()()`

这意味着压缩的对象是“当前会话完整消息”，不是某个局部片段。

### 3.2 拼一个总结请求

然后加上一条总结请求消息：

- “详细但简洁地总结”
- “重点说明做了什么”
- “现在正在做什么”
- “涉及哪些文件”
- “接下来要做什么”

这部分 prompt 的作用是把总结目标限制在“继续工作所需信息”上，而不是让模型泛泛而谈。

### 3.3 调用 `querySonnet`

压缩是通过 `querySonnet(...)` 完成的：

- messages：原始消息 + 总结请求
- systemPrompt：摘要专用 system prompt
- maxThinkingTokens：`0`
- tools：仍然传入，但这一步的目标不是工具执行

对应代码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L34)

`maxThinkingTokens = 0` 的含义很明确:

- 不鼓励长思考
- 不走复杂推理轨道
- 目标是快速生成可压缩摘要

### 3.4 解析摘要文本

模型返回后，代码只接受文本内容。

如果没有文本，直接报错。
如果返回的是 API error，也直接报错。

对应代码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L47)

这说明压缩阶段只允许“可读摘要”，不接受结构化工具输出。

## 4. 压缩后保留了什么

`/compact` 不只是“生成摘要”，它还会重建会话状态。

保留的内容：

1. 一条新的 user message，提示“清历史但保留摘要”。
2. 模型生成的 summary response。
3. 一个新的 fork 会话起点。

对应代码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L75)

这条桥接消息的作用是给新会话一个明确的语义：

- 上一段历史已经结束
- 现在只继承摘要
- 继续往下做

## 5. 压缩后丢掉了什么

`/compact` 会主动丢掉旧状态：

1. 旧的 messages 被清空
2. 旧的屏幕被清空
3. `getContext.cache` 被清空
4. `getCodeStyle.cache` 被清空

对应代码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L75)

这里很关键：

- 清空 `messages` 是为了释放上下文窗口
- 清空 `getContext.cache` 是为了让下一轮重新读取项目上下文
- 清空 `getCodeStyle.cache` 是为了让下一轮重新读 `CLAUDE.md`

也就是说，`compact` 不只是压缩“对话”，它同时刷新“项目记忆”。

## 6. `CLAUDE.md` 在压缩里的位置

`CLAUDE.md` 不是被 compact 自己总结出来的，而是在下一轮会话里重新进入上下文。

它的读取机制在：

- [src/utils/style.ts](/home/zhan/claude-code-sourcemap/src/utils/style.ts#L9)
- [src/context.ts](/home/zhan/claude-code-sourcemap/src/context.ts#L157)

摘要后，新会话再次启动时会重新拼接：

1. `CLAUDE.md` 内容
2. git 状态
3. 目录结构
4. README
5. 用户显式 context

所以这套系统的真实逻辑是：

- `compact` 负责压缩对话历史
- `CLAUDE.md` 负责提供持久项目记忆
- 两者一起组成“压缩后可继续”的上下文

## 7. 额外的实现细节

### 7.1 token 计数被“重写”

压缩后，代码会把 summary response 的 `usage.input_tokens` 设成 `0`。

对应代码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L64)

这不是模型压缩本身，而是 UI / token 统计层面的处理。

作用是：

- 让上下文大小警告尽快消失
- 让 summary 作为“新会话起点”更自然

### 7.2 为什么还要 `clearTerminal()`

它会先清空终端，再重置消息流。

这意味着用户视觉上看到的是“旧会话结束，新会话开始”，而不是“同一会话里插入一个摘要块”。

对应代码：

- [src/commands/compact.ts](/home/zhan/claude-code-sourcemap/src/commands/compact.ts#L75)

## 8. 压缩流程图

可以把它理解成下面这条链：

1. 读取当前 `messages`
2. 追加一条总结请求
3. 用摘要 system prompt 调 `querySonnet`
4. 拿到 summary response
5. 清空旧消息和缓存
6. 插入“summary + 继续指令”
7. 新会话继续

## 9. 设计原因

这种设计有三个直接好处：

1. 对话历史不会无限膨胀。
2. 摘要是可控的，模型每次都知道要保留什么。
3. `CLAUDE.md` 和 `context` 这类长期记忆不会被 compact 误删，因为它们会在下一轮重新读取。

换句话说，这不是“把信息忘掉”，而是“把信息重写成更短的形式，然后重新加载”。
