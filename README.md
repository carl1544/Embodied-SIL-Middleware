# Embodied-SIL-Middleware# AgentKit

[![Python 3.10](https://img.shields.io/badge/python-3.10-blue)](https://www.python.org/)
[![stdlib only](https://img.shields.io/badge/deps-stdlib%20only-success)](#)
[![tests](https://img.shields.io/badge/chaos%20tests-133%20passing-brightgreen)](tests/test_body.py)
[![stage](https://img.shields.io/badge/stage-SIL-orange)](#)

**A high-fault-tolerance, process-isolated body-control middleware for embodied agents. Pure software-in-the-loop. No third-party dependencies. No fabricated measurements.**

一个基于进程隔离与异步对抗的具身智能底层控制中间件。当前是纯软件在环（SIL）。零第三方依赖。没有测量就不造数。

The agent kernel (model ↔ tool loop, sandbox, memory, audit, fast / mid / slow) lives in the same repo and is documented below. The body is not inside that loop. The model does not stop because a joint failed to arrive. `body_servo` is a tool that hands a goal to a separate field process.

> Architecture diagram goes here. Paste your Mermaid block under this line.
>
> ```mermaid
> %% paste architecture here
> %% Task  --goal-->  Field (PD, own clock, saturate)
> %% Field --V/A-->  AbstractBoard  --angle-->  Field
> %% open_board() returns RogueBoard | PhoneBoard | RealBoard
> ```

## What this is / 这是什么

A task process sends a goal. A field process owns the clock, the PD reflex, and the saturation clamp. The plant, when a port is set, is opaque: the field only reads and writes one byte line. The stop condition is the measurement, not the step count.

任务进程只发目标。现场进程自己走钟，自己做 PD，自己限幅。给了端口，现场只读写一行字节。停不停只看实测。

This is SIL. The cut is not closed. `Current` talks to `SimBoard`, whose angle is the integral of the command inside the peer. `open_board()` currently returns `RogueBoard`, whose angle is a foreign clock. Neither is an encoder. A real shaft that the field cannot see the state of is the thing that cuts the joint.

## Architecture / 核心架构

| Layer | Where | Owns |
|---|---|---|
| Task | `agentkit/body/servo.py` `Body` / `Servo` | Goal only. Stops on \(\lvert \text{goal} - \text{measured} \rvert \le \text{tol}\). No plant. |
| Field | `agentkit/body/plant_proc.py` `Field` | Own `dt` clock. `PDReflex`: \(u = k_p e + k_d \dot{e}\). Saturation. |
| Wire | `agentkit/body/drive.py` | One line, 8N1. `M` + number. Voltage or current in, angle out. |
| HAL | `agentkit/body/board.py` `AbstractBoard` | `read_angle()` / `send_voltage()`. |
| Factory | `open_board()` in the same file | **One line.** Swap the peer by changing the `return`. |

The field does not import the plant when a port is set. The task does not import the field. The clock is the field's. A missed sample is not angle 0: the held measurement is reused, the derivative is 0 if the error did not change, and history is kept.

## The adversary / 流氓板

No hardware. The peer that `open_board()` returns is `RogueBoard`: a separate process with its own clock.

```text
angle = sin(ω t)          amplitude 1
voltage is parsed and discarded
```

The field cannot see this rule. It only reads the byte line. Zero voltage, the angle still moves. Voltage at the rail, the angle still stays in \([-1, 1]\). A goal of 2 never settles, because \(\lvert\sin\rvert \le 1\).

This is a code adversary, not a plant and not an encoder. It exists so the field is forced to track a number it did not compute. It is not a model of friction, deadzone, or sensor noise. Adding those inside the same integrator would not cut the joint.

`PhoneBoard` has no phone. `read_angle` and `send_voltage` raise. `RealBoard("")` raises. Neither falls back to `RogueBoard` or to `SimBoard`.

## Invariants / 工程底线

These are locked by `tests/test_body.py`. Breaking one fails the suite.

1. **No fabricated data.** Timeout, garbage, and a dead PTY return no angle. That is not 0. Disconnect does not fall back to a simulated plant.
2. **Dimension isolation.** The wire carries V or A (`accept_unit` rejects duty, `%`, Nm, pwm). The reply is an angle. The two are not the same quantity even when the numbers match.
3. **No derivative kick.** `set_goal` calls `reflex.reset()`, which clears `_prev_error`. The first tick after a goal change is P only.
4. **Saturation.** Default clamp is \([-8, 8]\), applied to the command before it is written. The unit stays V or A.
5. **Arrival is a measurement.** Settled means \(\lvert \text{goal} - \text{measured} \rvert \le \text{tol}\). There is no `settled=True` argument.

## Tests / 测试战绩

**133 / 133 chaos and invariant checks pass.**

```sh
python3 tests/test_body.py
```

Coverage includes: the plant has no angle-writer, the reflex never sees the plant, delay, saturation, a bad gain that must not be called done, empty ticks that keep history, a goal change that does not spike D, a dead PTY that does not crash and does not invent a sample, dirty bytes, stalls, and the `RogueBoard` path where a full-scale voltage cannot drag the angle out of the adversary's clock.

A pinned snapshot of an earlier suite lives in `tests/baseline/`. The live suite is `tests/test_body.py`.

## Swap the peer / 换一行

```python
# agentkit/body/board.py
def open_board() -> AbstractBoard:
    return RogueBoard()          # SIL adversary, today
    # return RealBoard("/dev/ttyACM0")   # when a port exists
    # return PhoneBoard()               # raises until a phone exists
```

`RealBoard` speaks the same byte line. It does not simulate. If the port cannot be opened, it fails. It does not quietly become `SimBoard`.

## What this is not / 不是什么

- Not a closed physical loop. The angle is still either the peer's integral (`SimBoard`) or a foreign clock (`RogueBoard`).
- Not ROS 2, and not a hardware-abstraction layer that hides a plant inside the field.
- Not a claim that 133 passing tests mean a shaft was turned. They mean the field refuses to lie when the shaft is absent.

---

## Agent kernel

一个纯 Python 标准库的自主 agent 内核，旁边接了一条还没碰到真机的身体。零第三方依赖。

内核是模型 ↔ 工具循环：受限沙箱、长期记忆、会话、审计、快 / 中 / 慢循环。身体不在这条循环里。任务进程只发目标，现场进程自己走钟。对面现在不是编码器。

> 八张分层图在 `docs/blueprints/`，逐框对应的实现见 [ARCHITECTURE.md](ARCHITECTURE.md)。
> 快循环到上下文压缩已经落在代码里。慢循环的训练器要外接 GPU，见 [deploy/README.md](deploy/README.md)。
> 身体是后加上的，见上面。它没有并进那些循环。

## 它做了什么

代码按架构图的层次组织：`shared/` 是横切的护栏与数据结构，`fast/` 是第一刀（快循环）。

| 文件 | 职责 | 关键设计 |
|---|---|---|
| `agentkit/llm.py` | 模型接入 | 只依赖 `urllib`；任何 OpenAI 兼容端点改两个环境变量即可；`MockLLM` 让你没 key 也能跑通链路 |
| `agentkit/loop.py` | **主循环** | 只做编排：Brain 决策 → Policy 检查 → 执行 → Observation → Retry → History |
| `agentkit/fast/brain.py` | 决策盒 | 吃「提示 + 历史 + 工具 schema」，吐 `Action`；流式/非流式与空响应兜底都收在这里 |
| `agentkit/shared/schema.py` | 数据结构 | `Action`（code/final/ask_user）+ `Observation`（带 exit_code / error_kind） |
| `agentkit/shared/policy.py` | **安全绳·Policy** | 无网 / 超时 / 目录限制三条检查，拒绝时给出理由与规则名，理由进审计 |
| `agentkit/shared/audit.py` | **安全绳·Audit** | append-only JSONL + **链式哈希**（改一行即断链）；`verify()` / `replay()` / `tail()` |
| `agentkit/fast/retry.py` | 错误分类 | 7 类错误 → 该不该重试 + 「最小修复」提示（越界与重复**不重试**） |
| `agentkit/fast/artifacts.py` | 结构化产物 | 自动识别 md / html / 表格，落盘 `data/artifacts/` 并维护索引 |
| `agentkit/fast/history.py` | 轨迹纪录 | 面向机器，一步一条（动作/观测/分类/耗时/是否重试）—
