# Claude Code - Decompiled Source Code (v2.1.88)

> **声明 / Disclaimer**：
> 本仓库的代码来源于官方发行的 `@anthropic-ai/claude-code` CLI 工具（版本 2.1.88）安装包中的 Source Map (`cli.js.map`) 逆向提取。
> 本项目仅作逆向安全研究、代码架构学习、Prompt 设计参考使用，所有的版权均归 Anthropic 所有。请勿用于未授权的商业用途。
>
> 官方主页: [https://claude.com/product/claude-code](https://claude.com/product/claude-code)

---

## 📖 项目简介

本仓库包含了 **Claude Code**（Anthropic 官方出品的首款基于终端的前沿 Agentic 编码工具）的 TypeScript 核心业务逻辑源码。
虽然受限于 Source Map 提取的先天限制（如被构建工具 Tree-shaking 裁掉的分支代码、动态生成的环境宏代码会在解包后丢失，导致部分模块无法在未经补丁修复的情况下直接运行），但这套跨越 4700 多个文件的源码依然是**整个开源界乃至业界最具参考价值的大语言模型客户端/Agent 工程架构实现库**之一。

### 为什么这份源码极具研究价值？
与市面上大多封装 API 和流式输出的简单套壳产品不同，Claude Code 代表了当前 AI 工具在终端生态内的最高工业化标准。它内部集成了一套极端鲁棒的 LLM 调度引擎、基于细粒度的层级代理（Hierarchical Agents）协同模式、原生沙盒安全分类器以及 React 在终端的高优动态渲染机制。

无论你是想要开发高级 AI Agent、自动驾驶编程工具（Auto-Coder）、还是终端图形化（TUI）程序，本项目都提供了一套接近业界天花板级别的“标准答案”。

---

## 🏗 核心技术架构深度剖析

这套工具打破了我们对于“命令行应用=简单脚本”的刻板印象，下面对其各个核心子系统的架构进行深度拆解分析：

### 1. 组件渲染层：Ink 驱动的终端 React 生态 (`src/components/`, `src/screens/`)
Claude Code 在终端上实现了网页体验：支持气泡框、动画加载、甚至可以滚动的代码 Diff 显示。
* **基于 React 架构**：大量使用了 [Ink](https://github.com/vadimdemedes/ink) 使得终端可以像浏览器 DOM 一样响应式重构。通过 `src/components/Message.tsx` 等组件，将后端 Agent 的流式输出（Streaming Block）与 React 的 State 进行周期性绑定，实现打字机效果。
* **多界面切换层**：`src/screens/` 包含不同的视图场景（任务模式、聊天交互模式、全局 Loading 屏等）。
* **亮点细节**：它内部封装了非常精良的组件，例如平滑无闪烁的进度加载指示器（Spinner）、智能 Markdown 转终端高亮的渲染管线。

### 2. 行为工具箱与执行沙盒 (`src/tools/`)
这是 Agent 抓取代码及对系统环境进行影响的“手和脚”，极具模块化设计：
* **标准操作能力**：
  - `BashTool`: 核心。不是简单的 `exec` 封装，内部实现了命令长驻分离、环境变量沙盒隔离以及执行超时、死循环控制。
  - `FileEditTool` / `FileReadTool`: 代码的直接更改口。使用了差异比对合并算法而不是单纯的文件覆盖。
  - `GlobTool` / `GrepTool`: 高级查询工具，借助底层的 `ugrep` / `find` 实现极其快速的本地上下文语料注入。
* **高层级控制逻辑**：
  - `AgentTool`: **神级设计**。当主 Claude 发现单次思考不足以解决宏大问题，可通过此 Tool 唤起一个带独立思考环境的 Sub-Agent（子代理）进行专门攻坚，主代理等候结果汇总。
* **外部设备与网络协议**：
  - `MCPTool`: 包含了 Chrome 协议 (`src/utils/claudeInChrome`) 配置和通用扩展等协议注入。这也佐证了 Claude 官方深度整合 MCP 标准的布局方案。

### 3. 多模型与代理协同核心 (`src/coordinator/`)
Anthropic 用“工头-打工仔”模式在本地重构了任务控制链。
* 在复杂项目中，主进程担任 Coordinator（协调者），负责总体进度监督、用户意图解析、任务拆解以及决定是否要分配下一个 Worker 代理。
* 这极大地减少了超大任务下的记忆雪崩效应，也意味着它具备类似 “Agent Swarm” （智能体群集）的局部原型实现。

### 4. 上下文缓存与提示词魔法 (`src/utils/messages.ts`, 各模块 `prompt.ts`)
在这里你可以直接领略到 Anthropic 家内部是如何“喂”大模型的：
* **Context Collapsing (上下文折叠/微缩)**：当对话非常长导致 Token 超出限制时，工具通过启发式算法保留关键执行堆栈、裁剪冗余步骤回显。
* **Prompt Engineering**：仔细翻阅每个 Tool 目录下的 `prompt.ts`！例如 `AskUserQuestionTool/prompt.ts`，你可以学到如何用最精简且不易跑飞的结构化标记语言限制模型的撒野。

### 5. 本地权限分类与防越权隔离系统 (`src/utils/permissions/`)
为了防止 Agent 失控把电脑删了，它设计了三重安全防线：
* **基线隔离**：高危指令（如 `rm -rf`, `git push`）进入强校验名单。
* **AI YOLO Classifier**：除了静态规则拦截，这里还有一个用小型 / 高速分析模型实现的智能分类器（`yoloClassifier.ts`）。如果用户在设置中开启自动同意模式（`--yes` / Yolo mode），每次执行未授权的高危操作前，这个本地逻辑会将行为上下文秘密抛给分类器，如果判断是有害的，分类器会实时拦截行为。

### 6. 通讯与远端协议栈 (`src/services/`)
与原厂服务端交互的神经中枢。
* `api/claude.ts`: 处理大模型最底层的 API 调用，包含了详尽的错误重试、指数级退避方案与流控制解析。
* `analytics/`: 监控上报与全埋点打点机制。

---

## 🛠 给开发者的魔改及学习建议

由于这是通过 Source Map 逆向出来的提取包，部分构建流程中内联的（Inlined）、被树抖动（Tree-shaking）剔掉或由 `bun:bundle` 引入的文件存在残缺，因此：
🚨 **直接运行 `ts-node src/entrypoints/cli.tsx` 将会导致一连串级联模块报错。在现版本源码环境下试图跑出一个未经任何构建步骤打补丁的原生可用的执行端是完全不现实的。**

如果你想开发/定制属于你的功能：

1. **研究“圣经”**：将本项目用作代码搜索与学习设计的**全能活字典**。
2. **文本级补丁修改法**：
   * 在本源码这里搜寻到你想改的模块或提示词语句。
   * 用文本编辑器找到那个 13MB 左右的官方运行时包（`package/cli.js`）。
   * 直接在原本打包好（无残缺）但难以肉眼追踪的大文件中，利用你在这里找到的关键字做替换打补丁！
3. **抽取优良血统**：这里的 `src/tools/` 子文件夹里包含的所有实现逻辑都可以毫无阻力地抄下来封装进你自己的 LangChain 或普通 Agent 中，作为强大的行动插件使用。

---

*“理解世界顶尖 AI 产品突破工程瓶颈的解题思路，从翻阅它的底层构建开始。”*
