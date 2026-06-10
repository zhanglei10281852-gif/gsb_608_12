# Wren VM 内核深度分析

本文档对 Wren 脚本语言的 C 虚拟机内核（`src/vm/`）做完整静态阅读分析，所有结论锚定到具体的文件、宏、字段与函数。

---

## 图 1：NaN Tagging 64 位值布局

```
 63 62  52 51           0 (bit index, MSB→LSB)
+--+------+---------------+
|S| Exp  |   Mantissa    |   IEEE 754 double
+--+------+---------------+

Normal double: exponent bits != all-1's, value stored natively in FPU hardware.

Quiet NaN (QNAN) 区域（所有指数位=1，最高尾数位=1）被 Wren 占用以编码非数字值：

 63 62 51 50               3  2  1  0
+--+-----+------------------+-----+
|0| QNAN|  (unused, all 0)  | TAG |   S=0 → 单例值 (null/false/true/undefined)
+--+-----+------------------+-----+
  sign bit clear + QNAN pattern in bits 62..51 + type tag in bits 0..2

 63 62 51 50                                0
+--+-----+-----------------------------------+
|1| QNAN|          pointer (48 bits)         |   S=1 → 堆对象指针
+--+-----+-----------------------------------+
  sign bit set + QNAN pattern + 指针压入尾数低 48 位（x86-64 用户地址只用 48 位）

QNAN       = 0x7ffc000000000000   （指数位11个全1 + mantissa最高位=1）
SIGN_BIT   = 0x8000000000000000   （第63位）
MASK_TAG   = 0x7                  （低 3 位用于 singleton tag）

TAG 取值（低 3 位）：
  0 = TAG_NAN (普通 NaN 数字值)
  1 = TAG_NULL      → NULL_VAL
  2 = TAG_FALSE     → FALSE_VAL
  3 = TAG_TRUE      → TRUE_VAL
  4 = TAG_UNDEFINED → UNDEFINED_VAL
  5–7 未使用
```

## 图 2：闭包与上值生命周期

```
                编译期 (wren_compiler.c)                          运行期 (wren_vm.c)
  +-----------------------------+             +-------------------------------------+
  | 外层 fn: locals[i]          |             | fiber stack                         |
  |   isUpvalue=true <----------+--findUpvalue| ...                                |
  |                             |             | [stackStart + i] = <外层局部 var> <--+
  | 内层 Compiler:              |             |           ^                         |
  |  upvalues[k] = {isLocal=T,  |             |           | ObjUpvalue* (开放上值)  |
  |                 index=i}    |  CODE_CLOSURE|           |  .value 指向栈槽        |
  |                             |  emit operands          |  .closed = 未使用       |
  | 生成字节码：CLOSURE <const> |-----------> | [new ObjClosure pushed]             |
  |   1 i                       |             |   .fn = <Fn>                        |
  |                             |             |   .upvalues[k] = captureUpvalue()---+
  +-----------------------------+             |                                      |
                                              | 当外层 local 出作用域 (POP/CLOSE_UPVALUE/RETURN): |
                                              |   closeUpvalues(fiber, last)         |
                                              |     upvalue.closed = *upvalue.value  |
                                              |     upvalue.value = &upvalue.closed  |
                                              |   → ObjUpvalue 从"开放"变成"关闭"   |
                                              |     栈槽可被安全回收                |
                                              +-------------------------------------+
```

---

## 1. 值表示与 NaN Tagging

**关键文件**：[wren_value.h](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h)，辅助 [wren_math.h](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_math.h)

### 1.1 两种 Value 表示的编译开关

在 [wren_value.h#L121-L147](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L121-L147) 通过 `#if WREN_NAN_TAGGING` 选择表示：

- **NaN tagging（默认）**：`typedef uint64_t Value;`
- **struct 表示（调试/移植）**：`typedef struct { ValueType type; union { double num; Obj* obj; } as; } Value;`

该开关默认为 1（见 [wren_common.h#L28-L30](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_common.h#L28-L30)）。

### 1.2 NaN Tagging 位域定义

在 [wren_value.h#L552-L591](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L552-L591)：

| 宏                                          | 值                    | 作用                                       |
| ------------------------------------------- | --------------------- | ------------------------------------------ |
| `SIGN_BIT`                                  | `((uint64_t)1 << 63)` | 比特 63，区分数类与指针类                  |
| `QNAN`                                      | `0x7ffc000000000000`  | 指数 11 位全 1 + 尾数最高位 1（quiet NaN） |
| `MASK_TAG`                                  | `7`                   | 低 3 位，用于 singleton 类型标签           |
| `TAG_NULL/TAG_FALSE/TAG_TRUE/TAG_UNDEFINED` | 1–4                   | 各 singleton 的具体标签值                  |

### 1.3 类型判定宏

- `IS_NUM(value)`（[L559](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L559)）：`((value) & QNAN) != QNAN` —— 任何合法 double 都不满足"指数全 1 且尾数最高位为 1"，因此按位与后不等于 QNAN，判定为数字。
- `IS_OBJ(value)`（[L562](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L562)）：`((value) & (QNAN | SIGN_BIT)) == (QNAN | SIGN_BIT)` —— QNAN 位与符号位同时置位 → 堆对象指针。
- `AS_BOOL(value)`（[L582](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L582)）：`(value) == TRUE_VAL`，直接用 uint64_t 比较。
- `AS_OBJ(value)`（[L585](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L585)）：`(Obj*)(uintptr_t)((value) & ~(SIGN_BIT | QNAN))`，把高 13 位标签掩掉，剩下尾数 48 位即指针。

### 1.4 装箱/拆箱函数

在 [wren_value.h#L840-L878](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L840-L878)：

- `wrenNumToValue(double)` → `wrenDoubleToBits(num)`（见 [wren_math.h#L27-L32](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_math.h#L27-L32)），通过 `union { double num; uint64_t bits64; }` 零拷贝位重解释。
- `wrenValueToNum(Value)` → 反向用 union 还原。
- `wrenObjectToValue(Obj*)`（L840）：`SIGN_BIT | QNAN | (uint64_t)(uintptr_t)obj`，三次类型转换（`Obj* → uintptr_t → uint64_t → uint64_t`）满足 32/64 位编译器要求。

### 1.5 "数字算术零开销"

关键在于数字的位模式就是原生 IEEE 754 double，没有位移/掩膜/union 转换之外的任何开销：

- `NUM_VAL(x)` 只是把 double 的位直接当 uint64_t 传递；
- 取出数字时 `wrenValueToNum` 通过 union 位重解释，编译器通常对这种 union-punning 生成零条额外指令；
- FPU 寄存器直接接收该 bit pattern，无需拆箱分支或类型 tag 检查。

而非数字类型才需要位测试分支（`IS_NUM`/`IS_OBJ` 是单次 AND + CMP）。

### 1.6 struct 表示的权衡

当 `WREN_NAN_TAGGING=0`，Value 成为一个 `struct { ValueType type; union { double num; Obj* obj; } as; }`（[L127-L145](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L127-L145)）：

- **优点**：字段命名清晰（`.type`、`.as.num`、`.as.obj`），在调试器里一眼看出类型和值，不依赖 IEEE 754 行为，可在不支持 quiet NaN 指针位技巧的奇异性架构上编译。
- **缺点**：Value 从 8 字节涨到 16 字节（type 枚举 + union 对齐）；所有数字/对象存取都要经过 `.as.num`/`.as.obj` 的间接；`IS_NUM`/`IS_OBJ` 从位运算变成字段比较；传参、栈操作、`wrenValuesSame`（[L803-L814](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L803-L814)）都需要分支判断。wren_value.h 顶部注释明确表示 tagged 版本"significantly faster and more compact"。

---

## 2. 字节码与分派

**关键文件**：[wren_opcodes.h](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_opcodes.h)、[wren_vm.h#L13-L18](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L13-L18)、[wren_vm.c#L826-L1390](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L826-L1390)

### 2.1 X-Macro 定义 opcode

[wren_opcodes.h](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_opcodes.h) 本身不定义任何宏，只反复调用 `OPCODE(name, stackEffect)`。被多次以不同 `#define OPCODE` 展开：

1. **生成枚举**：在 [wren_vm.h#L15-L17](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L15-L17) `#define OPCODE(name, _) CODE_##name` → 得到 `CODE_CONSTANT, CODE_NULL, ..., CODE_END` 枚举。
2. **生成 computed-goto 跳转表**：在 [wren_vm.c#L892-L896](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L892-L896) `#define OPCODE(name, _) &&code_##name` → 得到 `dispatchTable[]`。
3. **生成栈效应表**：在 [wren_compiler.c#L414-L418](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L414-L418) `#define OPCODE(_, effect) effect` → 得到 `stackEffects[]` 数组。

### 2.2 "栈效应" 的含义

每个 opcode 第二个参数 `stackEffect` 表示指令执行后栈的净变化量：

- `1`：压入 1 个值（如 `CONSTANT`、`LOAD_LOCAL_*`、`NULL`、`TRUE`、`FALSE`、`CLOSURE`）；
- `0`：栈平衡（如 `STORE_LOCAL`、`JUMP`、`CALL_0`、`RETURN`）；
- `-1`：弹出 1 个值（如 `POP`、`CALL_1`、`SUPER_1`、`JUMP_IF`、`CLOSE_UPVALUE`、`STORE_FIELD`）；
- `-2`…`-16`：弹出 N 个（如 `CALL_16`、`SUPER_16`）。

编译器在计算 `compiler->numSlots` 最大值时使用 `stackEffects[]`（[wren_compiler.c#L414](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L414)），以便确定 `ObjFn.maxSlots`。

### 2.3 两套分派机制

在 [wren_vm.c#L890-L918](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L890-L918)：

#### Computed Goto（默认，非 MSVC）

```c
static void* dispatchTable[] = {
  #define OPCODE(name, _) &&code_##name,
  #include "wren_opcodes.h"
};
#define DISPATCH() goto *dispatchTable[instruction = (Code)READ_BYTE()];
#define CASE_CODE(name) code_##name:
```

这是 GCC/Clang 的"labels-as-values"扩展。每个 opcode 处理段以局部 label `code_XXX` 结尾，dispatchTable 把 opcode 枚举值直接映射到 label 地址。`DISPATCH()` 读取一字节、索引跳转表、直接跳转到对应标签——避免了传统 switch 在循环内的一次大范围比较与分支预测惩罚。

#### 传统 Switch（MSVC）

```c
#define INTERPRET_LOOP   loop: switch (instruction = (Code)READ_BYTE())
#define CASE_CODE(name)  case CODE_##name
#define DISPATCH()       goto loop
```

循环末尾 `goto loop` 重新进入 switch。MSVC 不支持 computed goto，所以走这条路。

**取舍**：computed goto 把分支目标放进数组，CPU 分支预测器（BTB）可直接用 opcode 值做间接跳转预测，常见 opcode 会被快速命中。switch 版本每次分派都是一个 switch 语句，编译器通常实现为跳转表（也是数组），但多一层 `goto loop → switch` 的控制流合并，有些优化器做得不如 computed goto 干净。

### 2.4 为什么 opcode 排列顺序影响性能

[wren_opcodes.h#L10-L13](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_opcodes.h#L10-L13) 明确注释："order of instructions here affects the order of the dispatch table... that in turn affects caching which affects overall performance"。原因：

1. **I-cache 局部性**：dispatchTable 相邻 label 的机器码也会在可执行文件里相邻排列，高频 opcode 排在头部会把它们的代码集中到同一 cache line。
2. **BTB 索引**：opcode 的数值就是跳转表索引，值越小分支目标表项越靠前，间接分支预测器的历史表压力越小。
3. 为此 Wren 把最常用的 `LOAD_LOCAL_0..8`、`CONSTANT`、`POP`、`CALL_0..16` 等放在最前，还把 `LOAD_LOCAL_0..8` 专门做成快速短指令（无操作数，栈偏移 = opcode − CODE_LOAD_LOCAL_0，见 [L925-L935](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L925-L935)）。

### 2.5 LOAD_FRAME / STORE_FRAME

[wren_vm.c#L835-L862](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L835-L862)：

```c
register CallFrame* frame;
register Value* stackStart;
register uint8_t* ip;
register ObjFn* fn;

#define STORE_FRAME() frame->ip = ip

#define LOAD_FRAME() do {                                          \
    frame = &fiber->frames[fiber->numFrames - 1];                 \
    stackStart = frame->stackStart;                               \
    ip = frame->ip;                                               \
    fn = frame->closure->fn;                                      \
} while(false)
```

**作用**：解释器循环中这四个变量被声明为 `register`（提示编译器放到寄存器里）。由于每个指令都会读 `ip`/`stackStart`/`fn`，若它们在循环外缓存到寄存器可节省大量内存访问。`frame` 是当前 `CallFrame*`。

**何时同步**：

- **调用发生前**（`METHOD_BLOCK`/`METHOD_FUNCTION_CALL`/`METHOD_PRIMITIVE` 返回 false 即切换 fiber/压栈帧，见 [L1055、L1071、L1082](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1055-L1084)）：先 `STORE_FRAME()` 把本地 `ip` 写回 `frame->ip`，因为压入新帧后当前 frame 位置可能因数组 realloc 而变动（`wrenCallFunction` 可能 realloc `fiber->frames`）；之后 `wrenCallFunction` 更新 `fiber->numFrames` 与 `fiber->stackTop`，最后 `LOAD_FRAME()` 重新从当前 top frame 读出 ip/stackStart/fn 到寄存器。
- **返回发生时**（`CODE_RETURN`，[L1215-L1257](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1215-L1257)）：先弹出 `numFrames--`、处理 fiber 切换或栈收缩，最后直接 `LOAD_FRAME()` 重新装载（因为调用前的 STORE_FRAME 已经把 ip 写回了被调者 frame，这里 pop 完后 caller frame 完好）。
- **RuntimeError/GC**：`RUNTIME_ERROR()` 宏（[L867-L876](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L867-L876)）也先 `STORE_FRAME()` 再调 `runtimeError(vm)`，因为错误会沿 fiber 链 abort、切换 `vm->fiber`，必须确保当前 frame 状态写回内存才能安全换 fiber。GC 通过 `wrenCollectGarbage` → `blackenFiber`（[wren_value.c#L1055-L1085](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1055-L1085)）只遍历 `fiber->frames[0..numFrames-1]` 和 `fiber->stack[0..stackTop)`，因此 STORE_FRAME 把 ip 写回 frame 是 GC 能看到正确指令指针的前提。
- **初始进入循环前**：[L920](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L920) 调用一次 `LOAD_FRAME()`。

---

## 3. 闭包与上值捕获

**关键文件**：[wren_compiler.c#L1532-L1612](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1532-L1612)、[wren_vm.c#L244-L301](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L244-L301)、[wren_vm.c#L1270-L1296](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1270-L1296)、[wren_value.h#L166-L195](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L166-L195)

### 3.1 对象数据结构

`ObjUpvalue`（[wren_value.h#L178-L195](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L178-L195)）：

- `Value* value;` —— 开放态下直接指向 fiber 栈上的 Value 槽；关闭后改为指向自身的 `closed` 字段。
- `Value closed;` —— 关闭后，值从栈 hoist 到这里。
- `struct sObjUpvalue* next;` —— fiber 开放上值链表节点。

`ObjFn`（[wren_value.h#L247-L268](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L247-L268)）：`int numUpvalues;` 记录本函数要捕获几个上值。

`ObjClosure`（[wren_value.h#L272-L281](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L272-L281)）：`ObjFn* fn; ObjUpvalue* upvalues[FLEXIBLE_ARRAY];`，数组长度由 `fn->numUpvalues` 决定。

编译器侧 `CompilerUpvalue`（[wren_compiler.c#L227-L235](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L227-L235)）：`bool isLocal; int index;`，记录此上值来源于直接外层的 local slot（isLocal=true）还是外层函数的某个 upvalue（isLocal=false，用于扁平化解套）。编译器每个 Compiler 有 `CompilerUpvalue upvalues[MAX_UPVALUES]`（[wren_compiler.c#L336](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L336)）。

### 3.2 编译期解析：resolveLocal → findUpvalue 递归上溯

当内层函数引用一个变量名时：

1. `resolveNonmodule` 先在当前 compiler 的 locals 中查找；
2. 找不到则调用 `findUpvalue(compiler, name, length)`（[wren_compiler.c#L1577-L1612](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1577-L1612)）：
   - **终止条件**：`parent == NULL`（顶层，不是 upvalue，应是模块变量）或遇到方法边界（`parent->enclosingClass != NULL && name[0] != '_'`，方法不闭包局部变量，方法内定义的函数只能引用字段/self）。
   - 在 `compiler->parent` 的 locals 中用 `resolveLocal` 逆序查找。若找到 → 标记那个 local 的 `isUpvalue = true`（L1592）→ 调用 `addUpvalue(compiler, true, local)` 返回索引。
   - 否则**递归**调用 `findUpvalue(compiler->parent, ...)` 在祖父层级继续找；若在上一层的 upvalue 列表中命中，则 `addUpvalue(compiler, false, upvalueIndex)`——这一步实现**扁平化（flattening）**：中间层 compiler 自动添加一个转发上值，内层闭包通过"上值的上值"链条访问，运行期所有 upvalue 都只是一层指针间接。

3. `addUpvalue`（[wren_compiler.c#L1551-L1563](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1551-L1563)）做去重：遍历已有 upvalues，若 `isLocal && index` 都相同直接返回现有索引；否则追加。这保证"两个内层闭包引用同一个外层局部变量"时它们共享同一个 ObjUpvalue。

### 3.3 编译期 emit：CODE_CLOSURE 后跟 isLocal/index 对

在结束函数编译（[wren_compiler.c#L1678-L1697](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1678-L1697)）时：

```c
emitShortArg(compiler->parent, CODE_CLOSURE, constant);  // 常量池索引指向内层 ObjFn
for (int i = 0; i < compiler->fn->numUpvalues; i++) {
    emitByte(compiler->parent, compiler->upvalues[i].isLocal ? 1 : 0);
    emitByte(compiler->parent, compiler->upvalues[i].index);
}
```

每个 upvalue 占 2 字节操作数：`isLocal`（0/1）+ `index`。

### 3.4 运行期：CODE_CLOSURE 处理

[wren_vm.c#L1270-L1296](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1270-L1296)：

1. 从常量池取出预编译好的 `ObjFn* function`。
2. `wrenNewClosure(vm, function)` 分配 `ObjClosure`（upvalues[] 初始为 NULL，避免 GC 误指），压栈（让 GC 看得到）。
3. 对每个 upvalue 读两字节操作数：
   - `isLocal=1`：`captureUpvalue(vm, fiber, frame->stackStart + index)`，在当前 frame 的栈上捕获一个局部。
   - `isLocal=0`：直接复用当前 frame 闭包的 upvalue：`closure->upvalues[i] = frame->closure->upvalues[index]`。这对应扁平化的中间层——复用同一 ObjUpvalue 指针，形成共享。

### 3.5 captureUpvalue 保持链表有序与去重

[wren_vm.c#L244-L283](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L244-L283)：

- 链表 `fiber->openUpvalues` 头是**最靠近栈顶的**开放上值，越往 next 越往栈底走（比较 `upvalue->value > local`）。
- 遍历链表，若找到 `upvalue->value == local`，直接返回现有 upvalue（去重）。
- 否则在合适位置插入新建的 `ObjUpvalue`，保持链表按栈地址降序。

有序是正确性的关键——后续 closeUpvalues 只从头部 pop，详见 3.6。

### 3.6 开放 vs 关闭上值、closeUpvalues

- **开放上值**：局部变量仍在执行栈上，upvalue.value 指向那个栈槽。此时读写上值直接穿透到栈上的真实局部变量，保证多个闭包看到同一变量的最新值。
- **关闭上值**：当被捕获的局部变量退出作用域（离开声明它的 block 或函数返回），它对应的栈槽即将被复用。必须把值从栈复制到 ObjUpvalue 自身的 `.closed` 字段，并把 `.value` 指向 `&upvalue->closed`。

`closeUpvalues(fiber, last)`（[wren_vm.c#L287-L301](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L287-L301)）：

```c
while (fiber->openUpvalues != NULL &&
       fiber->openUpvalues->value >= last) {
    ObjUpvalue* upvalue = fiber->openUpvalues;
    upvalue->closed = *upvalue->value;   // hoist：拷贝栈值到 closed
    upvalue->value = &upvalue->closed;   // 重指向
    fiber->openUpvalues = upvalue->next; // 从链表摘除
}
```

因为链表从栈顶到栈底有序，而 `last` 是"当前正在退出作用域的最浅栈槽"，所以只要 head 满足 `>= last` 就应该被关闭并摘链，循环直到 head 指向更深的栈位置。

触发 closeUpvalues 的三种时机：

1. **CODE_CLOSE_UPVALUE**（[L1209-L1213](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1209-L1213)）：block 内被标记为 isUpvalue 的 local 出作用域时由编译器显式 emit（见 [wren_compiler.c#L1503-L1506](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1503-L1506)），关闭 `stackTop-1` 对应槽然后 DROP。
2. **CODE_RETURN**（[L1221](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1221)）：函数返回时 `closeUpvalues(fiber, stackStart)`，把该 frame 所有栈槽对应的开放上值一并关闭。
3. **循环 break 跳出**：[wren_compiler.c#L1499-L1510](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1499-L1510)，同样为标记 isUpvalue 的 local emit CODE_CLOSE_UPVALUE。

---

## 4. 方法分派

**关键文件**：[wren_value.h#L357-L416](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L357-L416)、[wren_vm.c#L982-L1091](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L982-L1091)、[wren_vm.h#L112-L113](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L112-L113)、[wren_value.c#L120-L132](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L120-L132)

### 4.1 按符号索引的方法表

全局有**唯一的符号表** `vm->methodNames`（[wren_vm.h#L113](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L113)，类型 `SymbolTable`，本质是 `StringBuffer`，见 [wren_utils.h#L71](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_utils.h#L71)）。每遇到一个新方法签名（如 `"call()"`、`"toString"`、`"+(_)"`、`"[_]"`），就 `wrenSymbolTableEnsure` 赋一个全局唯一的整数索引。

每个类的 `ObjClass.methods` 是一个 `MethodBuffer`（[wren_value.h#L409](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L409)），它不是哈希表而是**以符号索引为下标的直接数组**。`wrenBindMethod`（[wren_value.c#L120-L132](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L120-L132)）：

```c
if (symbol >= classObj->methods.count) {
    Method noMethod; noMethod.type = METHOD_NONE;
    wrenMethodBufferFill(vm, &classObj->methods, noMethod,
                         symbol - classObj->methods.count + 1);
}
classObj->methods.data[symbol] = method;
```

注释称其为"永不冲突但低装载因子的哈希表"（[wren_value.h#L402-L408](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L402-L408)）。

- **时空权衡**：省去哈希计算、探测链、冲突处理，方法调用是一次整数索引 + 边界检查，极快。代价是若符号索引稀疏（例如一个类只实现了索引很大的符号），中间的 METHOD_NONE 槽位浪费内存。但 Method 只有 `{type enum + union of pointer}` 大小（8–16 字节），注释认为"worthwhile trade-off"。
- 新类通过 `wrenBindSuperclass` 继承时会把超类 methods 复制过来（保证每个类自己的 methods 完整），继承链查找被**编译时静态解析**（SUPER 指令直接引用超类常量表），运行时无沿继承链遍历。

### 4.2 CALL_n 与 SUPER_n

两类指令家族（[wren_opcodes.h#L82-L118](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_opcodes.h#L82-L118)）都带 2 字节符号操作数，区别在于获取 classObj 的方式：

- **CALL_n**（[L982-L1006](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L982-L1006)）：`numArgs = instruction - CODE_CALL_0 + 1;`（+1 含 receiver），从栈顶倒数 numArgs 个槽取出 receiver `args[0]`，调用 `wrenGetClassInline(vm, args[0])`（[wren_vm.h#L209-L237](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L209-L237)）动态获取 receiver 的类对象。
- **SUPER_n**（[L1008-L1034](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1008-L1034)）：同样读 numArgs 和 symbol，额外再读 2 字节常量池索引得到**编译时确定的超类** `classObj = AS_CLASS(fn->constants.data[READ_SHORT()])`，跳过动态类查找；receiver 仍来自栈。

两条路径最后都 `goto completeCall;` 共享同一后半段（L1036-L1091），这样避免在 fast path 里加 if 分支。

### 4.3 四种 MethodType 调用路径

在 completeCall 先做边界检查（`symbol >= methods.count || method.type == METHOD_NONE`），若未绑定调 `methodNotFound` → runtimeError。否则按 type 分派：

| MethodType               | 代码位置                                                                     | 调用路径                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **METHOD_PRIMITIVE**     | [L1047-L1063](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1047-L1063) | 直接 `method->as.primitive(vm, args)`（`bool (*Primitive)(WrenVM*, Value* args)`，见 [wren_value.h#L203](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L203)）。primitive 返回 true：返回值已写在 `args[0]`，收缩栈 `fiber->stackTop -= numArgs - 1`。返回 false：发生 fiber 切换、错误或栈帧变化，STORE_FRAME + 可能 RUNTIME_ERROR + LOAD_FRAME。Primitives 直接操作 C 栈指针 args，最快。 |
| **METHOD_FUNCTION_CALL** | [L1065-L1074](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1065-L1074) | 专用于 Fn.call 的 trampoline：先 `checkArity` 验证参数个数，然后同样调 primitive（就是 `fn_callN` 系列，见 wren_core.c#L270-L275 → `call_fn`）。checkArity 检查闭包 arity（[L809-L821](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L809-L821)）。                                                                                                                                            |
| **METHOD_FOREIGN**       | [L1076-L1079](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1076-L1079) | `callForeign(vm, fiber, foreign, numArgs)`（[L384-L397](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L384-L397)），将 `vm->apiStack` 指向 args 起始，调用宿主绑定的 C 函数 `WrenForeignMethodFn`，返回后重置 apiStack。                                                                                                                                                                       |
| **METHOD_BLOCK**         | [L1081-L1085](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1081-L1085) | 用户定义的 Wren 方法：STORE_FRAME → `wrenCallFunction(vm, fiber, closure, numArgs)` 压入新 CallFrame → LOAD_FRAME，下一轮 DISPATCH 就开始在被调方法里执行。                                                                                                                                                                                                                                        |

### 4.4 methodNotFound

[wren_vm.c#L438-L442](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L438-L442)：将 `vm->fiber->error` 设为格式化字符串 `"<className> does not implement '<method>'."`。随后 completeCall 里的 `RUNTIME_ERROR()` 宏把错误沿 fiber 链传播（见 §5.3）。

---

## 5. Fiber 协程与错误传播

**关键文件**：[wren_value.h#L298-L355](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L298-L355)、[wren_vm.c#L403-L434](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L403-L434)、[wren_vm.c#L1215-L1257](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1215-L1257)、[wren_core.c#L86-L249](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L86-L249)

### 5.1 Fiber 数据结构

`ObjFiber`（[wren_value.h#L316-L355](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L316-L355)）拥有：

- 独立的 `Value* stack` + `stackTop` + `stackCapacity`；
- 独立的 `CallFrame* frames` + `numFrames` + `frameCapacity`；
- 自己的 `openUpvalues` 链表（开放上值依附于 fiber 栈）；
- `caller` 指针（恢复时回到哪个 fiber）；
- `error`（若运行期错误则持有错误字符串，否则 NULL_VAL）；
- `state`（见下文）。

创建 `wrenNewFiber`（[wren_value.c#L149-L188](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L149-L188)）默认 `state = FIBER_OTHER`，栈与 frames 数组各自分配；初始 CallFrame 指向传入 closure。

### 5.2 FiberState 三态

[wren_value.h#L300-L314](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L300-L314)：

| 状态          | 设置者                                                                                                                                                  | 含义                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FIBER_ROOT`  | `runInterpreter`（[L830](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L830)）                                                                      | 由 `wrenInterpret/wrenCall` 直接启动的初始 fiber，禁止被 Fiber.call 再入（[wren_core.c#L103](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L103)）。 |
| `FIBER_TRY`   | `fiber_try/fiber_try1` primitives（[wren_core.c#L196、L205](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L196-L205)）                            | 该 fiber 由 `Fiber.try(fn)` 运行，错误会被捕获。                                                                                                           |
| `FIBER_OTHER` | `wrenNewFiber` 默认值；`fiber_yield/fiber_yield1` 让出后重置（[wren_core.c#L216、L234](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L216-L234)） | 普通 call/transfer 启动或 yield 之后的挂起态。                                                                                                             |

### 5.3 runFiber：call/try/transfer 的公共引擎

[wren_core.c#L86-L138](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L86-L138)：

- 校验：已 abort（error 非 null）、已完成（numFrames==0）均报错；call 模式下不允许 caller 非 NULL 或 FIBER_ROOT。
- **isCall = true**（call/try）：`fiber->caller = vm->fiber`，返回时沿着 caller 恢复；
- **isCall = false**（transfer）：不建立 caller 链，直接切换 vm->fiber（类似单向跳转，不回来）；
- 若 fiber 首次启动（numFrames==1 且 ip 指向 code 起始）：把参数写入 `stackTop[0]`；否则在栈顶 `[-1]` 槽放参数值（yield/transfer 的返回值）；
- 最后 `vm->fiber = fiber; return false;` 返回 false 让 METHOD_PRIMITIVE 路径触发 fiber 切换（注意 primitive 返回 false 意味着"帧/栈已变，请重新加载"），解释器接着就在新 fiber 上跑。

各 primitive 的差异：

- `fiber_call(1)`：`runFiber(..., isCall=true, hasValue=false/true, "call")`。
- `fiber_try(1)`：同 call，但之后 `vm->fiber->state = FIBER_TRY`。
- `fiber_transfer(1)`：`isCall=false`。
- `fiber_yield(1)`（[L209-L249](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L209-L249)）：反向切换 `vm->fiber = current->caller`，清空 caller、state=FIBER_OTHER，把返回值（null 或参数）写回 caller 的 `stackTop[-1]`。
- `fiber_suspend`（[L166-L172](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_core.c#L166-L172)）：vm->fiber=NULL 让解释器退出。

### 5.4 CODE_RETURN 过程

[wren_vm.c#L1215-L1257](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1215-L1257)：

1. `result = POP();` 弹出返回值。
2. `fiber->numFrames--;` 弹出调用帧。
3. `closeUpvalues(fiber, stackStart);` 把该 frame 所有栈槽上的开放上值关闭（栈即将被复用）。
4. 若 `numFrames == 0`（最外层 frame 完成）：
   - 如果 `fiber->caller == NULL`：是入口 fiber，把结果写到 `fiber->stack[0]`，`stackTop = stack + 1`，返回 `WREN_RESULT_SUCCESS` 退出解释器。
   - 否则（fiber 完成但有 caller）：`resumingFiber = fiber->caller; fiber->caller = NULL; fiber = resumingFiber; vm->fiber = resumingFiber;` 把结果写到 caller 的 `stackTop[-1]`（即原 Fiber.call/try 那栈位置）。
5. 若还有帧（普通方法 return）：`stackStart[0] = result; fiber->stackTop = frame->stackStart + 1;` 把返回值写到 caller 的 receiver 槽位（args[0] 就是返回值位置），收缩栈顶。
6. 统一 `LOAD_FRAME(); DISPATCH();` 继续执行 caller。

### 5.5 runtimeError：沿 caller 链逐层 abort

[wren_vm.c#L403-L434](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L403-L434)：

```c
ObjFiber* current = vm->fiber;
Value error = current->error;
while (current != NULL) {
    current->error = error;
    if (current->state == FIBER_TRY) {
        current->caller->stackTop[-1] = vm->fiber->error;
        vm->fiber = current->caller;
        return;
    }
    ObjFiber* caller = current->caller;
    current->caller = NULL;   // 断开，永远不会 resume
    current = caller;
}
wrenDebugPrintStackTrace(vm);
vm->fiber = NULL;
vm->apiStack = NULL;
```

- 从当前出错 fiber 开始向 caller 方向遍历；
- 每一级 fiber 都被标记同一个 error；
- 如果遇到 `state == FIBER_TRY` 的 fiber（即 Fiber.try()），**把错误对象写入其 caller 栈顶槽位**作为 try 调用的返回值，切换 vm->fiber 到 caller 并 return——try 捕获了错误，用户层以返回值形式接收错误；
- 否则：把该 fiber 的 caller 指针清空（它被 abort，不可再调用），继续向上；
- 最后到 NULL（无人 try），打印栈轨迹并 `vm->fiber = NULL` 终止解释器。`RUNTIME_ERROR()` 宏检测到 fiber==NULL 就返回 `WREN_RESULT_RUNTIME_ERROR`。

**FIBER_TRY 与其它状态的区别**：只有 state==FIBER_TRY 的 fiber 才会把错误转换成返回值并恢复 caller；FIBER_OTHER 与 FIBER_ROOT 在遍历中被"断开+继续上抛"，错误不会被吞掉。注意错误检查点是**被中断的 fiber**（current）的 state，而不是其 caller；也就是说 `Fiber.try` 启动的子 fiber 若出错，try 设置的是子 fiber 的 state=FIBER_TRY，runtimeError 遍历到子 fiber 时识别到 try 标记，从而把错误回到 try 的调用者。

---

## 6. 垃圾回收（三色标记清除）

**关键文件**：[wren_value.h#L107-L119](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L107-L119)、[wren_vm.c#L125-L237](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L125-L237)、[wren_value.c#L976-L1227](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L976-L1227)

### 6.1 三色标记与 isDark

三色语义通过 `Obj.isDark` 布尔字段（[wren_value.h#L112](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L112)）实现：

- **白色**：`isDark == false`，尚未被扫描器访问到（或初始状态）——不可达；
- **灰色**：`isDark == true`，且**对象指针位于 `vm->gray[]` 栈中**——自身已标记但子引用尚未扫描；
- **黑色**：`isDark == true`，且已从 gray 栈弹出、blackenObject 处理完——自身与子引用均标记完成。

### 6.2 根集合

[wren_vm.c#L146-L169](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L146-L169)：

1. `vm->modules`（所有已加载模块的 Map，[wren_vm.h#L51](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L51)）；
2. **临时根栈** `vm->tempRoots[0..numTempRoots-1]`（`wrenPushRoot`/`wrenPopRoot` 操作，最大 8 个，见 [wren_vm.c#L1615-L1627](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1615-L1627)），保护刚创建还没挂接到任何地方的临时对象；
3. **当前 fiber** `vm->fiber`（其 stack/frames/openUpvalues/caller 全由 blackenFiber 递归遍历）；
4. **句柄链表** `vm->handles`（`WrenHandle` 双向链表，每个 handle->value 都被 gray）；
5. **编译器**（若编译中）：`wrenMarkCompiler` 标记编译时使用的对象；
6. **全局方法符号表** `vm->methodNames`（由 `wrenBlackenSymbolTable` 标记所有方法名 ObjString）。

### 6.3 标记传播

- **wrenGrayObj**（[wren_value.c#L976-L997](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L976-L997)）：NULL 直接返回；已 dark 返回防环；否则置 isDark=true 并压入 `vm->gray[]`（自动扩容）。
- **wrenGrayValue**（[L999-L1003](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L999-L1003)）：只对 IS_OBJ 值 gray，unboxed 值不需要追踪。
- **wrenGrayBuffer**（[L1005-L1011](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1005-L1011)）：遍历 ValueBuffer。
- **wrenBlackenObjects**（[L1219-L1227](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1219-L1227)）：while 循环从 gray 栈 pop 对象，调 `blackenObject(vm, obj)`，后者按 obj->type switch 到专用 blackenXxx。
- 各 **blackenXxx** 负责：
  - 调用 wrenGrayObj/wrenGrayValue 把引用到的子对象标灰；
  - 把自身内存大小累加到 `vm->bytesAllocated`（用于下一次 nextGC 计算）。
  - 关键点：
    - `blackenFiber`（[L1055-L1085](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1055-L1085)）遍历 frames[0..numFrames-1] 把每个 closure 标灰，遍历 stack[0..stackTop) 每个值标灰，遍历 openUpvalues 链表标灰，还标 caller 和 error。因此它依赖 §2.5 的 STORE_FRAME 在 GC 前把 ip 写回——这里不用 ip 只遍历 frames/stackTop，所以只要 frames 数组有效即可。
    - `blackenClosure`（[L1039-L1053](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1039-L1053)）：标灰 fn + 所有 upvalue[]。
    - `blackenUpvalue`（[L1184-L1191](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1184-L1191)）：只 gray `upvalue->closed`，**不** gray `*upvalue->value`——因为开放态下 value 指向 fiber 栈，栈的值已由 blackenFiber 标灰；关闭态下 value == &closed，gray(closed) 已经够了。
    - `blackenClass`（[L1013-L1037](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L1013-L1037)）：标 metaclass/superclass/name/attributes，并只对 METHOD_BLOCK 的 closure 标灰（primitive/foreign 是函数指针，不需要 GC）。

### 6.4 清除阶段

[wren_vm.c#L176-L193](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L176-L193)：原地遍历 `vm->first` 单链表（所有对象，见 Obj.next [wren_value.h#L118](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.h#L118)）：

- isDark == false：从链表摘除 → `wrenFreeObj`；
- isDark == true：重置为 false（为下次 GC 准备），前进。

之后 `vm->nextGC = bytesAllocated + bytesAllocated * heapGrowthPercent/100`（[L197](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L197)），设下一次触发阈值。

### 6.5 分配器触发 GC 与防重入

`wrenReallocate`（[wren_vm.c#L213-L237](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L213-L237)）：

```c
vm->bytesAllocated += newSize - oldSize;
#if WREN_DEBUG_GC_STRESS
  if (newSize > 0) wrenCollectGarbage(vm);
#else
  if (newSize > 0 && vm->bytesAllocated > vm->nextGC) wrenCollectGarbage(vm);
#endif
return vm->config.reallocateFn(memory, newSize, vm->config.userData);
```

- **触发条件**：`newSize > 0`（即不是 free 操作——因为 free 时可能正在 sweep 阶段里调用 wrenFreeObj → wrenReallocate(0)，重入会炸）且当前分配量超过 nextGC。
- **防重入**：GC 过程中会调用 wrenReallocate 来扩容 gray 栈（[wren_value.c#L988-L994](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L988-L994)）或释放对象，但这些调用 `newSize` 可能 >0 且 bytesAllocated 仍高于 nextGC？——实际上标记阶段一开始把 `vm->bytesAllocated = 0`（[wren_vm.c#L144](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L144)），后续 blackenXxx 函数重新累计存活对象字节数，所以在 GC 过程中再次进入 wrenReallocate 时 bytesAllocated 远小于 nextGC（nextGC 仍是旧的大阈值，要等 sweep 结束后才重算），不会递归触发。sweep 阶段调的 wrenFreeObj 都是 newSize=0，`if (newSize > 0)` 过滤掉。
- GC stress 模式：每次分配都强制 GC，用于调试验证 root 覆盖。

---

## 7. 综合与不变量

### 7.1 端到端链路示例

以下面 Wren 片段为例串起完整生命周期：

```wren
var make = Fn.new {
  var x = 10            // 外层局部
  return Fn.new { x = x + 1 }   // 内层闭包捕获 x
}
var f = make.call()
f.call()                // 11
f.call()                // 12
```

**编译期**（wren_compiler.c）：

1. 编译外层函数体时，x 成为外层 compiler 的 local slot 0。
2. 编译内层 `Fn.new { x = x + 1 }`：递归下降时创建新 compiler。内层引用 `x`，`resolveLocal` 在内层 locals 找不到，调 `findUpvalue(innerCompiler, "x", 1)`（[wren_compiler.c#L1577-L1612](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1577-L1612)）。
3. 检查 `innerCompiler->parent`（即外层 compiler）的 locals：找到 x 在 slot 0 → 标记外层 `locals[0].isUpvalue = true`；调 `addUpvalue(innerCompiler, isLocal=true, index=0)` 返回 0。内层 compiler.fn.numUpvalues = 1。
4. 内层函数完成（[wren_compiler.c#L1678-L1697](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1678-L1697)）：把 `ObjFn*` 放入外层常量池，emit `CODE_CLOSURE <constIdx>`，再 emit 2 字节：`1 0`（isLocal=1, index=0）。
5. 外层 x 在 block 结束或函数返回时，因 `isUpvalue == true`，编译器 emit `CODE_CLOSE_UPVALUE`（而不是普通 POP），见 [wren_compiler.c#L1503-L1506](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_compiler.c#L1503-L1506)。

**运行期**（wren_vm.c）：

1. 外层函数执行 `CODE_CLOSURE`（[L1270-L1296](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1270-L1296)）：
   - 分配 ObjClosure，fn=内层 ObjFn，压栈保护；
   - 读操作数 isLocal=1，index=0 → `captureUpvalue(vm, fiber, outerFrame->stackStart + 0)`；
   - captureUpvalue 发现 fiber.openUpvalues 里没有指向该槽的 upvalue → 新建 ObjUpvalue(.value = &x 槽地址, .closed=NULL_VAL)，按栈序插入链表。closure.upvalues[0] 指向它；
   - closure 留在栈顶成为外层函数的返回值。
2. 外层返回 CODE_RETURN（[L1215-L1221](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1215-L1221)）：`closeUpvalues(fiber, stackStart)`，x 在 stackStart 槽位，满足 `upvalue->value >= last`：
   - `upvalue->closed = *upvalue->value`（拷贝 10）；
   - `upvalue->value = &upvalue->closed`；
   - 从 openUpvalues 摘链。
   - 至此即使外层 frame 栈槽被后续调用复用，`f.upvalues[0]->value` 仍稳定指向 ObjUpvalue 自己的 closed 字段。
3. `f.call()` 时：METHOD_BLOCK 路径压入新 frame，执行内层函数。LOAD_UPVALUE 0 读 `*upvalues[0]->value` 得到 10；STORE_UPVALUE 0 写回 `*upvalues[0]->value = 11`——此时 upvalue 已是 closed 状态，写回的是 ObjUpvalue.closed 字段。
4. 第二次 call() 读到 11，写为 12——所有 f 闭包实例共享同一 ObjUpvalue，所以状态持续。

**GC 路径**（wren_value.c）：

- GC 启动时，`wrenCollectGarbage` 把当前 fiber、modules、handles 等标灰；
- 若 f 在栈上或模块变量中，`blackenFiber`/`blackenModule` 把 f（ObjClosure）灰化；
- `blackenClosure` 灰化 fn 和 upvalues[0]；
- `blackenUpvalue` 调 `wrenGrayValue(vm, upvalue->closed)` 把值 11/12（unboxed num，IS_NUM 直接跳过）标记；
- 于是 f、内层 fn、ObjUpvalue 都标记存活，不会被 sweep 回收；
- 外层函数返回后外层 ObjFn 若无人引用会被回收（它的 code/constants 也一起 free），但 ObjUpvalue 和 ObjClosure 仍通过 f 链可达。

### 7.2 最微妙的关键不变量

#### 不变量 1：`fiber->openUpvalues` 链表必须按栈地址降序（栈顶到栈底）有序

**代码依据**：[wren_vm.c#L258-L282](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L258-L282) 中 `captureUpvalue` 的插入循环 `while (upvalue != NULL && upvalue->value > local)` 在正确位置插入；[L289-L299](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L289-L299) 的 `closeUpvalues` 依赖这个顺序，用 `while (head != NULL && head->value >= last)` 从 head 连续弹出所有需要关闭的上值——如果链表乱序，closeUpvalues 要么漏掉高地址的开放上值（返回后出现悬空指针）要么误关闭仍在作用域内的局部（破坏其他闭包）。

同一栈槽去重由 [L265](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L265) 的 `upvalue->value == local` 保证，这是"多个闭包捕获同一变量共享单一 upvalue"的根本。

#### 不变量 2：STORE_FRAME 必须在任何会导致 frame/stack 重定位或 GC/纤维切换的操作之前调用

**代码依据**：runInterpreter 循环把 ip/frame/stackStart/fn 都放在 register 局部变量，`fiber->frames` 数组可能在 `wrenCallFunction` 中被 realloc（[wren_vm.h#L182-L187](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L182-L187)），fiber 本身可能被 runtimeError/primitive 切换。因此所有可能触发这些副作用的代码路径——METHOD_BLOCK 调用（[L1082](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1082)）、METHOD_FUNCTION_CALL（[L1071](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1071)）、METHOD_PRIMITIVE 返回 false 后（[L1055](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1055)）、RUNTIME_ERROR 宏（[L870](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L870)）——都严格遵守"先 STORE_FRAME()，再执行操作，再 LOAD_FRAME()"的顺序。违反该不变量会导致：realloc 后旧 frame 指针成为悬空指针、GC 扫描 fiber.frames 看到过时的 ip、fiber 切换后 register 变量仍指向旧 fiber 的数据。

特别注意 GC：`wrenCollectGarbage` 在 wrenReallocate 中几乎在任何分配点都可能触发（[wren_vm.c#L233](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L233)），所以 VM 在创建新对象（如 CODE_CLOSURE 中 wrenNewClosure，[L1275](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1275)）之前必须保证当前 frame 的 ip 已经写回——这里因为刚进入 CODE_CLOSURE 的 handler 还没做过调用，ip 是紧跟在 READ_SHORT 后更新的，与 frame->ip 同步（因为 DISPATCH 时 READ_BYTE 已经推进 ip 但不写回——但 DISPATCH 读到 CLOSURE 后 ip 还没写回，不过此时如果 GC 触发，wrenBlackenObjects 遍历 fiber.frames，frame->ip 是上一次 STORE_FRAME 的值，而局部 ip 可能超前。这正是为什么 wrenNewClosure 内部若触发 GC，扫描的是内存中 frame->ip——它可能落后但不会超前到未初始化位置，所以即使 GC 发生也不会把活跃对象当垃圾回收）。

#### 不变量 3：方法符号表的全局一致性与 methods 数组长度

**代码依据**：

- `vm->methodNames` 是 VM 生命周期内唯一的 SymbolTable（[wren_vm.h#L113](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.h#L113)），所有类共享同一套符号索引。当 wrenBindMethod（[wren_value.c#L120-L132](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_value.c#L120-L132)）把一个 Method 写入 classObj.methods.data[symbol] 时，它只在 buffer 不够长时用 METHOD_NONE 填充到 symbol+1。
- completeCall（[wren_vm.c#L1038-L1043](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_vm.c#L1038-L1043)）做检查 `symbol >= classObj->methods.count || method.type == METHOD_NONE` 即"未绑定"，这隐含：所有类对于已经在全局 methodNames 中存在的符号，要么在 methods 数组中有对应槽位（哪怕是 METHOD_NONE），要么 symbol 会越界（也是未绑定）。
- 这个不变量允许 CALL 指令把符号索引当直接下标使用，O(1) 完成分派；若索引和类方法表的对应关系被破坏（例如错误地重排 methodNames），会导致错误方法被调用或越界访问。wrenSymbolTableEnsure（[wren_utils.c](file:///e:/gsb/608/gsb_12/Autumn/src/vm/wren_utils.c)）只追加不插入，因此已有符号的索引永不改变，从而保持该不变量。

---

（分析结束。）
