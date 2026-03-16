# ch3-T3L3 任务实现记录

## 1. 任务要求回顾

- 在 `ch3-T3L3` 上实现用户态贪吃蛇；
- 支持轮询式输入与中断式输入两种控制方式；
- `cases.toml` 仅保留贪吃蛇；
- 调整 `ch3-T3L3` 与 `user-T3L3` 的 `Cargo.toml` 元数据；
- 依赖改为远程 crates，不使用本地 path。

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

### 2.4 用户态库扩展

- 修改 `user-T3L3/src/lib.rs`：
  - 新增 `getchar_poll() -> Option<u8>`；
  - 新增 `getchar_blocking() -> u8`（内部 `sched_yield` 等待）；
  - `getchar()` 改为复用 `getchar_blocking()`。

### 2.5 贪吃蛇程序

- 新增 `user-T3L3/src/bin/snake.rs`：
  - 纯字符终端渲染（ASCII 边框 + 蛇身 + 食物）；
  - 控制按键：`W/A/S/D`，退出键 `Q`；
  - 启动选择输入模式：
    - `p`：轮询模式（固定 tick + 非阻塞读输入）；
    - `i`：中断式风格（阻塞等待输入事件驱动推进）。

## 3. 测试与验证策略

1. 先验证配置是否可解析：`cargo check`。
2. 再验证内核是否能正确构建用户程序：`cargo build`。
3. 最后运行：`cargo run`，观察：
   - 启动菜单可选择 `p/i`；
   - 两种模式都可控制蛇移动；
   - `Q` 可退出。

> 由于贪吃蛇是交互式程序，自动 checker（按 chapter 3 原测试）与本任务目标不完全一致，需结合人工交互验证。

## 4. 风险与后续优化

- 当前 interrupt-like 模式是“事件驱动语义”，并非完整外设中断链路；
- 若后续推进到 chapter 9，可替换为真实 SupervisorExternal + 输入设备队列；
- 可以补充更完整的判定（例如允许头部进入旧尾格）与速度分级。
