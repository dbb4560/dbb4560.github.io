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
代码有300行，就不附上了！

如图两个代码( pmtest3 vs pmtest2 ) 对比：

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191029163919.png)

可以看出代码变化不多，删除了 TestRead 和  TestWrite 函数！ 删除了LABEL_DESC_TEST 描述符和相应的段选择子！

增加了 LABEL_DESC_LDT 描述符！


描述符：

新增

` LABEL_DESC_LDT:    Descriptor       0,        LDTLen - 1, DA_LDT	; LDT `

修改了数据段描述符：

` LABEL_DESC_DATA:   Descriptor       0,       DataLen - 1, DA_DRW+DA_DPL1	; Data `

增加了LDT 段选择子
`SelectorLDT		equ	LABEL_DESC_LDT		- LABEL_GDT`

新增了如下代码：

```

    ; 初始化 LDT 在 GDT 中的描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_LDT
	mov	word [LABEL_DESC_LDT + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_LDT + 4], al
	mov	byte [LABEL_DESC_LDT + 7], ah

	; 初始化 LDT 中的描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_CODE_A
	mov	word [LABEL_LDT_DESC_CODEA + 2], ax
	shr	eax, 16
	mov	byte [LABEL_LDT_DESC_CODEA + 4], al
	mov	byte [LABEL_LDT_DESC_CODEA + 7], ah

```

```
	; Load LDT
	mov	ax, SelectorLDT
	lldt	ax

	jmp	SelectorLDTCodeA:0	; 跳入局部任务
```

```

; LDT
[SECTION .ldt]
ALIGN	32
LABEL_LDT:
;                            段基址       段界限      属性
LABEL_LDT_DESC_CODEA: Descriptor 0, CodeALen - 1, DA_C + DA_32 ; Code, 32 位

LDTLen		equ	$ - LABEL_LDT

; LDT 选择子
SelectorLDTCodeA	equ	LABEL_LDT_DESC_CODEA	- LABEL_LDT + SA_TIL
; END of [SECTION .ldt]


; CodeA (LDT, 32 位代码段)
[SECTION .la]
ALIGN	32
[BITS	32]
LABEL_CODE_A:
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子(目的)

	mov	edi, (80 * 12 + 0) * 2	; 屏幕第 10 行, 第 0 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'L'
	mov	[gs:edi], ax

	; 准备经由16位代码段跳回实模式
	jmp	SelectorCode16:0
CodeALen	equ	$ - LABEL_CODE_A
; END of [SECTION .la]

```

第218行 ` lldt	ax ` 加载  lldt 跟 lgdt有点类似，负责加载ldtr，它的操作数是一个选择子，这个选择子对应的就是用来描述LDT 的那个描述符（标号LABEL_DESC_LDT）.

LDT 跟GDT差不多，LDT 中只有一个描述符（标号LABEL_LDT_DESC_CODEA处），这个描述符跟GDT 中的描述符没什么分别！选择子是不一样的，多了SA_TIL 属性！ 定义在pm.inc
定义如下 ： `SA_TIL EQU 4 `

分析一下就知道SA_TIL 将选择子SelectorLDTCodeA的TI位置为1。
这位是区别LDT 和 GDT 的关键所在！ TI 被置位，那么系统将从当前的LDT 中寻找相应的操作符。 也就是用到SelectorLDTCodeA时，系统会从LDT 中找到标号LABEL_LDT_DESC_CODEA描述符，并跳转到相应的段中。


本代码就只打印一个字符"L"

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191030131253.png)