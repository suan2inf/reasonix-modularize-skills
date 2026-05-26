# Modularize — Cache-Aware Task Decomposition for Reasonix

> 大任务 → 拆成独立模块 → 每个模块作为 subagent skill 执行 → 共享 SYSTEM 前缀让 DeepSeek 缓存跨模块命中。

## 为什么需要

Reasonix 的 Append-Only 会话配合 DeepSeek 前缀缓存虽然省 token，但超长会话的代价是：
- 上下文膨胀 → 生成速度线性退化
- 注意力稀释 → 模型遗忘早期信息
- 修正历史 → 缓存立即失效，费用暴增

**Modularize 的解法**：任务开始前就拆成独立模块，每个模块作为 Reasonix subagent 执行。
subagent 有独立上下文、不污染主会话。最关键的是：**所有模块的 SYSTEM prompt 完全一致**（基于 prefix.md），DeepSeek 的前缀缓存跨 subagent 持续命中。

## 缓存机制（核心）

```
┌─────────────────────────────────────────────┐
│ SYSTEM prompt（所有模块完全一致 → 缓存命中）  │
│ ┌─────────────────────────────────────────┐ │
│ │ prefix.md  ← 项目概述 + 规范，字节不变    │ │
│ └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│ USER prompt（每模块不同，不破坏 SYSTEM 缓存） │
│ ┌─────────────────────────────────────────┐ │
│ │ contracts/{self}.md  ← 本模块接口契约    │ │
│ │ outputs/{dep}/result.md ← 上游产出摘要   │ │
│ │ "实现 auth 模块"                         │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘

prefix.md 字节不变 → DeepSeek 只需计算新增 USER 内容的 KV → 缓存命中
```

关键区分：**契约不在 prefix.md 里**——prefix.md 只含项目概述和规范（通常 < 100 行），确保精简且字节稳定。契约放在 USER message 中，不影响缓存命中。

## 命令参考

| 命令 | 阶段 | 说明 |
|------|------|------|
| `/decompose <任务描述>` | Phase 1 | 分析任务 → 拆成模块依赖图 → 用户确认 |
| `/contracts` | Phase 2 | 为每个模块生成接口契约（TS 签名 + 行为描述） |
| `/schedule` | Phase 3 | 计算执行波次（拓扑排序） |
| `/execute <模块>` | Phase 4 | 以 subagent 模式执行指定模块 |
| `/execute-all` | Phase 4 | 按波次依次执行全部模块 |
| `/integrate` | Phase 5 | 验证所有模块产出是否符合契约 |
| `/status` | 任意 | 查看进度和状态 |
| `/revise <模块>` | 异常恢复 | 重新执行失败模块及其下游（不波及无关模块） |

## 与 Reasonix 原生系统的关系

Modularize 利用 Reasonix 的三个原生能力：

| Reasonix 能力 | Modularize 如何使用 |
|---------------|---------------------|
| **Skill 系统 + subagent** | 每个模块封装为 `runAs: subagent` 的 skill，独立上下文执行 |
| **Memory（`remember`/`recall_memory`）** | 模块执行时把关键决策写入 project memory → 下游模块自动继承 |
| **缓存优先循环（Pillar 1）** | prefix.md 作为字节稳定的 SYSTEM prompt，实现跨 subagent 缓存命中 |

## 路由表

执行前判断当前阶段，只加载对应 phase 指令文件：

```
状态               → 加载文件
─────────────────────────────────────
/decompose        → phases/01-analyze.md
/contracts        → phases/02-contracts.md
/schedule         → phases/03-schedule.md
/execute*         → phases/04-execute.md
/integrate        → phases/05-integrate.md
/revise           → phases/04-execute.md + 05-integrate.md
/decompose <子>    → phases/01-analyze.md（递归拆分）
```

## 文件结构

```
.reasonix/skills/modularize/      ← 本 Skill（手动安装）
├── SKILL.md                       ← 本文件（入口 + 路由）
├── phases/                        ← 按需加载的阶段指令
│   ├── 01-analyze.md
│   ├── 02-contracts.md
│   ├── 03-schedule.md
│   ├── 04-execute.md              ← 核心：subagent 编排
│   └── 05-integrate.md
├── templates/                     ← 产出模板
│   ├── contract.template.md        ← 模块契约模板
│   ├── prefix.template.md          ← 共享前缀模板
│   └── result.template.md          ← 模块产出摘要模板
├── reference/                     ← 参考文档
│   ├── dependency-graph.md         ← DAG 算法说明
│   └── contract-format.md          ← 契约 DSL 规范
└── examples/
    └── blog-system.md

.reasonix/modules/                 ← 任务运行时产出
├── prefix.md                       ← 共享前缀（字节稳定）⚡
├── plan.json                       ← 依赖图 + 波次 + 状态机
├── contracts/                      ← 各模块接口契约（放入 user message）
│   ├── database.md
│   └── auth.md
├── outputs/                        ← 各模块执行结果
│   ├── database/
│   │   └── result.md
│   └── auth/
│       └── result.md
└── skills/                         ← 自动生成的模块 subagent skill（安装到这里）
    ├── modularize-database.md
    └── modularize-auth.md
```

## 核心原则

1. **prefix.md 字节稳定** — 不含时间戳、会话 ID、动态路径、模块专属内容
2. **契约在 USER 不在 SYSTEM** — contract.md 放入 user message，不打破 SYSTEM 缓存
3. **优先用 Reasonix 原生机制** — subagent 执行模块、Memory 传递知识
4. **失败只修故障模块** — 集成失败时仅重跑问题模块及其下游
5. **按需加载** — 执行 Phase 4 时不加载其他 phase 指令，保持上下文精简
