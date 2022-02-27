---
title: C++ 中的 static 关键字
date: 2019-02-23
tags:
 - C++
---
> **static** 是 c++ 的关键字，顾名思义是表示静态的含义。它在 c++ 中既可以修饰变量也可以修饰函数。那当我们使用 static 时，编译器究竟做了哪些事情呢？

早先被问到 static 关键字，感觉既熟悉又陌生。熟悉是都知道如何去使用它，陌生又来自不知道它究竟对我们程序做了什么。今天就来好好复习下这个关键字，本文的重点也在第三部分。
<!--more-->
------
### 目录
- <a href="#1">1 先说说 **extern**</a>
  - <a href="#1.1">1.1 **extern** 用于修饰变量</a>
  - <a href="#1.2">1.2 **extern** 用于修饰函数</a>
  - <a href="#1.3">1.3 **extern** 用于指定编译类型</a>
- <a href="#2">2 再讲讲 **static**</a>
  - <a href="#1.2">2.1 **static** 用于修饰变量</a>
  - <a href="#2.2">2.2 **static** 用于修饰函数</a>
- <a href="#3">3 关于 **text**、**bss** 与 **data** 段</a>
  - <a href="#3.1">3.1 局部变量的编译</a>
  - <a href="#3.2">3.2 全局变量的编译</a>
------
先看一下示例代码:  
test1.cpp
```C++
#include <iostream>
extern int a_int;
extern void func2();

static char c_array[10000];

void func1() {
  static int a_tmp = 0;
  std::cout << a_tmp++ << std::endl;
  return;
}

int main(int argc, char **argv) {
  a_int = 1;

  //静态局部变量示例
  for (auto i = 0; i < 5; i++) {
    func1();
  }

  //比较静态全局变量的地址示例
  std::cout << static_cast<const void *>(c_array) << std::endl;
  func2();
  return 0;
}
```
test2.cpp
```C++
#include <iostream>

int a_int;
static char c_array[1000];
void func2() {
  std::cout << static_cast<const void *>(c_array) << std::endl;
  return 0;
}

```
### <a name="1">1 先说说 **extern**</a>
**extern 关键字用于告诉编译器，在其他的模块中寻找相应的定义**

为什么 static 前要先说  extern 呢？因为他们就像相互对立的一对关键字，所以 extern 与 static 一起用时编译器会报错～
#### <a name="1.1">1.1 **extern** 用于修饰变量</a>
以示例代码中的 a_int 变量为例，假设其他的变量和函数不存在
我们先**将 extern 关键字去掉(test1.cpp:2)**，然后执行步骤:    
1.  编译`g++ -c -o test1.o test1.cpp`  
2.  查看符号`nm test1.o`  

```
00000000000000d0 S _a_int
```
可以看到 a_int 为一个未初始化的符号。说明符号在 test1.o 中已经被定义了。此时直接编译（`g++ -o test1 test1.cpp`）是不会报错的。

然后我们再**将 extern 关键字加上(test1.cpp:2)**，并重复上面步骤观察符号  
`nm test1.o`  
会发现 test1.o 中没有该符号的定义。并且再编译会报错：
```
Undefined symbols for architecture x86_64:
  "_a_int", referenced from:
      _main in test1-ed3c01.o
ld: symbol(s) not found for architecture x86_64
```
很明显，链接器没有找到 a_int 的定义。此时只需要将 test2.o 加入再编译（g++ -o test test1.cpp test2.cpp）就可以啦
`注：此时如果去掉 main 函数中对 a_int 变量的引用，也可以编译通过，毕竟 a_int 在程序中实际没有用到`
#### <a name="1.2">1.2 **extern** 用于修饰函数</a>
以示例代码中的 func2（） 函数为例  
因为在 main 函数中调用了 func2（），所以需要在 main 之前进行函数声明。  但此时的函数声明无论加不加 extern 其实**并无多少区别**  
#### <a name="1.3">1.3 **extern** 用于指定编译类型</a>
因为 C++ 编译时会进行 [name mangling[wiki]](https://en.wikipedia.org/wiki/Name_mangling)，导致所看到的函数与实际编译后的符号差距很大。在某些情况下会导致链接时找不到符号的问题  
此时可以使用
```C
extern "C"
{
  ...
}
```
这样在范围内的代码都将按照 C 的格式进行编译
### <a name="2">2 再讲讲 **static**</a>
static 关键字在我看来的作用是
1.  能够改变变量的存储方式
2.  能够改变变量与函数的访问范围

#### <a name="2.1">2.1 **static** 用于修饰变量</a>
我们都知道当程序经过编译后：
- 函数体内的局部变量会保存在栈中，局部变量随着函数的调用和返回进行构造与析构，并且在函数返回后无法使用。
- 全局变量保存在静态数据区直到程序退出时才会被析构掉。所以在整个程序内全局变量都可以使用(当然要考虑到作用域)。  

对于**局部变量**，当我们在变量前加上 **static** 时，就是告诉了编译器将该变量放入静态数据区。既函数退出时不会将该变量析构掉，当我们下次再调用改函数依然可以取得内存中的这个变量。  
例如 test1.cpp 中，每次调用 func1() 时 a_tmp 变量都不会被销毁，最后输出
```
0
1
2
3
4
```

对于**全局变量**，加上 static 关键字后该变量只能用于当前的文件。
例如 test1.cpp 中的 c_array，加上 static 后只能在当前源文件使用。  
此时如果我们再在 test2.cpp 中定义一个同名的全局静态数组进行编译（`g++ -o test test1.cpp test2.cpp`）并且输出他们的地址  
```
test1.cpp[c_array]0x10bb5e100
test2.cpp[c_array]0x10bb60810
```
可以看到两个地址是不同的，所以虽说是同名的两个全局变量。但都经过 static 修饰后，他们实际还是两个地址不同相互独立的变量。

那么再试一下，将 test1.cpp 中的
```C++
static char c_array[10000];
```
修改为
```C++
extern char c_array[10000];
```
然后再编译(`g++ -o test test1.cpp test2.cpp`)可以看到
```
Undefined symbols for architecture x86_64:
  "_c_array", referenced from:
      _main in test1-5d6201.o
ld: symbol(s) not found for architecture x86_64
```
这当然是因为 test2.cpp 中的 c_array 还是有 static 进行修饰的，导致我们无法在 test1.cpp 文件中访问到。那就将 static 去掉，看到结果
```
test1.cpp[c_array]0x10b1e2100
test2.cpp[c_array]0x10b1e2100
```
它们的地址相同对应的同一块内存，是同一个变量！

#### <a name="2.2">2.2 **static** 用于修饰函数</a>
static 对于函数于变量其实比较类似，它限定了函数只能在当前的模块中使用。  
假如我们将 test2.cpp 中的 func2() 函数加上 **static** 关键字，那么编译（`g++ -o test test1.cpp test2.cpp`）也会报错找不到符号
```
Undefined symbols for architecture x86_64:
  "func2()", referenced from:
      _main in test1-80a5c0.o
ld: symbol(s) not found for architecture x86_64
```
### <a name="3">3 关于 **text**、**bss** 与 **data** 段</a>
---
关于数据段、编译、链接方面的知识非常推荐看看  
[<<程序员的自我修养:链接、装载与库>>](https://www.amazon.cn/dp/B0027VSA7U/ref=sr_1_1?ie=UTF8&qid=1550913082&sr=8-1&keywords=程序员的自我修养)
#### <a name="3.1">3.1 局部变量的编译</a>
是否曾经好奇函数内的临时变量经过编译会变成什么样子？  

假设我们写了如下代码，并编译成名为 test 的可执行文件
```C++
int main() {
  char s1[11] = "helloworld";
  char s2[11] = "helloworld";
  return 0;
}
```
那么可以通过 `objdump -DS test `观察到 main 函数中有如下片段（有省略）
```
Disassembly of section .text:
.....
00000000004005b0 <main>:
4005b4:	48 b8 68 65 6c 6c 6f 	movabs $0x726f776f6c6c6568,%rax
4005bb:	77 6f 72
4005be:	48 89 45 f0          	mov    %rax,-0x10(%rbp)
4005c2:	66 c7 45 f8 6c 64    	movw   $0x646c,-0x8(%rbp)
4005c8:	c6 45 fa 00          	movb   $0x0,-0x6(%rbp)
4005cc:	48 b8 68 65 6c 6c 6f 	movabs $0x726f776f6c6c6568,%rax
4005d3:	77 6f 72
4005d6:	48 89 45 e0          	mov    %rax,-0x20(%rbp)
4005da:	66 c7 45 e8 6c 64    	movw   $0x646c,-0x18(%rbp)
4005e0:	c6 45 ea 00          	movb   $0x0,-0x16(%rbp)
......
```
观察下 0x646c 和 0x726f776f6c6c6568，转化成 ascii 就是  
`100 108 114 111 119 111 108 108 101 104`
对应的字符  
'd' 'l' 'r' 'o' 'w' 'o' 'l' 'l' 'e' 'h'，看出来了吧，编译器将 "helloworld" 以立即数的方式写到了 text 段内。
然后通过 `readelf -a test`会发现并没有 s1 与 s2 的符号。

现在将代码改为这样又会如何？
```C++
static char s1[11] = "helloworld";
static char s2[11] = "helloworld";
```
继续通过 `objdump -DS test`观察发现 main 中发生了改变
```
Disassembly of section .text:
00000000004005b0 <main>:
  4005b0:	55                   	push   %rbp
  4005b1:	48 89 e5             	mov    %rsp,%rbp
  4005b4:	b8 00 00 00 00       	mov    $0x0,%eax
  4005b9:	5d                   	pop    %rbp
  4005ba:	c3                   	retq
  4005bb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
```
通过 `readelf -a test`可以看到新增了两个地址不同的符号，由此可见 static 确实改变了变量的存储方式
```
Symbol table '.symtab' contains 66 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
   37: 000000000060102c    11 OBJECT  LOCAL  DEFAULT   24 _ZZ4mainE2s2
   38: 0000000000601037    11 OBJECT  LOCAL  DEFAULT   24 _ZZ4mainE2s1
```
那么如果指向常量呢?稍微改下
```C++
const char *s1 = "helloworld";
const char *s2 = "helloworld";
```
继续通过 objdump 观察到 main 中有这两代码，很明显了 0x400660存储着我们的 "helloworld"的字符串常量
```
00000000004005b0 <main>:
  4005b4:	48 c7 45 f8 60 06 40 	movq   $0x400660,-0x8(%rbp)
  4005bb:	00
  4005bc:	48 c7 45 f0 60 06 40 	movq   $0x400660,-0x10(%rbp)
```
找到这个地址，发现这个地址属于 **.rodata** 段。这就是我们常说用来保存字面值常量的数据段。
```
Disassembly of section .rodata:
0000000000400658 <__dso_handle>:
	...
  400660:	68 65 6c 6c 6f       	pushq  $0x6f6c6c65
  400665:	77 6f                	ja     4006d6 <__dso_handle+0x7e>
  400667:	72 6c                	jb     4006d5 <__dso_handle+0x7d>
  400669:	64                   	fs
```
观察下十六进制的值，就是我们的 "helloworld" 没错啦。
#### <a name="3.2">3.2 全局变量的编译</a>
那么对于全局变量又应该是如何存储的呢？  
首先我们知道无论静态还是非静态的变量都应该存储在静态数据区。我们熟悉的静态数据区就有 **.bss** 和 **.data**。  
**.bss** 在编译时实际上不占据空间，只有在运行时才会由被分配空间。那么还是来验证下
```C++
char a_array[10000];
static char b_array[10000];
int main() {
  return 0;
}
```
编译一下(`g++ -o test test.cpp`)，然后通过 size 命令观察(`size test`)
```
text	   data	    bss	    dec	    hex	filename
1320	    588	  20048	  21956	   55c4	test
```
可以看出 a_array 和 b_array 都实际记录在 **.bss** 段，并且 **.data**段的大小显然不符合我们定义的数组大小。通过 `ll test`会发现文件大小不足10000 字节，所以可以肯定的是申请的这两个数组在编译时并为被分配内存。

那么继续改一下看看
```C++
char a_array[10000] = "helloworld";
static char b_array[10000];
```
继续使用 `size  test`看下
```
text	   data	    bss	    dec	    hex	filename
1320	  10616	  10032	  21968	   55d0	test
```
data 段和文件都多出了 10000 多字节！！！  
这就是因为 a_array 进行了初始化，所以编译器为其分配了内存。同理如果 b_array 也进行了初始化，那么大小还会增加。  
**tips:**  
**如果进行了初始化，但是内存中还是 0 值的话，编译器依旧不会为其分配内存的，例如
`int a_array[10000] = {0};`**

------
对于能指出文中的错误不胜感激！ :D
