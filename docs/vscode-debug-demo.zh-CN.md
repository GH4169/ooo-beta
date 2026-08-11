# 用 VS Code 单步调试 4 指令乱序执行 Demo

这篇文档使用仓库自带的 4 条固定指令，展示程序从取指、寄存器重命名、发射、取操作数、执行到按序退休的过程。Demo 使用单发射配置，输出稳定、运行时间短，适合第一次阅读项目时逐周期打断点。

## 1. 这个项目在模拟什么

该项目不是完整的 CPU 或 MIPS 模拟器，而是一个教学型的乱序执行数据通路 RTL 模型。目前主要支持 `ADD` 和 `BEQ`，重点模拟：

- 物理寄存器重命名；
- 指令队列中的乱序调度；
- 结果前递；
- Active List（类似 ROB）中的按序退休；
- 分支预测失败回退和精确异常恢复。

程序的顶层循环在 `main.cpp`。每调用一次 `datapath(false, ...)`，就模拟一个完整时钟周期。`datapath.cpp` 前半部分先计算组合逻辑（代码中称为 Tick），后半部分再统一更新时序状态（Tock），所以阅读时不要把普通 C++ 的语句先后顺序误认为硬件部件真的依次工作。

一次正常迭代的调用关系如下：

```text
main()
  ├─ Fetch::qGetInsts()       从固定 trace 取得下一条指令
  ├─ datapath(false, ...)     推进整个乱序数据通路一个周期
  │    ├─ Stage 2 Map         重命名并进入 Active List
  │    ├─ Stage 3 Dispatch    放入指令队列
  │    ├─ Stage 4 Issue       选择就绪指令
  │    ├─ Stage 5 Operand     读寄存器或接收前递
  │    ├─ Stage 6 Execute     ALU 执行、写回并标记完成
  │    └─ Stage 7 Retire      从 Active List 头部按序退休
  ├─ Fetch::aAccept()         移除本周期已被 datapath 接收的指令
  └─ simTimer += TICK_CYC     进入下一周期
```

## 2. Demo 内容与构建配置

固定指令来自 `test.h`：

```text
s0: ADD R4, R0, R8
s1: ADD R2, R0, R4
s2: ADD R4, R0, R8
s3: ADD R8, R4, R8
```

这里同时包含两类值得观察的依赖：

- RAW：`s1` 读取 `s0` 写入的 `R4`；`s3` 读取 `s2` 写入的最新版 `R4`；
- WAW：`s0` 和 `s2` 都写 `R4`，但重命名后分别写不同的物理寄存器，因此不会互相阻塞或覆盖未退休结果。

`make demo` 会生成独立的 `build/ooo-demo`，等价的关键编译配置为：

```text
TRACE_RANDOM=0          使用 test.h，而不是 100000 条随机轨迹
UARCH_USE_BASELINE=0    decode/issue/execute/retire 宽度降为 1
DEBUG_LEVEL=DEBUG_FULL  启用一致性断言和逐周期流水线日志
-O0 -g3                 保留适合源码单步的调试信息
```

这些选项只作用于 Demo 可执行文件，不会修改默认 `make` 所使用的随机、宽发射回归配置。

## 3. 在终端复现

需要安装 `g++`、GNU Make 和 GDB。在项目根目录执行：

```bash
make demo
./build/ooo-demo
```

流水线日志中的字母含义为：

| 标记 | 阶段 |
| --- | --- |
| `D` | Decode/Map 后进入数据通路 |
| `I` | Issue，指令被调度 |
| `O` | Operand Fetch，读取或等待操作数 |
| `E` | Execute，执行并写回 |
| `R` | Retire，按程序顺序退休 |

第一次运行时重点找下面几行：

```text
cyc1:D    :s0 ... td=t32 ts1=t0 ts2=t8
cyc2:D    :s1 ... td=t33 ts1=t0 ts2=t32
cyc3:D    :s2 ... td=t34 ts1=t0 ts2=t8
cyc4:D    :s3 ... td=t35 ts1=t34 ts2=t8
```

它们说明：逻辑寄存器 `R4` 的两次写入分别被重命名为 `t32` 和 `t34`；`s1` 的源操作数指向 `t32`，而更晚的 `s3` 指向 `t34`。这正是寄存器重命名消除 WAW 假依赖、同时保留 RAW 真依赖的核心。

最后会看到本次实测的退出摘要：

```text
Exiting: 67 cycles; 4 instructions completed.
```

程序在固定 trace 结束后还会空跑一段时间，让流水线完全排空，因此 4 条指令对应 67 个仿真周期是预期行为。

## 4. 在 VS Code 中启动调试

仓库已经提供：

- `.vscode/tasks.json`：执行 `make demo`；
- `.vscode/launch.json`：用 GDB 启动 `build/ooo-demo`，并在 `main` 停住；
- `.vscode/gdb-no-debuginfod.sh`：启动 GDB 前禁用系统库调试符号下载，避免网络或代理异常阻塞启动；
- `.vscode/c_cpp_properties.json`：让代码补全按 Demo 的三个编译宏解析条件编译分支。

操作步骤：

1. 用 VS Code 打开项目根目录；如果项目位于 WSL，请使用 VS Code 的 WSL 窗口打开它。
2. 安装或启用 Microsoft C/C++ 扩展，使 `cppdbg` 调试器可用。
3. 打开“运行和调试”视图，选择“调试：4 指令乱序流水线 Demo”。
4. 按 `F5`。VS Code 会先自动构建，然后在 `main()` 停住。
5. 用 `F10` 单步越过函数，用 `F11` 进入函数，用 `Shift+F11` 跳出函数，用 `F5` 继续到下一个断点。

如果启动任务提示找不到 `/usr/bin/gdb`，先在当前 Linux/WSL 环境安装 GDB；如果 GDB 位于别处，修改 `.vscode/gdb-no-debuginfod.sh` 中的路径。

WSL 中偶尔会显示 `GDB: Failed to set controlling terminal: Operation not permitted`。这是集成终端控制权警告，不影响断点和单步。启动脚本会移除 `DEBUGINFOD_URLS`，避免 GDB 通过不可用的网络或代理下载系统库调试符号时一直显示“正在运行”。

## 5. 推荐的第一轮断点

按下面顺序观察，不必一开始就在 `datapath.cpp` 中逐行走完近千行代码。

1. `main.cpp` 的 `fetchedInsts=fetch.qGetInsts()`：观察 `fetchedInsts.howmany` 和 `fetchedInsts.inst[0]`。
2. `main.cpp` 的 `datapath(false, ...)`：每次停下代表新一周期即将推进；把 `cycle`、`simTimer`、`accept` 加入监视窗口。
3. `fetch.cpp` 的 `Trace::getNextTraced()` 调用处：观察 4 条指令如何从 `test.h` 进入 fetch bundle。
4. `datapath.cpp` 中 `Stage 2 Map` 下的 `rmap.a2SetMapSS(...)`：观察逻辑目的寄存器如何获得新的 `RenameTag`。
5. `datapath.cpp` 中 `Stage 3 Dispatch` 下的 `instq[j].a3Insert(...)`：观察重命名后的操作进入哪个指令队列。
6. `datapath.cpp` 中 `Stage 4 Issue` 下的 `instq[i].a4Issue(...)`：观察就绪指令如何释放其消费者。
7. `datapath.cpp` 中 `Stage 6 Execute` 下的 `rf.a6Write(...)`：观察结果写入物理寄存器并在 Active List 中标记完成。
8. `datapath.cpp` 中 `Stage 7 Retire` 下的 `activelist.a7Retire(...)`：验证即使执行顺序可能变化，退休仍严格按程序顺序发生。

`datapath()` 在启动时还会以 `I_Reset=true` 调用一次。若在函数入口下断点，第一次停止是复位，之后 `I_Reset=false` 的调用才是实际周期。可以给入口断点添加条件：

```cpp
I_Reset == false
```

想只观察某个周期，也可以给 `main.cpp` 的 `datapath(false, ...)` 断点添加条件，例如：

```cpp
cycle == 3
```

## 6. 建议加入监视窗口的表达式

在 `main.cpp` 中：

```cpp
cycle
simTimer / TICK_CYC
fetchedInsts.howmany
fetchedInsts.inst[0].opcode
accept
```

进入 `datapath()` 后：

```cpp
I_Reset
I_2FetchedInsts.howmany
numToRename_2
numToDispatch_2L3
issueBndl_4[0].valid
oprndFetchBndl_4L5[0].valid
executeBndl_5L6[0].valid
retireBndl_7.howmany
```

其中一些变量只在对应代码块的作用域内可见；显示“无法求值”时，继续运行到变量声明之后即可。

## 7. 如何把日志和断点对应起来

单发射配置下，一条没有阻塞的指令大致沿着 `D → I → O → E → R` 前进，但依赖关系会影响 `I` 出现的时间。日志是在 Tock 状态更新阶段打印的，因此推荐采用下面的节奏：

1. 在 `main.cpp` 的 `datapath(false, ...)` 处按一次 `F5`，确定当前周期；
2. 在感兴趣的 stage action 处停下，查看 bundle 和重命名 tag；
3. 继续运行，观察调试控制台中同一周期的 `D/I/O/E/R` 日志；
4. 回到下一个 `datapath(false, ...)` 断点，比较流水线寄存器怎样跨周期保留状态。

理解这个 Demo 后，可以把 `UARCH_USE_BASELINE` 改为 `1` 再单独构建实验版本，观察同样 4 条指令如何在宽发射配置下乱序发射和并行退休。默认 `make` 已经采用宽发射配置，但同时会恢复随机 trace；做后续实验时建议继续通过独立编译参数控制，而不是直接反复修改头文件。

## 8. 清理

```bash
make clean
```

该命令会删除默认构建产物和 `build/ooo-demo`，不会删除源码、VS Code 配置或本文档。下次按 `F5` 时，预启动任务会自动重新构建 Demo。
