---
id: 302
title: c reference manual读书笔记(一)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=302'
permalink: /2011/07/03/c-reference-manual%e8%af%bb%e4%b9%a6%e7%ac%94%e8%ae%b0%e4%b8%80/
categories:
  - 语言规范
tags:
  - c
  - 语言规范
  - 读书笔记
translate_title: c-reference-manual-reading-notes-1
date: 2011-07-03 09:46:54
---
我读的是第五版的影印版，简单的做了一些笔记.下面是第二章的笔记.

1 c 源文件的字符集(character set)包含在ISO/IEC 10646的Latin block中. 包括5种类型，分别是 52个lating大小写字母，10个数字(0-9)，空格，horizontal tab(ht) vertical tab(VT) form feed(FF) 以及29个graphic 字符(这个可以去看c99的手册).而不在集合内的字符可能会出现在注释/字符常量/字符串常量/文件名中.在c中还有执行字符集(execution character)的概念,一般来说编译和运行都在相同的电脑，则source字符集和执行字符集都是相同的.

2 在c的源程序中，blank, end-of-line, vertical tab, form feed, horizontal-tab都会被认为是空格(whitespace characters).

3 c89中要求逻辑行的最大长度是509个字符，而c99是4095个字符.

4 c语言中可以使用 9个trigraphs(比如??/表示\)，而trigraphs的翻译会在词法分析之前(gcc默认编译没有打开trigraph，必须添加-trigraphs命令，或者使用-std命令指定c标准).

5 多字节字符(multibytes characters),主要针对非英语的环境.编码实现分为state-independent和state-depend,顾名思义，一个是编码依赖于前一个多字节字符，一个是不依赖于前一个多字节字符.
  
<!--more-->


  
6 c语言中的comments是在预处理器处理之前就被remove的(标准c将会替换comments为为一个空格). 只有一些非标准的c实现才支持nestable comments,因此在c中注释大段的代码最好使用#if 0 #endif这种形式.c99添加了c++的注释形式(//).

7 c编译器会按照从左到右的顺序尽可能的收集最长的正确的token(比如b&#8212;x也就等同与b&#8211; &#8211; x).c语言还包括了一些alternate token spellings,比如<%等同于{. 8 一个标示符(identifier)不能以一个数字开始，并且它不能和预定义的keyword相同(大小写敏感).它是由latin字母，数字，下划线组成。c99开始，标示符也可能会包含一些universal character name. 9 两个identifier如果拼写相同(大小写敏感)，则他们就是一样的。这里建议使用下划线+全小写的命名方式。由于不同的c标准对于identifier的长度限制不同，因此使用extern的时候要小心。 10 c99一共包括37个关键字,在c99中添加了一个预定义的identifier， __func__,它主要用于debug程序，也就是当前的函数名. 11 c中的常量(contant)包括4种类型，分别是整数，浮点数，字符，字符串. 12 整数常量可以使用16进制，8进制以及10进制表示。而且常量末尾可以跟上一个suffix声明这个常量的类型(l/L表示long,ll/LL表示long long, u/U表示unsigned).一个整数常量在不溢出的条件下，永远是正数。负号不是整数常量的一部分，负号是一个一元操作符。如果一个整数超过了它所表示的类型的范围，则结果是未定义的.相关的范围在limits.h以及stdint.h和inttypes.h(c99)中。 并且c99提供了一套可移植(portable)宏(INTX_c,UINTX_c, INTMAX_c等)来控制整数常量的类型和大小. ``` # define INT8_C(c) c # define INT16_C(c) c # define INT32_C(c) c # if __WORDSIZE == 64 # define INT64_C(c) c ## L # else # define INT64_C(c) c ## LL # endif /\* Unsigned. \*/ # define UINT8_C(c) c # define UINT16_C(c) c # define UINT32_C(c) c ## U # if \__WORDSIZE == 64 # define UINT64_C(c) c ## UL # else # define UINT64_C(c) c ## ULL # endif /\* Maximal type. \*/ # if __WORDSIZE == 64 # define INTMAX_C(c) c ## L # define UINTMAX_C(c) c ## UL # else # define INTMAX_C(c) c ## LL # define UINTMAX_C(c) c ## ULL # endif ``` 13 浮点常量，c99可以使用16进制来表示浮点了。如果一个浮点数不能被精确表示，实现将会选择一个最接近的表示.如果浮点数对于它所能表示的范围太大或者太小，则结果也是未定义的. 14 字符常量,如果不是多字节的字符常量(不是以L开头的)都会有类型int.也就是会从类型char转换为int(符号扩展)，比如 下面的代码 ```u_char d; char e; d = '\377'; e = '\377'; ``` e其实就是-1，而d就是255.并且sizeof('c)在c中其实就是sizeof(int). 15 字符串常量, 每一个包含n个字符的字符串常量，在运行时都会静态的分配n+1个字符的长度的内存，其中最后一个1保存\0.永远不要去修改字符串常量，因为这块内存有可能是只读的。而且不要依赖于所有的字符串常量都是存储在不同的地址，c标准允许相同的字符串常量存放在一个地址。 16 转义符，在c语言中所有的(\+小写字母)都是为将来的语言扩展所保留的.如果紧跟着\的字符没有定义(非数字，x，n/t/b/r/f/v/\/'/"/a/?)，则结果也是未定义的. 16 c++有类型char，比如sizeof('c')在c和c++中是不一样的，c++里面就是sizeof(char),而c是sizeof(int).