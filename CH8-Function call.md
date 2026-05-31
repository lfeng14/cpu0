这一章主线：在前面栈帧/全局/PIC/ctrlflow 的基础上，把“函数调用整条链路”接起来：从栈帧布局和参数传递 ABI（MIPS 风格），到 `LowerFormalArguments`/`LowerCall` 的 DAG 结构，再到 ADJCALLSTACK* 伪指令、byval/struct/tail call/vararg/dynamic alloc 各种 corner case，以及 call 相关 MC/reloc 支持。

***

## MIPS 栈帧模型与 Cpu0 ABI 选择

作者先用 MIPS ABI 当模板：前 4 个参数放 $a0–$a3，剩余的落栈，sum_i 的例子展示 main 把第 5、6 个实参放在 16($sp)、20($sp)，callee 从 48($sp)、52($sp) 取回（调用者/被调者栈帧叠加）。  
Cpu0 设计了两个调用约定：  
- S32：所有参数走栈（`-cpu0-s32-calls=true`）；  
- O32：前两个 i32 参数走寄存器 A0/A1，其余走栈（`-cpu0-s32-calls=false`），对齐 long long/byval 也遵循类似 MIPS 规则。

***

## CC 函数：CC_Cpu0S32 / CC_Cpu0O32

调用约定由 CC 函数家族驱动：  
- `CC_Cpu0S32`：简单，把每个参数分配到栈，8/16 位提升到 32 位，使用 `CCState::AllocateStack` 返回偏移，再 `addLoc(getMem(...))`。  
- `CC_Cpu0O32`：使用 IntRegs = {A0, A1}，前两个 i32（或某些 float）分配到寄存器，溢出部分落栈，同时处理 i64/f64 时寄存器对齐问题（如首个 64 位必须落在 A0）。

`Cpu0CC::fixedArgFn()` 根据 Subtarget ABI 返回这两个函数之一；`Cpu0CC::allocateRegs` 则为 byval 参数分配一串连续寄存器并记录 ByValArgInfo。

***

## LowerFormalArguments：实参来源（Reg vs Stack）

`LowerFormalArguments` 做的是“把 formal 参数从物理位置搬到 DAG 里的 SSA 值”：  
- 用 `CCState CCInfo(..., ArgLocs, ...)` + `Cpu0CC::analyzeFormalArguments` 先填满 `ArgLocs`，每个元素 `CCValAssign` 描述一个实参在 reg/stack 上的位置。  
- 对 `VA.isRegLoc()`：  
  - 调 `addLiveIn` 把物理寄存器（A0/A1）标记为 live-in 并建立虚寄存器映射；  
  - `getCopyFromReg` 建 DAG 节点；  
  - 如果是 i8/i16 被提升到 i32，根据 `LocInfo` （SExt/ZExt/AExt）加 `AssertSext/AssertZext` + `TRUNCATE`；  
  - 若 RegVT/ValVT 为 {i32,f32} 或 {i64,f64}，用 BITCAST 还原浮点类型。  
- 对 `VA.isMemLoc()`：  
  - 用 `CreateFixedObject(size, LocMemOffset)` 固定栈对象；  
  - `getFrameIndex` + `getLoad` 生成 load DAG；  
  - load 的 chain value 进入 `OutChains`，最后统一 `TokenFactor`。

同时处理：  
- byval：调用 `copyByValRegs` 把寄存器部分拷到栈、建立 frame object。  
- SRet：找到带 isSRet 标志的参数，为其创建虚寄存器，copy 到 Reg 并在 Chain 上挂一个 `CopyToReg`，后面 ReturnLowering 会用这个 Reg 传回 struct 指针。

***

## LowerCall：CALLSEQ_START/END + 实参搬运

`LowerCall` 负责 call 语句：把虚寄存器中的实参，搬到“调用点规定的位置”（寄存器或 out-args 栈帧），加上 call 本身 DAG。  
基本步骤：  
- 分析 Outgoing args：用 Cpu0CC/CCState 类似 formal side，得到每个实参对应的 Reg/Mem 信息；  
- 对 reg 参数：生成 `CopyToReg` 把值写入 A0/A1 等寄存器；  
- 对栈参数：用 `Store` 把实参写入 caller 栈帧中某个 FI；  
- 插入 `CALLSEQ_START`，调整栈指针，`Cpu0FrameLowering` 后续会将其变成 addiu $sp, -N；  
- 生成 `Cpu0ISD::JmpLink`：  
  - 函数符号节点为 `tglobaladdr` 或 `texternalsym`，匹配到 `JSUB`（或 `JALR`）；  
  - 根据 PIC/非 PIC 加 GOT/CALL16 表达式；  
- 插入 `CALLSEQ_END` 恢复栈指针，生成接收返回值的 `CopyFromReg` nodes（如从 V0 读 i32 返回值）。

***

## JSUB/JALR + Cpu0JmpLink SDNode

在 `.td` 中定义 `Cpu0JmpLink` SDNode（有 chain、glue、可变参数），pattern：  
- `(Cpu0JmpLink (i32 tglobaladdr:$dst)) -> JSUB tglobaladdr`;  
- `(Cpu0JmpLink (i32 texternalsym:$dst)) -> JSUB texternalsym`。  

`JumpLink`/`JumpLinkReg` 模板设置 `isCall=1, hasDelaySlot=1, isTerminator/isReturn/isBarrier` 等标志，JSUB 用 PC-relative call（fixup_Cpu0_CALL16），JALR 则用寄存器间接跳转（ra=LR=14）。

***

## CALL16 relocation 与 GOT_CALL

为支持 PIC 下 调用，MC 层引入 CALL16 relocation：  
- `Cpu0FixupKinds.h` 定义 `fixup_Cpu0_CALL16`；  
- `Cpu0ELFObjectWriter::getRelocType` 把它映射为 `R_CPU0_CALL16`；  
- `Cpu0MCExpr::CEK_GOT_CALL` 表示 `%call16(sym)`，`Cpu0MCCodeEmitter::getExprOpValue` 遇到该 kind 时生成 fixup_Cpu0_CALL16；  
- `Cpu0MCCodeEmitter::getJumpTargetOpValue` 对 JSUB/JMP/BAL push 一个 `fixup_Cpu0_PC24` 用于 PC-relative branch offset。

`Cpu0MCInstLower::LowerSymbolOperand` 根据 MachineOperand target flags 选择 CEK_GOT_CALL 等 TargetKind，并将 ExternalSymbol/Global 提取为 MCSymbolRefExpr，保证 call 的符号 / GOT 表达式在 obj 中被正确重定位。

***

## 入参栈槽范围与 Cpu0FunctionInfo

`Cpu0FunctionInfo` 扩展了 MachineFunctionInfo，增加：  
- InArgFIRange：保存 LowerFormalArguments 创建的所有入参栈 frame index 范围；  
- OutArgFIRange：保存 LowerCall 创建的所有出参栈 frame index 范围（除 $gp 相关）；  
- GPFI：`$gp` 保存点的 frame index，DynAllocFI：动态分配区（用于 alloca/vla）。  

这些信息用于 FrameLowering 调整 SP 或 special-case 处理“这是出参栈槽” vs “普通栈槽”。

***

## 栈上传入参数：loadRegFromStackSlot

MachineInstr 层的 `loadRegFromStackSlot`（前面章节写过）配合 LowerFormalArguments 的 FI，使 Cpu0SEInstrInfo/FrameLowering 在 prologue 生成类似：`ld $A0, offset($sp)` 的 load，把 incoming stack args 从 caller 帧搬到寄存器或临时局部变量。  
`GetMemOperand(..., FI, ...)` 返回 FI 对应的 MemoryOperand，统一用 frame index + displacement 的形式表达入参栈访问。

***

## 结构体参数与返回（sret/byval）

章节专门处理两种 struct 相关情况：  
- sret：返回值通过隐藏参数指针传递，被 LowerFormalArguments 抓出来，存入 `Cpu0FunctionInfo::SRetReturnReg`，ReturnLowering 会在函数返回时从这个指针写 struct 内容。  
- byval struct：`ArgFlags.isByVal()` 为真，`Cpu0CC::handleByValArg` 为它分配一段 caller 栈空间和可能的整数参数寄存器；`copyByValRegs` 把 byval 的寄存器部分复制到栈对象，对 callee 来说它始终像一个栈上的 struct 拷贝。

***

## 函数调用优化：尾调用 / 递归

本章也提到 Cpu0 的函数调用优化接口：  
- `isEligibleForTailCallOptimization(Cpu0CCInfo, NextStackOffset, FI)` 决定一个 call 是否可以变 tailcall（如不增长栈帧、调用同一个函数等）；  
- 递归优化：对自递归调用可以在 LowerCall 中识别并转为 loop 形式，避免多次展开栈。  

这些具体实现没完全展开，但接口已挂在 Cpu0TargetLowering 上，说明 LLVM 调用路径为：在 tail-call friendly 情况下，用 `RET` 替代正常 call+ret。

***

## 其他特性：vararg、动态栈、intrinsics

章节最后列举一些“已接好但不详讲”的 features：  
- PIC 模式下的全局变量访问结合 GP/GOT（前一章 globalvar 的内容）；  
- 可变参数（vararg）：通过在 Cpu0FunctionInfo 中记录 VarArgsFrameIndex，并使用 CCState 把超出固定参数的部分放栈，供 va_start/va_arg 使用。  
- 动态栈分配 / VLA：`DynAllocFI` 指向一个“动态分配区域”的 FI，结合 `Cpu0ISD::DynAlloc` 节点处理 alloca；  
- 与函数相关的 intrinsics：`frameaddress/returnaddress`、`eh.return/eh.dwarf`、`bswap`，以及添加 target specific intrinsic 的接口。

***

你现在更想深入哪块：`LowerFormalArguments`/`LowerCall` 整个 calling convention DAG 链路，还是 JSUB/JALR + CALL16/GOT_CALL 这套 MC/reloc 部分？  
