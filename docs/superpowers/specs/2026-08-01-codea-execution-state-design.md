# Codea V1 执行状态机制设计

日期：2026-08-01

状态：设计评审通过，允许执行配套实施计划

## 1. 目标

为 Codea V1 的 Task 0～Task 21 提供可恢复、可审计的执行进度记录。会话中断、执行模型更换或任务阻塞后，接手者能够确定当前 Task、当前 Step、最近验证结果和下一步操作，避免重复执行或越过门禁。

该机制只管理研发计划的执行状态，不属于 Codea 产品运行时，也不代替 Git 历史、实施计划或 Task 验收报告。

## 2. 文件组成

- `docs/execution-state.yaml`：唯一机器可读状态源。
- `docs/task-reports/task-XX.md`：每个 Task 的最终验收报告。
- `scripts/check-execution-state.sh`：验证状态枚举、唯一活动任务、当前 Step 和 Gate 约束。

实施计划仍然是任务内容的唯一来源。状态文件只保存执行位置、验证结果和证据引用，不复制完整 Task 步骤。

## 3. 状态模型

Task 状态：

- `pending`：尚未开始。
- `in_progress`：当前正在执行。
- `blocked`：存在阻塞，不能继续执行。
- `awaiting_acceptance`：实现和自动验证已经完成，等待人工验收。
- `completed`：人工验收通过，Task 正式结束。

合法流转：

```text
pending -> in_progress
in_progress -> blocked
blocked -> in_progress
in_progress -> awaiting_acceptance
awaiting_acceptance -> in_progress
awaiting_acceptance -> completed
```

验证状态：

- `not_run`：尚未执行。
- `pass`：要求的验证命令全部通过。
- `fail`：验证已经执行但失败。
- `unable_to_run`：因环境、权限或依赖问题无法执行验证；此时 Task 状态必须为 `blocked`。

三层判定含义：

1. `verification.status`：只记录构建、测试、校验脚本等命令是否真实执行成功。
2. `taskGate.status`：根据当前 Task 计划检查必需文件、交付物、验证结果和特定门禁是否全部满足。Task 1 的该字段汇总 S1～S6；其他 Task 按各自验收标准判断。
3. `humanAcceptance.accepted`：记录用户是否明确验收。它不属于自动验证，也不属于 Task Gate。

只有验证通过、Task Gate 通过且人工验收通过，Task 才能进入 `completed`。

Task Gate 状态：

- `not_evaluated`：尚未根据当前 Task 的完整验收标准进行判断。
- `pass`：必需交付物、验证结果和专项门禁全部满足。
- `fail`：已经完成判断，但至少一项验收标准不满足。
- `unable_to_evaluate`：因缺少证据、环境或依赖，无法完成门禁判断；此时 Task 状态必须为 `blocked`。

## 4. 状态文件结构

```yaml
schemaVersion: 1
project: codea-v1
plan: docs/superpowers/plans/2026-07-30-codea-v1-plan.md

current:
  task: 0
  step: 1
  status: pending
  nextAction: Start Task 0 Step 1

checkpoint:
  commit: d8e1bd6b23b9b5a573eaeaff5bbc9bf7350f6632
  updatedAt: "2026-08-01T00:00:00Z"

verification:
  status: not_run
  commands: []
  errorSummary: null
  recoveryAdvice: null

taskGate:
  status: not_evaluated

humanAcceptance:
  accepted: false

tasks:
  "0":
    status: pending
    completedSteps: []
    verificationStatus: not_run
    taskGateStatus: not_evaluated
    humanAccepted: false
    checkpoint: d8e1bd6b23b9b5a573eaeaff5bbc9bf7350f6632
    report: docs/task-reports/task-00.md
  "1":
    status: pending
    completedSteps: []
    verificationStatus: not_run
    taskGateStatus: not_evaluated
    humanAccepted: false
    checkpoint: null
    report: docs/task-reports/task-01.md
```

以上 Commit 和时间仅为格式示例，初始化时必须写入实际值。

`tasks` 必须包含 Task 0～Task 21。只有当前 Task 记录 Step 级进度；已经完成的 Task 保留状态、完成步骤、验证结果、Task Gate、人工验收、检查点和报告路径。

## 5. 更新规则

1. 开始一个 Step 前，将 `current.step` 和 `current.status` 更新为对应 Step 与 `in_progress`。
2. Step 完成后，将 Step 加入 `completedSteps`，记录下一步；形成 Git 检查点后再更新 `checkpoint.commit`。
3. 验证执行失败时记录 `verification.status = fail`；验证无法执行时记录 `verification.status = unable_to_run`。两种情况都将 Task 状态改为 `blocked`，并记录失败命令、错误摘要和恢复建议。
4. 恢复 `blocked` 状态前，不得清除原始错误证据；修复后重新执行对应验证。
5. 当前 Task 的全部实现、自动验证和 Task Gate 均通过后，状态改为 `awaiting_acceptance`。
6. 只有人工明确验收后，才能将 Task 改为 `completed` 并把下一个 Task 改为 `in_progress`。
7. 同一时间最多一个 Task 和一个 Step 处于 `in_progress`。
8. 不得跳过 `pending` Task，不得从 `blocked` 或 `awaiting_acceptance` 直接进入下一 Task。

## 6. 中断恢复流程

接手者开始工作前必须：

1. 阅读 `AGENTS.md`、技术设计、实施计划和 `docs/execution-state.yaml`。
2. 运行 `git status`，核对当前 HEAD 与 `checkpoint.commit`。
3. 如果存在未提交变更，先判断它们是否属于当前 Step，不得直接覆盖或丢弃。
4. 如果状态为 `in_progress`，从当前 Step 恢复，并重新运行该 Step 的验证。
5. 如果状态为 `blocked`，先处理记录的阻塞项。
6. 如果状态为 `awaiting_acceptance`，停止实现并等待人工验收。
7. 如果 Commit、状态文件和工作区互相矛盾，停止执行并报告，不得自行猜测进度。

## 7. Task 验收

每个 Task 完成后生成 `docs/task-reports/task-XX.md`，沿用项目交接文档中的交付模板，至少包含：

- Task 编号与名称
- 完成内容
- 文件变更
- 执行命令
- 验证结果
- 与计划偏差
- 未解决问题
- Gate 结论

Task 报告、状态文件和实际 Git Commit 必须相互一致。

## 8. 校验脚本

`scripts/check-execution-state.sh` 至少校验：

- YAML 可以解析且 `schemaVersion` 受支持。
- Task 0～Task 21 全部存在。
- 状态值属于允许枚举。
- 最多一个 Task 为 `in_progress`、`blocked` 或 `awaiting_acceptance`。
- `current.status` 必须和 `tasks[current.task].status` 一致。
- 除初始化时允许当前 Task 为 `pending` 外，`current.task` 必须指向唯一活动 Task。
- `completed` Task 必须有验收报告、`taskGateStatus = pass`、`verificationStatus = pass` 和 `humanAccepted = true`。
- 当前 Task 未完成时，后续 Task 不得进入活动或完成状态。

校验失败必须返回非零退出码。

## 9. 与 Task 0 的关系

状态机制在 Task 0 正式执行前初始化。Task 0 开始时将状态从 `pending` 更新为 `in_progress`；Task 0 构建和测试通过后生成 `task-00.md`，状态进入 `awaiting_acceptance`。人工验收通过前不得开始 Task 1。
