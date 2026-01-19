---
layout: post
title: 初探 Coding Agent：OpenCode
tags:
  - AI
---

写代码 + AI，会是什么形式？
- 在 ChatGPT/DeepSeek/Kimi 网页上提问，然后 copy 对话框里的代码去用？一次成功，或者手动改，或者把报错输到对话框让 AI 给新的代码？
- 在 IDE 里智能补全，几个字符，一片连续多行，或者不连续的多片多行？
- 在 IDE 侧边栏对话框（或命令行对话框）里描述一个任务要求，让 AI 自己完成开发、测试，直接交付可用的代码？

上面三种形态，就是过去两年里我经历的变化，不久前我用第三种形态（Trae + GPT 5.2 模型），两天狂写了上万行移动端的 C++ 单元测试代码。

而这种形态，就是 Coding Agent，基于 LLM 打造的 Agent，能独立完成编程任务，交付可用的代码。今天我们就来分析一个开源的 Coding Agent：[OpenCode](https://opencode.ai/)，对标 Claude Code，但开源且可以使用任意模型。

我们先回顾几个基本的东西，最后再分析 OpenCode 的核心流程。

## Coding Agent 本质

- LLM 的本质：一个获取给定字符串的下一个单词的函数。
- 持续获取下一个单词，直到遇到结束符，就完成了任务：续写。
- 对 LLM 进行针对性的训练（SFT 或 RL），以及构造好给 LLM 的初始输入，就能让这个续写看起来是回答问题。
- 通过一些工具调用，让 LLM 能阅读、修改、调试代码仓库，加上针对性的训练，也能让上面的过程看起来是解决问题。

## LLM 应用开发基本套路

LLM 应用开发，就是调用 LLM 的 sdk，最具有代表性（也是很多家都兼容的）就是 OpenAI 的了，下面是一个简单的例子：

```javascript
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
// 你自己维护一份“消息历史”，每次请求都把它完整带上（服务端本身不替你记忆）
const messages = [
  { role: "system", content: "你是一个简洁的中文助手，回答不超过80字。" },
  { role: "user", content: "我在深圳，周末想去爬山，有什么建议？" },
  { role: "assistant", content: "你更偏好轻松路线还是强度大一些？是否需要地铁可达？" },
  { role: "user", content: "轻松点，地铁可达，顺便推荐带什么。" },
];
const resp = await client.responses.create({
  model: "gpt-4o-mini",
  // Responses API 支持把对话按 role/content 结构作为 input 传入
  input: messages,
});
console.log(resp.output_text);
// 如果要继续多轮：把本次 assistant 输出 append 回 messages 再发下一次
messages.push({ role: "assistant", content: resp.output_text });
```

- 其实就是给 LLM 输入，拿到输出。
- 把历史消息都作为输入，就能实现多轮对话。
- LLM 不输出文本，而是 tool-call 信息，那我们的代码就去调用相应的工具函数，然后把结果添加到消息列表里，再调用一次 LLM，就实现了工具调用的支持。

## Vercel AI SDK

我们也可以使用更强大的 [Vercel AI SDK](https://ai-sdk.dev/)（OpenCode 就用了它）：

- **统一的模型调用方式**：文本生成、结构化输出、对话、多 provider 适配（OpenAI/Anthropic/Google/Azure 等，视你接入的 provider 包而定）。
- **Streaming-first（流式优先）**：把模型输出以流（`ReadableStream`）形式提供，改善首 token 延迟与交互体验。
- **Tools / Function calling**：把外部能力（检索、数据库、业务 API）以工具方式暴露给模型，支持“模型决定调用—服务端执行—结果回填—继续生成”的闭环。
- **更好的 DX**：前端（如 React/Next.js）侧提供常用 hooks（例如 `useChat`），减少处理 SSE/流拼接/状态机的样板代码。

AI SDK 实现了工具调用框架，我们只需要把工具注册进去就行，由框架自动实现 LLM tool-call 消息解析、工具调用、tool-result 回填给模型，大大简化了我们的工作。

```javascript
export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    messages,
    tools: { // 这里注册工具
      searchDocs: tool({
        description: "在内部文档中检索",
        parameters: ...,
        execute: async ({ query }) => { // 工具函数体，会被框架调用
          return [{ title: "xxx", snippet: "..." }];
        },
      }),
    },
  });

  return result.toDataStreamResponse();
}
```

AI SDK 的核心数据结构是 Message 和 Part。

### 1) 为什么要有 **message / part** 两层
Vercel AI SDK 把“对话”抽象成一个 **message 列表**（按轮次组织），但一条 message 的内容不再只是一段字符串，而是由多个 **parts** 组成（按片段/事件组织）。这样做主要为了解决：

- **流式输出**：一条 assistant 回复会不断增量到达（token-by-token），自然对应“多个 part 逐步追加”。
- **工具调用**：同一条 assistant message 里既可能有自然语言，也可能有 tool call、tool result 等“非文本片段”。
- **多模态/富内容**：图片、文件、结构化数据、引用等都可以是独立 part，而不是硬塞进字符串里。

可以把它理解为：  
- `message` = 对话中的一条“发言”（有角色/时间/ID/元信息）  
- `part` = 这条发言里的一个“内容片段/事件”（文本、工具调用、工具结果、图片等）

### 2) `message` 的设计（UI 层 vs Model 层）
AI SDK 通常会区分两类 message：

#### 2.1 UI/客户端用的 Message（更“富”）
用于 `useChat()` 之类的前端状态管理与渲染。常见字段形态（概念化）：

```ts
type UIMessage = {
  id: string;
  role: "user" | "assistant" | "system" | "tool";
  // 兼容老用法：把纯文本拼在 content 里（通常由 parts 派生/汇总）
  content?: string;
  // 新的富内容表达：一条消息由多个 parts 构成
  parts?: Array<Part>;
  createdAt?: Date;

  // 允许挂额外元数据（实现里可能叫 data/annotations/metadata 等）
  experimental_attachments?: unknown;
};
```

要点：
- **UIMessage 往往同时保留 `content` 与 `parts`**：  
  - `content` 便于简单渲染/兼容旧代码  
  - `parts` 承载真实语义（工具调用、结构化数据等）

#### 2.2 传给模型的 Message（更“规范/可序列化”）
服务端调用模型时常用的是更接近“LLM 消息协议”的结构（概念化）：

```ts
type ModelMessage =
  | { role: "system"; content: Array<Part> | string }
  | { role: "user"; content: Array<Part> | string }
  | { role: "assistant"; content: Array<Part> | string }
  | { role: "tool"; content: Array<Part> | string; toolCallId: string };
```

要点：
- 模型层的 message 更关注 **可被 provider 转换**（OpenAI/Anthropic/Google 等格式不一，SDK 做统一）。
- `tool` 角色通常用于把 **工具执行结果**回填给模型（并用 `toolCallId` 对应某次调用）。

### 3) `part` 的设计（一个可扩展的“内容片段”联合类型）
`part` 通常是一个 **带 `type` 判别字段的 union**（discriminated union），便于扩展与类型安全。下面是最典型的几类（概念化，字段名会随版本略有差异，但语义一致）：

#### 3.1 文本类
```ts
type TextPart = {
  type: "text";
  text: string;
};
```

用途：
- 普通自然语言输出/输入
- 流式时会不断追加新的 `TextPart` 或对最后一个 `TextPart` 进行增量拼接（具体实现策略可能不同）

#### 3.2 工具调用（tool invocation / tool call）
```ts
type ToolCallPart = {
  type: "tool-call";
  toolCallId: string;
  toolName: string;
  args: unknown; // 通常是 JSON object，且会被 schema 校验（如 zod）
};
```

用途：
- assistant 决定要调用某个 tool 时，先产出一个 tool-call part（可能也是流式逐步补全 args）

#### 3.3 工具结果（tool result）
```ts
type ToolResultPart = {
  type: "tool-result";
  toolCallId: string;
  toolName: string;
  result: unknown; // 工具执行的返回值（可为 JSON）
  isError?: boolean;
};
```

用途：
- 服务端执行 tool 后，把结果作为 part 回填到对话中
- 模型可基于 result 继续生成最终回答

> **toolCallId 是关键**：用它把 “tool-call” 与 “tool-result” 配对，支持并发/多次调用。

#### 3.4 数据/自定义片段（data / annotations）
```ts
type DataPart = {
  type: "data";
  data: unknown; // 例如检索命中的文档列表、引用、UI 指令等
};
```

用途：
- 不想让模型“读到”的 UI 元信息（或想单独渲染的结构化信息）
- 与 RAG 检索结果、引用标注、调试信息等配合

#### 3.5 多模态（例如图片）
不同 provider 支持不同，SDK 用 part 统一表达（概念化）：
```ts
type ImagePart = {
  type: "image";
  image: string; // 可能是 URL / base64 data URL 等
  mimeType?: string;
};
```

### 4) 一个完整例子：同一条 assistant message 里既有文本又有工具调用
```ts
const messages: UIMessage[] = [
  {
    id: "u1",
    role: "user",
    parts: [{ type: "text", text: "帮我查一下今天深圳天气，并给出穿衣建议" }],
  },
  {
    id: "a1",
    role: "assistant",
    parts: [
      { type: "text", text: "我先查询一下深圳今日天气。" },
      {
        type: "tool-call",
        toolCallId: "tc_123",
        toolName: "getWeather",
        args: { city: "深圳", date: "2026-01-17" },
      },
    ],
  },
  {
    id: "t1",
    role: "tool",
    parts: [
      {
        type: "tool-result",
        toolCallId: "tc_123",
        toolName: "getWeather",
        result: { temp: 18, condition: "阴", wind: "3级" },
      },
    ],
  },
  {
    id: "a2",
    role: "assistant",
    parts: [
      { type: "text", text: "今天深圳约 18°C、阴天、风力 3 级。建议穿薄外套…" },
    ],
  },
];
```

你可以看到：
- **工具调用与结果不需要塞进纯文本**，结构清晰、渲染也更容易。
- UI 可以对不同 part 做不同展示（例如 tool-call 显示“正在查询…”）。

## Coding Agent 开发基本套路

Coding Agent 的实现，最核心的就是 ReAct（Reasoning and Acting）循环：让 Agent 在解决复杂任务时，把“推理”和“调用外部工具/环境执行”交替进行的范式。其核心是一个不断迭代的 **`Thought → Action → Observation → (repeat)`** 循环，直到完成任务或触发停止条件。

### 循环的三步：`Thought / Action / Observation`

1. **Thought（思考/推理）**  
   Agent 基于用户目标与当前上下文，判断“下一步需要什么信息/要做什么操作”，并形成计划性的推理步骤。

2. **Action（行动/工具调用）**  
   Agent 按约定格式发出一次“动作”，通常是 **调用工具/函数**（如搜索、代码执行、API、计算等），把任务推进到可验证的外部世界。

3. **Observation（观察/结果回灌）**  
   运行时环境执行该动作，把输出（例如：检索结果、编译错误、测试失败日志、API 返回等）作为 **Observation** 回传给 Agent；Agent 再把它纳入上下文，决定是否需要调整策略并进入下一轮 Thought。

### 在工程实现上：本质是一个“追加历史的 while 循环”

很多手写/框架实现里，ReAct Loop 在代码层面就是：**读取历史 → 让模型产出下一步 → 执行工具 → 把结果追加回历史**，循环往复。  
关键状态通常是一个不断增长的 `messages`（或 trace）列表：Agent 视角里“世界”基本就体现在这份历史里。

### coding agent 场景下，Observation 往往是什么

在“写代码/改代码/修 bug”的任务里，**Observation** 常见是：

- 代码执行输出、报错栈、lint 结果、编译失败信息、单测失败用例等（都属于“环境或工具返回的结果/反馈”）。
- Agent 会根据这些反馈回到 Thought，决定下一步是“改实现/补依赖/调整调用方式/缩小排查范围/换工具或策略”等。

### 何时停止循环（常见停止条件）

- **任务目标已达成**：Agent 判断已获得足够信息并可给出最终答案/最终代码。
- **达到最大步数/迭代上限**：为避免无限循环，通常设置最大步骤数，到达后强制结束。

## OpenCode 核心流程

最后，我们来看一下 OpenCode 这个具体的 Coding Agent 的 ReAct 循环实现。

1) **内层：单次 LLM 调用的流式 ReAct 执行器**（消费流事件并落库）
- 位置：`packages/opencode/src/session/processor.ts:45` 的 `SessionProcessor.create(...).process()`
- 作用：消费 `LLM.stream()` 的 `fullStream` 事件流；把 text/reasoning/tool-call/tool-result 等事件写入 session parts；处理重试、权限拒绝、context overflow 等情况，最终返回 `continue/stop/compact`。

2) **外层：按 step 推进的 session loop**（调度任务 + 决定是否继续下一轮）
- 位置：`packages/opencode/src/session/prompt.ts:257` 的 `SessionPrompt.loop()`
- 作用：从 session 历史里取最新 user/assistant 状态，优先处理待办任务（`subtask/compaction`），否则启动一轮内层 “LLM + tools”；根据返回值 `continue/stop/compact` 决定下一步。

### 内层 process（`SessionProcessor.process`）核心流程

位置：`packages/opencode/src/session/processor.ts:45-400`

- 调用 `LLM.stream(streamInput)` 获取流式输出（`processor.ts:53`，LLM 实现在 `packages/opencode/src/session/llm.ts:46`）。
- `for await` 消费事件并落库：
  - `text-*`：构建并增量写入最终回答（`processor.ts:279-326`）。
  - `reasoning-*`：构建并增量写入推理片段（`processor.ts:62-101`）。
  - `tool-*`：维护 tool part 状态机：pending → running → completed/error（`processor.ts:103-221`）。
  - `start-step/finish-step`：tracking snapshot/patch、写 usage/cost、触发 summary、并检测 overflow（`processor.ts:225-277`）。
- 防止重复调用同一工具形成死循环：检测最近 3 个 tool part 是否“同 tool + 同 input”，触发 `doom_loop` 权限询问（`processor.ts:143-168`）。
- 异常与重试：可重试错误会更新 `SessionStatus` 为 retry 并退避重试；不可重试则写入 message error 并发布事件（`processor.ts:339-363`）。
- 统一收尾：把未完成的 tool part 标为 aborted，写 assistant 完成时间；最终返回：`compact`（需压缩）/`stop`（被拒绝或错误）/`continue`（进入下一轮）（`processor.ts:378-400`）。

### 外层 loop（`SessionPrompt.loop`）核心流程

位置：`packages/opencode/src/session/prompt.ts:257-632`

- **并发/重入**：如果同一个 `sessionID` 已在跑，会把当前调用挂到回调队列，等待正在跑的 loop 产出下一条 assistant 消息后再返回（`prompt.ts:258-264`）。
- **每轮 step**：读取消息历史（`MessageV2.stream` + `filterCompacted`），反向扫描拿到 `lastUser / lastAssistant / lastFinished`，并收集 `compaction/subtask` 任务 parts（`prompt.ts:274-290`）。
- **退出条件**：当最新 assistant 已以非 `tool-calls/unknown` 的原因完成且领先于最新 user 时，直接退出（`prompt.ts:294-301`）。
- **优先处理任务**：
  - `subtask`：手动执行 `TaskTool`，将结果写成 tool part，并插入一条 synthetic user message 引导下一轮继续（`prompt.ts:315-479`）。
  - `compaction`：调用 `SessionCompaction.process(...)`（`prompt.ts:481-492`）。
  - 上下文溢出：检测 `lastFinished.tokens`，必要时创建 compaction 任务（`prompt.ts:494-507`）。
- **正常处理（进入内层 ReAct）**：
  - 创建本轮 assistant message + `SessionProcessor`（`prompt.ts:518-546`）。
  - `resolveTools(...)` 把 `ToolRegistry`/`MCP` 工具包装成 LLM 可调用的 tools（`prompt.ts:552-559`, `prompt.ts:641+`）。
  - 对消息做插件 transform + “stay on track” 包裹（`prompt.ts:568-590`）。
  - 调用 `processor.process(...)`，拿到 `stop/compact/continue` 后决定 break / 创建 compaction / 进入下一轮（`prompt.ts:591-620`）。

### 外层 loop 里 `compaction/subtask` 任务从哪来

外层 loop 反向扫描历史时，把 message 的 parts 里 `type === "compaction" || "subtask"` 当作“待办任务”收集（`prompt.ts:287-290`）。它们的来源分别是：

- `subtask`：通常由 `SessionPrompt.command()` 在执行命令模板时生成：当目标 agent 为 `subagent`（或 command 强制 `subtask=true`）时，会把这次 command 写成一条 **user message 的 `subtask` part**，并通过 `SessionPrompt.prompt()` 入库（`prompt.ts:1562-1584`）。
- `compaction`：由 `SessionCompaction.create()` 生成：新建一条 **user message**，并写入一个 `compaction` part（`session/compaction.ts:195-223`）。触发点包括：
  - 外层 loop 检测到 context overflow（`prompt.ts:494-507`）。
  - 内层 `processor.process()` 返回 `"compact"`（`prompt.ts:612-619`）。
  - 手动 summarize API：`POST /session/:sessionID/summarize`（`server.ts:1163-1219`）。

### 为什么 compaction 要先写入 `compaction` part，再在下一轮 `loop` 里 `process`

这不是“排队而已”，而是把 compaction 变成一个**可持久化的待办任务**，并提供“压缩边界锚点”：

- `MessageV2.filterCompacted()` 会在历史里寻找“已经完成的 compaction 锚点”，从而截断压缩前的长历史：它需要看到一条 **user message**，且该 message 的 parts 里含 `type: "compaction"`，并且存在对应的 `assistant summary + finish`（`message-v2.ts:587-602`）。
- `SessionCompaction.create()` 创建的正是这条锚点消息（`session/compaction.ts:195-223`）；随后外层 loop 在下一轮把它当作 `lastUser`，再调用 `SessionCompaction.process({ parentID: lastUser.id, ... })`（`prompt.ts:481-492`），从而让 compaction 产出的 summary assistant message 的 `parentID` 指回这条锚点 user message（`session/compaction.ts:104-112`），满足 `filterCompacted` 的配对逻辑。
- 这种“任务入库 → loop 消费”的模式也和 `subtask` 一致，带来可恢复性：即使中途 abort/重启，只要任务 part 已经写入 session 历史，后续再次 `SessionPrompt.loop()` 仍能捞到并继续执行。

### 工具执行链路（Action/Observation 回灌点）

- 工具集来源：`ToolRegistry.tools(...)` + `MCP.tools()`（在 `prompt.ts:686-814` 组装）。
- 执行时机：模型输出 `tool-call` 事件后，由 AI SDK 调用对应 tool 的 `execute(...)`；tool 的结果以 `tool-result` 事件回到 `SessionProcessor.process` 并写入 session parts。
- 统一钩子：每次工具执行前后触发 `Plugin.trigger("tool.execute.before/after", ...)`，便于扩展（`prompt.ts:694-714`）。
