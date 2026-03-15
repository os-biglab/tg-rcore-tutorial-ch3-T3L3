# ch3-T1L1 Exercise 实现报告（sys_trace）

本文对应 `exercise.md` 要求，说明当前 `tg-rcore-tutorial-ch3-T1L1` 是如何实现 `sys_trace`（ID=410）以及系统调用计数能力的。

## 1. 目标与约束

根据题目要求，需要在内核中新增 `trace` 系统调用，具备三种语义：

1. `trace_request = 0`：按 `*const u8` 读取当前任务内存；
2. `trace_request = 1`：按 `*mut u8` 写入当前任务内存（`data as u8`）；
3. `trace_request = 2`：查询当前任务某 syscall 编号调用次数，且本次 `trace` 调用也要被统计；
4. 其他 `trace_request` 返回 `-1`。

实现约束：

- 保持 ch3 现有 `no_std` / 裸机风格；
- 不实现地址隔离前提下的安全检查（与题目说明一致）；
- 尽量最小改动接入既有 syscall/调度框架。

---

## 2. 实现方案概览

本实现采用“两处改动、一个闭环”：

1. 在 `task.rs` 中为每个任务维护 syscall 计数；
2. 在 `main.rs` 的 `impl Trace for SyscallContext` 中实现 `trace` 三种请求；
3. 借助既有 `handle_syscall()` 路径，保证 `trace` 本次调用计入统计后再查询。

---

## 3. 代码落点与职责

## 3.1 `src/task.rs`：系统调用计数

### 3.1.1 计数结构

- 新增（或扩展）每任务计数字段：
  - `syscall_counts: [usize; 5]`
- 覆盖 syscall：`WRITE/EXIT/SCHED_YIELD/CLOCK_GETTIME/TRACE`。

### 3.1.2 ID 到索引映射

- 通过 `syscall_idx(id: SyscallId) -> Option<usize>` 将离散 syscall ID 映射到计数数组索引。

### 3.1.3 计数时机（关键）

在 `handle_syscall()` 里：

1. 先从 `a7` 读出 syscall id；
2. 若可映射，先 `self.syscall_counts[idx] += 1`；
3. 再调用 `tg_syscall::handle(...)` 进入具体 syscall 实现。

这保证了当 `trace_request=2` 查询次数时，“当前这次 trace 调用”已经计入统计，满足题目要求。

### 3.1.4 查询接口

- 提供 `syscall_count(&self, syscall_id: usize) -> usize`；
- 未覆盖的 syscall id 返回 `0`。

---

## 3.2 `src/main.rs`：`trace` 系统调用实现

在 `impl Trace for SyscallContext` 中实现：

- `trace_request = 0`：`unsafe { *(id as *const u8) as isize }`
- `trace_request = 1`：`unsafe { *(id as *mut u8) = data as u8 }; 0`
- `trace_request = 2`：
  - 通过 `caller.entity` 还原当前任务 `TaskControlBlock`；
  - 调用 `tcb.syscall_count(id)` 返回计数。
- 其他分支：`-1`。

实现遵循题目提示：在 ch3 尚无地址空间隔离前，允许不做用户指针安全检查。

---

## 4. 与原调度框架的集成方式

`trace` 无需新增调度事件类型，沿用已有机制：

1. 用户态执行 `ecall`；
2. 内核进入 `UserEnvCall` 分支；
3. `TaskControlBlock::handle_syscall()` 统一分发并写回返回值；
4. `trace` 作为普通 syscall 返回 `SchedulingEvent::None`，任务继续运行。

因此改动集中且不会破坏 ch3 原有的时间片调度逻辑。

---

## 5. 验证建议（对应练习要求）

在 `tg-rcore-tutorial-ch3-T1L1` 目录执行：

```bash
cargo run --features exercise
```

或运行脚本：

```bash
./test.sh exercise
```

重点观察：

- 读写请求是否按预期修改同一任务内存字节；
- `trace_request=2` 的统计是否包含当前这次 `trace` 调用。

---

## 6. 设计取舍与边界

当前实现是“通过练习的最小版本”，主要取舍：

- 优点：改动小、路径清晰、与现有框架耦合低；
- 限制：
  - `unsafe` 指针读写不做边界/权限检查；
  - 计数表仅覆盖当前章节关注的 syscall；
  - 不支持跨任务追踪。

这些限制与 ch3 教学阶段一致，可在后续引入地址空间与进程模型后继续扩展。