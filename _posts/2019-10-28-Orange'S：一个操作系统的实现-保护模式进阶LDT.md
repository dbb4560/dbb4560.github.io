---
layout:     post
title:      Orange'S：一个操作系统的实现
subtitle:   保护模式进阶LDT
date:       2019-10-28
header-img: img/post-first-blog-web.jpg
catalog: true
tags:
    - x86
    - 操作系统
    - 保护模式
    - 进阶
    - LDT
  
---

### 保护模式进阶LDT

之前讲述了进入保护模式显示字符串和返回dos的步骤！

LDT与GDT是差不多的，区别在于
1. 全局（Global）和局部（local）.
2. LDT表存放在LDT类型的段之中，此时GDT必须含有LDT的段描述符；
3. LDT本身是一个段，而GDT不是。

查找GDT在线性地址中的基地址，需要借助GDTR；而查找LDT相应基地址，需要的是GDT中的段描述符。访问LDT需要使用段选择符，为了减少访问LDT时候的段转换次数，LDT的段选择符，段基址，段限长都要放在LDTR寄存器之中。

本节对应的是pmtest3.asm
代码有300行，就不附上了

如图两个代码( pmtest3 vs pmtest2 ) 对比：

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191029163919.png)

