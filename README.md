# Modularize — Cache-Aware Task Decomposition for Reasonix

> 把大任务拆成独立模块，每个模块作为 Reasonix subagent 执行，共享一份字节稳定的 SYSTEM prompt。DeepSeek 前缀缓存跨模块持续命中。

## 这是什么

Modularize 是一个 [Reasonix](https://github.com/esengine/DeepSeek-Reasonix) Skill。

使用 Reasonix 做大型编码任务时，会话遵循 Append-Only 规则以保持前缀缓存稳定。但超长会话下上下文持续膨胀，生成越来越慢，注意力越过历史时容易丢失早期决策。

Modularize 换了一种做法：**任务开始前就拆分**。先分析依赖关系、定义每个模块的接口契约，然后每个模块作为隔离的 Reasonix subagent 独立执行。所有 subagent 共享同一份 `prefix.md`（项目概述 + 编码规范，不含任何模块专属内容），DeepSeek 的 SYSTEM prompt 字节始终不变，前缀缓存在跨模块执行时保持命中。

## 安装

```bash
git clone https://github.com/suan2inf/reasonix-modularize-skills.git ~/modularize-tmp
cp ~/modularize-tmp/SKILL.md ~/.reasonix/skills/modularize.md
mkdir -p ~/.claude/skills/modularize
cp -r ~/modularize-tmp/* ~/.claude/skills/modularize/
rm -rf ~/modularize-tmp
```

重启 Reasonix 后生效。使用：

```
reasonix code /path/to/project

/skill modularize 描述你的任务
```

## 工作流

五个阶段，模型按 SKILL.md 中的指令自动流转：

| 阶段 | 做了什么 | 产出 |
|------|---------|------|
| **1. 分析** | 读任务描述 → 画出模块依赖图（DAG）→ 用户确认 | `plan.json` |
| **2. 契约** | 为每个模块生成 TypeScript 接口契约 + 行为描述 + 错误语义 → 汇总为共享前缀 | `contracts/*.md`, `prefix.md` |
| **3. 调度** | Kahn 拓扑排序 → 划分执行波次 → 用户确认 | `plan.json` 补充 waves |
| **4. 执行** | 每个模块封装为 `runAs: subagent` skill，在隔离上下文中实现并测试 | 代码 + `result.md` |
| **5. 验证** | 逐个比对契约 vs 实际产出，检查上下游接口对齐 | 通过 / 标记失败模块 |

## Vibe Coding 实验

以下是在两个规模的项目上纯自然语言驱动的测试结果。**每次只输入一行任务描述，确认每一阶段输出，不手动修改代码**。

### 小型：CLI 笔记工具

| 指标 | 数据 |
|------|------|
| 模块数 | 5（models / storage / commands / cli / main） |
| 源文件 | 5 个 `.ts` 文件 |
| 代码量 | 496 行 |
| 总 prompt token | ~40,000 |
| 缓存命中率 | 98% |
| `tsc --noEmit` | ✅ 通过 |
| 运行时 | add / list / search 正常 |

### 中型：Task Manager REST API

| 指标 | 数据 |
|------|------|
| 模块数 | 6（database / validation / auth / tasks / tags / api） |
| 源文件 | 23 个 `.ts` 文件 |
| 代码量 | 1,319 行 |
| 总 prompt token | 4,389,645 |
| 缓存命中率 | 97.1% |
| `tsc --noEmit` | ✅ 通过 |
| 依赖安装 | 11 个 npm 包（均纯 JS，无 native addon） |

## 已知问题

### 子 agent 原生模块编译卡死

**表现**：子 agent 内执行 `npm install better-sqlite3` 时，node-gyp 编译原生模块挂起无响应，导致该模块执行中断。

**根因**：Reasonix 子 agent 运行环境对原生 C++ 模块编译的支持不稳定。`better-sqlite3` 等需要 node-gyp 的包可能在子 agent 中不正常工作。

**现状缓解**：
- Phase 2 契约要求优先纯 JS 包，避免 native addon
- 中型测试中由 `better-sqlite3` 切换为 `sql.js`（纯 JS SQLite 实现）未出现问题
- Pre-flight 阶段主 agent 统一执行一次 `npm install`，子 agent 不再独立安装依赖

**未根治**：根本解决需要 Reasonix 子 agent 运行时本身对原生模块编译提供可靠支持。

### 子 agent 工具调用开销

**表现**：子 agent 完成一个模块平均 20-25 轮工具调用，每轮一次 API 往返。相同工作放在主 agent 直接完成只需数轮。

**根因**：子 agent 每轮需重新探索上下文、读取契约和上游产出，主 agent 已持有全部上下文。

**现状缓解**：契约放在 SYSTEM prompt（而非 USER）中，后续子 agent 可命中前缀缓存。但工具调用次数未实质减少。

---

## 缓存机制

```
Subagent A (database)                   Subagent B (auth)
┌────────────────────────┐             ┌────────────────────────┐
│ SYSTEM:                 │             │ SYSTEM:                 │
│   prefix.md             │  ← 完全相同 → │   prefix.md             │  ← 字节不变
│   (项目概述+编码规范)    │             │   (项目概述+编码规范)    │     缓存命中
├────────────────────────┤             ├────────────────────────┤
│ USER:                   │             │ USER:                   │
│   contracts/db.md       │             │   contracts/auth.md     │  ← 每模块不同
│   "实现 database"        │             │   outputs/db/result.md   │     不影响
└────────────────────────┘             │   "实现 auth"            │     缓存
                                        └────────────────────────┘
```

prefix.md 仅有项目概述和编码规范（不含模块专属内容），所有 subagent 的 SYSTEM prompt 完全相同，DeepSeek KV Cache 跨 session 命中。

## 文件结构

```
~/.reasonix/skills/modularize.md      ← Reasonix Native 格式
~/.claude/skills/modularize/          ← Claude 兼容格式（完整文件）
├── SKILL.md                           ← 自包含工作流
├── README.md
├── architecture.drawio
├── phases/
├── templates/
│   ├── contract.template.md
│   ├── prefix.template.md
│   └── result.template.md
└── reference/

.reasonix/modules/                     ← 运行时产出（自动生成）
├── prefix.md                           ← 共享前缀
├── plan.json                           ← 依赖图 + 波次 + 状态
├── contracts/                          ← 接口契约
└── outputs/                            ← 模块执行结果
```

## License

MIT
