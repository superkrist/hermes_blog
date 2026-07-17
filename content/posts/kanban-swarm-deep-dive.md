---
title: "Hermes Agent v0.16 Kanban Swarm 功能深度解读"
date: 2026-07-17T12:35:00+08:00
draft: false
tags:
  - Hermes Agent
  - Kanban Swarm
  - 多智能体
  - AI工具
categories:
  - 技术分享
author: "superkrist"
description: "深入解读 Hermes Agent v0.16 的 Kanban Swarm 多智能体编排功能——架构、用法与设计哲学"
toc: true
---

> 当你的 AI 编码助手开始像一个真正的开发团队那样运作——有人设计方案，有人并行实现，有人 review，有人汇总——而你只需要一行命令。

想象一个场景：你要调研 2026 年 AI 融资趋势并写一份报告。你希望一个 researcher 去搜集数据、一个 analyst 去做市场分析、一个 reviewer 核查事实，最后让一个 writer 润色输出。在传统工具里，你需要手动在不同角色间切换 prompt，或者自己写 Python 脚本串联多个 LLM 调用。但在 Hermes Agent v0.16 中，一条 CLI 命令就够了。

这就是 **Kanban Swarm**——Hermes Agent 内置的多智能体编排原语。它不是外部框架，不是插件，而是深植于 Hermes 核心的分派引擎中的一等公民。本文将带你深入理解它的架构设计、使用方法和设计哲学。

---

## 一、Kanban Swarm 是什么？

官方定义非常简洁：Kanban Swarm v1 是 Hermes Agent 内置的多智能体编排原语——一条 CLI 命令即可创建完整的 **planning root → parallel workers → gated verifier → synthesizer** 任务拓扑图，所有任务状态持久化在 SQLite 中，由 gateway 内嵌的 dispatcher 自动分派到指定 profile 的 worker 进程。

把它翻译成人话：**你可以用一条命令描述一个目标，Kanban Swarm 自动拆成多个子任务，分配给不同的 AI profile 并行执行，然后由一个验证者把关、一个综合者汇总输出。** 整个过程由 SQLite 驱动的看板系统管理状态，crash 了会自动重试，完成了会自动推进下一步。

它解决的核心痛点是：多步骤、多角色的 AI 工作流，之前在代码层面需要大量胶水代码和状态管理，现在变成了声明式配置。

---

## 二、架构设计：一张图看懂 Swarm 拓扑

Kanban Swarm v1 的拓扑结构是一个标准的树形流水线：

```
planning root (done immediately)
├─ parallel specialist workers (ready)
│   ├─ worker-1: researcher
│   ├─ worker-2: analyst
│   └─ worker-3: data-collector
└─ verifier (todo → ready when all workers done)
    └─ synthesizer (todo → ready when verifier done)
```

### 2.1 四层角色的职责

| 角色 | 职责 | 触发时机 | 输出 |
|------|------|----------|------|
| **root** | 共享黑板 + 审计锚点 | 创建即完成 | 结构化 JSON comments |
| **worker** | 并行执行独立子任务 | 创建时立即就绪 | 各自的 completion metadata |
| **verifier** | 验证所有 worker 输出，以 `{"gate": "pass"}` 放行 | 所有 worker done | gate pass/fail 信号 |
| **synthesizer** | 整合最终交付物 | verifier pass | 最终成果（报告/代码/文档） |

### 2.2 为什么这个拓扑设计很重要？

1. **并行加速**：worker 之间没有依赖，dispatcher 会同时分派它们到不同进程中执行。三个 worker 各跑 2 分钟，总耗时是 2 分钟而不是 6 分钟。

2. **质量门控**：verifier 不是摆设。它在所有 worker 完成后才运行，能看到完整的并行输出，做出全局判断。如果 gate fail，整个 swarm 会阻塞等待人工介入——不会悄悄产出有问题的结果。

3. **共享黑板机制**：worker 之间不直接通信，而是通过 root task 上的结构化 JSON comments 交换信息。每条 comment 以 `[swarm:blackboard] ` 为前缀，后写覆盖前写。这避免了多 agent 直接对话带来的 token 爆炸，同时保持了信息的可追溯性。

### 2.3 状态机与容错

Kanban Swarm 运行在 Hermes 的 Kanban 状态机之上，核心状态流转如下：

```
triage → todo → ready → running → done
                              → blocked (需要人工介入)
                              → crashed → reclaim → ready (自动重试)
                              → gave_up (熔断)
```

几个关键的容错机制：

- **心跳检测**：在跑 worker 需周期性发送 heartbeat（默认 4 小时无心跳即回收），dispatcher 会定期扫描 stale worker 并重新分派。
- **Circuit breaker**：连续失败达到阈值（默认 2 次）后自动熔断为 `gave_up`，防止无限重试消耗 API 费用。
- **Dependency promotion**：verifier 的 parents 是所有 worker ids，synthesizer 的 parents 是 verifier。SQLite 层面的 parent→child 边保证只有所有 parent 完成，child 才自动晋升为 ready。这是纯数据驱动的状态推进，不需要任何调度器去轮询检查"worker 们跑完了吗"。

---

## 三、v0.16 带来了什么？（与 v0.15 的关键差异）

v0.15 "The Velocity Release" 首次引入了 Kanban Swarm。v0.16 "The Surface Release" 在此基础上做了三个关键增强：

### 3.1 `goal_mode` — 让 worker 自动迭代直到完成

这是 v0.16 最亮眼的新特性。普通 task 是一次性执行：worker 启动，跑一轮，输出结果，结束。这意味着复杂的子任务（比如"调研 AI 市场格局"）可能一轮跑不完。

`goal_mode` 解决了这个问题：当 task 标记为 `goal_mode: true` 时，worker 进入 `/goal` 循环模式——每轮执行后由一个内置 judge 检查完成度，如果未完成且 budget 未耗尽，自动继续下一轮。这相当于给每个 worker 装了一个"干到满意为止"的驱动力。

配置方式：

```yaml
# config.yaml
kanban:
  goal_max_turns: 20  # goal_mode worker 的最大轮次
```

### 3.2 文件附件 — 让图片和文档进入 worker 的视野

v0.16 新增了 task 级别的文件附件支持。更酷的是：**图片附件会自动注入到 worker 的 vision 上下文中**。这意味着你可以：

- 给 researcher 附带一份 PDF 参考文档
- 给 designer 附带一张设计稿截图
- 给 reviewer 附带原始需求截图

这在之前需要手动把文件内容塞进 task body，现在直接 attach 即可，dispatcher 会在 spawn worker 时自动处理。

### 3.3 更精细的并发与技能控制

| 维度 | v0.15 | v0.16 |
|------|-------|-------|
| 默认 assignee | ❌ 必须显式指定 | ✅ `default_assignee` fallback |
| 每 profile 并行上限 | 无控制 | `max_in_progress` 可配 |
| 技能加载 | 静态 | 环境门控 + 可裁剪 |
| synthesizer 默认技能 | `avoid-ai-writing`（不存在！） | `humanizer`（内置） |

最后一行值得单独提一下：v0.15 的 synthesizer 默认技能 `avoid-ai-writing` 是一个**不存在的技能**，这导致所有 swarm 的 synthesizer 在启动时静默失败。v0.16 修复为内置的 `humanizer` 技能，这是一个典型的 dogfooding 过程中发现并修复的问题。

---

## 四、实战：从命令行到生产配置

### 4.1 最小可用示例

```bash
# 调研 AI 融资趋势
hermes kanban swarm "Research AI funding landscape in 2026" \
  --worker researcher:"Collect and analyze AI funding data" \
  --worker analyst:"Analyze market trends for top 5 VC firms" \
  --verifier reviewer \
  --synthesizer writer
```

这条命令会：
1. 创建一个 root task（立即标记 done）
2. 创建两个 worker task（assignee: researcher, analyst），状态 ready
3. 创建一个 verifier task（assignee: reviewer），状态 todo，等待两个 worker 完成
4. 创建一个 synthesizer task（assignee: writer），状态 todo，等待 verifier 完成

### 4.2 带技能和优先级的进阶用法

```bash
hermes kanban swarm "Build authentication module" \
  --worker backend-dev:"Design database schema":requesting-code-review \
  --worker frontend-dev:"Build login UI" \
  --verifier reviewer \
  --synthesizer writer \
  --priority high
```

注意 `--worker` 的格式：`profile:title[:skill,skill]`。技能以逗号分隔，会强制加载到对应 worker 的上下文中。

### 4.3 生产级 config.yaml

```yaml
kanban:
  dispatch_in_gateway: true          # dispatcher 运行在 gateway 内（默认）
  dispatch_interval_seconds: 60      # 分派间隔
  failure_limit: 2                   # circuit breaker 阈值
  dispatch_stale_timeout_seconds: 14400  # 4h 无心跳回收

  # 并发控制 — v0.16 新增
  max_in_progress: 3                 # 每 profile 最多并行 worker
  default_assignee: default          # 未指定 assignee 时的 fallback

  # 自分解 — v0.15 引入
  auto_decompose: true               # triage 任务自动分解
  orchestrator_profile: default      # 分解后的父任务 assignee
```

### 4.4 典型使用场景

| 场景 | 配置思路 |
|------|----------|
| **单人开发流水线** | design → implement → test 三步依赖链，三个 role 实际指向同一 profile |
| **多 worker 并行农场** | 翻译、转录、文案三个 profile 同时从共享 board 拉任务 |
| **调研 → 报告** | researcher 调研 → verifier 核查 → writer 汇总 |
| **代码 review 闭环** | PM spec → engineer implement → reviewer reject → retry → pass |
| **熔断恢复** | worker 崩溃 → dispatcher 检测 → reclaim → 重新分派 |

---

## 五、与竞品的定位差异

把 Kanban Swarm 放到多智能体编排的大图景里看，它的定位非常独特：

| 维度 | Kanban Swarm (Hermes) | CrewAI | AutoGen (Microsoft) | LangGraph (LangChain) |
|------|----------------------|--------|---------------------|----------------------|
| **设计哲学** | 声明式拓扑 + 持久化状态 | 角色扮演团队 | 对话驱动多 agent | 有向图 + 类型化状态 |
| **编排方式** | 固定四层拓扑 (root→worker→verifier→synth) | 用户自定义 (sequential/hierarchical/consensual) | 用户自定义对话流 | 用户定义 state graph |
| **状态持久化** | ✅ SQLite 原生，跨 session | 单次 transcript | 单次对话 | checkpoint 可选 |
| **启动方式** | 一条 CLI 命令 | Python 脚本 | Python 脚本 | Python 脚本 |
| **容错** | 心跳 + 熔断 + 自动 reclaim | 需自行实现 | 需自行实现 | 需自行实现 |
| **用户界面** | Telegram/WhatsApp/Discord/CLI | 需自行构建 | 需自行构建 | 需自行构建 |
| **适合场景** | 个人/小团队工作流自动化 | 企业级多 agent 生产线 | 研究型对话实验 | 生产级 agent 工作流 |

### 关键洞察：Kanban Swarm 不做的事情

它**不是**一个通用多 agent 框架。你不会在 Kanban Swarm 里定义任意拓扑图、自定义 agent 间的对话协议、或者实现复杂的条件路由。它的设计是"把最常见、最可靠的多 agent 模式固化为原语"。

CrewAI 给你一个灵活的剧组，你可以导演任意剧本，但你需要自己搭舞台、卖票、管后勤。Kanban Swarm 给你一个预制好的流水线，你只需要挂上合适的人，开动机器。**前者灵活但重，后者约束多但轻。**

### Hermes Agent 的独特语境

还有一个容易被忽略的区别：Kanban Swarm 运行在 Hermes Agent 这个大语境中。这意味着：
- 它天然享有 Hermes 的持久记忆（memory + session search）
- 它可以被 Telegram 消息触发
- 它的结果可以直接通过 gateway 投递到你的聊天窗口
- 你可以随时通过 `kanban comment` 中途干预

而 CrewAI 或 LangGraph 通常需要一个外层服务来包装这些能力。

---

## 六、已知限制与展望

### 6.1 当前限制

1. **固定拓扑**：Swarm v1 只有四层。你不能自定义更多层级或分支条件。如果需要更复杂的 DAG，你得用 `kanban_create` 手动构建或使用 `delegate_task` 并行。

2. **硬编码的技能引用**：verifier 默认技能硬编码为 `requesting-code-review`，synthesizer 为 `humanizer`。自定义 body 和 skills 的支持在 issue #34273 追踪中。

3. **SQLite 单机限制**：不支持跨主机共享 kanban.db。这意味着你没法把 worker 分布到多台机器上（除非通过 SSH backend）。

4. **Profile 不存在时静默丢弃**：如果你在 `--verifier` 里指定了一个不存在的 profile，task 会卡在 ready 状态永远不执行。务必先在 `hermes profile list` 确认。

5. **auto_decompose 的安全争议**：triage 任务会立即分解并分派 worker，没有给用户留下填写上下文的时间窗口。

### 6.2 未来方向

从 v0.17 "The Reach Release" 和社区 issue 来看，几个明确的演进方向：

- **`delegate_task(background=True)`**（v0.17 已落地）：允许在一个 session 内异步派发子任务，结果自动回到对话中。这为 Swarm 内部的 worker 提供了更灵活的委托能力。
- **自定义 verifier/synthesizer 配置**：社区呼声很高，预计未来版本会放开。
- **`/swarm` slash command**：在聊天界面通过 slash command 创建 swarm（issue #35600）。
- **跨主机的 Kanban 同步**：这是走向生产级部署的关键一步。

---

## 七、总结

Kanban Swarm v1 代表了一种务实的多智能体设计哲学：**不是给你一个无限灵活的画布让你画任意拓扑，而是把最可靠的四层流水线固化为一条 CLI 命令。**

它的核心价值在于：
- **零胶水代码**：从"想法"到"并行多 agent 执行"只有一行命令
- **可靠性内建**：心跳、熔断、自动 reclaim、依赖晋升都是内置的
- **状态持久化**：所有任务状态在 SQLite 中，crash 后能恢复
- **与 Hermes 生态无缝集成**：memory、gateway、messaging、skills 全部可用

如果你想在个人工作流中引入多智能体协作——调研、代码 review、多语言翻译、内容生产——又不想引入外部框架的复杂度，Kanban Swarm 是目前最轻量的入口。它不是 CrewAI 或 LangGraph 的替代品，但当你需要的是"一个可靠的流水线"而不是"一个可编程的剧院"时，它正中靶心。

---

*本文基于 Hermes Agent v0.16.0 (2026.6.5) 版本撰写，源码参考 [hermes_cli/kanban_swarm.py](https://github.com/NousResearch/hermes-agent/blob/main/hermes_cli/kanban_swarm.py) (278 lines)。*
