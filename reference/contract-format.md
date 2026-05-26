# 契约 DSL 规范

## 概述

Modularize 的契约使用 **Markdown + TypeScript 代码块** 作为 DSL。选择 Markdown 而非 YAML/JSON 的原因：

- 人类可直接阅读（代码审查、文档）
- TypeScript 代码块可被语法高亮
- 集成验证时可提取代码块做签名比对
- 与 Reasonix 的 skill 体系风格一致

## 文件格式

### 必需字段

| 字段 | 位置 | 说明 |
|------|------|------|
| 模块名 | `# 模块：xxx` 标题 | 唯一标识，与 plan.json 中的 name 一致 |
| 依赖关系 | 元信息节 | 列出 `依赖` 和 `被依赖` |
| 类型定义 | `## 类型定义` → ` ```typescript ... ``` ` | 对外暴露的 interface/type |
| 接口签名 | `## 接口签名` → ` ```typescript ... ``` ` | 对外暴露的函数签名 |
| 上游依赖接口 | `## 上游依赖接口` → ` ```typescript ... ``` ` | 从上游模块 contract 中复制的类型签名 |
| 行为描述 | `## 行为描述` → 逐个函数 | 每个函数的语义说明 |
| 错误语义 | `## 错误语义` → 列表 | 本模块可能抛出的错误 |

### 可选字段

| 字段 | 说明 |
|------|------|
| 执行波次 | 生成时由 Phase 2 填入，Phase 3 更新 |
| 预计复杂度 | low / medium / high |
| 文件输出 | 预期产出的文件树 |
| 非功能需求 | 性能/安全/日志等要求 |

## TypeScript 代码块规范

### 接口签名格式

```
## 接口签名

```typescript
export function register(
  email: string,
  password: string
): Promise<User>

export function login(
  email: string,
  password: string
): Promise<Token>

export function verify(
  token: Token
): Promise<User>
```
```

要求：
- 每行一个函数（便于解析）
- 参数分行写（便于版本对比）
- 返回类型用 Promise<T> 包裹异步函数

### 类型定义格式

```
## 类型定义

```typescript
export interface User {
  id: string
  email: string
  name: string
  createdAt: Date
}

export type Token = string

export class EmailConflictError extends Error {
  constructor(email: string)
}
```
```

## 行为描述格式

```
### register

- **做什么**：接收邮箱和密码，校验格式和强度，存入数据库，返回创建的用户对象
- **输入**：
  - `email`: 有效的邮箱地址，格式为 `xxx@yy.zz`
  - `password`: 明文密码，至少 8 位，包含大小写字母和数字
- **输出**：`Promise<User>` — 新创建的用户对象（不含密码）
- **异常**：
  - `EmailConflictError` — 邮箱已存在
  - `ValidationError` — 邮箱格式或密码强度不合规
- **边界条件**：
  - 空字符串：抛出 ValidationError
  - 超长输入：邮箱 > 254 字符或密码 > 128 字符 → 抛出 ValidationError
  - 并发注册同一邮箱：保证只创建一个用户
```

## 上游依赖接口格式

这部分从上游模块的"接口签名"和"类型定义"中精确复制。

```
## 上游依赖接口

本模块使用的上游模块接口：

### database 模块

```typescript
// 复制自数据库模块的接口契约
export function query<T>(sql: string): Promise<T[]>
export function execute(sql: string): Promise<void>
export function connect(uri: string): Promise<DB>

export interface DB {
  close(): Promise<void>
}
```
```

要求：
- **不能凭记忆写** — 必须从上游 contract 中切实复制
- 只写本模块实际用到的上游接口（不写用不到的）
- 签名必须与上游 contract 完全一致

## 集成验证解析规则

验证时要提取的信息（从 Markdown 中）：

1. 从 `## 接口签名` 代码块提取：函数名 + 参数类型 + 返回类型
2. 从 `## 类型定义` 代码块提取：interface/type/class 定义
3. 从 `## 上游依赖接口` 提取：本模块声明的上游接口列表
4. 从 result.md 的 "对外接口（与契约对照）" 表提取：实际签名

比对逻辑：contract 中的签名 vs result.md 中的实际签名 → 逐字符匹配或结构化匹配。
