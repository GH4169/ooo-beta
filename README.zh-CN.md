# OOO 超标量处理器学习指南

这是一个用 C++ 编写的教学型 RTL（寄存器传输级）模型，模拟类似 MIPS R10000 的推测执行、乱序执行和按序退休流水线。当前模型只实现寄存器数据流上的 `ADD` 和 `BEQ`，不包含完整的取指前端，也还没有 `LW/SW`。

## 1. 环境要求

在 Ubuntu/Debian 虚拟机中安装构建工具：

```bash
sudo apt update
sudo apt install -y build-essential
```

本次验证使用的是 Ubuntu 26.04、GCC 15.2 和 GNU Make 4.4。项目不依赖第三方 C++ 库。

## 2. 编译并运行默认 demo

在项目根目录执行：

```bash
make clean
make -j2
./ooo > output
tail -n 12 output
```

`make` 会把所有 `.cpp` 编译成目标文件，并链接出可执行文件 `ooo`。直接运行 `./ooo` 时，程序会生成并执行一条 100,000 条指令的随机轨迹；输出包含每个周期的流水线事件，因此文件比较大。

本机已实际运行过这个 demo，末行摘要应类似：

```text
Exiting: 43157 cycles; 84056 instructions completed.
```

随机轨迹使用 `rand()`，所以不要把每一行流水线日志当作跨平台的固定输出；仓库提供的参考文件用于当前默认构建配置的回归比较。

## 3. 回归测试（推荐先跑这个）

```bash
make regress1
```

该目标会运行 `ooo`，把输出写入 `output`，再执行：

```bash
diff -w output reference1
```

没有任何 `diff` 输出并且退出码为 `0` 就表示通过。本机已验证 `regress1` 通过。

## 4. 一个更容易观察的 4 条指令 demo

默认是宽发射随机轨迹。要观察寄存器重命名和 RAW/WAW 依赖，可以切换到仓库自带的固定测试：

1. 编辑 [`trace.h`](trace.h)，将

   ```cpp
   #define TRACE_RANDOM (1)
   ```

   改为 `#define TRACE_RANDOM (0)`。

2. 编辑 [`uarch.h`](uarch.h)，将

   ```cpp
   #define UARCH_USE_BASELINE (1)
   ```

   改为 `#define UARCH_USE_BASELINE (0)`。这个配置把 decode/issue/execute/retire 宽度都降到 1，日志更容易逐周期阅读。

3. 重新编译并运行：

   ```bash
   make clean
   make
   ./ooo
   ```

[`test.h`](test.h) 中的默认指令序列是：

```text
s0: ADD R4, R0, R8
s1: ADD R2, R0, R4
s2: ADD R4, R0, R8
s3: ADD R8, R4, R8
```

重点观察日志中的 `D`（Decode）、`I`（Issue）、`O`（Operand fetch）、`E`（Execute）和 `R`（Retire），以及 `td/ts1/ts2` 的物理寄存器标签变化。完成学习后，把 `TRACE_RANDOM` 和 `UARCH_USE_BASELINE` 改回 `1`，恢复默认配置。

## 5. 另一个重命名配置

将 [`uarch.h`](uarch.h) 中的：

```cpp
#define UARCH_ROB_RENAME (0)
```

改成 `1`，可以改用 ROB（重排序缓冲）进行寄存器重命名。默认宽发射配置下可用：

```bash
make clean
make
make regress2
```

`regress2` 会将输出与 `reference2` 比较。切换这个宏后必须重新编译，因为它是编译期配置。

## 6. 建议的源码阅读顺序

| 文件 | 作用 |
| --- | --- |
| [`main.cpp`](main.cpp) | 仿真主循环：每个周期调用 fetch 和 datapath，并处理 rewind/restart |
| [`arch.h`](arch.h) | 架构状态、寄存器名和操作码定义 |
| [`uarch.h`](uarch.h) | 发射宽度、ROB/物理寄存器重命名等微架构参数 |
| [`trace.h`](trace.h)、[`trace.cpp`](trace.cpp) | 随机或固定指令轨迹生成 |
| [`fetch.h`](fetch.h)、[`fetch.cpp`](fetch.cpp) | 向 datapath 提供指令，并处理分支回退/异常重启 |
| [`datapath.h`](datapath.h)、[`datapath.cpp`](datapath.cpp) | 核心流水线：rename、dispatch、issue、operand fetch、execute、retire |
| [`rmap.*`](rmap.h)、[`regfile.*`](regfile.h) | 寄存器映射表和寄存器文件 |
| [`instq.*`](instq.h)、[`activelist.*`](activelist.h) | 保留站/指令队列和 Active List（ROB） |
| [`magic.*`](magic.h)、[`exception.*`](exception.h) | 参考执行结果、断言检查、精确异常 |

## 7. 调试输出

在 [`sim.h`](sim.h) 中选择 `DEBUG_LEVEL`：

- `DEBUG_FULL`：断言和流水线日志（默认）
- `DEBUG_VERBOSE`：额外打印指令队列和 Active List
- `DEBUG_SILENT`：保留断言，不打印普通流水线日志
- `DEBUG_NONE`：关闭断言和日志

修改头文件中的宏后执行 `make clean && make`，避免旧目标文件继续使用旧配置。

## 8. 清理生成文件

```bash
make clean
```

这会删除目标文件、`ooo` 可执行文件和回归生成的 `output`，不会删除源码或 `reference1/reference2`。

## 9. 模型边界

该项目的目标是帮助理解乱序超标量 datapath 的时序和一致性检查，不是可启动的完整 CPU，也不能运行 MIPS/Linux 程序。建议先阅读项目原有的英文 [`README.md`](README.md) 和其中引用的 R10000 论文，再结合上述 4 条指令 demo 单步观察。
