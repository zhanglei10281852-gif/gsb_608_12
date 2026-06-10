# Wren VM 内核静态分析

> 静态阅读 `src/vm/` 下源码后的逐题解读。所有结论锚定到具体文件 / 函数 / 宏 / 字段。

---

## 0. 必备示意图

### 图 A：NaN-Tagging 的 64 位布局

参考 [wren_value.h](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L494-L594)。

```
bit 63                                                            bit 0
┌──┬──────────────┬─┬─────────────────────────────────────────────────┐
│S │ Exponent(11) │Q│             Mantissa(51)                        │
│  │  1 1 1 1 1 1 │ │                                                 │
│  │  1 1 1 1 1 1 │1│                                                 │
└──┴──────────────┴─┴─────────────────────────────────────────────────┘
                  ▲
                  └─ Quiet-NaN bit（QNAN 含此位置 1）

布局规则（WREN_NAN_TAGGING 开启时）：
• 任何不满足 "exponent 全 1 且 QNAN 位为 1" 的位串 → 是普通 double（IS_NUM 为真）。
• "exponent 全 1 + QNAN 位 = 1，且 SIGN_BIT = 0" → singleton：低 3 位 MASK_TAG 区分
      000=NAN  001=NULL  010=FALSE  011=TRUE  100=UNDEFINED  101..111=未用
• "exponent 全 1 + QNAN 位 = 1，且 SIGN_BIT = 1"     → 堆指针：低 51 位放 Obj*（48 位指针够用）。

      S    Exp + QNAN                Mantissa(51)
NUM   ?    not all-ones     │ <-- 数值位 -->                  │
NULL  0    QNAN bits (set)  │ 0...0 001                      │
FALSE 0    QNAN bits (set)  │ 0...0 010                      │
TRUE  0    QNAN bits (set)  │ 0...0 011                      │
UNDEF 0    QNAN bits (set)  │ 0...0 100                      │
OBJ*  1    QNAN bits (set)  │      Obj* low 51 bits          │
```

### 图 B：闭包上值生命周期

```
   [outer fn 栈帧]                       open   ──▶ ObjUpvalue {value→stack, closed=NULL_VAL}
   ┌────────────┐         捕获时          (在 fiber->openUpvalues 链上，栈高→栈低有序)
   │ outer.x ◀──┼──── ObjUpvalue.value
   │  ...       │                                         │
   └────────────┘         作用域离开 / CODE_CLOSE_UPVALUE  │
                                                          ▼
                                       closed ──▶ ObjUpvalue { closed = (从栈复制),
                                                               value  = &self.closed }
                                              脱离 fiber->openUpvalues，
                                              内层 ObjClosure.upvalues[i] 仍持有它，
                                              直到 GC 不再可达。
```

---

## 1. 值表示与 NaN tagging（[wren_value.h](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h)）

`Value` 在开启 `WREN_NAN_TAGGING` 时就是一个 `uint64_t`（[行 121-123](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L121-L123)）。利用 IEEE-754 的 quiet-NaN 编码空间编码"非数字"信息：

- **`SIGN_BIT = 1<<63`**（[行 553](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L553)）：用于区分 singleton（0）与堆指针（1）。
- **`QNAN = 0x7ffc000000000000`**（[行 556](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L556)）：指数位全 1 + 高位 mantissa 1，标识 quiet NaN，是"非数字"的判别签名。
- **`MASK_TAG = 7`**（[行 569](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L569)）和 `TAG_*` 0..7（[行 572-579](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L572-L579)）：在 singleton 编码中用尾部 3 bit 枚举 `NAN/NULL/FALSE/TRUE/UNDEFINED`。

判定与还原宏：
- `IS_NUM(v)  ((v & QNAN) != QNAN)`（[行 559](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L559)）：只要 QNAN 位组中有一位不是 1，就是真正的 double。
- `IS_OBJ(v)  ((v & (QNAN|SIGN_BIT)) == (QNAN|SIGN_BIT))`（[行 562](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L562)）：QNAN 全置 + 符号位 1。
- `AS_OBJ(v)  ((Obj*)(uintptr_t)(v & ~(SIGN_BIT|QNAN)))`（[行 585](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L585)）：把 QNAN 与符号位掩掉，剩下 51 bit 直接还原指针（在 64 位机上指针只用 48 位）。
- `AS_BOOL(v) (v == TRUE_VAL)`（[行 582](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L582)）：直接位串相等，不必读 union。
- 数→Value 与 Value→数走 `wrenDoubleToBits/FromBits`（[wren_math.h](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_math.h#L20-L32)）的 `union` 重解释；这意味着普通 double 在 `Value` 中**位级别原样存放**——这是"数字算术零开销"的根因（[行 549-549 注释](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L546-L549)）：`AS_NUM`/`NUM_VAL` 不做任何屏蔽、不读类型字段，CPU 浮点运算直接处理这些位。

**关闭 NaN tagging 的回退**（[行 127-145](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L127-L145)）：`Value` 退化为 `{ ValueType type; union { double num; Obj* obj; } as; }`。
- 调试更友好：调试器能直接看到 `type` 与 `as`，不必心算 NaN 位；条件断点用 `value.type == VAL_NUM` 即可。
- 可移植性：不依赖目标机器只用 48 位地址、QNAN 编码、以及 union 重解释合法等隐含前提，因此对古老或非 IEEE-754 平台更稳。
- 代价：每个 Value 至少 16 字节（type + 8 字节联合 + 对齐），栈拷贝/参数传递成本翻倍；数字 `AS_NUM` 还要走 union 字段读，编译器有时无法消除。

---

## 2. 字节码与分派（[wren_opcodes.h](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_opcodes.h) + [runInterpreter](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L826-L1390)）

### X-Macro
[wren_opcodes.h](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_opcodes.h) 整个文件只有 `OPCODE(name, stackEffect)` 行。第二个参数是"栈效应"——这条指令执行后栈顶相对偏移。例如 `OPCODE(CALL_3, -3)`（[行 85](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_opcodes.h#L85)）表示弹掉 3 个参数后接收者位置变成结果，净变化 -3。`OPCODE(CONSTANT, 1)`（[行 16](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_opcodes.h#L16)）压一个常量。

外部用三种方式同一份表：
1. [wren_vm.h L13-L18](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.h#L13-L18) 定义 `enum Code { CODE_##name, ... }`。
2. 编译器通过宏推算栈效应来动态维护 `stackSize` 上限。
3. 解释器构造 computed-goto 的 `dispatchTable`（见下）。

### 两套分派
- **Computed goto**（[wren_vm.c L890-L906](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L890-L906)）：定义 `static void* dispatchTable[]` 用 `&&code_NAME` 取标签地址；`DISPATCH()` 直接 `goto *dispatchTable[READ_BYTE()]`。每条指令末尾自己 dispatch，CPU 间接跳转预测器为每个 opcode 单独训练，命中率高。
- **传统 switch**（[L910-L918](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L910-L918)）：`switch (READ_BYTE())`，所有 opcode 共享同一个 `goto loop` 跳转点；编译器会给整张 switch 生成单一间接跳转，预测器分布更弱，但代码可移植到任何标准 C。

`#if WREN_COMPUTED_GOTO` 让选择在编译期决定。

### 为什么 opcode 顺序影响性能
`dispatchTable[]` 与 `switch` 跳表都按 `enum` 顺序生成，这决定了对应的分支表项在指令缓存里的相对位置。频繁相邻执行的 opcode（例如 `LOAD_LOCAL` 后常跟随某个 `CALL_n`、`CONSTANT` 后常跟 `STORE_LOCAL`）若代码片段在物理指令缓存上接近，则取指更连续。文件顶部注释[wren_opcodes.h L10-L13](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_opcodes.h#L10-L13) 明确警告"调换顺序前请跑 benchmark"。

### LOAD_FRAME / STORE_FRAME
[L835-L862](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L835-L862) 把 4 个高频值缓存进局部寄存器变量：
```c
register CallFrame* frame;
register Value*     stackStart;
register uint8_t*   ip;
register ObjFn*     fn;
```
- `LOAD_FRAME()` 从 `fiber->frames[numFrames-1]` 拉取并把 `ip = frame->ip`、`stackStart = frame->stackStart`、`fn = frame->closure->fn`；之后每次 `READ_BYTE()` / `READ_SHORT()` 只动一个寄存器。
- `STORE_FRAME()` 仅在调用前/返回后/RUNTIME_ERROR 前 把 `ip` 写回 `frame->ip`。

调用 `wrenCallFunction`（压栈帧，[wren_vm.h L178-L196](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.h#L178-L196)）、`CODE_RETURN`（弹栈帧，[L1215-L1257](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1215-L1257)）、`RUNTIME_ERROR()` 跨 fiber 切换（[L867-L876](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L867-L876)）时，必须 `STORE_FRAME` 把 `ip` 写回，再 `LOAD_FRAME` 重新取出新栈顶帧的状态。GC 则因为只可能由 `wrenReallocate` 在分配路径触发（[L213-L237](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L213-L237)），而所有可能分配的 opcode（`CLOSURE`、`CONSTRUCT`、`IMPORT_*`、`CLASS`、`METHOD_*`）在分配前都不依赖陈旧的本地缓存：注意 `CODE_CLOSURE`（[L1270-L1296](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1270-L1296)）先 `wrenNewClosure` → 立刻 `PUSH(OBJ_VAL(closure))` 把闭包放回 fiber 栈，使 `blackenFiber`（[wren_value.c L1055-L1085](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L1055-L1085)）能扫到它。

---

## 3. 闭包与上值捕获

### 编译期解析（[wren_compiler.c findUpvalue / addUpvalue](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1548-L1612)）
单遍编译器维护一条 `Compiler` 链：每个嵌套函数有自己的 `Compiler`，`compiler->parent` 指向外层。

`findUpvalue(compiler, name)`（[L1577-L1612](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1577-L1612)）逻辑：
1. 父链为空 → 返回 -1。
2. 越过方法边界（`compiler->parent->enclosingClass != NULL` 且名字不是静态字段 `_xxx`）→ 返回 -1。
3. 在 *直接* 父函数中尝试 `resolveLocal`；找到则把父 local 标 `isUpvalue = true`（[L1592](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1592)），并 `addUpvalue(compiler, isLocal=true, index=local)`。
4. 否则递归 `findUpvalue(compiler->parent, ...)`；若沿链找到，则在当前函数登记一个 `isLocal=false, index=父函数的 upvalue 索引` —— 这就是"扁平化闭包"：每一层中间函数都隐式增加一条 upvalue 槽，把外层的本地变量层层"管道化"传入最深处。

`addUpvalue`（[L1551-L1564](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1551-L1564)）以 `(isLocal, index)` 为键去重，复用已有索引；返回该函数的 `upvalues[]` 序号。

### 字节码：CODE_CLOSURE 后的操作数
编译器结束内层函数时（[L1680-L1696](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1680-L1696)）会向**外层**发射：
```
CODE_CLOSURE  <2 字节常量索引 →ObjFn>
   重复 numUpvalues 次：
     <1 字节 isLocal>  <1 字节 index>
```

### 运行期填充（[wren_vm.c CODE_CLOSURE](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1270-L1296)）
```c
ObjFn* function = AS_FN(fn->constants.data[READ_SHORT()]);
ObjClosure* closure = wrenNewClosure(vm, function);
PUSH(OBJ_VAL(closure));            // 先入栈，免被 GC
for (int i = 0; i < function->numUpvalues; i++) {
  uint8_t isLocal = READ_BYTE();
  uint8_t index   = READ_BYTE();
  if (isLocal)
    closure->upvalues[i] = captureUpvalue(vm, fiber, frame->stackStart + index);
  else
    closure->upvalues[i] = frame->closure->upvalues[index];   // 直接共享外层闭包的 upvalue
}
```

`captureUpvalue`（[L244-L283](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L244-L283)）按 *栈位置降序* 在 `fiber->openUpvalues` 上查找：
- 相同 `value` 指针 → 复用同一个 `ObjUpvalue`（多个闭包共享同一个变量是关键正确性保证）。
- 否则插入到链表合适位置以保持有序。

### 开放 vs 关闭上值
- **开放**（open）：`ObjUpvalue.value` 直接指向 `fiber->stack` 中的某个 `Value` 槽；只要那个局部还在作用域内、它的 Value 就是"实时"的（[wren_value.h L178-L195](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L178-L195)）。所有开放上值挂在 `fiber->openUpvalues` 单向链表上，按指针地址降序。
- **关闭**（closed）：作用域结束时把栈上的 Value **拷贝**进 `ObjUpvalue.closed`，并把 `value = &closed`。[wren_vm.c closeUpvalues](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L287-L301)：
  ```c
  while (fiber->openUpvalues != NULL && fiber->openUpvalues->value >= last) {
    ObjUpvalue* upvalue = fiber->openUpvalues;
    upvalue->closed = *upvalue->value;
    upvalue->value  = &upvalue->closed;
    fiber->openUpvalues = upvalue->next;
  }
  ```
  调用者：
  - `CODE_CLOSE_UPVALUE`（[L1209-L1213](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1209-L1213)）：单个被捕获局部离开块作用域时由编译器在 [discardLocals](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1492-L1517) 中发射（看到 `locals[local].isUpvalue` 时）。
  - `CODE_RETURN`（[L1215-L1257](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1215-L1257)）：函数返回时 `closeUpvalues(fiber, stackStart)` 一次性关闭整个帧上所有被捕获的局部。

注意：链表按降序排序使 `closeUpvalues` 能从头部线性截断 ──"所有 `value >= last` 的连续段"。

---

## 4. 方法分派

### 方法表 = 全局符号 → 槽位
所有类共享一张 **全局方法名符号表** `vm->methodNames`（[wren_vm.h L113](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.h#L113)）。每个签名（如 `add(_)`、`call(_,_)`）首次出现时被 `wrenSymbolTableEnsure` 分配一个 *单调递增* 的整数 `symbol`。`ObjClass.methods`（[wren_value.h L401-L416](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L401-L416)）是 `MethodBuffer`，按相同 symbol 索引；类没有该方法时槽里是 `METHOD_NONE`（[wren_value.c wrenBindMethod L120-L132](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L120-L132) 会把 buffer 撑大到 `symbol+1`）。

注释把它叫做"a hash table that never has collisions but has a really low load factor"（[wren_value.h L406-L408](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L406-L408)）。
- **时间**：调用就是一次 O(1) 数组下标访问 `classObj->methods.data[symbol]`，没有哈希计算、没有探测；
- **空间**：每个类的 `methods` 数组最大长度 = 整个程序里出现的不同签名数量。多数类只实现少数方法，会留下大量 `METHOD_NONE` 空槽。Wren 选择小内存浪费换运行期热路径上的最少分支。

### CALL_n / SUPER_n 的差异（[wren_vm.c L965-L1092](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L965-L1092)）
两者都从字节码读 16-bit `symbol`，再算 `args = stackTop - numArgs`：
- `CALL_n`：`classObj = wrenGetClassInline(vm, args[0])` —— 用 *接收者运行期类型* 查类（[wren_vm.h L209-L237](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.h#L209-L237)）。
- `SUPER_n`：`classObj = AS_CLASS(fn->constants.data[READ_SHORT()])` —— 直接读取编译期写入常量池的 *父类对象*；这样 `super.foo()` 不会被子类覆盖再次调度。

之后两者用 `goto completeCall;` 进入同一段尾代码（[L1036-L1091](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1036-L1091)）。

### 四种 MethodType（[wren_value.h L357-L388](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L357-L388)，dispatch 在 [L1045-L1090](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1045-L1090)）

| 类型 | 来源 | 调用路径 | 关键行 |
|---|---|---|---|
| `METHOD_PRIMITIVE` | C 原语，受参数指针访问 fiber 栈 | 直接调用 `method->as.primitive(vm, args)`；返回 `true` 则把结果留在 `args[0]` 并下移 `stackTop`；返回 `false` 表示发生错误 / fiber 切换 / 栈帧改变，立刻 `STORE_FRAME` + 重新 `LOAD_FRAME` | [L1047-L1063](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1047-L1063) |
| `METHOD_FUNCTION_CALL` | `Fn.call(...)` 的特殊 primitive | 先 `checkArity` 校验参数个数（[L809-L821](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L809-L821)），再 `STORE_FRAME` → `as.primitive` → `LOAD_FRAME` | [L1065-L1074](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1065-L1074) |
| `METHOD_FOREIGN` | host 注册的 C 回调 | `callForeign`（[L384-L397](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L384-L397)）：设 `apiStack`，调 `foreign(vm)`，把 `stackTop` 压到 `apiStack+1`（保留单个返回槽），清 `apiStack` | [L1076-L1079](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1076-L1079) |
| `METHOD_BLOCK` | 用户 Wren 代码 | `STORE_FRAME` → `wrenCallFunction(vm, fiber, method->as.closure, numArgs)` 压新帧 → `LOAD_FRAME` 切到新帧执行 | [L1081-L1085](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1081-L1085) |

### 方法不存在
`completeCall:` 一开始判断（[L1037-L1043](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1037-L1043)）：
```c
if (symbol >= classObj->methods.count ||
    (method = &classObj->methods.data[symbol])->type == METHOD_NONE) {
  methodNotFound(vm, classObj, symbol);
  RUNTIME_ERROR();
}
```
[methodNotFound](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L438-L442) 用 `vm->methodNames.data[symbol]->value` 反查名字，构造 `"@ does not implement '$'."` 错误对象写进 `fiber->error`，再走 `RUNTIME_ERROR()` 流程。

---

## 5. Fiber 协程与错误传播

### 每 Fiber 独立栈与帧（[wren_value.h ObjFiber L316-L355](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L316-L355)）
每个 `ObjFiber` 自带：
- `stack / stackTop / stackCapacity` —— 该 fiber 专属值栈；
- `frames / numFrames / frameCapacity` —— 该 fiber 自己的调用帧序列；
- `openUpvalues` —— 仅作用于本 fiber 栈位置；
- `caller` —— "谁通过 call/try 启动了我，等我 return/yield 时回到它"；
- `error`、`state`（`FIBER_TRY` / `FIBER_ROOT` / `FIBER_OTHER`，[L300-L314](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L300-L314)）。

之所以每 fiber 一套栈是因为 yield 必须能让本 fiber 的所有局部、临时、调用帧整体冻结起来，下次 `transfer` 或 `call` 进来时 `runInterpreter` 重新执行 `LOAD_FRAME()` 即可继续。

### 协程驱动
- `Fiber.call/transfer/try/yield` 全是 primitive（实现在 [wren_core.c](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_core.c)），它们调整 `vm->fiber`、`fiber->caller`、`fiber->state` 然后返回 `false`，让 `runInterpreter` 在 `METHOD_PRIMITIVE` 分支检测到 fiber 切换：
  ```c
  if (method->as.primitive(vm, args)) { /* 普通成功 */ }
  else {
    STORE_FRAME();
    fiber = vm->fiber;
    if (fiber == NULL) return WREN_RESULT_SUCCESS;
    if (wrenHasError(fiber)) RUNTIME_ERROR();
    LOAD_FRAME();
  }
  ```
  ([L1045-L1063](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1045-L1063))。

### CODE_RETURN（[L1215-L1257](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1215-L1257)）
1. `result = POP()`，`fiber->numFrames--`；
2. `closeUpvalues(fiber, stackStart)` —— 把当前帧上所有被捕获的局部全部 hoist 到堆；
3. 如果 `numFrames == 0`：fiber 已结束。
   - `fiber->caller == NULL` → 程序退出，把 `result` 写回 `stack[0]`，返回 `WREN_RESULT_SUCCESS`；
   - 否则切回 `caller`，把 `result` 放在 caller 的 `stackTop[-1]`（即 caller 之前发起 call 时压在那个位置上的占位），并清 `caller` 链；
4. 否则正常返回：把 `result` 写到 `stackStart[0]`（receiver 槽），`stackTop = frame->stackStart + 1`；
5. `LOAD_FRAME()` 切到上一帧继续。

### runtimeError（[L399-L434](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L399-L434)）
当任意路径写下 `vm->fiber->error` 后被 `RUNTIME_ERROR()` 触发：
```c
ObjFiber* current = vm->fiber;
Value error = current->error;
while (current != NULL) {
  current->error = error;                    // 沿链 abort 时复制同一错误
  if (current->state == FIBER_TRY) {         // 找到 try() 启动者
    current->caller->stackTop[-1] = current->error;  // 把错误作为 try 的返回值
    vm->fiber = current->caller;
    return;                                  // 进入 caller，继续 dispatch
  }
  ObjFiber* caller = current->caller;
  current->caller = NULL;                    // 取消挂接，永不再恢复
  current = caller;
}
wrenDebugPrintStackTrace(vm);                // 没人捕获 → 终止
vm->fiber = NULL; vm->apiStack = NULL;
```

`FiberState` 区别：
- `FIBER_TRY`：被 `Fiber.try()` 启动 → `runtimeError` 命中后立即把错误以 *值* 的形式给 caller，**不 abort** 它；
- `FIBER_ROOT` / `FIBER_OTHER`：错误向上传播一级；caller 也照做；最终若顶端仍是 `FIBER_OTHER`/`FIBER_ROOT`，整个 VM 抛出 `WREN_RESULT_RUNTIME_ERROR`。

---

## 6. 垃圾回收

三色（白/灰/黑）标记-清除：
- 白 = 未触及；灰 = `isDark == true` 但子节点未扫；黑 = `isDark == true` 且子节点也已 gray。
- "Gray stack" 是 `vm->gray / vm->grayCount / vm->grayCapacity`（[wren_vm.h L73-L77](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.h#L73-L77)）。

### 根集合（[wrenCollectGarbage L125-L211](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L125-L211)）
1. 重置 `vm->bytesAllocated = 0`（之后由黑化阶段重新累加每个对象的尺寸）；
2. `wrenGrayObj(vm->modules)` —— 全局 module 注册表；
3. `vm->tempRoots[0..numTempRoots]` —— 临时根（`wrenPushRoot` / `wrenPopRoot`）；
4. `vm->fiber` —— 当前活跃 fiber；
5. 所有 `WrenHandle*` 链表 —— host 持有的句柄；
6. `vm->compiler` —— 若编译器在跑，遍历它的临时栈（防止编译期分配的 ObjFn / 字符串被回收）；
7. `vm->methodNames` —— 全局符号表里持有的 ObjString*。

### 标记传播（[wrenBlackenObjects L1219-L1227](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L1219-L1227)）
循环：从 gray stack 弹一个对象，按 `obj->type` 分派到 `blackenClass / blackenClosure / blackenFiber / ...`；每个 `blacken*` 把所有它引用的对象再 `wrenGrayObj`（若未 dark），并加上 `vm->bytesAllocated += sizeof(...)` 记账。直到 gray stack 为空。

`wrenGrayObj`（[L976-L997](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L976-L997)）：
```c
if (obj == NULL || obj->isDark) return;
obj->isDark = true;
// 必要时扩 gray buffer
vm->gray[vm->grayCount++] = obj;
```
"已 dark 直接 return" 是消除环路的关键。

闭包黑化（[blackenClosure L1039-L1053](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L1039-L1053)）会标记 `closure->fn` 与每个 `closure->upvalues[i]`；上值黑化（[blackenUpvalue L1184-L1191](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L1184-L1191)）会 `wrenGrayValue(upvalue->closed)` —— 这一步对开放上值是 NULL_VAL 无副作用，对关闭上值则保住堆中数据存活。Fiber 黑化（[blackenFiber L1055-L1085](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L1055-L1085)）扫所有调用帧的 closure、整段 `[stack, stackTop)` 的 Value、整个 `openUpvalues` 链以及 `caller` 与 `error`。

### 清除（[L176-L193](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L176-L193)）
所有对象都串在单链表 `vm->first` 上（`Obj.next`，[wren_value.h L117](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.h#L117)）。一次线性遍历：
```c
Obj** obj = &vm->first;
while (*obj != NULL) {
  if (!((*obj)->isDark)) {           // 白色 → 释放
    Obj* unreached = *obj;
    *obj = unreached->next;
    wrenFreeObj(vm, unreached);
  } else {
    (*obj)->isDark = false;          // 黑色 → 复位为白以备下次 GC
    obj = &(*obj)->next;
  }
}
```
最后调整 `nextGC = bytesAllocated + bytesAllocated * heapGrowthPercent / 100`，并不低于 `minHeapSize`。

### 触发与防重入（[wrenReallocate L213-L237](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L213-L237)）
```c
vm->bytesAllocated += newSize - oldSize;
#if WREN_DEBUG_GC_STRESS
  if (newSize > 0) wrenCollectGarbage(vm);
#else
  if (newSize > 0 && vm->bytesAllocated > vm->nextGC) wrenCollectGarbage(vm);
#endif
return vm->config.reallocateFn(memory, newSize, vm->config.userData);
```
关键点：
- **只在 `newSize > 0`（也即 *分配* 而非 *释放*）的路径上 GC**。`wrenFreeObj` → `wrenReallocate(... newSize=0)` 才不会让 sweep 阶段递归再触发一次 GC，从而防止重入。
- GC 用的 `gray buffer` 自身是通过 `vm->config.reallocateFn` 直接调用、不走 `wrenReallocate`（[L988-L994](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L988-L994)），避免 GC 中扩容触发新 GC。

---

## 7. 综合示例与关键不变量

### 示例 Wren 代码

```wren
class Counter {
  construct new(start) {
    var n = start                 // 外层局部
    _step = Fn.new {              // 内层函数捕获 n
      n = n + 1
      return n
    }
  }
  next { _step.call }
}

var c = Counter.new(10)
System.print(c.next)              // 11
System.print(c.next)              // 12
```

### 编译阶段
- 外层 `construct new(start)` 的编译器看到 `Fn.new { ... }`，进入新 `Compiler` 编内层匿名 fn。
- 内层引用 `n` → `findUpvalue(name="n")`：父级 local index = 1（假设 `start` 占 0、`n` 占 1）；标 `parent->locals[1].isUpvalue = true`，`addUpvalue(inner, isLocal=true, index=1)` → 返回 0。
- 内层结束时（[wren_compiler.c L1680-L1696](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1680-L1696)）向 *外层* 字节码追加 `CODE_CLOSURE <const>  1 1`。
- 外层退出 `n` 所在块时，`discardLocals` 看到 `isUpvalue=true`，发射 `CODE_CLOSE_UPVALUE`（[L1503-L1505](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_compiler.c#L1503-L1505)）。

### 运行阶段
1. 调用 `Counter.new(10)` → 进入 `construct` 帧，`stackStart[0]=this`，`stackStart[1]=10`，本地 `n` 占据 `stackStart[2]`。
2. 执行 `CODE_CLOSURE`：分配 `ObjClosure`，PUSH（避免 GC），读两个 byte `(1,1)`：`isLocal=1` → `captureUpvalue(vm, fiber, frame->stackStart + 1)`（[L1286-L1287](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1286-L1287)），把它链进 `fiber->openUpvalues`，写入 `closure->upvalues[0]`。
3. 闭包被赋值给字段 `_step`，存活于堆里。
4. 构造函数返回前编译器发射的 `CODE_CLOSE_UPVALUE`（针对 `n`）执行 → `closeUpvalues(fiber, fiber->stackTop-1)`（[L1209-L1213](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1209-L1213)）→ 把 `n` 的当前值（10）拷进 `ObjUpvalue.closed`，`upvalue->value = &upvalue->closed`，从 `openUpvalues` 摘除。`CODE_RETURN` 时 `closeUpvalues(fiber, stackStart)` 再兜底关一次（已不在链上则空操作）。
5. 后续 `c.next` → 调度到 `next` getter → 它走 `_step.call` → `Fn.call` 走 `METHOD_FUNCTION_CALL` 路径 → 压新帧执行内层函数。`LOAD_UPVALUE 0`（[L1094-L1099](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1094-L1099)）通过 `frame->closure->upvalues[0]->value` 间接读到 `closed`，修改后写回；下次再调用读到的 11、12。

### GC 阶段（任意时刻）
- 根 `vm->fiber`（运行中）→ blackenFiber 扫到 `c`（在栈/字段/句柄链上）→ blackenInstance 扫 `c->fields[_step]` → grayObj `ObjClosure` → blackenClosure 扫 `closure->fn`、`closure->upvalues[0]`（即那个 `ObjUpvalue`）→ blackenUpvalue 把 `upvalue->closed` 当作 Value gray（这里是 Num，非 Obj，不入 gray stack）。
- 若 `n` 所在的局部仍未关闭（即闭包尚未被关上之前 GC），那么该 `ObjUpvalue` 同时被 blackenFiber 的 "Open upvalues" 循环（[wren_value.c L1069-L1075](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L1069-L1075)）扫到，且其 `value` 指向的栈槽也在 `for slot in [stack, stackTop)` 那段被遍历到，二者一致地保活。

### 最难追踪的关键不变量

1. **`fiber->openUpvalues` 必须按栈位置降序**。
   - 依据：[captureUpvalue L256-L282](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L256-L282) 通过 `while (upvalue != NULL && upvalue->value > local)` 单调下行寻找位置，并按"前一个、当前、新建"链法保留有序；
   - [closeUpvalues L289-L300](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L289-L300) 利用同一不变量从头部连续截断 `value >= last` 的所有节点。
   - 一旦顺序破坏，要么会创建重复的 `ObjUpvalue`（多个闭包看不到同一份变量），要么 `closeUpvalues` 漏关导致悬挂栈指针。

2. **GC 之前必须 `STORE_FRAME`，GC 之后必须 `LOAD_FRAME`，期间所有"刚分配但未入栈"的对象必须挂在 tempRoots 上**。
   - 依据：解释器把 `ip / fn / frame / stackStart` 缓存到寄存器（[L835-L862](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L835-L862)）；只有 `frame->ip` 是 GC 唯一的"权威"位置，否则 `blackenFiber` 看到的 `frames[i].closure` 是对的，但若 `frame->ip` 旧了就无法在错误回放时给出正确行号。
   - 例：`compileInModule`（[L453-L497](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L453-L497)）调用 `wrenMapSet(vm->modules, ...)` 前 `wrenPushRoot(module)`；又如 `CODE_CLOSURE` 先 `PUSH(OBJ_VAL(closure))` 再去 `captureUpvalue`/`wrenNewUpvalue`（后者会 `ALLOCATE` 触发可能的 GC）。
   - 类似的还有 `wrenMakeHandle`（[L1476-L1493](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.c#L1476-L1493)）在 `ALLOCATE` 前 `wrenPushRoot`。

3. **方法符号表索引在所有类间是同一个全局编号**。
   - 依据：`vm->methodNames` 是 VM 唯一一张全局符号表（[wren_vm.h L111-L113](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_vm.h#L111-L113)）。`wrenSymbolTableEnsure` 给签名 `"add(_)"` 永远返回同一个整数；编译器将该整数烧入字节码里 `CALL_n` / `SUPER_n` 的 16-bit 操作数，运行期 `classObj->methods.data[symbol]` 直接索引。
   - `wrenBindMethod` 必要时 `wrenMethodBufferFill(...METHOD_NONE..., symbol - count + 1)`（[wren_value.c L120-L132](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L120-L132)）确保槽存在；从父类继承时 `wrenBindSuperclass` 把父表整段复制到子类（[L80-L82](file:///e:/gsb/608/gsb_12/Summer/src/vm/wren_value.c#L80-L82)）。
   - 若编译器与 VM 不共用同一张符号表（比如 hot-reload 时被 reset），所有现有字节码里的 symbol 就指错位置 → 静默调错方法。它的"低装载因子表"模型完全建立在"全程同一编号"之上。

---

*以上分析基于本仓库 `src/vm/` 当前快照的源码，仅做静态阅读与推理，未运行/编译/修改任何代码。*
