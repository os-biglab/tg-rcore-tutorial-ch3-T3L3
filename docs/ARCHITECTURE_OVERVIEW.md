# tg-rcore-tutorial-ch3-T3L3 架构总览

## 1. 目标与范围

`tg-rcore-tutorial-ch3-T3L3` 基于 chapter 3 的多道程序内核，完整实现了一个用户态贪吃蛇系统。整体目标分为两部分：

- **内核侧**：保留 chapter 3 的多任务调度主线，补齐输入与图形相关 syscall，负责装载用户程序、处理中断、提供 framebuffer 和输入能力；
- **用户侧**：实现贪吃蛇玩法、输入模式选择、窗口渲染与按键控制，并通过 syscall 与内核交互。

本 crate 仍为 `no_std`、`no_main` 的 RISC-V S-mode 教学内核。

## 2. 总体模块结构

### 2.1 内核代码

- `src/main.rs`
  - 内核入口 `_start` 与 `rust_main`；
  - 多任务轮转执行主循环；
  - Trap 分发（时钟中断 / `UserEnvCall` / 外部中断 / 其他异常）；
  - syscall trait 的平台实现（`IO`/`Process`/`Scheduling`/`Clock`/`Trace`）；
  - 图形与输入子系统的统一接线。
- `src/task.rs`
  - `TaskControlBlock`：上下文、完成状态、用户栈；
  - syscall 计数（配合 `trace`）；
  - `SchedulingEvent`：`None`/`Yield`/`Exit`/`UnsupportedSyscall`。
- `src/uart.rs`
  - UART 初始化、非阻塞读取、接收中断开关；
  - 作为 `STDIN` 的输入来源。
- `src/plic.rs`
  - PLIC priority / enable / threshold / claim / complete 封装；
  - 为 UART 中断链路提供支持。
- `src/gpu.rs`
  - VirtIO GPU 初始化；
  - framebuffer 地址、长度、宽高导出；
  - framebuffer flush 接口。
- `build.rs`
  - 自动解析 `tg-rcore-tutorial-user-T3L3/cases.toml`；
  - 构建用户程序并生成 `APP_ASM`；
  - 写入链接脚本。

### 2.2 用户代码

- `tg-rcore-tutorial-user-T3L3/src/lib.rs`
  - 封装 `framebuffer_info()` / `framebuffer_flush()`；
  - 提供 `getchar_poll()` / `getchar_blocking()`；
  - 通过 syscall 连接内核输入与图形能力。
- `tg-rcore-tutorial-user-T3L3/src/bin/snake.rs`
  - 实现贪吃蛇核心逻辑、输入模式切换、增量渲染；
  - 支持轮询模式与 interrupt-like 模式；
  - 直接写 framebuffer，并在必要时调用 flush。

## 3. 内核侧实现流程

### 3.1 启动与装载

1. `rust_main` 启动后清零 BSS，初始化 console 和 syscall 分发器。
2. 根据 `tg_linker::AppMeta::locate()` 读取打包进镜像的用户程序。
3. 为每个应用初始化 `TaskControlBlock`，并记录入口地址。
4. 进入 Round-Robin 主循环，按任务轮流执行。

### 3.2 Trap 与调度

主循环在用户程序返回后依据 `scause` 做分支：

- `SupervisorTimer`：时间片到期，切换任务；
- `SupervisorExternal`：处理外部中断，主要是输入链路；
- `UserEnvCall`：处理 syscall；
- 其他异常：终止任务。

### 3.3 输入子系统

T3L3 在 chapter 3 基础上补齐了 `STDIN` 的 `read` 路径：

- `IO::read` 支持 `fd == STDIN`；
- 轮询模式下优先返回已缓存输入，否则非阻塞探测 UART；
- 中断模式下通过 UART RX 中断 + PLIC claim/complete 将字符放入输入缓存；
- 若当前无输入，返回 `-2` 表示暂不可读。

### 3.4 图形子系统

图形侧采用 VirtIO GPU + framebuffer 的方式：

- `gpu.rs` 初始化 GPU 设备并申请 framebuffer；
- 内核通过自定义 syscall 向用户态暴露 framebuffer 信息；
- 用户程序可直接写 framebuffer，再调用 flush 同步到显示设备。

## 4. 用户态贪吃蛇实现流程

### 4.1 游戏状态

`snake.rs` 中维护：

- 蛇身数组与长度；
- 当前移动方向；
- 食物位置与随机种子；
- 游戏结束 / 退出状态。

### 4.2 输入模式

启动后先让用户选择输入模式：

- `p`：轮询模式，主循环持续调用 `getchar_poll()`；
- `i`：interrupt-like 模式，通过 `getchar_blocking()` 和内核输入机制驱动。

用户按键 `W/A/S/D` 控制方向，`Q` 退出。

### 4.3 渲染策略

渲染采用 framebuffer 直写，而不是字符终端输出：

- 第一次进入时绘制完整背景和棋盘；
- 后续每帧只比较“上一帧状态”和“当前帧状态”；
- 只重绘变化的格子、模式条、分数条和结束提示条；
- 最后统一调用 `framebuffer_flush()`。

这样可以显著降低每帧写入 framebuffer 的数据量。

## 5. 用户程序装载策略

`tg-rcore-tutorial-user-T3L3/cases.toml` 在 T3L3 中被收敛为仅保留 `snake`：

- `[ch3]` 仅 1 个 case：`snake`；
- `[ch3_exercise]` 同样仅 `snake`。

这样内核启动后只装载并运行贪吃蛇应用，避免无关 case 干扰调试。

## 6. 关键设计取舍

- 未引入 chapter 9 的完整外设中断链路，而是优先在 chapter 3 框架下完成可运行闭环；
- 输入侧采用 UART 非阻塞读取 + PLIC 接线，降低实现复杂度；
- 图形侧采用 framebuffer syscall 直写，保证用户态渲染的灵活性；
- 用户态渲染采用增量更新，减少 framebuffer 写入量。

## 7. 可扩展方向

- 将 interrupt-like 模式升级为真正的外设中断驱动；
- 增加输入缓冲队列与多字符读取；
- 把渲染进一步拆成“脏矩形合并”或“区域刷新”；
- 为用户态应用提供更标准的非阻塞标志位和事件接口。
