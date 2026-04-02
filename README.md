# Claude Code Source Dump (v2.1.88)

这是从 Anthropic 官方 `@anthropic-ai/claude-code` npm 包里通过 source map 逆向提取出来的 TypeScript 源码。

> **说明**：代码版权归 Anthropic 所有。本项目只做研究和学习交流，别拿去干违规的事。

## 简介
官方平时发版只给了一个打包好的 13MB 多的大文件 `cli.js`。不过通过解包工具处理里面遗留的 source map (`cli.js.map`)，就可以还原出这套目录结构极其完整的原始 TS 代码。

虽然因为打包器（Bun）的 tree-shaking 策略，一些被跨平台宏屏蔽的代码、以及没用到的依赖在解包过程中丢了，导致无法直接开箱编译运行，但这套源码拿来扒架构、抄 Prompt 是再好不过的“活字库”了。

## 目录结构 & 挖宝指南

代码全在 `src/` 下面，随便列几个值得翻一翻的模块：

* **`src/tools/`**
  核心工具链。重点可以看看 `BashTool` 是怎么做后台常驻进程和命令超时的；还有个神级套娃设计 `AgentTool`，Claude 遇到复杂问题时会调用这个工具生出一个带独立环境的 Sub-Agent（子代理）去解决局部分支任务。
  
* **`src/coordinator/`**
  专门处理主 Agent 和子 Worker 之间协同逻辑的调度器，大项目开发中通过这种方式能有效防止对话过长导致的上下文爆炸。

* **`src/components/` & `src/screens/`**
  用的 Ink 框架写的 React 命令行 UI。如果好奇像打字机一样带有进度条的复杂动画在命令行里怎么防闪烁，这里的组件逻辑是满分作业。

## 核心架构拆解文档 (深度研究建议必读)

为了方便学习，我们在 `docs/` 目录下对 Claude Code 最核心的八大黑科技做了极致详尽的代码剖析和解读，强烈建议去翻一翻：

1. [`01_Architecture.md`](./docs/01_Architecture.md)：React TUI 无闪烁渲染与主循环事件流
2. [`02_Prompts_And_Context.md`](./docs/02_Prompts_And_Context.md)：Context Collapsing 上下文抗爆炸折叠算法
3. [`03_Tools_Sandbox.md`](./docs/03_Tools_Sandbox.md)：基于 Node PTY 的沙盒工具调用与进程隔离
4. [`04_Security_Permissions.md`](./docs/04_Security_Permissions.md)：高危操作时的拦截网与 Yolo 轻量分类器机制
5. [`05_LLM_API_Services.md`](./docs/05_LLM_API_Services.md)：退避重试网络通讯与端到端统计 Telemetry
6. [`06_Directory_Reference.md`](./docs/06_Directory_Reference.md)：找补丁专用的代码目录全局字典
7. [`07_Original_System_Prompts.md`](./docs/07_Original_System_Prompts.md)：模型原厂内部组装系统提示词的满分打样
8. [`08_MCP_Integration.md`](./docs/08_MCP_Integration.md)：基于原生 Model Context Protocol 的外接服务大总管
9. [`09_Memory_And_Storage.md`](./docs/09_Memory_And_Storage.md)：利用 FileRead/Write 实现跨终端长期记忆（Memdir / Coworker 索引）

* 以上文档的深入理解可以极大提升你个人构建甚至克隆 "Devin" 级工程的能力。

## 正确的魔改姿势
如果你提取这个是想破解或加上自己的功能，**不要试图在这个仓库里直接修改并强行编译**，缺失的碎文件和第三方依赖太多，会卡在无限修 Bug 上。

最干脆的办法：
1. 把这个仓库当作“字典”。
2. 在这里全局搜索弄懂你想改的逻辑、拦截的权限或者是想要覆盖的 Prompt。
3. 打开本地 npm 中真正跑的那个 13MB 的打包文件 `cli.js`，全局搜索对应的关键字符串，直接在那打补丁替换。
