# Modularize — Cache-Aware Task Decomposition for Reasonix

> **大任务 → 拆成独立模块 → 每个模块作为 Reasonix subagent 执行 → SYSTEM prefix 字节稳定 → DeepSeek 前缀缓存跨模块持续命中**

## 这是什么？

Modularize 是一个 [Reasonix](https://github.com/esengine/DeepSeek-Reasonix) Skill。它把一个大型编码任务拆成多个独立的功能模块，每个模块在隔离的 subagent 中执行，同时共享一份字节稳定的 SYSTEM prompt，让 DeepSeek 的前缀缓存机制在跨模块执行时依然持续命中。

**一句话**：把"一个超长会话跑到底"变成"N 个小会话各干各的，但缓存一份也不会浪费"。

## 实测效果

> **测试项目**：CLI 命令行笔记工具，5 个模块（models → storage → commands → cli → main），6 个子 agent 会话

| 指标 | Modularize | 单体长会话（预估） |
|------|-----------|-------------------|
| 总上下文 | **~40K token** | 150K+ token |
| 缓存命中率 | **98%** | 随会话线性下降 |
| `tsc --noEmit` | ✅ 零错误 | — |
| 运行时 | add/list/search 全通 | — |

5 个独立模块跑完仅 40K token，缓存命中 98%。同等任务用单体长会话，上下文轻松破 150K，且缓存命中率随堆叠历史持续衰减。

## 解决什么问题？

### Reasonix 的困境

Reasonix 的核心理念是 **Append-Only 会话 + DeepSeek 前缀缓存**。在短任务上效果极佳——94% 缓存命中率，4 亿 token 才花 12 美元。

但长任务有三重陷阱：

| 问题 | 原因 | 后果 |
|------|------|------|
| **上下文膨胀** | Append-Only 只追加不删除，历史越积越多 | 生成速度从 50 tok/s 掉到 20 tok/s，注意力越过历史时 O(n) |
| **注意力稀释** | 50K token 上下文中，模型"遗忘"第 5K token 的决策 | 后期代码与前期设计自相矛盾 |
| **修正即重算** | 一旦修改早期对话，SYSTEM prompt 字节变化，整个 KV Cache 失效 | 那轮费用暴增，后续会话缓存重新开始冷启动 |

### 传统"阶段总结"为什么不行

常见的做法是"跑一段，让 AI 总结一段，新会话以总结开头"。

问题在于：**总结丢失执行细节**。下游模块需要知道上游用了哪个库、接口签名是什么、为什么选这个方案而不是那个。模糊的摘要里没有这些。

### Modularize 的解法

**任务开始前就拆分**，而不是任务跑完再总结。

1. 先分析所有模块依赖 → 定义每个模块的接口契约（精确的 TypeScript 签名 + 行为描述）
2. 每个模块在自己的 subagent 中执行，上下文只包含：项目概述 + 自己的契约 + 上游模块的实际产出
3. 核心：所有 subagent 的 **SYSTEM prompt 完全相同**（来自 `prefix.md`），DeepSeek 只需缓存一次，后续 subagent 全部命中

## 缓存机制（核心创新）

```
Subagent A (database)                    Subagent B (auth)
┌────────────────────────┐              ┌────────────────────────┐
│ SYSTEM:                 │              │ SYSTEM:                 │
│   prefix.md             │   ← 相同 →   │   prefix.md             │  ← 字节不变
│   (项目概述+编码规范)    │              │   (项目概述+编码规范)    │     缓存命中！
├────────────────────────┤              ├────────────────────────┤
│ USER:                   │              │ USER:                   │
│   contracts/db.md       │              │   contracts/auth.md     │  ← 每模块不同
│   "实现 database"        │              │   outputs/db/result.md  │     不影响缓存
└────────────────────────┘              │   "实现 auth"            │
                                        └────────────────────────┘

prefix.md 不含模块专属内容 → 所有 subagent 的 SYSTEM 完全一致 → DeepSeek KV Cache 跨模块命中
```

关键区分：**契约不在 prefix.md 里**。prefix.md 只有项目概述和编码规范（< 100 行）。契约放在 USER message 中，不影响 SYSTEM 字节稳定性。

## 安装

```bash
# 克隆到 Reasonix 技能目录
git clone https://github.com/suan2inf/reasonix-modularize-skills.git ~/modularize-tmp

# 安装为 Reasonix Native 格式
cp ~/modularize-tmp/SKILL.md ~/.reasonix/skills/modularize.md

# 同时保留完整文件（Claude 兼容格式）
mkdir -p ~/.claude/skills/modularize
cp -r ~/modularize-tmp/* ~/.claude/skills/modularize/

rm -rf ~/modularize-tmp
```

重启 Reasonix 后生效。验证：

```
/skill list          # 应看到 modularize
/skill modularize    # 手动调用
```

## 用法

### 启动 Reasonix

```bash
reasonix code /path/to/your-project
```

### 开始一个新任务

直接自然语言描述即可（模型读取 modularize skill 后自动触发）：

```
帮我拆分一个任务：写一个带用户注册登录、文章发布、评论功能的博客系统
```

或者显式调用：

```
/skill modularize 写一个带用户注册登录的博客系统
```

### 完整工作流

```
用户: 帮我拆分一个任务：写带注册登录的博客系统

AI: 我分析了你的任务，建议拆成以下模块：
    ┌── database ──┐
    │              ├──► auth ──┐
    └──────────────┘           ├──► api ──► frontend
                               │
    ┌── storage ───────────────┘
    └──────────────┘
    确认？

用户: OK

AI: 开始生成契约...

    --- database 模块契约 ---
    ## 类型定义
    export interface DB { ... }
    ## 接口签名
    export function connect(uri: string): Promise<DB>
    export function query<T>(sql: string): Promise<T[]>
    ...
    确认？

用户: OK

AI: [继续生成 auth / storage / api / frontend 契约...]
    全部契约已确认。prefix.md 已生成。

    执行计划（共 4 波）：
    Wave 0: database, storage
    Wave 1: auth
    Wave 2: api
    Wave 3: frontend
    确认？

用户: OK

AI: Wave 0 开始...
    ✅ database 完成 — 接口全部符合契约
    ✅ storage 完成 — 接口全部符合契约
    
    Wave 1 开始...
    ✅ auth 完成
    
    Wave 2 开始...
    ✅ api 完成
    
    Wave 3 开始...
    ✅ frontend 完成
    
    集成验证通过 ✓  — 所有模块符合契约

用户: 完美
```

### 异常恢复

```
AI: 集成验证报告
    ✅ database — 全部匹配
    ⚠️ auth    — register 签名多了可选参数 name
    ⚠️ api     — 调用了 auth 未声明的函数

用户: /skill modularize /revise auth

AI: 检查 auth 契约是否需要修正...
    契约正确，是 auth 的实现偏离了。
    重新执行 auth subagent...
    ✅ auth 已修正
    自动继续执行 api...
    ✅ api 已修正
    集成验证通过 ✓
```

## 五阶段详解

| 阶段 | 做什么 | 输入 | 输出 |
|------|--------|------|------|
| **Phase 1: 分析** | 任务 → 模块依赖图（DAG） | 任务描述 | `plan.json` |
| **Phase 2: 契约** | 每个模块生成接口契约 + 项目前缀 | 模块列表 | `contracts/*.md`, `prefix.md` |
| **Phase 3: 调度** | Kahn 拓扑排序 → 执行波次 | 依赖图 | waves 计划 |
| **Phase 4: 执行** | 每个模块作为 subagent 执行 | 契约 + 上游产出 | 模块代码 + `result.md` |
| **Phase 5: 验证** | 契约 vs 实际产出比对 | 所有 contracts + results | 通过 / 修复 |

## 文件结构

```
~/.reasonix/skills/modularize.md      ← Reasonix Native 格式（入口）
~/.claude/skills/modularize/          ← Claude 兼容格式（完整文件）
├── SKILL.md                           ← 自包含工作流（模型直接读取）
├── README.md
├── architecture.drawio                ← 架构图（draw.io 打开）
├── phases/                            ← 各阶段详细指令（可选参考）
├── templates/                         ← 产出模板
│   ├── contract.template.md            ← 模块契约模板
│   ├── prefix.template.md              ← 共享前缀模板
│   └── result.template.md              ← 模块产出摘要模板
├── reference/                         ← 参考文档
│   ├── dependency-graph.md             ← Kahn 算法详解
│   └── contract-format.md              ← 契约 DSL 规范
└── examples/

.reasonix/modules/                     ← 任务运行时产出（自动生成）
├── prefix.md                           ← 共享前缀（字节稳定 ⚡）
├── plan.json                           ← 依赖图 + 波次 + 状态
├── contracts/                          ← 各模块接口契约
│   ├── database.md
│   └── auth.md
└── outputs/                            ← 各模块执行结果
    ├── database/
    │   └── result.md
    └── auth/
        └── result.md
```

## 设计原则

| 原则 | 说明 |
|------|------|
| **prefix.md 字节稳定** | 不含时间戳、会话 ID、动态路径、模块专属内容 |
| **契约在 USER 不在 SYSTEM** | contract.md 放入 user message，不破坏 SYSTEM 缓存 |
| **失败只修故障模块** | 集成失败时仅重跑问题模块及其下游 |
| **契约先于实现** | 所有接口在 Phase 2 冻结，Phase 4 严格遵循 |
| **利用 Reasonix 原生能力** | subagent 隔离执行 + Memory 传递知识 + Pillar 1 缓存循环 |

## 与现有方案的对比

| 方案 | 拆分时机 | 缓存利用 | 模块间通信 | 适用场景 |
|------|----------|----------|------------|----------|
| **Modularize** | 开始前 | **主动设计** | 精确契约 | Reasonix 长任务 |
| OpenSpec | 事后归档 | 不考虑 | Spec 文档 | Claude Code 流程管控 |
| 阶段性总结 | 运行中 | 被动 | 模糊摘要 | 临时上下文压缩 |
| 一直开着 | 不拆 | 前期命中后期失效 | 全量历史 | 短任务 |

## 为什么只支持 Reasonix

- **Skill 系统 + `runAs: subagent`** — 天然的模块隔离载体，无需自己实现调度
- **Pillar 1（Cache-first loop）** —与 Modularize 的 prefix.md 策略深度共振  
- **Memory 系统** — `remember`/`recall_memory` 解决跨模块知识传递
- **不做多工具兼容** — 工具本身的机制才是最优载体，抽象层只会增加复杂度

## 依赖

- [Reasonix](https://github.com/esengine/DeepSeek-Reasonix) ≥ 0.51.0
- DeepSeek API Key
- Node.js ≥ 22

## License

MIT
