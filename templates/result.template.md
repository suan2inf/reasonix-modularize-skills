# {{module_name}} — 执行结果

> 此文件供下游模块和集成验证使用。必须精确、结构化。

## 状态

- **状态**：{{status}}（✅ 完成 / ⚠️ 部分完成 / ❌ 失败）
- **执行时间**：{{execution_time}}
- **文件数**：{{file_count}}
- **代码行数**：{{line_count}}

## 产出文件清单

```
{{#each output_files}}
{{this}}
{{/each}}
```

## 对外接口（与契约对照）

| 接口 | 契约签名 | 实际签名 | 状态 |
|------|----------|----------|------|
{{#each interface_check}}
| {{name}} | `{{contract_sig}}` | `{{actual_sig}}` | {{match_status}} |
{{/each}}

## 实现决策

> 以下决策对下游模块开发有影响，请务必阅读。

{{#each decisions}}
### {{title}}
- **选择**：{{choice}}
- **原因**：{{reason}}
- **替代方案**：{{alternatives_considered}}
- **对下游影响**：{{downstream_impact}}
{{/each}}

## 已知限制

{{#each limitations}}
- {{this}}
{{/each}}

## 未覆盖的测试场景

{{#each untested_scenarios}}
- {{this}}
{{/each}}

## Memory 写入

> 以下关键信息通过 Reasonix `remember` 写入 project memory，下游模块自动获得。

{{memory_items}}

## 集成验证清单

- [ ] 所有导出函数签名与契约一致
- [ ] 所有导出类型与契约一致
- [ ] 测试全部通过
- [ ] 无 lint 错误
- [ ] 无未处理的异常路径
