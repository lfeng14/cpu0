这一章主线是：在 Cpu0 上把“IR 里的全局变量访问”根据 `-relocation-model` 和 `-cpu0-use-small-section` 四种组合，翻译成不同的 SelectionDAG 形态，再通过 `.td` pattern 映射到 `lui/ori` 或 GP 相对 / GOT 访问序列。

***

## 选项空间与两类寻址模式

文中先给出“Cpu0 global variable options”：`-relocation-model` 为 `static` 或 `pic`，`-cpu0-use-small-section` 为 `false` 或 `true`，组合成 4 种模式。  
按系统编程教科书分类，对应两类寻址：Absolute Addressing（static + 非 small section ）和 Position-Independent Addressing（PIC 以及某些 static+small section 变体），其中 small section 会走 GP 相对寻址。

***

## UseSmallSection / ReserveGP / NoCpload 选项

`Cpu0Subtarget` 新增 `UseSmallSection` 字段，对应命令行 `-cpu0-use-small-section`，默认 false；同时定义 `Cpu0ReserveGP` 和 `Cpu0NoCpload`，将来用于链接器和 `.cpload` 处理。  
`UseSmallSectionOpt` 控制 small section 是否可用，`IsGlobalInSmallSection` 系列函数通过 subtarget 的 `useSmallSection()` 和全局大小阈值 `SSThreshold` 判断一个 Global 是否放进 `.sdata/.sbss`。

***

## TargetObjectFile：.data/.bss vs .sdata/.sbss

`Cpu0TargetObjectFile::SelectSectionForGlobal` 根据 `SectionKind` 和 `IsGlobalInSmallSection`，选择 `.sdata/.sbss` 用于小全局（有无初始化分别对应），否则退回 ELF 默认 `.data/.bss`。  
`IsGlobalInSmallSectionImpl` 只对非函数的 `GlobalVariable` 生效，使用模块 DataLayout 计算对象大小，大小在 `(0, SSThreshold]` 才进入 small section。

***

## GlobalAddress 的自定义 Lowering 入口

`Cpu0TargetLowering` 构造函数中调用 `setOperationAction(ISD::GlobalAddress, MVT::i32, Custom)`，让 `ISD::GlobalAddress` 走 `LowerOperation`。  
`LowerOperation` 对 `ISD::GlobalAddress` 分发到 `lowerGlobalAddress`，后者是本章所有模式分支的统一入口，内部根据 relocation model、small section 与 linkage 条件生成不同的 SelectionDAG 子树。

***

## 非 PIC + 非 small section：%hi/%lo 绝对地址

当 `!isPositionIndependent()` 且 global 不在 small section 时，`lowerGlobalAddress` 使用 `getAddrNonPIC` 形成功能 DAG：  
- `getTargetNode` 生成 `TargetGlobalAddress` 节点（带 `MO_ABS_HI/LO` flag）；  
- 用 `Cpu0ISD::Hi`/`Cpu0ISD::Lo` 包裹 hi/lo 节点；  
- 再 `ISD::ADD` 把两者相加，得到绝对地址 DAG `(add Cpu0ISD::Hi Cpu0ISD::Lo)`。

对应的 `.td` pattern：  
- `Cpu0Hi`/`Cpu0Lo` SDNode，`Pat<(Cpu0Hi tglobaladdr:$in), (LUi tglobaladdr:$in)>` 把 Hi 节点变为 `lui`;  
- `Pat<(Cpu0Lo tglobaladdr:$in), (ORi ZERO, tglobaladdr:$in)>` 把 Lo 变为 `ori`;  
- `Pat<(add CPURegs:$hi, (Cpu0Lo tglobaladdr:$lo)), (ORi CPURegs:$hi, tglobaladdr:$lo)>` 把 add hi+lo 变成 `ori`。

最终指令序列为：`lui $2, %hi(gI); ori $2, $2, %lo(gI); ld $2, 0($2)`，实现绝对地址装载。

***

## 非 PIC + small section：GP 相对加载

当 `!isPositionIndependent()` 且 `IsGlobalInSmallSection(GO)` 为真时，`lowerGlobalAddress` 构造 DAG：  
- `TargetGlobalAddress` 带 `MO_GPREL`;  
- 包裹为 `Cpu0ISD::GPRel` 节点；  
- `getRegister(Cpu0::GP)` 为 base；  
- `ISD::ADD(GP, GPRelNode)` → `(add register %GP, Cpu0ISD::GPRel)`。

`.td` 中 `Cpu0GPRel` SDNode 和 `Pat<(add CPURegs:$gp, (Cpu0GPRel tglobaladdr:$in)), (ORi CPURegs:$gp, tglobaladdr:$in)>` 把该 DAG 转为 `ori $2, $gp, %gp_rel(gI)`。  
结合 `ld $2, 0($2)`，构成 GP-relative 访问 `gI` 的序列：`ori $2, $gp, %gp_rel(gI); ld $2, 0($2)`，对应 small section 16-bit offset 模式。

***

## PIC 模式：local vs global & small vs large GOT

在 PIC 模式下，`lowerGlobalAddress` 针对不同场景使用三个 helper：`getAddrLocal`、`getAddrGlobal` 和 `getAddrGlobalLargeGOT`。  

- 对 internal/local linkage 的 GV，调用 `getAddrLocal` 构造 DAG `(add (load (wrapper $gp, %got(sym))), %lo(sym))`：  
  - `Cpu0ISD::Wrapper(getGlobalReg($gp), TargetGlobalAddress(MO_GOT))` 表示 `%got(sym)` 条目指针；  
  - `load` 从 GOT 中取出地址，再用 `Cpu0ISD::Lo` 处理低 16 位，加到 load 结果上。

- 对非 small section 全局，`getAddrGlobalLargeGOT` 构造 DAG `(load (Cpu0ISD::Wrapper (add Cpu0ISD::Hi, %GP), Cpu0ISD::Lo))`：  
  - 先 `Cpu0ISD::Hi` + `%GP` 得到高地址基，  
  - 再 `Wrapper(Hi+GP, Lo)`，最后 `load`。

- 对 small GOT 情况，`getAddrGlobal` 通过 `Wrapper($gp, %got(sym))` + `load`，即 `(load (wrapper $gp, %got(sym)))`。

***

## PIC + 非 small section：%got_hi/%got_lo

表 31 中，`-relocation-model=pic -cpu0-use-small-section=false` 生成 DAG `(load (Cpu0ISD::Wrapper (add Cpu0ISD::Hi, Register %GP), Cpu0ISD::Lo))`，被 `.td` pattern 匹配为：`lui $2, %got_hi(gI); add $2, $2, $gp; ld $2, %got_lo(gI)($2)`。  
这里 Hi/Lo flags 使用 `MO_GOT_HI16/MO_GOT_LO16`，对应 ELF reloc `R_CPU0_GOT_HI16/R_CPU0_GOT_LO16`，link/load 阶段填充 GOT 入口地址。

***

## PIC + small section：%got(gI)($gp)

另一种 PIC + small section 组合（`-cpu0-use-small-section=true`）下，DAG 为 `(load (Cpu0ISD::Wrapper %GP, TargetGlobalAddress(MO_GOT)))`，`.td` 的 `WrapperPat` 将其映射为 `ORiOp $gp, node`。  
在汇编中呈现为 `ld $2, %got(gI)($gp)`，即直接用 `$gp` + GOT offset 访问 GOT entry，再 `ld` 得到真正地址。

***

## getGlobalReg 与 GLOBAL_OFFSET_TABLE

`Cpu0TargetLowering::getGlobalReg` 从 `Cpu0FunctionInfo` 中取出 global base reg（通常是 `$gp`），并通过 `SelectionDAG::getRegister` 生成对应 SDValue。  
DAGToDAG selector 中 case `ISD::GLOBAL_OFFSET_TABLE` 使用 `getGlobalBaseReg`（从 MachineFunctionInfo 取 GP，生成 `CurDAG->getRegister` 节点）替换原节点，让 GOT 基址在 isel 时落实到具体物理寄存器。

***

## SelectAddr：Wrapper / 非 PIC Address 选择

`Cpu0DAGToDAGISel::SelectAddr` 在 isel 阶段负责将地址 SDValue 分解成 base+offset：  
- 若 `Addr` opcode 为 `Cpu0ISD::Wrapper`，直接取其两个 operand 分别作为 `Base` 和 `Offset`，用于 PIC 下 GOT/GLOBL load。  
- 非 PIC 时，若 `Addr` 是 `TargetExternalSymbol` 或 `TargetGlobalAddress`，直接返回 false，交给其他 pattern 处理 hi/lo 形式。

这样 load/store pattern 可以统一写在 `.td` 中，使用 `Base+Offset` 形式对绝对地址和 GP-relative 地址进行匹配。

***

## 寄存器保留 GP 与 small section 约束

`Cpu0RegisterInfo::getReservedRegs` 把 `Cpu0::GP` 标记为 reserved，避免寄存器分配器分配 `$gp` 给普通变量，从而保证 `$gp` 始终用作 global base。  
注释中强调：static + small section = `$gp` reserved & fixed global base；PIC 模式下也需要保留 `$gp` 给 GOT，`Cpu0ReserveGP` 和 `Cpu0NoCpload` 则为未来链接器/运行时提供更多策略控制。

***

## 总结表：四种模式的 DAG 与指令

文中给了两个表：  
- 表 30（static）总结 `cpu0-use-small-section` false/true 下的 addressing mode（absolute vs GP-relative）、Legalized SelectionDAG 形态和最终 Cpu0 指令（`lui/ori` vs `ori $gp,%gp_rel`）。  
- 表 31（pic）总结 small section false/true 的 DAG 形态（`Wrapper` + load 或 Hi/Lo+Wrapper+load）及指令序列（`ld $2,%got(gI)($gp)` vs `lui/add/ld %got_hi/%got_lo`），并指出所有 PIC 情况下 relocation 需在 link/load 阶段解决。

***

你写自己后端时，现在更想优先搞清楚的是哪块：`lowerGlobalAddress` 里各模式分支如何裁剪到你自己的 ABI，还是 `.td` 里的 Hi/Lo/GPRel/Wrapper pattern 与 ELF reloc 的对应关系？  
