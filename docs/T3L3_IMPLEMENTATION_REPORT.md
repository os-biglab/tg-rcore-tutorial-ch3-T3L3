# ch3-T3L3 任务实现记录

## 1. 任务要求回顾

- 在 `ch3-T3L3` 上实现用户态贪吃蛇；
- 支持轮询式输入与中断式输入两种控制方式；
- `cases.toml` 仅保留贪吃蛇；
- 调整 `ch3-T3L3` 与 `user-T3L3` 的 `Cargo.toml` 元数据；
- 依赖改为远程 crates，不使用本地 path；
- 最终将实现过程整理为“内核侧 + 用户侧”两条主线。

## 2. 修改清单

### 2.1 配置与元数据

- 更新 `tg-rcore-tutorial-ch3-T3L3/Cargo.toml`
  - 仓库信息改为 `ch3-T3L3` 对应地址；
  - 依赖版本统一为 `0.4.8` 系列（含 `build-dependencies`）。
- 更新 `tg-rcore-tutorial-user-T3L3/Cargo.toml`
  - 元数据改为 `user-T3L3` 对应信息；
  - `tg-console`、`tg-syscall` 由 path 依赖改为 crates.io 依赖；
  - 追加 `[workspace]` 避免父 workspace 冲突。

### 2.2 测例裁剪

- 重写 `tg-rcore-tutorial-user-T3L3/cases.toml`
  - `[ch3]` 仅保留 `snake`；
  - `[ch3_exercise]` 仅保留 `snake`。

### 2.3 内核能力扩展（输入）

- 修改 `ch3-T3L3/src/main.rs` 的 `impl IO for SyscallContext`：
  - 新增 `read` syscall 支持；
  - 支持 `STDIN`，一次读取 1 字节；
  - 无输入时返回 `-2`。
- 新增 UART 轮询函数（RISC-V）：
  - 读取 `0x1000_0000` UART；
  - 检查 LSR bit0 决定是否有可读字节。

### 2.4 内核能力扩展（图形）

- 新增 framebuffer 相关 syscall 后端：
  - `framebuffer_info()`：返回 framebuffer 基址、长度、宽、高；
  - `framebuffer_flush()`：刷新 framebuffer；
  - `set_input_mode()`：切换轮询 / 中断式输入语义。
- 将 GPU、UART、PLIC 逻辑拆分为独立模块：
  - `src/gpu.rs`：VirtIO GPU 初始化与 framebuffer 维护；
  - `src/uart.rs`：UART 设备访问；
  - `src/plic.rs`：PLIC 中断控制。

### 2.5 用户态库扩展

- 修改 `user-T3L3/src/lib.rs`：
  - 新增 `getchar_poll() -> Option<u8>`；
  - 新增 `getchar_blocking() -> u8`（内部 `sched_yield` 等待）；
  - `getchar()` 改为复用 `getchar_blocking()`。
  - 新增 `framebuffer_info()` / `framebuffer_flush()`，供图形渲染使用。

### 2.6 贪吃蛇程序

- 新增 `user-T3L3/src/bin/snake.rs`：
  - 先采用终端输出版本，随后改为 framebuffer 渲染版本；
  - 控制按键：`W/A/S/D`，退出键 `Q`；
  - 启动选择输入模式：
    - `p`：轮询模式（固定 tick + 非阻塞读输入）；
    - `i`：中断式风格（阻塞等待输入事件驱动推进）；
  - 渲染优化：背景首帧全量绘制，后续增量更新变化区域。

## 3. 测试与验证策略

1. 先验证配置是否可解析：`cargo check`。
2. 再验证内核是否能正确构建用户程序：`cargo build`。
3. 最后运行：`cargo run`，观察：
   - 启动菜单可选择 `p/i`；
   - 两种模式都可控制蛇移动；
   - `Q` 可退出。

> 由于贪吃蛇是交互式程序，自动 checker（按 chapter 3 原测试）与本任务目标不完全一致，需结合人工交互验证。

  ## 4. 遇到的问题与修复过程

  ### 4.1 输入按键无响应

  最初出现的问题是：用户在终端中按 `p` / `i` 没有反应。

  - 原因 1：串口输入路径只在中断链路里可达，轮询模式下没有稳定接入 UART 非阻塞读取；
  - 原因 2：运行器输入通道与 monitor 复用时，字符有时被上层拦截。

  修复方式：

  - 在内核 `IO::read` 中直接支持 `STDIN`；
  - 增加 UART 非阻塞探测；
  - 让运行器输入直连串口，避免输入被 monitor 吞掉。

  ### 4.2 GUI 不显示或只显示空白

  图形版本切换后，曾出现 framebuffer 没有正确刷新的情况。

  - 原因 1：自定义 syscall 号与 framebuffer flush 的编号发生冲突，导致用户态调用被错误分发；
  - 原因 2：旧构建产物缓存导致嵌入的用户程序并非最新版本，调试时容易误判。

  修复方式：

  - 重新分配 syscall 编号，避免 `flush` 与 `set_input_mode` 冲突；
  - 强制 clean rebuild，确认新用户程序被重新嵌入。

  ### 4.3 framebuffer 写入过多

  蛇程序每帧完整重画整个 framebuffer 时，渲染时间偏长。

  修复方式：

  - 改为增量渲染：只重画变化的格子、模式条、分数条和结束提示；
  - 将背景和棋盘基础图层只在首帧初始化。

  ### 4.4 内核静态区与用户程序地址冲突

  用户程序的 `base` 地址如果设置过低，会覆盖内核 `.bss` / `.boot` / heap 等静态区域，表现为 framebuffer 或输入功能异常。

  修复方式：

  - 将 `cases.toml` 中的 `base` 调整到更高地址；
  - 避免用户程序覆盖内核已占用的内存范围。

  ## 5. 后续优化方向

- 当前 interrupt-like 模式是“事件驱动语义”，并非完整外设中断链路；
- 若后续推进到 chapter 9，可替换为真实 SupervisorExternal + 输入设备队列；
- 可以补充更完整的判定（例如允许头部进入旧尾格）与速度分级。
