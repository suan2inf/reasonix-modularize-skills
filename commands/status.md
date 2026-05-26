查看 modularize 的当前进度和状态。

**立即执行**：

1. 读取 `.reasonix/modules/plan.json`
2. 如果不存在 → 提示用户先运行 /decompose
3. 显示：
   - 任务名称和状态
   - 每个模块的状态（pending / completed / failed）
   - 当前 Wave 和进度百分比
   - 下一步建议操作

不需要前置条件。
