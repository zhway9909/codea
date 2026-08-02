# Codea V1 Execution State Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Task 0 开始前建立可版本化、可校验、可恢复的研发执行状态，并把恢复与人工验收规则接入现有实施计划。

**Architecture:** `docs/execution-state.yaml` 是唯一机器可读状态源；`scripts/check-execution-state.sh` 使用 Python 3 + PyYAML 校验状态约束；每个 Task 使用独立 Markdown 报告保存验收证据。该机制只服务研发计划执行，不进入 Codea Runtime 或发行包。

**Tech Stack:** YAML 1.2、Bash、Python 3、PyYAML 6.x、Git

## Global Constraints

- 状态机制必须在 Task 0 正式执行前完成并提交。
- 同一时间最多一个 Task 处于 `in_progress`、`blocked` 或 `awaiting_acceptance`。
- Task 自动验证通过后只能进入 `awaiting_acceptance`，人工确认后才能进入 `completed`。
- `verification` 记录命令执行结果，`taskGate` 记录计划验收标准的整体判断，`humanAcceptance` 记录用户明确验收；三者不得混用。
- 缺少 Python 3 或 PyYAML 时，记录 `verification.status = unable_to_run`，并将 Task 标记为 `blocked`；不得跳过校验。
- 状态工具不属于 Codea Runtime，不得进入离线发行包。
- 不修改 Codea V1 的技术架构、Task 顺序或 Phase 0 门禁。

---

### Task E0: 初始化执行状态与恢复协议

**Files:**
- Create: `docs/execution-state.yaml`
- Create: `docs/task-reports/README.md`
- Create: `scripts/check-execution-state.sh`
- Create: `tests/execution-state/state_validator_test.sh`
- Modify: `AGENTS.md`
- Modify: `CLAUDE.md`
- Modify: `docs/superpowers/plans/2026-07-30-codea-v1-plan.md`

**Interfaces:**
- Consumes: `docs/superpowers/specs/2026-08-01-codea-execution-state-design.md`
- Produces: `scripts/check-execution-state.sh [state-file]`，成功返回 0，状态非法或依赖缺失返回非零。
- Produces: `docs/execution-state.yaml`，供所有后续 Task 在开始、阻塞、验证和验收时更新。

- [ ] **Step 1: 编写状态校验器测试**

创建 `tests/execution-state/state_validator_test.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
CHECKER="$ROOT_DIR/scripts/check-execution-state.sh"
STATE="$ROOT_DIR/docs/execution-state.yaml"
TMP_DIR="$(mktemp -d)"
trap 'rm -rf "$TMP_DIR"' EXIT

expect_fail() {
  local name="$1"
  local file="$2"
  if "$CHECKER" "$file" >/dev/null 2>&1; then
    echo "FAIL: $name unexpectedly passed"
    exit 1
  fi
}

"$CHECKER" "$STATE"

python3 - "$STATE" "$TMP_DIR" <<'PY'
import copy
import pathlib
import sys
import yaml

source = pathlib.Path(sys.argv[1])
target = pathlib.Path(sys.argv[2])
data = yaml.safe_load(source.read_text())

duplicate = copy.deepcopy(data)
duplicate["tasks"]["0"]["status"] = "in_progress"
duplicate["tasks"]["1"]["status"] = "blocked"
(target / "duplicate-active.yaml").write_text(yaml.safe_dump(duplicate, sort_keys=False))

unaccepted = copy.deepcopy(data)
unaccepted["current"]["status"] = "completed"
unaccepted["tasks"]["0"].update({
    "status": "completed",
    "verificationStatus": "pass",
    "taskGateStatus": "pass",
    "humanAccepted": False,
})
(target / "unaccepted-completed.yaml").write_text(yaml.safe_dump(unaccepted, sort_keys=False))

mismatch = copy.deepcopy(data)
mismatch["current"]["status"] = "in_progress"
(target / "current-mismatch.yaml").write_text(yaml.safe_dump(mismatch, sort_keys=False))

acceptance_mismatch = copy.deepcopy(data)
acceptance_mismatch["humanAcceptance"]["accepted"] = True
(target / "acceptance-mismatch.yaml").write_text(yaml.safe_dump(acceptance_mismatch, sort_keys=False))

missing_report = copy.deepcopy(data)
missing_report["current"]["status"] = "completed"
missing_report["verification"]["status"] = "pass"
missing_report["taskGate"]["status"] = "pass"
missing_report["humanAcceptance"]["accepted"] = True
missing_report["tasks"]["0"].update({
    "status": "completed",
    "verificationStatus": "pass",
    "taskGateStatus": "pass",
    "humanAccepted": True,
})
(target / "missing-report.yaml").write_text(yaml.safe_dump(missing_report, sort_keys=False))
PY

expect_fail "duplicate active tasks" "$TMP_DIR/duplicate-active.yaml"
expect_fail "completed without acceptance" "$TMP_DIR/unaccepted-completed.yaml"
expect_fail "current status mismatch" "$TMP_DIR/current-mismatch.yaml"
expect_fail "current acceptance mismatch" "$TMP_DIR/acceptance-mismatch.yaml"
expect_fail "completed without report" "$TMP_DIR/missing-report.yaml"

echo "Execution state validator tests passed."
```

- [ ] **Step 2: 运行测试并确认失败**

Run:

```bash
bash tests/execution-state/state_validator_test.sh
```

Expected: FAIL，因为 `scripts/check-execution-state.sh` 或 `docs/execution-state.yaml` 尚不存在。

- [ ] **Step 3: 创建初始状态文件**

创建 `docs/execution-state.yaml`。初始检查点固定指向已经通过评审的执行状态设计提交：

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
  commit: 02caeb6c6e8ed40deccdadd5eac1aaf5acfe4d7b
  updatedAt: "2026-08-01T16:14:57+09:00"
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
  "0": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-00.md}
  "1": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-01.md}
  "2": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-02.md}
  "3": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-03.md}
  "4": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-04.md}
  "5": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-05.md}
  "6": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-06.md}
  "7": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-07.md}
  "8": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-08.md}
  "9": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-09.md}
  "10": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-10.md}
  "11": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-11.md}
  "12": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-12.md}
  "13": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-13.md}
  "14": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-14.md}
  "15": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-15.md}
  "16": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-16.md}
  "17": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-17.md}
  "18": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-18.md}
  "19": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-19.md}
  "20": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-20.md}
  "21": {status: pending, completedSteps: [], verificationStatus: not_run, taskGateStatus: not_evaluated, humanAccepted: false, checkpoint: null, report: docs/task-reports/task-21.md}
```

- [ ] **Step 4: 实现状态校验器**

创建 `scripts/check-execution-state.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

STATE_FILE="${1:-docs/execution-state.yaml}"

if ! command -v python3 >/dev/null 2>&1; then
  echo "BLOCKED: python3 is required to validate execution state" >&2
  exit 2
fi

if ! python3 -c 'import yaml' >/dev/null 2>&1; then
  echo "BLOCKED: PyYAML is required to validate execution state" >&2
  exit 2
fi

python3 - "$STATE_FILE" <<'PY'
import pathlib
import sys
import yaml

path = pathlib.Path(sys.argv[1])
if not path.is_file():
    raise SystemExit(f"FAIL: state file not found: {path}")

try:
    state = yaml.safe_load(path.read_text())
except Exception as exc:
    raise SystemExit(f"FAIL: invalid YAML: {exc}")

task_states = {"pending", "in_progress", "blocked", "awaiting_acceptance", "completed"}
verification_states = {"not_run", "pass", "fail", "unable_to_run"}
task_gate_states = {"not_evaluated", "pass", "fail", "unable_to_evaluate"}

if state.get("schemaVersion") != 1:
    raise SystemExit("FAIL: schemaVersion must be 1")

tasks = state.get("tasks")
if not isinstance(tasks, dict) or set(tasks) != {str(i) for i in range(22)}:
    raise SystemExit("FAIL: tasks must contain exactly Task 0 through Task 21")

for task_id, task in tasks.items():
    if task.get("status") not in task_states:
        raise SystemExit(f"FAIL: Task {task_id} has invalid status")
    if task.get("verificationStatus") not in verification_states:
        raise SystemExit(f"FAIL: Task {task_id} has invalid verificationStatus")
    if task.get("taskGateStatus") not in task_gate_states:
        raise SystemExit(f"FAIL: Task {task_id} has invalid taskGateStatus")
    if task.get("status") == "completed":
        if task.get("verificationStatus") != "pass" or task.get("taskGateStatus") != "pass":
            raise SystemExit(f"FAIL: completed Task {task_id} must pass verification and Task Gate")
        if task.get("humanAccepted") is not True:
            raise SystemExit(f"FAIL: completed Task {task_id} requires human acceptance")
        if not pathlib.Path(task.get("report", "")).is_file():
            raise SystemExit(f"FAIL: completed Task {task_id} report is missing")

active = [task_id for task_id, task in tasks.items() if task["status"] in {"in_progress", "blocked", "awaiting_acceptance"}]
if len(active) > 1:
    raise SystemExit("FAIL: more than one Task is active")

current = state.get("current", {})
current_id = str(current.get("task"))
if current_id not in tasks:
    raise SystemExit("FAIL: current.task is invalid")
if current.get("status") != tasks[current_id]["status"]:
    raise SystemExit("FAIL: current.status does not match current Task")
if current.get("status") != "pending" and active != [current_id]:
    raise SystemExit("FAIL: current.task must be the unique active Task")

verification = state.get("verification", {})
task_gate = state.get("taskGate", {})
human_acceptance = state.get("humanAcceptance", {})
if verification.get("status") != tasks[current_id]["verificationStatus"]:
    raise SystemExit("FAIL: current verification does not match current Task")
if task_gate.get("status") != tasks[current_id]["taskGateStatus"]:
    raise SystemExit("FAIL: current task gate does not match current Task")
if human_acceptance.get("accepted") != tasks[current_id]["humanAccepted"]:
    raise SystemExit("FAIL: current acceptance does not match current Task")

seen_incomplete = False
for task_id in map(str, range(22)):
    status = tasks[task_id]["status"]
    if status != "completed":
        seen_incomplete = True
    elif seen_incomplete:
        raise SystemExit(f"FAIL: Task {task_id} completed before an earlier Task")

print(f"Execution state is valid: Task {current_id} Step {current.get('step')} ({current.get('status')})")
PY
```

- [ ] **Step 5: 创建 Task 报告目录说明**

创建 `docs/task-reports/README.md`：

```markdown
# Task Reports

每个 Task 完成自动验证后，在本目录创建 `task-XX.md` 验收报告。报告必须遵循项目交接模板，并与 `docs/execution-state.yaml` 和实际 Git Commit 一致。

状态进入 `awaiting_acceptance` 后停止开发；只有人工明确验收，才能将对应 Task 标记为 `completed`。
```

- [ ] **Step 6: 接入执行者入口和主实施计划**

在 `AGENTS.md` 和 `CLAUDE.md` 的“每个 Task 的工作流”之前增加：

````markdown
### 执行状态恢复

开始任何 Task 前必须先读取并校验 `docs/execution-state.yaml`：

```bash
./scripts/check-execution-state.sh
```

状态为 `blocked` 时先处理阻塞；状态为 `awaiting_acceptance` 时停止并等待人工验收。状态文件、Git Commit 或工作区互相矛盾时停止报告，不得猜测或跳过。
````

更新 `docs/superpowers/plans/2026-07-30-codea-v1-plan.md`：

- 在 Global Constraints 增加执行状态文件、单活动 Task、人工验收门禁三条约束。
- 把 Task 0 的三个残留路径改为 `tui/cmd/codea/main.go`、`tui/cmd/parity-runner/main.go`、`.editorconfig`。
- 把 Task 0 Commit 命令中的 `git init &&` 删除。
- Task 0 Files 增加 Modify `docs/execution-state.yaml` 和 Create `docs/task-reports/task-00.md`。
- Task 0 验证步骤增加 `./scripts/check-execution-state.sh`。
- Task 0 完成后将状态更新为 `awaiting_acceptance` 并停止，人工验收前不得开始 Task 1。

- [ ] **Step 7: 运行校验和测试**

Run:

```bash
chmod +x scripts/check-execution-state.sh tests/execution-state/state_validator_test.sh
bash tests/execution-state/state_validator_test.sh
./scripts/check-execution-state.sh
git diff --check
```

Expected:

```text
Execution state is valid: Task 0 Step 1 (pending)
Execution state validator tests passed.
Execution state is valid: Task 0 Step 1 (pending)
```

`git diff --check` 无输出并返回 0。

- [ ] **Step 8: 提交执行状态机制**

```bash
git add AGENTS.md CLAUDE.md docs/execution-state.yaml docs/task-reports/README.md scripts/check-execution-state.sh tests/execution-state/state_validator_test.sh docs/superpowers/plans/2026-07-30-codea-v1-plan.md
git commit -m "chore: add recoverable task execution state"
```

提交后重新运行：

```bash
./scripts/check-execution-state.sh
git status --short
```

Expected: 状态校验通过；工作区只允许存在尚未提交的本实施计划文档，否则必须解释来源。
