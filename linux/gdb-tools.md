# gdb工具的使用 

http://wiki.swoole.com/wiki/page/442.html 

GDB是GNU开源组织发布的一个强大的UNIX下的程序调试工具，可以用来调试C/C++开发的程序，PHP和Swoole是使用C语言开发的，所以可以拥GDB来调试PHP+Swoole的程序。

gdb调试是命令行交互式的，需要掌握常用的指令。

## 使用方法

```sh 
gdb -p 进程ID
gdb php
gdb php core
```

gdb有3种使用方式：

- 跟踪正在运行的PHP程序，使用gdb -p 进程ID
- 使用gdb运行并调试PHP程序，使用gdb php -> run server.php 进行调试
- PHP程序发生coredump后使用gdb加载core内存镜像进行调试 gdb php core

    如果PATH环境变量中没有php，gdb时需要指定绝对路径，如gdb /usr/local/bin/php

## 常用指令

- p：print，打印C变量的值
- c：continue，继续运行被中止的程序
- b：breakpoint，设置断点，可以按照函数名设置，如b zif\_php\_function，也可以按照源代码的行数指定断点，如b src/networker/Server.c:1000
- t：thread，切换线程，如果进程拥有多个线程，可以使用t指令，切换到不同的线程
- ctrl + c：中断当前正在运行的程序，和c指令配合使用
- n：next，执行下一行，单步调试
- info threads：查看运行的所有线程
- l：list，查看源码，可以使用l 函数名 或者 l 行号
- bt：backtrace，查看运行时的函数调用栈
- finish：完成当前函数
- f：frame，与bt配合使用，可以切换到函数调用栈的某一层
- r：run，运行程序

## zbacktrace

zbacktrace是PHP源码包提供的一个gdb自定义指令，功能与bt指令类似，与bt不同的是zbacktrace看到的调用栈是PHP函数调用栈，而不是C函数。

下载php-src，解压后从根目录中找到一个.gdbinit文件，在gdb shell中输入

```sh 
source .gdbinit
zbacktrace
```

.gdbinit还提供了其他更多指令，可以查看源码了解详细的信息。


---------

Date: 2015-10-09  Author: alex cai <cyy0523xc@gmail.com>
