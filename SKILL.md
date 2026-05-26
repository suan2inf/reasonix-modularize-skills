---
name: modularize
description: "Cache-aware task decomposition — split large coding tasks into independent modules, define interface contracts, schedule execution waves, and execute each module as an isolated Reasonix subagent. Invoke when user needs to break down a complex task, build a multi-module system, or wants structured contract-driven development."
argument-hint: "<task description or command>"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Edit
  - WebFetch
---


<objective>
Split large coding tasks into independent modules executed as Reasonix subagents. All modules share a byte-stable SYSTEM prefix (prefix.md) so DeepSeek's prefix cache hits across subagent sessions.

Five phases: analyze → contracts → schedule → execute → integrate.
On failure: only re-run the broken module + its downstream dependents.
</objective>

<context>
Workflow files in `phases/`, templates in `templates/`, references in `reference/` — available if more detail is needed, but the core instructions are here.
</context>

<process>

## Phase 1: Task Analysis → Module Decomposition

**Trigger:** natural language task description or `/decompose`

### Step 1 — Scout existing context
- Read `.reasonix/modules/prefix.md` and `.reasonix/modules/plan.json` if they exist (resume or recursive split)
- Run `ls -R src/` to see existing code

### Step 2 — Propose module decomposition
Analyze the user's task and identify functional module boundaries.

**Principles:**
- Each module = one cohesive feature unit (e.g. "user auth", not "auth.ts file")
- Minimal inter-module dependencies
- Shared infrastructure (db, logging) splits first
- Mark oversized modules as recursive-splittable

**Output format:**
```
我分析了你的任务，建议拆成以下模块：

┌── database ──┐
│              ├──► auth ──┐
└──────────────┘           ├──► api ──► frontend
                           │
┌── storage ───────────────┘
└──────────────┘

| 模块 | 职责 | 依赖 | 复杂度 |
|------|------|------|--------|
| database | 数据库连接、查询封装 | 无 | medium |
| storage | 文件存储 | 无 | low |
| auth | 注册、登录、鉴权 | database | medium |
| api | REST 接口层 | auth, storage | high |
| frontend | 前端页面 | api | high |

确认？
```

### Step 3 — Write plan.json on confirmation
Path: `.reasonix/modules/plan.json`
```json
{
  "task_id": "<kebab-case>",
  "task_description": "<original task>",
  "modules": [
    { "name": "database", "depends_on": [], "complexity": "medium" },
    { "name": "auth", "depends_on": ["database"], "complexity": "medium" }
  ],
  "waves": [],
  "status": "analyzed",
  "created_at": "<ISO timestamp>"
}
```

If the user's description is too vague — ask clarifying questions. Do not guess.

## Phase 2: Interface Contracts + Shared Prefix

**Trigger:** user says "OK" after Phase 1, or `/contracts`

### Step 1 — Load templates
Read `templates/contract.template.md` and `templates/prefix.template.md` for the exact format.

### Step 2 — Generate contracts per module
For each module (topological order — root dependencies first):

Generate `contracts/<module>.md` containing:
- **Type definitions** in ```typescript blocks — machine-parseable
- **Interface signatures** — export function signatures exactly as they'll appear in code
- **Behavior descriptions** — what each function does, inputs, outputs, errors, edge cases
- **Upstream dependency interfaces** — copy-pasted from upstream module contracts (not from memory)

Show each contract to the user for review. Allow batch approval if user prefers.

### Step 3 — Generate prefix.md
After all contracts are approved, generate `prefix.md`:

**CRITICAL: prefix.md contains ONLY project overview + conventions. No module-specific content.**
- Project description, tech stack, directory structure
- Coding conventions (indentation, naming, imports, error handling)
- Module list (just names + status, no contract details)
- **NO timestamps, NO session IDs, NO dynamic content**

Why: prefix.md is the byte-stable SYSTEM prompt shared across all subagent sessions. It must be identical everywhere. Module contracts go in USER messages.

### Step 4 — Update plan.json
`status` → `contracted`

## Phase 3: Dependency Graph → Wave Scheduling

**Trigger:** user says "OK" after Phase 2, or `/schedule`

### Kahn algorithm:
1. Compute in-degree for each module (how many dependencies)
2. Modules with in-degree == 0 → Wave 0
3. Remove Wave 0 modules, decrement in-degrees of dependents
4. New in-degree == 0 → Wave 1
5. Repeat until all assigned
6. If modules remain → cycle detected → report error

**Output:**
```
执行计划（共 4 波）：

Wave 0 (可并行): database, storage
Wave 1: auth
Wave 2: api
Wave 3: frontend

确认？
```

On confirmation: write waves to plan.json, `status` → `ready`.

## Phase 4: Subagent Execution (CORE)

**Trigger:** user says "OK" after Phase 3, or `/execute [module]` or `/execute-all`

### /execute-all — Auto wave execution
```
For each wave in order:
  For each module in wave:
    1. Check upstream deps are completed
    2. Generate subagent skill: .reasonix/modules/skills/modularize-<module>.md
       with frontmatter: runAs: subagent
    3. Invoke: /skill modularize-<module>
    4. Subagent runs with:
       SYSTEM   = prefix.md (byte-stable → cache hits)
       USER     = contracts/<module>.md + upstream result.md(s) + "Implement <module>"
    5. Verify output: outputs/<module>/result.md exists
    6. Update plan.json status
    7. Report progress
```

### /execute <module> — Single module
Check upstream deps first. Same subagent flow. Used for manual control or after /revise.

### /revise <module> — Recovery
1. Identify failed module
2. Check if contract needs fixing → if so, review downstream contracts too
3. Regenerate subagent skill, re-execute
4. Then /execute-all to pick up downstream

### Subagent skill template
When generating `.reasonix/modules/skills/modularize-<module>.md`:
```markdown
---
description: "Implement the <module> module for task <task_id>"
runAs: subagent
---
# Task: Implement <module>

## Project Context (from prefix.md)
<copy prefix.md content verbatim>

## Your Contract (from contracts/<module>.md)
<copy contract content verbatim>

## Upstream Module Outputs
<for each upstream dep: copy its result.md interface section>

## Requirements
1. Strictly follow the contract signatures
2. Only call upstream interfaces declared in their contracts
3. After completion write outputs/<module>/result.md using templates/result.template.md
4. Use /remember to write key decisions to project memory
5. Run tests before completing
```

### Progress reporting format
```
✅ auth 完成 — 接口全部符合契约
进度: Wave 1/3 — 2/5 模块
下一个: api
回复 OK 继续，或 /execute <module>
```

## Phase 5: Integration Verification

**Trigger:** all modules completed, or `/integrate`

### Step 1 — Per-module contract verification
For each module, compare:
- `contracts/<module>.md` → what was promised
- `outputs/<module>/result.md` → what was delivered

Output mismatch report:
```
集成验证报告
════════════════════
✅ database — 全部匹配
✅ storage  — 全部匹配
⚠️ auth    — register 签名多了可选参数 name
✅ api      — 全部匹配

结果: 4/5 通过，1 需修复 → /revise auth
```

### Step 2 — Upstream/downstream chain check
Verify that downstream module imports match upstream module exports.

### Step 3 — Finalize
All pass → `plan.json` status → `done`.
Partial failures → mark failed modules, prompt `/revise`.

## /status — Check progress
Read `plan.json`, display:
- Task name + status
- Per-module status (pending → in_progress → completed → failed)
- Current wave / total waves
- Suggested next action

</process>

<success_criteria>
- Task decomposed into functional modules with clear dependency graph
- All contracts defined with TypeScript signatures + behavior descriptions
- prefix.md is byte-stable (no timestamps, no dynamic content, no module details)
- Execution runs in waves via isolated subagents
- Cache hits sustained across subagent sessions (prefix.md unchanged)
- Integration verification passes all contracts
- On failure: only broken module + downstream re-run
</success_criteria>
