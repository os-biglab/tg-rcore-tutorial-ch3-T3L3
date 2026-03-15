# ch3-T1L1 软件架构总览

本文描述 `tg-rcore-tutorial-ch3-T1L1` 作为独立 crate 的实现结构、执行路径与模块分工。

## 1. 系统定位

`ch3-T1L1` 是一个运行在 RISC-V S 态的 `no_std` 裸机内核样例，位于教程第三章，目标是从批处理过渡到多道程序与分时多任务。

当前 crate 的核心能力：

- 多任务并发驻留（多个用户程序同时加载）；
- 时间片轮转调度（默认抢占式，时钟中断驱动）；
- 协作式让出（`yield`）；
- 基础系统调用（`write` / `exit` / `clock_gettime`）；
- 练习扩展系统调用 `trace`（feature `exercise` 场景）。

该项目强调教学可读性与最小实现闭环，不包含进程隔离与复杂内存保护机制。

---

## 2. 目录与模块职责

```text
tg-rcore-tutorial-ch3-T1L1/
├── .cargo/config.toml         # 默认目标平台、QEMU runner、tg-user 配置
├── build.rs                   # 生成 linker.ld，拉取/构建用户程序并生成 APP_ASM
├── Cargo.toml                 # crate 元信息、features（coop/exercise）与依赖
├── README.md                  # 章节文档与使用说明
├── exercise.md                # 练习题描述（sys_trace）
└── src/
    ├── main.rs                # 入口、调度主循环、Trap 分发、syscall 接口实现
    └── task.rs                # TaskControlBlock、syscall 计数、调度事件
```

---

## 3. 分层架构

```text
应用行为（用户程序 ecall / 运行）
      │
      ▼
系统调用接口层（main.rs::impls::SyscallContext）
      │
      ▼
任务控制与调度层（task.rs + main.rs 主循环）
      │
      ▼
硬件与特权层（riscv CSR + tg-sbi + sret/trap）
      │
      ▼
QEMU virt (RISC-V)
```

### 3.1 `main.rs`：系统编排层

负责内核生命周期主路径：

1. `_start` 手动建立内核栈并跳转 `rust_main`；
2. `rust_main` 清零 BSS、初始化控制台与 syscall 子系统；
3. 加载用户程序并初始化 TCB 数组；
4. 启用 S 态时钟中断（默认抢占）；
5. 进入轮转调度循环，处理：
   - `SupervisorTimer`（时间片到期）；
   - `UserEnvCall`（系统调用）；
   - 其他异常/中断（终止任务）；
6. 所有任务结束后调用 `shutdown(false)` 关机。

### 3.2 `task.rs`：任务模型与事件层

负责单任务状态管理：

- `TaskControlBlock` 包含：
  - 用户上下文 `LocalContext`；
  - 任务完成标记 `finish`；
  - 每任务独立用户栈；
  - 系统调用计数数组 `syscall_counts`；
- `handle_syscall()` 负责：
  - 从寄存器取 syscall id/参数；
  - 调用 `tg_syscall::handle`；
  - 写回返回值并推进 `sepc`；
  - 生成 `SchedulingEvent::{None,Yield,Exit,UnsupportedSyscall}`。

### 3.3 `build.rs`：构建编排层

负责“内核 + 用户程序”一体化构建：

- 生成链接脚本 `linker.ld`；
- 按 feature 选择测试集合：
  - 默认：`ch3`；
  - `exercise`：`ch3_exercise`；
- 拉取或复用 `tg-rcore-tutorial-user`；
- 编译用户程序并生成内嵌汇编 `app.asm`（环境变量 `APP_ASM`）。

---

## 4. 启动与运行流程

### 4.1 构建期（build-time）

1. Cargo 触发 `build.rs`；
2. 生成 `linker.ld` 与 `app.asm`；
3. 将用户程序二进制以内联数据方式打包进内核镜像。

### 4.2 运行期（run-time）

1. `_start` 设栈后跳转 `rust_main`；
2. 内核加载所有用户 app 到 TCB；
3. 调度循环以 Round-Robin 运行未完成任务；
4. trap 返回后根据事件继续当前任务或切换任务；
5. `remain == 0` 后正常关机。

---

## 5. 调度与系统调用闭环

### 5.1 抢占式（默认）

- 每次进入用户态前设置 `set_timer(time + 12500)`；
- 触发 `SupervisorTimer` 后清除定时器并切换到下一任务。

### 5.2 协作式（feature `coop`）

- 不设置定时中断；
- 由用户任务通过 `yield` 主动让出 CPU。

### 5.3 syscall 路径

- 用户态 `ecall` -> `UserEnvCall`；
- 主循环调用 `TaskControlBlock::handle_syscall()`；
- 根据返回事件驱动调度决策；
- 在 `exercise` 场景中 `trace` 同样走该路径并参与计数。

---

## 6. 配置与外部依赖

### 6.1 关键配置

`.cargo/config.toml`：

- 默认目标：`riscv64gc-unknown-none-elf`；
- `runner`：`qemu-system-riscv64 -machine virt -nographic -bios none -kernel`；
- `TG_USER_*` 环境变量供 `build.rs` 拉取/定位用户程序 crate。

### 6.2 主要依赖

- `tg-sbi`：控制台输出、定时器、关机；
- `tg-linker`：内核/应用链接布局与元数据；
- `tg-kernel-context`：上下文切换；
- `tg-syscall`：系统调用分发框架；
- `riscv`：CSR 读写。

---

## 7. 当前实现边界

为保持第三章教学最小实现，当前不包含：

- 地址空间隔离与用户指针安全校验；
- 线程/进程层级抽象；
- 优先级调度与复杂就绪队列；
- 文件系统与块设备栈集成。

这些能力可在后续章节继续演进。