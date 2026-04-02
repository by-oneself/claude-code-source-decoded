# 架构视图 (Architecture)

Claude Code 是一个极度复杂且工程化拉满的开源级项目典范。如果你只是把 AI 客户端想象成“接受一个输入字符串，丢给 API 然后打印响应流”，那么 Claude Code 会立刻打破这种偏见。

它不仅具备现代 GUI 的组件化体系，同时拥有复杂的多路协同架构。以下是对其核心架构的深度拆分。

---

## 1. 入口与引导流程 (`src/entrypoints/`)

程序的入口为 `src/entrypoints/cli.tsx`。

在这段代码中：
1. 它首先进行版本检查、环境降级防御与各类遥测（Telemetry）的引导化。
2. 它并未使用传统的 `readline`，而是用 React 把终端当作浏览器屏幕给接管了。
3. 它挂载了由 Ink (一个使用 React 渲染命令行 UI 的框架) 提供的 `<App />` 构建整套视图，此时你的各种键盘按键事件、窗口缩放都被 Ink 接管为 React 的 `window` 事件与全局状态逻辑。

---

## 2. 终端的 DOM 生态：React + Ink (`src/components/` & `src/screens/`)

这就是为什么 Claude Code 看起来异常平滑、能回滚历史还能局部更新动画的原因。
在以前的 Node.js 脚本中，终端文字打出去就是覆水难收。而通过 Ink：

* 全局大循环状态：系统有一个 `QueryEngine`，以及全局的 `appState`（存在 Jotai 等类似的全局 Store 中，虽然这里他们有大量自定义 Hook）。
* `src/screens/` 代表了不同的屏幕模式：
  * `Chat`：普通聊天终端。
  * `REPL`：一种深度拦截机制模式。
  * 等等。
* 组件渲染：例如 `src/components/Message.tsx`。当后端的大语言模型返回一个 Streaming Chunk 时，组件的状态 `content` 发生改变，React 就会发生局部 Re-render。Ink 在底层会利用控制台的转义字符（ANSI escape codes）把屏幕上旧的代码块直接无闪烁地重写为带上高亮的最新语法树结构（通过 `src/utils/claudemd.ts` 转写 Markdown）。

---

## 3. 多代理调度大纲：Coordinator 与 Worker (`src/coordinator/`)

最具有革命性的设计在于，Claude 把单个 Agent 设计成了**分形**树结构 —— 大任务可以无限衍生子 Agent。

### 痛点是什么？
普通的 Agent 解决复杂问题时：用户让它写一个中型项目，Agent 做了一步，遇到报错去查文件，再报错再去修改……过了大概 15 步，Prompt 过长（百万 token）导致执行极度迟缓，最后还会上下文忘却导致胡言乱语（哪怕模型拥有再恐怖的极限下线）。

### Anthropic 的解法
在 `src/coordinator/` 中，可以发现存在一层“发牌逻辑”。
1. 主循环（Main Agent / Coordinator）接到任务，思考后，并不是立马自己干。
2. 当需要一个连续动作突破时，主 Agent 调用了 `AgentTool` (这个工具的实现可见 `src/tools/AgentTool/AgentTool.tsx`)。
3. 一旦调用 `AgentTool`，主进程将被 “挂起（Pending/Paused）”。
4. 一个带有独立系统上下文和生命周期的**全新子代理（Sub-Agent / Worker）**启动。它接管了终端，独立思考、调用 Bash 去跑测试跑命令。
5. 当子代理解答完毕后，子代理销毁。**主代理苏醒**，并收到一句类似“工具调用返回结论：修复完毕”作为极其凝练的一句总结。
6. 就这样，主代理的 Token 被完美释放，它可以永远保持清醒的顶层视角，不被底下的代码烂泥潭搞爆上下文。

---

## 4. 数据同步与状态留存 (`src/memdir/` 和 `QueryEngine`)

模型如何保持对用户的长久“记忆”？
Claude Code 不像一些粗糙平台把所有历史对话塞进去。它实现了一套 `memdir`（记忆目录）逻辑。

1. 系统在用户根目录（或本地项目中）悄悄维护一个压缩的向量数据库级的小文本索引（提取摘要用）。
2. 当一个工作回话（Session）关闭时，会有后台逻辑把整个 Session 的高光时刻做摘要并归档到记忆中。
3. 配合 `QueryEngine.ts`，新的输入如果涉及历史上下文，会在构建系统提示词时被动态通过语义检索灌入。这个设计可以在不暴增 Token 成本的情况下提供出色的长期记忆。
