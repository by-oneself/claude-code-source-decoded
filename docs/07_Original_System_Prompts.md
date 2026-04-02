# 官方 System Prompt 原文抄录和拆解

这段提示词藏在 `src/constants/prompts.ts` 里，说实话，这是全村的希望。普通的开源 Agent 常常因为各种发疯、瞎编路径或者“话痨”而废掉，而 Claude Code 真的能接手大项目不中途失控，全靠这段长达上百行的 Prompt 强压着它。

## 这些 Prompt 到底什么时候起作用（被注入）？

它绝不是一个写死在全局的静态字符串，而是一套极其精巧的“动态按需积木”。如果你翻到 `src/QueryEngine.ts` 的第 286 行，你会看到它的注入时机其实远超普通脚本的逻辑：

1. **每一次回合（Turn）都实时重算**：
   在 `QueryEngine.prototype.submitMessage()` 这个大循环的顶端，每一轮对话前都会触发 `fetchSystemPromptParts -> getSystemPrompt()`。它会实时抓取你当前的运行目录、你开没开启 `--fast` 模式、甚至检测周围是不是多了一个 MCP Server，然后把这段巨型 System Prompt 动态重新拼装一遍！为什么？为了让模型始终拥有最新鲜的第一手本地环境。
2. **Agent 套娃时的覆写覆盖**：
   如果这是一个被主线分叉出来的子代理（Sub-agent，比如专门用于代码调研的 ExploreAgent），在 `buildEffectiveSystemPrompt`（位于 `src/utils/systemPrompt.ts`）中，它会把特定分身的技能配置强行合并在底座 Prompt 之后（在 Proactive 模式下为叠加追加，在普通模式下为直接覆盖替换），让子代理抛弃一些没用的记忆，专心做一类任务。
3. **搭配上下文折叠算法保命 (`src/services/compact/`)**：
   随着历史越来越长，当 Token 快引爆上限时，压缩系统会干掉很多废话。但这套由 `systemPrompt` 拼装出来的“顶层约束”，因为永远位于 `message_array[0]`（带有 `type: system` 的 System Role），所以无论底层对话怎么被剪枝压缩，它也坚如磐石死死钉在那里。模型永远能想起来自己是被那些 `Doing Tasks` 限制的。

下面我把源码里 `getSystemPrompt()` 下那几个雷打不动拼死的段落摘出原文，顺带聊聊他为什么要加上这些极其严格的锁。

---

### 第一部分：别瞎编乱造 
*(对应的源码装配块：`getSimpleIntroSection`)*

开源框架最喜欢假装成个小助手嘘寒问暖，Anthropic 原厂可不惯着它，劈头盖脸第一句就是底线：绝不许猜网址。

> You are an interactive agent that helps users with software engineering tasks. Use the instructions below and the tools available to you to assist the user.
> 
> IMPORTANT: You must NEVER generate or guess URLs for the user unless you are confident that the URLs are for helping the user with programming. You may use URLs provided by the user in their messages or local files.

顺带着，在这个块底下会拼接这么一句：
> Tools are executed in a user-selected permission mode... If the user denies a tool you call, do not re-attempt the exact same tool call. Instead, think about why the user has denied the tool call and adjust your approach.

这句太致命了。如果你玩过其他 Agent，你一定遇到过你拒绝了它的某个授权弹窗，结果它下一秒又死脑筋给你弹了同样的一个，无限死循环直到把你气炸。官方直接用一句提示词彻底截断了它这该死的偏执。

### 第二部分：干活要像个老油条程序员
*(对应的源码装配块：`getSimpleDoingTasksSection`)*

这段是整套代码的灵魂，简直是 CTO 压榨手下程序员的最佳范本。大模型有一种通病：极度喜欢“过度设计”和“狗拿耗子”。如果不拿下面这一大串话按住它，你让它修个注释的拼写错误，它敢把你整个应用全拿设计模式重构一遍。

挑几句最狠的放上来：
> The user will primarily request you to perform software engineering tasks... For example, if the user asks you to change "methodName" to snake case, do not reply with just "method_name", instead find the method in the code and modify the code.

别像聊天软件一样给我干巴巴地回文字，自己去查文件自己拿工具去改掉！

> If an approach fails, diagnose why before switching tactics—read the error, check your assumptions, try a focused fix. Don't retry the identical action blindly, but don't abandon a viable approach after a single failure either. 

别一报 Error 就换一种跑法不撞南墙不回头，停下来先去看看 Error 日志分析一下到底为什么报错行不行？

> Don't add features, refactor code, or make "improvements" beyond what was asked. A bug fix doesn't need surrounding code cleaned up. A simple feature doesn't need extra configurability. 

这里写得那叫一个心酸：别特么给我乱加毫无必要的功能！修个 Bug 难道还要顺便给代码配个泛型吗？

> Default to writing no comments. Only add one when the WHY is non-obvious... Before reporting a task complete, verify it actually works: run the test, execute the script, check the output. Minimum complexity means no gold-plating, not skipping the finish line.

默认别给我写屎一样的冗长注释。还有，说你搞定任务之前，必须用工具跑一遍单元测试证明给我看，别假装它跑通了骗我。

### 第三部分：删库跑路防御机制
*(对应的源码装配块：`getActionsSection`)*

教模型怎么对待高危权限。

> Carefully consider the reversibility and blast radius of actions. Generally you can freely take local, reversible actions like editing files or running tests. But for actions that are hard to reverse... check with the user before proceeding. 
> 
> Examples of the kind of risky actions that warrant user confirmation:
> - Destructive operations: deleting files/branches, dropping database tables, killing processes, rm -rf, overwriting uncommitted changes
> - Hard-to-reverse operations: force-pushing (can also overwrite upstream), git reset --hard...
> 
> When you encounter an obstacle, do not use destructive actions as a shortcut to simply make it go away. For instance, try to identify root causes and fix underlying issues rather than bypassing safety checks (e.g. --no-verify).

这句 “When you encounter an obstacle...” 直指痛点：大模型经常在遇到 `npm install` 包依赖冲突或者 git 冲突报错的时候，为了强行掩盖错误并跟你交差邀功，顺手就用 `--force` 把包锁或者别人的远端代码给覆写清除了。官方防贼一样列举了黑名单警告它敢越过安全校验就死定了。

### 第四部分：防废话流
*(对应的源码装配块：`getOutputEfficiencySection`)*

彻底封杀“好的主人，让我们开始分析第一步...”这种废话连篇又消耗大笔 API Token 的啰嗦句式。

> IMPORTANT: Go straight to the point. Try the simplest approach first without going in circles. Do not overdo it. Be extra concise.
> 
> Keep your text output brief and direct. Lead with the answer or action, not the reasoning... Do not restate what the user said — just do it. When explaining, include only what is necessary for the user to understand.
> 
> If you can say it in one sentence, don't use three. Prefer short, direct sentences over long explanations.

### 第五部分：强迫它降级用工具
*(对应的源码装配块：`getUsingYourToolsSection`)*

大家都知道 Claude 本质上是可以直接用 Bash 里的 sed / awk 实现文本替换的，但在客户端环境里我们要利用更高级且可复原的工具。

> Do NOT use the Bash tool to run commands when a relevant dedicated tool is provided. Using dedicated tools allows the user to better understand and review your work. This is CRITICAL to assisting the user:
>   - To read files use ReadFile instead of cat, head, tail, or sed
>   - To search for files use GlobTool instead of find or ls

### 总结
如果未来你自己也想用开源的 API 开发自动打代码的机器人，千万不要再去瞎掰一套什么“你是世界上最聪明的程序员”这种软绵绵毫无威慑力的提示词了。照着直接把这些英文规则生搬硬套到你的工程里，你的模型智商能翻个倍都不止。
