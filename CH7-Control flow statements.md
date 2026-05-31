这一章围绕“控制流 + 分支相关 backend 细节”展开：先用 pipeline 背景解释分支代价，再讲 C 控制流 → IR → DAG → Cpu0 两代 ISA（cpu032I/II）上的实现，然后扩展到长分支展开、删无用 jmp、延迟槽填充、条件指令和 phi 相关优化。

***

## Pipeline 与分支代价

前面用经典五级流水（IF/ID/EX/MEM/WB）和 super pipeline/cache bank 图重申：在 RISC pipeline 里分支属于关键性能路径，分支预测/延迟槽直接关系 CPI。  
Cpu0 假定像 MIPS 一样在 decode 阶段决策分支，并详细算了一条 `jne` 的 PC/offset 行为，说明 offset 编码必须和“更新 PC 的阶段”配合，否则会跳错地址。

***

## C 控制流 → IR → DAG → Cpu0

以简单的 if 例子说明：C 里的 if/else/while/for/goto/switch，经 Clang 变成 `(icmp + eq/ne/sgt/slt/sge/sle) + br`，再在 SelectionDAG 变成 `(seteq/setne/...)+brcond`。  
对 cpu032I，后端用 `CMP + JEQ/JNE/JGT/JGE/JLT/JLE`；对 cpu032II，改成 `(SLT/SLTu/SLTi/SLTiu)+BEQ/BNE`，即先用 SLT 得布尔，再用 beq/bne 分支，这就是表 34 中的 C→IR→DAG→两代 ISA 对应关系。

***

## brcond TableGen pattern（cpu032I）

这一节用 if 示例配合 `-view-isel-dags` 展示 DAG：`setcc` 生成 i32 布尔，再被 `brcond` 消费，目标是匹配为 `cmp + jne/jxx` 序列。  
`BrcondPatsCmp` multiclass 把 `(brcond (i32 (setne lhs, rhs)), bb)` 匹配为 `JNEOp (CMPOp lhs, rhs), bb`，以及 `(brcond cond, bb)` 匹配为 `JNEOp (CMPOp cond, ZERO), bb`，统一处理 seteq/ne/lt/le/ge 和 unsigned 版本。

***

## cmp+jne 的 SW 设计与寄存器类

`cmp` 写状态寄存器 SW，`jne` 只读 SW，不占用通用寄存器；SW 被放进两个寄存器类：CPURegs 与 SR，并在 GPROut 中排除 SW，使 allocator 不把 SW 分配给普通值。  
`getReservedRegs` 只真正保留 ZERO/AT/SP/LR/PC，不 reserve SW，这样 SW 既可被 `andi` 等逻辑指令访问，又能参与 `cmp` 等指令，而不会被 RA 分掉或破坏跳转语义。

***

## 分支 offset 计算与 PC 更新细节

作者用 `jne` 示例和 hexdump 精算：jne immediate 为 16，jne 与目标 basic block 间距为 20 字节（5 条指令），解释“IF 阶段 PC 先+4，然后 decode 时根据 imm 计算目标”为何需要 imm=16 才能落在 X+20。  
同时指出：若改成 EX 阶段执行分支，则应在硬件中设计 `PC = PC + 12 + offset` 之类的规则，这直接说明“硬件执行阶段”和 “汇编码 offset 语义” 要同步设计。

***

## cpu032I vs cpu032II 的分支编码

在同一个 if 程序上，cpu032I 输出 `cmp $sw, $4, $3; jne $sw, $BB0_2; jmp $BB0_1`，而 cpu032II 直接输出 `bne $4, $zero, $BB0_2; jmp $BB0_1`。  
cpu032II 结构更简洁、吞吐更高，但 offset 位宽从 24 降到 16，需要额外 long branch pass 解决长跳，表 34 用行“cpu032I vs cpu032II”概括这两种风格的实现差异。

***

## 长分支（long branch）展开 pass

由于 cpu032II 的 beq/bne 只有 16 位 offset，超过范围则生成非法代码，因此借用 MIPS LongBranch 思路写了 Cpu0LongBranch pass，作为 pre-emit pass 运行在 backend。  
pass 流程：先 `splitMBB` 保证每个 MBB 至多一个 branch，再 `initMBBInfo` 统计各块 size 和含 branch 的 MBB；`computeOffset` 算分支源目标间字节偏移，如果 imm 容不下（或强制选项 `-force-cpu0-long-branch` 打开），标记为 HasLongBranch，后续 `expandToLongBranch` 替换为长分支序列。

***

## expandToLongBranch：PIC 与非 PIC 序列

非 PIC 情况下，long branch 展开为简单 `jmp target; nop` 的新 basic block，把原 branch 的目标改成这个 longBrMBB；size 只增加两个指令。  
PIC 情况下用 BAL 构建更复杂序列：保存 LR 到栈、用 `LONG_BRANCH_LUi/LONG_BRANCH_ADDiu` 生成 `%hi($tgt-$baltgt)/%lo($tgt-$baltgt)`，通过 bal 跳到中间块，再用 `jr` 跳最终目标，这样能在不直接使用绝对地址的前提下实现长距离跳转。

***

## LONG_BRANCH_* 伪指令与 MCInstLower

`Cpu0InstrInfo.td` 定义两个伪指令 `LONG_BRANCH_LUi` 和 `LONG_BRANCH_ADDiu`，分别在 MC 层展开为 `lui` 和 `addiu` 带 `%hi($tgt-$baltgt)` / `%lo($tgt-$baltgt)`。  
`Cpu0MCInstLower::lowerLongBranch` 中，`createSub(BB1, BB2, Kind)` 构建 `MCSymbolRefExpr(BB1) - MCSymbolRefExpr(BB2)` 并包成 `Cpu0MCExpr`，分别作为 hi/lo 操作数，实现 hi/lo relocation 的差分表达式形式。

***

## getOppositeBranchOpc 与条件翻转

为了在 long branch 中复用 fallthrough，`replaceBranch` 会把原分支改成“反条件+落到 fallthrough”，因此需要 `getOppositeBranchOpc` 为 BEQ ↔ BNE 提供互逆映射。  
Cpu0SEInstrInfo 实现 `getOppositeBranchOpc`：BEQ→BNE，BNE→BEQ，不支持其他 opcode，否则触发 `llvm_unreachable`，这对 long branch 展开逻辑是核心依赖。

***

## 删除无用 jmp 的 backend pass

另一部分讲 Cpu0 backend 的小型优化 pass：Delete Useless JMP，即在 pre-emit 阶段去掉“顺序 fallthrough 时多余的 jmp”。  
结构上与 LongBranch 类似：注册 `createCpu0DelJmpPass` 到 `Cpu0PassConfig::addPreEmitPass`，遍历每个 MBB 的尾部，如果 `jmp` 目标就是下一个基本块，或者已由其他 branch 覆盖，则删除该 `jmp`，减少指令数量。

***

## 填充分支延迟槽（Delay Slot）

本章还提到“Fill Branch Delay Slot”，但实现留给读者参考 MIPS backend：在插入 branch 指令后，尝试从同 basic block 中挑一条安全可移动的指令放入 delay slot，否则填 nop。  
关键点：这个 pass 是典型 backend 特有优化，依赖目标 ISA 是否有 delay slot，以及 hazard/异常语义；Cpu0 目前延迟槽示例更多是教学用途，展示如何在 pass 管线里加入一个面向 MachineInstr 层的调度/重排 pass。

***

## 条件指令：select 与 select_cc

Clang 为某些 if/三目表达式生成 `select` / `select_cc` IR，以便后端通过条件 move 或 predicated execution 优化控制流。  
本章说明 `select` 会被 lower 成 `brcond+phi` 或 target-specific 的 `cmov` 样式 DAG 节点，在 Cpu0 上更多是走分支路径，同时指出你可以在自己的 backend 中为 select_cc 写专门 pattern，实现“无分支”条件执行。

***

## phi 节点与 phi 相关优化

章节中还介绍了 phi node：SSA 中的 φ 表达“多个前驱来的值合并”，在 SelectionDAG 中表现为 `phi` 节点，在 MachineIR/MIR 阶段被消解为一组从不同前驱块插入的 move。  
“Phi in optimization” 部分提到 LLVM 的标准 phi 优化（如合并冗余 phi、简化 φ + 常量折叠等），并给了若干例子展示 IR 层在 loop/if 上如何通过 phi 提供更好的 SSA 结构，供你对照 Cpu0 后端的寄存器分配和 coalescing 行为。

***

## RISC CPU 知识小结

结尾“RISC CPU knowledge” 小节回顾了：分支密度（平均每 7 条一分支）、条件分支硬件实现方式（decode/EX 阶段）、分支延迟槽、long branch 与 PC-relative 编码的一般技巧。  
结合 Cpu0 两版 ISA 的示例，作者希望你在写 backend 时把“控制流实现”当成 ISA 设计与 pipeline 设计结合点，而不仅仅是 TableGen 层面的 pattern 匹配问题。

你现在更想深入哪块：brcond → cmp/bxx/SLT 这一套 pattern 细节，还是 LongBranch pass 和 LONG_BRANCH_* 伪指令这套长分支展开机制？  
