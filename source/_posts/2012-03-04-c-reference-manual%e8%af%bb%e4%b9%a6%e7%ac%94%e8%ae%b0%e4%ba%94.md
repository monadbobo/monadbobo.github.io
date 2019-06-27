---
id: 371
title: c reference manual读书笔记(五)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=371'
permalink: /2012/03/04/c-reference-manual%e8%af%bb%e4%b9%a6%e7%ac%94%e8%ae%b0%e4%ba%94/
posturl_add_url:
  - 'yes'
categories:
  - 语言规范
tags:
  - c
  - 语言规范
  - 读书笔记
translate_title: c-reference-manual-reading-notes-five
date: 2012-03-04 10:58:23
---
这次主要是介绍一些c语言中的类型转换。

1 在c语言里面，除了位域之外所有的数据对象都是表示为一堆抽象的存储单元,每一个存储单元都是由很多位组成的，每位的值不是1就是0，并且每一个存储单元都必须有唯一的地址，并且他们的每一个的大小都是和char的大小一致。每个存储单元有多少位，c语言并没有要求，可是每个存储单元必须能够存储基本字符集的每一个字符。 在标准c中也称存储单元为byte。

2 c程序员一般来说不需要在意一些机器的对齐限制，因为编译器会做对齐操作，但是，c也给了程序员忽略对齐限制的能力，比如cast一个指针到另外的类型。不过这里要注意，假设你要将一个类型为S的指针转换成类型T的指针，此时如果S的对齐因子如果不小于T的对齐因子，那么这个转换就是安全的(也就是说此时你取T的值，是没问题的). 可是如果S的对齐因子如果小于T的对齐因子，此时会出现两种情况，第一种就是当使用转换后的指针时，直接出错。第二种就是硬件或者编译器来帮你找到一个合适的指针来使用这个地址。

3 c99中定义了指针类型uintptr_t和intptr_t这两个值并不是指针的大小，而是指针类型不会超过这两个值, 这里要注意，很多实现中函数指针和数据指针大小是不一样的。而NULL只是数据空指针。对象的值和对象类型的值是不一样的，因为会有一些padding，而这些padding是没有定义的，
  
<!--more-->


  
4 c语言中的类型转换有下面几种方式。
    
4.1 显式的转换
    
4.2 执行算数或者逻辑操作时，编译器会做隐试的转换
    
4.3 执行赋值操作时，也会做隐式的转换。
    
4.4 传递给一个函数的参数，也会做隐式的转换
    
4.5 函数的返回值也有可能会被隐式的转换为函数声明的返回值。

算数类型和指针都能被转换为整形，除了_Bool类型之外，整型类型之间的所有的转换都是尽量使得转换后的结果尽量和转换前相等。_Bool类型的转换很特殊，当转换一个算数类型的值到Bool，如果值是0，则就转换为0，负责就是1.

浮点数转整形，如果浮点数有非零的frac部分，则会直接被丢弃。这里有一个特殊情况，就是当浮点数的大小比目的类型的大小大，比如一个负的浮点数转换为一个无符整型，此时结果就是未定义的。

浮点数之间的转换，当从小的类型转为大的类型，比如float 到 double，比如double到 long double，这时转换结果都是不变的，可是如果从大的类型转为小的类型，这时也有两种情况，第一种是值刚好能容纳到小的类型中，此时则选择一个最接近原始值的值。而如果值太大不能容纳到小类型，此时结果就是未定义的。

Enum类型的转换和整形类似，而Struct和Union是不允许直接转换的。

Null指针并不意味着它的所有位都是0，不同的指针类型的Null指针可能有不同的内部表示。类型T的数组要转换成指向T的指针，其实就是这个数组的第一个元素。

标准c保证当一个指针转换为void \*，然后再转换回原来的类型，原始值是不会变化的。这个对于char \*也是适合的。

c99中包含了一个conversion rank，来帮助解释类型转换，rank越大，那么说明它的转换优先级越高，比如char的rank是20，int的rank是40(可以看下面从c99中截取的相关部分).而对应的转换规则也需要去翻阅c99文档。比如任何一个小于int的rank无符类型，则则都会转换为int类型。位域都的rank都是比int小的。

> — No two signed integer types shall have the same rank, even if they have the same
  
> representation.
  
> — The rank of a signed integer type shall be greater than the rank of any signed integer type with less precision.
  
> — The rank of long long int shall be greater than the rank of long int, which shall be greater than the rank of int, which shall be greater than the rank of short int, which shall be greater than the rank of signed char.
  
> — The rank of any unsigned integer type shall equal the rank of the corresponding signed integer type, if any.
  
> — The rank of any standard integer type shall be greater than the rank of any extended integer type with the same width.
  
> — The rank of char shall equal the rank of signed char and unsigned char.
  
> — The rank of _Bool shall be less than the rank of all other standard integer types.
  
> — The rank of any enumerated type shall equal the rank of the compatible integer type (see 6.7.2.2).
  
> — The rank of any extended signed integer type relative to another extended signed integer type with the same precision is implementation-defined, but still subject to the other rules for determining the integer conversion rank.
  
> — For all integer types T1, T2, and T3, if T1 has greater rank than T2 and T2 has greater rank than T3, then T1 has greater rank than T3.