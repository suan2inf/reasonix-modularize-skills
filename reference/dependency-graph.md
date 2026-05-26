# 依赖图算法说明

## 数据结构

模块依赖图是一个**有向无环图（DAG）**。

```
节点 = 模块
边 (A → B) = B 依赖 A（A 必须先完成，B 才能开始）
```

### plan.json 中的表示

```json
{
  "modules": [
    { "name": "A", "depends_on": [] },
    { "name": "B", "depends_on": ["A"] },
    { "name": "C", "depends_on": ["A", "B"] }
  ]
}
```

以上表示：A → B → C，且 C 也直接依赖 A。

## 算法：Kahn 拓扑排序

用于计算执行波次，保证：
- 每个模块的所有上游在它之前执行
- 无依赖关系的模块可以并行（同一波次）

### 伪代码

```
function kahn_waves(modules):
    // 1. 建图
    graph = {}
    in_degree = {}
    for m in modules:
        graph[m.name] = []           // m 被哪些模块依赖
        in_degree[m.name] = len(m.depends_on)

    for m in modules:
        for dep in m.depends_on:
            graph[dep].append(m.name)  // dep 完成 → 解锁 m

    // 2. 初始化：入度为 0 的模块入队
    queue = [m for m in modules if in_degree[m.name] == 0]
    waves = []
    wave_num = 0

    // 3. BFS 分层
    while queue:
        wave = []
        next_queue = []

        for name in queue:
            wave.append(name)

            for dependent in graph[name]:
                in_degree[dependent] -= 1
                if in_degree[dependent] == 0:
                    next_queue.append(dependent)

        waves.append({ "wave": wave_num, "modules": wave, "status": "pending" })
        queue = next_queue
        wave_num += 1

    // 4. 循环检测
    if total_modules_in_waves < len(modules):
        remaining = [m.name for m in modules if m.name not in all_wave_modules]
        error("循环依赖: " + remaining)

    return waves
```

### 示例

输入：
```
A: 无依赖
B: 依赖 A
C: 依赖 A
D: 依赖 B, C
```

执行轨迹：

| 步 | 队列 | 波次 | 说明 |
|----|------|------|------|
| 初始 | [A] | | A 入度=0 |
| 1 | [B, C] | Wave 0: [A] | 移除 A，B、C 入度分别 -1 |
| 2 | [D] | Wave 1: [B, C] | 移除 B、C，D 入度 -2 |
| 3 | [] | Wave 2: [D] | 移除 D |
| 完毕 | | 全部完成 |

输出波浪：
```
Wave 0: [A]
Wave 1: [B, C]  ← B 和 C 并行
Wave 2: [D]
```

## 循环依赖检测

如果输入存在环（A → B → C → A），算法结束后仍有模块未分配，即为循环依赖。

Reasonix 环境中，循环依赖是设计问题而非实现问题：
- 通常意味着两个模块职责边界不清
- 建议方案：抽离共享部分为第三个模块

## 时间复杂度

- O(V + E)，其中 V = 模块数，E = 依赖边数
- 对于 Modularize 场景（通常 V < 20），远低于 1ms，完全可接受
