这一章主线是：把 C 里的算术 / 逻辑运算（+ - * << >> % /）系统性地映射到 LLVM IR，再映射到 Cpu0 指令，同时引入 HILO / C0 这类特殊寄存器类，并教你用 Graphviz 看 SelectionDAG 的各阶段变换。

***

## 章节整体目标

本章先在 `Chapter4_1/` 中补齐加减乘移位等算术运算，然后讲 DAG 可视化，之后重点讲 `%` / `/` 的 DAG 形态和优化（除法转乘法），最后引入 HILO 和 C0 寄存器类以及相关 move 指令。  
编译示例 C/IR 程序，并用 `llc` 的 `-view-*-dags` 选项配合 Graphviz，帮助你观察从 IR → SelectionDAG → 指令选择前后的 DAG 变化，以便在 `.td` 里写 pattern。

***

## 溢出控制：EnableOverflow 与 ADD/SUB

在 `Cpu0Subtarget.cpp` 里定义了 `-cpu0-enable-overflow` cl::opt，Subtarget 构造时把它写进 `EnableOverflow` 字段，用来控制是否使用触发溢出的 add/sub 指令。  
在 `Cpu0InstrInfo.td` 中，借助 `Predicates` 拆出两套 pattern：`DisableOverflow` 走 `SUBu`（截断溢出），`EnableOverflow` 打开时使用 `ADD/SUB`（溢出引发异常），用于 debug 场景排查 bug。

***

## 算术指令：+ - * << >>

`Chapter4_1/Cpu0InstrInfo.td` 中定义了 `ADDu/ADD/SUBu/SUB/MUL`，并用 `ArithLogicR` 模板绑定到 DAG 节点 `add/sub/mul`，实现 C 的 `+ - *`。  
移位部分，用 `shift_rotate_imm32` 和 `shift_rotate_reg` 模板生成 `ROL/ROR/SHR/SRA/SRAV/SHLV/SHRV/ROLV/RORV`，分别对应立即数版和寄存器版移位 / 旋转，匹配 LLVM IR 节点 `shl/sra/srl/rotl/rotr`。

***

## 逻辑移位 vs 算术移位：lshr / ashr

LLVM 对右移有两种 IR 指令：`lshr`（逻辑右移，填 0）和 `ashr`（算术右移，符号扩展）。  
在 SelectionDAG 上，它们的 IR node 名分别叫 `srl`（lshr）和 `sra`（ashr），映射到 MIPS 的 `srl/sra`，Cpu0 则用 `shr/sra`，表 22 给了有符号 / 无符号例子，说明必须同时支持 `lshr` 和 `ashr` 才能覆盖 C 中带符号和无符号的所有情况。

***

## C >> 的实现细节

C 语言标准允许对负数右移的行为由实现定义，多数编译器采用算术右移（符号扩展），等价于 MIPS `sra`。  
表 22 汇总了：bitcode 中 `>>` 对应 `lshr` 或 `ashr`，IR 节点 `srl/sra`，MIPS/Cpu0 的具体指令，以及对有符号 / 无符号样例的结果，提醒你不能把 “x >> 1 等价于 x / 2” 简单套在所有类型上。

***

## 左移 << 与溢出的语义

表 23 对 `<<` 做了类似总结：bitcode 和 IR 节点都是 `shl`，MIPS 指令 `sll`，Cpu0 指令 `shl`。  
作者用 `x << 1` 视作 `x * 2` 的视角说明：只要结果没溢出（仍在有符号 / 无符号范围内），`shl` 都满足这个等价关系，一旦溢出则行为未定义，寄存器中任何值都是“可接受”的。

***

## HILO 寄存器类与乘除单元

在 `Cpu0RegisterInfo.td` 中新增 `HI` 和 `LO` 寄存器，并定义 `HILO` RegisterClass，把二者捆成一个寄存器类，供 `MULT/MULTu/SDIV/UDIV` 这类指令使用。  
`Cpu0InstrInfo.td` 定义了 `Mult/Div/MoveFromLOHI/MoveToLOHI` 等模板，以及 `MULT/MULTu/SDIV/UDIV/MFHI/MFLO/MTHI/MTLO` 指令，模拟 MIPS 样式的“乘除单元输出放 HI/LO” 的设计。

***

## C0 寄存器类与协处理器操作

同一文件还定义了 `MoveFromC0/MoveToC0/C0Move` 模板，以及 `MFC0/MTC0/C0MOVE` 指令，对应协处理器 0 的访问。  
这是为了演示 “非通用寄存器类” 的处理：除通用 CPURegs 外，还要支持 HILO / C0 等寄存器类，并在 copy / move 指令中特殊对待这些寄存器。

***

## TargetLowering：Div/Rem 的 OperationAction

在 `Cpu0ISelLowering.cpp` 的构造函数中，作者将 `ISD::SDIV/SREM/UDIV/UREM` 的 OperationAction 设为 `Expand`，表示 SelectionDAG 不直接选目标指令，而是用展开变换替代。  
同时用 `setTargetDAGCombine(ISD::SDIVREM/UDIVREM)` 注册目标侧 DAGCombine 钩子，以便在 DAGCombine 阶段把 `(div, rem)` 合并为单个 `Cpu0ISD::DivRem/DivRemU` 节点。

***

## Target DAGCombine：DivRem → LO/HI

`performDivRemCombine` 将 `ISD::SDIVREM/UDIVREM` 合并成一个 Glue 节点 `Cpu0ISD::DivRem` 或 `DivRemU`，并以 Glue 连接后续的 `CopyFromReg LO/HI`。  
如果原节点的商（value 0）有用，则插入 `CopyFromLo`；余数（value 1）有用则插入 `CopyFromHi`，并用 `ReplaceAllUsesOfValueWith` 把原节点的 uses 重写为新的 Copy 节点，从而实现“一个硬件 div 指令产生商+余数，用 MFLO/MFHI 读出”。

***

## SelectionDAG 指令选择：MULHS/MULHU

`Cpu0SEISelDAGToDAG.cpp` 中 `selectMULT` 用 `CurDAG->getMachineNode` 创建 `MULT/MULTu` 并产生 Glue，再根据需要插入 `MFLO/MFHI` 进行结果读取。  
`trySelect` 中对 `ISD::MULHS/MULHU` 特判，通过 `selectMULT` 直接生成读取 HI 的版本，并用 `ReplaceNode` 替换原节点，体现了 “高位乘法” 使用 HILO 设计的传统 MIPS 风格实现。

***

## copyPhysReg 里对 HI/LO 的特殊处理

`Cpu0SEInstrInfo::copyPhysReg` 里，对 CPURegs→CPURegs copy 使用 `ADDu Dest, Src, ZERO`，对 HI/LO→CPURegs 使用 `MFHI/MFLO`，对 CPURegs→HI/LO 使用 `MTHI/MTLO`。  
这保证了通用寄存器和特殊寄存器之间的复制均通过合法指令完成，避免在 MachineInstr 层出现“不存在的 move hi, lo, reg” 的假指令。

***

## Graphviz 显示 SelectionDAG 各阶段

本章给出了 `llc` 的一系列 `-view-*-dags` 选项：`-view-dag-combine1-dags`（初始 DAG）、`-view-legalize-dags`、`-view-dag-combine2-dags`、`-view-isel-dags`、`-view-sched-dags`。  
说明它们对应的阶段：初始 DAG → type legalization → legalization → isel 前 DAG → 调度前 DAG，并强调 `-view-isel-dags` 对 backend 开发最重要，因为你写 `.td` pattern 时需要精确看到 isel 前的 DAG 结构。

***

## srem 的 DAG 形态：除法被乘法替换

示例 `ch4_1_mult.cpp` 中 `(b+1)%12` 被编译为 `srem`，再在 DAGCombine 中变成乘法加减组合（利用常数 0x2AAAAAAB），以避免昂贵的 div 指令。  
图 27 的 DAG 中 `mulhs` 节点是关键：`mulhs(t8, 0x2AAAAAAB)` 取 64 位乘法结果的高 32 位，然后通过 `sra/srl/add/mul/sub` 序列恢复余数，整套算式推导证明 `(b+1)%12` 的结果仍正确。

***

## ARM 风格 SMMUL 实现方案

后面给出一个“ARM 风格”实现：在 `.td` 中定义 `SMMUL/UMMUL`，把 `mulhs/mulhu` 直接映射到单条指令，而不是通过 HI/LO 结构。  
同时在 `Cpu0ISelDAGToDAG.cpp` 中关闭原有 `MULHS/MULHU` 特判，让 TableGen 选择 `SMMUL/UMMUL`，并展示 `-view-sched-dags` 下的 DAG，说明如何用另一种硬件接口重写 IR 映射。

***

## 逻辑（AND/OR/XOR）与其他算术的关系

逻辑指令部分（AND/OR/XOR 等）结构与算术类似：C 运算符 → LLVM IR（`and/or/xor`）→ `.td` pattern → Cpu0 指令。  
作者强调：从这一章起，重点从“类层次结构”切到“C 运算符–IR–指令”三段映射，以及在 `.td` 里写 pattern + 用 DAG 可视化定位 pattern 匹配错误，这是写 backend 时最常用的工作流。

***

你接下来更想深入哪一块：是 `%`/`/` 那套 `srem` → mulhs 变换和 DAGCombine，还是 HILO/C0 寄存器类与 copy/ISel 的细节？  
