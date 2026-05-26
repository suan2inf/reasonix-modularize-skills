# Phase 5: 集成验证 + 失败处理

> 加载时机：全部模块执行完毕后，或用户输入 `/integrate` 时

## 目标

验证所有模块的产出是否符合契约，检测接口不匹配，失败时精确重跑问题模块。

## 前置条件

- 所有模块 status = `completed`（允许有 `failed`）
- 每个模块的 `outputs/<module>/result.md` 存在

## 执行流程

### Step 1: 逐模块比对契约

对每个模块，读取：
- `contracts/<module>.md` — 原始契约
- `outputs/<module>/result.md` — 执行结果

比对：
1. **导出函数签名**：契约中的每个 export function ↔ 实际代码中的函数签名
2. **导出类型**：契约中的每个 interface/type ↔ 实际代码中的类型定义
3. **文件清单**：result.md 中的文件列表 ↔ 实际磁盘上的文件

输出比对报告：

```
集成验证报告
═══════════════════════════════════════

✅ database — 所有接口符合契约
✅ storage  — 所有接口符合契约
⚠️ auth    — 1 处不匹配
   register: 契约签名 (email: string, password: string) => Promise<User>
             实际签名 (email: string, password: string, name?: string) => Promise<User>
             差异: 增加了可选参数 name，未在契约中声明
✅ api      — 所有接口符合契约
✅ frontend — 所有接口符合契约

结果: 4/5 通过，1 个需要修复
```

### Step 2: 上下游接口匹配检查

检查依赖链上的接口调用是否一致：

```
上游模块 database → 下游模块 auth:
  auth 调用 database.query<User>(sql)
  database 实际导出 database.query<T>(sql: string): Promise<T[]>
  → 匹配 ✓

上游模块 auth → 下游模块 api:
  api 调用 auth.verify(token)
  auth 实际导出 auth.verify(token: Token): Promise<User>
  → 匹配 ✓
```

### Step 3: 不匹配处理

如果发现不匹配：

**情况 1：实现偏离了契约**
```
auth.license 的实现与契约不一致。
建议: /revise auth — 按契约重新实现
```

**情况 2：契约本身需要修正**
```
auth.register 的契约缺少 name 参数，但实现更合理。
建议: 先修改 contracts/auth.md，然后 /revise auth 及下游
```

**情况 3：下游使用了未导出的接口**
```
api 模块调用了 auth._internalHash()，但 auth 的契约未声明此函数。
建议: 要么把 _internalHash 加入 auth 的契约，要么从 api 中移除该调用
```

### Step 4: 全部通过

```
集成验证通过 ✓

所有 5 个模块的接口契约全部匹配。
上游 → 下游调用链一致。
模块关系图无反模式。

任务 "{{task_name}}" 状态: ✅ DONE

产出文件:
  .reasonix/modules/plan.json
  .reasonix/modules/prefix.md
  .reasonix/modules/contracts/*
  .reasonix/modules/outputs/*

你可以回复 /archive 归档本次任务，或开始新任务 /decompose。
```

### Step 5: 更新状态

- 全部匹配 → `plan.json` status → `done`
- 部分不匹配 → 标记问题模块, status 保持 `in_progress`，提示用户 `/revise`

## 部分验证

如果用户只想验证特定模块：

```
/integrate auth api
```
只检查 auth 和 api（及其上下游依赖链）。

## 注意

- 不匹配 ≠ 模块要全部重写。多数情况只需小修正
- 如果多个模块共用同一个不匹配问题，优先修最上游的模块
- 修正上游后，自动提醒用户下游也需要验证
