# Wren VM 内核深度分析

## NaN Tagging 位布局图

```
IEEE 754 双精度浮点数 (64 bits)
┌─────────┬───────────────────────┬──────────────────────────────────────────┐
│ Sign(1)  │ Exponent(11)          │ Mantissa(52)                             │
│ S        │ EEEEEEEEEEE           │ MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM │
└─────────┴───────────────────────┴──────────────────────────────────────────┘

NaN Tagging 编码 (WREN_NAN_TAGGING 开启时, Value = uint64_t)
┌─────────┬───────────────────────┬──────────────────────────────────────────┐
│    S     │      11 bits          │              52 bits                     │
├─────────┼───────────────────────┼──────────────────────────────────────────┤
│         │                       │                                          │
│  数字   │  正常指数 (非全1)      │  原样 double —— 算术零开销               │
│         │                       │                                          │
├─────────┼───────────────────────┼──────────────────────────────────────────┤
│         │                       │                                          │
│  0      │ 1 1111 1111 11        │ 1xxxxxxxxx...xxxxxxxx  [TAG: 低3位]      │
│  (0)    │   (QNAN 掩码)         │ ↑最高mantissa位=1(quiet NaN)             │
│         │                       │                                          │
│         │   → 单例值 (singleton)                                           │
│         │                       │   TAG_NAN       = 0  (NaN 本身)          │
│         │                       │   TAG_NULL      = 1  (null)              │
│         │                       │   TAG_FALSE     = 2  (false)             │
│         │                       │   TAG_TRUE      = 3  (true)              │
│         │                       │   TAG_UNDEFINED = 4  (undefined)         │
│         │                       │   TAG_UNUSED*   = 5,6,7                  │
│         │                       │                                          │
├─────────┼───────────────────────┼──────────────────────────────────────────┤
│         │                       │                                          │
│  1      │ 1 1111 1111 11        │ 1xxxxxxxxx...xxxxxxxx  [低51位=指针]     │
│ (SIGN   │   (QNAN 掩码)         │ ↑最高mantissa位=1                        │
│  _BIT)  │                       │   64位系统仅用低48位地址, 足够存放        │
│         │                       │                                          │
│         │   → 堆对象指针 (Obj*)                                            │
│         │                       │   AS_OBJ: value & ~(SIGN_BIT | QNAN)     │
│         │                       │                                          │
└─────────┴───────────────────────┴──────────────────────────────────────────┘

关键常量:
  SIGN_BIT = 1 << 63                (0x8000000000000000)
  QNAN     = 0x7ffc000000000000     (指数全1 + 最高mantissa位=1 + 次高=0)
  MASK_TAG = 7                      (低3位掩码, 提取单例标签)
```

## 闭包上值生命周期图

```
编译期                              运行期
┌────────────────────┐              ┌─────────────────────────────────────────┐
│ Compiler A (外层)   │              │ Fiber 栈                                │
│  locals: [x(slot2)]│              │  ... | slot0 | slot1 | slot2(x) | ... │
│         isUpvalue=T│              └─────────────────────────────────────────┘
│                     │                         ↑
│ Compiler B (内层)   │              ┌──────────┘
│  findUpvalue(B,"x") │              │  开放上值 (open upvalue)
│   → resolveLocal(A) │              │  ObjUpvalue { .value ──→ 栈slot2 }
│     = slot2         │              │  fiber->openUpvalues 链表 (按栈位置降序)
│   → addUpvalue(B,   │              │
│      isLocal=T,2)   │              │  CODE_CLOSURE 执行时:
│                     │              │    closure->upvalues[i] = captureUpvalue(...)
│  upvalues[0] =      │              │
│   {isLocal=T,idx=2} │              │  ─── 局部变量 x 离开作用域 ───
│                     │              │
│ endCompiler(B):     │              │  CODE_CLOSE_UPVALUE / closeUpvalues():
│  emit CODE_CLOSURE  │              │    upvalue->closed = *upvalue->value  (hoist)
│  emit (1, 2)        │              │    upvalue->value  = &upvalue->closed
│   isLocal=1, idx=2  │              │
└────────────────────┘              │  关闭上值 (closed upvalue)
                                    │  ObjUpvalue { .value ──→ .closed }
                                    │  闭包此后通过 .closed 访问变量
                                    └─────────────────────────────────────────┘

GC 标记链路:
  根集合(fiber) → fiber->stack[?] → Value(ObjClosure)
    → closure->upvalues[i] → ObjUpvalue
      → upvalue->closed (若已关闭) 或 *upvalue->value (若开放, 指向栈)
```

---

## 1. 值表示与 NaN Tagging

### 1.1 核心思想

Wren 的 `Value` 类型在 `WREN_NAN_TAGGING` 开启时定义为 `uint64_t`（[wren_value.h:123](src/vm/wren_value.h#L123)）。它利用 IEEE 754 双精度浮点数中 NaN 的冗余位模式，在一个 64 位字里同时编码三种值：

1. **double 数字**：原样存储，不做任何变换
2. **单例值**（null / true / false / undefined）：用 `QNAN | TAG_*` 编码
3. **堆对象指针**（Obj*）：用 `SIGN_BIT | QNAN | 指针` 编码

### 1.2 关键常量与宏

| 常量/宏 | 定义位置 | 值/行为 | 作用 |
|---------|---------|---------|------|
| `SIGN_BIT` | [wren_value.h:553](src/vm/wren_value.h#L553) | `(uint64_t)1 << 63` | 区分单例(0)与指针(1) |
| `QNAN` | [wren_value.h:556](src/vm/wren_value.h#L556) | `0x7ffc000000000000` | quiet NaN 的位模式：指数全1 + mantissa最高位1 + 次高位0 |
| `MASK_TAG` | [wren_value.h:569](src/vm/wren_value.h#L569) | `7` | 提取单例标签的低3位掩码 |
| `TAG_NULL` | [wren_value.h:573](src/vm/wren_value.h#L573) | `1` | null 的标签 |
| `TAG_FALSE` | [wren_value.h:574](src/vm/wren_value.h#L574) | `2` | false 的标签 |
| `TAG_TRUE` | [wren_value.h:575](src/vm/wren_value.h#L575) | `3` | true 的标签 |
| `TAG_UNDEFINED` | [wren_value.h:576](src/vm/wren_value.h#L576) | `4` | undefined 的标签 |

**判定宏**：

- `IS_NUM(value)`（[wren_value.h:559](src/vm/wren_value.h#L559)）：`(value & QNAN) != QNAN`——只要高位的 NaN 标记不完整，就是正常 double。
- `IS_OBJ(value)`（[wren_value.h:562](src/vm/wren_value.h#L562)）：`(value & (QNAN | SIGN_BIT)) == (QNAN | SIGN_BIT)`——同时满足 NaN 位和符号位，才是堆指针。
- `AS_OBJ(value)`（[wren_value.h:585](src/vm/wren_value.h#L585)）：`(Obj*)(uintptr_t)(value & ~(SIGN_BIT | QNAN))`——剥离高 13 位标签，还原指针。
- `AS_BOOL(value)`（[wren_value.h:582](src/vm/wren_value.h#L582)）：`(value) == TRUE_VAL`——直接与 `TRUE_VAL` 做整数比较。

**构造宏**：

- `NUM_VAL(num)` → `wrenNumToValue(num)` → `wrenDoubleToBits(num)`（[wren_math.h:27](src/vm/wren_math.h#L27)）：直接 `union` 重解释，零开销。
- `OBJ_VAL(obj)` → `wrenObjectToValue(obj)`（[wren_value.h:841](src/vm/wren_value.h#L841)）：`SIGN_BIT | QNAN | (uint64_t)(uintptr_t)(obj)`。
- `BOOL_VAL(b)` → `b ? TRUE_VAL : FALSE_VAL`（[wren_value.h:66](src/vm/wren_value.h#L66)）。

### 1.3 为何数字算术零开销

因为 `Value` 就是 `uint64_t`，而一个合法的 double 的位模式永远不会与 NaN tagging 的编码冲突（合法 double 的指数位不全为 1，或全 1 时是 ±Infinity 而非 NaN）。因此 `AS_NUM` 只需 `wrenDoubleFromBits(value)`——一次 `union` 重解释，无需任何掩码操作。`NUM_VAL` 同理。CPU 对 double 的算术指令可以直接作用于这个 64 位值，只需先通过 `union` 转回 `double` 类型。

### 1.4 关闭 NaN Tagging 时的 struct 表示

当 `WREN_NAN_TAGGING` 未定义时（[wren_value.h:127-147](src/vm/wren_value.h#L127-147)），`Value` 变为一个带标签的联合体：

```c
typedef struct {
  ValueType type;   // VAL_FALSE, VAL_NULL, VAL_NUM, VAL_TRUE, VAL_UNDEFINED, VAL_OBJ
  union {
    double num;
    Obj* obj;
  } as;
} Value;
```

**权衡**：
- **优点**：可移植性更好（不依赖 NaN 位模式假设，可在不支持 IEEE 754 或 64 位指针超过 48 位的平台上工作）；调试时类型字段直观可见，便于在调试器中检查。
- **缺点**：`Value` 占 16 字节而非 8 字节，内存带宽翻倍；每次类型判断需要访问 `type` 字段而非做位运算；`IS_NUM` 变为 `value.type == VAL_NUM`，比位掩码慢。

---

## 2. 字节码与分派

### 2.1 X-Macro 定义 Opcode

[wren_opcodes.h](src/vm/wren_opcodes.h) 使用 X-Macro 技术定义所有 opcode：

```c
OPCODE(CONSTANT, 1)
OPCODE(NULL, 1)
OPCODE(FALSE, 1)
...
```

每个 `OPCODE(name, stackEffect)` 调用有两个参数：指令名和**栈效应**。栈效应表示该指令对栈净增减的值数（正为压栈，负为弹栈）。例如 `POP` 的栈效应为 `-1`，`CONSTANT` 为 `1`，`CALL_1` 为 `-1`（弹出一个参数加接收者，压入一个返回值，净减 1）。

在 [wren_compiler.c:414-418](src/vm/wren_compiler.c#L414-418)，X-Macro 被用来生成 `stackEffects[]` 数组：

```c
static const int stackEffects[] = {
  #define OPCODE(_, effect) effect,
  #include "wren_opcodes.h"
  #undef OPCODE
};
```

在 [wren_vm.h:14-18](src/vm/wren_vm.h#L14-18)，X-Macro 生成 `Code` 枚举：

```c
typedef enum {
  #define OPCODE(name, _) CODE_##name,
  #include "wren_opcodes.h"
  #undef OPCODE
} Code;
```

### 2.2 Computed Goto vs Switch 分派

[wren_vm.c:890-918](src/vm/wren_vm.c#L890-918) 定义了两套分派机制：

**Computed Goto（`WREN_COMPUTED_GOTO` 开启时）**：

```c
static void* dispatchTable[] = {
  #define OPCODE(name, _) &&code_##name,
  #include "wren_opcodes.h"
  #undef OPCODE
};

#define DISPATCH() goto *dispatchTable[instruction = (Code)READ_BYTE()]
#define CASE_CODE(name) code_##name
```

`dispatchTable` 是一个函数指针数组，每个元素指向对应 opcode 处理代码的标签地址（GCC/Clang 的 labels-as-values 扩展）。`DISPATCH()` 通过一次数组索引 + 间接跳转完成分派，避免了 `switch` 的边界检查和分支预测不友好的跳转表。

**传统 Switch**：

```c
#define INTERPRET_LOOP loop: switch (instruction = (Code)READ_BYTE())
#define CASE_CODE(name) case CODE_##name
#define DISPATCH() goto loop
```

每次分派回到循环顶部，重新进入 `switch`。编译器通常会将 `switch` 编译为跳转表，但仍有额外的边界检查和分支预测开销。

**取舍**：Computed goto 在主流编译器（GCC/Clang）上对解释器循环有 15-25% 的性能提升，因为分支预测更准确、CPU 流水线更顺畅。但它是编译器扩展，不可移植；在不支持的平台回退到 `switch`。

**Opcode 排列顺序对性能的影响**：[wren_opcodes.h:10-13](src/vm/wren_opcodes.h#L10-13) 的注释明确指出，opcode 的定义顺序直接决定了 `dispatchTable` 的布局。频繁执行的指令（如 `LOAD_LOCAL_*`、`CALL_*`）如果相邻排列，CPU 的指令缓存和分支预测会更高效。因此修改顺序需要跑基准测试验证。

### 2.3 LOAD_FRAME / STORE_FRAME 缓存机制

[wren_vm.c:835-862](src/vm/wren_vm.c#L835-862) 定义了关键宏：

```c
register CallFrame* frame;
register Value* stackStart;
register uint8_t* ip;
register ObjFn* fn;

#define STORE_FRAME() frame->ip = ip

#define LOAD_FRAME()                        \
  do {                                      \
    frame = &fiber->frames[fiber->numFrames - 1]; \
    stackStart = frame->stackStart;         \
    ip = frame->ip;                         \
    fn = frame->closure->fn;                \
  } while (false)
```

**加速原理**：`ip`、`stackStart`、`fn` 在解释器循环中被极其频繁地访问。将它们缓存到 `register` 局部变量中，避免了每次访问都要通过 `fiber->frames[numFrames-1]->...` 的多级指针间接寻址。

**同步时机**：
- **调用前**（`METHOD_BLOCK`、`IMPORT_MODULE` 等）：先 `STORE_FRAME()` 把 `ip` 写回当前帧，再 `wrenCallFunction()` 推入新帧，然后 `LOAD_FRAME()` 刷新局部变量。
- **返回时**（`CODE_RETURN`）：弹出帧后 `LOAD_FRAME()` 刷新。
- **GC / Fiber 切换时**：`RUNTIME_ERROR()` 宏先 `STORE_FRAME()`，处理错误后可能切换 fiber，再 `LOAD_FRAME()`。
- **Primitive 返回 false**（表示 fiber 切换或帧变化）：`STORE_FRAME()` → 刷新 fiber → `LOAD_FRAME()`。

---

## 3. 闭包与上值捕获

### 3.1 编译期：findUpvalue 递归解析

当内层函数引用外层局部变量时，编译器通过 [wren_compiler.c:1577](src/vm/wren_compiler.c#L1577) 的 `findUpvalue` 递归沿 `Compiler->parent` 链向上查找：

1. **在直接外层找局部变量**：调用 `resolveLocal(compiler->parent, name, length)`（[wren_compiler.c:1587](src/vm/wren_compiler.c#L1587)）。若找到，标记该局部为 `isUpvalue = true`（[wren_compiler.c:1592](src/vm/wren_compiler.c#L1592)），然后调用 `addUpvalue(compiler, true, local)`（[wren_compiler.c:1594](src/vm/wren_compiler.c#L1594)），其中 `isLocal=true` 表示捕获的是外层的局部变量。

2. **递归查找更外层**：若直接外层没有，递归调用 `findUpvalue(compiler->parent, name, length)`（[wren_compiler.c:1603](src/vm/wren_compiler.c#L1603)）。若在更外层找到，则当前层捕获的是外层的上值，调用 `addUpvalue(compiler, false, upvalue)`（[wren_compiler.c:1606](src/vm/wren_compiler.c#L1606)），`isLocal=false` 表示捕获的是外层的上值。这自动实现了**闭包扁平化**：中间每一层都会登记对应的 upvalue，形成一条传递链。

3. **方法边界截断**：如果外层是方法（`enclosingClass != NULL`）且变量名不以 `_` 开头，则停止查找（[wren_compiler.c:1584](src/vm/wren_compiler.c#L1584)），因为方法不会捕获外层局部变量。

### 3.2 addUpvalue 去重

[wren_compiler.c:1551-1564](src/vm/wren_compiler.c#L1551-1564)：`addUpvalue` 先遍历已有的 upvalues 列表，如果 `(isLocal, index)` 对已存在则直接返回已有索引，避免重复登记。这确保了多个闭包引用同一个外层变量时共享同一个 upvalue 槽位。

### 3.3 CODE_CLOSURE 指令与运行期填充

编译器在 `endCompiler` 中生成 `CODE_CLOSURE` 指令（[wren_compiler.c:1688](src/vm/wren_compiler.c#L1688)），其后紧跟每个 upvalue 的 `(isLocal, index)` 操作数对（[wren_compiler.c:1692-1696](src/vm/wren_compiler.c#L1692-1696)）。

运行期 `CODE_CLOSURE` 的处理在 [wren_vm.c:1270-1296](src/vm/wren_vm.c#L1270-1296)：

```c
ObjFn* function = AS_FN(fn->constants.data[READ_SHORT()]);
ObjClosure* closure = wrenNewClosure(vm, function);
PUSH(OBJ_VAL(closure));

for (int i = 0; i < function->numUpvalues; i++)
{
  uint8_t isLocal = READ_BYTE();
  uint8_t index = READ_BYTE();
  if (isLocal)
    closure->upvalues[i] = captureUpvalue(vm, fiber, frame->stackStart + index);
  else
    closure->upvalues[i] = frame->closure->upvalues[index];
}
```

- `isLocal=1`：捕获当前帧的局部变量（`frame->stackStart + index`），通过 `captureUpvalue` 在 fiber 的开放上值链表中查找或创建。
- `isLocal=0`：直接复用当前帧闭包已有的 upvalue（`frame->closure->upvalues[index]`），实现共享。

注意：闭包先被 `PUSH` 到栈上（[wren_vm.c:1276](src/vm/wren_vm.c#L1276)），这确保了在后续 `captureUpvalue` 可能触发 GC 时，闭包对象已被栈引用而不会被回收。

### 3.4 开放上值与关闭上值

**开放上值（Open Upvalue）**：`ObjUpvalue.value` 指向 fiber 栈上的某个 slot，变量仍在栈上存活。`fiber->openUpvalues` 是一个按栈位置**降序**排列的链表（链头指向最靠近栈顶的上值，[wren_value.h:343-344](src/vm/wren_value.h#L343-344)）。

**关闭上值（Close Upvalue）**：当被捕获的局部变量离开作用域时，值从栈上"hoist"（提升）到 `ObjUpvalue.closed` 字段中。

`closeUpvalues` 函数（[wren_vm.c:287-301](src/vm/wren_vm.c#L287-301)）：

```c
static void closeUpvalues(ObjFiber* fiber, Value* last)
{
  while (fiber->openUpvalues != NULL &&
         fiber->openUpvalues->value >= last)
  {
    ObjUpvalue* upvalue = fiber->openUpvalues;
    upvalue->closed = *upvalue->value;     // 把栈上的值复制到 closed
    upvalue->value = &upvalue->closed;     // 指针改为指向 closed
    fiber->openUpvalues = upvalue->next;   // 从开放链表中移除
  }
}
```

**触发时机**：
1. `CODE_CLOSE_UPVALUE`（[wren_vm.c:1209-1213](src/vm/wren_vm.c#L1209-1213)）：编译器在 `discardLocals` 中为每个 `isUpvalue=true` 的局部变量发射此指令（[wren_compiler.c:1503-1506](src/vm/wren_compiler.c#L1503-1506)）。
2. `CODE_RETURN`（[wren_vm.c:1221](src/vm/wren_vm.c#L1221)）：`closeUpvalues(fiber, stackStart)` 关闭当前帧栈起点以上的所有开放上值。

`captureUpvalue`（[wren_vm.c:244-283](src/vm/wren_vm.c#L244-283)）在插入新上值时保持链表按栈位置降序：它从链头开始遍历，跳过 `value > local` 的节点，在正确位置插入。这保证了 `closeUpvalues` 可以从链头开始连续关闭，无需搜索。

---

## 4. 方法分派

### 4.1 符号索引与 methods 表

Wren 使用全局符号表 `vm->methodNames`（[wren_vm.h:113](src/vm/wren_vm.h#L113)）将方法签名字符串映射为整数 symbol。每个 `ObjClass` 的 `methods` 字段（`MethodBuffer`，[wren_value.h:409](src/vm/wren_value.h#L409)）是一个以 symbol 为索引的数组。

[wren_value.h:402-408](src/vm/wren_value.h#L402-408) 的注释将其描述为"永不冲突但低装载因子的哈希表"——因为 symbol 是全局递增的整数，不同方法名必然映射到不同索引，不存在哈希冲突。但一个类只实现少量方法，methods 数组中大部分槽位为 `METHOD_NONE`，空间利用率低。

**时空权衡**：方法调用时只需 `classObj->methods.data[symbol]` 一次数组索引，O(1) 查找，极快。代价是每个类的 methods 数组长度等于全局 symbol 的最大值 +1，对于定义了很多方法名的系统来说，单个类的 methods 表可能很稀疏。由于 `Method` 结构体很小（一个 `MethodType` 枚举 + 一个 union），这个空间浪费在大多数场景下是可接受的。

### 4.2 CALL_n 与 SUPER_n 的差异

[wren_vm.c:982-1034](src/vm/wren_vm.c#L982-1034)：

- **CALL_n**：从栈上的接收者（`args[0]`）动态获取类：`classObj = wrenGetClassInline(vm, args[0])`（[wren_vm.c:1005](src/vm/wren_vm.c#L1005)）。
- **SUPER_n**：从常量表中读取超类：`classObj = AS_CLASS(fn->constants.data[READ_SHORT()])`（[wren_vm.c:1033](src/vm/wren_vm.c#L1033)）。这确保了方法分派从超类开始查找，而不是从当前 `this` 的类开始。

两者之后都跳转到 `completeCall` 标签共享后续逻辑。

### 4.3 四类方法的调用路径

[wren_vm.c:1045-1091](src/vm/wren_vm.c#L1045-1091)：

1. **METHOD_PRIMITIVE**：直接调用 `method->as.primitive(vm, args)`。若返回 `true`，结果在 `args[0]`，栈顶调整 `fiber->stackTop -= numArgs - 1`；若返回 `false`，表示发生了 fiber 切换或帧变化，需要 `STORE_FRAME()` → 刷新 fiber → `LOAD_FRAME()`。

2. **METHOD_FUNCTION_CALL**：处理 `Fn.call` 的特殊原语。先 `checkArity` 检查参数数量，然后 `STORE_FRAME()` → 调用原语 → `LOAD_FRAME()`。与普通 primitive 不同，它总是需要帧同步。

3. **METHOD_FOREIGN**：调用 `callForeign(vm, fiber, method->as.foreign, numArgs)`（[wren_vm.c:384-397](src/vm/wren_vm.c#L384-397)）。设置 `vm->apiStack` 指向参数区，调用 C 函数，然后恢复栈顶。

4. **METHOD_BLOCK**：`STORE_FRAME()` → `wrenCallFunction(vm, fiber, method->as.closure, numArgs)` 推入新帧 → `LOAD_FRAME()`。这是最重的调用路径，涉及帧分配和栈扩展。

### 4.4 方法缺失处理

[wren_vm.c:1038-1043](src/vm/wren_vm.c#L1038-1043)：如果 `symbol >= classObj->methods.count` 或 `method->type == METHOD_NONE`，调用 `methodNotFound`（[wren_vm.c:438-442](src/vm/wren_vm.c#L438-442)），设置 `fiber->error` 为格式化的错误字符串，然后 `RUNTIME_ERROR()`。

---

## 5. Fiber 协程与错误传播

### 5.1 Fiber 的独立栈与帧

每个 `ObjFiber`（[wren_value.h:316-355](src/vm/wren_value.h#L316-355)）拥有：
- `stack` / `stackTop` / `stackCapacity`：独立的值栈
- `frames` / `numFrames` / `frameCapacity`：独立的调用帧数组
- `openUpvalues`：该 fiber 的开放上值链表
- `caller`：调用该 fiber 的父 fiber
- `error`：运行时错误对象
- `state`：`FIBER_TRY` / `FIBER_ROOT` / `FIBER_OTHER`

这使得每个 Fiber 的执行状态完全隔离，可以独立暂停和恢复。

### 5.2 协程驱动方式

- **call**：通过 `Fiber.call()` 原语，将当前 fiber 设为被调 fiber 的 `caller`，切换执行。
- **try**：通过 `Fiber.try()` 原语，与 call 类似但将被调 fiber 的 `state` 设为 `FIBER_TRY`，表示错误可被捕获。
- **yield**：通过 `Fiber.yield()` 原语，将控制权交回 `caller`，当前 fiber 的 `numFrames > 0` 表示未完成。
- **transfer**：通过 `Fiber.transfer()` 原语，直接切换到目标 fiber，不设置 `caller` 关系。

### 5.3 CODE_RETURN 的过程

[wren_vm.c:1215-1257](src/vm/wren_vm.c#L1215-1257)：

1. `POP()` 取返回值到 `result`。
2. `fiber->numFrames--` 弹出当前帧。
3. `closeUpvalues(fiber, stackStart)` 关闭当前帧范围内的开放上值。
4. **如果帧数为 0**（fiber 完成）：
   - 若 `caller == NULL`：将结果存入 `fiber->stack[0]`，返回 `WREN_RESULT_SUCCESS`。
   - 否则：切换到 `caller` fiber，将结果存入 `caller->stackTop[-1]`。
5. **如果帧数 > 0**（普通函数返回）：
   - `stackStart[0] = result`：结果放在调用者期望的位置。
   - `fiber->stackTop = frame->stackStart + 1`：收缩栈。
6. `LOAD_FRAME()` 刷新局部变量。

### 5.4 runtimeError 与错误传播

[wren_vm.c:403-434](src/vm/wren_vm.c#L403-434)：

```c
static void runtimeError(WrenVM* vm)
{
  ObjFiber* current = vm->fiber;
  Value error = current->error;

  while (current != NULL)
  {
    current->error = error;

    if (current->state == FIBER_TRY)
    {
      current->caller->stackTop[-1] = vm->fiber->error;
      vm->fiber = current->caller;
      return;
    }

    ObjFiber* caller = current->caller;
    current->caller = NULL;
    current = caller;
  }

  wrenDebugPrintStackTrace(vm);
  vm->fiber = NULL;
  vm->apiStack = NULL;
}
```

沿 `caller` 链逐层向上：
- 每个经过的 fiber 都被设置 `error` 并断开 `caller` 链（因为不会再恢复）。
- 如果遇到 `state == FIBER_TRY` 的 fiber，说明该 fiber 是通过 `try()` 调用的，错误可被捕获。将错误值放入 `caller` 的栈顶（作为 `try` 的返回值），切换到 `caller` 继续执行。
- 如果整条链都没有 `FIBER_TRY`，则打印堆栈跟踪，将 `vm->fiber` 设为 `NULL`，解释器退出。

**FIBER_TRY 与其他状态的区别**：`FIBER_TRY` 表示该 fiber 的调用者愿意捕获错误；`FIBER_ROOT` 是初始 fiber；`FIBER_OTHER` 是普通调用。只有 `FIBER_TRY` 会在 `runtimeError` 中被特殊处理。

---

## 6. 垃圾回收

### 6.1 三色标记清除的实现

Wren 使用简化的三色标记清除算法，用 `isDark` 布尔字段代替三色：

- **白色**（未标记）：`isDark == false`，表示未被访问。
- **灰色**（已标记但子对象未遍历）：`isDark == true` 且在 `vm->gray` 栈中。
- **黑色**（已标记且子对象已遍历）：`isDark == true` 且已从 gray 栈弹出。

### 6.2 根集合

[wren_vm.c:125-170](src/vm/wren_vm.c#L125-170) 中 `wrenCollectGarbage` 标记以下根：

1. **modules 映射**：`wrenGrayObj(vm, (Obj*)vm->modules)`（[wren_vm.c:146](src/vm/wren_vm.c#L146)）
2. **临时根**：`vm->tempRoots[0..numTempRoots-1]`（[wren_vm.c:149-152](src/vm/wren_vm.c#L149-152)）
3. **当前 fiber**：`wrenGrayObj(vm, (Obj*)vm->fiber)`（[wren_vm.c:155](src/vm/wren_vm.c#L155)）
4. **句柄链表**：遍历 `vm->handles`，对每个 `handle->value` 调用 `wrenGrayValue`（[wren_vm.c:158-163](src/vm/wren_vm.c#L158-163)）
5. **编译器**：`wrenMarkCompiler(vm, vm->compiler)`（[wren_vm.c:166](src/vm/wren_vm.c#L166)）
6. **方法名符号表**：`wrenBlackenSymbolTable(vm, &vm->methodNames)`（[wren_vm.c:169](src/vm/wren_vm.c#L169)）

### 6.3 isDark 与 gray 栈驱动传播

`wrenGrayObj`（[wren_value.c:976-997](src/vm/wren_value.c#L976-997)）：
1. 若 `obj == NULL` 或 `obj->isDark == true`，直接返回（避免循环）。
2. 设 `obj->isDark = true`（标记为灰色）。
3. 将 `obj` 压入 `vm->gray` 栈。

`wrenBlackenObjects`（[wren_value.c:1219-1227](src/vm/wren_value.c#L1219-1227)）：
```c
while (vm->grayCount > 0)
{
  Obj* obj = vm->gray[--vm->grayCount];
  blackenObject(vm, obj);
}
```

不断从 gray 栈弹出对象，调用 `blackenObject`（[wren_value.c:1193-1217](src/vm/wren_value.c#L1193-1217)），根据 `obj->type` 分派到对应的 `blacken*` 函数，将子对象标灰。当 gray 栈为空时，所有可达对象都已变黑。

### 6.4 清除阶段

[wren_vm.c:176-193](src/vm/wren_vm.c#L176-193)：

```c
Obj** obj = &vm->first;
while (*obj != NULL)
{
  if (!((*obj)->isDark))
  {
    Obj* unreached = *obj;
    *obj = unreached->next;
    wrenFreeObj(vm, unreached);
  }
  else
  {
    (*obj)->isDark = false;   // 重置为白色，为下次 GC 做准备
    obj = &(*obj)->next;
  }
}
```

遍历全局对象链表 `vm->first`，回收 `isDark == false`（白色）的对象，保留 `isDark == true`（黑色）的对象并重置其 `isDark` 为 `false`。

### 6.5 GC 触发与防重入

`wrenReallocate`（[wren_vm.c:213-237](src/vm/wren_vm.c#L213-237)）：

```c
vm->bytesAllocated += newSize - oldSize;

#if WREN_DEBUG_GC_STRESS
  if (newSize > 0) wrenCollectGarbage(vm);
#else
  if (newSize > 0 && vm->bytesAllocated > vm->nextGC) wrenCollectGarbage(vm);
#endif
```

- 每次分配/释放都更新 `bytesAllocated`。
- 当 `bytesAllocated > nextGC` 且正在分配新内存（`newSize > 0`）时触发 GC。
- `WREN_DEBUG_GC_STRESS` 模式下每次分配都触发 GC，用于测试。
- **防重入**：GC 过程中会调用 `wrenReallocate` 来释放内存（`newSize == 0`），此时 `newSize > 0` 为假，不会再次触发 GC。GC 结束后重新计算 `nextGC`（[wren_vm.c:197-198](src/vm/wren_vm.c#L197-198)）：`nextGC = bytesAllocated + bytesAllocated * heapGrowthPercent / 100`，且不低于 `minHeapSize`。

---

## 7. 综合与不变量

### 7.1 完整链路示例

考虑以下 Wren 代码：

```wren
var fn
{
  var x = 42
  fn = Fn.new { x }
}
fn.call()
```

**编译阶段**：

1. 编译外层块时，`x` 是局部变量（slot 1）。
2. 编译内层函数 `{ x }` 时，`findUpvalue` 在外层 Compiler 中找到 `x`（`resolveLocal` 返回 slot 1），标记 `x.isUpvalue = true`，调用 `addUpvalue(inner, isLocal=true, index=1)`。
3. `endCompiler(inner)` 生成：
   - `CODE_CLOSURE constant_idx`（内层函数在常量表中的索引）
   - `(1, 1)`（isLocal=1, index=1，表示捕获外层的第 1 个局部变量）

**运行阶段**：

1. 执行 `CODE_CLOSURE`：创建 `ObjClosure`，读取操作数 `(isLocal=1, index=1)`，调用 `captureUpvalue(vm, fiber, frame->stackStart + 1)`。此时 `x` 在栈上，创建开放上值 `ObjUpvalue { .value → 栈slot1 }`，插入 `fiber->openUpvalues` 链表。
2. 闭包被赋值给模块变量 `fn`。
3. 外层块结束，`discardLocals` 为 `x` 发射 `CODE_CLOSE_UPVALUE`。
4. 执行 `CODE_CLOSE_UPVALUE`：调用 `closeUpvalues(fiber, fiber->stackTop - 1)`，将 `x` 的值 42 从栈复制到 `upvalue->closed`，`upvalue->value` 改为指向 `&upvalue->closed`，从开放链表移除。
5. 调用 `fn.call()`：通过 `METHOD_FUNCTION_CALL` → `METHOD_BLOCK` 路径，推入新帧执行闭包。
6. 闭包体执行 `CODE_LOAD_UPVALUE 0`：读取 `frame->closure->upvalues[0]->value`，即 `&upvalue->closed`，得到 42。

**GC 阶段**：

1. 根集合中的 `vm->modules` → 模块变量 `fn` → `ObjClosure`。
2. `blackenClosure`：标灰 `closure->fn` 和 `closure->upvalues[0]`（`ObjUpvalue`）。
3. `blackenUpvalue`：标灰 `upvalue->closed`（值为 42，是数字，不需要进一步追踪）。
4. 闭包和上值都被标记为存活，不会被回收。

### 7.2 最微妙的关键不变量

#### 不变量 1：开放上值链表必须按栈位置降序排列

**代码依据**：`captureUpvalue`（[wren_vm.c:244-283](src/vm/wren_vm.c#L244-283)）在插入新上值时遍历链表跳过 `value > local` 的节点，在正确位置插入以保持降序。`closeUpvalues`（[wren_vm.c:287-301](src/vm/wren_vm.c#L287-301)）依赖此顺序：它从链头开始，只要 `openUpvalues->value >= last` 就连续关闭。如果链表无序，`closeUpvalues` 可能遗漏应该关闭的上值，导致悬挂指针。

**违反后果**：开放上值指向已出栈的栈位置，读取到错误值或导致内存不安全。

#### 不变量 2：GC 期间栈/帧缓存与 fiber 状态必须一致

**代码依据**：`RUNTIME_ERROR()` 宏（[wren_vm.c:867-876](src/vm/wren_vm.c#L867-876)）先 `STORE_FRAME()` 把 `ip` 写回帧，再调用 `runtimeError`（可能触发 GC），然后 `LOAD_FRAME()` 刷新。`CODE_RETURN`（[wren_vm.c:1215-1257](src/vm/wren_vm.c#L1215-1257)）在弹出帧后立即 `LOAD_FRAME()`。如果 GC 发生时 `ip` 未写回帧，`blackenFiber`（[wren_value.c:1055-1085](src/vm/wren_value.c#L1055-1085)）遍历栈和帧时，帧中的 `ip` 可能指向过时位置，导致后续恢复执行时指令指针错误。

**违反后果**：GC 后恢复执行时 `ip` 错误，执行错误指令，可能崩溃。

#### 不变量 3：方法符号表索引与全局符号表的对应关系

**代码依据**：`wrenBindMethod`（[wren_value.c](src/vm/wren_value.c) 中声明，[wren_vm.c:348-382](src/vm/wren_vm.c#L348-382) 中 `bindMethod` 调用）将方法以 `symbol` 索引存入 `classObj->methods.data[symbol]`。`CALL_n` 指令的 `symbol` 操作数来自编译期 `wrenSymbolTableEnsure`（[wren_vm.h:113](src/vm/wren_vm.h#L113) 的 `methodNames`）。如果 `classObj->methods.count <= symbol`，则访问越界。`completeCall` 中的检查（[wren_vm.c:1038-1039](src/vm/wren_vm.c#L1038-1039)）先判断 `symbol >= classObj->methods.count`，此时视为 `METHOD_NONE`。

**违反后果**：如果类的 methods 数组未扩展到足够大小，方法调用会越界访问内存。Wren 通过 `wrenBindMethod`（[wren_value.c](src/vm/wren_value.c)）自动扩展 methods 数组来保证这一点，但在引导阶段（核心类初始化）需要格外小心。
