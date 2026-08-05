---
title: "从 Prompt Engineering 到 Loop Engineering：AI 交互范式的四次进化"
date: 2026-08-05T14:00:00+08:00
draft: false
tags:
- AI
- Prompt Engineering
- Loop Engineering
- Agent
- LLM
categories:
- AI工程
mermaid: true
---

> 2026年，写提示词的技能正在被「设计闭环系统」的能力取代。本文梳理从 Prompt → Context → Harness → Loop 的完整演进脉络。

---

## 一、引言：提示词已死？

2026年6月，Google 工程师 Addy Osmani 发表了一篇引发广泛讨论的文章，系统化地命名了一个新学科：**Loop Engineering（循环工程）**。同期，Claude Code 的创造者 Boris Cherny 公开表示：「我不再为 Claude 写提示词了。我运行一些循环，让循环去提示 Claude。」英伟达 CEO 黄仁勋更是直接宣布：「提示词编写已过时。」

这些信号指向同一个趋势：**AI 工程的核心正在从「如何写好提示词」迁移到「如何设计自主运行的闭环系统」**。

但这不意味着 Prompt Engineering 真的「死」了——它只是被降维了。本文将从完整演进脉络展开，梳理这条技术线的发展全景。

---

## 二、四阶段演进全景

AI 工程的核心交互范式，在三年里经历了四次根本性转变：

{{< mermaid >}}
flowchart LR
    A["<b>Prompt Engineering</b><br/>2023<br/><br/>写更好的指令<br/>角色设定 · 分步引导 · 少样本示例"]
    B["<b>Context Engineering</b><br/>2024-2025<br/><br/>管理信息环境<br/>RAG · 上下文压缩 · 知识库注入"]
    C["<b>Harness Engineering</b><br/>2025-2026<br/><br/>装上工具和安全带<br/>MCP协议 · API调用 · 沙箱执行"]
    D["<b>Loop Engineering</b><br/>2026-现在<br/><br/>设计自主闭环系统<br/>目标驱动 · 自动验证 · 停止条件"]
    A --> B --> C --> D
{{< /mermaid >}}

四次转变背后是同一个驱动力：**如何让模型从「回答问题」进化为「自主完成任务」。**

---

## 三、第一层：Prompt Engineering 的黄金时代与黄昏

### 3.1 经典范式

2023年是 Prompt Engineering 的黄金时代。当时的主流认知是：输出质量取决于提示词的精细程度。核心手段包括：

- **角色设定**：「你是一名有10年经验的后端架构师」
- **分步引导**：「先分析问题，再给出方案，最后写代码」
- **少样本示例**：在 prompt 中提供 2-3 个问答范例

### 3.2 致命局限

随着应用深入，经典 Prompt Engineering 的局限暴露无遗：

| 维度 | 问题 |
|------|------|
| **可编程性** | 不可编程，无法嵌入自动化流水线 |
| **记忆** | 无状态，每次对话从零开始 |
| **验证** | Agent 说「完成了」，但无法证明它真的做对了 |
| **规模** | 面对 100 个任务就要写 100 条 prompt |
| **一致性** | 同一 prompt，不同模型、不同时间，结果差异巨大 |

### 3.3 重新定位：从核心到接口

MIT 的研究表明，**一半的 AI 收益来自「如何提问」而非模型本身**。但这并不巩固 Prompt Engineering 的地位——恰恰相反，它揭示了一个关键洞察：真正重要的不是 prompt 文本有多精致，而是**你给模型展示了什么信息、赋予了它什么能力、以及如何验证它的产出**。

Prompt Engineering 在 2026 年的新定位是**接口层**，而非核心逻辑层。一个「最短可行指令（Minimum Viable Prompt）」的标准模板是：

1. **目标**（1-3 句话）
2. **约束**（硬性要求与禁止项）
3. **完成定义**（什么算「做完」）
4. **验证方式**（测试命令、预期输出、可机械化判定的证据）

### 3.4 版本化：Prompt 管理的工程化

2026 年的一个并行趋势是 prompt 从「手写字符串」进化为「受管理的工程资产」。**Langfuse**、**PromptLayer** 等 PromptOps 平台引入了：

- **版本仓库**：保存历史版本，支持回溯和对比
- **模板化**：变量占位 + 运行时渲染
- **批量评测**：用固定测试集自动比较不同版本效果
- **发布控制**：只有通过评测的版本才能进入生产

**提示**：类似软件工程，prompt 正在经历从「草稿」到「代码」的身份转变。

---

## 四、第二层：Context Engineering — 被低估的关键

### 4.1 核心洞察

Context Engineering 在 2024-2025 年兴起，其核心发现是：**模型实际「看到」的上下文远比你输入的 prompt 多得多**。系统提示词、对话历史、RAG 检索注入的文档、工具调用结果、项目文件——所有这些共同构成了模型的「信息环境」。

### 4.2 关键实践

- **上下文压缩**：保留关键信息，丢弃噪音（Claude Code 的 context compression pipeline 是典型案例）
- **动态注入**：通过知识库实时检索相关信息
- **系统提示词设计**：全局约束 + 行为规范
- **多 Agent 共享**：结构化信息在 Agent 间流转

### 4.3 遗留问题

Context Engineering 解决了「信息供给」问题，但留下了更深层的缺口：**信息有了，模型仍不能自主行动；上下文窗口再大，状态终究不持久。** 这直接催生了下一层。

---

## 五、第三层：Harness Engineering — 模型外面的工程结构

### 5.1 核心理念

> **模型是马，提供动力；Harness 是马具，控制方向、速度和安全。**

Harness Engineering 在 2025-2026 年走向台前，解决的核心问题是：**如何包裹模型，让它安全、可靠、可控地自主完成任务。**

### 5.2 Harness 的七大构件

| 构件 | 职责 |
|------|------|
| **Context**（信息环境） | 管理并注入模型所需信息 |
| **Orchestration**（编排） | 复杂任务拆解和编排执行顺序 |
| **Reasoning Core**（推理核心） | 模型推理与工具调用的循环 |
| **Policy & Guard**（权限与护栏） | 规定什么能做什么不能做 |
| **State**（持久状态） | 跨会话保存任务进度 |
| **Verification**（验证） | 完成任务前必须验证，防止「改完就宣称搞定」 |
| **Observability**（可观测性） | 全链路追踪、成本统计与评估 |

### 5.3 行业实践：Claude Code 的六层架构

业界分析师将 Claude Code 拆解为 6 层结构，其中**模型只是循环中的一个节点**，真正的工程价值在外层的控制结构中：

1. 用户交互层
2. 上下文管理层（压缩、注入、缓存）
3. 工具调用层（MCP、Bash、文件操作）
4. 权限与安全层
5. 循环与编排层（`/goal`、`/loop`、`/schedule`）
6. 验证与停止层（Hook、Skill、Sub-agent）

---

## 六、第四层：Loop Engineering — 2026 的核心战场

### 6.1 正式定义

> **Loop Engineering is replacing yourself as the person who prompts the agent. You design the system that does it instead.**
> —— Addy Osmani, June 2026

循环工程的核心闭环：

{{< mermaid >}}
flowchart TD
    P["<b>Plan</b><br/>规划下一步"]
    A["<b>Act</b><br/>执行动作"]
    O["<b>Observe</b><br/>观察结果"]
    V{"<b>Verify</b><br/>验证结果"}
    D["✅ 任务完成"]
    P --> A --> O --> V
    V -- "未通过" --> P
    V -- "通过" --> D
{{< /mermaid >}}

### 6.2 五大构成要素 + 记忆层

| 要素 | 说明 |
|------|------|
| **Automations**（自动化触发） | Cron、Hook、事件驱动——让循环自行启动 |
| **Worktrees**（工作隔离） | Git worktree 为每个 Agent 创建独立的文件系统视图 |
| **Skills**（可复用知识） | 代码风格、架构规范、踩过的坑——一次写入，每轮自动加载 |
| **Plugins & Connectors**（工具连接） | MCP 协议连接 Issue Tracker、数据库、API |
| **Sub-agents**（制衡分离） | 写的和检查的不是同一个 Agent |
| **+ External State**（外部状态） | Markdown 文件或项目看板记录进度，断点可恢复 |

### 6.3 七种实现模式（从简到繁）

| 模式 | 做法 | 可靠性 |
|------|------|--------|
| **① Prompt 内自检** | 在 prompt 里让模型自己验证自己 | ⭐ 最脆弱 |
| **② Ralph Loop** | 外层 bash `while` + `claude -p`，测试结果机械化判定 | ⭐⭐ |
| **③ 内置命令** | Claude Code 的 `/goal`、`/loop`、`/schedule` | ⭐⭐⭐ |
| **④ Stop Hook** | Hook 运行测试，用 exit 2 + stderr 阻止停止直到测试通过 | ⭐⭐⭐⭐ |
| **⑤ Verification Skill** | SKILL.md 定义必须完成的验证步骤 | ⭐⭐⭐⭐ |
| **⑥ Maker-Checker 子 Agent** | 独立 evaluator（无 Write/Edit 权限）做验收 | ⭐⭐⭐⭐ |
| **⑦ Agent SDK** | 外部代码控制迭代、验证、停止决策 | ⭐⭐⭐⭐⭐ |

### 6.4 从内层到外层：双层控制架构

{{< mermaid >}}
flowchart TD
    subgraph OL["🔁 Operating Loop（外层）"]
        D1["决定：继续 / 重试 / 开PR / 暂停 / 升级给人"]
        D2["「一批任务能不能被持续运营」"]
    end
    subgraph HA["⚙️ Harness（内层）"]
        D3["管上下文 · 管工具 · 管停止条件"]
        D4["「这一次任务能不能可靠推进」"]
    end
    subgraph MO["🧠 模型（推理核心）"]
    end
    OL -- "驱动" --> HA
    HA -- "包裹" --> MO
{{< /mermaid >}}

关键原则：**不要把内层问题交给外层重试，也不要把外层问题塞给内层 Agent 临场拿捏。**

---

## 七、学术脉络

Loop Engineering 并非凭空出现，其学术基础可以追溯到 2022 年：

| 时间 | 框架 | 核心贡献 |
|------|------|---------|
| 2022 | **ReAct**（姚顺宇 et al.） | 确立 Think → Act → Observe 基础周期 |
| 2023 | **Reflexion** | 引入错误反馈机制，Agent 从失败中学习 |
| 2023 | **Tree of Thoughts** | 多元路径搜索替代单线推理 |
| 2024 | **ReflAct** | 目标状态反射，世界锚定的决策 |
| 2025 | **Geometric Dynamics of Agentic Loops** | Agent 循环的几何动力学分析 |
| 2025 | **FORGE** | 无需权重更新的自进化 Agent 记忆 |
| 2026 | **Loop Engineering**（Osmani） | 系统化命名，六板块 + 工程方法论 |
| 2026 | **Meta-Harness**（Lee et al.） | Harness 本身成为可搜索、可优化对象 |

---

## 八、三条铁律

贯穿从 Prompt 到 Loop 的整个演进，有三条经实践检验的铁律：

### 8.1 Make 和 Check 必须分离

> 「同一个模型既写代码又给自己打分，等于让学生批改自己的试卷。」

生成代码用大模型，验收用独立的小模型（如 Haiku）——这是 Claude Code `/goal` 系统的核心规则，也是 Loop Engineering 的底线纪律。

### 8.2 验证必须是机械化证据

AI 的自我报告（「我感觉完成了」「看起来没问题」）不是验证。**测试通过、命令 exit 0、diff 符合预期——只有这些是可依赖的证据。** 一条直接的标准：**如果你不能事先说清楚「做完」长什么样，这个任务就不适合放进 Loop。**

### 8.3 停止条件必须显式可判定

像 `test.sh exit 0` 一样清晰，而不是「修好这个 bug」。没有硬性停止条件的循环等于一台永远运转的机器——不是在勤奋工作，而是不知道什么时候该停。

---

## 九、三大风险

### 9.1 理解债务（Comprehension Debt）

> 「你可以外包你的思考，但不能外包你的理解。」—— Andrej Karpathy

Loop 产出代码的速度远超人类审查的速度。代码合并进仓库，但没有人真正理解它是怎么工作的。**实际成本不是 Token 账单，而是未来调试的成本。**

### 9.2 认知投降（Cognitive Surrender）

当 Loop 自动运行时，开发者容易被诱惑不再自己判断，直接接受一切结果。同样一套 Loop，**有人用它加速自己透彻理解的工作，有人用它避开理解工作本身**——结果截然相反。

### 9.3 Token 成本失控

Anthropic 的内部数据：Agent 使用的 token 约为普通聊天的 **4 倍**，多 Agent 系统约为 **15 倍**。缺乏硬性停止条件的循环系统可能跑到预算耗尽。判断标准：**若验收率低于 50%，系统在亏损。**

---

## 十、行业实践概览

| 玩家 | 实践 | 特点 |
|------|------|------|
| **Anthropic** | Claude Code | 循环架构内嵌（/goal, /loop, hooks, skills），Managed Agents |
| **OpenAI** | Codex CLI | 并行最多 8 个 Agent，隔离云沙箱，合并结果 |
| **Google** | Addy Osmani 论文 | 学术定义 + 工程最佳实践 |
| **Cloudflare** | Flue | 不可变日志记录，进程死后另一个从 log 恢复 |
| **Kong** | CI Repair Agent | 拉历史 CI 日志做 signal grounding，Haiku 做验证 |
| **Datadog** | CI 自动修复 | 按类型分组，只处理高价值类别 |

开源工具生态也在快速成型：**loopspec-mcp**、**loopy**、**loom**、**harness-forge** 等项目在 GitHub 上活跃迭代。

---

## 十一、未来方向

| 方向 | 含义 |
|------|------|
| **Meta Context Engineering** | 让「如何设计上下文」本身成为 AI 可学习的技能 |
| **Intent Engineering** | 从定义「怎么做」转向定义「要什么」 |
| **Specification Engineering** | 把企业策略写成机器可读的规范 |
| **Auto-Harness / Meta-Harness** | Harness 自动合成，且 harness 本身成为可优化对象 |

自治级别应走渐进路线：先开 PR 不自动合并 → 只处理 lint/单元测试 → 不碰 migration/auth/billing 等高危路径 → 待成功率、误报率、回滚率、成本稳定后再扩大范围。

---

## 十二、结语

回顾从 Prompt 到 Loop 的四个阶段，一条主线贯穿始终：

> **AI 工程的竞争力正在从「模型能力」向外层「控制结构」迁移。** 第一阶段我们学会如何与模型对话；第二阶段学会管理模型的信息环境；第三阶段学会给模型装上工具和安全带；第四阶段学会设计让模型自主运转的闭环系统。

Prompt Engineering 没有消亡——它被吸收进了更大的体系。决定团队生产力的不再是「谁写的 prompt 更聪明」，而是「谁设计的循环更可靠」。

**人退出的不是判断，是当人肉调度器的苦役。Loop 越强，人需要越清醒。**

---

*本文基于 2026 年 8 月的公开资料与行业调研整理。*
