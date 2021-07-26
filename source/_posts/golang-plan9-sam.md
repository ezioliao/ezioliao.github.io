---
title: 深入理解golang汇编
catalog: true
comments: true
indexing: true
header-img: ../../../../img/default.jpg
top: false
tocnum: true
date: 2021-07-25 17:03:14
subtitle:
tags:
- golang
- 汇编
categories: tech

---
# 深入理解golang汇编

## 什么是plan9汇编
Plan9 汇编语言是 Plan9 操作系统的汇编器支持的汇编语言。虽然plan9 OS并不算成功，但因为Golang的开发团队和Plan9 OS的团队基本差不多，golang选择Plan9汇编也就情有可原了。

Plan9汇编基本沿用的是用AT&T格式。
## 常量与全局变量
定义变量一般需要俩个步骤，分别是定义和赋值(因为go中所有变量都是初始化了的).
其中定义的语法是：
```
GLOBL symbol(SB), width
```
赋值的语法是：
```
DATA symbol+offset(SB)/width, value
```
这里`offset`是相对`symbol`的偏移，`width`是这次被赋值的内存长度，`value`是值。

### 常量
Go汇编语言中常量以$美元符号为前缀。常量的类型有整数常量、浮点数常量、字符常量和字符串常量等几种类型。以下是几种类型常量的例子：
```
$1           // 十进制
$0xf4f8fcff  // 十六进制
$1.5         // 浮点数
$'a'         // 字符
$"abcd"      // 字符串
```

字符串常量标识为：
```
# 先定义gopher字符串
GLOBL ·NameData(SB),$8
DATA  ·NameData(SB)/8,$"gopher"

# 然后赋值字符串的地址和长度
GLOBL ·Name(SB),$16
DATA  ·Name+0(SB)/8,$·NameData(SB)
DATA  ·Name+8(SB)/8,$6
```
其中`$·NameData(SB)`可以看作是地址常量

### 全局变量
变量根据作用域和生命周期有全局变量和局部变量之分。
> * 全局变量一般有着较为固定的内存地址，声明周期跨越整个程序运行时间。
> * 局部变量一般是函数内定义的的变量，只有在函数被执行的时间才被在栈上创建，当函数调用完成后将回收（暂时不考虑闭包对局部变量捕获的问题）。

全局变量一般有定义和赋值俩个操作， 要定义全局变量，首先要声明一个变量对应的符号，以及变量对应的内存大小。导出变量符号的语法如下：
```
GLOBL symbol(SB), width
```
GLOBL汇编指令用于定义名为symbol的变量，变量对应的内存宽度为width(内存的宽度必须是**2的指数倍**，编译器最终会保证变量的真实地址对齐到机器字倍数)，内存宽度部分必须用常量初始化。
>
需要注意的是，在Go汇编中我们无法为count变量指定具体的类型。在汇编中定义全局变量时，我们只关心变量的名字和内存大小，变量最终的类型只能在Go语言中声明。

变量定义之后，我们可以通过DATA汇编指令指定对应内存中的数据，语法如下：
```
DATA symbol+offset(SB)/width, value
```
这里`offset`是相对`symbol`的偏移，`width`是这次被赋值的内存长度，`value`是值。
一个示例：
```
GLOBL ·count(SB),$4   #定义全局变量count，占4个字节
DATA ·count+0(SB)/1,$1  #开始为count变量的4个字节赋值，分别赋值1、2、3、4
DATA ·count+1(SB)/1,$2
DATA ·count+2(SB)/1,$3
DATA ·count+3(SB)/1,$4

// or

DATA ·count+0(SB)/4,$0x04030201 #注意x86是小端
```
### 更多go类型示例
- **int型变量**
    ```
    GLOBL ·int32Value(SB),$4     // var int32Value int32
DATA ·int32Value+0(SB)/1,$0x01  // 第0字节
DATA ·int32Value+1(SB)/1,$0x02  // 第1字节
DATA ·int32Value+2(SB)/2,$0x03  // 第3-4字节

    GLOBL ·uint32Value(SB),$4    //var uint32Value uint32
DATA ·uint32Value(SB)/4,$0x01020304 // 第1-4字节
    ```

- **bool型变量**
    ```
    GLOBL ·boolValue(SB),$1   // 未初始化

    GLOBL ·trueValue(SB),$1   // var trueValue = true
DATA ·trueValue(SB)/1,$1  // 非 0 均为 true

    GLOBL ·falseValue(SB),$1  // var falseValue = true
DATA ·falseValue(SB)/1,$0
    ```

- **数组型变量**
    ```
GLOBL ·num(SB),$16
DATA ·num+0(SB)/8,$0
DATA ·num+8(SB)/8,$0
    ```

- **float型变量**
    ```
    GLOBL ·float32Value(SB),$4
DATA ·float32Value+0(SB)/4,$1.5      // var float32Value = 1.5

    GLOBL ·float64Value(SB),$8
DATA ·float64Value(SB)/8,$0x01020304 // bit 方式初始化
    ```

- **string类型变量**
    ```
    GLOBL ·helloworld(SB),$16   // var helloworld string

    GLOBL text<>(SB),NOPTR,$16  // 准备私有变量text，字符串的真正内容
DATA text<>+0(SB)/8,$"Hello Wo"
DATA text<>+8(SB)/8,$"rld!"

    DATA ·helloworld+0(SB)/8,$text<>(SB) // StringHeader.Data
DATA ·helloworld+8(SB)/8,$12         // StringHeader.Len

    ```

- **slice类型变量**
    ```
    GLOBL ·helloworld(SB),$24            // var helloworld []byte("Hello World!")
DATA ·helloworld+0(SB)/8,$text<>(SB) // StringHeader.Data
DATA ·helloworld+8(SB)/8,$12         // StringHeader.Len
DATA ·helloworld+16(SB)/8,$16        // StringHeader.Cap

    GLOBL text<>(SB),$16
DATA text<>+0(SB)/8,$"Hello Wo"      // ...string data...
DATA text<>+8(SB)/8,$"rld!"          // ...string data...
    ```

- **map/channel类型变量**
`map`和`channel`比较特殊，它们只是一种未知类型的指针，无法直接初始化。在汇编代码中我们只能为类似变量定义并进行0值初始化：
    ```
    var m map[string]int
GLOBL ·m(SB),$8  // var m map[string]int
DATA  ·m+0(SB)/8,$0


    var ch chan int
GLOBL ·ch(SB),$8 // var ch chan int
DATA  ·ch+0(SB)/8,$0
    ```

### 内存布局
下图是代码段的内存布局
```
GLOBL ·num(SB),$16
DATA ·num+0(SB)/8,$0
DATA ·num+8(SB)/8,$0
```
![内存布局][1]

## 函数
### 基本语法
函数的定义的语法如下：
```
TEXT [pkg]·symbol(SB), [flags,] $framesize[-argsize]
```
其中：

- `symbol`是函数符号，可以省略包名。
- `flag`是函数的一些标志，以后再讲。
- `framesize`是栈的大小，其中包含调用其它函数时准备调用参数的隐式栈空间
- `argsize`是caller传递的参数+返回值大小，注意构成`argsize`的对象并是不在callee的调用栈中的，而是在caller的栈中。可省略，编译器会自动推导出来。

作为全局标识符的全局函数，和全局变量一样，名字一般都是基于SB伪寄存器的相对地址。

### 伪寄存器
Plan9提供了4个伪寄存器，之所以是"伪"，是因为这几个寄存器是不存在的，都是编译器根据其它真实的物理寄存器给推导出来的。这4个伪寄存器对Plan9汇编函数来说很重要，参数、返回值、局部变量都是依赖这4个伪寄存器的。4个伪寄存器的官方定义如下：

- FP: Frame pointer: arguments and locals.
- PC: Program counter: jumps and branches.
- SB: Static base pointer: global symbols.
- SP: Stack pointer: top of stack.

这里再补充一些个人理解：

- `FP`是caller的栈顶指针，用于记录callee的参数；
- `SP`是callee的栈底指针(即callee的物理BP)，用来记录callee的局部变量。
- `SB`和`SP`没啥可说的。

尤其需要注意的是：
> * 在手写代码时，伪 SP 和硬件 SP 是不一样的，区分方法是看该 SP 前是否有 symbol。如果有 symbol，那么即为伪寄存器，如果没有，那么说明是硬件 SP 寄存器。但务必注意，对于编译输出(go tool compile -S / go tool objdump)的代码来讲，目前所有的 SP 都是硬件寄存器 SP，无论是否带 symbol。
> * SP 和 FP 的相对位置是会变的，所以不应该尝试用伪 SP 寄存器去找那些用 FP + offset 来引用的值，例如函数的入参和返回值。SP和FP的相对位置之所以会变化，是因为caller BP（见下图函数栈布局）是在编译期由编译器插入的，用户手写代码时，计算 frame size 时是不包括这个 caller BP 部分的。是否插入 caller BP 的主要判断依据是:
    - 函数的栈帧大小> 0
    - framepointer_enabled != 0 && goarch == "amd64" && goos != "nacl"
> 如果编译器在最终的汇编结果中没有插入 caller BP，伪 SP 和伪 FP 之间只有 8 个字节的 caller 的 return address，而插入了 BP 的话，就会多出额外的 8 字节。

一个典型的函数栈布局如下：
```
                       -----------------
                       current func arg0
                       ----------------- <----------- FP(pseudo FP)
                        caller ret addr
                       +---------------+
                       | caller BP(*)  |
                       ----------------- <----------- SP(pseudo SP，实际上是当前栈帧的 BP 位置)
                       |   Local Var0  |
                       -----------------
                       |   Local Var1  |
                       -----------------
                       |   Local Var2  |
                       -----------------                -
                       |   ........    |
                       -----------------
                       |   Local VarN  |
                       -----------------
                       |               |
                       |               |
                       |  temporarily  |
                       |  unused space |
                       |               |
                       |               |
                       -----------------
                       |  call retn    |
                       -----------------
                       |  call ret(n-1)|
                       -----------------
                       |  ..........   |
                       -----------------
                       |  call ret1    |
                       -----------------
                       |  call argn    |
                       -----------------
                       |   .....       |
                       -----------------
                       |  call arg3    |
                       -----------------
                       |  call arg2    |
                       |---------------|
                       |  call arg1    |
                       -----------------   <------------  hardware SP 位置
                       | return addr   |
                       +---------------+
```

### 汇编函数示例
**示例一**
math.go:
```
package main

import "fmt"

func add(a, b int) int // 汇编函数声明

func sub(a, b int) int // 汇编函数声明

func mul(a, b int) int // 汇编函数声明

func main() {
    fmt.Println(add(10, 11))
    fmt.Println(sub(99, 15))
    fmt.Println(mul(11, 12))
}
```
math.s:
```
#include "textflag.h" // 因为我们声明函数用到了 NOSPLIT 这样的 flag，所以需要将 textflag.h 包含进来

// func add(a, b int) int
TEXT ·add(SB), NOSPLIT, $0-24
    MOVQ a+0(FP), AX // 参数 a
    MOVQ b+8(FP), BX // 参数 b
    ADDQ BX, AX    // AX += BX
    MOVQ AX, ret+16(FP) // 返回
    RET

// func sub(a, b int) int
TEXT ·sub(SB), NOSPLIT, $0-24
    MOVQ a+0(FP), AX
    MOVQ b+8(FP), BX
    SUBQ BX, AX    // AX -= BX
    MOVQ AX, ret+16(FP)
    RET

// func mul(a, b int) int
TEXT ·mul(SB), NOSPLIT, $0-24
    MOVQ  a+0(FP), AX
    MOVQ  b+8(FP), BX
    IMULQ BX, AX    // AX *= BX
    MOVQ  AX, ret+16(FP)
    RET
    // 最后一行的空行是必须的，否则可能报 unexpected EOF

```
把这两个文件放在任意目录下，执行 `go build` 并运行就可以看到效果了。

**示例二**
来写一段简单的代码证明伪 SP、伪 FP 和硬件 SP 的位置关系。
spspfp.s:
```
#include "textflag.h"

// func output(int) (int, int, int)
TEXT ·output(SB), $8-48
    MOVQ 24(SP), DX // 不带 symbol，这里的 SP 是硬件寄存器 SP
    MOVQ DX, ret3+24(FP) // 第三个返回值
    MOVQ perhapsArg1+16(SP), BX // 当前函数栈大小 > 0，所以 FP 在 SP 的上方 16 字节处
    MOVQ BX, ret2+16(FP) // 第二个返回值
    MOVQ arg1+0(FP), AX
    MOVQ AX, ret1+8(FP)  // 第一个返回值
    RET
```
spspfp.go:
```
package main

import (
    "fmt"
)

func output(int) (int, int, int) // 汇编函数声明

func main() {
    a, b, c := output(987654321)
    fmt.Println(a, b, c)
}
```
执行上面的代码，可以得到输出:
```
987654321 987654321 987654321
```
和代码结合思考，可以知道我们当前的栈结构是这样的:
```
------
ret2 (8 bytes)
------
ret1 (8 bytes)
------
ret0 (8 bytes)
------
arg0 (8 bytes)
------ FP
ret addr (8 bytes)
------
caller BP (8 bytes)
------ pseudo SP
frame content (8 bytes)
------ hardware SP
```

  [1]: https://common-1256796170.cos.ap-nanjing.myqcloud.com/blog/melon_park/asm_go_mem_layout.png


## 参考
https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-04-func.html
https://xargin.com/plan9-assembly/
