+++
date = '2026-02-01T01:03:44+08:00'
draft = false
title = 'Karpathy: LLM 编程随笔'
tags = ['AI', 'Agent']
categories = ['随笔']
+++

> 原文来源：[Andrej Karpathy X Post (2025年12月)](https://x.com/karpathy/status/2015883857489522876)

---

## Key Takeaways

1. **编程范式发生相变** — 从 80% 手动编码转变为 80% agent 编码，仅用几周时间。这是作者近 20 年编程生涯中最大的工作流变化。

2. **LLM 像"有点粗心的初级开发者"** — 不再犯语法错误，而是犯概念性错误：擅自做假设不验证、过度复杂化代码、不清理死代码、过于谄媚不反驳。仍需像老鹰一样盯着它们。

3. **耐力成为新杠杆** — Agent 永不疲倦、永不沮丧，能在人类早就放弃的地方坚持 30 分钟最终取得胜利。耐力瓶颈被大幅突破。

4. **声明式优于命令式** — 不要告诉 LLM 做什么，给它成功标准让它循环。先写测试再实现、先写朴素算法再优化——这是获得杠杆的关键。

5. **不是加速，是扩展** — 主要效果不是做原来的事更快，而是能做以前不值得做或不会做的事。

6. **编程变得更有趣** — 苦差事被移除，剩下创造性部分。但会分化工程师：喜欢编码的 vs 喜欢构建的。

7. **2026 预测** — 垃圾内容末日年 + 10X 工程师的倍数可能大幅增长 + 通才可能超越专才。

---

## 中文翻译

### 过去几周大量使用 Claude 编程的一些随笔记录

**编程工作流。** 鉴于 LLM 编程能力的最新提升，和很多人一样，我从 11 月的 80% 手动+自动补全编程、20% agent，迅速转变为 12 月的 80% agent 编程、20% 编辑+润色。也就是说，我现在真的主要用英语编程了，有点不好意思地用文字告诉 LLM 写什么代码……用语言。这有点伤自尊，但用大块"代码动作"操作软件的能力实在太有用了，特别是一旦你适应了它、配置好它、学会使用它、理解了它能做和不能做什么。这是我近 20 年编程生涯中基础编程工作流最大的变化，而且发生在短短几周内。我预计类似的情况正发生在两位数百分比的工程师身上，而大众对此的认知感觉还停留在低个位数百分比。

**IDE/agent 集群/易错性。** 我认为"不再需要 IDE"的炒作和"agent 集群"的炒作目前都过头了。模型确实仍然会犯错，如果你有任何真正在意的代码，我建议像老鹰一样盯着它们，旁边开一个大大的 IDE。错误的类型变了很多——不再是简单的语法错误，而是微妙的概念性错误，就像一个有点粗心、急躁的初级开发者会犯的那种。最常见的一类是模型替你做出错误的假设，然后就照着跑下去而不去验证。它们也不会管理自己的困惑，不会寻求澄清，不会暴露不一致性，不会呈现权衡取舍，不会在该反驳的时候反驳，而且仍然有点过于谄媚。在 plan 模式下会好一些，但确实需要一种轻量级的内联 plan 模式。它们还特别喜欢把代码和 API 搞复杂，让抽象臃肿，不会自己清理死代码，等等。它们会用 1000 行代码实现一个低效、臃肿、脆弱的结构，然后你得说"呃，你不能直接这样做吗？"它们会说"当然可以！"然后立刻把它缩减到 100 行。它们有时还会作为副作用修改/删除它们不喜欢或不够理解的注释和代码，即使这与手头的任务完全无关。尽管在 CLAUDE.md 中做了一些简单的尝试来修复这些问题，以上情况仍然会发生。尽管有这些问题，这仍然是一个巨大的净改进，很难想象回到手动编程。总结一下，每个人都在发展自己的工作流，我目前的方式是：左边在 ghostty 窗口/标签页中开几个小的 CC 会话，右边是 IDE 用来查看代码和手动编辑。

**韧性。** 看一个 agent 不懈地工作在某件事上真是太有趣了。它们永远不会累，永远不会沮丧，它们就是不停地尝试，换作一个人早就放弃改天再战了。看着它在某件事上挣扎很长时间，最后在 30 分钟后取得胜利——这是一个"感受 AGI"的时刻。你意识到耐力是工作的核心瓶颈，而有了 LLM，这个瓶颈被大幅扩展了。

**加速。** 如何衡量 LLM 辅助带来的"加速"并不清楚。当然，我感觉在原本要做的事情上快了很多，但主要效果是我做了比原计划多得多的事情，因为 1) 我可以写出各种以前不值得写的代码，2) 我可以处理以前因为知识/技能不足而无法处理的代码。所以这当然是加速，但可能更多的是扩展。

**杠杆。** LLM 非常擅长循环直到达到特定目标，这是大部分"感受 AGI"魔力的来源。不要告诉它做什么，给它成功标准然后看它跑。让它先写测试，然后通过测试。把它放在浏览器 MCP 的循环中。先写一个很可能正确的朴素算法，然后让它在保持正确性的同时优化。把你的方法从命令式改为声明式，让 agent 循环更久，获得杠杆。

**乐趣。** 我没有预料到使用 agent 编程会让编程感觉*更有趣*，因为很多填空式的苦差事被移除了，剩下的是创造性的部分。我也感觉更少被阻塞/卡住（这不好玩），而且体验到更多勇气，因为几乎总有办法和它携手合作取得一些正向进展。我也从其他人那里看到了相反的感受；LLM 编程会把工程师分成主要喜欢编码的人和主要喜欢构建的人。

**退化。** 我已经注意到我手动写代码的能力正在慢慢退化。生成（写代码）和判别（读代码）是大脑中不同的能力。主要由于编程中涉及的所有小的、大多是语法层面的细节，你可以很好地审查代码，即使你难以写出它。

**垃圾末日。** 我正在为 2026 年做准备，这将是 github、substack、arxiv、X/instagram 以及所有数字媒体上垃圾内容末日之年。我们还会看到更多 AI 炒作式的生产力表演（这还可能吗？），与此同时也有真正的实际改进。

**问题。** 我脑海中的几个问题：
- "10X 工程师"会怎样——平均和最强工程师之间的生产力比例？这个比例很可能会增长*很多*。
- 装备了 LLM，通才是否会越来越超越专才？LLM 更擅长填空（微观）而不是宏观战略。
- 未来 LLM 编程会是什么感觉？像玩星际争霸？玩异星工厂？演奏音乐？
- 社会有多少是被数字知识工作所瓶颈的？

**总结：这让我们处于什么位置？** LLM agent 能力（特别是 Claude 和 Codex）在 2025 年 12 月左右跨越了某种连贯性阈值，在软件工程及相关领域引发了相变。智能部分突然感觉远远领先于其他一切——集成（工具、知识）、新组织工作流的必要性、流程、更广泛的扩散。2026 年将是高能量的一年，整个行业将消化这种新能力。

---

## 英文原文

A few random notes from claude coding quite a bit last few weeks.

**Coding workflow.** Given the latest lift in LLM coding capability, like many others I rapidly went from about 80% manual+autocomplete coding and 20% agents in November to 80% agent coding and 20% edits+touchups in December. i.e. I really am mostly programming in English now, a bit sheepishly telling the LLM what code to write... in words. It hurts the ego a bit but the power to operate over software in large "code actions" is just too net useful, especially once you adapt to it, configure it, learn to use it, and wrap your head around what it can and cannot do. This is easily the biggest change to my basic coding workflow in ~2 decades of programming and it happened over the course of a few weeks. I'd expect something similar to be happening to well into double digit percent of engineers out there, while the awareness of it in the general population feels well into low single digit percent.

**IDEs/agent swarms/fallability.** Both the "no need for IDE anymore" hype and the "agent swarm" hype is imo too much for right now. The models definitely still make mistakes and if you have any code you actually care about I would watch them like a hawk, in a nice large IDE on the side. The mistakes have changed a lot - they are not simple syntax errors anymore, they are subtle conceptual errors that a slightly sloppy, hasty junior dev might do. The most common category is that the models make wrong assumptions on your behalf and just run along with them without checking. They also don't manage their confusion, they don't seek clarifications, they don't surface inconsistencies, they don't present tradeoffs, they don't push back when they should, and they are still a little too sycophantic. Things get better in plan mode, but there is some need for a lightweight inline plan mode. They also really like to overcomplicate code and APIs, they bloat abstractions, they don't clean up dead code after themselves, etc. They will implement an inefficient, bloated, brittle construction over 1000 lines of code and it's up to you to be like "umm couldn't you just do this instead?" and they will be like "of course!" and immediately cut it down to 100 lines. They still sometimes change/remove comments and code they don't like or don't sufficiently understand as side effects, even if it is orthogonal to the task at hand. All of this happens despite a few simple attempts to fix it via instructions in CLAUDE.md. Despite all these issues, it is still a net huge improvement and it's very difficult to imagine going back to manual coding. TLDR everyone has their developing flow, my current is a small few CC sessions on the left in ghostty windows/tabs and an IDE on the right for viewing the code + manual edits.

**Tenacity.** It's so interesting to watch an agent relentlessly work at something. They never get tired, they never get demoralized, they just keep going and trying things where a person would have given up long ago to fight another day. It's a "feel the AGI" moment to watch it struggle with something for a long time just to come out victorious 30 minutes later. You realize that stamina is a core bottleneck to work and that with LLMs in hand it has been dramatically increased.

**Speedups.** It's not clear how to measure the "speedup" of LLM assistance. Certainly I feel net way faster at what I was going to do, but the main effect is that I do a lot more than I was going to do because 1) I can code up all kinds of things that just wouldn't have been worth coding before and 2) I can approach code that I couldn't work on before because of knowledge/skill issue. So certainly it's speedup, but it's possibly a lot more an expansion.

**Leverage.** LLMs are exceptionally good at looping until they meet specific goals and this is where most of the "feel the AGI" magic is to be found. Don't tell it what to do, give it success criteria and watch it go. Get it to write tests first and then pass them. Put it in the loop with a browser MCP. Write the naive algorithm that is very likely correct first, then ask it to optimize it while preserving correctness. Change your approach from imperative to declarative to get the agents looping longer and gain leverage.

**Fun.** I didn't anticipate that with agents programming feels *more* fun because a lot of the fill in the blanks drudgery is removed and what remains is the creative part. I also feel less blocked/stuck (which is not fun) and I experience a lot more courage because there's almost always a way to work hand in hand with it to make some positive progress. I have seen the opposite sentiment from other people too; LLM coding will split up engineers based on those who primarily liked coding and those who primarily liked building.

**Atrophy.** I've already noticed that I am slowly starting to atrophy my ability to write code manually. Generation (writing code) and discrimination (reading code) are different capabilities in the brain. Largely due to all the little mostly syntactic details involved in programming, you can review code just fine even if you struggle to write it.

**Slopacolypse.** I am bracing for 2026 as the year of the slopacolypse across all of github, substack, arxiv, X/instagram, and generally all digital media. We're also going to see a lot more AI hype productivity theater (is that even possible?), on the side of actual, real improvements.

**Questions.** A few of the questions on my mind:
- What happens to the "10X engineer" - the ratio of productivity between the mean and the max engineer? It's quite possible that this grows *a lot*.
- Armed with LLMs, do generalists increasingly outperform specialists? LLMs are a lot better at fill in the blanks (the micro) than grand strategy (the macro).
- What does LLM coding feel like in the future? Is it like playing StarCraft? Playing Factorio? Playing music?
- How much of society is bottlenecked by digital knowledge work?

**TLDR** Where does this leave us? LLM agent capabilities (Claude & Codex especially) have crossed some kind of threshold of coherence around December 2025 and caused a phase shift in software engineering and closely related. The intelligence part suddenly feels quite a bit ahead of all the rest of it - integrations (tools, knowledge), the necessity for new organizational workflows, processes, diffusion more generally. 2026 is going to be a high energy year as the industry metabolizes the new capability.
