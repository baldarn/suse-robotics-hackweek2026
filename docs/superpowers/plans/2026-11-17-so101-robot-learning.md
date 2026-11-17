# SO-101 Robot Learning Stack — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The SO-101 autonomously performs learned tasks (ACT), orchestrated by an agent via MCP, with Grafana telemetry and GitOps deployment (K3s + Fleet).

**Architecture:** LeRobot does all the "robotics" work (teleop, record, train, async inference: policy server on the 3090 desktop, robot client on the Pi). Our code is a small Python package `so101lab` whose core is **robotd**: an HTTP service on the Pi that owns the arm, launches LeRobot's `robot_client` as a subprocess for each episode, handles stop/home/timeout, and exposes Prometheus telemetry and camera snapshots. The MCP server and the evaluation CLI are thin clients of robotd.

**Tech Stack:** Python 3.12, uv, LeRobot (`feetech` extra), FastAPI + uvicorn, prometheus_client, httpx, `mcp` (FastMCP), pydantic v2, PyYAML, OpenCV, pytest. K3s, Fleet (standalone), Prometheus + Grafana Helm charts.

**Spec:** `docs/specs/2026-11-17-so101-robot-learning-design.md`

## Global Constraints

- Minimum bar: task 1 succeeds in ≥ 7/10 autonomous attempts; this and the evaluation are never cut.
- Cut order if behind schedule: Fleet → local LLM → K3s → Grafana → task 2.
- Task 1 `cube_to_box`: cube in a ~15×15 cm zone, roughly constant orientation, fixed box, success = released inside the box, 30 s timeout.
- Task 2 `box_to_zone`: from the box back to the starting zone, 30 s timeout.
- ~50 episodes per task; 2–3 cubes of different colors, alternated (one at a time); cameras fixed and never moved between recording and inference.
- Existing calibration (on this laptop): `~/.cache/huggingface/lerobot/calibration/robots/so_follower/follower.json` and `teleoperators/so_leader/leader.json` → robot type `so_follower` id `follower`, teleop type `so_leader` id `leader`. Must be copied unchanged to the Pi and the desktop.
- Policy server and training run **natively** on the desktop (no GPU in K3s). Robot client on the Pi.
- Episodes: end on timeout or `stop`, then rest pose; in the official evaluation the outcome is marked by the human; the cube is reset by hand during evaluation.
- The arm must never drop: every connect/disconnect from our code and from the robot client uses `disable_torque_on_disconnect=false`.
- Secrets (HF token, API keys) only in env/`.env` (gitignored), never in argv, logs or HTTP output.
- LeRobot commands: CLI and flag names change between versions → every LeRobot command in this plan must be checked with `--help` on the installed version and, if different, corrected in `docs/runbook.md`.

## Review Focus

1. `run_task` while an episode is already running → 409, no second `robot_client` (the serial port is exclusive).
2. `robot_client` dying immediately (policy server down, wrong repo) → episode `ended_by="error"` with the stderr tail, robotd back to `idle`, arm under torque and returned home.
3. `stop` when nothing is running, or called twice → idempotent no-op, never raises.
4. Invalid catalog committed (duplicate name, timeout out of range) → blocked by the test on the repo file before Fleet deploys it.
5. `snapshot` during an episode or with the camera unavailable → 409 / 503 immediately, never blocks.

---

## Phase 0 — Hardware runbook (user, Nov 17–29)

Manual work in front of the robot. The agent prepares scripts and `docs/runbook.md`; the user runs them and records the actual commands.

### Task R1: Pi 5, LeRobot and command verification

**Files:**
- Create: `docs/runbook.md`, `scripts/setup-node.sh`

- [ ] **Step 1:** OS on the Pi: SLE Micro or openSUSE MicroOS/Tumbleweed aarch64, half-day timebox (boot, USB serial, V4L2 cameras, K3s); otherwise Raspberry Pi OS 64-bit. Record the choice and the issues in the runbook.
- [ ] **Step 2:** `scripts/setup-node.sh` (Pi and desktop): installs uv, clones the repo, `uv sync --extra robot`, copies the calibration folder to `~/.cache/huggingface/lerobot/calibration/`.
- [ ] **Step 3:** Check and record the actual names in the runbook: `lerobot-find-port`, `lerobot-find-cameras`, `lerobot-teleoperate --help`, `lerobot-record --help`, `lerobot-train --help`, and the async inference modules (`python -m lerobot.async_inference.policy_server --help`, `python -m lerobot.async_inference.robot_client --help`). In particular, record the name of the torque-on-disconnect flag.
- [ ] **Step 4:** Teleop from the Pi: `lerobot-teleoperate --robot.type=so_follower --robot.id=follower --robot.port=<port> --teleop.type=so_leader --teleop.id=leader --teleop.port=<port>`.
  Expected: the follower tracks the leader.
- [ ] **Step 5:** Commit runbook and script.

### Task R2: Cameras, workstation and task 1 dataset

**Files:**
- Create: `scripts/record.sh`, `deploy/k8s/robotd/robot.yaml` (first version, schema in Task 1)

- [ ] **Step 1:** Mount the wrist and front cameras; mark the 15×15 cm zone and the box position with tape. Photo of the workstation in the runbook.
- [ ] **Step 2:** `lerobot-find-cameras` → stable devices (`/dev/v4l/by-id/...`, not `/dev/videoN`) in `robot.yaml`; 640×480 @ 30 fps.
- [ ] **Step 3:** `scripts/record.sh <task> <n_episodes> <instruction>` calling `lerobot-record` with cameras `wrist` and `front`, `--dataset.repo_id=$HF_USER/so101_<task>`, 30 s episodes, 10 s reset, push to the Hub. `HF_USER` read from `.env`.
- [ ] **Step 4:** 2 test episodes, inspected with the LeRobot dataset visualizer. Expected: both cameras sharp, joints consistent.
- [ ] **Step 5:** Record 50 `cube_to_box` episodes covering the zone uniformly, alternating 2–3 colors, ~10 episodes with distractor objects on the table, lighting similar to what is planned for the demo.

### Task R3: ACT training and async inference test

**Files:**
- Create: `scripts/train-act.sh`, `scripts/policy-server.sh`

- [ ] **Step 1:** `scripts/train-act.sh <task> <version>` on the desktop: `lerobot-train --policy.type=act --dataset.repo_id=$HF_USER/so101_<task> --output_dir=outputs/act_<task>_<version> --policy.device=cuda --policy.repo_id=$HF_USER/act_so101_<task>_<version>` (pushes the model). Each new version = new repo `_v2`, `_v3`… (the catalog points to a repo, not a revision).
- [ ] **Step 2:** Overnight training. Expected: loss decreasing, model on the Hub.
- [ ] **Step 3:** `scripts/policy-server.sh`: starts the policy server on the desktop on `0.0.0.0:8081`.
- [ ] **Step 4:** From the Pi, by hand, `robot_client` pointing to the desktop with the freshly trained policy and torque-on-disconnect disabled. Expected: the arm attempts the task; at the end of the run it stays under torque.
  Fallback if the Pi↔desktop chain causes trouble (latency, camera bandwidth, drivers) and can't be fixed within half a day: follower and cameras on the desktop, robotd runs there (same code, `robot.yaml` with the desktop's devices). Record it in the runbook.
- [ ] **Step 5:** 10 informal attempts; record the success rate in the runbook. Below 5/10 → 30 more episodes (R2 Step 5) and a new version.

---

## Phase 1 — `so101lab` software (can be prepared before the 30th, testable without hardware)

Layout:

```
pyproject.toml                    # so101lab package, entry point `so101lab`
src/so101lab/config.py            # RobotSettings from robot.yaml
src/so101lab/catalog.py           # Task + load_catalog
src/so101lab/client_cmd.py        # robot_client argv for a Task
src/so101lab/hardware.py          # Arm protocol, FakeArm, LeRobotArm, interpolate
src/so101lab/runner.py            # EpisodeRunner: state, subprocess, timeout, stop
src/so101lab/robotd/app.py        # FastAPI: API and metrics
src/so101lab/robotd/metrics.py    # Prometheus metrics
src/so101lab/robotd_client.py     # robotd HTTP client (used by MCP and eval)
src/so101lab/mcp_server.py        # MCP server
src/so101lab/eval.py              # interactive evaluation
src/so101lab/cli.py               # robotd | mcp | eval | capture-home | validate-catalog
deploy/k8s/robotd/                # robot.yaml, tasks.yaml, manifests, kustomization, fleet.yaml
deploy/k8s/robotd-fake/           # second, simulated "device"
deploy/k8s/monitoring/            # Prometheus + Grafana + dashboard
deploy/docker/robotd.Dockerfile
tests/
```

### Task 1: Bootstrap, configuration and catalog

**Files:**
- Create: `pyproject.toml`, `.gitignore` (includes `.env`, `outputs/`), `.env.example`, `src/so101lab/__init__.py`, `src/so101lab/config.py`, `src/so101lab/catalog.py`, `deploy/k8s/robotd/robot.yaml`, `deploy/k8s/robotd/tasks.yaml`
- Test: `tests/test_config.py`, `tests/test_catalog.py`

**Interfaces:**
- Produces:
  - `CameraSettings(name: str, device: str, width: int = 640, height: int = 480, fps: int = 30)`
  - `RobotSettings(robot_type: str = "so_follower", robot_id: str = "follower", port: str, cameras: list[CameraSettings], snapshot_camera: str = "front", policy_server: str, fps: int = 30, home_pose: dict[str, float] = {}, home_duration_s: float = 2.0)`; validator: `snapshot_camera` must be one of the camera names.
  - `load_settings(path: Path) -> RobotSettings`
  - `Task(name: str  # ^[a-z0-9_]+$, description: str, policy_type: Literal["act", "smolvla"] = "act", policy_repo: str, timeout_s: float = 30.0  # 0 < x <= 300, instruction: str | None = None)`; `Task.prompt` property = `instruction or description`.
  - `class CatalogError(ValueError)`; `load_catalog(path: Path) -> dict[str, Task]` (YAML: `tasks: [ ... ]`).

- [ ] **Step 1:** `uv init --package` in the repo, base dependencies (`fastapi uvicorn prometheus-client httpx mcp pydantic pyyaml opencv-python-headless`), extra `robot = ["lerobot[feetech]"]`, dev `pytest respx`.
- [ ] **Step 2: Failing tests**

```python
def test_load_catalog_ok(tmp_path): ...  # 2 tasks → dict with keys "cube_to_box", "box_to_zone"; timeout_s == 30.0
def test_duplicate_name_raises(tmp_path): ...  # CatalogError, message contains the duplicate name
def test_timeout_out_of_range_raises(tmp_path): ...  # timeout_s 0 and 301 → CatalogError
def test_bad_name_raises(tmp_path): ...  # "Cube To Box" → CatalogError
def test_repo_catalog_is_valid():  # Review Focus 4
    tasks = load_catalog(REPO / "deploy/k8s/robotd/tasks.yaml")
    assert {"cube_to_box", "box_to_zone"} <= tasks.keys()
def test_snapshot_camera_must_exist(tmp_path): ...  # snapshot_camera="top" with cameras wrist/front → ValidationError
def test_repo_robot_yaml_loads(): ...  # load_settings on the repo file
```

- [ ] **Step 3:** Run `uv run pytest tests/test_catalog.py tests/test_config.py -v` → FAIL (missing modules).
- [ ] **Step 4:** Implement `config.py` and `catalog.py`. `load_catalog` turns every `ValidationError` into a `CatalogError`. Initial `tasks.yaml` with the two tasks (English descriptions, for future SmolVLA use: "Put the cube in the box", "Take the cube out of the box and put it back in the zone"), `policy_repo` = `<HF_USER>/act_so101_<task>_v1`.
- [ ] **Step 5:** Run → PASS. Commit `feat: settings and task catalog`.

### Task 2: Robot client command

**Files:**
- Create: `src/so101lab/client_cmd.py`
- Test: `tests/test_client_cmd.py`

**Interfaces:**
- Consumes: `Task`, `RobotSettings` (Task 1)
- Produces: `build_client_command(task: Task, settings: RobotSettings, python: str = sys.executable) -> list[str]`

- [ ] **Step 1: Failing tests** — the argv contains exactly: `-m lerobot.async_inference.robot_client` (or the module recorded in R1), `--robot.type=so_follower`, `--robot.id=follower`, `--robot.port=<port>`, `--robot.disable_torque_on_disconnect=false` (flag name from R1), `--robot.cameras=<json>` with keys `wrist` and `front` and their device/width/height/fps, `--server_address=<policy_server>`, `--policy_type=act`, `--pretrained_name_or_path=<policy_repo>`, `--policy_device=cuda`, `--task=<task.prompt>`, `--fps=30`. Extra test: no argv element contains the `HF_TOKEN` value set with `monkeypatch.setenv`.
- [ ] **Step 2:** Run → FAIL.
- [ ] **Step 3:** Implement; the JSON shape of `--robot.cameras` is the one documented in the runbook (R1 Step 3).
- [ ] **Step 4:** Run → PASS. Commit `feat: robot client command builder`.

### Task 3: Arm interface (Fake + LeRobot) and capture-home

**Files:**
- Create: `src/so101lab/hardware.py`, `src/so101lab/cli.py` (`capture-home` subcommand)
- Test: `tests/test_hardware.py`

**Interfaces:**
- Consumes: `RobotSettings`
- Produces:
  - `MotorTelemetry(position: float, temperature_c: float, voltage_v: float, load: float)` (dataclass)
  - `class Arm(Protocol)`: `connect() -> None`, `disconnect() -> None`, `is_connected: bool`, `read_joints() -> dict[str, float]`, `read_telemetry() -> dict[str, MotorTelemetry]`, `go_home(pose: dict[str, float], duration_s: float) -> None`, `snapshot(camera: str) -> bytes` (JPEG)
  - `class CameraError(RuntimeError)`
  - `interpolate(start: dict[str, float], end: dict[str, float], steps: int) -> list[dict[str, float]]` (linear, last element == `end`, only the keys of `end`)
  - `FakeArm(settings)`: 6 motors (`shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll, gripper`), deterministic telemetry (temperature 35.0), records calls in `calls: list[str]`, `snapshot` returns a valid 64×48 JPEG; `fail_camera: bool` → `CameraError`.
  - `LeRobotArm(settings)`: LeRobot robot **without cameras** (only the robot client opens the cameras), `disable_torque_on_disconnect=False`; telemetry via `robot.bus.sync_read("Present_Temperature" | "Present_Voltage" | "Present_Load")` (raw voltage / 10, to be checked on the servo); `go_home` = `interpolate` at `settings.fps` over `duration_s` with `send_action`; `snapshot` = OpenCV on the camera device, single frame, released immediately, 2 s timeout → `CameraError`.

- [ ] **Step 1: Failing tests** — `interpolate` (3 steps between 0 and 30 → 10, 20, 30; extra keys in `start` ignored); `FakeArm.go_home` records `"go_home"` and brings `read_joints()` to the pose; `FakeArm.snapshot` starts with `b"\xff\xd8"`; `fail_camera` → `CameraError`.
- [ ] **Step 2:** Run → FAIL. **Step 3:** Implement. **Step 4:** Run → PASS.
- [ ] **Step 5:** `so101lab capture-home --config deploy/k8s/robotd/robot.yaml`: reads the current positions with `LeRobotArm` and writes `home_pose` to the file. Manual check on the Pi: arm placed in the rest pose by hand → command → values in `robot.yaml`; `go_home` from another pose gets there smoothly without dropping.
- [ ] **Step 6:** Commit `feat: arm interface, fake arm, capture-home`.

### Task 4: EpisodeRunner

**Files:**
- Create: `src/so101lab/runner.py`
- Test: `tests/test_runner.py`

**Interfaces:**
- Consumes: `Arm`, `Task`, `RobotSettings`, `build_client_command`
- Produces:
  - `class RunnerState(StrEnum)`: `IDLE, RUNNING, HOMING`
  - `EpisodeResult(episode_id: str, task: str, started_at: float, duration_s: float, ended_by: Literal["timeout", "stop", "exited", "error"], outcome: Literal["unknown", "success", "failure"] = "unknown", error: str | None = None)` (pydantic)
  - `class ProcessHandle(Protocol)`: `poll() -> int | None`, `interrupt() -> None` (SIGINT), `kill() -> None`, `wait(timeout: float) -> int`, `stderr_tail(lines: int = 20) -> str`
  - `Spawner = Callable[[list[str]], ProcessHandle]`; `subprocess_spawner(argv) -> ProcessHandle` (Popen, stderr to a temp file, new process group)
  - `class Busy(RuntimeError)`
  - `EpisodeRunner(arm: Arm, spawner: Spawner, settings: RobotSettings, clock: Callable[[], float] = time.monotonic, poll_interval: float = 0.1)` with: `state`, `current_task: str | None`, `elapsed_s: float | None`, `last_result: EpisodeResult | None`, `outcomes: dict[str, dict[str, int]]` (per task: `success`, `failure`), `run(task) -> EpisodeResult` (blocking), `stop() -> bool`, `home() -> None`, `set_outcome(episode_id, outcome) -> EpisodeResult` (`KeyError` if it isn't the last episode).

`run` sequence: non-blocking lock (otherwise `Busy`) → `RUNNING` → `arm.disconnect()` → spawn → poll until: process exited, `stop()` called, or `elapsed >= task.timeout_s` → if still alive `interrupt()`, `wait(5)`, then `kill()` → `arm.connect()` → `HOMING` + `go_home` → `IDLE`. `ended_by`: `stop` / `timeout` / `exited` (exit code 0) / `error` (exit code ≠ 0, `error = stderr_tail()`). Homing also happens after an error (`finally` block).

- [ ] **Step 1: Failing tests** (FakeArm + fake spawner with a controlled clock):

```python
def test_timeout_interrupts_and_homes(): ...      # ended_by == "timeout"; handle interrupted; arm.calls ends with ["connect", "go_home"]; state IDLE
def test_stop_during_run(): ...                   # stop() from another thread → True; ended_by == "stop"
def test_stop_when_idle_is_noop(): ...            # Review Focus 3: stop() → False twice, no exception
def test_run_while_running_raises_busy(): ...     # Review Focus 1: second run → Busy; spawner called only once
def test_client_crash_reports_error(): ...        # Review Focus 2: exit 1 after 0.2 s → ended_by "error", error contains the stderr tail, state IDLE, go_home called
def test_kill_if_interrupt_ignored(): ...         # wait(5) times out → kill() called
def test_set_outcome_counts(): ...                # success → outcomes["cube_to_box"]["success"] == 1; unknown id → KeyError
def test_home_while_running_raises_busy(): ...
```

- [ ] **Step 2:** Run → FAIL. **Step 3:** Implement. **Step 4:** Run → PASS. Commit `feat: episode runner`.

### Task 5: robotd — HTTP API and metrics

**Files:**
- Create: `src/so101lab/robotd/__init__.py`, `src/so101lab/robotd/app.py`, `src/so101lab/robotd/metrics.py`; Modify: `src/so101lab/cli.py` (`robotd` subcommand)
- Test: `tests/test_robotd.py`

**Interfaces:**
- Consumes: `EpisodeRunner`, `Arm`, `load_catalog`, `load_settings`
- Produces: `create_app(runner: EpisodeRunner, arm: Arm, catalog: dict[str, Task], settings: RobotSettings, telemetry_interval: float = 1.0) -> FastAPI`; CLI `so101lab robotd --config PATH --catalog PATH [--fake] [--host 0.0.0.0] [--port 8080]`.

| Method and path | Response |
|---|---|
| `GET /healthz` | `{"ok": true}` |
| `GET /tasks` | `[{name, description, timeout_s, policy_repo}]` |
| `POST /tasks/{name}/run` | `EpisodeResult` · 404 unknown task · 409 busy (sync endpoint, runs in the threadpool) |
| `POST /stop` | `{"stopped": bool}` |
| `POST /home` | 200 `{"ok": true}` · 409 busy |
| `GET /status` | `{state, current_task, active_policy, elapsed_s, last_result, outcomes, telemetry, telemetry_stale}` |
| `GET /snapshot` | `image/jpeg` · 409 if not idle · 503 on `CameraError` |
| `POST /episodes/{id}/outcome` body `{"outcome": "success"\|"failure"}` | `EpisodeResult` · 404 |
| `GET /metrics` | Prometheus format |

Telemetry: background thread reading `arm.read_telemetry()` every `telemetry_interval`, only in `IDLE`; during episodes the last value is kept with `telemetry_stale=true` (the serial port belongs to the robot client). Metrics: `so101_motor_temperature_celsius{motor}`, `so101_motor_voltage_volts{motor}`, `so101_motor_load{motor}`, `so101_motor_position{motor}`, `so101_state{state}` (1 on the active state), `so101_telemetry_stale`, `so101_episodes_total{task,ended_by}`, `so101_outcomes_total{task,outcome}`, `so101_episode_duration_seconds{task}` (histogram), `so101_active_policy_info{task,policy_repo}`. Per-app Prometheus registry (not global) so tests are isolated.

- [ ] **Step 1: Failing tests** (TestClient, FakeArm, fake spawner): one test per table row, plus: concurrent run → 409 (Review Focus 1); `GET /snapshot` during a run → 409, and with `fail_camera` → 503 in < 3 s (Review Focus 5); `/metrics` contains `so101_motor_temperature_celsius{motor="gripper"} 35.0` and, after a run ending in timeout, `so101_episodes_total{ended_by="timeout",task="cube_to_box"} 1.0`; with `HF_TOKEN=sekret` in the environment, `sekret` appears in no response (`/status`, `/tasks`, `/metrics`, errors).
- [ ] **Step 2:** Run → FAIL. **Step 3:** Implement. **Step 4:** Run → PASS.
- [ ] **Step 5:** Local smoke test: `uv run so101lab robotd --fake --config deploy/k8s/robotd/robot.yaml --catalog deploy/k8s/robotd/tasks.yaml & sleep 2; curl -s localhost:8080/status; kill %1` → JSON with `"state":"idle"`.
- [ ] **Step 6:** Commit `feat: robotd http api and metrics`.

> With `--fake`, the spawner runs `python -c "import time; time.sleep(3600)"` instead of the robot client, so timeout and stop behave realistically.

### Task 6: robotd client and MCP server

**Files:**
- Create: `src/so101lab/robotd_client.py`, `src/so101lab/mcp_server.py`; Modify: `cli.py` (`mcp` subcommand)
- Test: `tests/test_robotd_client.py`, `tests/test_mcp_server.py`

**Interfaces:**
- Consumes: robotd API (Task 5)
- Produces:
  - `RobotdClient(base_url: str, transport: httpx.BaseTransport | None = None)` with `list_tasks() -> list[dict]`, `run_task(name: str) -> dict` (HTTP timeout = the task's `timeout_s` + 60 s), `stop() -> dict`, `home() -> dict`, `status() -> dict`, `snapshot() -> bytes`, `set_outcome(episode_id: str, outcome: str) -> dict`; `class RobotdError(RuntimeError)` with `status_code` and a readable message (409 → "robot busy", 404 → "unknown task: <name>", 503 → "camera unavailable").
  - `build_server(client: RobotdClient) -> FastMCP` with tools `list_tasks`, `run_task(name)`, `stop`, `go_home`, `get_status`, `get_snapshot` (returns a JPEG MCP image). The tool descriptions tell the agent: after `run_task` the outcome is "unknown", use `get_snapshot` to judge it; `stop` halts the arm immediately.
  - CLI `so101lab mcp --robotd URL [--transport stdio|http] [--port 8090]` (default stdio; `http` = streamable HTTP for Open WebUI in Task 10; `ROBOTD_URL` as default).

- [ ] **Step 1: Failing tests** — `RobotdClient` with `httpx.MockTransport`: every method calls the right path; 409/404/503 → `RobotdError` with the messages above. `build_server`: the registered tool names are exactly the six listed; `run_task` on a busy robot returns an error text to the agent instead of raising.
- [ ] **Step 2:** Run → FAIL. **Step 3:** Implement. **Step 4:** Run → PASS.
- [ ] **Step 5:** End-to-end test with fake robotd: register the MCP server in Claude Code (`claude mcp add so101 -- uv run so101lab mcp --robotd http://localhost:8080`) and ask "what tasks can you do? run cube_to_box and tell me if it succeeded". Expected: the agent calls `list_tasks`, `run_task`, `get_snapshot`.
- [ ] **Step 6:** Commit `feat: mcp server`.

### Task 7: Interactive evaluation

**Files:**
- Create: `src/so101lab/eval.py`; Modify: `cli.py` (`eval` subcommand)
- Test: `tests/test_eval.py`

**Interfaces:**
- Consumes: `RobotdClient`
- Produces:
  - `EvalAttempt(index: int, episode_id: str, outcome: Literal["success", "failure"], duration_s: float, ended_by: str)`; `EvalReport(task: str, policy_repo: str, started_at: str, attempts: list[EvalAttempt])` with `successes: int`, `summary: str` → `"7/10 (70%)"`.
  - `run_eval(client: RobotdClient, task: str, n: int, ask: Callable[[str], str], out_dir: Path) -> EvalReport`; writes `out_dir/<YYYY-MM-DD-HHMM>-<task>.json` and appends a line to `out_dir/README.md`.
  - CLI `so101lab eval --task cube_to_box -n 10 --robotd URL --out evals/`.

For each attempt: `ask("Reposition the cube, then press Enter")` → `run_task` → `ask("[s]uccess / [f]ailure / [r]etry")`. `r` = attempt not counted (fault, manual intervention) and repeated. Episode with `ended_by == "error"` → reported on screen and automatically repeated, not counted. Each outcome is sent to robotd with `set_outcome` (feeds the metrics).

- [ ] **Step 1: Failing tests** — fake answers `["", "s", "", "f", "", "r", "", "s"]` with n=3 → 3 counted attempts, `summary == "2/3 (67%)"`, `set_outcome` called 3 times, JSON written; errored episode → repeated and not counted; invalid input ("x") → asked again.
- [ ] **Step 2:** Run → FAIL. **Step 3:** Implement. **Step 4:** Run → PASS. Commit `feat: interactive evaluation`.

---

## Phase 2 — Infrastructure (Wed Dec 2; can be cut in this order: Fleet → K3s → Grafana)

### Task 8: Container, K3s and Grafana

**Files:**
- Create: `deploy/docker/robotd.Dockerfile`, `deploy/k8s/robotd/{kustomization.yaml,deployment.yaml,service.yaml}`, `deploy/k8s/robotd-fake/{kustomization.yaml,deployment-patch.yaml}`, `deploy/k8s/monitoring/{fleet.yaml,prometheus-values.yaml,grafana-values.yaml,kustomization.yaml}`, `deploy/k8s/monitoring/dashboards/so101.json`, `scripts/build-image.sh`
- Test: `tests/test_deploy.py`

Decisions:
- Image built **on the node itself** and imported with `k3s ctr images import` (`scripts/build-image.sh`), `imagePullPolicy: Never`: no registry.
- K3s server on the desktop (label `so101/role=desktop`), Pi as agent (label `so101/role=robot`).
- robotd: `nodeSelector so101/role=robot`, `privileged: true`, hostPath `/dev` and the calibration folder, **`Recreate`** strategy (never two pods on the same serial port), NodePort `30080`. `robot.yaml` and `tasks.yaml` via `configMapGenerator` (a commit to the catalog = pod rollout).
- robotd-fake: kustomize overlay with `--fake`, `nodeSelector so101/role=desktop`, NodePort `30081`: it is the fleet's second "device".
- Prometheus (chart `prometheus-community/prometheus`, server only) scraping both Services; Grafana (chart `grafana/grafana`) with datasource and dashboard provisioned from ConfigMaps (sidecar with label `grafana_dashboard`).
- Dashboard: per-motor temperature and load, state, episodes by outcome, episode duration, active policy, `instance` variable to pick the real or fake robot.

- [ ] **Step 1: Failing tests** — `kubectl kustomize deploy/k8s/robotd` and `deploy/k8s/robotd-fake` succeed (subprocess); the robotd Deployment has `strategy.type == "Recreate"` and nodeSelector `robot`; the fake one has `--fake` in its args; every `expr` in the dashboard JSON uses only metrics defined in `so101lab.robotd.metrics`.
- [ ] **Step 2:** Run → FAIL. **Step 3:** Write Dockerfile, manifests and dashboard. **Step 4:** Run → PASS.
- [ ] **Step 5:** On the hardware: install K3s (server on the desktop, agent on the Pi), label the nodes, build+import the images, `kubectl apply -k` the three folders. Expected: `curl http://<pi>:30080/status` responds, Grafana shows both the real and the fake temperatures.
- [ ] **Step 6:** Commit `feat: k3s deployment and grafana dashboard`.

### Task 9: Fleet GitOps

**Files:**
- Create: `deploy/k8s/robotd/fleet.yaml`, `deploy/k8s/robotd-fake/fleet.yaml`, `deploy/fleet/gitrepo.yaml`, "Fleet" section in `docs/runbook.md`

- [ ] **Step 1:** Repo published on GitHub (public, no secrets: check with `git grep -iE "hf_|token|sk-"` → no results).
- [ ] **Step 2:** Install standalone Fleet via Helm on the desktop; `gitrepo.yaml` points to the repo, branch `main`, paths `deploy/k8s/robotd`, `deploy/k8s/robotd-fake`, `deploy/k8s/monitoring`.
- [ ] **Step 3:** Demo check: a commit changing `cube_to_box`'s `policy_repo` from `_v1` to `_v2` → Fleet applies it → the robotd pod is recreated → `GET /status` and `so101_active_policy_info` show `_v2` on both the real and the fake robot.
- [ ] **Step 4:** Commit `feat: fleet gitops`.

### Task 10: Local LLM agent (private AI)

**Files:**
- Create: `deploy/local-ai/compose.yaml` (or K3s manifests if K3s is up), `docs/local-ai.md`, `evals/agent/` (comparison results)

Decisions: Ollama + Open WebUI on the desktop with access to the 3090; the MCP server (Task 6) runs with `--transport http` and is registered in Open WebUI as a tool server; Open WebUI listens on the LAN only.

- [ ] **Step 1:** Start Ollama + Open WebUI. Expected: working chat with a local model.
- [ ] **Step 2:** Compare 2–3 open models with tool calling + vision: with the ACT policy server loaded, `nvidia-smi` shows > 1 GB of free GPU memory and ACT inference doesn't slow down (robot client fps unchanged).
- [ ] **Step 3:** Register the MCP server in Open WebUI. Expected: the model sees the six tools.
- [ ] **Step 4:** Fixed scenario, 5 runs per model (and for Claude as a reference), with fake and then real robotd: "what tasks can you do?", "put the cube in the box and check you did it", "stop!" during an episode. Record in `evals/agent/` whether the right tools were called, whether the photo was judged correctly, and response time.
- [ ] **Step 5:** Pick the model for the demo; if no local model calls the tools reliably (< 4/5 in the scenario), the demo uses Claude and the comparison goes into the README. Commit `feat: local llm agent`.

---

## Phase 3 — Hackweek and wrap-up

### Task 11: Official evaluation, task 2, demo and documentation

- [ ] **Step 1 (Mon):** First autonomous run of `cube_to_box` via robotd; `so101lab eval -n 10`. If < 7/10: more episodes + new version (R2/R3), repeat.
- [ ] **Step 2 (Mon–Tue):** Record 50 `box_to_zone` episodes, overnight training, 10-attempt eval.
- [ ] **Step 3 (Thu):** Agent demo: "put the cube in the box, check you did it, then put it back" (1→2 chain with `get_snapshot`).
- [ ] **Step 4 (Fri):** `README.md` (what it is, architecture, evaluation results with links to the reports in `evals/`, links to datasets and models on the Hub, video), demo video edited from the third camera's footage, hackweek page.
- [ ] **Step 5:** Commit `docs: results and demo`.

### Stretch: language-conditioned SmolVLA

Only if Task 11 Steps 1–3 are done. Dataset `so101_pick_color` with 3 cubes on the table and a per-episode instruction ("Put the red cube in the box"), SmolVLA fine-tuning, new catalog task with `policy_type: smolvla`; `run_task` accepts an optional `instruction` parameter that overrides `Task.prompt` (added to Tasks 2/5/6 with their tests).
