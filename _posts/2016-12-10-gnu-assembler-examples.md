---
layout: post
categories: assembler os gnu
---
本文翻译自[gnu汇编例子](http://cs.lmu.edu/~ray/notes/gasexamples)


GNU Assembler，它的汇编器也叫gas，是GNU操作系统的默认汇编器。 它适用于许多不同的架构，并支持多种汇编语言语法。 本文下面的示例，仅适用于使用x86-64平台的Linux操作系统。

<!-- more -->
本文目录：

* 开始
* 使用C库
* 64位C代码调用约定
* C和汇编混合编程
* 命令行参数
* 浮点指令
* 数据段
* 递归
* SIMD并行性
* Saturation算术
* 本地变量和堆栈帧


## 入门
这是经典的Hello World程序，使用Linux系统调用write和exit，用于64位系统（汇编指令与32位不同）：

{% highlight  linenos %}
# ----------------------------------------------------------------------------------------
# Writes "Hello, World" to the console using only system calls. Runs on 64-bit Linux only.
# To assemble and run:
#
#     gcc -c hello.s && ld hello.o && ./a.out
#
# or
#
#     gcc -nostdlib hello.s && ./a.out
# ----------------------------------------------------------------------------------------

        .global _start

        .text
_start:
        # write(1, message, 13)
        mov     $1, %rax                # system call 1 is write
        mov     $1, %rdi                # file handle 1 is stdout
        mov     $message, %rsi          # address of string to output
        mov     $13, %rdx               # number of bytes
        syscall                         # invoke operating system to do the write

        # exit(0)
        mov     $60, %rax               # system call 60 is exit
        xor     %rdi, %rdi              # we want return code 0
        syscall                         # invoke operating system to exit
message:
        .ascii  "Hello, world\n"
{% endhighlight %}

用gcc编译

```bash
$ gcc -c hello.s && ld hello.o && ./a.out
Hello, World
```

如果您使用的是OSX或Windows操作系统，系统调用号和使用的寄存器可能会有所不同（gcc、as当然也支持这两个平台）。

## 使用C库
一般来说，都会需要使用C库。 下面是调用C库（puts函数）的Hello World：

```gas
# ----------------------------------------------------------------------------------------
# Writes "Hola, mundo" to the console using a C library. Runs on Linux or any other system
# that does not use underscores for symbols in its C library. To assemble and run:
# 
#  filename: hola.s
#     gcc hola.s && ./a.out
# ----------------------------------------------------------------------------------------

        .global main

        .text
main:                                   # This is called by C library's startup code
        mov     $message, %rdi          # First integer (or pointer) parameter in %rdi
        call    puts                    # puts(message)
        ret                             # Return to C library code
message:
        .asciz "Hola, mundo"       
```

运行

```bash
$ gcc hola.s && ./a.out
Hola, mundo
```

## 64位C代码的调用约定(Calling Conventions)
64位调用约定有一些更详细的说明，并在AMD64 ABI参考文献中进行了全面的解释。你也可以在维基百科获取他们的信息。最重要的几点是（再次，对于64位Linux，而不是Windows）：

* 从左到右，传递寄存器的参数。分配寄存器的顺序是：
    *    对于整数和指针，rdi，rsi，rdx，rcx，r8，r9。
    *    对于浮点（float，double），xmm0，xmm1，xmm2，xmm3，xmm4，xmm5，xmm6，xmm7
* 额外的参数被push到堆栈，从右到左，并在调用后被删除。
* 参数压入堆栈后，调用call指令，所以当被调用函数开始运行时，返回地址为(%rsp)，第一个内存参数的地址为8(%rsp)等。
* 在call之前，堆栈指针RSP必须对齐到16字节边界。ok，但进行call是将8字节的返回地址压入了堆栈了，所以当被调用函数运行时，%rsp并不对齐。你必须通过push某个东西或从%rsp中减去8，来增加额外的空间。
* 需要被调用函数保留的唯一寄存器（calle-save寄存器）是：rbp，rbx，r12，r13，r14，r15。其他寄存器都可以被随意改变。
* 被调用函数也应该保存XMCSR和x87控制字的控制位，但是x87指令在64位代码中很少见，所以可以不用担心。
* 整数以 rax 或 rdx:rax 返回，浮点值在 xmm0 或 xmm1:xmm0 中返回。

该程序输出前几个斐波纳契数字，演示了如何保存和恢复寄存器：

```gas
# -----------------------------------------------------------------------------
# A 64-bit Linux application that writes the first 90 Fibonacci numbers.  It
# needs to be linked with a C library.
#
# Assemble and Link:
#     gcc fib.s
# -----------------------------------------------------------------------------

        .global main

        .text
main:
        push    %rbx                    # we have to save this since we use it

        mov     $90, %ecx               # ecx will countdown to 0
        xor     %rax, %rax              # rax will hold the current number
        xor     %rbx, %rbx              # rbx will hold the next number
        inc     %rbx                    # rbx is originally 1
print:
        # We need to call printf, but we are using eax, ebx, and ecx.  printf
        # may destroy eax and ecx so we will save these before the call and
        # restore them afterwards.

        push    %rax                    # caller-save register
        push    %rcx                    # caller-save register

        mov     $format, %rdi           # set 1st parameter (format)
        mov     %rax, %rsi              # set 2nd parameter (current_number)
        xor     %rax, %rax              # because printf is varargs

        # Stack is already aligned because we pushed three 8 byte registers
        call    printf                  # printf(format, current_number)

        pop     %rcx                    # restore caller-save register
        pop     %rax                    # restore caller-save register

        mov     %rax, %rdx              # save the current number
        mov     %rbx, %rax              # next number is now current
        add     %rdx, %rbx              # get the new next number
        dec     %ecx                    # count down
        jnz     print                   # if not done counting, do some more

        pop     %rbx                    # restore rbx before returning
        ret
format:
        .asciz  "%20ld\n"

```

运行：

```bash
$ gcc fib.s && ./a.out
                   0
                   1
                   1
                   2
                   3
                 ...
  420196140727489673
  679891637638612258
 1100087778366101931
 1779979416004714189
```

## 和C语言的混合编程

这个64位例子，是一个非常简单的函数，它读入3个64位整数并返回最大值。 它演示了如何提取整数参数：它们将被push到堆栈，以便在进入该函数时，它们将分别在rdi，rsi和rdx中。 返回值是一个整数，所以把它放在了rax中。

```gas
# -----------------------------------------------------------------------------
# A 64-bit function that returns the maximum value of its three 64-bit integer
# arguments.  The function has signature:
#
#   int64_t maxofthree(int64_t x, int64_t y, int64_t z)
#
# Note that the parameters have already been passed in rdi, rsi, and rdx.  We
# just have to return the value in rax.
# -----------------------------------------------------------------------------

        .globl  maxofthree
        
        .text
maxofthree:
        mov     %rdi, %rax              # result (rax) initially holds x
        cmp     %rsi, %rax              # is x less than y?
        cmovl   %rsi, %rax              # if so, set result to y
        cmp     %rdx, %rax              # is max(x,y) less than z?
        cmovl   %rdx, %rax              # if so, set result to z
        ret   
```

调用这段汇编代码的C程序：

```c
/*
 * callmaxofthree.c
 *
 * A small program that illustrates how to call the maxofthree function we wrote in
 * assembly language.
 */

#include <stdio.h>
#include <inttypes.h>

int64_t maxofthree(int64_t, int64_t, int64_t);

int main() {
    printf("%ld\n", maxofthree(1, -4, -7));
    printf("%ld\n", maxofthree(2, -6, 1));
    printf("%ld\n", maxofthree(2, 3, 1));
    printf("%ld\n", maxofthree(-2, 4, 3));
    printf("%ld\n", maxofthree(2, -6, 5));
    printf("%ld\n", maxofthree(2, 4, 6));
    return 0;
}

```
汇编、链接、运行这两段代码：

```bash
$ gcc -std=c99 callmaxofthree.c maxofthree.s && ./a.out
1
2
3
4
5
6
```

## 命令行参数

大家知道在C中，main只是一个简单的函数，它有两个自己的参数：

     int main（int argc，char ** argv）

下面这个例子，简单地打印一个程序的命令行参数，每行一个：

```gas
# -----------------------------------------------------------------------------
# A 64-bit program that displays its commandline arguments, one per line.
#
# On entry, %rdi will contain argc and %rsi will contain argv.
# -----------------------------------------------------------------------------

        .global main

        .text
main:
        push    %rdi                    # save registers that puts uses
        push    %rsi
        sub     $8, %rsp                # must align stack before call

        mov     (%rsi), %rdi            # the argument string to display
        call    puts                    # print it

        add     $8, %rsp                # restore %rsp to pre-aligned value
        pop     %rsi                    # restore registers puts used
        pop     %rdi

        add     $8, %rsi                # point to next argument
        dec     %rdi                    # count down
        jnz     main                    # if not done counting keep going

        ret
format:
        .asciz  "%s\n"
```

运行结果：

```bash
$ gcc echo.s && ./a.out 25782 dog huh $$
./a.out
25782
dog
huh
9971
$ gcc echo.s && ./a.out 25782 dog huh '$$'
./a.out
25782
dog
huh
$$
```

请注意，就C Library而言，命令行参数始终是字符串。 如果要将它们视为整数，需要调用atoi。 这是一个计算x的y次方的小程序。 该示例的另一个功能是它显示了如何将值限制为32位。

```gas
# -----------------------------------------------------------------------------
# A 64-bit command line application to compute x^y.
#
# Syntax: power x y
# x and y are integers
# -----------------------------------------------------------------------------

        .global main

        .text
main:
        push    %r12                    # save callee-save registers
        push    %r13
        push    %r14
        # By pushing 3 registers our stack is already aligned for calls

        cmp     $3, %rdi                # must have exactly two arguments
        jne     error1

        mov     %rsi, %r12              # argv

# We will use ecx to count down form the exponent to zero, esi to hold the
# value of the base, and eax to hold the running product.

        mov     16(%r12), %rdi          # argv[2]
        call    atoi                    # y in eax
        cmp     $0, %eax                # disallow negative exponents
        jl      error2
        mov     %eax, %r13d             # y in r13d

        mov     8(%r12), %rdi           # argv
        call    atoi                    # x in eax
        mov     %eax, %r14d             # x in r14d

        mov     $1, %eax                # start with answer = 1
check:
        test    %r13d, %r13d            # we're counting y downto 0
        jz      gotit                   # done
        imul    %r14d, %eax             # multiply in another x
        dec     %r13d
        jmp     check
gotit:                                  # print report on success
        mov     $answer, %rdi
        movslq  %eax, %rsi
        xor     %rax, %rax
        call    printf
        jmp     done
error1:                                 # print error message
        mov     $badArgumentCount, %edi
        call    puts
        jmp     done
error2:                                 # print error message
        mov     $negativeExponent, %edi
        call    puts
done:                                   # restore saved registers
        pop     %r14
        pop     %r13
        pop     %r12
        ret

answer:
        .asciz  "%d\n"
badArgumentCount:
        .asciz  "Requires exactly two arguments\n"
negativeExponent:
        .asciz  "The exponent may not be negative\n"
```

运行结果：

```bash
$ ./power 2 19
524288
$ ./power 3 -8
The exponent may not be negative
$ ./power 1 500
1
```

练习：重写这个例子，使用64位整数。 需要用strtol替换掉atoi。

## 浮点指令
浮点的参数放在xmm寄存器中。 这是一个简单的函数，对double数组中的值进行求和：

```gas
# -----------------------------------------------------------------------------
# A 64-bit function that returns the sum of the elements in a floating-point
# array. The function has prototype:
#
#   double sum(double[] array, unsigned length)
# -----------------------------------------------------------------------------

        .global sum
        .text
sum:
        xorpd   %xmm0, %xmm0            # initialize the sum to 0
        cmp     $0, %rsi                # special case for length = 0
        je      done
next:
        addsd   (%rdi), %xmm0           # add in the current array element
        add     $8, %rdi                # move to next array element
        dec     %rsi                    # count down
        jnz     next                    # if not done counting, continue
done:
        ret    
```

调用他的c代码：

```c
/*
 * callsum.c
 *
 * Illustrates how to call the sum function we wrote in assembly language.
 */

#include <stdio.h>

double sum(double[], unsigned);

int main() {
    double test[] = {
        40.5, 26.7, 21.9, 1.5, -40.5, -23.4
    };
    printf("%20.7f\n", sum(test, 6));
    printf("%20.7f\n", sum(test, 2));
    printf("%20.7f\n", sum(test, 0));
    printf("%20.7f\n", sum(test, 3));
    return 0;
}
```
运行：

```bash
$ gcc callsum.c sum.s && ./a.out
          26.7000000
          67.2000000
           0.0000000
          89.1000000
          
```

## 数据段

在大多数操作系统上，代码段是只读的，数据段仅用于初始化数据，并且有一个特殊的.bss段用于未初始化的数据。

下面是一段代码，它计算命令行参数的平均值，预期为整数，并将结果显示为浮点数。注意特殊的是：代码里有 .text .data两个段标识，分别代表代码段和数据段的开始

```gas
# -----------------------------------------------------------------------------
# 64-bit program that treats all its command line arguments as integers and
# displays their average as a floating point number.  This program uses a data
# section to store intermediate results, not that it has to, but only to
# illustrate how data sections are used.
# -----------------------------------------------------------------------------

        .globl  main

        .text
main:
        dec     %rdi                    # argc-1, since we don't count program name
        jz      nothingToAverage
        mov     %rdi, count             # save number of real arguments
accumulate:
        push    %rdi                    # save register across call to atoi
        push    %rsi
        mov     (%rsi,%rdi,8), %rdi     # argv[rdi]
        call    atoi                    # now rax has the int value of arg
        pop     %rsi                    # restore registers after atoi call
        pop     %rdi
        add     %rax, sum               # accumulate sum as we go
        dec     %rdi                    # count down
        jnz     accumulate              # more arguments?
average:
        cvtsi2sd sum, %xmm0
        cvtsi2sd count, %xmm1
        divsd   %xmm1, %xmm0            # xmm0 is sum/count
        mov     $format, %rdi           # 1st arg to printf
        mov     $1, %rax                # printf is varargs, there is 1 non-int argument

        sub     $8, %rsp                # align stack pointer
        call    printf                  # printf(format, sum/count)
        add     $8, %rsp                # restore stack pointer

        ret

nothingToAverage:
        mov     $error, %rdi
        xor     %rax, %rax
        call    printf
        ret

        .data
count:  .quad   0
sum:    .quad   0
format: .asciz  "%g\n"
error:  .asciz  "There are no command line arguments to average\n"

```

## 递归

汇编做递归感觉会比较怪，但也许会令你惊讶的是，实现递归函数没有什么特殊的要求。 你需要做的只是，按照通常的方法小心的保存寄存器。 下面是一个例子。

C语音版本：

```c
uint64_t factorial(unsigned n) {
    return (n <= 1) ? 1 : n * factorial(n-1);
}
```

汇编版本：

```gas
# ----------------------------------------------------------------------------
# A 64-bit recursive implementation of the function
#
#     uint64_t factorial(unsigned n)
#
# implemented recursively
# ----------------------------------------------------------------------------

        .globl  factorial

        .text
factorial:
        cmp     $1, %rdi                # n <= 1?
        jnbe    L1                      # if not, go do a recursive call
        mov     $1, %rax                # otherwise return 1
        ret
L1:
        push    %rdi                    # save n on stack (also aligns %rsp!)
        dec     %rdi                    # n-1
        call    factorial               # factorial(n-1), result goes in %rax
        pop     %rdi                    # restore n
        imul    %rdi, %rax              # n * factorial(n-1), stored in %rax
        ret
```

用C语音调用这个递归函数：

```c
/*
 * An application that illustrates calling the factorial function defined elsewhere.
 */

#include <stdio.h>
#include <inttypes.h>

uint64_t factorial(unsigned n);

int main() {
    for (unsigned i = 0; i < 20; i++) {
        printf("factorial(%2u) = %lu\n", i, factorial(i));
    }
}
```

## SIMD并行性
XMM寄存器可以对浮点值进行运算，每条指令可以进行一次或多次的算术运算。 

指令形式如下：

`operation xmmregister_or_memorylocation，xmmregister`

对于浮点加法运算，指令如下：

* addpd：  做2个双精度加法
* addps：  做一个双精度加法，使用寄存器的低64位
* addsd：  做4个单精度加法
* addss：  做1个单精度加法，使用寄存器的低32位

练习：  实现一个计算浮点数组的函数，一次计算4个。
## Saturation算术
Saturation 算术是一种特殊的算术，其中所有的操作（如加法和乘法）被限制在最小值和最大值之间的固定范围内。

XMM寄存器也支持这种操作，可以在浮点处理器上进行整数的算术运算。 指令的格式如下：

` operation xmmregister_or_memorylocation，xmmregister`

对于整数加法，指令如下：

* paddb： 16字节加法
* paddw： 8个word 的加法
* paddd： 4个dword的加法
* paddq： 2个qword的加法
* paddsb：16字节加法，带符号(80..7F，括号内表示Saturation的范围，下同)
* paddsw：8个word的加法，无符号(8000..7FFF)
* paddusb：16字节加法，无符号(00..FF)
* paddusw：8个word的加法，无符号 (00..FFFF)

## 本地变量和堆栈帧
首先，请阅读[Eli Bendersky](http://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)的文章，里面的概述比本文的简要说明更完整。

当调用函数时，调用者首先将参数放入正确的寄存器中，然后发出call指令。 超出寄存器数目的额外的参数,将在call指令之前被push到堆栈。

call指令将返回地址放在堆栈顶部。 

比如在下面这段代码：

```c
long example(long x, long y) {
    long a, b, c;
    b = 7;
    return x * b + y;
}
```

在进入example函数时，x将在%edi中，y将在%esi中，返回地址将位于堆栈的顶部。 我们在哪里可以放置局部变量？ 最简单的选择是当然还是堆栈，当然如果有足够的registers，也可以使用。

如果你在遵循标准ABI的计算机上运行，则可以将%rsp保留不变，并通过%rsp访问“额外的参数”和本地变量，例如：

```
                +----------+
         rsp-24 |    a     |
                +----------+
         rsp-16 |    b     |
                +----------+
         rsp-8  |    c     |
                +----------+
         rsp    | retaddr  |
                +----------+
         rsp+8  | caller's |
                | stack    |
                | frame    |
                | ...      |
                +----------+
```

这样函数就会简化为：

```gas
        .text
        .globl  example
example:
        movl    $7, -16(%rsp)
        mov     %rdi, %rax
        imul    8(%rsp), %rax
        add     %rsi, %rax
        ret
```

如果我们的函数是要call另一函数，那么则必须调整%rsp来正确返回。

在Windows上，却不能使用此方法，因为如果发生interrupt，堆栈指针上方的所有内容都将被抹去。 这在大多数其他操作系统上都不会发生，因为堆栈指针有一个128字节的“red zone”，发生interrupt后是安全的。 在这种情况下，你可以在进入被调用函数后，立即在堆栈上开出一段内存出来：

```
example:
        sub     $24, %rsp
```

此时的堆栈看起来是这样：

```
                +----------+
         rsp    |    a     |
                +----------+
         rsp+8  |    b     |
                +----------+
         rsp+16 |    c     |
                +----------+
         rsp+24 | retaddr  |
                +----------+
         rsp+32 | caller's |
                | stack    |
                | frame    |
                | ...      |
                +----------+
```

下面是最后的代码。要注意，必须在返回之前恢复堆栈指针(add $24, %rsp)！

```gas
        .text
        .globl  example
example:
        sub     $24, %rsp
        movl    $7, 8(%rsp)
        mov     %rdi, %rax
        imul    8(%rsp), %rax
        add     %rsi, %rax
        add     $24, %rsp
        ret
```
[英文原文](http://cs.lmu.edu/~ray/notes/gasexamples/)
