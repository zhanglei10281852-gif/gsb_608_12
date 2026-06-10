# Wren VM 内核深度分析

> 本文档是对 Wren 脚本语言虚拟机内核的静态源码分析，所有结论均锚定到具体文件/函数/宏/字段。

---

## 图示

### 图一：NaN Tagging 位布局图

```
63  62       52 51                                                    0
┌──┬───────────┬───────────────────────────────────────────────────────┐
│ S│  Exponent │                     Mantissa                          │
└──┴───────────┴───────────────────────────────────────────────────────┘
  1    11 bits                          52 bits

┌───────────────────────────────────────────────────────────────────────┐
│                     Normal double (IS_NUM = true)                     │
│   任意数值：指数位不全为 1（非 NaN）                                    │
│   ⇒ 直接作为 double 使用，零开销                                        │
└───────────────────────────────────────────────────────────────────────┘

┌──┬───────────┬──┬────────────────────────────────────────────────────┐
│ 0│ 11111111111│ 1│              Tag (3 bits)                         │
└──┴───────────┴──┴────────────────────────────────────────────────────┘
  ↑    QNAN     ↑最高尾数位
  SIGN_BIT=0   └─ 安静 NaN 标志
  Singleton
  TAG_NULL=1, TAG_FALSE=2, TAG_TRUE=3, TAG_UNDEFINED=4

┌──┬───────────┬──┬────────────────────────────────────────────────────┐
│ 1│ 11111111111│ 1│           Object pointer (51 bits)                 │
└──┴───────────┴──┴────────────────────────────────────────────────────┘
  ↑    QNAN     ↑
  SIGN_BIT=1
  Heap Object Pointer
  (64位平台实际只用48位地址，绰绰有余)
```

**关键掩码定义**（[wren_value.h L552-L595](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L552-L595)）：

| 宏 | 值（十六进制） | 作用 |
|---|---|---|
| `SIGN_BIT` | `1 << 63` | 最高位，区分对象指针(1)与单例值(0) |
| `QNAN` | `0x7ffc000000000000` | 指数位全1 + 最高尾数位=1，即安静 NaN |
| `MASK_TAG` | `7` | 低3位掩码，提取单例类型标签 |

---

### 图二：闭包上值生命周期图

```
编译期 (Compiler Chain)
┌───────────────────────────────────────────────────────────┐
│  Outer Compiler  ──parent──►  Inner Compiler              │
│  ┌──────────┐            ┌──────────┐                    │
│  │ locals[] │            │ upvalues[]│                   │
│  │ [x]      │◄──查找───  │ (isLocal=true, index=0)│     │
│  │  isUpvalue=true│       └──────────┘                    │
│  └──────────┘             addUpvalue 去重                 │
│      ▲                                                    │
│      └──── resolveLocal / findUpvalue(递归)               │
└───────────────────────────────────────────────────────────┘
          │
          ▼ 生成 CODE_CLOSURE + (isLocal, index) 操作数对

运行期 (Fiber Stack)
┌───────────────────────────────────────────────────────────┐
│  fiber->stack                                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Outer frame: [receiver, x, y, ...]                │  │
│  │                        ▲                            │  │
│  └────────────────────────┼────────────────────────────┘  │
│                           │ 开放上值                       │
│                  ┌───────────────────┐                   │
│  openUpvalues ──►│ ObjUpvalue        │                   │
│                  │ value = &stack[x] │─── 指向栈上变量    │
│                  │ closed = NULL_VAL │                   │
│                  │ next = ...        │                   │
│                  └───────────────────┘                   │
│                                                           │
│  ◄── 按栈地址从高到低排序（头=栈顶附近）                 │
│                                                           │
│  变量出作用域 → closeUpvalues() → 关闭上值：              │
│                  ┌───────────────────┐                   │
│                  │ ObjUpvalue        │                   │
│                  │ value = &closed   │─── 指向自身内部   │
│                  │ closed = <值>     │    closed 字段    │
│                  │ (从 openUpvalues 链表移除)            │
│                  └───────────────────┘                   │
│                                                           │
│  ObjClosure.upvalues[i]  ──► ObjUpvalue (共享)          │
└───────────────────────────────────────────────────────────┘

GC 标记期
┌───────────────────────────────────────────────────────────┐
│  根: ObjClosure (被栈/其他对象引用)                       │
│       │                                                   │
│       ▼ blackenClosure                                    │
│  closure->fn  (gray)                                      │
│  closure->upvalues[i] (gray)                              │
│       │                                                   │
│       ▼ blackenUpvalue                                    │
│  upvalue->closed (gray if is obj)                         │
│                                                           │
│  根: ObjFiber                                             │
│       ▼ blackenFiber                                      │
│  栈槽 (grayValue)                                         │
│  openUpvalues 链表 (grayObj)                              │
│  frames[i].closure (grayObj)                              │
│  caller (grayObj)                                         │
└───────────────────────────────────────────────────────────┘
```

---

## 1. 值表示与 NaN Tagging

### 1.1 设计目标

Wren 是动态类型语言，每个变量可以是任意类型。核心类型 `Value` 需要在单个存储单元中同时表示：数字（double）、单例值（null/true/false/undefined）、堆对象指针。

NaN tagging 技术利用 IEEE 754 双精度浮点数中大量的 NaN 位模式，将非数字值"塞"进这些本无意义的位组合里。

### 1.2 核心宏详解

所有宏定义在 [wren_value.h L552-L595](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L552-L595)。

**SIGN_BIT** (`(uint64_t)1 << 63`)
- 最高位（第63位）。
- 对于 NaN 编码的值：SIGN_BIT=1 表示对象指针，SIGN_BIT=0 表示单例值。

**QNAN** (`0x7ffc000000000000`)
- Quiet NaN 掩码。指数位（62-52位）全1，且最高尾数位（第51位）为1。
- 只要 `(value & QNAN) == QNAN`，这个值就是 NaN（不是正常数字）。
- 注意 QNAN 同时设置了指数位和最高尾数 Quiet 位。

**MASK_TAG** (`7`)
- 低3位掩码。用于从单例值中提取类型标签。

**TAG_\\*** (TAG_NULL=1, TAG_FALSE=2, TAG_TRUE=3, TAG_UNDEFINED=4)
- 单例值的类型标签，占低3位。

### 1.3 判定与还原宏

**IS_NUM(value)** → `((value) & QNAN) != QNAN`
- 如果不是 quiet NaN，那就是正常数字。
- 这是最常走的路径，只需一次位与比较。

**IS_OBJ(value)** → `((value) & (QNAN | SIGN_BIT)) == (QNAN | SIGN_BIT)`
- 必须同时是 QNAN 且 SIGN_BIT 为 1。
- 两者都满足才是堆对象指针。

**AS_OBJ(value)** → `(Obj*)(uintptr_t)((value) & ~(SIGN_BIT | QNAN))`
- 清除 SIGN_BIT 和 QNAN 位，剩下的就是地址位。
- 64位平台实际只用48位地址，51位绰绰有余。

**AS_BOOL(value)** → `(value) == TRUE_VAL`
- 直接与常量 `TRUE_VAL` 比较。
- 因为 NaN tagged 的单例值有唯一的位模式，所以直接比较就行。

**AS_NUM** / **NUM_VAL**
- 数字值"零开销"：`wrenValueToNum` 和 `wrenNumToValue` 直接通过 `wrenDoubleFromBits` / `wrenDoubleToBits` 做类型双关转换（[wren_math.h L20-L32](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_math.h#L20-L32)）。
- 因为正常 double 的位模式在 NaN tagging 下不做任何修改，所以数字的算术运算就是原生 double 运算，完全零开销。

### 1.4 单例值常量

```c
#define NULL_VAL      ((Value)(uint64_t)(QNAN | TAG_NULL))      // 0x7ffc000000000001
#define FALSE_VAL     ((Value)(uint64_t)(QNAN | TAG_FALSE))     // 0x7ffc000000000002
#define TRUE_VAL      ((Value)(uint64_t)(QNAN | TAG_TRUE))      // 0x7ffc000000000003
#define UNDEFINED_VAL ((Value)(uint64_t)(QNAN | TAG_UNDEFINED)) // 0x7ffc000000000004
```

注意 TAG_NAN=0 对应真正的 NaN 双精度值（`0x7ffc000000000000`），但这个值在 Wren 中被当作数字（因为 IS_NUM 会返回 false？不——QNAN本身 IS_NUM 返回 false，但它实际是 NaN，仍被当作 num 类型处理）。看 [wrenGetClassInline](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.h#L209-L237)，TAG_NAN 返回 `vm->numClass`，说明 NaN 值在 Wren 中仍是数字类型。

### 1.5 关闭 NaN Tagging 的 struct 表示

当 `WREN_NAN_TAGGING` 为 0 时，改用结构体表示（[wren_value.h L127-L145](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L127-L145)）：

```c
typedef enum { VAL_FALSE, VAL_NULL, VAL_NUM, VAL_TRUE, VAL_UNDEFINED, VAL_OBJ } ValueType;

typedef struct {
    ValueType type;   // 类型标签
    union {
        double num;   // 数字值
        Obj* obj;     // 对象指针
    } as;
} Value;
```

**权衡分析**：
- **调试友好**：可以在调试器里直接看到 `type` 字段，直观知道值的类型。NaN tagged 版本只能看到一串十六进制。
- **可移植性更好**：不依赖 NaN 位模式的细节（虽然 IEEE 754 是标准，但某些特殊平台可能有差异）。
- **体积更大**：16字节（enum 4字节 + 对齐 + union 8字节）vs NaN tagging 的 8字节，整整大一倍。
- **速度更慢**：每次类型判定都要访问 struct 的 type 字段，再访问 union 数据，有两次内存访问且可能破坏缓存局部性。NaN tagging 只需一次位运算。
- **算术运算有开销**：每次都要先检查 `type == VAL_NUM`，再取 `as.num`。

---

## 2. 字节码与分派

### 2.1 X-Macro 定义 opcode

[wren_opcodes.h](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_opcodes.h) 使用 X-Macro 模式定义所有 opcode：

```c
OPCODE(CONSTANT, 1)    // 压入一个常量，栈效应 +1
OPCODE(NULL, 1)        // 压入 null，栈效应 +1
OPCODE(CALL_0, 0)      // 调用0参数方法，栈效应 0（receiver保留，返回值替换）
OPCODE(RETURN, 0)      // 返回，栈效应 0
// ... 更多
```

`OPCODE(name, stackEffect)` 宏在被包含时由调用方定义。这样同一份定义可以生成多份代码：
- 在 [wren_vm.h L15-L18](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.h#L15-L18) 中生成 `Code` 枚举（`CODE_CONSTANT`, `CODE_NULL` 等）。
- 在 `runInterpreter` 的 computed goto 模式中生成 `dispatchTable` 标签地址表。
- 在 switch 模式中生成 case 分支。

### 2.2 栈效应（Stack Effect）

每个 opcode 的第二个参数是"栈效应"——执行这条指令后栈大小的净变化量：
- **正数**：栈增长（压入值）。如 `CONSTANT` 压入一个常量，效应 +1。
- **负数**：栈缩小（弹出值）。如 `CALL_3` 弹出3个参数，效应 -3（receiver 保留作为返回值槽）。
- **零**：栈大小不变。如 `RETURN` 弹出当前帧但压入返回值，净效应为0。

栈效应主要用于文档和理解，不直接用于运行时计算（实际栈操作由每条指令的实现代码管理）。

### 2.3 两套分派机制

`runInterpreter` 函数（[wren_vm.c L826-L1370](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L826-L1370)）支持两种指令分派方式，由 `WREN_COMPUTED_GOTO` 宏控制。

#### Computed Goto（推荐/默认，GCC/Clang）

使用"标签作为值"（Labels as Values）GCC 扩展：

```c
static void* dispatchTable[] = {
    #define OPCODE(name, _) &&code_##name,
    #include "wren_opcodes.h"
    #undef OPCODE
};

#define DISPATCH()  goto *dispatchTable[instruction = (Code)READ_BYTE()]
#define CASE_CODE(name) code_##name
```

每个 opcode 对应一个代码标签（如 `code_CONSTANT`），`dispatchTable` 存储这些标签的地址。分派时读取 opcode，直接跳转到对应的代码地址。

**优势**：
- 比 switch 少一次分支预测失败的开销。
- 每个 opcode 的分派是直接跳转，不是经过 switch 的统一跳转。
- 对指令缓存更友好。

#### Switch（MSVC 等不支持 computed goto 的编译器）

```c
#define INTERPRET_LOOP  loop: DEBUG_TRACE_INSTRUCTIONS(); switch (instruction = (Code)READ_BYTE())
#define CASE_CODE(name) case CODE_##name
#define DISPATCH()     goto loop
```

传统的 switch 循环。每次分派都回到 switch 的顶部，由编译器生成跳转表。

**劣势**：
- 所有 opcode 共享同一个跳转点，分支预测压力大。
- 通常比 computed goto 慢 10-20%。

### 2.4 opcode 顺序与性能

[wren_opcodes.h 注释](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_opcodes.h#L10-L13) 明确指出：

> Note that the order of instructions here affects the order of the dispatch table in the VM's interpreter loop. That in turn affects caching which affects overall performance.

原因：
- dispatchTable 按 opcode 顺序排列，执行频率高的 opcode 排在前面可以提高缓存命中率。
- 对于 switch 分派，编译器生成的跳转表也是按顺序的，同样影响缓存。
- 相近的 opcode（如 LOAD_LOCAL_0 到 LOAD_LOCAL_8，CALL_0 到 CALL_16）排列在一起可以利用空间局部性。

### 2.5 LOAD_FRAME / STORE_FRAME 帧缓存

`runInterpreter` 把频繁访问的调用帧字段缓存到局部变量中（[wren_vm.c L835-L862](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L835-L862)）：

```c
register CallFrame* frame;
register Value* stackStart;
register uint8_t* ip;
register ObjFn* fn;
```

**STORE_FRAME()** 把局部 `ip` 写回 `frame->ip`，在调用/返回/GC 之前执行。

**LOAD_FRAME()** 从 `fiber->frames` 的顶帧重新加载所有局部变量：
- `frame = &fiber->frames[fiber->numFrames - 1]`
- `stackStart = frame->stackStart`
- `ip = frame->ip`
- `fn = frame->closure->fn`

**为什么这么做？**
- 这些变量在指令执行中被极其频繁地访问（每次 READ_BYTE、PUSH、POP 都用）。
- 放在局部变量（最好是寄存器变量 `register`）里比每次都从内存读取快得多。
- 但调用函数、切换调用帧、发生 GC 时，这些缓存可能失效，必须先 STORE_FRAME 同步回内存，操作完成后再 LOAD_FRAME 重新加载。

**同步时机**（在 [wren_vm.c](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c) 中搜索 STORE_FRAME/LOAD_FRAME）：
- 调用方法前：`STORE_FRAME()` → `wrenCallFunction()` → `LOAD_FRAME()`（[L1082-L1084](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1082-L1084)）。
- 运行时错误：`STORE_FRAME()` → `runtimeError()` → `LOAD_FRAME()`（[L870-L875](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L870-L875)）。
- 执行原语后如果 fiber 切换了：`STORE_FRAME()` → fiber 可能变了 → `LOAD_FRAME()`（[L1055-L1062](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1055-L1062)）。
- RETURN 指令后：`LOAD_FRAME()`（[L1255](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1255)）。
- GC 安全点：因为 GC 会遍历栈和帧，所以触发 GC 的分配操作必须在帧状态已同步到内存时进行。

---

## 3. 闭包与上值捕获

### 3.1 编译期：resolveUpvalue / addUpvalue

Wren 编译器是单遍的，使用 `Compiler` 结构体的 `parent` 指针形成嵌套链（[wren_compiler.c L320-L378](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_compiler.c#L320-L378)）。

当内层函数引用外层局部变量时，通过 `findUpvalue` 函数递归解析（[wren_compiler.c L1577-L1612](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_compiler.c#L1577-L1612)）：

**查找过程**（`findUpvalue`）：
1. 如果父编译器为空（到达顶层），返回 -1（没找到）。
2. 如果遇到方法边界且不是字段（`_` 开头），停止查找（方法不捕获外部局部变量）。
3. 先在**直接外层**的 locals 中查找（`resolveLocal`）：
   - 找到的话，标记该局部 `isUpvalue = true`（因为它被闭包捕获了，出作用域时需要关闭）。
   - 调用 `addUpvalue(compiler, true, local)` 登记为上值，返回索引。
4. 如果外层没有，**递归**查找更外层（`findUpvalue(compiler->parent, ...)`）：
   - 找到的话，调用 `addUpvalue(compiler, false, upvalue)`——这是一个"间接"上值，通过外层的上值来引用。
   - 这实现了闭包的**扁平化**（flattening）：中间层函数也会获得相应的上值。

**去重登记**（`addUpvalue`，[wren_compiler.c L1551-L1564](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_compiler.c#L1551-L1564)）：
- 遍历已有的 upvalues 数组，检查 `isLocal` 和 `index` 是否相同。
- 相同则直接返回已有索引，避免重复捕获同一个变量。
- 不同则添加新条目，返回新索引。

**CompilerUpvalue 结构**（[wren_compiler.c L227-L235](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_compiler.c#L227-L235)）：
- `isLocal`：true 表示捕获的是外层的局部变量，false 表示捕获的是外层的上值。
- `index`：局部变量索引或上值索引。

### 3.2 CODE_CLOSURE 指令与运行期填充

编译器遇到函数字面量时，生成 `CODE_CLOSURE` 指令，后面跟：
- 一个 16 位常量索引（指向常量池中的 ObjFn）。
- 然后是 N 对操作数，每对两个字节：`(isLocal, index)`，N = `fn->numUpvalues`。

运行期由 `runInterpreter` 中的 `CODE_CLOSURE` 分支处理（[wren_vm.c L1270-L1296](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1270-L1296)）：

```c
CASE_CODE(CLOSURE): {
    ObjFn* function = AS_FN(fn->constants.data[READ_SHORT()]);
    ObjClosure* closure = wrenNewClosure(vm, function);
    PUSH(OBJ_VAL(closure));
    
    for (int i = 0; i < function->numUpvalues; i++) {
        uint8_t isLocal = READ_BYTE();
        uint8_t index = READ_BYTE();
        if (isLocal) {
            // 捕获父帧的局部变量 → 创建/复用开放上值
            closure->upvalues[i] = captureUpvalue(vm, fiber,
                                                  frame->stackStart + index);
        } else {
            // 复用当前闭包已有的上值（间接捕获）
            closure->upvalues[i] = frame->closure->upvalues[index];
        }
    }
    DISPATCH();
}
```

关键点：
- **先 push 再填 upvalues**：防止闭包在创建 upvalues 过程中被 GC 回收。
- **captureUpvalue** 负责去重和维护有序链表。

### 3.3 开放上值 vs 关闭上值

**ObjUpvalue 结构**（[wren_value.h L178-L195](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L178-L195)）：

```c
typedef struct sObjUpvalue {
    Obj obj;           // 对象头（因为是 GC 管理的）
    Value* value;      // 指向当前值的位置（栈上 或 &closed）
    Value closed;      // 关闭后的值存储在这里
    struct sObjUpvalue* next; // 开放上值链表的下一个
} ObjUpvalue;
```

**开放上值（Open Upvalue）**：
- `value` 指向栈上的局部变量。
- 还在 `fiber->openUpvalues` 链表中。
- 变量出作用域之前都是开放的。

**关闭上值（Closed Upvalue）**：
- `value` 指向 `&upvalue->closed`（自身内部的存储）。
- 已经从 `openUpvalues` 链表中移除。
- 值被"提升"（hoist）出栈，生命周期不再受栈帧限制。

### 3.4 openUpvalues 链表与 closeUpvalues

**链表特性**（[wren_vm.c L244-L283](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L244-L283)）：
- 头节点是栈位置最高的（最靠近栈顶）。
- 链表按栈地址**从高到低**排序。
- 这样设计是因为变量出作用域时总是从栈顶方向开始关闭，刚好可以从链表头开始处理。

**captureUpvalue**（创建/复用开放上值）：
1. 如果没有开放上值，直接创建新的作为头节点。
2. 否则从头部（栈顶方向）往下遍历：
   - 找到地址相同的 → 返回已有上值（去重）。
   - 找到地址比目标小的 → 说明应该插在它前面，保持有序。
3. 创建新上值，插入到正确位置。

**closeUpvalues(fiber, last)**（[wren_vm.c L287-L300](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L287-L300)）：
- 关闭所有栈地址 >= last 的开放上值。
- 因为链表是从高到低排的，所以从头部开始连续处理即可。
- 每个被关闭的上值：
  1. `upvalue->closed = *upvalue->value`（把栈上的值复制进去）。
  2. `upvalue->value = &upvalue->closed`（指针改指向内部）。
  3. 从链表移除。

**触发关闭的时机**：
- `CODE_CLOSE_UPVALUE` 指令：单个局部变量出作用域时（[wren_vm.c L1209-L1213](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1209-L1213)）。
- `CODE_RETURN` 指令：整个函数返回，关闭该帧所有上值（[wren_vm.c L1221](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1221)）。

---

## 4. 方法分派

### 4.1 符号表索引的方法表

**全局方法符号表**（[wren_vm.h L112-L113](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.h#L112-L113)）：
- `vm->methodNames` 是一个 `SymbolTable`（本质是 `StringBuffer`，字符串数组）。
- 所有类的所有方法名共享这一张全局符号表。
- 每个方法名对应一个唯一的整数索引（符号）。

**类的方法表**（[wren_value.h L392-L416](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L392-L416)）：
- `ObjClass.methods` 是 `MethodBuffer`（动态数组）。
- 方法直接用**符号索引**作为数组下标来访问。
- 注释称为"永不冲突但低装载因子的哈希表"。

**wrenBindMethod**（[wren_value.c L120-L132](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L120-L132)）：
- 如果 symbol 超过当前 methods 数组大小，就用 `METHOD_NONE` 填充扩容。
- 然后直接 `methods.data[symbol] = method`。

**时空权衡**：
- **时间**：方法调用是 O(1) 数组索引，没有哈希计算、没有冲突探测。极快。
- **空间**：每个类的 methods 数组大小等于最大方法符号索引+1，中间可能有大量空洞（METHOD_NONE）。装载因子低。
- 因为 Method 本身很小（type 枚举 + 一个指针联合体），空间浪费是可接受的。

### 4.2 CALL_n 与 SUPER_n

两类调用指令共享大部分代码，用 `goto completeCall` 汇合（[wren_vm.c L982-L1091](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L982-L1091)）。

**CALL_n**（普通方法调用）：
- `numArgs = instruction - CODE_CALL_0 + 1`（从 opcode 编号算出参数个数）。
- `symbol = READ_SHORT()`（方法符号，16位）。
- `args = fiber->stackTop - numArgs`（参数起始位置，第一个是 receiver）。
- `classObj = wrenGetClassInline(vm, args[0])`（取 receiver 的类）。

**SUPER_n**（超类方法调用）：
- 前半部分和 CALL 一样。
- 但 classObj 不是 receiver 的类，而是**从常量池读取的超类**（`fn->constants.data[READ_SHORT()]`）。
- 编译器在编译时就确定了 super 调用应该从哪个超类开始查找，把超类对象放进常量池。

### 4.3 四类方法的调用路径

在 `completeCall` 标签后，根据 `method->type` 分支：

**METHOD_PRIMITIVE**（原语方法，C 实现）：
- 直接调用 `method->as.primitive(vm, args)`。
- 原语函数返回 true 表示正常返回，结果在 `args[0]`；返回 false 表示出错或 fiber 切换了。
- 原语可以直接操作栈，是最底层的内置功能。

**METHOD_FUNCTION_CALL**（函数 .call 的特殊处理）：
- 先调用 `checkArity` 检查参数个数匹配。
- 然后调用原语，但用 STORE_FRAME/LOAD_FRAME 包裹（因为 .call 可能会修改调用帧？实际上这个原语就是处理 Fn 调用的）。
- 这是一种特殊的原语类型，专用于函数的 `.call` 方法。

**METHOD_FOREIGN**（外部 C 方法）：
- 调用 `callForeign(vm, fiber, method->as.foreign, numArgs)`。
- 外部方法由宿主程序注册，通过 Wren C API 交互。
- 有独立的调用约定和错误处理。

**METHOD_BLOCK**（用户定义的 Wren 方法）：
- `STORE_FRAME()` 同步当前帧状态。
- `wrenCallFunction(vm, fiber, method->as.closure, numArgs)` 压入新的调用帧。
- `LOAD_FRAME()` 重新加载新帧的缓存。
- 控制权转移到被调用的方法体。

### 4.4 方法缺失处理

如果符号索引超出 methods 数组范围，或者对应位置是 `METHOD_NONE`：
- 调用 `methodNotFound(vm, classObj, symbol)`（[wren_vm.c L438-L442](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L438-L442)）。
- 生成错误消息：`"<class> does not implement '<method>'."`。
- 然后 `RUNTIME_ERROR()` 触发运行时错误处理。

---

## 5. Fiber 协程与错误传播

### 5.1 Fiber 的独立执行上下文

每个 `ObjFiber` 都有独立的栈和调用帧（[wren_value.h L316-L355](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L316-L355)）：

```c
typedef struct sObjFiber {
    Obj obj;
    Value* stack;          // 栈数组
    Value* stackTop;       // 栈顶指针
    int stackCapacity;     // 栈容量
    CallFrame* frames;     // 调用帧数组
    int numFrames;         // 当前帧数
    int frameCapacity;     // 帧容量
    ObjUpvalue* openUpvalues; // 开放上值链表
    struct sObjFiber* caller; // 调用者 fiber
    Value error;           // 错误值（null 表示无错误）
    FiberState state;      // fiber 状态
} ObjFiber;
```

**为什么每个 Fiber 需要独立的栈和帧？**
- 协程的核心是可以暂停和恢复执行。
- 暂停时需要保留当前的调用栈、指令指针、局部变量等全部状态。
- 每个 fiber 有自己的栈，切换 fiber 就是切换 `vm->fiber` 指针。

### 5.2 Fiber 的驱动方式

四种主要操作，都在 [wren_core.c](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_core.c) 的 fiber 原语中实现：

| 操作 | 函数 | 设置 caller | state | 行为 |
|---|---|---|---|---|
| **call** | `prim_fiber_call` | ✓ 设为当前 fiber | FIBER_OTHER | 调用 fiber，完成后返回 caller |
| **try** | `prim_fiber_try` | ✓ 设为当前 fiber | **FIBER_TRY** | 类似 call，但会捕获错误 |
| **yield** | `prim_fiber_yield` | ✗（清掉 caller） | FIBER_OTHER | 暂停并返回 caller |
| **transfer** | `prim_fiber_transfer` | ✗（不设 caller） | FIBER_OTHER | 直接转移，不建立调用关系 |

核心函数 `runFiber`（[wren_core.c L86-L138](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_core.c#L86-L138)）：
- `isCall=true` 时建立 caller 关系（call 和 try 用）。
- `isCall=false` 时不建立（transfer 用）。
- fiber 首次启动时初始化参数（如果函数有参数的话）。
- 恢复时把值放到 `stackTop[-1]`（yield/transfer 的返回值槽）。
- 设置 `vm->fiber = fiber`，返回 false 告诉解释器切换了 fiber。

### 5.3 CODE_RETURN：从调用帧返回

`runInterpreter` 中的 `CODE_RETURN` 分支（[wren_vm.c L1215-L1257](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1215-L1257)）：

1. 弹出返回值：`Value result = POP()`
2. 当前帧退栈：`fiber->numFrames--`
3. 关闭该帧所有开放上值：`closeUpvalues(fiber, stackStart)`
4. **如果是 fiber 的最后一帧**（`numFrames == 0`）：
   - 如果没有 caller，fiber 完成，返回成功，结果存在 `stack[0]`。
   - 如果有 caller，切换回 caller，结果存到 caller 的栈顶 `stackTop[-1]`。
5. **如果还有帧**（普通函数返回）：
   - 返回值存到 `stackStart[0]`（receiver 槽，即调用者期望结果的位置）。
   - 栈顶调整为 `frame->stackStart + 1`（只留返回值一个槽）。
6. `LOAD_FRAME()` 重新加载当前帧缓存。

### 5.4 错误传播与 runtimeError

`runtimeError` 函数（[wren_vm.c L403-L434](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L403-L434)）沿 fiber 的 caller 链逐层传播错误：

```c
static void runtimeError(WrenVM* vm) {
    ObjFiber* current = vm->fiber;
    Value error = current->error;
    
    while (current != NULL) {
        current->error = error;  // 每层都设置 error
        
        // 找到 try 的 fiber，错误被捕获
        if (current->state == FIBER_TRY) {
            current->caller->stackTop[-1] = vm->fiber->error;
            vm->fiber = current->caller;
            return;
        }
        
        // 否则解除 caller 关系（fiber 已 abort，不会再返回）
        ObjFiber* caller = current->caller;
        current->caller = NULL;
        current = caller;
    }
    
    // 没人捕获，打印栈追踪，vm->fiber = NULL
    wrenDebugPrintStackTrace(vm);
    vm->fiber = NULL;
}
```

**FIBER_TRY 与其他状态的关键区别**：
- `FIBER_TRY`：错误会在此处被捕获。caller 的 try 调用返回错误值，执行继续。
- `FIBER_ROOT`：根 fiber，错误继续向上传播（但 root 之上没人了，就终止）。
- `FIBER_OTHER`：普通调用/转移，错误沿 caller 链继续传播。

传播过程中每层 fiber 的 `error` 字段都会被设置，且 `caller` 指针被清除（因为 fiber 已经 abort 了，不会再被恢复）。

---

## 6. 垃圾回收

### 6.1 三色标记清除实现

Wren 使用经典的**三色标记清除**（Tricolor Mark-Sweep）垃圾收集器。由 `Obj.isDark` 字段标记颜色（[wren_value.h L108-L119](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.h#L108-L119)）。

颜色对应关系：
- **白色**：`isDark = false`（未到达，可回收）。
- **灰色**：`isDark = true` 且还在 gray 栈中（已发现但未扫描其引用）。
- **黑色**：`isDark = true` 且已从 gray 栈弹出处理过（已扫描所有引用）。

注意：只用一个 `isDark` 位就实现了三色，靠的是"是否在 gray 栈中"这个隐含状态。

### 6.2 根集合

GC 从根集合开始标记。根在 `wrenCollectGarbage` 函数中被"染灰"（[wren_vm.c L125-L173](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L125-L173)）：

| 根 | 代码位置 | 说明 |
|---|---|---|
| modules map | `wrenGrayObj(vm, (Obj*)vm->modules)` | 所有已加载的模块 |
| 临时根 | `vm->tempRoots[i]` | 分配过程中临时保护的对象 |
| 当前 fiber | `wrenGrayObj(vm, (Obj*)vm->fiber)` | 当前执行的协程 |
| Handles | `vm->handles` 链表 | C API 持有的句柄 |
| 编译器 | `vm->compiler` | 如果正在编译，编译器使用的对象 |
| 方法符号表 | `wrenBlackenSymbolTable` | 全局方法名字符串 |

### 6.3 标记传播：wrenGrayObj + wrenBlackenObjects

**wrenGrayObj**（[wren_value.c L976-L997](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L976-L997)）：
1. 如果 obj 是 NULL，忽略。
2. 如果 `obj->isDark` 已经是 true，返回（防止重复处理和循环）。
3. 设置 `obj->isDark = true`（变灰）。
4. 压入 `vm->gray` 栈中。
5. gray 栈不够时自动扩容（2倍）。

**wrenBlackenObjects**（[wren_value.c L1219-L1227](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L1219-L1227)）：
- 循环弹出 gray 栈顶的对象。
- 对每个对象调用 `blackenObject`，根据类型分派到具体的 `blacken*` 函数。
- `blacken*` 函数扫描对象的所有引用，调用 `wrenGrayObj` 或 `wrenGrayValue` 标记引用的对象（变灰）。
- 处理完的对象就是黑色的了。

**各类型的 blacken 函数**（[wren_value.c L1013-L1191](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L1013-L1191)）：
- `blackenClass`：标记 metaclass、superclass、方法闭包、类名、属性。
- `blackenClosure`：标记 fn 和所有 upvalues。
- `blackenFiber`：标记所有帧的 closure、栈上所有值、openUpvalues 链表、caller、error。
- `blackenFn`：标记常量池和 module。
- `blackenInstance`：标记 classObj 和所有字段。
- `blackenList`：标记所有元素。
- `blackenMap`：标记所有 entry 的 key 和 value。
- `blackenModule`：标记所有模块变量、变量名字符串、模块名。
- `blackenUpvalue`：标记 closed 值（如果关闭了的话）。
- `blackenString` / `blackenRange`：无引用（字符串和范围是叶子对象）。

### 6.4 清除阶段

在 `wrenCollectGarbage` 中（[wren_vm.c L176-L193](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L176-L193)）：

```c
Obj** obj = &vm->first;
while (*obj != NULL) {
    if (!((*obj)->isDark)) {
        // 白色对象：没被标记，回收
        Obj* unreached = *obj;
        *obj = unreached->next;
        wrenFreeObj(vm, unreached);
    } else {
        // 黑色对象：保留，重置 isDark 为下一次 GC 做准备
        (*obj)->isDark = false;
        obj = &(*obj)->next;
    }
}
```

- 遍历全局对象链表（`vm->first` 开始的链表，新对象插在头部）。
- 白色（`isDark=false`）的对象从链表移除并释放。
- 黑色的对象重置 `isDark = false`，变回白色，为下一次 GC 做准备。

### 6.5 内存分配与 GC 触发

**wrenReallocate**（[wren_vm.c L213-L237](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L213-L237)）是唯一的内存分配入口：

```c
void* wrenReallocate(WrenVM* vm, void* memory, size_t oldSize, size_t newSize) {
    vm->bytesAllocated += newSize - oldSize;
    
    #if WREN_DEBUG_GC_STRESS
        if (newSize > 0) wrenCollectGarbage(vm);
    #else
        if (newSize > 0 && vm->bytesAllocated > vm->nextGC) 
            wrenCollectGarbage(vm);
    #endif
    
    return vm->config.reallocateFn(memory, newSize, vm->config.userData);
}
```

**触发条件**：
- 分配新内存（`newSize > 0`）且 `bytesAllocated > nextGC` 时触发 GC。
- `nextGC` 在每次 GC 后计算：`bytesAllocated + bytesAllocated * heapGrowthPercent / 100`。
- 默认 `heapGrowthPercent = 50`（每次 GC 后堆增长 50% 再触发下一次）。
- 最小不低于 `minHeapSize`（默认 1MB）。

**防止重入**：
- GC 过程中也会调用 `wrenReallocate`（比如 gray 栈扩容、释放对象等）。
- 但是 `wrenCollectGarbage` 开头会重置 `bytesAllocated = 0`，然后在 blacken 阶段重新累加。
- GC 内部的分配（gray 栈扩容等）会增加 `bytesAllocated`，但因为 `nextGC` 在 GC 结束时才更新，所以不会重入。
- 而且 GC 运行时对象都在被标记，不会再触发回收逻辑。

**WREN_DEBUG_GC_STRESS** 模式：
- 每次分配都触发 GC，用于测试 GC 根标记的正确性。
- 开发调试时很有用，可以尽早发现 GC 相关 bug。

---

## 7. 综合与不变量

### 7.1 完整链路示例：闭包捕获变量

让我们用一小段 Wren 代码串起整个流程：

```wren
var makeCounter = Fn.new {
  var count = 0          // 外层局部变量
  Fn.new {               // 内层函数，捕获 count
    count = count + 1
    count
  }
}

var counter = makeCounter.call()
counter.call()  // 1
counter.call()  // 2
```

#### 阶段一：编译

1. 编译器遇到外层函数字面量 `Fn.new { ... }`，创建外层 Compiler。
2. 编译到 `var count = 0`，在 locals 数组添加 count（slot 0）。
3. 遇到内层 `Fn.new { ... }`，创建内层 Compiler，parent 指向上层。
4. 内层引用 `count` 时，`resolveNonmodule` → `findUpvalue`：
   - 在内层 locals 中找不到。
   - 递归调用 `findUpvalue(parent, "count", ...)`。
   - 外层 resolveLocal 找到 count（index=0）。
   - 设置 `locals[0].isUpvalue = true`。
   - `addUpvalue(innerCompiler, true, 0)` → 返回上值索引 0。
5. 内层函数编译完成，`fn->numUpvalues = 1`。
6. 生成 `CODE_CLOSURE` 指令，后跟 `(true, 0)` 操作数对。
7. 外层函数中 `count` 出作用域时生成 `CODE_CLOSE_UPVALUE`。

#### 阶段二：运行

**执行 makeCounter.call() 时**：
1. 调用外层函数，创建新调用帧，栈上分配 count=0。
2. 执行到内层函数字面量（`CODE_CLOSURE`）：
   - 从常量池取 ObjFn。
   - 创建 ObjClosure，先 push 到栈上（GC 安全）。
   - 读取操作数 `(isLocal=true, index=0)`。
   - `captureUpvalue(vm, fiber, stackStart + 0)`：
     - openUpvalues 为空，创建新 ObjUpvalue。
     - `upvalue->value = &stack[slot]`（指向栈上的 count）。
     - 插入 openUpvalues 链表头部。
     - `closure->upvalues[0] = upvalue`。
3. 外层函数返回（`CODE_RETURN`）：
   - 返回值是闭包。
   - `closeUpvalues(fiber, stackStart)`：
     - 遍历 openUpvalues，发现 `upvalue->value >= stackStart`。
     - `upvalue->closed = *upvalue->value`（复制 count 的值 0）。
     - `upvalue->value = &upvalue->closed`（改指内部）。
     - 从 openUpvalues 链表移除。
4. 闭包作为 makeCounter 的返回值交给 caller。

**执行 counter.call() 时**：
1. 调用闭包，进入内层函数。
2. 读取/写入 count 时用 `LOAD_UPVALUE` / `STORE_UPVALUE` 指令。
3. `*upvalues[index]->value` 读写的是上值内部的 closed 字段。
4. 因为上值已经关闭了，所以值持久存在。
5. 多次调用 counter 共享同一个 ObjUpvalue，所以 count 能累加。

#### 阶段三：GC 标记

假设 GC 发生在 counter 存活期间：

1. 根标记：counter 变量在栈上 → `blackenFiber` 扫描栈 → `wrenGrayValue` 发现是对象 → `wrenGrayObj` 标记 ObjClosure。
2. `blackenClosure`：
   - 标记 `closure->fn`（ObjFn）。
   - 标记 `closure->upvalues[0]`（ObjUpvalue）。
3. `blackenUpvalue`：
   - 标记 `upvalue->closed`（如果是对象的话，这里是数字 0/1/2，不用标记）。
   - 因为上值已关闭，所以它的值就在内部。
4. 整个链路：栈 → ObjClosure → ObjUpvalue → (closed 值) → 全部存活。

### 7.2 关键不变量

以下是我认为最微妙、最难跟踪的几个关键不变量，各附代码依据。

#### 不变量 1：开放上值必须按栈地址从高到低有序

**含义**：`fiber->openUpvalues` 链表中的 ObjUpvalue 按 `value` 指针降序排列（栈顶在前，栈底在后）。

**为什么重要**：
- 变量出作用域时总是从栈顶方向开始的（后声明的先销毁）。
- 有序链表让 `closeUpvalues` 可以从头部开始连续处理，是 O(k) 而不是 O(n)。
- `captureUpvalue` 去重查找也依赖有序性。

**代码依据**：
- `captureUpvalue` 的插入逻辑（[wren_vm.c L256-L282](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L256-L282)）：
  - `while (upvalue != NULL && upvalue->value > local)` 向栈底方向遍历。
  - 找到第一个 `upvalue->value == local` 说明已存在。
  - 找到 `upvalue->value < local` 说明应该插在当前位置之前。
  - 这保证了链表始终按 `value` 降序。
- `closeUpvalues` 的处理逻辑（[wren_vm.c L287-L300](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L287-L300)）：
  - `while (fiber->openUpvalues != NULL && fiber->openUpvalues->value >= last)`。
  - 直接从头部连续移除，不需要遍历整个链表。
  - 只有在有序的前提下，这种连续头部移除才是正确的。

#### 不变量 2：GC 期间栈/帧缓存与 fiber 状态必须一致

**含义**：触发 GC 时，`runInterpreter` 中缓存的局部变量（ip、stackStart、frame、fn）必须已经同步回 fiber 的内存结构中。

**为什么重要**：
- GC 标记阶段从 `vm->fiber` 出发遍历栈和调用帧。
- 如果 `ip` 还在寄存器/局部变量里，没写回 `frame->ip`，GC 不一定会出错（因为 GC 不关心 ip），但栈指针 `stackTop` 肯定必须是最新的（因为 GC 要扫描 `stack` 到 `stackTop` 之间的值）。
- 更深层的问题是：如果分配触发 GC 时，新创建的对象还没有被任何根引用，就会被错误回收。

**代码依据**：
- `STORE_FRAME` 宏（[wren_vm.c L851](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L851)）只同步 `ip`，不同步 stackTop（因为 stackTop 直接操作 fiber 的，不是缓存的）。
- 在可能触发分配的操作之前，对象会被 push 到栈上（如 `CODE_CLOSURE` 中先 PUSH 闭包再填 upvalues），确保 GC 能找到它。
- `wrenPushRoot` / `wrenPopRoot`（[wren_vm.h L199-L202](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.h#L199-L202)）使用临时根数组保护分配过程中还没被其他对象引用的新对象。
- `wrenNewClosure` 中先初始化 upvalues 数组为 NULL（[wren_value.c L144](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L144)），防止 GC 扫描到未初始化内存。

#### 不变量 3：方法符号表索引与全局符号表一一对应

**含义**：每个类的 `methods.data[i]` 对应的方法名就是 `vm->methodNames.data[i]`，索引全局一致。

**为什么重要**：
- 方法调用时，编译器把方法名解析为符号索引，存在字节码里。
- 运行时直接用这个索引到 class 的 methods 数组查找。
- 如果索引不对应，就会调用错误的方法（或者找不到方法）。
- 这是整个方法分派系统正确性的基石。

**代码依据**：
- `PRIMITIVE` 宏（[wren_primitive.h L8-L17](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_primitive.h#L8-L17)）注册原语方法时，用 `wrenSymbolTableEnsure` 从名字获得符号，再用这个符号索引绑定方法。
- `wrenBindMethod`（[wren_value.c L120-L132](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L120-L132)）直接用 symbol 作为数组下标。
- `wrenBindSuperclass` 继承方法时（[wren_value.c L80-L83](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_value.c#L80-L83)），按索引逐个复制，保持索引一致。
- `CALL_n` 指令的 `symbol` 操作数（[wren_vm.c L1001](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L1001)）直接来自编译器的符号解析结果。
- `methodNotFound` 用 `vm->methodNames.data[symbol]` 找回方法名（[wren_vm.c L440-L441](file:///e:/gsb/608/gsb_12/Winter/src/vm/wren_vm.c#L440-L441)），证明了索引的对应关系。
