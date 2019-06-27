---
id: 319
title: c reference manual读书笔记(三)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=319'
permalink: /2011/07/30/c-reference-manual%e8%af%bb%e4%b9%a6%e7%ac%94%e8%ae%b0%e4%b8%89/
categories:
  - 语言规范
tags:
  - c
  - 语言规范
  - 读书笔记
translate_title: c-reference-manual-reading-notes-three
date: 2011-07-30 15:12:44
---
这次主要是针对c语言中的声明(declaration)定义.

1 在c语言中声明的作用域(scope)可以分为4种，分别是 block/local scope, prototype scope, function scope以及file scope.其中file scope是最顶层的域. 如果一个声明不是external的，则默认这个声明就是被限制在它所出现的文件中.

2 一个标示符在它被完全声明之前是不能使用的。更精确的说就是从我们定义一个声明到这个声明的结束之间不能出现这个标示符.可是下面的例子是完全正确的，这是因为使用intsize(sizeof)之前，它已经被声明了:
  
```
  
static int intsize = sizeof(intsize)
  
```
  
而下面的例子则会编译出错，因为Test在使用之前还没有被声明.
  
```
  
typedef struct {Test *s} Test;
  
```

3 c语言中也支持forward reference，不过只允许下面三种类型,分别是 在goto语句中的label，label都是在goto语句之后声明的；还有就是允许在struct,union,array类型的完全声明之前使用当前的声明，比如下面的例子
  
```
  
typedef struct s {struct s *n} T;
  
```
  
最后一种例外是函数的声明，这个原因是在c99之前，c语言允许隐式(implicitly)的声明，也就是如果一个函数在调用之前没有被声明，则在函数被调用的地方会隐式的声明这个函数(后面会详细介绍这个)。不过c99是不允许隐式的声明的.
  
<!--more-->


  
4 c语言中允许5种类型的overload，overload是指相同名字的标示符。他们分别是 预处理器的宏名字，goto语句中的label，结构体/union/enum中的tag或者元素,最后剩下的归于一类.

5 之所以需要将overload分为5种类型，是为了更好的判断重复的声明。在c语言中如果相同的overloading类型中有2个或以上的相同名字的声明（在相同的block或者top-level(也就是file cope))，则将会报错 duplicate声明.比如下面的例子中howmany会报错，而str不会，这是因为str是属于同一个overloading class:
  
```
  
int howmany;
  
char str[10];
  
typedef double howmany;
  
struct str {int a;} x;
  
```
  
不过重复的声明也有例外，比如extern的声明，它可以出现任意多个.

6 c语言中的scope规则是，一个name的scope是开始于它定义的那个点，而不是它所被定义的block的开头，因此就会出现在相同的block的不同部分出现两个不冲突的定义同时被使用,比如下面的例子,将不会出错:
  
```
  
int i;
      
i = 10;
      
{
          
int j = i;
          
int i = 5;
      
}
  
```

7 c语言中变量和函数都有生命周期，分为3种类型的生命周期，分别是static, local,以及dynamic.其中所有的函数都是static的.也就是程序启动时创建，退出时销毁.

8 c语言中静态变量默认会被初始化为0.而其他类型的变量，默认不会被初始化，初始值是没有定义的.并且静态变量只被初始化一次，哪怕包含这个静态变量的函数或者block被执行多次.

9 如果有多个extern修饰的声明，他们的类型是不兼容的，则结果是未定义的，不过在gcc中会直接报错.

10 c语言中有5种存储类修饰符(storage-class-specifier),分别是auto/extern/register/static/typedef.所有的top-level(file) scope默认都是extern的，除非你显式的制定存储类修饰符.而在block或者函数中的声明默认都是auto的.

11 c99之前函数的定义或者定义允许忽略返回值类型，默认为int，不过c99就不允许了.
  
```
  
test()
  
{} //默认返回值是int
  
[c/]

12 type qualifier包括const/volatile/restrict.如果有多个相同的type qualifier，在c99中则会忽略后面的.在一个top-level(file)的一个const声明，默认是extern的.

13 初始化&#8212;整形的初始化，如果声明是static或者extern的，则初始化的值必须是常量，否则可以是任何表达式,而默认的static的整形的初始化值是0.
  
浮点的初始化和整形的规则类似，不过它的初始化值是0.0.
  
指针的初始化,也是必须使用常量表达式，不过这里的常量表达式是指指向一个类型T的指针,static 指针的初始化值是NULL.下面是几个例子.
  
```
  
static int k; //k为0
  
static int i = 0;
  
static int j = i; //错误
  
static float m;//m为0.0
  
static float n = m;//错误
  
static int *t;//t为NULL
  
static int *u = &k;//u为k的指针
  
```

14 数组的初始化，如果初始化没有指定数组的大小，则数组的大小就是初始化时候的大小.如果初始化给的值的个数小于数组本身的大小，则剩余的元素将会被初始化为他们的各自类型默认的初始化值.比如下面的例子:
  
```
  
int array[3] = {1, 2}; //array[2]将会是0
  
```
  
enum的初始化，和上面的整形规则类似。
  
结构体的初始化和数组的初始化有些类似，静态或者external的结构体类型能够被编译器自动初始化，每个元素将会被初始化为他们自己的默认值.静态或者external的结构体初始化不能使用另外一个结构体对它赋值。
  
```
  
struct st {
      
int s;
  
};
  
static struct st s2 = {1};
  
static struct st s5 = s2;//出错
  
```
  
联合体的初始化，和结构体类似.不过有一个要注意的地方，就是union如果使用{}进行初始化，则初始化的为第一个元素，另外一个元素的值是未定义的，比如下面的例子:
  
```
  
union un {
      
float s2;
      
int s;
  
};

union un2 {
      
int s;
      
float s2;
  
};

union un n= {1.1};//s2将会是1.1,而s是一个很大的整数.
  
union un2 n2 = {1.1};//s将会是1,而s2则是0
  
```

15 designated初始化，也就是指定初始化结构体/联合体/数组某几个元素，这时候另外的元素都会被初始化为它的默认值，不过联合体有例外，它只会初始化所指定的元素，而另外的元素的值是未定义的。

16 处理external的声明的一些规则，不同的编译器可能会使用不同的规则，不过一般来说都会选择下面的4种规则之一.Initializer/Omitted storage class(c++)/Common/Mixed common,具体这几个类型我就不介绍了，需要了解的可以去google.我这里就简要介绍下Initializer,这是因为standard就是吸收了这种模型。后面会详细分析这个.
  
在Initializer模型中，如果在top-level中出现了一个Initializer的声明，则这个声明就是定义的地方，另外所有的都是对这个声明的引用，不过全局(所有文件)只能有一个定义.比如下面的定义:
  
```
  
int x = 0;
  
```