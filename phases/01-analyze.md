# Phase 1: 任务分析 → 模块拆分

> 加载时机：用户输入 `/decompose <任务描述>` 时

## 目标

将用户的自然语言任务描述转换为结构化的模块依赖图（DAG），让用户确认或调整。

## 执行流程

### Step 1: 读取上下文

先读以下文件以了解项目现状：
1. `prefix.md` — 如果存在（表示之前已经执行过 /decompose，本次是递归拆分），读取以了解父任务上下文
2. 当前项目的目录结构 — `ls -R src/` 看已有代码
3. 已有 contract 文件 — `ls .reasonix/modules/contracts/` 看已定义模块

### Step 2: 分析任务并提案

根据用户的任务描述，识别功能边界，提出模块拆分方案。

**拆分原则**：
- 每个模块 = 一个内聚的功能单元（如「用户注册登录」而不是「写 auth.ts 文件」）
- 模块间依赖尽量少（低耦合）
- 模块内部尽量完整（高内聚）
- 共享基础设施（数据库、日志等）优先拆为独立模块
- 如果模块太大，标记为"可递归拆分"

**输出格式**：
```
我分析了你的任务，建议拆成以下模块：

┌── database ──┐
│              ├──► auth ──┐
└──────────────┘           ├──► api ──► frontend
                           │
┌── storage ───────────────┘
└──────────────┘

模块详情：

| 模块 | 职责 | 依赖 | 复杂度 | 可递归？ |
|------|------|------|--------|----------|
| database | 数据库连接、迁移、查询封装 | 无 | medium | 否 |
| storage | 文件存储（上传/下载） | 无 | low | 否 |
| auth | 注册、登录、鉴权 | database | medium | 是 |
| api | REST 接口层 | auth, storage | high | 是 |
| frontend | 前端页面 | api | high | 是 |

确认这个拆分方案？你可以：
- 回复"确认"或"OK" → 进入 Phase 2
- 回复修改意见 → 我调整方案
- 对某个模块回复 "/decompose <模块名>" → 递归拆分子模块
```

### Step 3: 确认并写入 plan.json

用户确认后，写入初始 `plan.json`：

```json
{
  "task_id": "<kebab-case-task-name>",
  "task_description": "<原始任务描述>",
  "modules": [
    { "name": "database", "depends_on": [], "complexity": "medium", "recursive": false },
    { "name": "storage",  "depends_on": [], "complexity": "low",    "recursive": false },
    { "name": "auth",     "depends_on": ["database"], "complexity": "medium", "recursive": true },
    { "name": "api",      "depends_on": ["auth", "storage"], "complexity": "high", "recursive": true },
    { "name": "frontend", "depends_on": ["api"], "complexity": "high", "recursive": false }
  ],
  "waves": [],
  "status": "analyzed",
  "created_at": "<ISO timestamp>",
  "sub_modules": {}
}
```

写入路径：`.reasonix/modules/plan.json`

### Step 4: 告知用户下一步

```
模块拆分已保存到 plan.json。

下一步运行 /contracts 为每个模块生成接口契约，
或直接回复"/contracts"开始。
```

## 递归拆分

如果用户对某个已存在的模块执行 `/decompose <模块名>`：

1. 读取父任务的 plan.json
2. 将该模块标记为 `recursive: true`
3. 为子模块创建独立的子拆分：
   - 子模块的 prefix.md 继承父模块的 prefix.md
   - 子模块的 plan.json 写入 `sub_modules[父模块名]` 中
4. 子模块的契约 → 写入 `.reasonix/modules/contracts/<父模块>/<子模块>.md`
5. 子模块的产出 → 写入 `.reasonix/modules/outputs/<父模块>/<子模块>/`

## 边界情况

- 用户输入的任务描述太模糊 → 追问澄清，不要猜
- 项目已有代码 → 在提案中标注哪些模块可以复用现有代码
- 项目已有 plan.json → 本次是追加/修改，不是覆盖
