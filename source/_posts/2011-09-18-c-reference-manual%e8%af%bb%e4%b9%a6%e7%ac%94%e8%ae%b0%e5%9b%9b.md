---
id: 353
title: c reference manual读书笔记(四)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=353'
permalink: /2011/09/18/c-reference-manual%e8%af%bb%e4%b9%a6%e7%ac%94%e8%ae%b0%e5%9b%9b/
categories:
  - 语言规范
tags:
  - c
  - 语言规范
  - 读书笔记
translate_title: c-reference-manual-reading-notes-(four)
date: 2011-09-18 14:21:01
---
主要是介绍c语言中的类型

1 c语言里面分为数值类型(Arithmetic，包括整数，浮点.enum 等),指针类型，数组类型，结构类型，联合类型，函数类型以及void类型。c99中添加了_Bool _Complex _Imaginary三种新的类型。

2 标准c只定义了整数类型的最小精度。char至少8位，short/int至少16位，long至少32位。 long long至少64位。所有的范围都保存在limits.h中。

3 一个有符号的整数所能表示的范围，不仅依赖于位数，还依赖于2进制编码，比如现在的计算机一般都是使用2进制补码的表示(two&#8217;s complement)。这里要注意整数类型默认都是signed.

4 一个有符号的整数和无符号的混合的表达式(四则运算，比较等),都会先将有符号的整数转为无符号整数再进行。而所有的无符号的算术运算，最后都会把结果对2的n次方取摸。
  
<!--more-->


  
5 char类型分为3种，分别是char/signed char/unsigned char,这里要注意 第一种是plain char，这个就是依赖于编译器的实习了，有些编译器会把char当作signed，有些会当作unsigned。所以这里要非常非常注意，如果一定要用有符号的char，则就使用signed char，或者直接使用int。

6 浮点数分为3种类型，float/double/long double.这三种类型的使用和short/int/long很类似。有关浮点数的范围等定义都在float.h中。

7 指针的大小依赖于实现，在c99中提供了一个类型intptr_t来表示指针的大小，而在以前只能假设指针的大小至少和long一样。
  
在c里面空指针NULL，是一个宏。

8 void指针不能被dereference.并且void指针的实现可能会不同于其他类型的指针。函数指针和数据指针也有可能是不同大小的。

9 一般来说不管什么时候，当一个数组被分配内存的时候，它的大小必须是已知的，但是也有一些例外的情况，比如extern修饰的数组的时候，它的size可以被忽略，因为它在别处被定义。
  
```
  
extern int a[];

int sum(int n)
  
{
     
int i, s = 0;
     
for (i = 0; i < n; i++) {
          
s+a[i];
      
}

return s;
  
}
  
```

10 变长数组不能被声明为static或者extern的，并且变长数组只能声明在block域中。还有一种叫做variably modified types，这种就是变长数组只是作为整个声明的一部分，比如下面这种
  
```
  
void f(int b_size) {
   
static int (*c) \[5\]\[b_size\];
  
}
  
```
  
不过要注意无论如何变长数组只能出现在block中，包括上面的variably modified types.

11 变长数组作为函数参数，它如果使用了另外的参数，则严格依赖于c的词法分析。
  
```
  
//正确
  
void f(int r, int a[r]);
  
//错误
  
void f(int a[r], int r);
  
```

12 enum类型在c语言中表示一个整数的集合，第一个值默认是0.

13 结构体中的位域可以有3种类型，分别是 unsigned int/signed int/int，其中int和char类似，也就是依赖于编译器的实现，有可能是无符，也有可能是有符号。 并且位域不能被取地址。

14 结构体的结束地址和起始地址必须拥有相同的对齐.也就是说看结构体的第一个元素的对齐策略，比如下面的例子：
  
```

#include <stdio.h>
  
#include <stdlib.h>

typedef void (*f) (void);

struct S {
      
int c1;
      
char c;
  
};

struct S2 {
      
double value;
      
char name[10];
  
};

int main() {
      
printf("%d %d\n", sizeof(struct S), sizeof(struct S2));
  
}
  
```
  
假设double必须以8个字节对齐, int必须以4个字节对齐，则这里结果会是8 和24。也就是添加了一些对齐。

15 union类型，很多和struct类似，比如一个union的大小就是这个union中最大的元素加上对应的padding。标准c允许函数的extern 和 static 分开修饰，typedef定义的新的类型名，能够被block内部所重定义，比如下面的例子：
  
```
  
typedef int T;
  
int main()
  
{
      
float T;
  
}
  
```
  
T就被重定义。