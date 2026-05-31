这一章主线：在已有 “只输出汇编” 的 Cpu0 backend 基础上，接入 MC 层（MCCodeEmitter、MCAsmBackend、MCExpr、ELFObjectWriter、FixupKinds），让 `llc -filetype=obj` 能产出 big/little-endian ELF obj，并介绍 target registration 结构。

***

## 从汇编到 ELF obj

前面章节只支持 `llc -filetype=asm`，到这里才支持 `-filetype=obj`，否则会报 “target does not support generation of this file type!”。  
Chapter5_1 增加后，`llc -march=cpu0` 生成 big-endian ELF32，`-march=cpu0el` 生成 little-endian ELF32，用 `objdump -s` 可看到 `.text` 段的指令编码，并且可以手算验证 opcode/reg/imm 的 bitfield。

***

## MC 层数据流概念

这一章对应前文 Backend Structure 图 22 的“汇编输出路径”，但焦点换成“机器码对象文件路径”：`MachineInstr → MCInst → bytes → ELF obj`（图 30）。  
关键在于，`Cpu0AsmPrinter` 把 MachineInstr 转为 MCInst，后续编码与写入由 `MCCodeEmitter` 和 `MCAsmBackend` 接管，写入 ELF obj 的 relocation 和 section 元信息由 `MCELFObjectWriter` 子类处理。

***

## Cpu0MCCodeEmitter：指令编码

`Cpu0MCCodeEmitter` 继承 `MCCodeEmitter`，构造时持有 `MCInstrInfo` 和 `MCContext`，并记录当前是否小端（big/little 用不同工厂函数）。  
核心职责：  
- `encodeInstruction`：调用 TableGen 生成的 `getBinaryCodeForInstr` 得到 32-bit 编码，检查 pseudo/未实现 opcode，然后调用 `EmitInstruction` 写出 4 字节（按大小端）。  
- `getMachineOpValue`：把 `MCOperand` 的 Reg/Imm/Expr 编成实际 bitfield，寄存器用 `MCRegisterInfo::getEncodingValue`，立即数直接返回，表达式交给 `getExprOpValue` 处理 fixup。

***

## 指令字段布局与 getMemEncoding

`getMemEncoding` 负责 load/store 等“基址+偏移”操作数编码：base reg 在 bits 20–16，offset 在 15–0，返回 `(OffBits & 0xFFFF) | (RegBits << 16)`。  
这与 Cpu0 ISA 设计呼应——所有字段按 4-bit 对齐（8-bit opcode、4-bit reg、16-bit imm），方便你从 objdump 输出中直接按 nibble 拆字段检查正确性。

***

## MCExpr/Cpu0MCExpr：目标特定表达式

`Cpu0MCExpr` 继承 `MCTargetExpr`，封装各种 `%hi()/%lo()/gp_rel/got/tls` 风格的目标特定表达式（`Cpu0ExprKind`）。  
它提供 `create` 和 `createGpOff` 工厂函数，把符号或子表达式包装为目标 expr，并在 `printImpl` 中打印 `%hi(sym)`、`%lo(sym)`、`%got(sym)` 等形式，evaluate/evaluateRelocatable 则委托子表达式。

***

## FixupKinds：Cpu0::Fixups 枚举

`Cpu0FixupKinds.h` 定义了目标特定 fixup 种类：`fixup_Cpu0_32`, `fixup_Cpu0_HI16`, `fixup_Cpu0_LO16`, `fixup_Cpu0_GPREL16`, `fixup_Cpu0_GOT`, `fixup_Cpu0_GOT_HI16`, `fixup_Cpu0_GOT_LO16` 等。  
枚举顺序必须与 `Cpu0AsmBackend` 中 `MCFixupKindInfo Infos[...]` 一致，保证 Kind→Info 映射正确，这样 MC 层才能知道某 fixup 占多少位、是否 PCRel、是否 Constant 等。

***

## getExprOpValue 与 fixup 记录

`Cpu0MCCodeEmitter::getExprOpValue` 根据 `MCExpr::ExprKind` 处理常量、二元 expr 和目标 expr：  
- Constant/binary：直接折叠为整数常量（如 `a+b`）。  
- Target expr（`Cpu0MCExpr`）：根据 kind 选具体 `Cpu0::Fixups`，在 `Fixups` 数组里 push 一个 `MCFixup`（offset=0，kind=FixupKind），返回 0，表示数值由 linker 填充。

这意味着“所有 relocation 所需信息存于 fixup 记录中”，编码阶段只写出占位 0，后续由 AsmBackend/ELFObjectWriter 结合 fixup 和符号生成真实重定位项。

***

## Cpu0AsmBackend：fixup 应用

`Cpu0AsmBackend` 继承 `MCAsmBackend`，构造时根据 Triple 的大小端选择父类支持类型（little/big）。  
主要职责：  
- `createObjectTargetWriter()`：返回 `createCpu0ELFObjectWriter(TheTriple)`，把 target 接到 ELF writer。  
- `applyFixup`：调用 `adjustFixupValue` 预处理 Value（如 HI16 取 `(Value>>16)&0xFFFF`，GOT 同理），然后从 Data 中按 offset/NumBytes 读当前 value，按大小端布局合入新值，再写回。

***

## adjustFixupValue：HI/LO/GOT 处理

`adjustFixupValue` 针对不同 fixup：  
- `FK_GPrel_4/FK_Data_4/fixup_Cpu0_LO16`：直接使用低 16 位。  
- `fixup_Cpu0_HI16/fixup_Cpu0_GOT`：取 `(Value >> 16) & 0xFFFF`，并（按注释）可以在 bit 15 为 1 时加 1 做 rounding（实际实现保留为简单 hi 部分）。  

这对应经典 MIPS 风格 HI/LO 对：HI 存 `((sym+offset)>>16)`，LO 用 `sym+offset` 的低 16 位，linker 需要在重定位时保持两者协同。

***

## getFixupKindInfo 与 HasLLD

`getFixupKindInfo` 返回每个目标 fixup 的 name、offset、bits、flags，其中 JSUBReloRec flag 依据 `HasLLD`（命令行 `-has-lld`）决定是否设置 `FKF_IsPCRel` 和 `FKF_Constant`。  
这样可以根据有没有 lld linker 调整某些 PC-relative fixup 的行为，兼容不同链接器实现的期望（例如 JSUB 的 relocs）。

***

## writeNopData

`writeNopData` 被实现为简单返回 true（但没实际输出 pattern），表示“能够写 NOP 序列”，这里简化处理。  
真实后端通常会在这里输出最优 nop 序列（如 `sll $0,$0,0` 或特定 fill pattern），用于对齐和填充代码空隙。

***

## Cpu0ELFObjectWriter：RelocType 映射

`Cpu0ELFObjectWriter` 继承 `MCELFObjectTargetWriter`，构造时设置 `ELF::EM_CPU0` 机器号和 `HasRelocationAddend`（N32/N64 风格）。  
`getRelocType` 根据 FixupKind 决定具体 ELF relocation：  
- `FK_Data_4/fixup_Cpu0_32` → `R_CPU0_32`。  
- `fixup_Cpu0_GPREL16` → `R_CPU0_GPREL16`。  
- `fixup_Cpu0_GOT` → `R_CPU0_GOT16`。  
- `fixup_Cpu0_HI16`/`LO16` → `R_CPU0_HI16`/`R_CPU0_LO16`。  
- GOT_HI16/GOT_LO16 → `R_CPU0_GOT_HI16`/`R_CPU0_GOT_LO16`。

***

## needsRelocateWithSymbol 策略

`needsRelocateWithSymbol` 决定 relocation 指向 symbol 还是 section：  
- 默认返回 true，即保守地认为需要 symbol。  
- `R_CPU0_GPREL16` 返回 false，因为 GP-relative 可以按 section 处理。  
- `R_CPU0_GOT16` 理论上应已处理完（`unreachable`），注释说 Cpu0 PIC 下应该 OK 但未完全确认。  

同时注释中强调这是“极度保守”的实现，未来应基于白名单明确哪些 reloc 必须保持 symbol 以配对 hi/lo 等组合重定位。

***

## InstPrinter 与 Cpu0MCExpr 集成

`Cpu0InstPrinter.cpp` 增加 `#include "MCTargetDesc/Cpu0MCExpr.h"`，使汇编打印时能识别并输出 `%hi/%lo/%got` 等表达式格式。  
这样在 obj → asm 的反汇编中，relocation 对应的表达式形式能够被正确还原，便于阅读和再汇编。

***

## Target 注册结构

这一章还扩展了 MCTargetDesc 层的 CMake 列表，加入 `Cpu0AsmBackend.cpp`, `Cpu0MCCodeEmitter.cpp`, `Cpu0MCExpr.cpp`, `Cpu0ELFObjectWriter.cpp`, `Cpu0TargetStreamer.cpp` 等文件。  
配合之前的 TargetMachine 注册，这些组件通过 `createCpu0MCCodeEmitterEB/EL`, `createCpu0AsmBackend`, `createCpu0ELFObjectWriter` 等工厂函数接入 LLVM TargetRegistry，使 `llc/as/objdump` 能全路径处理 Cpu0 目标的 obj.

***

## 章节工作流：验证 obj 输出

章节里给了一套完整工作流：  
- 用前面章的 IR/asm 示例（如 `ch4_1_math.bc`）运行 `llc -march=cpu0 -filetype=obj` 得到 `.o`。  
- 用 `objdump -s` 查看 `.text` 内容，逐条对照汇编 `addiu/st` 的 opcode/reg/imm，将 0x09ddffc8 解码为 addiu $sp,-56，0x022b0034 解码为 `st $2,52($fp)`，并验证大小端字节序。

这一步帮助你从“DAG/MI 层思维”切换到“MC/obj 层思维”，确认 fixup/reloc/endian/bitfield 是否如预期工作。

***

你现在在做自己的后端时，更想优先搞清楚哪一块：指令编码这层（MCCodeEmitter + FixupKinds），还是 ELF relocation/GP/GOT 那一套（Cpu0ELFObjectWriter + Cpu0MCExpr）？  
