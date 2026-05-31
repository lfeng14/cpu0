这一章主要是把“一个后端从 IR 到汇编”的骨架搭出来：从 TargetMachine，到 AsmPrinter/ISel，再到返回寄存器 LR、prologue/epilogue、栈槽布局、DAG/MC 层的数据流。下面我按目录帮你提炼“核心对象 + 它在 pipeline 里负责什么 + 你写 backend 时要注意什么”。

***

## 整体 backend 数据流

文中先用图 16–19 概括了从 LLVM IR 到汇编/二进制、再到装配/反汇编的通用数据结构：MachineInstr → MCInst → 二进制，或者反向从二进制到 MCInst/汇编。  
关键点是：编码/打印/匹配/反汇编那几个函数（encodeInstruction、printInst、MatchAndEmitInstruction、getInstruction）在编译器、汇编器和反汇器之间共享，这也是 LLVM MC 层复用性的基础。

***

## TargetMachine 结构与职责

Cpu0TargetMachine 继承 LLVMTargetMachine，封装了 DataLayout、Subtarget、ABI、对象文件 lowering（TLOF）、PassConfig 等信息，是后端的“入口对象”。  
构造函数里用 computeDataLayout 构造 DL 字符串（endian、指针大小、对齐、stack alignment 等），并根据 isLittle 配置大小端，同时选取重定位模型和 code model。

***

## 大端/小端 TargetMachine 派生类

Cpu0ebTargetMachine 和 Cpu0elTargetMachine 分别表示 big-endian 和 little-endian 目标机，通过构造参数中的 isLittle 区分大小端，并各自注册到 TargetRegistry 中。  
外部 C 接口 LLVMInitializeCpu0Target 里用 RegisterTargetMachine 注册这两个 TM，这就是 llc -march=cpu0 / cpu0el 能被识别的原因。

***

## Subtarget 与 getSubtargetImpl

Cpu0TargetMachine 内部维护 DefaultSubtarget 和一个 SubtargetMap（以 CPU+FS 为 key），getSubtargetImpl(Function &F) 根据函数属性（CPU/特性字符串）选择或创建对应的 Cpu0Subtarget。  
resetTargetOptions(F) 确保 TM/TargetOptions 与函数 codegen 选项一致，然后按需创建新的 Subtarget；这就是 per-function subtarget 特性的基础，实现像 ARM/MIPS 那样的 per-function feature。

***

## PassConfig 与后端 pass pipeline

内部类 Cpu0PassConfig 继承 TargetPassConfig，并重载 getCpu0TargetMachine / getCpu0Subtarget，createPassConfig 返回该类型实例，控制后端 pass 的顺序和组合。  
你后面所有和 selection、schedule、register allocation 相关的后端 pass（比如插入 ISel、branch relax、peep-hole 等），都是通过 Cpu0PassConfig 来注册进 pipeline 的。

***

## TargetObjectFile 与小数据段

Cpu0TargetObjectFile 继承 TargetLoweringObjectFileELF，在 Initialize 中创建 .sdata 和 .sbss 的 MCSection，并保存 Cpu0TargetMachine 指针；SSThreshold 决定 small data/bss 的阈值。  
这些对象文件 lowering 逻辑用于把小的全局变量放入 .sdata/.sbss，以支持 GP-relative 访问、减少指令大小；这也是很多 RISC 架构（如 MIPS）的常见优化策略。

***

## CallingConv .td 与 Callee Saved 集合

Cpu0CallingConv.td 描述了 Cpu0 的调用约定，包括 CCIfSubtarget 等条件，以及 CSR_O32 这类 CalleeSavedRegs 定义，其中包括 LR、FP 和一串 S 寄存器序列。  
这里的 CSR 集合会被 FrameLowering / PrologueEpilogue 插入器使用，用于在函数 prologue/epilogue 里保存/恢复被调保存寄存器，这是调用约定正确性的基础。

***

## TargetInstrInfo / Cpu0InstrInfo

TargetInstrInfo 是 Instruction level 接口，继承 MCInstrInfo，负责指令属性查询、branch 分解/插入、堆栈操作等；TargetInstrInfoImpl 提供默认实现，并内建 CallFrameSetup/Destroy opcode。  
Cpu0InstrInfo 继承 TableGen 生成的 Cpu0GenInstrInfo，并持有 Subtarget 引用，负责 getRegisterInfo、GetInstSizeInBytes 等接口，后者通过 opcode/desc.getSize 返回 MI 的字节数，用于布局/调度等。

***

## Cpu0InstrInfo.td 中的 Predicate

Cpu0InstrInfo.td 定义了一系列 Predicate（Ch3_1、Ch3_2...Ch12_1、Ch_all 等）和 AssemblerPredicate，用来根据 Subtarget Feature 控制不同章节功能的开启，例如 hasChapter3_1。  
此外还有 HasCmp/HasSlt、EnableOverflow/DisableOverflow 等 predicate，用于 TableGen pattern 中条件启用某些指令或 pattern，比如根据 ISA 版本使用 CMP 还是 SLT 序列。

***

## ISel Lowering：Cpu0TargetLowering

Cpu0ISelLowering.h 中定义 Cpu0ISD::NodeType 枚举（JmpLink、TailCall、Hi、Lo、GPRel、ThreadPointer、Ret、EH_RETURN、DivRem/DivRemU、Wrapper、DynAlloc、Sync 等），用于目标特定 DAG 节点编码。  
Cpu0TargetLowering 继承 TargetLowering，负责把 LLVM IR lower 成 SelectionDAG，提供 getTargetNodeName、调用约定 lower（CallingConvLowering）和各种 operation lowering，是 IR→DAG 的核心组件。

***

## DAGToDAG 指令选择器

这一章的“Add Cpu0DAGToDAGISel class”部分（源码在 Cpu0Subtarget/ISel 相关文件）定义了基于 SelectionDAGISel 的 DAG-to-DAG selector，利用 TableGen 生成的 pattern 将 DAG 节点转换为目标 MachineInstr。  
这一步与 Cpu0TargetLowering 紧密配合：前者生产目标特定 DAG 节点，后者通过 pattern 匹配生成具体指令，是“IR → DAG → MachineInstr”阶段的关键连接点。

***

## AsmPrinter 与 MC 层接口

Add AsmPrinter 小节中，作者接入了 Cpu0AsmPrinter，让 llc 能把 MachineInstr 通过 MCStreamer/MCInst 输出为汇编文本，printInst 由 Cpu0InstPrinter 复用编译/反汇器。  
重要的是：Cpu0MCCodeEmitter 的 emitInstruction 既被 llc 使用也被 as 使用，因此编码逻辑必须目标无关、可重入，保证编译器和汇编器对指令编码一致。

***

## LR（返回地址寄存器）处理

Handle return register $lr 小节讨论了 LR 的保存与恢复：调用通过 JmpLink 之类的节点把返回地址写入 LR，函数返回时通过 Ret 节点读取 LR 并跳转。  
LR 通常也作为 Callee Saved 寄存器之一，在 prologue/epilogue 中被保存到栈上并恢复，特别是在需要嵌套调用或支持异常时，保持 LR 正确性非常关键。

***

## FrameLowering：Cpu0FrameLowering

Cpu0FrameLowering 继承 TargetFrameLowering，封装 stack grow 方向、对齐和 hasFP 逻辑；构造函数中设置 StackGrowsDown 和对齐（Align(Alignment)），并保存 Cpu0Subtarget 引用。  
hasFP 逻辑（变量大小 alloca、需要 realign、禁止 FP elimination 或 frame address 被取）和 x86/ARM 等大致一致，是是否保留独立 FP 寄存器的判断入口。

***

## FrameLowering：Cpu0SEFrameLowering 与 Prologue/Epilogue

Cpu0SEFrameLowering 继承 Cpu0FrameLowering，专门为 SE（32/64-bit）变体实现 emitPrologue/emitEpilogue，负责调整 SP、保存/恢复 Callee-saved、设置/销毁 FP 等。  
当前章节中 emitPrologue/emitEpilogue 还是空壳，后续章节会填充为 MBB 中插入具体栈帧操作指令的逻辑，实现注释里描述的栈帧布局（local、alloca、saved GP、saved FP/LR 等）。

***

## 栈帧布局与 FI/StackOffset 约定

FrameLowering 源码注释详细描述了 Cpu0 的栈帧：SP 向下增长（栈顶在低地址），但 prologue 先减 SP，然后所有访问用正偏移（把栈当成向上增长），方便 ABI 对齐和访问。  
函数参数被视为“前一帧”的一部分、其 FrameIndex 为负值，eliminateFrameIndex 时通过 StackSize + offset 把它们映射到当前 SP 上合适的偏移，保证 caller/callee 在 ABI 上对齐。

***

## Data operands DAGs

Data operands DAGs 小节提到 SelectionDAG 上对数据操作数的建模：立即数拆分（Hi/Lo）、GP-relative、byval 参数拆分、struct/数组访问等都通过 DAG 节点描述。  
这些节点在 ISel 阶段被 pattern 匹配成指令序列，例如 32-bit 立即数通过 LUi(Hi) + ORi(Lo) 实现，GPRel 则映射到基于 GP 的加载/存储，这对支持复杂寻址和重定位类型是必需的。

***

## 分章节 feature 与教学结构

章节中大量代码“几乎全部从 Mips 拷贝改名为 Cpu0”，并通过 FeatureChapterX_Y 控制启用，这是为了教学：你可以逐章启用/验证某一类功能（比如 frame lowering、branch、load/store）。  
理解这套 feature gating 和 Subtarget->hasChapterX_Y() 的关系，有助于你在实际项目中做“渐进式 bringup”，先让 core ISA 跑起来，再逐步打开扩展或复杂特性。

***

如果你现在打算照着这章动手写自己的后端，你目前更想优先搞清楚的是哪块：TargetMachine / Subtarget 这一层，还是 FrameLowering / prologue-epilogue 的实现细节？
