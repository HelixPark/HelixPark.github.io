---
title: OpenClaw 核心运行流程深度解析
date: 2026-03-23 18:00:00
categories:
  - 技术
tags:
  - OpenClaw
  - Agent
---

> 本文基于知乎文章《OpenClaw核心运行流程》整理扩写，保留原文核心内容，并在细节处补充举例说明，帮助大家更直观地理解 OpenClaw 的设计思想。

OpenClaw 的爆火激发了无数开发者的学习热情。本文将围绕一张核心运行流程图，完整介绍 OpenClaw 的底层设计，让大家感受到 Agent 设计的魅力。

![OpenClaw核心运行流程图](/images/openclaw/01-core-flow.jpg)

---

## 1. 概述

OpenClaw 是一个通过 **Gateway**，连接即时通讯平台（Channel，如 Telegram、Discord、Slack 等）与本地 AI Agent，能 **24×7 运行**的个人助手。

它不仅仅是一个简单的消息转发器，而是一个具备完整**会话管理、并发控制、记忆检索**以及丰富工具支持的复杂 Agent 运行时环境。

> **个人评价**：OpenClaw 的完成度非常高，尽管因为各种安全问题被质疑，但其产品思想和围绕这个核心思想设计的各种代码组件和交互，值得开发者反复学习。

![OpenClaw架构图](/images/openclaw/02-architecture.jpg)

本文将深入剖析 OpenClaw 的核心运行流程，结合实际代码和流程图讲解其底层设计原理，并重点介绍 OpenClaw 的 Agent 设计。

---

## 2. 核心交互流程：从消息到 Agent

下图以 Telegram 为例，说明从收到消息，到运行 Agent，最后回调发送消息的一次核心运行流程。

### 2.1 Gateway

如果 AI 助手需要 24×7 运行，那么必然需要一个**持久运行的控制平面**。这个控制平面需要：保持与所有消息渠道的长连接、管理会话状态、响应客户端请求、处理定时任务。**Gateway 就是这个控制平面。**

> **举个例子**：你可以把 Gateway 想象成一个永不关机的"总机"，它同时接听来自 Telegram、Discord、Slack 等多个"电话线路"的消息，并把每条消息转接给对应的 AI 处理单元。

Gateway 核心就是一个 HTTP 和 WebSocket 服务。其启动时与注册的 Channel（比如 Telegram 机器人）建立 WebSocket 连接，随时准备接收消息。

```typescript
// src/gateway/server.impl.ts
// 简化版 Gateway 启动流程
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer> {
  // 1. 设置端口环境变量
  process.env.OPENCLAW_GATEWAY_PORT = String(port);

  // 2. 加载并验证配置
  let configSnapshot = await readConfigFileSnapshot();

  // 3. 创建 WebSocket 服务器
  const wsServer = new WebSocket.Server({ port, host });

  // 4. 注册核心处理器
  const channelManager = createChannelManager(configSnapshot.config);
  const agentEventHandler = createAgentEventHandler(configSnapshot.config);
  const cronService = buildGatewayCronService(configSnapshot.config);

  // 5. 启动通道连接（WhatsApp、Telegram 等）
  await channelManager.startAll();

  // 6. 返回 close 方法用于优雅关闭
  return {
    close: (opts) => shutdownGateway(opts),
  };
}
```

![Gateway启动流程](/images/openclaw/03-gateway.jpg)

Gateway 通过系统服务管理保持 24×7 运行：

```bash
# macOS 启动 Gateway
launchctl load ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# Linux 启动 Gateway
systemctl --user enable --now openclaw-gateway.service
```

### 2.2 消息接入与分发

Telegram 使用的是 **grammY** 作为机器人框架，注册监听事件，回复普通消息。

```typescript
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(opts.token, client ? { client } : undefined);
const processMessage = createTelegramMessageProcessor({bot, ...});
registerTelegramHandlers({ cfg, accountId, bot, processMessage, ...});
```

其中 `createTelegramMessageProcessor` 里使用核心分发函数 `dispatchTelegramMessage` 负责处理来自 Telegram 的消息，以及注册回复消息的回调函数 `deliverReplies`。

> **举个例子**：你在 Telegram 给机器人发了一条"帮我查一下明天的天气"，`dispatchTelegramMessage` 就像一个"调度员"，它接收这条消息，交给 AI 处理，然后把 AI 的回复通过 `deliverReplies` 发回给你。

```typescript
// src/telegram/bot-message-dispatch.ts
export const dispatchTelegramMessage = async ({
  context,
  bot,
  cfg,
  runtime,
  // ... 更多参数
}) => {
  const { msg, chatId, isGroup, historyKey, route } = context;

  // 1. 分发消息
  const { queuedFinal } = await dispatchReplyWithBufferedBlockDispatcher({
    ctx: ctxPayload,
    cfg,
    dispatcherOptions: {
      // 2. 回复消息回调
      deliver: async (payload, info) => {
        const result = await deliverReplies({
          replies: [payload],
          chatId: String(chatId),
          token: opts.token,
          runtime,
          bot,
          // ... 更多选项
        });
      },
      onError: (err, info) => {
        runtime.error?.(`telegram reply failed: ${String(err)}`);
      },
    },
  });
};
```

![消息接入与分发](/images/openclaw/04-channel.jpg)

一些特殊消息能力，需要通过 `message` 工具来发送，而非直接通过 `deliverReplies` 来发送消息。后续工具技能部分会具体介绍。

### 2.3 Session Key 机制

**SessionKey** 是 OpenClaw 中用于标识和路由会话的核心概念。

由于 OpenClaw 支持非常多的 Channel 账号，以及私聊、群组、Thread 等各种会话形式，需要一种命名方式来**唯一识别不同的会话**。

> **举个例子**：就像快递单号一样，每一次对话都有一个唯一的"单号"（SessionKey）。无论你是在 Telegram 私聊机器人，还是在某个群组里 @它，系统都能通过这个"单号"找到对应的对话上下文，不会张冠李戴。

```typescript
// src/routing/session-key.ts
export function buildAgentMainSessionKey(params: {
  agentId: string;
  mainKey?: string | undefined;
}): string {
  const agentId = normalizeAgentId(params.agentId);
  const mainKey = normalizeMainKey(params.mainKey);
  return `agent:${agentId}:${mainKey}`;
}

export function buildAgentPeerSessionKey(params: {
  agentId: string;
  mainKey?: string | undefined;
  channel: string;
  accountId?: string | null;
  peerKind?: "dm" | "group" | "channel" | null;
  peerId?: string | null;
  // ...
}): string {
  // 对于 DM: agent:main:channel:account:dm:peerId
  // 对于群组: agent:main:channel:group:groupId
  // ...
}
```

SessionKey 的格式示例：

- 主会话：`agent:main:main`
- Telegram 私聊：`agent:main:telegram:default:dm:123456789`
- Telegram 群组：`agent:main:telegram:group:1001234567890`

![Session Key机制](/images/openclaw/05-session-key.jpg)

### 2.4 Agent Loop

OpenClaw 支持调用已有的 CLI Agent（比如 Claude Code 等）；但默认情况下也嵌入了基于 **Pi-Agent 框架**执行 Agent 运行时。该框架具备极高的扩展性，满足 OpenClaw 定制化需求（比如大模型供应商支持、Session 管理、工具定制化、流式输出、消息订阅）。

如下图所示，消息进入 AgentSession 后，通过 **ReAct 范式**执行 Agent Loop（包括工具调用），通过注册 reply 回调（监听 `message_end` 事件）将 AI 回复的消息通过机器人发送给 Telegram。

> **什么是 ReAct 范式？** 简单说就是 AI 的"思考-行动"循环：AI 先**思考**（Reasoning）要做什么，然**行动**（Acting）调用工具，看到工具结果后再思考下一步，如此循环，直到任务完成。就像你解一道数学题：先想思路，再动笔算，看中间结果，再继续算。

![Agent Loop流程](/images/openclaw/06-agent-loop.jpg)

#### 故障转移机制

为了能 24×7 持续运行，不能因为一些异常就停止。因此 OpenClaw 实现了完整的**故障转移机制**：

- **Auth Profile 轮换**：当一个 API Key 遇到速率限制或认证失败时，自动切换到下一个可用的 Profile。就像你有多张银行卡，一张刷不了就自动换下一张。
- **上下文溢出自动压缩**：当会话过长时，自动压缩历史消息。就像笔记本写满了，自动帮你做摘要腾出空间。
- **思考级别降级**：当模型不支持扩展思考模式时，自动降级到基本模式。

![故障转移机制](/images/openclaw/07-failover.jpg)

```typescript
// src/agents/pi-embedded-runner/run.ts
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult> {
  // 会话级别并发控制（串行处理）
  const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
  // 全局并发控制（默认并发度 4）
  const globalLane = resolveGlobalLane(params.lane);

  return enqueueSession(() =>
    enqueueGlobal(async () => {
      // 主执行循环，支持故障转移
      while (true) {
        const attempt = await runEmbeddedAttempt({ ... });

        // 处理上下文溢出，自动压缩
        if (isContextOverflowError(errorText)) {
          if (!overflowCompactionAttempted) {
            const compactResult = await compactEmbeddedPiSessionDirect({ ... });
            if (compactResult.compacted) {
              continue; // 使用压缩后的会话重试
            }
          }
        }

        // 处理认证/速率限制故障转移
        if (shouldRotate) {
          const rotated = await advanceAuthProfile();
          if (rotated) continue;
        }

        return { payloads, meta: { durationMs: Date.now() - started, ... } };
      }
    }),
  );
}
```

### 2.5 Agent 核心设计

OpenClaw 并没有从 0 构造 Agent 核心，而是使用开源的 **Pi-Agent 框架**。但 OpenClaw 也定制了其中核心组件，将其打造成开箱即用的、能力丰富的本地个人助手。

下面重点从**并发控制（Queue）、会话管理（Session）、记忆系统（Memory）和工具技能（Tools & Skills）**几个角度介绍。

---

## 3. 队列与并发控制

为了解决群组聊天或高频交互中的"消息竞争"问题，OpenClaw 参考 Pi Agent 设计了一套精密的 **Queue 系统**。

> **什么是消息竞争？** 想象你在一个群里，AI 正在回答 A 的问题，这时 B 和 C 同时又发来了新消息。如果没有队列管理，AI 可能会乱套——同时处理多条消息，上下文混乱，回复错位。Queue 系统就是解决这个问题的。

### 3.1 队列模式（Queue Mode）

OpenClaw 设计了四种核心队列模式，应对不同场景：

- **collect（收集模式，默认）**：将所有排队的消息合并成单个后续回复。适合"等 AI 忙完，把积压的消息打包一起处理"的场景。
- **steer（转向模式）**：立即注入到当前 agent 回合中。适合"紧急插话"，比如 AI 正在执行任务，你突然说"等等，先停下"。
- **followup（跟进模式）**：当前运行结束后，为下一个 agent 回合排队。适合"排队等候"，一条一条处理。
- **steer-backlog（转向+积压模式）**：现在转向当前回合，然后保留消息用于后续回合。

```typescript
// src/auto-reply/reply/queue/types.ts
export type QueueMode = "steer" | "followup" | "collect" | "steer-backlog" | "interrupt" | "queue";

export type QueueSettings = {
  mode: QueueMode;
  debounceMs?: number;  // 防抖延迟（毫秒），默认 1000ms
  cap?: number;         // 队列容量上限，默认 20
  dropPolicy?: "old" | "new" | "summarize"; // 默认 summarize
};
```

**Collect 模式的消息示例**（AI 忙碌时积压的消息会被合并成这样一条）：

```json
{
  "id": "e1c9d464",
  "message": {
    "content": [
      {
        "text": "[Queued messages while agent was busy]\n\n---\nQueued #1\n[Slack x +1s 2026-02-09 16:58 GMT+8] 算了\n\n---\nQueued #2\n[Slack x +4s 2026-02-09 16:58 GMT+8] 查一下天津的",
        "type": "text"
      }
    ],
    "role": "user"
  }
}
```

![队列系统](/images/openclaw/08-queue.jpg)

### 3.2 队列处理逻辑

**Steer 模式**：利用 pi-agent 的 steer 能力，在 Agent loop 中插入消息。

```typescript
// 如果存在运行中的 session，则插入消息
if (shouldSteer && isStreaming) {
  const handle = ACTIVE_EMBEDDED_RUNS.get(sessionId);
  void handle.queueMessage(text);
}

// pi-agent 的 agent-loop
let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];
while (true) {
  // 如果存在待插入的消息，则直接加入到状态中
  if (pendingMessages.length > 0) {
    for (const message of pendingMessages) {
      currentContext.messages.push(message);
    }
    pendingMessages = [];
  }
  // ...
}
```

**Follow Up 和 Collect 模式**：

```typescript
// src/auto-reply/reply/queue/drain.ts
export function scheduleFollowupDrain(key, runFollowup) {
  void (async () => {
    while (queue.items.length > 0 || queue.droppedCount > 0) {
      await waitForQueueDebounce(queue);

      if (queue.mode === "collect") {
        // 批量收集模式：合并所有等待消息为一个 Prompt
        const items = queue.items.splice(0, queue.items.length);
        const prompt = buildCollectPrompt({
          title: "[Queued messages while agent was busy]",
          items,
          renderItem: (item, idx) => `---\nQueued #${idx + 1}\n${item.prompt}`.trim(),
        });
        await runFollowup({ prompt });
        continue;
      }
      // Followup 模式：处理下一个消息
      const next = queue.items.shift();
      await runFollowup(next);
    }
  })();
}
```

### 3.3 并发控制

OpenClaw 使用**两层并发控制**：

- **会话级别**：同一会话内的消息串行处理，避免状态混乱。就像同一个窗口只能一个人办理业务。
- **全局级别**：默认并发度为 4，允许最多 4 个会话同时处理。就像银行同时开 4 个窗口。

Telegram 还专门使用 `bot.use(sequentialize(getTelegramSequentialKey))` 序列化所有消息。

```typescript
// src/agents/pi-embedded-runner/run/lanes.ts
export async function runEmbeddedPiAgent(params) {
  // 全局并发度 4：即保障 active 的会话，不超过 4 个
  return enqueueSession(() =>
    enqueueGlobal(async () => { ... })
  );
}
```

---

## 4. 会话管理：Session

> **一般 session 的对话数据也被称为短期记忆**

与 Agent 的多轮交互需要维持一次会话中历史所有消息（UserMessage、AI Message、Tool Result Message 等）。

> **举个例子**：你和 AI 聊了 10 条消息，第 11 条问"你刚才说的那个方案具体怎么做？"，AI 需要"记得"前 10 条消息才能回答。这些消息就是 Session 里的短期记忆。

OpenClaw 作为本地 Agent，使用 Pi Agent 自带的 Session 管理工具，采用**本地文件系统**作为 Session 存储工具，解决会话数据的持久化问题。

### 4.1 存储结构

物理存储路径：`~/.openclaw/agents/<agentId>/sessions/`

- `session.json`：记录所有 Session 的元数据映射（相当于"目录"）
- `<sessionId>.jsonl`：存储具体的对话日志（JSON Lines 格式，便于追加写入）

> **为什么用 .jsonl 格式？** JSONL（JSON Lines）每行是一条独立的 JSON 记录，非常适合"只追加不修改"的日志场景。就像流水账，每次对话新增一行，不需要重写整个文件，既高效又安全。

![Session存储结构](/images/openclaw/09-session-storage.jpg)

### 4.2 生命周期管理

Agent Session 并不会一直共享同一个上下文，否则上下文窗口很容易超长。因此 OpenClaw 实现了自动化的**会话生命周期管理**：

- **每日重置**：每天自动生成新的 SessionId（通过检测日期变化）。就像每天早上开一个新的工作日志本。
- **空闲归档**：默认 **60 分钟**无交互后归档当前 Session。就像会议室超时自动释放。
- **子 Agent 管理**：子 Agent 的 Session 同样遵循 60 分钟自动归档策略。

因此一个 SessionKey，可能存在多个 SessionId。比如同样在 Telegram 私聊机器人，今天对话使用的上下文和昨天是完全不同的。

![Session生命周期](/images/openclaw/10-session-lifecycle.jpg)

### 4.3 Session 加载和管理

获取 Session 的所有配置（`session.json`）：

```typescript
// src/config/sessions/store.ts
export function loadSessionStore(storePath, opts = {}) {
  let store = {};
  try {
    const raw = fs.readFileSync(storePath, "utf-8");
    const parsed = JSON5.parse(raw);
    if (isSessionStoreRecord(parsed)) {
      store = parsed;
    }
  } catch {
    // 忽略缺失/无效的 store；我们会重新创建
  }
  return structuredClone(store);
}
```

根据 SessionKey 和对应的 SessionId，获取当前会话的历史文件 `sessions/<sessionId>.jsonl`，作为 Pi Agent 的 SessionManager：

```typescript
// 构造 Session Manager，创建 AgentSession
sessionManager = guardSessionManager(
  SessionManager.open(params.sessionFile), {
    agentId: sessionAgentId,
    sessionKey: params.sessionKey,
    ...
  }
);

({ session } = await createAgentSession({
  cwd: resolvedWorkspace,
  model: params.model,
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  ...
}));
```

会话新增消息，加锁后存入文件（防止并发写入冲突）：

```typescript
export async function updateSessionStore(storePath, mutator) {
  return await withSessionStoreLock(storePath, async () => {
    // 在锁内重新读取以避免覆盖并发写入
    const store = loadSessionStore(storePath, { skipCache: true });
    const result = await mutator(store);
    await saveSessionStoreUnlocked(storePath, store);
    return result;
  });
}
```

---

## 5. 记忆系统：Memory

OpenClaw 拥有一个完善的**记忆系统**，通过对记忆相关的 Markdown 文件的实时索引和混合检索来实现**长期记忆**。

> **短期记忆 vs 长期记忆**：Session 里的对话历史是短期记忆（每天重置），而 Memory 系统是长期记忆（跨天持久保存）。就像你的工作日志（短期）和个人笔记本（长期）的区别。

下图将介绍记忆存储、记忆索引、记忆检索的流程：

![记忆系统总览](/images/openclaw/11-memory-system.jpg)

### 5.1 记忆存储

除了 Agent 固定加载的一些相关文件（比如 `AGENTS.md`、`USER.md`、`IDENTITY.md` 等）之外，还需要 Agent 记住其他类型的事情或特定日期发生的事情，这些记忆存储在 `~/.openclaw/workspace` 下：

- `MEMORY.md` 或 `memory.md`：全局长期记忆
- `memory/*.md`：目录中的所有 Markdown 文件
- 额外路径：通过 `memorySearch.extraPaths` 配置

```typescript
// src/memory/internal.ts
export async function listMemoryFiles(workspaceDir, extraPaths) {
  const result = [];
  const memoryFile = path.join(workspaceDir, "MEMORY.md");
  const memoryDir = path.join(workspaceDir, "memory");

  // 添加主记忆文件
  await addMarkdownFile(memoryFile);

  // 递归遍历 memory 目录
  await walkDir(memoryDir, result);

  // 处理额外路径
  for (const inputPath of normalizedExtraPaths) {
    if (stat.isDirectory()) {
      await walkDir(inputPath, result);
    } else if (inputPath.endsWith(".md")) {
      result.push(inputPath);
    }
  }

  return deduped;
}
```

**Memory 写入机制**有三种触发方式：

- **memoryFlush（session 自动压缩）**：只在 context tokens 快满的情况下执行。
- **prompt 触发**：使用类似"记住我"、"记住这个"、"今天"等，Agent 会保存记忆到本地文件。
- **sessionFlush**：`/new` 新 session 保存旧 session 到 `memory/<YYYY-MM-DD>-slug.md`。

### 5.2 混合检索

通过给 Agent 提供 `memory_search` 和 `memory_get` 工具，允许其在合适的时候通过 query 检索历史记忆中相关的文本片段（**RAG 范式**）。

> **什么是 RAG？** RAG（Retrieval-Augmented Generation，检索增强生成）就是"先查资料，再回答"。AI 不是凭空回答，而是先从记忆库里检索相关内容，再基于这些内容生成回答。就像你回答问题前先翻笔记本。

在 OpenClaw 中，使用了典型的**混合检索方案**，即同时通过**关键词精确搜索 + 向量语义检索**，对候选结果计算加权得分，给 Agent 展示最相关的几条。

> **关键词搜索 vs 向量搜索**：关键词搜索就像 Ctrl+F，精确匹配文字；向量搜索则理解语义，比如搜"苹果手机"也能找到"iPhone"相关内容。两者结合，既精确又智能。

基于本地个人 Agent 的定位，OpenClaw 的精确检索和向量检索都默认使用 **SQLite** 作为数据库存储（结果是一个 `agents.sqlite` 文件）。

![混合检索流程](/images/openclaw/12-memory-index.jpg)

Memory 检索工具：

```typescript
// src/agents/tools/memory-tool.ts
export function createMemorySearchTool(options) {
  return {
    name: "memory_search",
    description:
      "Mandatory recall step: semantically search MEMORY.md + memory/*.md " +
      "before answering questions about prior work, decisions, dates, people, " +
      "preferences, or todos; returns top snippets with path + lines.",
    execute: async (_toolCallId, params) => {
      const query = readStringParam(params, "query", { required: true });
      const results = await manager.search(query, { maxResults, minScore });
      return jsonResult({ results });
    },
  };
}
```

MemoryManager 执行混合检索：

```typescript
// src/memory/manager.ts
export class MemoryIndexManager {
  async search(query, opts) {
    // 关键词搜索
    const keywordResults = await this.searchKeyword(cleaned, candidates);

    // 向量搜索
    const queryVec = await this.embedQueryWithTimeout(cleaned);
    const vectorResults = await this.searchVector(queryVec, candidates);

    // 合并结果（加权融合）
    const merged = this.mergeHybridResults({
      vector: vectorResults,
      keyword: keywordResults,
      vectorWeight: hybrid.vectorWeight,
      textWeight: hybrid.textWeight,
    });

    return merged.filter((entry) => entry.score >= minScore).slice(0, maxResults);
  }
}
```

下面简单示例如何使用 **sqlite-vec** 执行 KNN 向量检索：

```javascript
db.exec(`CREATE VIRTUAL TABLE vec_items USING vec0(embedding float[4])`);

// 插入向量
const insert = db.prepare(`INSERT INTO vec_items(rowid, embedding) VALUES (?, ?)`);
insert.run(BigInt(1), new Float32Array([0.1, 0.1, 0.1, 0.1]));

// KNN 搜索（找最近的 3 个）
const query = new Float32Array([0.15, 0.15, 0.15, 0.15]);
const rows = db.prepare(`
  SELECT rowid, distance FROM vec_items
  WHERE embedding MATCH ? ORDER BY distance LIMIT 3
`).all(query);
```

![向量检索示意](/images/openclaw/13-hybrid-search.jpg)

### 5.3 索引写入

记忆检索依赖的本地 SQLite 数据库，是在特定时机触发索引写入的。比如，在监听到 Memory 文件变更后，会将其同步索引到本地数据库中，方便后续进行记忆检索。

单个文件的索引写入分为三个部分：

1. **文件分块**：先将文件根据 `chunkTokens` 配置分为固定大小的块，为了增强前后文的感知，允许一定 token 的分块重叠。就像把一本书切成若干段落，相邻段落有少量重叠，避免语义断裂。
2. **Embedding 计算和缓存**：利用配置的远程或本地 embedding 模型计算 chunk 的语义向量。同时由于经常多次索引，因此可以利用 embedding 缓存来避免重复计算。
3. **写入本地数据库**：写入索引检索依赖的 `chunks_vec`（向量表）和 `chunks_fts`（全文搜索表）两种表。

---

## 6. 工具技能（Tools & Skills）

OpenClaw 提供了极其丰富且开箱即用的工具技能，这也是其能大火的原因之一。

除了通用 CLI Agent 都有的**文件系统访问、Shell 命令执行、webSearch 工具**外，还有获取 Gateway、Session 等运行状态的**自感知能力**，以及内置的多个实用的插件工具（比如 bird、message、browser、天气等）。

![工具技能总览](/images/openclaw/14-tools-skills.jpg)

### 6.1 工具示例：message

下面介绍 OpenClaw 比较独特的 `message` 工具。与特定 Channel 集成，能够达到丝滑的交互效果：

- **发送多条消息**：AI 可以在最终回复用户之前或之后，先发送一张图片、一个文件或者另一条补充消息。（主动式 Agent 的基础）
- **富文本与复杂交互**：显式设置 `buttons`（内联按钮）、`card`（卡片）、`poll`（投票）等高级功能。
- **引用与回复**：精准控制 `replyTo`（回复特定消息 ID），而不仅仅是回复当前最后一条。

> **举个例子**：AI 出了一道日语练习题，并通过 `message` 工具发送了带选项按钮的消息，用户直接点击按钮作答，而不是手动输入文字。这种交互体验远超普通聊天机器人。

```json
// message 工具调用示例
{
  "action": "send",
  "buttons": "[[{\"text\":\"A. 下午好\", \"callback_data\":\"n5_quiz_wrong\"}, {\"text\":\"B. 再见\", \"callback_data\":\"n5_quiz_correct\"}], [{\"text\":\"C. 谢谢\", \"callback_data\":\"n5_quiz_wrong\"}, {\"text\":\"D. 早上好\", \"callback_data\":\"n5_quiz_wrong\"}]]",
  "channel": "telegram",
  "message": "📚 **日语N5练习题**\n\n**さようなら** 的中文意思是什么？",
  "target": "123456"
}
```

![message工具示例](/images/openclaw/15-message-tool.jpg)

### 6.2 Skills 示例：bird

复习一下 Skills 的工作原理：通过**文件系统 + 渐进式披露**，标准化和模块化技能提示词。

Skills 将从三个位置加载：

- **内置 Skills**：随安装包一起发布（npm 包或 OpenClaw.app），比如 bird、github 等
- **托管/本地 Skills**：`~/.openclaw/skills`
- **工作区 Skills**：`<workspace>/skills`

> **什么是"渐进式披露"？** 就是按需加载提示词。不是一次性把所有技能的说明都塞给 AI，而是 AI 需要用某个技能时，才加载对应的详细说明。这样既节省 token，又保持灵活性。

下面以 bird skills 为例，介绍如何搜索和总结 Twitter（X）上的内容：

![bird skill示例](/images/openclaw/16-bird-skill.jpg)

### 6.3 自定义工具和技能

OpenClaw 还允许用户自定义加入工具和技能。官方推荐使用 `clawhub` 命令安装 [clawhub.ai](https://clawhub.ai/) 里的技能，或通过 `plugin` 命令安装插件：

```bash
# 技能安装，以 artifacts-builder 为例
npm i -g clawhub
clawhub install artifacts-builder

# 插件安装
openclaw plugins list
openclaw plugins install @openclaw/voice-call
```

你也可以直接让 OpenClaw 帮你安装，或者直接将 skills 添加到目录 `.openclaw/workspace/skills/`。

### 6.4 工具策略

OpenClaw 支持**多个层级的工具配置策略**，即特定的场景可以配置不同的工具组合可见性：

- **全局策略**：`config.tools`
- **全局按提供商策略**：`config.tools.byProvider[providerOrModelId]`
- **Agent 策略**：`config.agents.[agentId].tools`
- **Agent 按提供商策略**：`config.agents.[agentId].tools.byProvider[providerOrModelId]`
- **群组策略**：`config.groups.[groupId].tools`（针对特定群组）

> **举个例子**：你可以配置"在 Telegram 群组里，AI 只能用搜索工具，不能执行 Shell 命令"，而"在私聊里，AI 可以使用所有工具"。这种细粒度的权限控制，在安全性和灵活性之间取得了很好的平衡。

![工具策略配置](/images/openclaw/17-tool-strategy.jpg)

---

## 7. 总结

整体来看，OpenClaw 在 Agent 核心层面没有特别多新颖的地方，但作为一个**完成度特别高的产品**——24×7 运行的本地个人助手，其架构设计思想和扩展集成值得我们反复学习。

本文从核心交互流程出发，了解消息接入和分发链路，以及 SessionKey 的机制，一步一步深入 Agent 核心设计：

- **Queue** 如何解决复杂的并发消息问题
- **Session** 如何管理文件解决短期记忆的持久化问题
- **Memory** 如何通过记忆文件保存和检索长期记忆
- **Tools & Skills** 如何提供丰富的工具能力和灵活的扩展机制

OpenClaw 的成功，不在于某一项技术的突破，而在于将这些技术**有机地整合在一起**，形成了一个真正可用、好用的本地 AI 助手。这种系统工程能力，才是最值得学习的地方。

---

> 原文来源：[OpenClaw核心运行流程 - 知乎](https://zhuanlan.zhihu.com/p/2004505448938767308)
