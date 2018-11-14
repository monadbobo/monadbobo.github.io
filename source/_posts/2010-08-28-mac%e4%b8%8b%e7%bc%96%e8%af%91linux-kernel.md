---
id: 37
title: mac下编译linux kernel
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=37'
permalink: /2010/08/28/mac%e4%b8%8b%e7%bc%96%e8%af%91linux-kernel/
views:
  - '10'
bot_views:
  - '18'
categories:
  - kernel
  - mac
tags:
  - kernel
  - mac
translate_title: compile-linux-kernel-under-mac
date: 2010-08-28 09:13:06
---
这里我的gcc是4.5.1 binutils 是2.20.1 ，内核是2.6.35.3.
  
首先需要交叉编译gcc和binutils
  
port安装 gcc编译依赖的库gpm，mpfr和mpc.

然后开始编译gcc，这里有个要注意的就是需要指定gmp的include和lib路径，下面是我的config：
  
<!--more-->

> ./configure &#8211;prefix=/opt/local/x86_64_cross &#8211;target=x86_64-pc-linux &#8211;program-prefix=x86_64-elf- &#8211;without-included-gettext &#8211;enable-languages=c,c++ &#8211;without-headers &#8211;disable-nls &#8211;enable-obsolete &#8211;with-newlib &#8211;disable-libgfortran &#8211;enable-shared &#8211;with-fpmatch=sse &#8211;build=x86_64-apple-darwin10 &#8211;with-gmp-include=/opt/local/include/ &#8211;with-gmp-lib=/opt/local/lib/

然后make all-gcc,然后会遇到下面这个错误：

> Undefined symbols:
    
> &#8220;_iconv_close&#8221;, referenced from:
        
> __cpp_destroy_iconv in libcpp.a(charset.o)
        
> __cpp_destroy_iconv in libcpp.a(charset.o)
        
> __cpp_destroy_iconv in libcpp.a(charset.o)
        
> __cpp_destroy_iconv in libcpp.a(charset.o)
        
> __cpp_destroy_iconv in libcpp.a(charset.o)
        
> __cpp_convert_input in libcpp.a(charset.o)
    
> &#8220;_iconv&#8221;, referenced from:
        
> _convert_using_iconv in libcpp.a(charset.o)
        
> _convert_using_iconv in libcpp.a(charset.o)
        
> _convert_using_iconv in libcpp.a(charset.o)
        
> _convert_using_iconv in libcpp.a(charset.o)
       
> (maybe you meant: __cpp_destroy_iconv, _cpp_init_iconv )
    
> &#8220;_iconv_open&#8221;, referenced from:
        
> _init_iconv_desc in libcpp.a(charset.o)
  
> ld: symbol(s) not found 

这个问题需要这样解决：

修改 host-x86_64-apple-darwin10/gcc/Makefile找到这一行：LIBICONV = -liconv 然后添加－L/usr/libiconv。

此时可以开始编译kernel了，首先需要前面交叉编译的工具加入到path，然后开始make menuconfig，最后

> make ARCH=x86_64 CROSS_COMPILE=x86_64-pc-linux-

这个时候一般会遇到这个编译错误：

> arch/x86/boot/header.S: Assembler messages:
  
> arch/x86/boot/header.S:236: Error: can&#8217;t resolve \`VO__end&#8217; {\*UND\* section} &#8211; \`VO__text&#8217; {\*UND\* section}

这个问题是由于mac下使用的是freebsd的sed，而linux的编译使用的是gnu的sed，因此这里我们需要这样：
  
[shell]
  
x86_64-pc-linux-nm vmlinux | sed -n -E &#8216;s/^([0-9a-fA-F]*) . (_text|_end)$$/\#define VO_\2 0x\1/p&#8217; > arch/x86/boot/voffset.h
  
[/shell]
  
这个问题解决之后继续编译此时又会遇到这个错误：

> arch/x86/boot/header.o: In function \`payload_offset&#8217;:
  
> (.header+0x57): undefined reference to \`ZO_input_data&#8217;
  
> arch/x86/boot/header.o: In function \`payload_length&#8217;:
  
> (.header+0x5b): undefined reference to \`ZO_z_input_len&#8217;
  
> make[1]: \*** [arch/x86/boot/setup.elf] Error 1
  
> make: \*** [bzImage] Error 2

和上面的错误类似，也是由于不同的sed版本引起的，因此我们只需要执行下面的命令：

[shell]
  
x86_64-pc-linux-nm arch/x86/boot/compressed/vmlinux | sed -n -E &#8216;s/^([0-9a-fA-F]\*) . (startup_32|input_data|_end|z_.\*)$$/\#define ZO_\2 0x\1/p&#8217; > arch/x86/boot/zoffset.h
  
[/shell]

ok，接下来编译的话还会遇到下面的错误：

> arch/x86/boot/tools/build.c:32:27: error: sys/sysmacros.h: No such file or directory
  
> make[1]: \*** [arch/x86/boot/tools/build] Error 1
  
> make: \*** [bzImage] Error 2

这个解决更简单，那就是直接删除include这句。

最后重新make 然后编译成功。