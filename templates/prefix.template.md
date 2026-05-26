# {{project_name}} — 共享前缀

> 此文件是字节稳定的 SYSTEM prompt。
> 在所有模块 subagent 的 SYSTEM 中完全一致，确保 DeepSeek 前缀缓存跨模块命中。
> **禁止在此文件中写入**：时间戳、会话 ID、模块专属内容、动态路径。

## 项目概述

{{project_description}}

## 技术栈

| 类别 | 选型 |
|------|------|
| 语言 | {{language}} |
| 运行时 | {{runtime}} |
| 包管理 | {{package_manager}} |
| 主要框架 | {{framework}} |
| 数据库 | {{database}} |
| 测试框架 | {{test_framework}} |

## 目录结构

```
{{project_root}}/
├── src/
│   ├── modules/           # 功能模块（按 Modularize 拆分）
{{#each module_dirs}}
│   │   └── {{this}}/
{{/each}}
│   └── shared/            # 模块间共享代码
└── tests/
```

## 编码规范

- 缩进：{{indent_spaces}} 空格
- 文件名：{{file_naming}}（如 kebab-case、camelCase、PascalCase）
- 函数命名：{{function_naming}}
- 类型命名：{{type_naming}}
- 常量命名：{{constant_naming}}
- 引号：{{quote_style}}（单引号 / 双引号）
- 分号：{{semicolons}}（必须 / 可选 / 省略）
- 最大行宽：{{max_line_width}} 字符

## 导入规范

```typescript
// 本地模块导入
import { Thing } from "../modules/{{example}}/index.js";

// 共享模块导入
import { SharedType } from "../shared/types.js";

// npm 依赖导入
import { something } from "package-name";
```

## 错误处理惯例

- {{error_handling_style}}
- 不要在模块之间传播实现细节异常（上游的内部错误应转为契约约定的错误类型）

## 测试规范

- 测试文件位置：{{test_location}}
- 测试命名：{{test_naming}}
- 每个模块必须具备的测试类型：{{required_test_types}}

## 模块清单

| 模块 | 依赖 | 波次 | 状态 |
|------|------|------|------|
{{#each module_list}}
| {{name}} | {{depends_on}} | Wave {{wave}} | {{status}} |
{{/each}}

## Git 规范

- 分支命名：{{branch_naming}}
- 提交信息格式：{{commit_format}}
