# Deep Research 项目深度分析报告

## 1. 项目概述

**Deep Research** 是一个基于 Next.js 构建的现代化 Web 应用，旨在利用大语言模型（LLM）和网络搜索技术，快速生成关于任意主题的深度研究报告。项目核心理念是通过一个多步骤、自动化的流程，模拟人类研究员的工作方式：从提出问题、规划研究、收集信息到最终撰写报告。

该项目具备高度的可扩展性和灵活性，支持多种主流 LLM（如 Gemini, OpenAI, Anthropic, Ollama 等）和多种搜索引擎（如 Tavily, Searxng, Exa 等）。其设计注重用户隐私，所有数据默认在浏览器本地处理和存储。同时，项目也提供了服务端 API（SSE 和 MCP），允许将其深度研究能力作为服务（SaaS）集成到其他应用中。

## 2. 技术栈分析

根据 `package.json` 文件和项目结构，该项目的技术栈如下：

- **前端框架**: [Next.js](https://nextjs.org/) (v15)，使用 React (v19) 和 TypeScript。项目利用了 Next.js 的 App Router、Edge Runtime 以及强大的构建和路由功能。
- **UI 组件库**: [Shadcn/UI](https://ui.shadcn.com/)，这是一个基于 Radix UI 和 Tailwind CSS 的组件集合，提供了高质量、可访问的 UI 构建块。
- **CSS 框架**: [Tailwind CSS](https://tailwindcss.com/)，用于快速构建现代化的、响应式的用户界面。
- **状态管理**: [Zustand](https://github.com/pmndrs/zustand)，一个轻量级、灵活的 React 状态管理库。
- **AI/LLM 集成**:
    - **Vercel AI SDK**: 核心库，用于与各种 LLM API 进行交互，特别是其提供的 `streamText` 功能，实现了报告生成过程的流式输出。
    - **多模型支持**: 项目通过 `@ai-sdk/*` 系列包（如 `@ai-sdk/google`, `@ai-sdk/openai`）和自定义 Provider，实现了对 Google Gemini, OpenAI, Anthropic, Azure, Ollama 等多种模型的支持。
- **表单处理**: [React Hook Form](https://react-hook-form.com/) 配合 [Zod](https://zod.dev/) 进行表单验证。
- **Markdown 处理**:
    - `react-markdown`, `remark-gfm`, `rehype-highlight` 等库用于渲染最终报告的 Markdown 内容。
    - `@xiangfa/mdeditor` 用于提供“所见即所得”的 Markdown 编辑体验。
- **图表渲染**: [Mermaid.js](https://mermaid.js.org/)，用于在报告中渲染流程图、序列图等。
- **本地存储**: [LocalForage](https://github.com/localForage/localForage)，用于在浏览器中持久化存储研究历史和用户设置。
- **国际化 (i18n)**: [i18next](https://www.i18next.com/)，支持多语言界面（英语、中文、西班牙语等）。
- **PWA 支持**: `@serwist/next` 用于将应用构建为渐进式网络应用（PWA），可以“安装”到桌面或主屏幕。
- **打包与构建**: 使用 Next.js 内置的构建系统，并支持 Turbopack 进行开发环境加速。`pnpm` 作为包管理器。

## 3. 项目结构分析

项目的代码结构清晰，遵循了现代 Next.js 应用的最佳实践。

- **`src/app/`**: Next.js App Router 的核心目录。
    - **`page.tsx`**: 应用的主页面，是用户交互的入口。
    - **`layout.tsx`**: 全局布局文件，包含了 Provider（主题、i18n）、Header 等通用组件。
    - **`api/`**: 后端 API 路由。这是项目的“大脑”所在，分为多个子目录：
        - **`ai/`**: 代理各种 LLM 服务的 API 请求。`next.config.ts` 中的 `rewrites` 配置将 `/api/ai/[provider]` 的请求转发到实际的 LLM API 地址，这是一种常见的隐藏 API Key 和解决 CORS 问题的方法。
        - **`search/`**: 代理各种搜索引擎服务的 API 请求，逻辑与 `ai/` 类似。
        - **`sse/`**: 实现了 Server-Sent Events (SSE) API，用于流式传输深度研究的实时进度和结果。
        - **`mcp/`**: 实现了自定义的“模型上下文协议”（Model Context Protocol），这是一种更高级的 API 形式，允许将研究功能作为服务被其他程序调用。
- **`src/components/`**: 可复用的 React 组件。
    - **`ui/`**: 从 Shadcn/UI 生成的基础 UI 组件。
    - **`Research/`**: 与研究流程相关的核心组件，如 `Topic` (主题输入)、`SearchResult` (搜索结果展示)、`FinalReport` (最终报告)。
    - **`MagicDown/`**: 强大的 Markdown 编辑器和查看器组件。
- **`src/hooks/`**: 自定义 React Hooks。
    - **`useDeepResearch.ts`**: **项目的核心逻辑 Hook**。它封装了整个研究流程的所有步骤，包括调用 LLM 生成问题、制定计划、执行搜索、撰写报告等。
    - **`useAiProvider.ts`**, **`useWebSearch.ts`**: 分别用于创建和管理对 AI 模型和搜索引擎的访问。
- **`src/store/`**: Zustand 的状态存储。
    - **`task.ts`**: 存储当前研究任务的状态，包括用户查询、研究计划、各个子任务的进展和结果、最终报告等。这是驱动 UI 实时更新的核心数据源。
    - **`setting.ts`**, **`history.ts`**: 分别管理用户设置和历史研究记录。
- **`src/utils/`**: 通用工具函数和核心业务逻辑。
    - **`deep-research/`**: 包含了深度研究过程的核心实现。
        - **`index.ts`**: 定义了 `DeepResearch` 类，这是为服务端 API (SSE/MCP) 准备的核心逻辑封装。
        - **`prompts.ts`**: **项目的灵魂所在**。这里定义了在研究流程中每个步骤所使用的 Prompt 模板，指导 LLM 如何思考和行动。
        - **`provider.ts`**, **`search.ts`**: 创建不同 AI 和搜索服务的提供者实例。
- **`public/`**: 静态资源，如图片、脚本等。
- **`docs/`**: 项目文档，如 API 文档和部署指南。
- **配置文件**:
    - **`next.config.ts`**: Next.js 配置文件，定义了 API 代理、PWA 配置、构建模式等。
    - **`package.json`**: 项目依赖和脚本。
    - **`env.tpl`**: 环境变量模板，指导用户如何配置 API Keys 等。

## 4. 核心机制与原理

`Deep Research` 的工作流程在 `README.md` 的流程图和 `useDeepResearch.ts` 的代码中得到了清晰的体现。这是一个精心设计的 **Agentic Workflow**（代理工作流），其核心步骤如下：

1.  **启动阶段 (Ask Questions / Write Report Plan)**:
    - 用户输入一个研究主题 (`query`)。
    - `useDeepResearch.ts` 中的 `askQuestions` 或 `writeReportPlan` 函数被调用。
    - 它会使用 `thinkingModel`（一个能力较强的 LLM）和 `generateQuestionsPrompt` 或 `writeReportPlanPrompt` 中的提示，生成一系列引导性问题或直接生成一份研究大纲（Report Plan）。
    - 这个大纲是整个研究过程的“蓝图”。

2.  **生成搜索查询 (Generate SERP Queries)**:
    - 基于上一步生成的研究大纲，`deepResearch` 函数调用 LLM（同样是 `thinkingModel`），并使用 `generateSerpQueriesPrompt`。
    - 这个 Prompt 指示 LLM 将研究大纲分解为多个具体的、适合搜索引擎的查询关键词（SERP Queries），并为每个查询定义一个明确的研究目标 (`researchGoal`)。
    - LLM 被要求以 JSON 格式输出这些查询，便于程序解析。

3.  **信息收集 (Run Search Task)**:
    - `runSearchTask` 函数接收上一步生成的查询列表，并为每个查询启动一个并行的搜索任务（并行度可配置）。
    - 对于每个任务，它会：
        - **调用搜索引擎**: 使用 `useWebSearch` hook（或在服务器端使用 `createSearchProvider`）执行网络搜索。
        - **调用 LLM 总结**: 将搜索结果（网页片段）喂给 `taskModel`（一个可能更快速、成本更低的 LLM），并使用 `processSearchResultPrompt` 指示它根据 `researchGoal` 总结信息，并生成带有引用的“学习内容”（`learning`）。
        - **流式更新**: 总结过程是流式的，结果会实时更新到 `taskStore` 中，UI 上会显示每个子任务的进展。

4.  **（可选）审查与深化研究 (Review Search Result)**:
    - 在第一轮信息收集后，系统可以进入一个审查阶段 (`reviewSearchResult`)。
    - 它会将已有的学习成果 (`learnings`) 再次提交给 `thinkingModel`，并使用 `reviewSerpQueriesPrompt` 提问：“基于现有信息，是否需要进行更深入或补充性的研究？”
    - 如果 LLM 认为需要，它会生成新一轮的 SERP Queries，并重复步骤 3。这个循环可以实现研究的逐步深化。

5.  **生成最终报告 (Write Final Report)**:
    - 当所有信息收集任务完成后，`writeFinalReport` 函数被调用。
    - 它将所有子任务的“学习内容”（`learnings`）以及收集到的所有引用来源（`sources`）和图片（`images`）整合起来。
    - 这些材料被喂给 `thinkingModel`，并使用 `writeFinalReportPrompt` 这个最复杂的 Prompt。
    - 该 Prompt 指示 LLM 扮演一个专业研究员的角色，基于提供的所有材料，按照研究大纲（`reportPlan`）的结构，撰写一篇全面、连贯、格式规范（Markdown）的最终报告。
    - 报告的生成同样是流式的，用户可以实时看到报告“被写出来”的过程。

## 5. 结论

**Deep Research** 是一个设计精良、功能强大的 AI 应用典范。它不仅是一个简单的“聊天机器人”，而是一个复杂的、自动化的信息处理系统。

**优点**:
- **工作流设计巧妙**: 模仿人类专家的研究模式，分步解决复杂问题，保证了最终报告的结构性和深度。
- **技术栈现代**: 采用 Next.js、TypeScript、Tailwind CSS 等流行技术，开发体验和应用性能俱佳。
- **高度可配置和可扩展**: 支持多种 LLM 和搜索引擎，并且通过 API 提供了强大的集成能力。
- **优秀的用户体验**: 流式输出、实时进度更新、PWA 支持、多语言等功能，都极大地提升了用户体验。
- **代码质量高**: 代码结构清晰，逻辑分离（UI、Hooks、Store、Utils），易于理解和维护。

**潜在应用**:
- 学术研究初稿撰写。
- 市场分析报告生成。
- 技术可行性分析。
- 任何需要快速、全面地了解一个新领域的场景。

总而言之，该项目是学习如何构建复杂 AI Agent 应用的绝佳案例，其在 Prompt Engineering、系统设计和前端工程方面的实践都值得深入研究。
