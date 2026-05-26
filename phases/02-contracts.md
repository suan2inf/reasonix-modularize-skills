# Phase 2: 接口契约生成 + 共享前缀

> 加载时机：Phase 1 确认后，或用户输入 `/contracts` 时

## 目标

1. 为每个模块生成精确的接口契约（遵照 `templates/contract.template.md`）
2. 汇总生成 `prefix.md`（遵照 `templates/prefix.template.md`）
3. 所有契约经用户审核确认后冻结，作为后续执行的基准

## 前置条件

- `.reasonix/modules/plan.json` 已存在，status = `analyzed`

## 执行流程

### Step 1: 加载模板和上下文

读取：
- `templates/contract.template.md` — 契约模板
- `templates/prefix.template.md` — 前缀模板
- `plan.json` — 模块列表
- 项目已有代码（如有）— 避免生成与现有代码冲突的契约

### Step 2: 按依赖顺序生成契约

按照拓扑顺序（无依赖的先来），为每个模块生成契约。

**对每个模块**：
1. 分析其在依赖图中的位置 → 确定上游依赖接口
2. 分析其职责 → 确定对外暴露的类型和函数
3. 填充 `contract.template.md` 模板 → 写入 `contracts/<module>.md`

**契约要求**：
- TypeScript 类型签名必须写在代码块中（```typescript），确保可机器解析
- 行为描述必须覆盖：做什么、输入、输出、异常、边界条件
- 上游依赖接口必须从上游模块的契约中精确复制（不是凭记忆写）
- 错误语义必须列出所有可能的异常类型

### Step 3: 逐模块审核

每生成一个模块的契约，展示给用户审核：

```
--- auth 模块契约 ---
[显示 contracts/auth.md 内容]

确认？回复：
- "OK" → 确认此模块，继续下一个
- 修改意见 → 我调整
- "skip" → 暂存，最后再看
```

注：如果用户不耐烦逐模块审核，可以一次性展示所有契约，等用户批量确认。

### Step 4: 生成 prefix.md

全部契约确认后，生成 `prefix.md`。

**关键约束**：
- **prefix.md 不含任何模块专属内容**（所有模块契约在各自的 contract.md 中）
- **prefix.md 不含时间戳、会话 ID、动态路径**
- prefix.md 只包含：项目概述、技术栈、目录结构、编码规范、模块清单（仅名字和状态）
- 如果项目已有 prefix.md，合并而非覆盖

### Step 5: 更新状态

更新 `plan.json`：`status` → `contracted`

### Step 6: 告知用户下一步

```
所有契约已生成，prefix.md 已就绪。

契约文件：.reasonix/modules/contracts/
共享前缀：.reasonix/modules/prefix.md

下一步运行 /schedule 计算执行顺序，
或直接回复"/schedule"。
```

## 注意事项

- 不要在 prefix.md 中写 `{{module_list}}` 的详细实现——只写模块名称表格
- 契约中的 TypeScript 类型必须可自洽（上下游类型引用对齐）
- 如果用户中途改了一个模块的契约，自动检查下游模块的契约是否需要同步更新
