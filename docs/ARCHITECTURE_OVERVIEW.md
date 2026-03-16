# tg-rcore-tutorial-ch3-T3L3 架构总览

## 1. 目标与范围

`tg-rcore-tutorial-ch3-T3L3` 基于 chapter 3 的多道程序内核，扩展目标为：

- 支持运行用户态贪吃蛇应用；
- 支持两种输入语义：轮询式输入（polling）与中断式风格输入（interrupt-like）；
- 保持 chapter 3 原有的多任务调度主线（TCB + 时钟中断 + syscall）。

本 crate 仍为 `no_std`、`no_main` 的 RISC-V S-mode 教学内核。

## 2. 总体模块结构

- `src/main.rs`
  - 内核入口 `_start` 与 `rust_main`；
  - 多任务轮转执行主循环；
  - Trap 分发（时钟中断 / `UserEnvCall` / 其他异常）；
  - syscall trait 的平台实现（`IO`/`Process`/`Scheduling`/`Clock`/`Trace`）。
- `src/task.rs`
  - `TaskControlBlock`：上下文、完成状态、用户栈；
  - syscall 计数（配合 `trace`）；
  - `SchedulingEvent`：`None`/`Yield`/`Exit`/`UnsupportedSyscall`。
- `build.rs`
  - 自动解析 `tg-rcore-tutorial-user-T3L3/cases.toml`；
  - 构建用户程序并生成 `APP_ASM`；
  - 写入链接脚本。
- `.cargo/config.toml`
  - 固定 `riscv64gc-unknown-none-elf` 目标；
  - 配置 QEMU `virt + -nographic + -bios none`；
  - 配置 `TG_USER_*` 环境变量。

## 3. 调度与 Trap 主路径

1. 内核启动后初始化 console + syscall 分发器。
2. 根据 `AppMeta` 批量初始化 `TaskControlBlock`。
3. 主循环采用 Round-Robin：
   - 设置下一次 timer（非 `coop`）；
   - `tcb.execute()` 进入 U-mode；
   - 返回后按 `scause` 分支处理。
4. 处理结果影响任务状态：
   - 时间片到期 -> 切换任务；
   - `yield` -> 切换任务；
   - `exit` / 异常 -> 标记完成并减少 `remain`。

## 4. 输入子系统（T3L3 扩展）

T3L3 在 chapter 3 基础上补齐了 `STDIN` 的 `read` 路径：

- `IO::read` 支持 `fd == STDIN`；
- 通过 UART MMIO 非阻塞探测字符：
  - `LSR` 的 DR 位（bit0）为 1 表示有数据；
  - 从数据寄存器读出 1 字节；
- 若当前无输入，返回 `-2`（“暂不可读”语义）。

这样用户态可以构建两种输入方式：

- 轮询式：反复调用非阻塞 `read`；
- 中断式风格：在用户态以“等待输入事件再推进”的方式驱动逻辑。

## 5. 用户程序装载策略

`tg-rcore-tutorial-user-T3L3/cases.toml` 在 T3L3 中被收敛为仅保留 `snake`：

- `[ch3]` 仅 1 个 case：`snake`；
- `[ch3_exercise]` 同样仅 `snake`（避免 feature 切换时无 case 报错）。

因此内核启动后只装载并运行贪吃蛇应用。

## 6. 关键设计取舍

- 未引入 chapter 9 的完整外设中断链路（PLIC + VirtIO input）；
- 在 chapter 3 框架下采用 UART 非阻塞读取，控制复杂度；
- 通过 syscall 语义分层（`-2` + `sched_yield`）实现“可轮询、可等待”的双模式输入。

## 7. 可扩展方向

- 将 interrupt-like 模式升级为真正外设中断驱动（SupervisorExternal）；
- 增加输入缓冲队列与多字符读取；
- 为用户态应用提供更标准的非阻塞标志位和事件接口。
