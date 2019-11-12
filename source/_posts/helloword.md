
---
title: helloworld生命周期
tags: [逆向,计算机系统]
categories: helloworld生命周期
date: 2019-08-21 14:20:00
 
---


在我们要学习一门编程语言的时候第一个程序基本上是helloworld，但就是这个简单的程序到底是如何运行的，在此做一下记录。
![image.png](https://upload-images.jianshu.io/upload_images/3941016-511f2343273f9a90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!--more-->

> 文章中的部分内容来自《深入理解计算机系统》第一章
helloworld  从到创建，执行，输出简单消息，再到终止，中间到底是如何运行的。简单介绍下。

##   代码
```c
 #include<stdio.h>
int main(int argc, char const *argv[])
{
	printf ("hello  world \n");
	return 0;
}
```

##  二进制文本
-   计算机中，数据的存储我们看到的是一串英文代码，其实存储的是2进制数据，helloworld 程序的代码可以用 ASCLL码表示。再把具体的ASCLL码转为为2进制。


![16进制的hello.c](https://upload-images.jianshu.io/upload_images/3941016-2bd37d49ef16eac2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 如图中的23对应的10进制数据为35 而35对应的为#
![image.png](https://upload-images.jianshu.io/upload_images/3941016-a0c71f1f0af07306.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##  程序被其他程序翻译成不同的格式

hello程序从一个高级的c程序开始因为这样更容易让人读懂。但是如果要运行hello这个程序，必须要转为编译成更低的机器语言。


1. 在Linux 中从源程序到目标程序是由编译器完成的。
![image.png](https://upload-images.jianshu.io/upload_images/3941016-bf20f3ee15935569.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 可以用file 命令查看目标程序的详细信息

![image.png](https://upload-images.jianshu.io/upload_images/3941016-be2300fbb8a4d43f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 > 可以看出这个程序的格式为elf(对应的Windows下的编译的为pie) 64为程序，因为处理器是向下兼容的如果要生成32位程序可以使用  (-m32)   


## 在编译中到底经历了哪些阶段

![image.png](https://upload-images.jianshu.io/upload_images/3941016-27119309c5367d45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. 预处理阶段（cpp）根据以#号开头的命令，修改原始的c程序，比如helloworld 中的第一行 #include<stdio.h>命令告诉预处理器读取系统头文件stdio.h 的内容.并把他直接插入程序文本中，得到一个c程序 ,通常是以 .i 为扩张名。stdio.h 文件在Linux中的（/usr/include目录下) 



![image.png](https://upload-images.jianshu.io/upload_images/3941016-ac34c48610009d5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




2. 编译阶段，汇编器(ccl)为把上面的helloworld.i  反义成hello.s他包含一个汇编语言程序，用（ida查看汇编程序）


![IDA查看](https://upload-images.jianshu.io/upload_images/3941016-ca0d9998e651a056.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![objdump查看](https://upload-images.jianshu.io/upload_images/3941016-f13d94bc7d0c3e12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


3.  汇编阶段：把源程序的helloworld.s  翻译成机器语言指令。并将结果保存在  helloworld.o文件中，helloworld.o 是一个2进制文件


4. 链接阶段： 源程序中的hellworld 调用了printf 函数  ，他是每一个C 编辑器提供的标准c库中的一个函数，printf 函数存在一个名为printf.o的单独的预编译好的目标文中中，而这个文件必须以某种形式合并到我们helloworld.o程序中链接器(ld)就负责这种合并。结果生产一个可执行的文件为helloworld。

### ida 查看链接的printf 函数

![image.png](https://upload-images.jianshu.io/upload_images/3941016-7c820f4d3de67eb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 运行helloworld程序：


![image.png](https://upload-images.jianshu.io/upload_images/3941016-07bad11a13279098.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




![image.png](https://upload-images.jianshu.io/upload_images/3941016-cc721e884c507feb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![image.png](https://upload-images.jianshu.io/upload_images/3941016-3f2b1409d002282b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


