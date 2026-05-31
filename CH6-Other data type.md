这一章主线是：在只支持 32 位 int/long 的基础上，把“指向局部变量的指针、char/short/bool、long long、数组/结构体、浮点（库函数）、以及 GEP 产生的带偏移全局地址”都接到 Cpu0 backend 上，包括 IR→ SelectionDAG 的形态、TableGen pattern、以及 TargetLowering/ISel 的补丁。

***

## 局部变量指针与 LEA

为了支持“指向栈上局部变量的指针”，作者在 `.td` 里加了 `mem_ea` Operand 和 `EffectiveAddress`/`LEA_ADDiu` 指令模板，专门用来生成 `addiu $ra, base+offset` 这种“取地址”操作。  
原因是：FrameIndex 只在 load/store 的 Operand 上由 Legalizer 特殊处理，而“单纯拿一个栈地址、存入指针”并不会被当成 load/store，因此用一个 mem ComplexPattern 把 `FI+offset` 匹配到 LEA 指令，实现 `p = &b`。

***

## char/short/bool 类型与 load-ext/store-trunc

对 char/short，核心是让 SelectionDAG 在看到 `sext/zext/trunc` 时，能匹配到 `lb/lbu/lh/lhu/sb/sh` 这些带扩展/截断语义的指令。  
实现方式：在 `.td` 定义 `sextloadi8/zextloadi8/sextloadi16_a/zextloadi16_a/truncstorei16_a` 这些 Load/Store DAG 节点，并用 `LB/LBu/LH/LHu/SH/SB` 模板分别绑定到 `load …, <…, sext from i8>` 等 DAG pattern，对应 signed/unsigned char/short cast 的 IR 形态（见表 32、33）。

***

## bool 类型与 BooleanContents

Cpu0 没有 i1 寄存器类型，所以 TargetLowering 构造函数里通过 `setBooleanContents(ZeroOrOneBooleanContent)` 指定“布尔扩展为 0/1 的 i32”，`setBooleanVectorContents(ZeroOrNegativeOneBooleanContent)` 指定向量布尔为全 0 或全 1。  
同时对所有整型 VT 把 `EXTLOAD/ZEXTLOAD/SEXTLOAD i1` 的行为设为 Promote，让 i1 load 自动提升为更宽的类型；在 bool 的简单测试 `verify_load_bool` 中，IR 的 `load i1` 最终被编译成 `sb` 储存 0/1 的一个字节。

***

## long long（i64）与带进位加减/乘法

Cpu0 的 long 是 32 位，long long 是 64 位，需要在 32 位 ISA 上模拟 i64 加减乘和移位。  
为此，TargetLowering 把 `SHL_PARTS/SRA_PARTS/SRL_PARTS` 标记为 Expand，让 DAG 用二个 i32 操作数表示 64 位移位；在 ISelDAGToDAG 中，增加 `selectAddESubE` 处理 `ADDE/SUBE`，通过 `SLTu` 或 `CMP+ANDi` 计算 carry，再用 `ADDu` 把 carry 加到高 32 位，实现 64 位加减；同时对 `SMUL_LOHI/UMUL_LOHI` 特判，调用 `selectMULT` 生成 `MULT/MULTu + MFLO/MFHI` 来得到 64 位乘法结果。

***

## 浮点类型：库函数实现与 clz/clo

当前 Cpu0 没有硬件浮点，float/double 运算通过调用 compiler-rt 库函数实现，比如 `a*b` 被 lower 成 `__mulsf3` 调用。  
为提升浮点库性能，章节顺带加入了整数指令 `clz/clo`（Count Leading Zeros/Ones），在 `.td` 里用 `CountLeading0/CountLeading1` 模板声明 `CLZ/CLO`，这些指令常在 compiler-rt 中被用到，如实现规范化和快速 log/exp。

***

## 数组和结构体：getelementptr 与 offset folding

对数组和 struct，LLVM 使用 `getelementptr` 表达偏移，譬如 `date.day` 生成 `GlobalAddress(@date) + 8` 的 DAG，而 `a[1]` 生成 `GlobalAddress(@a) + 4`。  
TargetLowering 默认在 static relocation model 下允许 `isOffsetFoldingLegal` 返回 true，于是 `add(GlobalAddress, Const)` 被合并为 `GlobalAddress + offset` 单节点，导致之后的 load 只看到 GA 本身，offset 丢失，从而生成 `ld $2, 0($2)` 而不是 `ld $2, 8($2)`；为修正这一点，Cpu0 重写 `isOffsetFoldingLegal` 返回 false，即“目标尚不支持把 offset 折叠进 GA”，迫使 offset 以立即数形式保留在 load 中。

***

## SelectAddr 中 FI+const 与 GA+const

在 ISelDAGToDAG 的 `SelectAddr` 里，作者补充了 `isBaseWithConstantOffset` 分支：当地址是 `Base+Const`，若 Base 是 FrameIndex，则用 TargetFrameIndex 作为 Base，否则 Base 原样；Const 转为 TargetConstant，用来匹配 `ld/st` 的 base+offset 寻址。  
这保证了不管是栈上的 `FI+offset`，还是全局的 `GA+offset`，都能以 base+16-bit offset 的形式匹配到 Cpu0 Load/Store pattern，解决 GEP 生成的 struct/array 偏移问题。

***

## 总结：类型支持的 IR→DAG→指令映射思路

这一章给出的表 32/33 很核心：用 C 代码 → bitcode → “Optimized legalized selection DAG” → Cpu0InstrInfo.td 中 pattern 的方式，把 char/short/bool 的 `sext/zext/trunc`、long long 的 `ADDC/ADDE/SUBC/SUBE/SMUL_LOHI`、以及 struct/array 的 GEP 全部连到了实际指令。  
对你写自己后端来说，关键思想是：先从 IR 观察出这些操作在 DAG 中的规范形态（尤其是 load-ext、store-trunc、*_PARTS、GlobalAddress+offset），再在 TargetLowering 中控制它们的 Legalization 结果，最后在 `.td`/DAGToDAG ISel 里一一写 pattern 或手写选择逻辑。

你接下来更想进一步琢磨的是哪一块：小整数类型（lb/lbu/lh/lhu + bool）、64 位 long long 的 ADDE/SUBE/MUL 逻辑，还是 GEP 和 isOffsetFoldingLegal 那套 struct/array offset 处理？  
