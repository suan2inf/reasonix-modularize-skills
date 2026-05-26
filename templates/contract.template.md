# 模块契约模板

> 使用时替换 `{{placeholder}}` 为实际值。
> 所有 TypeScript 代码块被视为接口约定，集成验证时进行签名比对。

# 模块：{{module_name}}

## 元信息
- **任务 ID**：{{task_id}}
- **依赖**：{{dependencies}}（逗号分隔，无依赖写"无"）
- **被依赖**：{{dependents}}（逗号分隔，无被依赖写"无"）
- **执行波次**：Wave {{wave_number}}
- **预计复杂度**：{{complexity}}（low / medium / high）

## 目标

{{one_paragraph_summary}} — 用一句话描述这个模块要做什么。

## 类型定义

```typescript
// 本模块对外暴露的类型
{{#each exported_types}}
{{this}}
{{/each}}
```

## 接口签名

```typescript
// 本模块对外暴露的函数/方法签名
{{#each exported_functions}}
{{this}}
{{/each}}
```

## 上游依赖接口

本模块使用的上游模块接口（从上游 contract 中提取）：

```typescript
// 来自 {{dependency_name}} 模块
{{#each upstream_imports}}
{{this}}
{{/each}}
```

## 行为描述

{{#each behavior_descriptions}}
### {{function_name}}

- **做什么**：{{what}}
- **输入**：{{inputs}}
- **输出**：{{outputs}}
- **异常**：{{errors}}
- **边界条件**：{{edge_cases}}

{{/each}}

## 错误语义

{{#each error_types}}
- **`{{name}}`**：{{description}}
{{/each}}

## 文件输出

本模块完成后应在以下位置产生产物：

```
{{output_file_tree}}
```

## 非功能需求

- **性能**：{{performance}}
- **安全**：{{security}}
- **日志**：{{logging}}
- **其他**：{{other_nfr}}
