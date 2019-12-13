# 一种C0栈式虚拟机

本文档给出一种栈式虚拟机的设计，目前是 C0 编译目标的标准。

## 章节目录

- 1 [概述](#概述)
- 2 [虚拟机结构](#虚拟机结构)
  - 2.1 [基本结构](#基本结构)
  - 2.2 [数据类型](#数据结构)
  - 2.3 [运行时数据结构](#运行时数据结构)
- 3 [输入文件](#输入文件)
  - 3.1 [二进制文件格式](#二进制文件格式)
  - 3.2 [二进制文件解析过程](#二进制文件格式)
  - 3.3 [文本文件格式](#文本文件格式)
- 4 [指令集](#指令集)
  - 4.1 [指令集概述](#指令集概述)
  - 4.2 [内存操作指令](#内存操作指令)
  - 4.3 [算术运算指令](#算术运算指令)
  - 4.4 [类型转换指令](#类型转换指令)
  - 4.5 [控制转移指令](#控制转移指令)
  - 4.6 [辅助功能指令](#辅助功能指令)
- 5 [运行时错误/异常](#运行时错误异常)
- A [附录-示例](#附录)

## 概述

本文的虚拟机的设计思路参考自编译教材的PL-0编译系统、PL-S编译系统以及JVM。

虚拟机的输入是一种二进制文件，该文件提供虚拟机运行所需要的信息（详情参阅后续章节[虚拟机结构](#虚拟机结构)和[输入文件格式](#输入文件格式)）。

运行时，虚拟机的数据主要存储在栈、堆和其他必要信息的表中（详情参阅后续章节[虚拟机结构](#虚拟机结构)）。

虚拟机指令的类型识别码大小均为 1 byte，这意味着虚拟机指令最多只有256种。指令的操作数主要来自运行时栈（如[`istore`](#Tstore)），少部分指令自身会带有操作数（如[`bipush`](#bipush)）。指令操作的结果（如果有），均会压入栈顶。（详情参阅后续章节[指令集](#指令集)）

在本文档最后的附录以及 github 仓库的 `example/` 目录，会给出一些示例。

## 虚拟机结构

### 基本结构

虚拟机的内存被抽象地视为一个一维数组。每 32 bits(4 bytes) 称为 1 **slot**，slot是虚拟机寻址的最小单位（后文说的地址，也均以 slot 为单位）。

虚拟机的地址从 0 开始计数，地址的大小为 31 bits。

运行时的内存主要包括[栈](#栈)、[堆](#堆)、[常量表](#常量表)和[函数表](#函数表)这四个可读的部分，其中只有[栈](#栈)和[堆](#堆)是可写的。虚拟机还有一些[寄存器](#寄存器)用于存储运行时信息，但是对于程序员是不可见的，并且无法通过指令直接读写。

### 数据类型

虽然C0支持的数据类型有很多种，但对于虚拟机的各种操作来说，只有两种数据类型：1-slot 类型和 2-slot 类型。这意味着逻辑上占用内存小于 1 slot 的数据类型，在运行时会被提升为 1 slot。

目前在操作上支持的数据类型有：

- `char`，有效占用为 8 bits，会被提升为1 slot，以无符号整数的形式存储
- `int`，有效占用为 32 bits，即 1 slot，以有符号补码整数的形式存储
- `double`，有效占用 64 bits，即 2 slot，存储时遵循 IEEE 754 standard
- 数组，是其元素的线性排列，如果单个元素不足 1 slot，每一个元素都会被提升至 1 slot，因为虚拟机没有实现位运算，所以不会为了节约内存而进行压缩（因此对于`char`数组会存在内存浪费）
  - 字符串字面量的内部存储形式是char数组
- 复合类型 `struct`，是其成员属性的线性排列，不足 1 slot 的元素均会被提升至 1 slot
- 复合类型 `union`，有效占用等值于其有效占用最大的成员属性的有效占用，不足 1 slot 会被提升为 1 slot
- 地址，有效占用为 31 bits，提升至 1 slot，以无符号整数的形式存储

### 运行时数据结构

虚拟机启动时，读取二进制文件中的信息，之后：加载启动代码、构造[常量表](#常量表)和构造[函数表](函数表)。

虚拟机初始化完成后，会运行启动代码，之后将传入虚拟机的参数压栈并调用`main`函数，如果找不到`main`函数会立即报错停机。通过命令行传入虚拟机的参数，会根据函数表中的信息进行截断或默认填充。

输入文件的具体信息，请参阅后续章节[输入文件格式](#输入文件格式)。

#### 寄存器

虚拟机运行时会有一些记录运行时信息的属性，这些寄存器对于程序员来说是无法操作的。

虚拟机的寄存器有：

- **IP** : instruction pointer，存储下一个将要执行的指令在当前函数的代码区的下标。
- **SP** : stack pointer，存储当前栈帧数据区最高**有效地址**的下一个地址。
- **BP** : bottom pointer， 存储当前栈帧数据区的最低地址（不一定是有效的）。

栈帧的概念，请参阅接下来的章节[栈](#栈)。

#### 常量表

源代码中可能会有一些占用字节数很大的常量，比如double字面量和字符串字面量，它们作为指令的组成部分时，会导致指令的体积剧增，影响取指效率。常量表是为了消除这种效率问题而生的数据结构。

常量表存储了常量的类型和值，目前支持的类型包括：`int`、`double`、字符串。要想获得常量值，只需要执行 [`loadc`](#loadc) 指令即可。

常量表通常也会存储字符串形式的函数名，目的是在虚拟机运行出错时能提供较准确的位置。

#### 函数表

函数表存储了函数的名字、参数占用的slot数、函数嵌套层级。

虚拟机通过 [`call`](#call) 指令执行函数调用时，会去函数表查阅该函数的信息并对栈进行一些操作。

#### 栈

运行时栈存储一切局部变量以及指令执行产生的中间结果。

栈根据函数调用，被分为一个个栈帧，通常来说一个栈帧的结构为：

```
|         | <-- SP
| 中间结果 |
| 局部变量 | 
| 传入参数 | <-- BP
| 内务信息 |
------------------
|   ...   |     ^
|   ...   | 调用者栈帧
|   ...   |     v
```

传入参数、局部变量、中间结果统称为数据区，三者都可以为空。可以发现，如果数据区为空，SP和BP会指向同一个 slot。

数据区下方还有一些隐藏的内务数据，一般来说是被调用者执行时需要的信息以及从被调用者返回时需要恢复的信息。

函数调用前后，栈中发生的变化如下：

- 在调用者栈帧准备参数
- 执行 [`call`](#call) 指令
  - 弹出调用者栈帧中准备好的参数
  - 在被调用者栈帧压入内务数据
  - 在被调用者栈帧压入参数
- 执行被调用者的指令，运算都发生在被调用者栈帧
- 执行返回指令（[`ret`](#ret) 或 [`Tret`](#Tret)） 
  - 如果函数返回值类型不是 `void`（执行的不是[`ret`](#ret)），弹出在被调用者栈帧的栈顶值
  - 根据被调用者栈帧的内务数据恢复数据
  - 舍弃被调用者栈帧，回到调用者栈帧
  - 在调用者栈帧压栈返回值
- 如果调用者不需要返回值，执行 `pop` 系列指令清除调用者栈帧得到的返回值

通常来说调用者栈帧和被调用者栈帧在地址上的关系是相连的，因此一种可能的（实际采用的）栈帧实现为：

```
|         | <-- SP 
|  数据区  | <-- BP
| prev BP |
|    SL   |  被调用者栈帧
| prev PC |     
______________________
|  数据区  |     
| prev BP |  调用者栈帧
|    SL   |    
| prev PC |     
______________________
|   ...   |
```

内务数据包括：

- **prev BP**： 存储调用者调用当前函数时BP寄存器的值。

- **SL**： static link，存储当前函数在源代码意义上的外层函数的BP寄存器的值。

- **prev PC**： 存储调用者调用当前函数时下一条要执行的语句的地址。

#### 堆

堆用于运行时动态分配内存，适用于一些编译期无法完全确定的内存管理任务，以及一些对于栈来说过于庞大的内存管理。需要获取堆内存时应当执行 [`new`](#new) 指令。

## 输入文件

### 二进制文件格式

输入文件的编码格式如下：

```c++
// i2,i3,i4的内容，以大端序（big-endian）写入文件
typedef int8_t  i1;
typedef int16_t i2;
typedef int32_t i4;
typedef int64_t i8;

// u2,u3,u4的内容，以大端序（big-endian）写入文件
typedef uint8_t  u1;
typedef uint16_t u2;
typedef uint32_t u4;
typedef uint64_t u8;

struct String_info {
    u2 length;
    u1 value[length];
};

struct Int_info {
    i4 value;
};

struct Double_info {
    u4 high_bytes;
    u4 low_bytes;
};

struct Constant_info {
    // STRING = 0,
    // INT = 1,
    // DOUBLE = 2
    u1 type;
    // 根据type决定是String_info 还是 Int_info 还是 Double_info
    u1 info[];
};

struct Instruction {
    u1 opcode;
    u1 operands[/* size depends on opcode */];
};

struct Function_info {
    u2          name_index; // name: CO_binary_file.strings[name_index]
    u2          params_size;
    u2          level;
    u2          instructions_count;
    Instruction instructions[instructions_count];
};

struct Start_code_info {
    u2          instructions_count;
    Instruction instructions[instructions_count];
}

struct C0_binary_file {
    u4              magic; // must be 0x43303A29
    u4              version;
    u2              constants_count;
    Constant_info   constants[constants_count];
    Start_code_info start_code;
    u2              functions_count;
    Function_info   functions[functions_count];
};
```

特别的是：**输入文件中的多字节*基础类型*以*大端序*（*big-endian*）存储**。

即使启动代码、常量表或函数表是空的，也必须有`instructions_count`、`constants_count`和`functions_count`，且值为`0x0000`。

### 二进制文件解析过程

顺序读取文件的字节，顺序并递归地校验/识别`C0_binary_file`的组成：

- 解析`magic`，如果其值不是`0x43303A29`，则报错
- 解析`version`，如果`version`比虚拟机版本高，则报错
- 解析`constants_count`
- 解析`constants_count`个`Constant_info`为`constants`，对于每次解析：
  - 解析`type`，如果type不在支持的值域内，则报错
  - 根据`type`继续解析内容
- 解析`start_code`
  - 解析`instructions_count`
  - 解析`instructions_count`条指令（`instructions`），对于每条指令
    - 解析第一个字节为`opcode`，如果不存在对应的指令，报错
    - 根据`opcode`判断是否存在指令参数并解析，对应地采取报错
- 解析`functions_count`
- 解析`functions_count`个`Function_info`为`functions`
  - 过程类似`constants`和`start_code`

解析过程中如果报错，或是文件不完整无法解析，都会导致停机。

如果解析完`functions`还有剩余的内容，也会报错。

### 文本文件格式

虚拟机标准并不指定文本文件的格式，但应当能够等价翻译为二进制文件。

这里只给出一种可行的方案：

```assembly
.constants:
    {index} {type} {value}
    ...
.start:
    {index} {opcode} {operands}
    ...
.functions:
    {index} {name_index} {params_size} {level}
    ...
.F0:
    {index} {opcode} {operands}
    ...
.F1:
    {index} {opcode} {operands}
    ...
...
.F{functions_count-1}:
    {index} {opcode} {operands}
    ...
```

其中的常量池的数值（`value`、`index`、`operands`、`name_index`、`params_size`、`level`、`F{index}`）支持十进制和十六进制两种表示法：

```
<number> ::= <decimal> | <hexadecimal>
<decimal> ::= '0' | <nonzero-digit>{<digit>}
<hexadecimal> ::= ('0x' | '0X')<hex-digit>{<hex-digit>}
<nonzero-digit> ::= '1'|'2'|'3'|'4'|'5'|'6'|'7'|'8'|'9'
<digit> ::= '0'| <nonzero-digit>
<hex-digit> ::= <digit>|'a'|'b'|'c'|'d'|'e'|'f'|'A'|'B'|'C'|'D'|'E'|'F'
```

对于无法精确表示的浮点数，比较推荐十六进制表示。

`index`均从0开始计数。

常量的`type`有 `I`(int)、`D`(double)、`S`(字符串)

字符串的值`value`使用其字面量表示，对于需要转义的内容，使用十六进制转义序列：

```
<escape-seq> ::= 
	'\x'<hex-digit><hex-digit>
```

## 指令集

### 指令集概述

本虚拟机的指令集主要包括以下指令：

- [内存操作指令](#内存操作指令)
  - 栈地址加载：[`loada`](#loada)
  - 常量加载：[`loadc`](#loadc)、[`Tpush`](#bipush)系列
  - 基于地址的内存读取：[`Tload`](#Tload)系列
  - 基于地址的内存写入：[`Tstore`](#Tstore)系列
  - 基于数组的内存读取：[`Taload`](#Taload)系列
  - 基于数组的内存写入：[`Tastore`](#Tastore)系列
  - 堆内存申请：[`new`](#new)
  - 栈内存申请：[`snew`](#snew)
  - 栈内存释放：[`pop`](#popN)系列
  - 栈内存拷贝：[`dup`](#dup)系列
- [算术运算指令](#算术运算指令)
  - 加法：[`Tadd`](#iadd)系列
  - 减法：[`Tsub`](#isub)系列
  - 乘法：[`Tmul`](#imul)系列
  - 除法：[`Tdiv`](#idiv)系列
  - 取负：[`Tneg`](#ineg)系列
  - 比较：[`Tcmp`](#icmp)系列
- [类型转换指令](#类型转换指令)： [`i2d`](#i2d)、[`i2c`](#i2c)、[`d2i`](#d2i)
- [控制转移指令](#控制转移指令)
  - 条件转移：[`jCOND`](#jCOND)
  - 无条件转移：[`jmp`](#jmp)
  - 函数调用：[`call`](#call)
  - 函数返回：[`ret`](#ret)、[`Tret`](#Tret)系列
- [辅助功能指令](#辅助功能指令)
  - 写标准输出：[`Tprint`](#Tprint)系列、[`sprint`](#sprint)、[`printl`](#printl)
  - 读标准输入：[`Tscan`](#Tscan)系列

上述部分指令的有一个前缀`T`，这说明它是一系列**操作数数据类型不同**的但**操作相似**的指令，`T`的可能取值有：

- `i`: `int`，1 slot 的有符号整数
- `d`: `double`， 2 slot 的 IEEE 754 浮点数
- `c`: `char`，1 slot 的无符号整数，只取最低字节
- `a`: 地址， 1 slot 的无符号整数

一条指令由指令名`opcode`和操作数序列表示，二进制表示中的`opcode`只占1个字节。

之后将以如下格式解释指令的含义：

```assembly
# 指令名
opcode

# 指令的格式
# param是参数的名字，size是参数占用的字节数
# 0x??是该指令的十六进制值，每一个指令都只占1个字节
格式： `opcode param1(size1), param2(size2)` (0x??)

# 代表指令执行前后栈内元素的变化，从左到右画出栈底到栈顶的内容
# operand是操作数的名字，size是操作数占用的slot数
# result是结果的名字，size是结果占用的slot数
# 如果size写为T而不是数值，则说明其占用内存和指令名中的T类型一致
# 如果size写为param的名字，则说明其由指令的参数决定
栈变化： 
|..., operand1(size1), operand2(size2)
|..., result1(size1), result2(size2)

# desciption，该指令的文本描述
这是一个指令
```

### 内存操作指令

#### nop

格式： `nop` (0x00)

什么都不做，执行前后栈不发生任何变化。

#### bipush

格式： `bipush byte(1)` (0x01)

栈变化：

|...

|..., value(1)

将单字节值`byte`提升至`int`值`value`后压入栈。

`byte`将按照8位无符号整数解释。

#### ipush

格式： `ipush value(4)` (0x02)

栈变化：

|...

|..., value(1)

将`int`值`value`压入栈。

`value`将按照32位有符号整数解释。

#### popN

格式： 

- `pop` (0x04)
- `pop2` (0x05)
- `popn count(4)` (0x06)

栈变化：

|..., slots(count)

|...

从栈顶弹出`count`个slot。

对于`pop`，`count`取1；对于`pop2`，`count`取2。

`count`按照32位无符号整数解释。

#### dup

格式： `dup` (0x07)

栈变化：

|..., value(1)

|..., value(1), value(1)

复制栈顶的1个slot并入栈

#### dup2

格式： `dup2` (0x08)

栈变化：

|..., value(2)

|..., value(2), value(2)

或栈变化：

|..., value1(1), value2(1)

|..., value1(1), value2(1), value1(1), value2(1)

复制栈顶的2个slot并入栈

#### loadc

格式： `loadc index(2)` (0x09)

栈变化：

|...

|..., value(?)

加载常量池下标为`index`的常量值`value`，`value`占用的slot数取决于常量的类型：

- `int`：1 slot 的数值
- `double`：2 slot的数值
- 字符串：1 slot的地址值
- 数组： 1 slot 的地址值

数据类型相关的内容见[运行时数据结构-常量表](#常量表)和[数据类型](#数据类型)。

`index`以16位无符号整数解释。

#### loada

格式： `loada level_diff(2), offset(4)` (0x0a)

栈变化：

|...

|..., address(1)

沿SL链向前移动`level_diff`次（移动到当前栈帧层次差为`level_diff`的栈帧中），加载该栈帧中栈偏移为`offset`的内存的栈地址值`address`。

`level_diff`以16位无符号整数解释。

`offset`以32位有符号整数解释

#### new

格式：`new` (0x0b)

栈变化：

|..., count(1)

|..., address(1)

弹出栈顶的`int`值`count`，在堆上分配连续的 大小为`count`个slot 的内存，然后将这段内存的首地址`address`压入栈。

内存的值保证被初始化为0。

#### snew

格式： `snew count(4)` (0x0c)

栈变化：

|...

|..., value(count)

在栈顶连续分配大小为 `count`个slot 的内存。

内存的值不保证被初始化为0。

`count`以32位无符号整数解释

#### Tload

格式： 

- `iload` (0x10)
- `dload` (0x11)
- `aload` (0x12)

栈变化：

|..., address(1)

|..., value(T)

从内存地址`address`处加载一个指定类型的值。

`address`可能是栈地址也可能是堆地址。

#### Taload

格式：

- `iaload` (0x18)
- `daload` (0x19)
- `aaload` (0x1a)

栈变化：

|..., address(1), index(1)

|..., value(T)

将地址`address`视为数组首地址，加载数组下标为`index`处的指定类型的值`value`。

可用于等价翻译 `address[index]` 。

`address`可能是栈地址也可能是堆地址。

#### Tstore

格式：

- `istore` (0x20)
- `dstore` (0x21)
- `astore` (0x22)

栈变化：

|..., address(1), value(T)

|...

将指定类型的值`value`存入内存地址`address`处。

C语言中等价于 `*address = value` 。

`address`可能是栈地址也可能是堆地址。

#### Tastore

格式： 

- `iastore` (0x28)
- `dastore` (0x29)
- `aastore` (0x2a)

栈变化：

|..., address(1), index(1), value(T)

|...

将地址`address`视为数组首地址，将指定类型的值`value`存入数组下标为`index`处。

可用于等价翻译 `address[index] = value` 。

`address`可能是栈地址也可能是堆地址。

### 算术运算指令

#### iadd

格式： `iadd` (0x30)

栈变化：

|..., lhs(1), rhs(1)

|..., result(1)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs+rhs`的值`result`压栈。

求值遵循补码运算：

- 如果`result`不在`int`的值域内，那么截断高位（自然溢出）

#### dadd

格式： `dadd` (0x31)

栈变化：

|..., lhs(2), rhs(2)

|..., result(2)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs+rhs`的值`result`压栈。

求值遵循IEEE浮点数运算：

- 如果 `lhs` 和 `rhs` 中的任意一个是 NaN，那么 `result` 也是 NaN
- 如果 `lhs` 和 `rhs` 都是 inf， 且符号相同，那么 `result` 也是相同符号的 inf
- 如果 `lhs` 和 `rhs` 都是 inf， 且符号不同，那么 `result` 是 NaN
- 如果 `lhs` 和 `rhs` 只有一个是 inf， 且另一个不是 inf 也不是 NaN，那么 `result` 是 inf，且符号和 inf 操作数相同
- 如果 `lhs` 和 `rhs` 都是 +0，或 -0，那么 `result` 也是相同符号的0
- 如果 `lhs` 和 `rhs` 一个是 +0， 一个是 -0，那么 `result` 是 +0
- 如果 `lhs` 和 `rhs` 一个是 0，另一个不是 inf 也不是 NaN 也不是0，那么 `result` 和非0操作数相同
- 如果 `lhs` 和 `rhs` 都不是 inf 也不是 NaN 也不是0，且符号不同，那么 `result` 是 +0
- 当 `result` 不在 `double` 值域内时，向偶取整

#### isub

格式： `isub` (0x34)

栈变化：

|..., lhs(1), rhs(1)

|..., result(1)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs-rhs`的值 `result` 压栈。

运算本质视为 `lhs+(-rhs)`，求值遵循补码运算：

- 如果 `result` 不在`int`的值域内，那么截断高位（自然溢出）

#### dsub

格式： `dsub` (0x35)

栈变化：

|..., lhs(2), rhs(2)

|..., result(2)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs-rhs`的值 `result` 压栈。

运算本质视为 `lhs+(-rhs)`，遵循IEEE浮点数运算，参见[`dadd`](#dadd)。

#### imul

格式： `imul` (0x38)

栈变化：

|..., lhs(1), rhs(1)

|..., result(1)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs*rhs`的值 `result` 压栈。

求值遵循补码运算：

- 如果 `result` 不在`int`的值域内，那么截断高位（自然溢出）

#### dmul

格式： `dmul` (0x39)

栈变化：

|..., lhs(2), rhs(2)

|..., result(2)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs*rhs`的值 `result` 压栈。

求值遵循IEEE浮点数运算：

- 如果 `lhs` 和 `rhs` 中的任意一个是 NaN，那么 `result` 也是 NaN
- 如果 `lhs` 和 `rhs` 只有一个是 inf， 且另一个是0，那么 `result` 是 NaN
- 如果 `lhs` 和 `rhs` 都不是 NaN，若 `lhs` 和 `rhs` 符号相同，则 `result`为正，否则为负
- 如果 `lhs` 和 `rhs` 只有一个是 inf， 且另一个不是 NaN 也不是0，那么 `result` 是 inf，符号的判断同上
- 当 `result` 不在 `double` 值域内时，向偶取整

#### idiv

格式： `idiv` (0x3c)

栈变化：

|..., lhs(1), rhs(1)

|..., result(1)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs/rhs`的值 `result` 压栈。

求值遵循补码运算：

- 如果 `result` 不在`int`的值域内，那么先截断高位（自然溢出），再向0取整
- 如果 `lhs` 是 `int` 的最小值，`rhs`是 -1，那么结果是 `int` 的最小值

如果 `rhs` 是0，会抛出异常

#### ddiv

格式： `ddiv` (0x3d)

栈变化：

|..., lhs(2), rhs(2)

|..., result(2)

弹出栈顶`rhs`和次栈顶`lhs`，将`lhs/rhs`的值 `result` 压栈。

求值遵循IEEE浮点数运算：

- 如果 `lhs` 和 `rhs` 中的任意一个是 NaN，那么 `result` 也是 NaN
- 如果 `lhs` 和 `rhs` 都是 inf，那么 `result` 是 NaN
- 如果 `lhs` 和 `rhs` 都是 0，那么 `result` 是 NaN
- 如果 `lhs` 和 `rhs` 都不是 NaN，若 `lhs` 和 `rhs` 符号相同，则 `result`为正，否则为负
- 如果 `lhs` 是 inf， `rhs` 不是 NaN 也不是 inf，那么 `result` 是 inf，符号的判断同上
- 如果 `rhs` 是 inf， `lhs` 不是 NaN 也不是 inf，那么 `result` 是 0，符号的判断同上
- 如果 `rhs` 是 0， `lhs` 不是 NaN 也不是 inf，那么 `result` 是 inf，符号的判断同上
- 当 `result` 不在 `double` 值域内时，向偶取整

#### ineg

格式： `ineg` (0x40)

栈变化：

|..., value(1)

|..., result(1)

弹出栈顶`value`，将`-value`的值`result`压栈。

求值遵循补码运算：

- 如果 `result` 不在`int`的值域内，那么截断高位（自然溢出）

#### dneg

格式： `dneg` (0x41)

栈变化：

|..., value(2)

|..., result(2)

弹出栈顶`value`，将`-value`的值`result`压栈。

- 求值遵循IEEE浮点数运算：
  - 如果 `value` 是 NaN，那么 `result` 也是 NaN
  - 如果 `value` 是 inf，那么 `result` 是相反符号的 inf
  - 如果 `value` 是 0，那么 `result` 是相反符号的 0

#### icmp

格式： `icmp` (0x44)

栈变化：

|..., lhs(1), rhs(1)

|..., result(1)

弹出栈顶`rhs`和次栈顶`lhs`，并将比较结果以`int`值 `result` 压栈。

比较遵循有符号数的大小比较，`result`遵循：

- 如果 `lhs` 等于 `rhs`，则`result`是0
- 如果 `lhs` 较大，则`result`是1
- 如果 `rhs` 较大，则`result`是-1

#### dcmp

格式： `dcmp` (0x45)

栈变化：

|..., lhs(2), rhs(2)

|..., result(1)

弹出栈顶`rhs`和次栈顶`lhs`，并将比较结果以`int`值 `result` 压栈。

比较遵循有符号数的大小比较，且`result`遵循：

- 如果 `lhs` 和 `rhs` 至少有一个是 NaN，则 `result` 是0
- 如果 `lhs` 和 `rhs` 是符号相同的 inf，则 `result` 是0
- 正数大于负数，+0和-0也如此
- +inf 大于任何非 NaN 数，-inf小于任何非 NaN 数
- 如果 `lhs` 等于 `rhs`，则`result`是0
- 如果 `lhs` 较大，则`result`是1
- 如果 `rhs` 较大，则`result`是-1

### 类型转换指令

#### i2d

格式： `i2d` (0x60)

栈变化：

|..., value(1)

|..., result(2)

弹出栈顶的`int`值`value`，转换为`double`值`result`并压栈。

由于`int`可以由`double`精确表示，因此不存在精度损失。

#### d2i

格式： `d2i` (0x61)

栈变化：

|..., value(2)

|..., result(1)

弹出栈顶的`double`值`value`，转换为`int`值`result`并压栈。

转换遵循如下规则：

- 如果 `value` 是 NaN，那么 `result` 是 0
- 如果 `value` 是 +inf 或比`int`的最大值还大， 那么 `result` 是 `int` 的最大值
- 如果 `value` 是 -inf 或比`int`的最小值还小， 那么 `result` 是 `int` 的最小值
- 其他情况下，将`value`向 0 取整得到`result`

#### i2c

格式： `i2c` (0x62)

栈变化：

|..., value(1)

|..., result(1)

弹出栈顶的`int`值`value`，截断到`char`的值域内，再进行零扩展得到`result`并压栈。

这个操作可能存在精度损失，也可能改变符号。

### 控制转移指令

#### jmp

格式：`jmp offset(2) ` (0x70)

栈不发生变化。直接进行跳转，之后的控制从当前函数代码区的地址`offset`处开始执行。

`offset`以16位无符号整数解释。

#### jCOND

格式：

- `je offset(2) ` (0x71)
- `jne offset(2) ` (0x72)
- `jl offset(2) ` (0x73)
- `jge offset(2) ` (0x74)
- `jg offset(2) ` (0x75)
- `jle offset(2) ` (0x76)

栈变化：

|..., value(1)

|...

条件跳转指令弹出栈顶的`int`值`value`，如果`value`满足特定条件，则进行跳转：

- `je`：`value`是0
- `jne`：`value`不是0
- `jl`：`value`是负数
- `jge`：`value`不是负数
- `jg`：`value`是正数
- `jle`：`value`不是正数

之后的控制从当前函数代码区的地址`offset`处开始执行。

`offset`以16位无符号整数解释。

#### call

格式：`call index(2)` (0x80)

调用者栈变化：

|..., params(?)

|...

被调用者栈被创建：

|..., params(?)

查找函数表中下标为`index`的函数，将其需要的参数全部弹栈，并在准备好新的内务信息之后将参数再次入栈，控制转移到该函数的开始。

`params`在被调用者栈中的布局与`params`在调用者栈时的布局完全一致。

细节参见[虚拟机结构-运行时数据结构-栈](#栈)。

`index`以16位无符号整数解释。

#### ret

格式：`ret` (0x88)

被调用者栈被销毁：

|...

清理栈，恢复内务信息，将控制转移到原来函数的`call`指令的下一条指令。

细节参见[虚拟机结构-运行时数据结构-栈](#栈)。

#### Tret

格式：

- `iret` (0x89)
- `dret` (0x8a)
- `aret` (0x8b)

被调用者栈被销毁：

|..., rtv(T)

调用者栈变化：

|...

|..., rtv(T)

将栈顶指定类型的值`rtv`弹栈作为返回值，清理栈，恢复内务信息，将返回值`rtv`压栈，将控制转移到原来函数的`call`指令的下一条指令。

细节参见[虚拟机结构-运行时数据结构-栈](#栈)。

### 辅助功能指令

#### Tprint

格式：

- `iprint` (0xa0)
- `dprint` (0xa1)
- `cprint` (0xa2)

栈变化：

|..., value(T)

|...

弹出栈顶的`value`，并根据一定格式输出到标准输出：

- `iprint`： `value`的十进制表示，类似 `printf("%d",value)`。
- `dprint`： `value`的十进制表示保留六位小数，类似 `printf("%.6lf",value)`。
- `cprint`： `value`最低字节对应的ascii字符，类似 `putchar(value)`。

#### sprint

格式： `sprint` (0xa3)

栈变化：

|..., addr(1)

|...

弹出栈顶的`addr`，将其视为一个字符串的首地址，对每个 slot 的值通过 `cprint` 输出，直到 slot 值是 0；类似 `printf("%s",str)`。

#### printl

格式：`printl` (0xaf)

栈无变化。输出换行。

#### Tscan

格式：

- `iscan` (0xb0) 
- `dscan` (0xb1)
- `cscan ` (0xb2)

栈变化：

|...

|..., value(T)

从标准输入根据一定格式解析字符，并压栈解析得到的值`value`：

- `iscan`： 一个可有符号的十进制整数，将其转换为`int`得到`value`。
- `dscan`： 一个可有符号的十进制浮点数，将其截断至`double`值域得到`value`。
- `cscan`： 一个字节值`value`

## 运行时错误/异常

虚拟机加载文件、初始化、执行指令中可能出现各种错误，本部分定义这些错误。

### 错误集

#### Invalid File

输入的格式文件不合法，是解析输入文件时因为属性错误或文件残缺导致的错误。

#### Main Function Not Found

输入文件中找不到main函数的定义。

#### Stack Overflow

栈内存不够用。

#### Heap Overflow

堆内存不够用。

#### Invalid Memory Access

内存访问错误，包括：

- 对不存在的内存进行读写
- 读写过高/过低的栈内存
- 读写不在使用的堆内存
- 访问虚拟机不应该被访问到的关键信息
- 加载不存在的常量
- 加载上述各种错误操作对应的地址

#### Invalid Instruction

指令不合法，主要发生于执行了不存在的opcode。

#### Divide By Zero

任意整数除以整数0。

#### Invalid Control Transfer

控制转移错误，包括：

- 跳转到不存在的代码地址
- 调用不存在的函数
- 函数返回异常

#### IO Error

各种输入输出导致的错误，主要是因为遇到文件尾。

## 附录

### 运行时示例

这里用一段 C0 代码举例：

```c++
double x;

int fun(int num) {
    int rtv = num/2;
    return rtv+1;
}

void main() {
    x = 7;
    fun(x);
    return;
}
```

其对应的常量表**可以**是（也可以将`int`型常数0、1、2存入，甚至可以将7转换为`double`型常量7.0存入）：

| index |  type  | value  |
| :---: | :----: | :----: |
|   0   | STRING | "fun"  |
|   1   | STRING | "main" |
|   2   |  INT   |   7    |

其对应的函数表**可以**是：

| index | name | size of parameters（单位 : slot） | level |
| :---: | :--: | :-------------------------------: | :---: |
|   0   | fun  |                 1                 |   1   |
|   1   | main |                 0                 |   1   |

其启动代码是为全局变量`x`分配空间：

```assembly
snew 2 # 在栈上分配2个slot的内存（double x）
```

fun的指令序列**可以**是：

```assembly
loada 0, 0 # 加载fun栈帧（作用域层次差为0的栈帧）中相对于BP偏移为0的内存的地址
iload      # 弹出栈顶的地址值，从该地址加载一个int值，压栈该int值
           # 以上两行即加载局部变量（函数传参）num的值
ipush 2    # 压栈int型常值2
idiv       # 弹出两个int型值，并进行int型除法运算，压栈结果
loada 0, 1 # 加载rtv的地址
iload      # 压栈rtv的值
ipush 1 
iadd
iret       # 将栈顶的int型值作为返回值返回
```

main的指令序列为：

```assembly
loada 1, 0 # 加载全局栈帧（作用域层次差为1的栈帧）中相对于BP偏移为0的内存的地址
loadc 2      # 加载常量表中的2号常量，int型的7
i2d        # 弹出一个int值，转换为double值并压栈
dstore     # 弹出一个double值，弹出一个地址值，将double存入该地址
loada 1, 0 #
dload      # 弹出一个地址值，从该地址加载一个double值，压栈该double值
d2i        # 弹出一个double值，转换为int值并压栈
call 0     # 调用fun（函数表中序号为0的函数）
pop        # 舍弃栈顶的1个slot
ret        # 直接返回
```

从进入main到返回，栈帧变化为，栈左侧数据代表相对于BP的偏移：

```assembly
# main栈帧，刚进入时：
# 0 |    ?    | <--SP,BP
#   | 内务信息 |
#0,1|    ?    | # 未初始化的变量x
#   | 内务信息 |
# 上方是main栈帧，下方是全局栈帧，之后省略全局栈帧

loada 1, 0
# 1 |    ?    | <--SP
# 0 |    &x   | <--BP
#   | 内务信息 |

loadc 2
# 2 |    ?    | <--SP
# 1 |    7    |
# 0 |    &x   | <--BP
#   | 内务信息 |

i2d
# 3 |    ?   | <--SP
#1,2|   7.0  |
# 0 |    &x  | <--BP
#   | 内务信息 |

dstore
# 0 |    ?    | <--SP,BP
#   | 内务信息 |
# 此时全局栈帧偏移为0和1的2个slot已经被赋值为了7.0

loada 1, 0
# 1 |    ?    | <--SP
# 0 |    &x   | <--BP
#   | 内务信息 |

dload
# 2 |    ?   | <--SP
#0,1|   7.0  | <--BP
#   | 内务信息 |

d2i
# 1 |    ?   | <--SP
# 0 |    7   | <--BP
#   | 内务信息 |

call 0
# 1 |    ?   | <--SP
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |
#   | 内务信息 |
# 下方是空空的main栈帧，之后省略main栈帧

loada 0, 0
# 2 |    ?   | <--SP
# 1 |  &num  | 
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

iload
# 2 |    ?   | <--SP
# 1 |    7   | 
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

ipush 2
# 3 |    ?   | <--SP
# 2 |    2   |
# 1 |    7   | 
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

idiv
# 2 |    ?   | <--SP
# 1 |    3   |       # 变量rtv
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

loada 0, 1
# 3 |    ?   | <--SP
# 2 |  &rtv  |
# 1 |    3   |       # 变量rtv
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

iload
# 3 |    ?   | <--SP
# 2 |    3   |
# 1 |    3   |       # 变量rtv
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

ipush 1
# 4 |    ?   | <--SP
# 3 |    1   |
# 2 |    3   |
# 1 |    3   |       # 变量rtv
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

iadd
# 3 |    ?   | <--SP
# 2 |    4   |
# 1 |    3   |       # 变量rtv
# 0 |    7   | <--BP # 参数num
#   | 内务信息 |

iret
# 回到main栈帧：
# 1 |    ?   | <--SP
# 0 |    4   | <--BP
#   | 内务信息 |

pop
# 0 |    ?   | <--SP,BP
#   | 内务信息 |

ret
# 回到全局栈帧
# 2 |    ?    | <--SP
#0,1|   7.0   | <--BP # 变量x
#   | 内务信息 |
```

### 二进制文件解析示例

下面是一个输入文件的二进制节选，只截取了开头的一部分：

```assembly
43 30 3a 29 00 00 00 01 00 02 00 00 03 78 79 7a 01 00 00 01 ff 00 01 02 de ad be ef ...
```

首先解析前四个字节为`u4`类型，由于多字节基础类型以大端序排列，根据`43 30 3a 29`知其值是`0x43303a29`，和`magic`的要求匹配。

之后解析`u4`类型的`version`，可知`version=0x0000001`。

之后解析`00 02` 为 `u2`类型，得到`constant_count=0x0002`，这说明常量表只有2个元素。

接下来该解析两个`Constant_info`。

`Constant_info`的第一个元素是单字节的`Constant_info.type`，根据下一个字节是`00`，可知`type=0x00`，因此这是一个`String_info`。

`String_info`的前两个字节是`length`，根据`00 03`知`length=0x0003`，因此接下来要解析一个长度为3的字符串。

由于`value`是`u1`的数组，因此逐字节解析为`u1`，解析3个即可，跟根据`78 79 7a`得到字符串为`"xyz"`。

到这里解析完了第一个`Constant_info`，开始解析下一个。

下一个字节是`01`，可知`type=0x01`，这是一个`Int_info`。

`Int_info`只有一个`u4`类型的元素，因此还需要解析4个字节，由大端序排列的`00 00 01 ff`知`value=0x000001ff`，即十进制中的`511`。

到这里常量池就解析完了。

之后解析`Start_code_info`，首先是`u2`类型的`instructions_count`，可以得知其值为`0x0001`。说明初始化代码总共有1条指令。

首先解析一个字节`02`，得知这是一个[`ipush`](#ipush)指令，其操作数只有一个，该操作数占4字节，因此接下来以大端序解析四个字节`de ad be ef`，得到指令的完整组成为`ipush 0xdeadbeef`。

由于启动代码总长是5字节，因此到这里启动代码就解析完了。之后解析函数表，与解析常量表和启动代码的过程类似，不再赘述。

### 文本汇编与二进制文件转换示例

编译`.c0`：

```c++
int g0 = 42;
double g1 = 1.0;

int fun(int num) {
    return -num;
}

int main() {
    return fun(-123456);
}
```

得到`.s0`：

```assembly
.constants:
0 S "fun"
1 S "main"
2 I 0xdeadbeef         # unused
3 D 0x1122334455667788 # unused
4 I -123456
5 D 0x3FF0000000000000 # 1.000000
.start:
0    bipush 42
1    loadc    5
.functions:
0 0 1 1                   # .F0 fun
1 1 0 1                   # .F1 main
.F0: #fun
0    loada 0, 0
1    iload
2    ineg
3    iret
.F1: #main
0    loadc 4
1    call 0  #fun
2    iret
```

得到`.o0`（手动添加了换行和注解）：

```assembly
# magic
43 30 3a 29 
# version
00 00 00 01 
# constants_count
00 06       
# constants[0]
00       # type=STRING
00 03    # length=3
66 75 6e # value="fun"
# constants[1]
00          # type=STRING
00 04       # length=4
6d 61 69 6e # value="main"
# constants[2]
01          # type=INT
de ad be ef # value=0xdeadbeef
# constants[3]
02                      # type=DOUBLE
11 22 33 44 55 66 77 88 # value=0x1122334455667788
# constants[4]
01          # type=INT
ff fe 1d c0 # value=-123456
# constants[5]
02                      # type=DOUBLE
3f f0 00 00 00 00 00 00 # value=1.000000
# start_code
00 02    # instructions_count=0x0002
    # start_code.instructions
    01 2a    # bipush 42
    09 00 05 # loadc 5
# functions_count
00 02
# function[0]
00 00 # name_index
00 01 # params_length
00 01 # level
00 04 # instructions_count
    # function[0].instructions
    0a 00 00 00 00 00 00 # loada 0, 0
    10                   # iload
    40                   # ineg
    89                   # iret
# function[1]
00 01 # name_index
00 00 # params_length
00 01 # level
00 03 # instructions_count
    # function[1].instructions
    09 00 04 # loadc 4
    80 00 00 # call 0
    89       # iret
```

极端优化`.s0`：

```assembly
.constants:
0 S "main"
1 I 123456
.start:
.functions:
0 0 0 1    # .F0 main
.F0: #main
0    loadc 1
1    iret
```

极端优化`.o0`：

```assembly
# magic
43 30 3a 29 
# version
00 00 00 01 
# constants_count
00 02
# constants[0]
00          # type=STRING
00 04       # length=4
6d 61 69 6e # value="main"
# constants[1]
01          # type=INT
00 01 e2 40 # value=123456
# start_code
00 00    # instructions_count
# functions_count
00 01
# function[0]
00 00 # name_index
00 00 # params_length
00 01 # level
00 02 # instructions_count
    # function[0].instructions
    09 00 01 # loadc 1
    89       # iret
```



