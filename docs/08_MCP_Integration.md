# MCP (Model Context Protocol) 集成揭秘

Claude Code 恐怕是目前市面上**最早也最原生**的 MCP (Model Context Protocol) 协议重度实践者之一。如果你仔细看 `src/services/mcp/` 目录，你会发现 Anthropic 团队花了大把心血来构建与外部服务器的连接总线。

## 什么是 MCP 在这里的具体应用？

在没有 MCP 以前，如果你想让你的命令行客户端支持连结 Slack、读取 Jira、调用 Notion 接口，你必须在 `claude-code` 本体仓库里用 TypeScript 写死对应的 API Tool。

但通过 MCP，Claude Code 实现了一个“外挂式服务器中枢”。

## 核心实现图景

1. **`MCPConnectionManager.tsx` (连接大总管)**：
   这是整个生命周期的核心。它监听着配置，能够通过子系统拉起本地的 MCP 脚本（基于 stdio 通讯）或者基于网络的 SSE。这里还包含了极其稳健的重连机制、断线心跳包处理（这也是经常出问题或者卡顿的灾区，官方做了很多兜底机制）。
2. **`client.ts` (标准化客户端)**：
   一旦拉起了 MCP Server，客户端就会把服务器声明的“工具池” (mcpResources) 直接透明代理并入当前 Claude 大模型的原生 `tools` 列表里。
   这就意味着：大模型在它的思维链视角里，它根本分不清哪个功能是 `claude-code` 源码自带的 BashTool，哪个功能是你临时安装进来的外置 MCP 插件。它都是一视同仁直接触发的。
3. **`auth.ts` / `xaaIdpLogin.ts` (企业级鉴权网关)**：
   这也是开源项目很难做到的部分——他们做了一套非常标准的 OAuth/IdP 体系集成，允许在命令行中发起网页鉴权拦截，处理各种 Token 刷新和过期。因为有的极其核心的内部资源必须要确保“人类坐在这里授权”才能跑。
4. **长按和环境透传 (`envExpansion.ts`)**：
   这是一个特别有趣且精妙的模块。在配置外接 MCP 进程的时候，你需要传递环境变量。Anthropic 没有用傻傻写死系统字典的方式，而是做了智能的 Bash 环境展开支持，这样开发者可以写 `$MY_NOTION_TOKEN`，系统拉起服务时会自动完成安全替换。

## 为什么要研究这段逻辑？

如果你未来想把任意一个普通 Agent 升级为一个能“连接万物”的企业级枢纽，照抄 `src/services/mcp/` 中基于 `@modelcontextprotocol/sdk` 的双向通信实现与异常捕获策略，能让你省掉无数个踩坑的夜晚。这是跨进程 Agent 间对话的基础設施。
