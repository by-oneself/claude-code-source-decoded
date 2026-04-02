# Claude Code - Decompiled Source Code (v2.1.88)

> **声明 / Disclaimer**：
> 本仓库的代码来源于官方发行的 `@anthropic-ai/claude-code` CLI 工具（版本 2.1.88）安装包中的 Source Map (`cli.js.map`) 逆向提取。
> 本项目仅作逆向安全研究、代码架构学习和调试使用，所有的版权均归 Anthropic 所有。请勿用于商业用途。
>
> 官方主页: [https://claude.com/product/claude-code](https://claude.com/product/claude-code)

---

## 📖 项目简介

本仓库包含了 Claude Code（Anthropic 官方首款基于终端界面的 Agentic 编码工具）的 TypeScript 核心业务逻辑的源码。
虽然部分由打包器（Bun）动态生成的代码或受跨平台宏隔离的死代码在提取中未能完全还原（比如缺少 `node_modules` 包配置、丢失的 `TungstenTool` 等在提取时被 Tree-shaking 清理的文件），但这依然是一份含金量极高的大语言模型（LLM）客户端/代理工程架构参考代码。

### 为什么这份源码很有价值？
Claude Code 不同于传统的一个简单的脚本工具，它内置了一整套极度成熟的 LLM 调度引擎、权限隔离、Agent 协同和终端 React 混合 UI渲染机制。
对于所有有志于开发高级 Agent、自动驾驶编程工具、终端交互程序的开发者，它提供了一份接近工业界前沿天花板的实战示范。

---

## 🏗 代码架构导读

核心源码位于 `src` 目录下，以下是本仓库关键模块的代码结构解析：

### 1. `src/tools/` (系统工具箱)
存放了供给 Claude 模型调用的“手和脚”，可以说是 Agent 最核心的动作组件：
* **工具类型（Primitives）**：`BashTool` (终端执行)、`FileReadTool/FileWriteTool/FileEditTool` (文件修改)、`GlobTool/GrepTool` (代码搜索匹配)。
* **MCP / Remote 相关**：`MCPTool`、`ListMcpResourcesTool`、`WebFetchTool` / `WebSearchTool` 等提供更宽泛的拓展能力。
* **分级套娃代理**：`AgentTool` 用于支持大模型唤起一个“子模型代理”去解决特定任务。

### 2. `src/coordinator/` (多 Agent 协同)
展示了 Anthropic 如何处理主副 Agent 间的协同调度。包含主控模式下的任务发派、中止和调度逻辑设计。

### 3. `src/components/` & `src/screens/` (终端 React TUI 界面)
基于 [Ink](https://github.com/vadimdemedes/ink) 框架编写的终端用户交互界面。
* 包含了控制台的聊天气泡、提示警告 (`Message.tsx`)、历史会话分页、加载与思考状态旋转器 (`Spinner.js`)。
* 完美展示了如何利用 React 组件周期来控制流式的 LLM 回答在命令行里的平滑渲染。

### 4. `src/services/` (后端服务通讯)
与 Anthropic 及 Claude API 后端接口对话的核心逻辑：
* 遥测/数据分析 (`analytics`)、性能跟踪。
* LLM 模型对话请求 (`api/claude.ts` 等)。
* 用户配置、云端设置同步。

### 5. `src/utils/` (工具链集合)
一个庞大的技术基础设施工具库：
* `permissions/` 核心亮点：演示了极其完备的本地权限管理（哪些命令需要弹窗询问，哪些可以静默执行）。其中包含基于大语言模型或启发式规则的 `yoloClassifier`（分类器安全预测）。
* `claudeInChrome/` MCP 与 Chrome Browser 集成的远程访问服务逻辑。
* 极度健壮的终端错误拦截 (`errors.ts`)。

### 6. `src/entrypoints/` (启动入口)
* `cli.tsx` - CLI 工具的主引导程序入口。

---

## 🛠 补充说明

由于这是通过 Source Map 逆向出来的提取包：
1. **直接运行会缺依赖报错**：它不具备原始项目的 `package.json`，同时也强依赖于打包器 [Bun](https://bun.sh/) 专有的 `bun:bundle` 构建时宏。直接 `ts-node src/entrypoints/cli.tsx` 会缺失部分由于 Tree-shaking 而未能提取出来的文件以及三方依赖。
2. **正确的魔改姿势**：建议使用本仓库作为**代码搜索活字典**，精准查找到你想改的函数或提示词（例如 `src/utils/messages.ts` 或是具体的提示词文件）后，去官方原始发布的 13MB 大体积 `cli.js` 对应位置利用搜索进行“直接文本补丁替换”。

---

*“理解世界顶尖 AI 产品的解题思路，从翻阅它的底层构建开始。”*
