# Agent 的长期记忆术：跨会话的记忆索引 (Memdir)

任何普通的 ChatGPT 对话，只要一关机一清空上下文，就成了永远不认识你的“陌生人”。而 Claude Code 是一个“Coworker”（同事），能保持数个月的项目熟悉感。秘诀都在 `src/memdir/` 这个文件包下。

## 物理存储机制

Claude Code 不是弄个高大上的向量数据库（Vector DB），而是秉持着极其纯粹极客主义精神，运用了**纯文件系统**持久化：

默认位置在：`.claude/memory/`

### 1. 索引文件：`MEMORY.md` 永远在场
系统利用一个核心文件 `MEMORY.md` 作为索引。无论当前的对话跑了多远开了什么分支方案，每次给到大模型的强制 System Prompt 里永远挂载着 `MEMORY.md` 的原文。

为了防止被撑爆，在 `src/memdir/memdir.ts` 中的 `truncateEntrypointContent` 写着：
> 如果 `MEMORY.md` 超过了 **200 行** 或者是 **25KB**，模型后端会把它无情砍掉劈成两截，并在末尾给模型追加一条强烈警告：`WARNING: MEMORY.md is [200] lines. Only part of it was loaded. Keep index entries to one line...` 迫使模型去自我精简它的索引表。 

### 2. Semantic Topic Files（主题记忆子模块）
`MEMORY.md` 里面是不存真正干货源码的，它只能写链接！
比如：
`- [user_preferences](frontend_stack.md) — The user absolutely hates Tailwind, only use Vanilla CSS.`

因为当模型看到这个指针时，模型如果有需要，会自主唤起 `FileReadTool` 去读取同级目录下的 `frontend_stack.md` 获取超长、超细致的工程记忆图谱。这是典型的 RAG (检索增强) 的原生低成本复现。

## 模型自主维护与日志记录

如果你自己维护这个文件表一定很痛苦，对吧？
错了！**这是 Agent 自主的。**
在 `teamMemPrompts.ts` 和系统内置逻辑中，当用户发出“别忘了这一点”或者大模型发现自己犯了个史诗级错误并刚刚修复后，大模型会偷偷唤起底层的 WriteTool，主动写一个 `feedback_.md` 并在 `MEMORY.md` 里插入索引。甚至，在代码中我们可以看到 `KAIROS daily-log mode` (助理日报模式)。

### 自动推演与压缩 (Daily Logs & Distillation) 
在带有特定环境变量和激活标记的场景下：
系统会自动建立 `logs/YYYY/MM/YYYY-MM-DD.md` 每日日志，在里面疯狂 Append（追加）每天用户交代下的事情，然后在深夜脚本跑批的时候（目前功能可能仍在特定旗舰组内测灰度），系统会启动一个后台闲时模型把长串的日报进行蒸馏提纯，“压缩”回主 `MEMORY.md` 和对应的标签文件里。

## 学习意义

如果你抱怨普通代理“记性不好”或者塞了太多文档导致“上下文发疯”，翻看 `memdir.ts` 里这段设计会如梦初醒：
**用强硬的 Prompt 卡死一个 Index 大小的文件在脑内作为全局路由，具体的事务让它作为动作实体用专用的搜查指针去取回。这才是现代长情境 Agent 应有的技术素养。**
