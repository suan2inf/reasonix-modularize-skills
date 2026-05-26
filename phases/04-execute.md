# Phase 4: 逐模块执行（核心）

> 加载时机：用户输入 `/execute <模块名>` 或 `/execute-all` 时

## 目标

将每个模块作为 Reasonix subagent skill 执行，利用：
1. `runAs: subagent` → 独立上下文，不污染主会话
2. `prefix.md` 作为 SYSTEM prompt → 跨 subagent 缓存命中
3. `remember` → 关键决策写入 project memory，下游自动获得

## 前置条件

- `plan.json` 存在，status = `ready` 或 `in_progress`
- `prefix.md` 已生成（字节稳定）
- 目标模块的 `contracts/<module>.md` 已就绪
- 目标模块的所有上游模块已完成（`outputs/<dep>/result.md` 存在）

## 执行流程

### Step 1: 检查上游依赖

```
模块 auth 的上游依赖: database

database 状态: ✅ 已完成
  → output: outputs/database/result.md ✓
  → 可以继续
```

如果上游未完成：
```
⚠️ auth 依赖 database，但 database 尚未完成。
请先执行 database，或运行 /execute-all 自动按波次调度。
```

### Step 2: 生成模块 subagent skill 文件

为要执行的模块生成一个 Reasonix subagent skill 文件，写入 `.reasonix/modules/skills/modularize-<module>.md`：

```markdown
---
description: "Implement the {{module}} module for {{task_id}}"
runAs: subagent
---

# 任务：实现 {{module}} 模块

## 项目上下文

> 以下内容是 SYSTEM prompt 的缓存关键，来自 prefix.md：
> 项目名称、技术栈、编码规范等

{{project_context_from_prefix}}

## 你的目标

实现 {{module}} 模块，严格遵循以下接口契约。

## 接口契约

> 这是你对外暴露的接口。下游模块将依赖这些签名。

{{contracts/module.md 内容}}

## 上游模块产出

> 这是你依赖的上游模块的接口。你只能使用以下签名。

{{#each upstream_deps}}
### {{name}} 模块
{{outputs/name/result.md 中对外接口部分}}
{{/each}}

## 执行要求

1. **严格遵循契约**：导出的函数签名、类型必须与契约完全一致
2. **只使用上游接口**：不要调用契约中未声明的上游函数
3. **完成后写 result.md**：遵照 templates/result.template.md 的格式
4. **关键决策记入 Memory**：使用 `/remember` 将以下信息写入 project memory：
   - 本模块使用了哪些库及版本
   - 任何可能影响下游模块的实现决策
   - 已知的性能/安全注意事项
5. **测试必须通过**：完成后运行 `npm test -- --testPathPattern={{module}}`

## 输出清单

完成后确认：
- [ ] 所有契约函数已实现
- [ ] 测试通过
- [ ] result.md 已写
- [ ] project memory 已更新
```

### Step 3: 安装并执行 subagent skill

```bash
# Reasonix 会自动加载 .reasonix/skills/ 下的 skill
/skill modularize-{{module}}
```

subagent 执行期间：
- 主会话不受 subagent 上下文污染
- subagent 完成后返回摘要
- 如果 subagent 失败 → 标记该模块为 FAILED → 通知用户

### Step 4: 验证产出

subagent 完成后，检查：
1. `outputs/<module>/result.md` 是否存在
2. result.md 中的"对外接口（与契约对照）"表 → 是否有不匹配项
3. 测试是否通过（从 result.md 状态判断）

### Step 5: 更新状态

更新 `plan.json`：
- 该模块 status → `completed`（或 `failed`）
- 如果该模块是当前 Wave 的最后一个 → 当前 Wave status → `completed`
- Plan status → 如果全部完成 → `executed`，否则 → 保持 `in_progress`

### Step 6: 告知进度

```
✅ auth 模块完成
   文件: src/modules/auth/index.ts, tests/auth.test.ts
   接口: register, login, verify — 全部符合契约

进度: Wave 1/3 — 2/5 模块完成
   ✅ database
   ✅ auth
   ⏳ api (等待执行)
   ⏳ storage (等待执行)
   ⏳ frontend (等待 api)

下一个可执行: api, storage
回复 /execute-all 继续，或 /execute <name> 手动执行。
```

## /execute-all 自动模式

`/execute-all` 按波次自动执行：

```
Wave 0 (并行):
  → 为 database 生成 subagent skill
  → 为 storage 生成 subagent skill
  → 依次执行两者

Wave 0 全部完成 →
Wave 1:
  → 为 auth 生成 subagent skill
  → 执行

...直至全部完成或遇到失败
```

**遇到失败时的行为**：
- 标记该模块为 FAILED
- 暂停后续 Wave（因为下游依赖失败模块）
- 输出失败报告
- 用户执行 `/revise <模块名>` 修复后继续

## /revise 恢复

```
/revise auth
```

1. 检查 auth 的 contract.md 是否需要修改（通常是实现不符合契约）
2. 如果契约需要修正 → 也检查 auth 的下游模块的 contract 是否受影响
3. 重新执行 auth（生成新的 subagent skill）
4. auth 完成后 → /execute-all 继续执行下游

## 缓存策略提醒

执行过程中，确保：
- **不修改 prefix.md**（一旦修改，所有后续 subagent 的 SYSTEM 缓存失效）
- **不添加时间戳到任何 SYSTEM prompt 内容中**
- **subagent skill 文件中的"项目上下文"部分来自 prefix.md，逐字复制，不加工**
