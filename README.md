# Modularize — Cache-Aware Task Decomposition for Reasonix

> 大任务 → 拆成独立模块 → 每个模块作为 Reasonix subagent 执行 → SYSTEM prefix 字节稳定 → DeepSeek 前缀缓存跨模块持续命中

## 问题

Reasonix 的 Append-Only 会话配合 DeepSeek 前缀缓存省 token，但超长会话会导致：
- 上下文膨胀 → 生成速度线性退化
- 注意力稀释 → 模型遗忘早期信息
- 修正历史 → 缓存立即失效，费用暴增

## 解法

**任务开始前**就拆成独立模块。每个模块作为一个 Reasonix subagent skill（`runAs: subagent`），拥有独立上下文。所有模块共享同一份 SYSTEM prompt（`prefix.md`），DeepSeek 前缀缓存跨 subagent 命中。

## 架构

```
System prompt (所有模块相同 → 缓存命中)
│
│  prefix.md  ← 项目概述 + 规范（字节不变）
│
User prompt (每模块不同，不影响缓存)
│
│  contracts/{module}.md  ← 接口契约
│  outputs/{dep}/result.md ← 上游产出
│
```

关键区分：**契约不在 prefix.md 里**——prefix.md 仅含项目概述和规范（< 100 行）。契约放在 user message 中，不影响 SYSTEM 的字节稳定性。

## 安装

```bash
git clone <this-repo> ~/.reasonix/skills/modularize
```

或手动复制：

```bash
cp -r modularize/ ~/.reasonix/skills/modularize
```

重启 Reasonix 后 `/skill list` 应能看到 `modularize`。

## 命令

| 命令 | 阶段 | 说明 |
|------|------|------|
| `/decompose <任务>` | Phase 1 | 分析任务 → 拆成模块依赖图 |
| `/contracts` | Phase 2 | 生成接口契约 + prefix.md |
| `/schedule` | Phase 3 | Kahn 拓扑排序 → 执行波次 |
| `/execute <模块>` | Phase 4 | Subagent 执行指定模块 |
| `/execute-all` | Phase 4 | 按波次自动执行全部 |
| `/integrate` | Phase 5 | 验证契约一致性 |
| `/status` | 任意 | 查看进度 |
| `/revise <模块>` | 异常恢复 | 仅重跑失败模块及其下游 |

## 示例

```bash
reasonix code my-blog-project
```

```
/decompose 写一个带用户注册登录的博客系统

→ AI 分析 → 展示模块依赖图 → 用户确认
→ /contracts → 逐个模块生成契约 → 确认
→ /schedule → 显示执行计划（波次）
→ /execute-all → 按波次执行

Wave 0: database, storage
Wave 1: auth
Wave 2: api
Wave 3: frontend
→ /integrate → 验证 ✓ → DONE
```

## 文件结构

```
.reasonix/skills/modularize/     ← 本 Skill（安装到这里）
├── SKILL.md                     ← 入口 + 路由
├── README.md
├── phases/                      ← 按需加载的阶段指令
├── templates/                   ← 产出模板
├── reference/                   ← 参考文档
└── examples/

.reasonix/modules/               ← 任务运行时产出
├── prefix.md                    ← 共享前缀（字节稳定 ⚡）
├── plan.json                    ← 依赖图 + 波次 + 状态
├── contracts/                   ← 各模块接口契约
└── outputs/                     ← 各模块执行结果
```

## 依赖

- [Reasonix](https://github.com/esengine/DeepSeek-Reasonix) ≥ v0.51.0
- DeepSeek API key
- Node ≥ 22

## 为什么只支持 Reasonix

- Reasonix 原生 skill 系统 + `runAs: subagent` = 天然的模块隔离载体
- Pillar 1（Cache-first loop）与 Modularize 的 prefix.md 策略深度共振
- Memory 系统（`remember`/`recall_memory`）解决跨模块知识传递
- 不做多工具兼容——工具本身的机制才是最优载体

## License

MIT
