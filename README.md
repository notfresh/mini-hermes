# Mini-Hermes — 从 Hermes 蒸馏最小 Agent 的两代实践

> 本项目不做代码，只做**总览和**：把两代"从 Hermes 剥离的最小 Agent"项目串成一条学习路线，讲清楚每一代蒸馏了什么、实现了什么、成就了什么。
> 本项目也会记录作者的新功能的设计思路。

🔖 **agent 原理系列（进行中）** —— 讲清 Agent 的基本概念与机制。第一篇：[Skill vs Plugin：内容与机制的分界线](docs/principle-20260901-skill-vs-plugin.md)。上下文专题已并入本系列，见下。

🔖 **上下文专题（进行中 · agent 原理系列子专题）** —— 深攻方向：搜索 / RAG / agent 记忆——上下文工程是Agent更加高效的方向。

- 总纲（顶级思考）：[基于生命周期识别的记忆管理](docs/thinking-20260826-lifecycle-based-memory-management.md) 
- 落地：[记忆分层规则引擎：决策思路复盘](docs/design-20260826-memory-rule-engine-decision.md)  
- 业界剖析：[Mem0 核心原理](docs/analysis-20260826-mem0-core-principles.md) · [Letta Code](docs/analysis-20260826-letta-core-principles.md) · [Graphiti/Zep](docs/analysis-20260826-graphiti-core-principles.md)
- Hermes 自身：[记忆系统综述（内置双文件 vs 外部插件）](docs/review-20260826-hermes-memory-system.md)
- 学术综述：[Agent 记忆 2026 综述导读（三维度框架 × 六挑战）](docs/analysis-20260826-agent-memory-survey-2026.md)

> 专题登记见文末 [七、专题登记](#七专题登记)

---

- **第一代** · [minimal-agent (mini-hermes-v1)](https://github.com/notfresh/mini-hermes-v1) — 单文件最小 Agent，验证"核心循环 5 行逻辑"
- **第二代** · [minimal-agent-v2 (mini-hermes-v2)](https://github.com/notfresh/mini-hermes-v2) — 五模块拆分的教学级 Agent 框架，逐步补齐 Hermes 进阶机制



## 一、这两个项目解决什么问题（价值）

**背景**：Hermes Agent 是一个 15 万行 Python 的生产级 Agent 框架，核心循环 `agent/conversation_loop.py` 就有 5194 行。直接读源码很容易迷失在并发、流式、回退链、上下文压缩等防御机制里，看不到本质。

**做法（蒸馏）**：从 Hermes 核心执行流程中剥离出最小骨架——把 5194 行的核心循环压缩到 30 行，把防御逻辑逐层还原成可单独讲解的模块。每个教学模块都有一张对照表，精确指向 Hermes 源码的对应文件与函数。

**做法（实现）**：不只是读，而是**自己动手实现一个最小可运行的 Agent**，用真实 LLM（DeepSeek 优先）跑通真实任务，验证蒸馏结论正确。

**学习路径**：蒸馏（从复杂系统抽骨架）→ 对比（与生产代码逐行对照）→ 重建（自己实现并验证）——两条主线：

| 主线 | V1 完成度 | V2 完成度 |
|------|-----------|-----------|
| 最小可用 Agent | ✅ 单文件跑通真实任务 | ✅ 五模块重构 |
| 工具系统（@tool 注册） | ✅ 7 个内置工具 | ✅ 继承 + 守卫检查点 |
| 技能框架（外置技能包） | ❌ | ✅ skills-framework 分支 |
| 任务规划（Plan Mode） | ❌ | ✅ 软约束 V1 → 硬约束 V2 |
| 权限控制（防越权） | ❌ | ✅ AgentIgnore 分支 |
| 会话管理 / REPL | ❌ | ✅ session_manager + REPL |

---

## 二、第一代：minimal-agent（V1）

🔗 GitHub：[https://github.com/notfresh/mini-hermes-v1](https://github.com/notfresh/mini-hermes-v1)（tag `v0.1`）

### 一句话定位

> **Agent = LLM + Tools + Loop**，核心循环就 5 行逻辑：

```python
while True:
    response = llm(messages, tools)    # 把上下文发给模型
    if response.tool_calls:            # 模型要调工具？
        execute(response.tool_calls)    # 执行，结果回添
        continue                       # 送回 LLM 继续
    return response.content            # 纯文本回复 → 完成
```

### 项目形态

```
minimal-agent/
├── minimal_agent.py    # 核心循环 + CLI 入口（~130 行）
├── tools.py            # @tool 注册中心 + 7 个内置工具（~200 行）
└── README.md
```

- 7 个内置工具：`read` / `write` / `ls` / `grep` / `find` / `head` / `bash`
- DeepSeek 优先，支持 `--provider openai` 切换
- `--verbose` 可看每一轮消息与 token 用量

### 第一代成就

1. **验证核心结论**：Agent 的本质就是"LLM ↔ Tools 循环"，5 行逻辑可以跑通真实任务（真实 API、真实工具调用）。
2. **建立对照表**：教学版与 Hermes 生产版逐项对应——`@tool` 装饰器 ↔ `tools/registry.py`、`agent_loop()` ↔ `agent/conversation_loop.py::run_conversation`、`_execute_tool_call()` ↔ `agent/tool_executor.py`。这份对照是后续所有工作的地基。
3. **明确"剥离了什么"**：串行执行 vs Hermes 并发 `_execute_tool_calls_concurrent`、无流式 vs `_interruptible_streaming_api_call`、无回退链 vs `_try_activate_fallback`——用"教学版没有的清单"反衬 Hermes 的工程复杂度。
4. **产出 hard-vs-soft-mode 软/硬约束对比 demo**（后移入 V2 仓库，成为 Plan Mode 教学的起点）。
5. **锁定 v0.1**：第一版教学用例正式冻结。

---

## 三、第二代：minimal-agent-v2（V2）

🔗 GitHub：[https://github.com/notfresh/mini-hermes-v2](https://github.com/notfresh/mini-hermes-v2)（tag `v2.0`）

### 一句话定位

在 V1 基础上按**五模块拆分**重构：核心骨架保持 ~30 行，防御逻辑全部下沉到模块，然后沿 Hermes 进阶机制逐分支补齐能力。

### 项目形态（12 个 .py）

```
minimal-agent-v2/
├── conversation_loop.py   # ConversationLoop 骨架（~30 行）
├── turn_context.py        # TurnContext：回合前准备（系统提示词/清洗/凭证预检）
├── loop_controller.py     # LoopController：还继续吗？（轮数/预算/中断/grace）
├── llm_client.py          # LLMClient：模型说什么？（重试/退避/错误分类）
├── tool_runner.py         # ToolRunner：工具查找/解析/防御/回填（+ 守卫检查点）
├── message_store.py       # MessageStore：消息增删改查/压缩
├── session_manager.py     # 会话 CRUD + 生命周期 + 延迟持久化
├── plan_mode.py           # PlanMode 状态机
├── skill_registry.py      # 外部技能扫描 + bootstrap 检测
├── agent_ignore.py        # AgentIgnore 路径权限校验
├── tools.py / cli.py      # 内置工具 / CLI + REPL
└── docs/                  # 设计文档体系（见下）
```

### 五模块各自回答的问题

| 模块 | 回答的问题 |
|------|-----------|
| LoopController | "还继续吗？"（轮数/预算/中断/grace） |
| LLMClient | "模型说什么？"（重试/退避/错误分类，调用方看不到） |
| ToolRunner | "工具结果是什么？"（查找/解析/防御/回填） |
| MessageStore | "消息放哪/太长怎么办？"（增删改查/压缩） |
| TurnContext | "回合开始前准备什么？"（系统提示词/清洗/凭证预检） |

### 与 Hermes 源码的对照

| V2 模块 | Hermes 对应 |
|---------|-------------|
| `ConversationLoop`（骨架） | `agent/conversation_loop.py :: run_conversation`（5194 行） |
| `TurnContext` | `agent/turn_context.py` + `agent/prompt_builder.py` |
| `LoopController` | `agent/iteration_budget.py` + 循环退出条件 |
| `LLMClient` | `agent/chat_completion_helpers.py`（重试/退避/错误分类） |
| `ToolRunner` | `tools/registry.py` + `agent/tool_executor.py` |
| `MessageStore` | 循环内 messages 操作（压缩/快照/持久化） |
| `skill_registry` | `skills.external_dirs` + `skill_view` 工具 |

### 分支演进史 —— 每一代（每个分支）的成就

V2 用**分支当版本线**，一条分支 = 一个进阶能力，拓扑：`main → skills-framework → plan-mode-v1 → plan-mode-v2 → agent-ignore`。

**① main — 五模块拆分（地基）**
- 把 V1 单文件拆成五模块 + CLI，核心骨架 ~30 行，防御逻辑全部下沉。
- 会话管理：每会话一个独立 JSON（`~/.minimal-agent-v2/sessions/`），**延迟持久化**——聊了第一条真实消息才落盘，没聊过的新会话不产生任何文件。
- REPL 交互（`/list`、`/switch`、`/new`、`/delete`）、上下文压缩（`compress_if_needed`）。
- 设计与报告文档体系：多任务并发路线图（结论 = 当前不实现，YAGNI）、三分支总结、Plan Mode 实现报告。

**② skills-framework — 技能框架插口（对齐 Hermes 技能系统）**
- V2 作为宿主提供通用技能插口，对应 Hermes 的 `skills.external_dirs` + `skill_view`。
- 技能内容完全外置：独立的 MinimalSuperPowers 技能包（蒸馏自 obra/superpowers + kimi-code）通过 `--skills-dir` 挂载。
- 宿主约定：技能索引注入 `<available_skills>`、`using-superpowers` 总开关全文只注入一次、`load_skill` 工具按需加载技能全文。
- 去重设计来自 kimi 源码：`_bootstrap_injected` 标记 + 消息 `origin.kind == "injection"` 检测（REPL 会话恢复场景）。
- 不挂任何技能包 = V2 原版行为（`--no-skills` 显式关闭）——插口零侵入。

**③ plan-mode-v1 — Plan Mode 软约束版**
- 纯提示词规划：`plan` 工具（写计划到 `~/.minimal-agent-v2/plans/`）+ system prompt 规划规则。
- 提示词强制包含：阶段目标 + **验收条件** + 阶段验证 + **整体验证**。
- 价值：先讲清楚"规划靠提示词也能实现"，为硬约束版提供对比基线。

**④ plan-mode-v2 — Plan Mode 硬约束版（状态机 + 守卫）**
- `PlanMode` 类（is_active/plan_path）+ `enter_plan_mode`/`exit_plan_mode` 工具。
- **`plan_guard` 守卫**：规划期 `write` 只允许写计划文件，违规直接 `DENIED`；`ToolRunner` 在工具执行前加检查点，`cli.py` 注入守卫——对齐 Kimi 的 plan-mode-guard-deny。
- REPL 新增 `/plan` 手动进入、`/run` 退出（人类介入入口，V3 审批的雏形）。
- **实测黄金场景**：`/plan` 后模型直接写业务文件 → 守卫 DENIED → 模型读到错误后自动调 `exit_plan_mode` → 正常执行——软约束 vs 硬约束的最生动演示（"谁在强制"：状态在框架内存 vs 模型上下文）。

**⑤ agent-ignore — AgentIgnore 路径权限（类 .gitignore 的权限控制）**
- 语法：每行 `绝对路径::权限码`（R/W/X 子集，Linux 语义），空权限码 = 全禁。
- 匹配 = 前缀命中 + 权限交集（做减法）：父目录禁止的权限，子目录加不回来；无命中 = RWX 全允许。
- 路径归一化（`normpath(abspath())`）防 `../` 绕过；工具映射表覆盖 read/write/bash 等（bash 从 command 字符串正则提取绝对路径逐个校验）。
- 检查点与 plan_guard 并列，拒绝时回填 `错误：AgentIgnore 权限校验失败` 给模型（不崩溃，模型可见可自纠）。
- 6 组测试断言覆盖：解析容错/前缀交集/防绕过/工具拦截/ToolRunner 集成/未启用放行。

---

## 四、两代对比

| 维度 | V1（mini-hermes-v1） | V2（mini-hermes-v2） |
|------|---------------------|----------------------|
| 形态 | 单文件（minimal_agent.py ~130 行） | 12 模块拆分 |
| 核心循环 | 5 行，与工具/CLI 同函数 | 骨架 ~30 行，防御下沉到模块 |
| 工具系统 | @tool 装饰器 + 7 工具 | 继承 + ToolRunner 守卫检查点 |
| 会话 | 无 | session_manager + REPL + 延迟持久化 |
| 技能 | 无 | 外置技能包插口（skills-framework） |
| 规划 | 无 | 软约束（提示词）→ 硬约束（守卫） |
| 权限 | 无 | AgentIgnore（R/W/X 交集匹配） |
| 版本 | v0.1 | v2.0 |
| 传承 | — | "V1 单文件 → V2 五模块"，@tool/工具集/CLI/DeepSeek 优先均继承 |

---

## 五、还没做的（路线图，对应 Hermes 进阶机制）

V2 README 中挂起的扩展方向，恰好就是 Hermes 里那些"被剥离掉的防御机制"：

- [ ] 流式输出（Hermes: `_interruptible_streaming_api_call`）
- [ ] 并发工具执行（Hermes: `_execute_tool_calls_concurrent`）
- [ ] 凭证自动轮换（Hermes: `_ensure_runtime_credentials`）
- [ ] 上下文 LLM 摘要压缩（Hermes: `context_compressor`）
- [ ] 记忆读写（Hermes: `memory_manager`）
- [ ] 子代理分发（Hermes: `delegate_tool`）

---

## 六、学习路径建议

1. **V1 起步**：读 `minimal_agent.py`，把 5 行核心循环背下来——这就是所有 Agent 的本质。
2. **V2 结构化**：看五模块如何把防御逻辑拆出去，用"五模块各自回答的问题"理解职责划分。
3. **进阶三件套**（按分支顺序）：skills-framework（技能外置）→ plan-mode（软 → 硬约束）→ agent-ignore（权限）。
4. **对照源码验证**：每个模块都能在 Hermes 源码里找到对应实现，带着对照表去读，心里才踏实。

---

## 七、专题登记

### agent 原理系列

> 讲清 Agent 的基本概念与机制。上下文专题已并入本系列。

- **[Skill vs Plugin：内容与机制的分界线](./docs/principle-20260901-skill-vs-plugin.md)** — agent 原理系列·第一篇。skill 与 plugin 的界限不在文件形态，在**加载机制**：skill 是内容（模型按需读取，渐进式披露），plugin 是代码（进程启动时 import 并执行 `register()`）。两个反例打掉"文档 vs 代码"的二分：plugin 可带 skill（superpowers 以插件形式分发一百多个 SKILL.md）、skill 可带 py 脚本（本机 35 个 skill 带 scripts/）。一句话判据：有没有被 Hermes 进程 import 并执行。

### 上下文专题（agent 原理系列·子专题）

> 深攻方向：搜索 / RAG / agent 记忆——"上下文是未来的方向"。上下文工程 = 让 agent 借鉴过往的智慧：检索、记忆、上下文管理。**总纲（顶级思考）先行，落地设计随后**。

- **[基于生命周期识别的记忆管理](./docs/thinking-20260826-lifecycle-based-memory-management.md)** — 上下文专题·总纲。按"问题→结论"四轮讨论整理：长短周期怎么判断价值 → 信息管理怎么省事 → 记忆管理双因子（属性决定天生该活多久，频率决定实际用得多不多）→ 上下文工程四档调度（常驻/检索/临时/不存）。核心结论：**频率 + 属性 + 有效期**三个维度决定一条信息放哪、留多久。
- **[记忆分层规则引擎：决策思路复盘](./docs/design-20260826-memory-rule-engine-decision.md)** — 总纲的第一块落地。agent 什么该长期记住、什么只配短期停留。五个决策：判断器用规则引擎不用 LLM 黑盒（可审计）、规则锚在知识生命周期（变更成本 ≈ 生命周期 ≈ 重建成本）、分层靠淘汰策略不靠预测（以使用定生死）、实现选最简档（12 条规则 = 45 行代码，YAGNI）、提取层与判断层分离（LLM 切句、规则定层）。附完整 12 条规则清单，与总纲三维模型逐条对应。
- **[Mem0 核心原理：把 LLM 当记忆秘书的"写时增改删"](./docs/analysis-20260826-mem0-core-principles.md)** — 业界项目剖析第一篇（绑定 [mem0ai/mem0](https://github.com/mem0ai/mem0)，本地源码实测）。核心机制：对话不存原文，LLM 提取成事实条目，写时对已有记忆做 **ADD/UPDATE/DELETE**（写时思考）；读时多信号混合打分（语义门槛先筛 + BM25 + 实体加成）。四个可抄的防幻觉细节：UUID→整数映射、hash 去重、原始消息落 SQLite、OSS 版时间能力关闭。文末对照主线：Mem0=LLM 主判 vs 我们=规则主判——它的 UPDATE/DELETE 是设计文档中 P3（用户纠正覆盖）的自动化弱化版，但不可审计、无生命周期分层、写成本高。
- **[Letta Code：让 agent 自己改自己的记忆](./docs/analysis-20260826-letta-core-principles.md)** — 业界项目剖析第二篇。⚠️ 仓库事实：`letta-ai/letta` 主分支已只是落地页，V1 Python 版退役归档，**当前实现在 [letta-ai/letta-code](https://github.com/letta-ai/letta-code)**（TypeScript/Bun）。核心机制：记忆 = **Memory Blocks**（persona/human 默认块，直接拼进系统提示词，每轮常驻）；agent 用 5 个记忆工具（memory / memory_apply_patch / memory_insert / memory_replace / memory_rethink）**在对话里自己编辑记忆**；全部记忆是文件且 git 跟踪（**MemFS**，可同步私人 GitHub 仓库，每次修改即一次 commit）；`/sleeptime` 配置 periodic dreaming（定期反思，reflection 子代理）。对照主线：Letta 的自我编辑 = 四维模型中"Agent 主动探测"维度的业界极致，但"为什么改"仍是 LLM 隐式判断，无规则层宪法、无 TTL 先验。
- **[Graphiti：给知识图谱装上时间轴（Zep 的开源核心）](./docs/analysis-20260826-graphiti-core-principles.md)** — 业界项目剖析第三篇。⚠️ 仓库事实：Zep 产品 = 托管平台（zep-cloud SDK），开源核心 = [getzep/graphiti](https://github.com/getzep/graphiti)（30.3k stars），`getzep/zep` 只是示例/集成仓。核心机制：对话增量提取实体/关系（不重建图），**EntityEdge 带双时间轴**（valid_at/invalid_at = 事实成立/失效，expired_at = 系统失效，reference_time = 来源时间）→ 可"时间旅行"查询当时的事实；Episodic（原文）/ Entity（语义）/ Community（label propagation 聚类 + LLM 摘要）三层；边失效机制自动淘汰矛盾旧事实；检索 = edge/node/episode/community 四 scope × fulltext/向量/bfs 三手段。对照主线：时间轴 = 设计里 TTL+provenance 的图增强版，验证了"记忆必须带时间+来源"的判断，但依赖图数据库部署重、仍 LLM 主判。
- **[Hermes 记忆系统综述：内置双文件 vs 可插拔外部记忆](./docs/review-20260826-hermes-memory-system.md)** — Hermes 自身机制综述。内置 MemoryStore（MEMORY.md + USER.md，2200+1375 字符上限，冻结快照 + 实时双状态、防投毒扫描、合并失败降级）；记忆全生命周期（回合开始 prefetch → `<memory-context>` 注入 → memory 工具读写 → 每 10 轮后台反思 fork）；可插拔架构 = MemoryProvider ABC（19 钩子）+ MemoryManager（内置永远在，外部最多一个，失败隔离）；**8 个外部记忆插件**（honcho / mem0 / holographic / hindsight / supermemory / byterover / openviking / retaindb）。对照结论：内置双文件对个人助手级够用（有界=强制蒸馏、零依赖、防投毒），升级判据 = 量大上向量检索、要历史上时间轴、要主动进化上自我编辑。
- **[Agent 记忆 2026 综述导读：三维度框架与六个开放挑战](./docs/analysis-20260826-agent-memory-survey-2026.md)** — 学术综述导读。原文《Rethinking Memory Mechanisms of Foundation Agents in the Second Half: A Survey》（[arXiv:2602.06052](https://arxiv.org/abs/2602.06052)，59 作者，83 页，收录 218 篇 2023Q1–2025Q4）。核心：**三维度框架**（memory substrate 基板 internal/external × cognitive mechanism 五种认知机制 sensory/working/episodic/semantic/procedural × memory subject 主体 user/agent-centric）+ 六开放挑战（自进化 / 多智能体 / 效率 / **终身个性化与可信记忆** / 多模态具身 / 真实评测）。对照结论：论文验证了"记忆是调度问题"和"判定必须可审计"两个判断；你的规则引擎 = learning policies 的可解释轻量版；MemGPT=工作记忆、Mem0=语义记忆、Graphiti=情景记忆+时间，Hermes Agent 未收录（太新）——正是空白机会。原文 PDF 存 `~/.hermes/knowledge/papers/`。
