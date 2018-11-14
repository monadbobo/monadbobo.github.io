---
id: 309
title: c reference manual读书笔记(二)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=309'
permalink: /2011/07/16/c-reference-manual%e8%af%bb%e4%b9%a6%e7%ac%94%e8%ae%b0%e4%ba%8c/
categories:
  - 语言规范
tags:
  - c
  - 语言规范
  - 读书笔记
translate_title: c-reference-manual-reading-notes-(two)
date: 2011-07-16 16:07:11
---
这次主要是预处理器和宏处理的部分。

1 预处理器主要是处理源代码中以#开头的行，预处理器执行完毕后的代码必定是一个合法的c程序.

2 预处理器的命令是完全不依赖于c语言的语法的.

3 预处理器不会parse源代码，预处理的词法处理和编译器的是不同的，预处理器能够理解合法的c标记，可是它也会忽略在c编译器中认为是不合法标记。比如下面的代码,对于预处理器来说，没有任何问题的.
  
```int3 i;```

4 standard c允许#后面有空格，但是一些老的c编译器是不允许的。如果#后面没有任何字符(除了空格),那么standard c则认为这是一个空行，一些老的编译器有可能有不同的行为.
  
<!--more-->


  
5 预处理器的执行是在宏处理之前的，因此如果一个宏扩展后是一个预处理命令，那么这个命令是不会被执行的，然后在编译的时候就会出错，比如下面的代码:
  
```
  
#define my_include #include
  
my_include <stdio.h>
  
```
  
预处理器处理完毕之后结果将会是:
  
```
  
#include <stdio.h>
  
```

6 通过前面我们知道 backslash(\)的处理是在预处理命令执行之前，因此下面的代码：
  
```
  
#define BACKSLASH \
  
#define ASTERISK *
  
```
  
将会被预处理作为
  
```
  
#define BACKSLASH #define ASTERISK *
  
```来处理

7 宏在注释/字符串/char/#include 中是被忽略的, 如果宏没有参数，则宏名字后面的第一个非空格字符紧跟着的序列将作为宏body.比如下面的代码
  
```
  
#define N 5 ; //N将会被替换为5 ;
  
#define A = 3 //A将会被替换为 = 3
  
```

8 带参数的宏，其中c99提供了变长参数的宏，而gcc也提供了这个扩展,不过用法略有不同.这里要注意，参数列表的左括号必须紧跟着宏名字(不能有空格,不过调用的时候没有这个限制)，如果有空格，则这个宏将被认为是没有参数的，body体是以左括号开始,比如下面的代码
  
```
  
#define Test (a,b) a+b

Test(1,2)//将会被替换为(a,b) a+b(1+2)
  
```

9 调用带参数的宏的时候，参数不能包含逗号，可是如果逗号是包含在括号里面，则可以使用逗号,比如下面的代码:
  
```
  
#define insert(stm) stm

insert({a=1,b=1;}) //这个将会出错
  
insert({(a=1,b=1);})//这个才是正确的
  
```

10 当调用一个带参数的宏时，会先对宏body的copy进行参数的替换，然后才是对整个宏被替换.具体顺序是这样子的，首先将实参和行参联系起来，然后复制一个宏body，并对其中包含的形参替换为实参，最后用这份宏body的拷贝替换整个宏.

11 一旦一个宏调用被扩展，则将会马上从扩展后的结果的开始进行宏的替换，因为有可能会有宏的嵌套，这里注意是结果的开始重新检索的，来看下面的例子
  
```
  
#define plus(x,y) add(y,x)
  
#define add(x,y) ((x)+(y))

plus(plus(a,b),c)
  
```
  
这里的宏替换顺序是这样子的：
  
```
  
add(c,plus(a,b))
  
c+plus(a,b)
  
c+add(b,a)
  
c+ b + a
  
```

12 标准c会在宏定义和调用时执行string的合法性检测，比如下面的代码在标准c中预编译阶段会出错的.
  
```
  
#define F "asdfa

printf(F asf");
  
```

13 include预编译指令中<>或者&#8221;&#8221;中不能包含空格,比如下面的例子:
  
```
  
#include < stdio.h>//这里将会出错
  
```

14 c99中对于所有的预处理器执行期间的算术操作都使用当前计算机的最大整数类型，比如intmaxt或者uintmax_t.这个其实主要是对于#if这类指令来说的.

15 #pragma是用来提供给编译器一些附加的信息的(因此各个编译器都实现了自己的信息,比如vc下的#prama once,gcc中的#pragma GCC dependency)，而在c99中还提供了一个函数_Pragma的操作符，比如_Pragma(&#8220;argument&#8221;)是等同于#pragma argument的.而在gcc中早已经有一个类似pragma的东西，那就是__attribute__,所以gcc中就很少很少使用pragma.更详细的可以看这里：
  
http://gcc.gnu.org/onlinedocs/cpp/Pragmas.html