# 第一个周期的 Rename：`datapath(false, ...)` 逐行阅读

本文只讲 `ooo-demo` 第一个正常周期里的寄存器重命名。这里的“第一个周期”是指：

```cpp
datapath(true, nothing, ...);       // 复位调用，不做 rename
fetchedInsts = fetch.qGetInsts();
datapath(false, fetchedInsts, ...); // 第一个正常周期
```

## 1. 先确定 Rename 的代码边界

在 [`datapath.cpp`](/home/zhourongyi/project/ooo-beta/datapath.cpp) 中，Rename 不是一个单独的函数，而是分成两部分：

| 代码位置 | 作用 | 是否修改持久状态 |
| --- | --- | --- |
| `326-416` | Stage 2 Map：决定接收多少条指令，并组合计算 `td/ts1/ts2/tdOld` | 否，只有查询和临时变量 |
| `386-389` | 调用 `rmap.q2GetMapSS()`，真正产生 rename tag | 否 |
| `936-981` | Tock 阶段：接受指令、提交新映射、设置 Busy | 是 |
| `1058-1063` | 把本周期的 rename 结果锁存到下周期 Stage 3 | 是，更新流水线寄存器 |

因此可以把 rename 概括为：

```text
Stage 2 Tick：计算 rename 结果
        |
        v
Stage 2 Tock：提交 RMap/ActiveList/Busy
        |
        v
Stage 3：下一个周期才 dispatch
```

这也是为什么第一个正常周期不会立刻把 `s0` 放入 InstQ：Stage 3 使用的是上一周期的 `*_2L3` 流水寄存器。

## 2. 第一个周期的输入和初始状态

`ooo-demo` 的编译选项是 `TRACE_RANDOM=0`、`UARCH_USE_BASELINE=0`，因此使用固定 trace 且所有宽度都是 1。第一条指令来自 [`test.h`](/home/zhourongyi/project/ooo-beta/test.h:31)：

```text
s0: ADD R4, R0, R8
```

复位后，物理寄存器映射表初始为恒等映射：

```text
R0 -> t0
R2 -> t2
R4 -> t4
R8 -> t8
...
```

物理文件重命名模式下，ActiveList 的第 0 个空闲槽提供 `t32`。所以本周期可预期的结果是：

```text
s0: ADD R4, R0, R8
        td  = t32
        ts1 = t0
        ts2 = t8
        tdOld = t4
```

## 3. `datapath.cpp:326-416`：Stage 2 Map 逐行解释

### 3.1 进入 Stage 2

```cpp
326 { 
328   // Stage 2 (Map) decides how many instructions can be renamed
329   // this cycle and lookup register renaming.
330 }
```

Stage 2 同时负责两件事：

1. 计算本周期最多接收多少条指令；
2. 查询当前 RMap，并生成重命名后的 `Operation`。

### 3.2 计算 `numToRename_2`

```cpp
336 numToRename_2 = UARCH_DECODE_WIDTH;
```

在 demo 中 `UARCH_DECODE_WIDTH=1`，所以初始上限是 1。

```cpp
338 fetchBndl_2 = I_2FetchedInsts;
```

把函数输入复制到 Stage 2 的临时变量。第一周期中：

```text
fetchBndl_2.howmany = 1
fetchBndl_2.inst[0] = s0
```

```cpp
339 numToRename_2 = MIN(numToRename_2, fetchBndl_2.howmany);
```

不能接收超过 FetchBundle 中已有的指令。这里仍然是 `MIN(1, 1)=1`。

```cpp
340 freeRegBndl_2 = activelist.q2GetFreeReg();
341 numToRename_2 = MIN(numToRename_2, freeRegBndl_2.howmany);
```

`q2GetFreeReg()` 查询 ActiveList 有多少空槽，并为每个槽提供一个新物理寄存器。物理文件重命名模式下，见 [`activelist.cpp:123-150`](/home/zhourongyi/project/ooo-beta/activelist.cpp:123)：

```text
freeRegBndl_2.howmany = 1
freeRegBndl_2.free[0] = t32
freeRegBndl_2.atag[0] = 0
```

所以 `numToRename_2` 仍为 1。`atag=0` 是 ActiveList 槽号，`free=t32` 是新目的物理寄存器。

```cpp
344 FOR_EXECUTE_WIDTH_i {
345   instqFree_2[i] = instq[i].q2NumSlots();
346   instqFreeTotal_2 += instqFree_2[i];
347 }
```

查询每个指令队列还有多少空槽。第一周期 InstQ 是空的，demo 中只有一个队列，所以：

```text
instqFree_2[0] = 16
instqFreeTotal_2 = 16
```

```cpp
348 instqFreeTotal_2 -= numToDispatch_2L3;
```

扣除上一个周期已经决定、将在本周期 Dispatch 的指令数。复位时：

```text
numToDispatch_2L3 = 0
```

因此第一周期仍有 16 个可用槽。

```cpp
349 instqFreeTotal_2 =
      (instqFreeTotal_2 >= 0) ? instqFreeTotal_2 : 0;
350 numToRename_2 = MIN(numToRename_2,
                       (ULONG)instqFreeTotal_2);
```

防止出现负数，并把 InstQ 容量也作为接收上限。这里是 `MIN(1,16)=1`。

### 3.3 扫描分支限制

```cpp
354 hasBR_2 = false;
355 for (i=0; i<numToRename_2; i++) {
361   if (fetchBndl_2.inst[i].opcode == BEQ) {
362     numToRename_2 = i + 1;
363     hasBR_2 = true;
364     break;
365   }
355 }
```

模型要求一个 decode bundle 中最多处理一条分支，而且分支必须是本 bundle 的最后一条。第一周期的 `s0` 是 `ADD`，所以：

```text
hasBR_2 = false
numToRename_2 = 1
```

下面的分支资源检查只在 `hasBR_2=true` 时执行；第一周期不会进入有效逻辑：

```cpp
369 if (hasBR_2) {
371   // 检查 ALU0 和 checkpoint 是否有资源
372   ...
374   hasBR_2 = false;
376   numToRename_2--;
377 }
```

### 3.4 为分支预留 checkpoint

```cpp
381 if (hasBR_2) {
383   newCheckPoint_2 = checkpoint.q2NextFree();
384 }
```

第一周期没有分支，因此不分配 checkpoint。后面 `renamedBndl_2.op[i].checkpoint` 虽然会被写入 `newCheckPoint_2`，但对 `ADD` 没有意义；这个字段只在 `BEQ` 执行时使用。

### 3.5 调用 `q2GetMapSS()` 生成 rename tag

```cpp
387 // **RENAME** up to numToRename_2 number of instructions
389 renamedBndl_2 =
      rmap.q2GetMapSS(numToRename_2,
                      fetchBndl_2.inst,
                      freeRegBndl_2.free);
```

这是 `datapath.cpp` 中明确标出的 Rename 调用。它只查询 RMap 和 free list，不立即修改 RMap。详细逻辑在 [`rmap.cpp:178-248`](/home/zhourongyi/project/ooo-beta/rmap.cpp:178)。

## 4. `rmap.cpp:178-248`：`q2GetMapSS()` 逐行解释

### 4.1 初始化 bundle

```cpp
179 RMapSS::q2GetMapSS(ULONG howmany, Instruction inst[], RenameTag free[])
181 ASSERT(howmany <= UARCH_DECODE_WIDTH);
183 RMapBundle renamed;
185 renamed.howmany = howmany;
```

第一周期传入 `howmany=1`，创建一个保存重命名结果的 `RMapBundle`。

```cpp
187 FOR_DECODE_WIDTH_i {
189   renamed.op[i].ts1 = ZeroRegTag;
190   renamed.op[i].ts2 = ZeroRegTag;
191   renamed.op[i].td  = ZeroRegTag;
192 }
```

先把所有槽初始化为 `t0`。这样当实际接收数量小于 decode 宽度时，未使用槽不会误把随机值当成目的寄存器；后面 Busy 设置循环也依赖这个约定。

### 4.2 生成目的寄存器 `td`

```cpp
194 for (i=0; i<howmany; i++) {
196   renamed.op[i].opcode = inst[i].opcode;
198   if (inst[i].rd != R0) {
199     renamed.op[i].td = free[i];
200   } else {
201     renamed.op[i].td = ZeroRegTag;
202   }
203 }
```

对 `s0`：

```text
inst[0].rd = R4 != R0
free[0]    = t32
=> renamed.op[0].td = t32
```

如果目的逻辑寄存器是 `R0`，就不分配真正的目的物理寄存器，使用 `ZeroRegTag`。

### 4.3 生成源寄存器 `ts1`

```cpp
206 FOR_DECODE_WIDTH_i {
208   renamed.op[i].ts1 = q2GetMap(inst[i].rs1);
```

`s0.rs1=R0`，所以 [`RMap::q2GetMap()`](/home/zhourongyi/project/ooo-beta/rmap.cpp:38) 对 `R0` 直接返回 `ZeroRegTag`：

```text
ts1 = t0
```

```cpp
209   for (j=i-1; j>=0; j--) {
210     if (inst[i].rs1 != R0 &&
211         inst[i].rs1 == inst[j].rd) {
212       renamed.op[i].ts1 = renamed.op[j].td;
213       break;
214     }
215   }
```

这段代码处理同一 bundle 内的前序指令依赖。例如同一周期前面有一条写 `R4` 的指令，后面的指令读 `R4`，后者应该直接使用前面指令新分配的 `td`。第一周期只有一条指令，`i=0` 时循环不会执行，因此 `ts1` 保持 `t0`。

### 4.4 生成源寄存器 `ts2`

```cpp
216   renamed.op[i].ts2 = q2GetMap(inst[i].rs2);
```

`s0.rs2=R8`，复位后的 RMap 中 `R8 -> t8`，所以：

```text
ts2 = t8
```

```cpp
217   for (j=i-1; j>=0; j--) {
218     if (inst[i].rs2 != R0 &&
219         inst[i].rs2 == inst[j].rd) {
220       renamed.op[i].ts2 = renamed.op[j].td;
221       break;
222     }
223   }
```

与 `ts1` 相同，这是同一 bundle 内的 RAW 依赖修正。第一周期没有前序槽，所以不执行。

### 4.5 保存旧目的映射 `tdOld`

在 `UARCH_ROB_RENAME=0` 时会编译下面的代码：

```cpp
224 if (inst[i].rd != R0) {
230   renamed.tdOld[i] = q2GetMap(inst[i].rd);
231   for (j=i-1; j>=0; j--) {
233     if (inst[i].rd != R0 &&
234         inst[i].rd == inst[j].rd) {
235       renamed.tdOld[i] = renamed.op[j].td;
236       break;
237     }
231   }
238 } else {
243   renamed.tdOld[i] = free[i];
244 }
```

对 `s0`，`rd=R4`，所以先查询旧映射：

```text
RMap[R4] = t4
=> tdOld = t4
```

`tdOld` 不是当前结果的目的寄存器，而是“重命名前 R4 对应的旧物理寄存器”。后续异常回退或退休释放旧映射时需要它。

如果同一个 bundle 中前面已经有一条也写 `R4` 的指令，后面的指令会把那条指令的 `td` 当作自己的 `tdOld`，从而正确处理同周期 WAW。

## 5. `datapath.cpp:391-415`：补充非 RMap 字段

`q2GetMapSS()` 只负责主要的寄存器 tag；返回后，`datapath.cpp` 补充其他字段：

```cpp
392 FOR_DECODE_WIDTH_i {
394   renamedBndl_2.op[i].opcode    = fetchBndl_2.inst[i].opcode;
395   renamedBndl_2.op[i].predTaken = fetchBndl_2.predTaken[i];
396   renamedBndl_2.op[i].oparity   = fetchBndl_2.oparity[i];
397 }
```

这些字段来自 Fetch，而不是 RMap。第一周期 `s0` 的操作码是 `ADD`。

```cpp
399 if (hasBR_2) {
400   ASSERT(numToRename_2);
401   ASSERT(fetchBndl_2.inst[numToRename_2-1].opcode == BEQ);
402 }
```

这是分支约束的断言。第一周期 `hasBR_2=false`，不会执行。

```cpp
404 FOR_DECODE_WIDTH_i {
406   renamedBndl_2.op[i].checkpoint = newCheckPoint_2;
407 }
```

给操作填 checkpoint。`s0` 不是分支，所以这个字段不参与后续 ADD 的执行。

```cpp
410 FOR_DECODE_WIDTH_i {
412   renamedBndl_2.op[i].dependOn = dependOnMask_0;
413 }
```

把当前未决分支掩码复制给操作。第一周期 checkpoint 表为空，因此：

```text
dependOn = 0000
```

到这里，Stage 2 Tick 产生的完整 `s0` 操作为：

```text
opcode=ADD, td=t32, ts1=t0, ts2=t8, dependOn=0000
```

## 6. `datapath.cpp:936-981`：Tock 阶段提交 Rename

上面的查询没有修改持久状态。真正提交发生在 Tock：

```cpp
937 if (!(exceptionPending_0 || maskIsSetSpeculation(rewindMask_6))) {
```

没有异常、没有分支错误回退时，才允许本周期 rename 生效。第一周期条件为真。

```cpp
940 OO_2Accept = numToRename_2;
```

输出本周期接收 1 条指令。函数返回后，`main.cpp` 用这个值调用 `fetch.aAccept(1)`。

```cpp
942 #if (DEBUG_LEVEL >= DEBUG_SILENT)
943 FOR_DECODE_WIDTH_i {
945   fetchBndl_2.cookie[i].op = renamedBndl_2.op[i];
946 }
```

Demo 开启调试时，把重命名后的 `Operation` 写入 cookie，便于后续日志打印 `td=t32 ts1=t0 ts2=t8`。它不负责修改 RMap。

```cpp
949 activelist.a2Accept(numToRename_2,
950                     fetchBndl_2.inst,
951                     fetchBndl_2.pcLike,
952                     renamedBndl_2.tdOld,
957                     fetchBndl_2.cookie);
```

将 `s0` 插入 ActiveList。此时保存：

```text
rd   = R4
tdOld= t4
pcLike=0
completed=false
exception=false
```

```cpp
960 rmap.a2SetMapSS(numToRename_2,
                   fetchBndl_2.inst,
                   freeRegBndl_2.free);
```

这才是 RMap 的写入动作。其实现见 [`rmap.cpp:251-260`](/home/zhourongyi/project/ooo-beta/rmap.cpp:251)：

```cpp
a2SetMap(inst[i].rd, free[i]);
```

所以第一周期完成：

```text
RMap[R4]: t4 -> t32
```

```cpp
962 FOR_DECODE_WIDTH_i {
965   ASSERT((i < numToRename_2) ? 1
             : tagEqual(renamedBndl_2.op[i].td, ZeroRegTag));
966   busy.a2SetBusy(tagToPRegIdx(renamedBndl_2.op[i].td));
967 }
```

将新目的物理寄存器标记为 Busy：

```text
Busy[t32] = true
```

循环会遍历整个 decode 宽度；超出 `numToRename_2` 的槽必须是 `ZeroRegTag`，因此不会错误占用物理寄存器。

```cpp
969 if (hasBR_2) {
974   checkpoint.a2New(newCheckPoint_2);
976   activelist.a2CheckPoint(newCheckPoint_2);
978   rmap.a2CheckPoint(newCheckPoint_2);
979 }
```

第一周期 `hasBR_2=false`，不执行。只有 `BEQ` 才会保存分支恢复点。

## 7. Rename 结果如何进入下一个流水级

在函数后面的流水线寄存器更新中：

```cpp
1058 if (!(exceptionPending_0 || maskIsSetSpeculation(rewindMask_6))) {
1059   numToDispatch_2L3 = numToRename_2;
1060   freeRegBndl_2L3   = freeRegBndl_2;
1061   fetchBndl_2L3     = fetchBndl_2;
1062   renamedBndl_2L3   = renamedBndl_2;
1063   hasBR_2L3         = hasBR_2;
1064 }
```

第一周期结束时，`s0` 的 rename 结果从临时变量 `renamedBndl_2` 变成持久的 `renamedBndl_2L3`。下一次调用时 Stage 3 才会使用它并执行 `instq[j].a3Insert(...)`。

## 8. 第一周期结束后的检查清单

调用 `datapath(false, fetchedInsts, ...)` 返回后，可以在调试器中检查：

```text
accept                 == 1
rewind                 == false
restart                == false
renamedBndl_2.op[0].td  == t32
renamedBndl_2.op[0].ts1 == t0
renamedBndl_2.op[0].ts2 == t8
RMap[R4]                == t32
Busy[t32]               == true
numToDispatch_2L3       == 1
```

一句话总结：`q2GetMapSS()` 负责“算出新 tag”，`a2SetMapSS()` 负责“把新 tag 写入 RMap”，ActiveList 保存旧映射和顺序信息，Busy 表记录结果尚未产生；这些动作共同构成第一个周期的 Rename。
