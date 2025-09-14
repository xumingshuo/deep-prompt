# Deep Research 项目深度分析报告 (详细版)

本文在上一版分析的基础上，深入到代码实现的细节，对项目的核心机制、数据流和扩展性设计进行更详尽的剖析。

## 1. 项目概述 (回顾)

**Deep Research** 是一个基于 Next.js 的 AI 应用，通过模拟人类研究员的思维过程（规划、搜集、整合、撰写），自动化地生成关于特定主题的深度研究报告。它整合了多种大语言模型（LLM）和网络搜索能力，设计上注重隐私、可扩展性和用户体验。

## 2. 技术栈分析 (回顾)

- **核心框架**: Next.js 15 (React 19), TypeScript
- **UI**: Shadcn/UI, Tailwind CSS
- **状态管理**: Zustand
- **AI/LLM**: Vercel AI SDK, 支持 Google, OpenAI, Anthropic, Ollama 等
- **数据处理**: React Hook Form, Zod, `react-markdown`, Mermaid.js
- **存储与部署**: LocalForage, PWA, Docker

## 3. 项目结构分析 (回顾)

项目结构清晰，`src/app` 存放页面和 API，`src/components` 存放 UI 组件，`src/hooks` 存放核心业务逻辑，`src/store` 存放全局状态，`src/utils` 存放工具函数和更底层的业务逻辑。

---

## 4. 核心机制深度剖析

### 4.1. Prompt 工程揭秘：AI 的指挥脚本

项目的“智能”很大程度上源于其精心设计的 Prompt。这些 Prompt 存储在 `src/constants/prompts.ts` 中，并通过 `src/utils/deep-research/prompts.ts` 中的函数进行格式化和组装。它们是指导 LLM 在研究流程中每一步具体行为的“指挥脚本”。

**阶段一：生成研究计划 (Writing the Report Plan)**

当用户输入主题后，`writeReportPlanPrompt` 函数会构建如下核心 Prompt：

```typescript
// from src/constants/prompts.ts (simplified)
export const reportPlanPrompt = `...
你是一名世界一流的 AI 研究员，擅长将复杂主题分解为结构清晰的研究计划。请为以下主题撰写一份详细的研究报告大纲。

**主题:** {query}

**要求:**
1.  报告应有清晰的标题。
2.  大纲应包含引言、核心章节和结论。
3.  每个核心章节都应列出具体的研究要点。
...`
```

这个 Prompt 为 LLM 设定了明确的**角色**（世界一流的 AI 研究员）和**任务**（为主题 `{query}` 创建一个结构化的大纲），并给出了具体的**格式要求**。这是确保后续所有步骤都围绕一个清晰结构展开的关键第一步。

**阶段二：生成搜索查询 (Generating SERP Queries)**

基于上一步的计划，`generateSerpQueriesPrompt` 函数会指示 LLM 将计划分解为具体的搜索任务。这是最关键的 Prompt 之一：

```typescript
// from src/constants/prompts.ts (simplified)
export const serpQueriesPrompt = `...
你是一个研究助理，任务是将研究计划分解为一系列搜索引擎查询（SERP Queries）。对于计划中的每个要点，生成一个或多个具体的查询，并为每个查询定义其“研究目标”。

**研究计划:**
{plan}

**输出格式要求:**
你必须严格按照以下 JSON Schema 格式输出一个 JSON 数组，不要有任何额外的解释或文本。

**JSON Schema:**
{outputSchema}
`

// from src/utils/deep-research/prompts.ts
// The outputSchema is dynamically generated using zodToJsonSchema
export function getSERPQuerySchema() {
  return z.array(
    z.object({
      query: z.string().describe("The SERP query."),
      researchGoal: z.string().describe("First talk about the goal of the research that this query is meant to accomplish..."),
    })
  );
}
```

这里的精妙之处在于：
1.  **角色转换**：LLM 从“研究员”变为“研究助理”，专注于执行分解任务。
2.  **强制 JSON 输出**：通过提供一个明确的 JSON Schema (`outputSchema`) 并三令五申地要求严格遵守，确保了 LLM 的输出是机器可读的。这避免了对自然语言输出进行不稳定的解析。
3.  **定义研究目标 (`researchGoal`)**：这至关重要。它不仅生成了搜索词，还让 LLM 思考“为什么我要搜这个？”以及“搜到之后要干什么？”。这个 `researchGoal` 会在下一步被用来指导信息的总结。

**阶段三 & 五：信息总结与最终报告撰写**

在信息收集后，`processSearchResultPrompt` 指示 LLM 根据 `researchGoal` 总结搜索结果。而在最后，`writeFinalReportPrompt` 则是一个集大成者，它向 LLM 提供了**所有**的上下文信息：研究计划、所有子任务的总结（learnings）、所有引用来源（sources）和图片，然后要求它撰写最终报告。这种将所有材料一次性提供给 LLM 的方式，使其能够进行全局性的整合和创作，而不是零散地拼接内容。

### 4.2. 核心状态管理：Zustand 的应用

整个研究过程的动态 UI 完全由 `src/store/task.ts` 中的 Zustand store 驱动。这个 store 设计得非常全面，它就是当前研究任务在内存中的“数据库”。

其核心数据结构 `TaskStore` 定义如下：

```typescript
// from src/store/task.ts
export interface TaskStore {
  id: string; // 当前任务的唯一ID，用于历史记录
  question: string; // 用户的原始问题
  resources: Resource[]; // 用户上传的本地知识库文件
  query: string; // 格式化后的研究主题
  reportPlan: string; // LLM生成的研究计划
  tasks: SearchTask[]; // **核心**：所有搜索子任务的列表
  finalReport: string; // 最终生成的报告
  // ... 其他如 title, sources, suggestion 等
}

// 其中 SearchTask 的结构
interface SearchTask {
  query: string; // 这个子任务的搜索查询
  researchGoal: string; // 这个子任务的研究目标
  state: "unprocessed" | "processing" | "completed" | "failed"; // 任务状态
  learning: string; // LLM对该任务搜索结果的总结
  sources?: Source[];
  images?: ImageSource[];
}
```

**数据流与 UI 响应：**

1.  **初始化**：当用户发起研究，`useDeepResearch` hook 会调用 `taskStore` 的 actions（如 `setQuery`, `updateReportPlan`）。
2.  **子任务更新**：在“信息收集”阶段，`runSearchTask` 函数会遍历 `tasks` 数组。对于每个任务，当 LLM 流式返回总结内容时，会频繁调用 `updateTask` action。
    ```typescript
    // from src/store/task.ts
    updateTask: (query, task) => {
      const newTasks = get().tasks.map((item) => {
        return item.query === query ? { ...item, ...task } : item;
      });
      set(() => ({ tasks: [...newTasks] }));
    },
    ```
    这个 action 遵循了不可变状态的原则：它创建一个新的 `tasks` 数组，而不是直接修改旧数组。Zustand 检测到状态变化，并通知所有订阅了该 store 的 React 组件（如 `SearchResult.tsx`）进行重新渲染，从而在 UI 上实时展示每个任务的流式生成内容和状态变化（例如，从未处理到处理中，再到完成）。
3.  **最终报告**：同理，`writeFinalReport` 函数通过 `updateFinalReport` action 持续更新 `finalReport` 字段，驱动最终报告区域的实时显示。

这种中心化的状态管理方式，使得复杂的、多步骤、多状态的业务逻辑变得清晰可控。

### 4.3. API 服务层：SSE 与 MCP

项目不仅是一个前端应用，还通过 Next.js API Routes 提供了强大的后端服务能力。

**A. Server-Sent Events (SSE) - `/api/sse`**

这是为了让外部客户端能以流式方式调用深度研究功能。`src/app/api/sse/route.ts` 的实现很典型：

1.  **创建可读流**：它创建了一个 `ReadableStream`。
2.  **实例化 `DeepResearch` 类**：它使用了 `src/utils/deep-research/index.ts` 中定义的 `DeepResearch` 类。这个类封装了与 `useDeepResearch` hook 几乎相同的核心逻辑，但它是为在服务器环境运行而设计的。
3.  **注入回调**：在实例化 `DeepResearch` 时，传入一个 `onMessage` 回调函数。
    ```typescript
    // from src/app/api/sse/route.ts
    const deepResearch = new DeepResearch({
      // ... provider configs
      onMessage: (event, data) => {
        // ... log progress
        controller.enqueue(
          encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)})}\n\n`)
        );
      },
    });
    ```
4.  **启动并推送**：当 `deepResearch.start()` 运行时，其内部的每一步（如 `writeReportPlan`, `runSearchTask`）都会调用这个 `onMessage` 回调。回调函数的作用就是将进度（`progress`）、消息（`message`）、错误（`error`）等事件，按照 SSE 格式编码后推送到流中。客户端接收到这些事件后，就可以实时了解研究的进展。

**B. Model Context Protocol (MCP) - `/api/mcp`**

MCP 是一个更高级、更结构化的协议，它将研究功能暴露为一组明确的“工具”。这使得其他 AI Agent 或智能系统可以像调用函数一样使用本项目的能力。

`src/app/api/mcp/server.ts` 中 `initMcpServer` 的实现揭示了其工作原理：

1.  **创建 MCP 服务器**：`const server = new McpServer(...)`。
2.  **定义工具**：使用 `server.tool()` 方法定义多个工具，每个工具都对应研究流程的一个步骤。
    ```typescript
    // from src/app/api/mcp/server.ts
    server.tool(
      "write-research-plan",
      writeResearchPlanDescription,
      {
        query: z.string().describe("The topic for deep research."),
        language: z.string().optional().describe("The response Language."),
      },
      async ({ query, language }, { signal }) => {
        // ...
        const deepResearch = initDeepResearchServer({ language });
        const result = await deepResearch.writeReportPlan(query);
        return {
          content: [{ type: "text", text: JSON.stringify({ reportPlan: result }) }],
        };
      }
    );
    ```
    这里的关键是：
    - **工具名**：`write-research-plan`。
    - **输入 Schema**：使用 `zod` 定义了工具的输入参数（`query` 和 `language`）及其类型和描述。MCP 客户端可以据此了解如何调用该工具。
    - **处理器**：一个 `async` 函数，它接收经过 Zod 验证后的参数，然后调用 `DeepResearch` 类中相应的方法 (`writeReportPlan`)，最后将结果封装成 MCP 协议规定的格式返回。

通过这种方式，MCP 将一个复杂的流程分解为一组定义清晰、输入输出明确的 RPC-like 服务，极大地提高了其作为后端服务被集成的能力和可靠性。

### 4.4. 扩展性设计：如何添加新 Provider

项目的另一个亮点是其良好的扩展性。以添加一个新的 AI Provider 为例，其设计模式非常清晰，主要涉及修改 `src/hooks/useAiProvider.ts`。

```typescript
// from src/hooks/useAiProvider.ts
function useModelProvider() {
  async function createModelProvider(model: string, settings?: any) {
    const { mode, provider, accessPassword } = useSettingStore.getState();
    const options: AIProviderOptions = { /* ... */ };

    switch (provider) {
      case "google":
        // ... google config
        break;
      case "openai":
        // ... openai config
        break;
      // ... other providers

      // 要添加一个名为 'new-provider' 的新提供商
      case "new-provider":
        // 1. 从 Zustand store 获取该 provider 专属的 API Key 和代理地址
        const { newProviderApiKey = "", newProviderApiProxy } = useSettingStore.getState();
        if (mode === "local") {
          // 2. 在本地模式下，配置其 API baseURL 和 apiKey
          options.baseURL = completePath(newProviderApiProxy || NEW_PROVIDER_DEFAULT_URL, "/v1");
          options.apiKey = multiApiKeyPolling(newProviderApiKey);
          // 3. 如果需要特殊请求头，在这里添加
          options.headers = { "X-Custom-Header": "value" };
        } else {
          // 4. 在代理模式下，指向 Next.js 的内部 API 代理路由
          options.baseURL = location.origin + "/api/ai/new-provider/v1";
        }
        break;

      default:
        break;
    }

    if (mode === "proxy") {
      options.apiKey = generateSignature(accessPassword, Date.now());
    }

    // 5. 调用通用的创建函数
    return await createAIProvider(options);
  }

  // ... getModel 和 hasApiKey 函数也需要添加相应的 case
}
```

添加新 Provider 的步骤清晰明了：
1.  在 `setting.ts` (Zustand store) 中为新的 Provider 添加 API Key 和其他配置的状态。
2.  在 `Setting.tsx` 组件中添加相应的输入框让用户可以配置。
3.  在 `useAiProvider.ts` 的 `createModelProvider`, `getModel`, `hasApiKey` 三个函数的 `switch` 语句中，添加新的 `case`。
4.  在 `next.config.ts` 的 `rewrites` 中为代理模式添加新的路由重写规则。
5.  在 `src/utils/deep-research/provider.ts` 的 `createAIProvider` 中，确保能正确处理新的 `provider` 标识，并实例化正确的 AI SDK 客户端。

这种基于 `switch` 和统一配置对象 (`AIProviderOptions`) 的模式，使得添加新的服务集成变得非常简单，只需在几个固定的地方添加相似的代码块即可。

## 5. 总结

通过对代码细节的深入分析，我们可以看到 **Deep Research** 不仅是一个功能强大的应用，更是一个在软件工程实践上表现出色的项目。其对 Prompt 工程的精妙运用、清晰的状态管理模型、分层的 API 设计以及良好的代码扩展性，共同构成了一个健壮、高效且易于维护的复杂 AI 系统。对于任何想要构建类似 Agentic AI 应用的开发者来说，这个项目都是一个极具价值的学习和参考范本。
