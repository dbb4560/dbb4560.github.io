---
layout:     post
title:      Orange'S：一个操作系统的实现
subtitle:   ELF文件格式解析
date:       2019-11-03
header-img: img/post-first-blog-web.jpg
catalog: true
tags:
    - x86
    - 操作系统
    - ELF文件
    - Linux
    - 解析

---

### ELF文件格式

ELF = Executable and Linkable Format，可执行连接格式，是UNIX系统实验室（USL）作为应用程序二进制接口（Application Binary Interface，ABI）而开发和发布的。扩展名为elf。

其主要有三种主要类型: 
适于连接的可重定位文件(relocatable file)，可与其它目标文件一起创建可执行文件和共享目标文件。 
适于执行的可执行文件(executable file)，用于提供程序的进程映像，加载的内存执行。 
共享目标文件(shared object file)，连接器可将它与其它可重定位文件和共享目标文件连接成其它的目标文件，动态连接器又可将它与可执行文件和其它共享目标文件结合起来创建一个进程映像。


文件格式

为了方便和高效，ELF文件内容有两个平行的视角:一个是程序连接角度，另一个是程序运行角度，如图所示。

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191119231917.png)

ELF header在文件开始处描述了整个文件的组织，Section提供了目标文件的各项信息（如指令、数据、符号表、重定位信息等），Program header table指出怎样创建进程映像，含有每个program header的入口，section header table包含每一个section的入口，给出名字、大小等信息。其中Segment与section的关系后面会讲到。

要理解这个图，我们还要认识下ELF文件相关的几个重要的结构体：


1，ELF文件头
像bmp、exe等文件一样，ELF的文件头包含整个文件的控制结构。它的定义如下，可在/usr/include/elf.h中可以找到文件头结构定义： 

```
/* The ELF file header.  This appears at the start of every ELF file.  */

#define EI_NIDENT (16)

typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf32_Half	e_type;			/* Object file type */
  Elf32_Half	e_machine;		/* Architecture */
  Elf32_Word	e_version;		/* Object file version */
  Elf32_Addr	e_entry;		/* Entry point virtual address */
  Elf32_Off	e_phoff;		/* Program header table file offset */
  Elf32_Off	e_shoff;		/* Section header table file offset */
  Elf32_Word	e_flags;		/* Processor-specific flags */
  Elf32_Half	e_ehsize;		/* ELF header size in bytes */
  Elf32_Half	e_phentsize;		/* Program header table entry size */
  Elf32_Half	e_phnum;		/* Program header table entry count */
  Elf32_Half	e_shentsize;		/* Section header table entry size */
  Elf32_Half	e_shnum;		/* Section header table entry count */
  Elf32_Half	e_shstrndx;		/* Section header string table index */
} Elf32_Ehdr;

```

其中e_ident的16个字节标识是个ELF文件（7F+’E’+’L’+’F’）。 
e_type表示文件类型，2表示可执行文件。 
e_machine说明机器类别，3表示386机器，8表示MIPS机器。 
e_entry给出进程开始的虚地址，即系统将控制转移的位置。 
e_phoff指出program header table的文件偏移。 
e_phentsize表示一个program header表中的入口的长度（字节数表示）。 
e_phnum给出program header表中的入口数目。类似的。 
e_shoff，e_shentsize，e_shnum 分别表示section header表的文件偏移，表中每个入口的的字节数和入口数目。 
e_flags给出与处理器相关的标志。 
e_ehsize给出ELF文件头的长度（字节数表示）。 
e_shstrndx表示section名表的位置，指出在section header表中的索引。

由于ELF文件力求支持从8位到32位不同架构的处理器，所以才定义了表中这些数据类型，从而让文件格式与机器无关！
  
名称     |        大小   |   对齐   |   用途
-|-|-|-

Elf32_Addr   |  4   | 4   |  无符号程序地址

Elf32_Half   |   2   |  2  |  无符号中等大小整数

Elf32_Off    |  4    |   4   |  无符号文件偏移

Elf32_Sword   |  4   |    4   |  有符号大整数

Elf32_word    |  4   |    4    |   无符号大整数

unsigned char |   1   |   1   |   无符号小整数


xxd命令看一下foobar 

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191120222451.png)

表示显示0-80个地址，16进制！

e_type 在foobar  是2表示可执行文件。 
e_machine的值是3表示386机器。 
e_entry给出进程开始的虚地址，即系统将控制转移的位置，在foobar中地址是0x80480A0。 
e_phoff指出program header table的文件偏移,这里的值是0x34。 
e_phentsize表示一个program header table表中的入口的长度（字节数表示），这里的值是0x20。 
e_phnum给出program header表中的入口数目。这里的值是3个，类似的。 
e_shoff，e_shentsize，e_shnum 分别表示section header表的文件偏移，这里的值是0x107C，表中每个入口的的字节数，这里的值是0x20和入口数目，这里的值是6个。 
e_flags给出与处理器相关的标志，针对IA32而言，此项目是0。 
e_ehsize给出ELF文件头的长度（字节数表示）。 
e_shstrndx表示section名表的位置，指出在section header表中的索引。


2,Program header
目标文件或者共享文件的program header table描述了系统执行一个程序所需要的段或者其它信息。目标文件的一个段（segment）包含一个或者多个section。Program header只对可执行文件和共享目标文件有意义，对于程序的链接没有任何意义。结构定义如下，可在/usr/include/elf.h中可以找到文件头结构定义： 
 
```

/* Program segment header.  */

typedef struct
{
  Elf32_Word	p_type;			/* Segment type */
  Elf32_Off	p_offset;		/* Segment file offset */
  Elf32_Addr	p_vaddr;		/* Segment virtual address */
  Elf32_Addr	p_paddr;		/* Segment physical address */
  Elf32_Word	p_filesz;		/* Segment size in file */
  Elf32_Word	p_memsz;		/* Segment size in memory */
  Elf32_Word	p_flags;		/* Segment flags */
  Elf32_Word	p_align;		/* Segment alignment */
} Elf32_Phdr;


```

实际上Program header描述的是系统准备程序运行所需的一个段（Segment）或其他信息。   举例，看看foobar 的情况。程序头表共有三项（e_phnum=3）,因为前面得知是0x34后面就是program，且大小是0x20.
所以偏移量是0x34~0x53 , 0x54~0x73 , 0x74~0x93.

xxd命令看一下地址

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191120223126.png)

其中p_type描述段的类型； 
p_offset给出该段相对于文件开关的偏移量； 
p_vaddr给出该段所在的虚拟地址； 
p_paddr给出该段的物理地址； 
p_filesz给出该段的大小，在字节为单元，可能为0； 
p_memsz给出该段在内存中所占的大小，可能为0； 
p_filesze与p_memsz的值可能会不相等。

Program header描述的是一个段在文件中的位置、大小以及它被放进内存后所在的位置和大小。如果我们想把一个文件加载进内存的话，需要的正是这个信息！可执行文件中，一个program header描述的内容称为一个段（segment）。Segment包含一个或者多个section!

在foobar 共有三个Program header 取值如表所示

名称     |  Program header 0  | Program header 1 | Program header 2
-|-|-|-

P_type   | 0x01  | 0x01  |  0x6474E551

p_offset   |   0x00   |  0x1000 |  0

p_vaddr    |  0x8048000    |   0x804A000   |  0

p_paddr  |  0x8048000   |    0x804A000    |  0

p_filesz  |  0x194  |    0x14    |   0

p_memsz  | 0x194    |    0x14    |   0

p_flags  |  0x05   |    0x06    |   0x07

p_align |  0x1000  |    0x1000    |   0x10


根据信息就知道foobar加载进内存之后的情形！

Section Header

目标文件的section header table可以定位所有的section，它是一个Elf32_Shdr结构的数组，Section头表的索引是这个数组的下标。有些索引号是保留的，目标文件不能使用这些特殊的索引。 
Section包含目标文件除了ELF文件头、程序头表、section头表的所有信息，而且目标文件section满足几个条件： 
目标文件中的每个section都只有一个section头项描述，可以存在不指示任何section的section头项。 
每个section在文件中占据一块连续的空间。 
Section之间不可重叠。 
目标文件可以有非活动空间，各种headers和sections没有覆盖目标文件的每一个字节，这些非活动空间是没有定义的。 
Section header结构定义如下，可在/usr/include/elf.h中可以找到文件头结构定义： 

```

/* Section header.  */

typedef struct
{
  Elf32_Word	sh_name;		/* Section name (string tbl index) */
  Elf32_Word	sh_type;		/* Section type */
  Elf32_Word	sh_flags;		/* Section flags */
  Elf32_Addr	sh_addr;		/* Section virtual addr at execution */
  Elf32_Off	sh_offset;		/* Section file offset */
  Elf32_Word	sh_size;		/* Section size in bytes */
  Elf32_Word	sh_link;		/* Link to another section */
  Elf32_Word	sh_info;		/* Additional section information */
  Elf32_Word	sh_addralign;		/* Section alignment */
  Elf32_Word	sh_entsize;		/* Entry size if section holds table */
} Elf32_Shdr;

```

其中sh_name指出section的名字，它的值是后面将会讲到的section header string table中的偏移，指出一个以null结尾的字符串。 
sh_type是类别。 
sh_flags指示该section在进程执行时的特性。 
sh_addr指出若此section在进程的内存映像中出现，则给出开始的虚地址。 
sh_offset给出此section在文件中的偏移。其它字段的意义不太常用，在此不细述。

文件的section含有程序和控制信息，系统使用一些特定的section，并有其固定的类型和属性（由sh_type和sh_info指出）。下面介绍几个常用到的section:“.bss”段含有占据程序内存映像的未初始化数据，当程序开始运行时系统对这段数据初始为零，但这个section并不占文件空间。“.data.”和“.data1”段包含占据内存映像的初始化数据。“.rodata”和“.rodata1”段含程序映像中的只读数据。“.shstrtab”段含有每个section的名字，由section入口结构中的sh_name索引值来获取。“.strtab”段含有表示符号表(symbol table)名字的字符串。“.symtab”段含有文件的符号表，在后文专门介绍。“.text”段包含程序的可执行指令。



Symbol Table

目标文件的符号表包含定位或重定位程序符号定义和引用时所需要的信息。符号表入口结构定义如下，可在/usr/include/elf.h中可以找到文件头结构定义： 

```

/* Symbol table entry.  */

typedef struct
{
  Elf32_Word	st_name;		/* Symbol name (string tbl index) */
  Elf32_Addr	st_value;		/* Symbol value */
  Elf32_Word	st_size;		/* Symbol size */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char	st_other;		/* Symbol visibility */
  Elf32_Section	st_shndx;		/* Section index */
} Elf32_Sym;

```

其中st_name包含指向符号表字符串表(strtab)中的索引，从而可以获得符号名。 
st_value指出符号的值，可能是一个绝对值、地址等。 
st_size指出符号相关的内存大小，比如一个数据结构包含的字节数等。 
st_info规定了符号的类型和绑定属性，指出这个符号是一个数据名、函数名、section名还是源文件名；并且指出该符号的绑定属性是local、global还是weak。


书上并没有继续分析sector 和symbol 的信息，不过网上有一大堆，可以参考学习！



### 从Loader到内核

之前学习的Loader需要做的工作就是
- 加载内核到内存
- 跳入保护模式

用Loader加载ELF 文件

定义了一个常量头文件，用于boot.asm 和 loader.asm之间共享！把FAT12文件有关的内容写进一个单独的文件（文件名为fat12hdr.inc）
在两个文件的开头相应位置分别包含进去！

```

1 	; FAT12 磁盘的头
2 	; ----------------------------------------------------------------------
3 	BS_OEMName	DB 'ForrestY'	; OEM String, 必须 8 个字节
4 
5 	BPB_BytsPerSec	DW 512		; 每扇区字节数
6 	BPB_SecPerClus	DB 1		; 每簇多少扇区
7 	BPB_RsvdSecCnt	DW 1		; Boot 记录占用多少扇区
8 	BPB_NumFATs	DB 2		; 共有多少 FAT 表
9 	BPB_RootEntCnt	DW 224		; 根目录文件数最大值
10	BPB_TotSec16	DW 2880		; 逻辑扇区总数
11	BPB_Media	DB 0xF0		; 媒体描述符
12	BPB_FATSz16	DW 9		; 每FAT扇区数
13	BPB_SecPerTrk	DW 18		; 每磁道扇区数
14	BPB_NumHeads	DW 2		; 磁头数(面数)
15	BPB_HiddSec	DD 0		; 隐藏扇区数
16	BPB_TotSec32	DD 0		; 如果 wTotalSectorCount 是 0 由这个值记录扇区数
17
18	BS_DrvNum	DB 0		; 中断 13 的驱动器号
19	BS_Reserved1	DB 0		; 未使用
20	BS_BootSig	DB 29h		; 扩展引导标记 (29h)
21	BS_VolID	DD 0		; 卷序列号
22	BS_VolLab	DB 'OrangeS0.02'; 卷标, 必须 11 个字节
23	BS_FileSysType	DB 'FAT12   '	; 文件系统类型, 必须 8个字节  
24	;------------------------------------------------------------------------
25
26
27	; -------------------------------------------------------------------------
28	; 基于 FAT12 头的一些常量定义，如果头信息改变，下面的常量可能也要做相应改变
29	; -------------------------------------------------------------------------
30	; BPB_FATSz16
31	FATSz			equ	9
32
33	; 根目录占用空间:
34	; RootDirSectors = ((BPB_RootEntCnt*32)+(BPB_BytsPerSec–1))/BPB_BytsPerSec
35	; 但如果按照此公式代码过长，故定义此宏
36	RootDirSectors		equ	14
37
38	; Root Directory 的第一个扇区号	= BPB_RsvdSecCnt + (BPB_NumFATs * FATSz)
39	SectorNoOfRootDirectory	equ	19
40
41	; FAT1 的第一个扇区号	= BPB_RsvdSecCnt
42	SectorNoOfFAT1		equ	1
43
44	; DeltaSectorNo = BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) - 2
45	; 文件的开始Sector号 = DirEntry中的开始Sector号 + 根目录占用Sector数目
46	;                      + DeltaSectorNo
47	DeltaSectorNo		equ	17
48

```

在boot.asm 开头代码就是

```

; 下面是 FAT12 磁盘的头, 之所以包含它是因为下面用到了磁盘的一些信息
%include	"fat12hdr.inc"

```

修改loader.asm ，把他内核放进内存

```

1  	org  0100h
2  
3  	BaseOfStack		equ	0100h
4  
5  	BaseOfKernelFile	equ	 08000h	; KERNEL.BIN 被加载到的位置 ----  段地址
6  	OffsetOfKernelFile	equ	     0h	; KERNEL.BIN 被加载到的位置 ---- 偏移地址
7  
8  
9  		jmp	LABEL_START		; Start
10 
11 	; 下面是 FAT12 磁盘的头, 之所以包含它是因为下面用到了磁盘的一些信息
12 	%include	"fat12hdr.inc"
13 
14 
15 	LABEL_START:			; <--- 从这里开始 *************
16 		mov	ax, cs
17 		mov	ds, ax
18 		mov	es, ax
19 		mov	ss, ax
20 		mov	sp, BaseOfStack
21 
22 		mov	dh, 0			; "Loading  "
23 		call	DispStr			; 显示字符串
24 
25 		; 下面在 A 盘的根目录寻找 KERNEL.BIN
26 		mov	word [wSectorNo], SectorNoOfRootDirectory	
27 		xor	ah, ah	; `.
28 		xor	dl, dl	;  | 软驱复位
29 		int	13h	; /
30 	LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
31 		cmp	word [wRootDirSizeForLoop], 0	; `.
32 		jz	LABEL_NO_KERNELBIN		;  | 判断根目录区是不是已经读完,
33 		dec	word [wRootDirSizeForLoop]	; /  读完表示没有找到 KERNEL.BIN
34 		mov	ax, BaseOfKernelFile
35 		mov	es, ax			; es <- BaseOfKernelFile
36 		mov	bx, OffsetOfKernelFile	; bx <- OffsetOfKernelFile
37 		mov	ax, [wSectorNo]		; ax <- Root Directory 中的某 Sector 号
38 		mov	cl, 1
39 		call	ReadSector
40 
41 		mov	si, KernelFileName	; ds:si -> "KERNEL  BIN"
42 		mov	di, OffsetOfKernelFile
43 		cld
44 		mov	dx, 10h
45 	LABEL_SEARCH_FOR_KERNELBIN:
46 		cmp	dx, 0				  ; `.
47 		jz	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR;  | 循环次数控制, 如果已经读完
48 		dec	dx				  ; /  了一个 Sector, 就跳到下一个
49 		mov	cx, 11
50 	LABEL_CMP_FILENAME:
51 		cmp	cx, 0			; `.
52 		jz	LABEL_FILENAME_FOUND	;  | 循环次数控制, 如果比较了 11 个字符都
53 		dec	cx			; /  相等, 表示找到
54 		lodsb				; ds:si -> al
55 		cmp	al, byte [es:di]	; if al == es:di
56 		jz	LABEL_GO_ON
57 		jmp	LABEL_DIFFERENT
58 	LABEL_GO_ON:
59 		inc	di
60 		jmp	LABEL_CMP_FILENAME	;	继续循环
61 
62 	LABEL_DIFFERENT:
63 		and	di, 0FFE0h		; else`. 让 di 是 20h 的倍数
64 		add	di, 20h			;      |
65 		mov	si, KernelFileName	;      | di += 20h  下一个目录条目
66 		jmp	LABEL_SEARCH_FOR_KERNELBIN;   /
67 
68 	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
69 		add	word [wSectorNo], 1
70 		jmp	LABEL_SEARCH_IN_ROOT_DIR_BEGIN
71 
72 	LABEL_NO_KERNELBIN:
73 		mov	dh, 2			; "No KERNEL."
74 		call	DispStr			; 显示字符串
75 	%ifdef	_LOADER_DEBUG_
76 		mov	ax, 4c00h		; `.
77 		int	21h			; / 没有找到 KERNEL.BIN, 回到 DOS
78 	%else
79 		jmp	$			; 没有找到 KERNEL.BIN, 死循环在这里
80 	%endif
81 
82 	LABEL_FILENAME_FOUND:			; 找到 KERNEL.BIN 后便来到这里继续
83 		mov	ax, RootDirSectors
84 		and	di, 0FFF0h		; di -> 当前条目的开始
85 
86 		push	eax
87 		mov	eax, [es : di + 01Ch]		; `.
88 		mov	dword [dwKernelSize], eax	; / 保存 KERNEL.BIN 文件大小
89 		pop	eax
90 
91 		add	di, 01Ah		; di -> 首 Sector
92 		mov	cx, word [es:di]
93 		push	cx			; 保存此 Sector 在 FAT 中的序号
94 		add	cx, ax
95 		add	cx, DeltaSectorNo	; cl <- KERNEL.BIN 的起始扇区号(0-based)
96 		mov	ax, BaseOfKernelFile
97 		mov	es, ax			; es <- BaseOfKernelFile
98 		mov	bx, OffsetOfKernelFile	; bx <- OffsetOfKernelFile
99 		mov	ax, cx			; ax <- Sector 号
100
101	LABEL_GOON_LOADING_FILE:
102		push	ax			; `.
103		push	bx			;  |
104		mov	ah, 0Eh			;  | 每读一个扇区就在 "Loading  " 后面
105		mov	al, '.'			;  | 打一个点, 形成这样的效果:
106		mov	bl, 0Fh			;  | Loading ......
107		int	10h			;  |
108		pop	bx			;  |
109		pop	ax			; /
110
111		mov	cl, 1
112		call	ReadSector
113		pop	ax			; 取出此 Sector 在 FAT 中的序号
114		call	GetFATEntry
115		cmp	ax, 0FFFh
116		jz	LABEL_FILE_LOADED
117		push	ax			; 保存 Sector 在 FAT 中的序号
118		mov	dx, RootDirSectors
119		add	ax, dx
120		add	ax, DeltaSectorNo
121		add	bx, [BPB_BytsPerSec]
122		jmp	LABEL_GOON_LOADING_FILE
123	LABEL_FILE_LOADED:
124
125		call	KillMotor		; 关闭软驱马达
126
127		mov	dh, 1			; "Ready."
128		call	DispStr			; 显示字符串
129
130		jmp	$
131
132
133	;============================================================================
134	;变量
135	;----------------------------------------------------------------------------
136	wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数
137	wSectorNo		dw	0		; 要读取的扇区号
138	bOdd			db	0		; 奇数还是偶数
139	dwKernelSize		dd	0		; KERNEL.BIN 文件大小
140
141	;============================================================================
142	;字符串
143	;----------------------------------------------------------------------------
144	KernelFileName		db	"KERNEL  BIN", 0	; KERNEL.BIN 之文件名
145	; 为简化代码, 下面每个字符串的长度均为 MessageLength
146	MessageLength		equ	9
147	LoadMessage:		db	"Loading  "
148	Message1		db	"Ready.   "
149	Message2		db	"No KERNEL"
150	;============================================================================
151
152	;----------------------------------------------------------------------------
153	; 函数名: DispStr
154	;----------------------------------------------------------------------------
155	; 作用:
156	;	显示一个字符串, 函数开始时 dh 中应该是字符串序号(0-based)
157	DispStr:
158		mov	ax, MessageLength
159		mul	dh
160		add	ax, LoadMessage
161		mov	bp, ax			; ┓
162		mov	ax, ds			; ┣ ES:BP = 串地址
163		mov	es, ax			; ┛
164		mov	cx, MessageLength	; CX = 串长度
165		mov	ax, 01301h		; AH = 13,  AL = 01h
166		mov	bx, 0007h		; 页号为0(BH = 0) 黑底白字(BL = 07h)
167		mov	dl, 0
168		add	dh, 3			; 从第 3 行往下显示
169		int	10h			; int 10h
170		ret
171	;----------------------------------------------------------------------------
172	; 函数名: ReadSector
173	;----------------------------------------------------------------------------
174	; 作用:
175	;	从序号(Directory Entry 中的 Sector 号)为 ax 的的 Sector 开始, 将 cl 个 Sector 读入 es:bx 中
176	ReadSector:
177		; -----------------------------------------------------------------------
178		; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
179		; -----------------------------------------------------------------------
180		; 设扇区号为 x
181		;                           ┌ 柱面号 = y >> 1
182		;       x           ┌ 商 y ┤
183		; -------------- => ┤      └ 磁头号 = y & 1
184		;  每磁道扇区数     │
185		;                   └ 余 z => 起始扇区号 = z + 1
186		push	bp
187		mov	bp, sp
188		sub	esp, 2			; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]
189
190		mov	byte [bp-2], cl
191		push	bx			; 保存 bx
192		mov	bl, [BPB_SecPerTrk]	; bl: 除数
193		div	bl			; y 在 al 中, z 在 ah 中
194		inc	ah			; z ++
195		mov	cl, ah			; cl <- 起始扇区号
196		mov	dh, al			; dh <- y
197		shr	al, 1			; y >> 1 (其实是 y/BPB_NumHeads, 这里BPB_NumHeads=2)
198		mov	ch, al			; ch <- 柱面号
199		and	dh, 1			; dh & 1 = 磁头号
200		pop	bx			; 恢复 bx
201		; 至此, "柱面号, 起始扇区, 磁头号" 全部得到 ^^^^^^^^^^^^^^^^^^^^^^^^
202		mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
203	.GoOnReading:
204		mov	ah, 2			; 读
205		mov	al, byte [bp-2]		; 读 al 个扇区
206		int	13h
207		jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 这时就不停地读, 直到正确为止
208
209		add	esp, 2
210		pop	bp
211
212		ret
213
214	;----------------------------------------------------------------------------
215	; 函数名: GetFATEntry
216	;----------------------------------------------------------------------------
217	; 作用:
218	;	找到序号为 ax 的 Sector 在 FAT 中的条目, 结果放在 ax 中
219	;	需要注意的是, 中间需要读 FAT 的扇区到 es:bx 处, 所以函数一开始保存了 es 和 bx
220	GetFATEntry:
221		push	es
222		push	bx
223		push	ax
224		mov	ax, BaseOfKernelFile	; ┓
225		sub	ax, 0100h		; ┣ 在 BaseOfKernelFile 后面留出 4K 空间用于存放 FAT
226		mov	es, ax			; ┛
227		pop	ax
228		mov	byte [bOdd], 0
229		mov	bx, 3
230		mul	bx			; dx:ax = ax * 3
231		mov	bx, 2
232		div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
233		cmp	dx, 0
234		jz	LABEL_EVEN
235		mov	byte [bOdd], 1
236	LABEL_EVEN:;偶数
237		xor	dx, dx			; 现在 ax 中是 FATEntry 在 FAT 中的偏移量. 下面来计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
238		mov	bx, [BPB_BytsPerSec]
239		div	bx			; dx:ax / BPB_BytsPerSec  ==>	ax <- 商   (FATEntry 所在的扇区相对于 FAT 来说的扇区号)
240						;				dx <- 余数 (FATEntry 在扇区内的偏移)。
241		push	dx
242		mov	bx, 0			; bx <- 0	于是, es:bx = (BaseOfKernelFile - 100):00 = (BaseOfKernelFile - 100) * 10h
243		add	ax, SectorNoOfFAT1	; 此句执行之后的 ax 就是 FATEntry 所在的扇区号
244		mov	cl, 2
245		call	ReadSector		; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界发生错误, 因为一个 FATEntry 可能跨越两个扇区
246		pop	dx
247		add	bx, dx
248		mov	ax, [es:bx]
249		cmp	byte [bOdd], 1
250		jnz	LABEL_EVEN_2
251		shr	ax, 4
252	LABEL_EVEN_2:
253		and	ax, 0FFFh
254
255	LABEL_GET_FAT_ENRY_OK:
256
257		pop	bx
258		pop	es
259		ret
260	;----------------------------------------------------------------------------
261
262
263	;----------------------------------------------------------------------------
264	; 函数名: KillMotor
265	;----------------------------------------------------------------------------
266	; 作用:
267	;	关闭软驱马达
268	KillMotor:
269		push	dx
270		mov	dx, 03F2h
271		mov	al, 0
272		out	dx, al
273		pop	dx
274		ret
275	;----------------------------------------------------------------------------
276
277

```

可以看出Loader.asm 比上次增加了很多代码，但是和boot.asm 对比发现，是几乎一样的！
的确如书上所说，增加了call	KillMotor		; 关闭软驱马达


加载内核代码写好了，还缺个内核的东东！


kernel.asm 代码实现功能还是一个字符功能！

```

1 	; 编译链接方法
2 	; $ nasm -f elf kernel.asm -o kernel.o
3 	; $ ld -s kernel.o -o kernel.bin    #‘-s’选项意为“strip all”
4 
5 	[section .text]	; 代码在此
6 
7 	global _start	; 导出 _start
8 
9 	_start:	; 跳到这里来的时候，我们假设 gs 指向显存
10		mov	ah, 0Fh				; 0000: 黑底    1111: 白字
11		mov	al, 'K'
12		mov	[gs:((80 * 1 + 39) * 2)], ax	; 屏幕第 1 行, 第 39 列
13		jmp	$
14

```

运行结果如下:

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191120234515.png)


### 跳入保护模式

现在内核加载进去内存了，要跳入保护模式了

GDT 和对应的选择子，三个描述符，分别是0~4GB的可读写段和一个指向显存开始地址的段！

loader.asm

```

%include	"load.inc"
%include	"pm.inc"

; GDT
;                            段基址     段界限, 属性
LABEL_GDT:	    Descriptor 0,            0, 0              ; 空描述符
LABEL_DESC_FLAT_C:  Descriptor 0,      0fffffh, DA_CR|DA_32|DA_LIMIT_4K ;0-4G
LABEL_DESC_FLAT_RW: Descriptor 0,      0fffffh, DA_DRW|DA_32|DA_LIMIT_4K;0-4G
LABEL_DESC_VIDEO:   Descriptor 0B8000h, 0ffffh, DA_DRW|DA_DPL3 ; 显存首地址

GdtLen		equ	$ - LABEL_GDT
GdtPtr		dw	GdtLen - 1				; 段界限
		dd	BaseOfLoaderPhyAddr + LABEL_GDT		; 基地址

; GDT 选择子
SelectorFlatC		equ	LABEL_DESC_FLAT_C	- LABEL_GDT
SelectorFlatRW		equ	LABEL_DESC_FLAT_RW	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT + SA_RPL3



BaseOfStack	equ	0100h
PageDirBase	equ	100000h	; 页目录开始地址: 1M
PageTblBase	equ	101000h	; 页表开始地址:   1M + 4K


```


前面学习的时候，描述符的段基址都是运行时计算后填入的相应的位置。现在我们也不知道段地址，自然也不知道程序运行时在内存的位置。

Loader 是我们自己加载的，段地址被确定为BaseOfLoader，所以在Loader中出现的标号(变量)的物理地址可以用下面的公式来表示：

标号（变量） 的物理地址 =  BaseOfLoader X 10h + 标号（变量）的偏移。

BaseOfLoader就同时在boot.asm 和loader.asm两个文件中使用，我们也把它和相应的声明放在load.inc 文件中。

load.inc 宏定义

```

BaseOfLoader	    equ	 09000h	; LOADER.BIN 被加载到的位置 ----  段地址
OffsetOfLoader	    equ	  0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址

BaseOfLoaderPhyAddr equ	BaseOfLoader*10h ; LOADER.BIN 被加载到的位置 ---- 物理地址

BaseOfKernelFile    equ	 08000h	; KERNEL.BIN 被加载到的位置 ----  段地址
OffsetOfKernelFile  equ	     0h	; KERNEL.BIN 被加载到的位置 ---- 偏移地址

```

都用%include “load.inc”包含！


Loader.asm 32位代码段只打印一个字符 ‘P’。


![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191122002433.png)

进入保护模式的代码不附上了！


初始化其他寄存器，

```

	mov	ax, SelectorFlatRW
	mov	ds, ax
	mov	es, ax
	mov	fs, ax
	mov	ss, ax
	mov	esp, TopOfStack

```

然后打开分页机制，打开之前需要得到可用内容的情况！

```

	; 得到内存数
	mov	ebx, 0			; ebx = 后续值, 开始时需为 0
	mov	di, _MemChkBuf		; es:di 指向一个地址范围描述符结构(ARDS)
.MemChkLoop:
	mov	eax, 0E820h		; eax = 0000E820h
	mov	ecx, 20			; ecx = 地址范围描述符结构的大小
	mov	edx, 0534D4150h		; edx = 'SMAP'
	int	15h			; int 15h
	jc	.MemChkFail
	add	di, 20
	inc	dword [_dwMCRNumber]	; dwMCRNumber = ARDS 的个数
	cmp	ebx, 0
	jne	.MemChkLoop
	jmp	.MemChkOK
.MemChkFail:
	mov	dword [_dwMCRNumber], 0
.MemChkOK:

```

显示内存信息

```


; 显示内存信息 --------------------------------------------------------------
DispMemInfo:
	push	esi
	push	edi
	push	ecx

	mov	esi, MemChkBuf
	mov	ecx, [dwMCRNumber];for(int i=0;i<[MCRNumber];i++)//每次得到一个ARDS
.loop:				  ;{
	mov	edx, 5		  ;  for(int j=0;j<5;j++)//每次得到一个ARDS中的成员
	mov	edi, ARDStruct	  ;  {//依次显示:BaseAddrLow,BaseAddrHigh,LengthLow
.1:				  ;               LengthHigh,Type
	push	dword [esi]	  ;
	call	DispInt		  ;    DispInt(MemChkBuf[j*4]); // 显示一个成员
	pop	eax		  ;
	stosd			  ;    ARDStruct[j*4] = MemChkBuf[j*4];
	add	esi, 4		  ;
	dec	edx		  ;
	cmp	edx, 0		  ;
	jnz	.1		  ;  }
	call	DispReturn	  ;  printf("\n");
	cmp	dword [dwType], 1 ;  if(Type == AddressRangeMemory)
	jne	.2		  ;  {
	mov	eax, [dwBaseAddrLow];
	add	eax, [dwLengthLow];
	cmp	eax, [dwMemSize]  ;    if(BaseAddrLow + LengthLow > MemSize)
	jb	.2		  ;
	mov	[dwMemSize], eax  ;    MemSize = BaseAddrLow + LengthLow;
.2:				  ;  }
	loop	.loop		  ;}
				  ;
	call	DispReturn	  ;printf("\n");
	push	szRAMSize	  ;
	call	DispStr		  ;printf("RAM size:");
	add	esp, 4		  ;
				  ;
	push	dword [dwMemSize] ;
	call	DispInt		  ;DispInt(MemSize);
	add	esp, 4		  ;

	pop	ecx
	pop	edi
	pop	esi
	ret
; ---------------------------------------------------------------------------


```

后面就是启动分页函数的SetupPaging

```


; 启动分页机制 --------------------------------------------------------------
SetupPaging:
	; 根据内存大小计算应初始化多少PDE以及多少页表
	xor	edx, edx
	mov	eax, [dwMemSize]
	mov	ebx, 400000h	; 400000h = 4M = 4096 * 1024, 一个页表对应的内存大小
	div	ebx
	mov	ecx, eax	; 此时 ecx 为页表的个数，也即 PDE 应该的个数
	test	edx, edx
	jz	.no_remainder
	inc	ecx		; 如果余数不为 0 就需增加一个页表
.no_remainder:
	push	ecx		; 暂存页表个数

	; 为简化处理, 所有线性地址对应相等的物理地址. 并且不考虑内存空洞.

	; 首先初始化页目录
	mov	ax, SelectorFlatRW
	mov	es, ax
	mov	edi, PageDirBase	; 此段首地址为 PageDirBase
	xor	eax, eax
	mov	eax, PageTblBase | PG_P  | PG_USU | PG_RWW
.1:
	stosd
	add	eax, 4096		; 为了简化, 所有页表在内存中是连续的
	loop	.1

	; 再初始化所有页表
	pop	eax			; 页表个数
	mov	ebx, 1024		; 每个页表 1024 个 PTE
	mul	ebx
	mov	ecx, eax		; PTE个数 = 页表个数 * 1024
	mov	edi, PageTblBase	; 此段首地址为 PageTblBase
	xor	eax, eax
	mov	eax, PG_P  | PG_USU | PG_RWW
.2:
	stosd
	add	eax, 4096		; 每一页指向 4K 的空间
	loop	.2

	mov	eax, PageDirBase
	mov	cr3, eax
	mov	eax, cr0
	or	eax, 80000000h
	mov	cr0, eax
	jmp	short .3
.3:
	nop

	ret
; 分页机制启动完毕 ----------------------------------------------------------



```


代码使用的字符串和变量的定义

```

LABEL_DATA:
; 实模式下使用这些符号
; 字符串
_szMemChkTitle:	db "BaseAddrL BaseAddrH LengthLow LengthHigh   Type", 0Ah, 0
_szRAMSize:	db "RAM size:", 0
_szReturn:	db 0Ah, 0
;; 变量
_dwMCRNumber:	dd 0	; Memory Check Result
_dwDispPos:	dd (80 * 6 + 0) * 2	; 屏幕第 6 行, 第 0 列
_dwMemSize:	dd 0
_ARDStruct:	; Address Range Descriptor Structure
  _dwBaseAddrLow:		dd	0
  _dwBaseAddrHigh:		dd	0
  _dwLengthLow:			dd	0
  _dwLengthHigh:		dd	0
  _dwType:			dd	0
_MemChkBuf:	times	256	db	0
;
;; 保护模式下使用这些符号
szMemChkTitle		equ	BaseOfLoaderPhyAddr + _szMemChkTitle
szRAMSize		equ	BaseOfLoaderPhyAddr + _szRAMSize
szReturn		equ	BaseOfLoaderPhyAddr + _szReturn
dwDispPos		equ	BaseOfLoaderPhyAddr + _dwDispPos
dwMemSize		equ	BaseOfLoaderPhyAddr + _dwMemSize
dwMCRNumber		equ	BaseOfLoaderPhyAddr + _dwMCRNumber
ARDStruct		equ	BaseOfLoaderPhyAddr + _ARDStruct
	dwBaseAddrLow	equ	BaseOfLoaderPhyAddr + _dwBaseAddrLow
	dwBaseAddrHigh	equ	BaseOfLoaderPhyAddr + _dwBaseAddrHigh
	dwLengthLow	equ	BaseOfLoaderPhyAddr + _dwLengthLow
	dwLengthHigh	equ	BaseOfLoaderPhyAddr + _dwLengthHigh
	dwType		equ	BaseOfLoaderPhyAddr + _dwType
MemChkBuf		equ	BaseOfLoaderPhyAddr + _MemChkBuf


```

可以看出使用的地址都加上了Loader的基地址了！

调用显示内存信息和启动分页的函数

```

	push	szMemChkTitle
	call	DispStr
	add	esp, 4

	call	DispMemInfo
	call	SetupPaging

```


### 重新放置内核

我们要根据内核的Program header table 的信息类似c语言进行内存复制

memcpy(p_vaddr, BaseOfLoaderPhyAddr + p_offset , p_filesz);

有多少个Program header，就复制多少次！

前面的提到的elf 中 program header都描述一个段，p_offset为段在文件中的偏移，p_filesz为段在文件中的长度，p_vaddr为段在内存中的虚拟地址。

书上提到了，比如ld生成的可执行文件p_vaddr的值总是一个类似于0x8048xxx的值，至少例子是这样的一个值。可是我们启动分页机制时地址都是对等映射的，内存地址0x8048xxx已经处于128Mb内存意外的（128Mb的十六进制表示0x8000000），如果计算机的内存小于128Mb的话，这个地址显然超出内存大小！  即使内存足够大，也不能让编译器决定内核加载到什么地方！有两个解决办法，一个是通过修改页表让0x80048xxx映射到较低的地址，另外一种方法就是通过修改ld的选项让他生成的可执行代码中p_vaddr的值变小。

显然第二种办法简单易行，把编译链接的命令行改为：

```

nasm -f elf -o kernel.o kernel.asm

ld -s -Ttext 0x30400 -o kernel.bin kernel.o

```

程序的入口地址就变成0x30400了，elf header等信息会位于0x30400之前！
此时的elf header 和 program header table情况如表所示

项目 |  值  |   说明 |
-|-|-

e_ident | .... |    |

e_type  |  2H  |  可执行文件  |

e_machine |  3H  |  80386   |

e_version  |  1H   |        |

e_entry    |  30400H  |  入口地址   |


e_phoff    |  34H   |   Program header table在文件中的偏移量 |

e_shoff    |  448H  |  Section header table 在文件中的偏移量 |

e_flags  | 0H  |     |

e_ehsizes | 34H |  ELF header 大小  |

e_phentsize  |  28H  |  每一个Section header 大小28H字节 |

e_shnum   |  4H    |   Section header table 中有四个条目 |

e_shstrndx  |  3H  |  包含节名称的字符串表示第三个节 |


Program header

项目 |  值  |   说明 |
-|-|-

p_type  |  1H  |  PT_LOAD  |

p_offset |  0H  |  段的第一个字节在文件中的偏移   |

p_vaddr  |  30000H   | 段的第一个字节在内存中虚拟地址       |

p_paddr    |  30000H  |   |


p_filesz    |  40DH   |  段在文件中的长度 |

p_memsz    |  40DH  | 段在内存中的长度 |

e_flags  | 5H  |     |

e_ehsizes | 1000H |    |



也就说，我们应该把文件从开头开始40d字节的内容放到内存30000h处。由于程序的入口在30400h处，所以，从这里看出实际代码只有0Dh + 1 个字节。 如图是操作的结果！

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191123195434.png)

星号省去的部分都是0.

从中可以看出，从400h到40dh 是仅有的代码，看代码 EB FE 就是代码的最后的"jmp $" （EB, FE 相当于jmp $死循环）！


下面需要将kernel.bin 根据ELf 文件信息转移到正确的位置！就是找出每个Program header，根据信息进行内存复制！

InitKernel :

```


	call	InitKernel

....


; InitKernel ---------------------------------------------------------------------------------
; 将 KERNEL.BIN 的内容经过整理对齐后放到新的位置
; 遍历每一个 Program Header，根据 Program Header 中的信息来确定把什么放进内存，放到什么位置，以及放多少。
; --------------------------------------------------------------------------------------------
InitKernel:
        xor   esi, esi
        mov   cx, word [BaseOfKernelFilePhyAddr+2Ch];`. ecx <- pELFHdr->e_phnum
        movzx ecx, cx                               ;/
        mov   esi, [BaseOfKernelFilePhyAddr + 1Ch]  ; esi <- pELFHdr->e_phoff
        add   esi, BaseOfKernelFilePhyAddr;esi<-OffsetOfKernel+pELFHdr->e_phoff
.Begin:
        mov   eax, [esi + 0]
        cmp   eax, 0                      ; PT_NULL
        jz    .NoAction
        push  dword [esi + 010h]    ;size ;`.
        mov   eax, [esi + 04h]            ; |
        add   eax, BaseOfKernelFilePhyAddr; | memcpy((void*)(pPHdr->p_vaddr),
        push  eax		    ;src  ; |      uchCode + pPHdr->p_offset,
        push  dword [esi + 08h]     ;dst  ; |      pPHdr->p_filesz;
        call  MemCpy                      ; |
        add   esp, 12                     ;/
.NoAction:
        add   esi, 020h                   ; esi += pELFHdr->e_phentsize
        dec   ecx
        jnz   .Begin

        ret
; InitKernel ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


```


现在就向内核转移了，为什么入口就是0x30400而不是其他的呢？？

应该不是随便指定的数据，的确不是随便指定的数据，前面章节我们存放Loader.bin 和Kernel.bin 位置也不是随便指定的数据！ 看一下内核加载之后内存的使用情况，你可能就明白了！

如图所示就是内存使用的分布图示：

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191123232418.png)

从这我们可以看出，我们一直用来显示的以0xB8000为开始的内存，就不能被os用在常规用途，在比如0x400 - 0x4FF 这段内存，里面存放了许多参数，也不能覆盖！

但观看到9FC00h这个数字的时候，通过终端15h得到的内存信息已经明确告诉我们，09FC00h ~ 09FFFFh 这段内存不能用作常规使用！即便0h ~ 09FBFFh 可以被使用，仍然把BIOS 参数区保护起来备用！

所以我们真正可以使用的内促是0500 ~ 09FBFFh这一段。那么，为什么指定的入口地址是0x30400离0x500还那么远呢？ 其实，之所市这么做是为了调试方便，因为dos都不占用0x30000以上的内存地址，把内核加载到这里，即便在Dos下调试也不会覆盖掉Dos内存。

所以，从0x90000开始的63kb留给了Loader.bin ，0x80000开始的64kb留给了Kernel.bin，0x30000开始的320kb留给了整理后的内核，而页目录和页表被放置在了1MB以上的内存空间。

我们为Loader.bin 留了63kb的空间！ 
一方面，因为它本质上是个.COM文件，另外一方面我们在写boot.asm 把文件加载了同一个段，文件再大也是不允许的，而且，一个Loader也不会有那么大，所以63kb足够使用了！

加载文件Kernel.bin 到内存的使用方法跟加载Loader.bin 是一样的！也是放在一个段中，所以他也不能超过64kb。不过，暂时来讲，我们的内核还没有那么大，可以作为权宜之计！

### 向内核交出控制权

下面向内核跳转！

```

	;***************************************************************
	jmp	SelectorFlatC:KernelEntryPointPhyAddr	; 正式进入内核 *
	;***************************************************************

```
KernelEntryPointPhyAddr的值是0x30400.  跟ld参数 -Ttext指定的值是一致的！若以后更改内核的位置，就改两个地方就行了！


![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191123234316.png)

看到一个K 就说明内核已经在执行了！


![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191123234316.png)

如图所示，cs ds es fs ss 表示的段统统指向内存地址0h， gs表示的段指向显存，这个是进入保护模式设置的！esp GDT等内容也在Loader中，下面对内核扩充的时候，都会挪到内核中，方便控制！


### 切换堆栈和GDT

现在我们可以使用c语言了！

代码如下：

```


; ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
;                               kernel.asm
; ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
;                                                     Forrest Yu, 2005
; ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

; ----------------------------------------------------------------------
; 编译连接方法:
; $ rm -f kernel.bin
; $ nasm -f elf -o kernel.o kernel.asm
; $ nasm -f elf -o string.o string.asm
; $ nasm -f elf -o klib.o klib.asm
; $ gcc -c -o start.o start.c
; $ ld -s -Ttext 0x30400 -o kernel.bin kernel.o string.o start.o klib.o
; $ rm -f kernel.o string.o start.o
; $ 
; ----------------------------------------------------------------------

SELECTOR_KERNEL_CS	equ	8

; 导入函数
extern	cstart

; 导入全局变量
extern	gdt_ptr

[SECTION .bss]
StackSpace		resb	2 * 1024
StackTop:		; 栈顶

[section .text]	; 代码在此

global _start	; 导出 _start

_start:
	; 此时内存看上去是这样的（更详细的内存情况在 LOADER.ASM 中有说明）：
	;              ┃                                    ┃
	;              ┃                 ...                ┃
	;              ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃■■■■■■Page  Tables■■■■■■┃
	;              ┃■■■■■(大小由LOADER决定)■■■■┃ PageTblBase
	;    00101000h ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃■■■■Page Directory Table■■■■┃ PageDirBase = 1M
	;    00100000h ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃□□□□ Hardware  Reserved □□□□┃ B8000h ← gs
	;       9FC00h ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃■■■■■■■LOADER.BIN■■■■■■┃ somewhere in LOADER ← esp
	;       90000h ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃■■■■■■■KERNEL.BIN■■■■■■┃
	;       80000h ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃■■■■■■■■KERNEL■■■■■■■┃ 30400h ← KERNEL 入口 (KernelEntryPointPhyAddr)
	;       30000h ┣━━━━━━━━━━━━━━━━━━┫
	;              ┋                 ...                ┋
	;              ┋                                    ┋
	;           0h ┗━━━━━━━━━━━━━━━━━━┛ ← cs, ds, es, fs, ss
	;
	;
	; GDT 以及相应的描述符是这样的：
	;
	;		              Descriptors               Selectors
	;              ┏━━━━━━━━━━━━━━━━━━┓
	;              ┃         Dummy Descriptor           ┃
	;              ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃         DESC_FLAT_C    (0～4G)     ┃   8h = cs
	;              ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃         DESC_FLAT_RW   (0～4G)     ┃  10h = ds, es, fs, ss
	;              ┣━━━━━━━━━━━━━━━━━━┫
	;              ┃         DESC_VIDEO                 ┃  1Bh = gs
	;              ┗━━━━━━━━━━━━━━━━━━┛
	;
	; 注意! 在使用 C 代码的时候一定要保证 ds, es, ss 这几个段寄存器的值是一样的
	; 因为编译器有可能编译出使用它们的代码, 而编译器默认它们是一样的. 比如串拷贝操作会用到 ds 和 es.
	;
	;


	; 把 esp 从 LOADER 挪到 KERNEL
	mov	esp, StackTop	; 堆栈在 bss 段中

	sgdt	[gdt_ptr]	; cstart() 中将会用到 gdt_ptr
	call	cstart		; 在此函数中改变了gdt_ptr，让它指向新的GDT
	lgdt	[gdt_ptr]	; 使用新的GDT

	;lidt	[idt_ptr]

	jmp	SELECTOR_KERNEL_CS:csinit
csinit:		; “这个跳转指令强制使用刚刚初始化的结构”——<<OS:D&I 2nd>> P90.

	push	0
	popfd	; Pop top of stack into EFLAGS

	hlt

```

代码用简单的4个语句就完成切换堆栈和更换GDT 的任务， 其中，stacktop定义在.bss段中，堆栈大小为2kb。操作GDT用到的gdt_ptr和cstart分别是一个全局变量和全局函数，他们定义在start.c中！


start.c

```


/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
                            start.c
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
                                                    Forrest Yu, 2005
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/

#include "type.h"
#include "const.h"
#include "protect.h"

PUBLIC	void*	memcpy(void* pDst, void* pSrc, int iSize);

PUBLIC	void	disp_str(char * pszInfo);

PUBLIC	u8		gdt_ptr[6];	/* 0~15:Limit  16~47:Base */
PUBLIC	DESCRIPTOR	gdt[GDT_SIZE];

PUBLIC void cstart()
{

	disp_str("\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n"
		 "-----\"cstart\" begins-----\n");

	/* 将 LOADER 中的 GDT 复制到新的 GDT 中 */
	memcpy(&gdt,				   /* New GDT */
	       (void*)(*((u32*)(&gdt_ptr[2]))),    /* Base  of Old GDT */
	       *((u16*)(&gdt_ptr[0])) + 1	   /* Limit of Old GDT */
		);
	/* gdt_ptr[6] 共 6 个字节：0~15:Limit  16~47:Base。用作 sgdt/lgdt 的参数。*/
	u16* p_gdt_limit = (u16*)(&gdt_ptr[0]);
	u32* p_gdt_base  = (u32*)(&gdt_ptr[2]);
	*p_gdt_limit = GDT_SIZE * sizeof(DESCRIPTOR) - 1;
	*p_gdt_base  = (u32)&gdt;
}


```

ctart() 首先把位于Loader中的原GDT全部复制给新的GDT中，然后把gdt_ptr中的内容换成新的GDT的基地址和界限。复制GDT使用的函数是memcpy。该函数体放在string.asm!

```


; ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
;                              string.asm
; ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
;                                                       Forrest Yu, 2005
; ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

[SECTION .text]

; 导出函数
global	memcpy


; ------------------------------------------------------------------------
; void* memcpy(void* es:pDest, void* ds:pSrc, int iSize);
; ------------------------------------------------------------------------
memcpy:
	push	ebp
	mov	ebp, esp

	push	esi
	push	edi
	push	ecx

	mov	edi, [ebp + 8]	; Destination
	mov	esi, [ebp + 12]	; Source
	mov	ecx, [ebp + 16]	; Counter
.1:
	cmp	ecx, 0		; 判断计数器
	jz	.2		; 计数器为零时跳出

	mov	al, [ds:esi]		; ┓
	inc	esi			; ┃
					; ┣ 逐字节移动
	mov	byte [es:edi], al	; ┃
	inc	edi			; ┛

	dec	ecx		; 计数器减一
	jmp	.1		; 循环
.2:
	mov	eax, [ebp + 8]	; 返回值

	pop	ecx
	pop	edi
	pop	esi
	mov	esp, ebp
	pop	ebp

	ret			; 函数结束，返回
; memcpy 结束-------------------------------------------------------------

```

Loader.asm中显示P 和 K 都删除了，新增加了Kliba.asm -- 第三章的内容！

start.c 里面已经增加了

```

disp_str("\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n"
		 "-----\"cstart\" begins-----\n");

```

makefile

```

##################################################
# Makefile
##################################################

BOOT:=boot.asm
LDR:=loader.asm
KERNEL:=kernel.asm
BOOT_BIN:=$(subst .asm,.bin,$(BOOT))
LDR_BIN:=$(subst .asm,.bin,$(LDR))
KERNEL_BIN:=$(subst .asm,.bin,$(KERNEL))

IMG:=a.img
FLOPPY:=/mnt/floppy/

.PHONY : everything

everything : $(BOOT_BIN) $(LDR_BIN) $(KERNEL_BIN)
	ld -m elf_i386 -s -Ttext 0x30400 -o kernel.bin kernel.o string.o start.o kliba.o
	dd if=$(BOOT_BIN) of=$(IMG) bs=512 count=1 conv=notrunc
	sudo mount -o loop $(IMG) $(FLOPPY)
	sudo cp $(LDR_BIN) $(FLOPPY) -v
	sudo cp $(KERNEL_BIN) $(FLOPPY) -v
	sudo umount $(FLOPPY)

clean :
	rm -f $(BOOT_BIN) $(LDR_BIN) $(KERNEL_BIN) *.o

$(BOOT_BIN) : $(BOOT)
	nasm $< -o $@

$(LDR_BIN) : $(LDR)
	nasm $< -o $@

$(KERNEL_BIN) : $(KERNEL) start.c string.asm
	nasm -f elf -o $(subst .asm,.o,$(KERNEL)) $<
	nasm -f elf -o string.o string.asm
	nasm -f elf -o kliba.o kliba.asm
	gcc -m32 -c -fno-builtin -o start.o start.c



```

编译启动后界面如下：

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191124002524.png)


### 整理文件夹

- boot.asm 和 Loader.asm放在单独的目录/boot中，包括头文件！
- klib.asm和string.asm放在/lib中，作为库的形象出现！
- kernel.asm和start.c放在/kernel里面。

结构就清晰了！

tree

.
|-- a.img
|-- bochsrc
|-- boot
|   |--boot.asm
|   |--include
|   |  |--fat12hdr.inc
|   |  |--load.inc
|   |  `--pm.inc
|   `-- loader.asm
|-- include
|   |-- const.h
|   |-- protect.h
|   `-- type.h
|-- kernel
|  |-- kernel.asm
|  `--start.c
`-- lib
   |-- klib.asm
   `--string.asm

### makefile

makefile 网上讲的很详细，当然书上也有将的比较细致！



就是文件多的话，make需要使用参数-f 指定使用makefile.boot，而不是默认使得makefile或者GNUmakefile！

gcc也增加了制定头文件目录 “-I include”。


```

#########################
# Makefile for Orange'S #
#########################

# Entry point of Orange'S
# It must have the same value with 'KernelEntryPointPhyAddr' in load.inc!
ENTRYPOINT	= 0x30400

# Offset of entry point in kernel file
# It depends on ENTRYPOINT
ENTRYOFFSET	=   0x400

# Programs, flags, etc.
ASM		= nasm
DASM		= ndisasm
CC		= gcc
LD		= ld
ASMBFLAGS	= -I boot/include/
ASMKFLAGS	= -I include/ -f elf
CFLAGS		= -I include/ -m32 -c -fno-builtin
LDFLAGS		=  -s -Ttext $(ENTRYPOINT)
DASMFLAGS	= -u -o $(ENTRYPOINT) -e $(ENTRYOFFSET)

# This Program
ORANGESBOOT	= boot/boot.bin boot/loader.bin
ORANGESKERNEL	= kernel.bin
OBJS		= kernel/kernel.o kernel/start.o lib/kliba.o lib/string.o
DASMOUTPUT	= kernel.bin.asm

# All Phony Targets
.PHONY : everything final image clean realclean disasm all buildimg

# Default starting position
everything : $(ORANGESBOOT) $(ORANGESKERNEL)

all : realclean everything

final : all clean

image : final buildimg

clean :
	rm -f $(OBJS)

realclean :
	rm -f $(OBJS) $(ORANGESBOOT) $(ORANGESKERNEL)

disasm :
	$(DASM) $(DASMFLAGS) $(ORANGESKERNEL) > $(DASMOUTPUT)

# We assume that "a.img" exists in current folder
buildimg :
	dd if=boot/boot.bin of=a.img bs=512 count=1 conv=notrunc
	sudo mount -o loop a.img /mnt/floppy/
	sudo cp -fv boot/loader.bin /mnt/floppy/
	sudo cp -fv kernel.bin /mnt/floppy
	sudo umount /mnt/floppy

boot/boot.bin : boot/boot.asm boot/include/load.inc boot/include/fat12hdr.inc
	$(ASM) $(ASMBFLAGS) -o $@ $<

boot/loader.bin : boot/loader.asm boot/include/load.inc \
			boot/include/fat12hdr.inc boot/include/pm.inc
	$(ASM) $(ASMBFLAGS) -o $@ $<

$(ORANGESKERNEL) : $(OBJS)
	$(LD) -m elf_i386  $(LDFLAGS) -o $(ORANGESKERNEL) $(OBJS)

kernel/kernel.o : kernel/kernel.asm
	$(ASM) $(ASMKFLAGS) -o $@ $<

kernel/start.o : kernel/start.c include/type.h include/const.h include/protect.h
	$(CC)  $(CFLAGS) -o $@ $<

lib/kliba.o : lib/kliba.asm
	$(ASM) $(ASMKFLAGS) -o $@ $<

lib/string.o : lib/string.asm
	$(ASM) $(ASMKFLAGS) -o $@ $<


```

` sudo make image `

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191124002524.png)


### 添加中断处理

如果要实现进程，需要一种控制权转换机制，就是中断！

之前提到了8259A和建立IDT 中断！

设置一个函数设置8259A

这些基本都是c语言代码了

```

init_8259A()
{


	/* Master 8259, ICW1. */
	out_byte(INT_M_CTL,	0x11);
...

}

...

```

详细看源码，这个c语言基础有啥好说的，自己直接看！


这个c语言跟前面的汇编代码是一样的效果！

其中调用的 out_byte 是汇编语言函数！ 还有in_byte

因为端口操作可能需要时间，多加了nop延时！


```

; ========================================================================
;                  void out_byte(u16 port, u8 value);
; ========================================================================
out_byte:
	mov	edx, [esp + 4]		; port
	mov	al, [esp + 4 + 4]	; value
	out	dx, al
	nop	; 一点延迟
	nop
	ret


; ========================================================================
;                  u8 in_byte(u16 port);
; ========================================================================
in_byte:
	mov	edx, [esp + 4]		; port
	xor	eax, eax
	in	al, dx
	nop	; 一点延迟
	nop
	ret

```

因为函数越来越多，增加了好几个头文件，比如proto.h 用来存放函数声明，可以看到start.c 中 函数 disp_str的声明也放在里面了。memcpy 函数也放在一个头文件里面了，这个头文件是新建立的，取名为string.h!

自然也要修改makefile，有个问题就是头文件越来越多，gcc提供了一个参数 “-M” 可以自动生成依赖关系

` gcc -M kernel/start.c -I include `


下一步初始化IDT  在start.c中！

EXTERN  定义在const.h中，在global.h 中，如果GLOBAL_VARIABLES_HERE被定义的话，EXTERN会被定义为空值！

start.c 修改后，在kernel添加两句，导入idt_ptr这个符号并加载IDT!

中断或异常发生时 eflags、cs、eip已经被压栈，如果有错误码的话，也会被压栈！
所以对异常处理的总体思想就是，如果有错误码，则直接把向量号压栈，然后执行一个函数exception_handler，如果没有错误码，则先在栈中压入一个0xFFFFFFFF,在把向量号压栈并随后执行exception_handler。

由于C调用约定是调用者恢复堆栈！不用担心excepttion_handler会破坏堆栈中的eip、cs以及eflags！

为了突出显示，excepttion_handler函数使用disp_color_str()，多一个增加设置颜色的函数！！

disp_int是将整数转换成字符串显示出来！

代码自己详细看！

有了异常处理函数，就设置IDT，把IDT代码放进init_port(),也位于protect.c中。

protect.c 只有调用一个函数，init_idt_desc()，它用来初始化一个门描述符。其中用到的函数指针类型是这样定义的

typedef void （*int_handler） ();

所有的一场处理函数都必须与此声明完全一致！

在init_port()中，所有的描述符都被初始化中断门。函数总用到了若干个宏，INT_VECTOR_开头的宏表示中断向量，DA_386IGATE表示中断门，在protect.h中定义，PROVILEGE_KRNL和PRIVILEGE_USER定义在const.h中！


init_8259A（）语句也放在这个函数中！


详细请看代码！


start.c

调用inti_port（）  修改makefile



ud2 指令用来产生一个#UD异常！ 

可以在kernel.asm 添加一条指令

```
 csinit:
          ud2 

```


初始化8259A和设置IDT 这两项任务，目的是为了有异常处理机制！


8259A中断例程可以看书上代码！


所有的中断都会触发一个函数spurious_irq() 这个函数定义如下


```

/*======================================================================*
                           spurious_irq
 *======================================================================*/
PUBLIC void spurious_irq(int irq)
{
        disp_str("spurious_irq: ");
        disp_int(irq);
        disp_str("\n");
}

```

看的出来就是打印IRQ号而已！

设置IDT,还是在protect.c中！

后面我们要设置IF位，所以要修改一下，打开键盘中断！

```

	/* Master 8259, OCW1.  */
	out_byte(INT_M_CTLMASK,	0xFD);

	/* Slave  8259, OCW1.  */
	out_byte(INT_S_CTLMASK,	0xFF);

```
我们写入了FD  -  1111 1101，于是键盘中断被打开，其他中断仍然处于屏蔽状态。最后在kernel.asm中添加sti指令设置IF 位！

csint：

    sti
    hlt



make，运行，当我们敲击键盘就会出现spurious_irq:0x1

表面IRQ号就是1，对应就是键盘中断！


### 字符串输出函数disp_str有bug会导致异常

本人使用代码第5章节f节调试代码，用随书的代码发现在cstart.c 中调用同一个disp_str 函数中，第一个显示正常，第二个显示乱码，本能反应应该是堆栈没有保护，看了源代码，并且对照第三章节的代码，发现代码对寄存器esi ax bi bx都进行了操作，但是只保存ebp进程寄存器，所以，增加了 
```

    push    esi
    push    edi
    push    eax
    push    ebx

...

    pop ebx
    pop eax
    pop edi
    pop esi

```

经过调试，发现不显示了！！！！这太奇怪了，感觉这个数很古老，尝试网上查找资料，发现是在这段代码不一致：

`  mov esi, [ebp + 24] ; pszInfo `

而原书的代码是 

`  mov esi, [ebp + 8] ; pszInfo `

感谢作者！<https://blog.csdn.net/w1300048671/article/details/79700831>

附上代码

```

; ========================================================================
;          void disp_str(char * info);
; ========================================================================
disp_str:
    push    ebp
    push    esi
    push    edi
    push    eax
    push    ebx
    mov ebp, esp

    mov esi, [ebp + 24] ; pszInfo call压栈占用了4个字节，压入5个寄存器，一个寄存器32位，4字节也就是共20字节 20+4 =24
    mov edi, [disp_pos]
    mov ah, 0Fh
.1:
    lodsb
    test    al, al
    jz  .2
    cmp al, 0Ah ; 是回车吗?
    jnz .3
    push    eax
    mov eax, edi
    mov bl, 160
    div bl
    and eax, 0FFh
    inc eax
    mov bl, 160
    mul bl
    mov edi, eax
    pop eax
    jmp .1
.3:
    mov [gs:edi], ax
    add edi, 2
    jmp .1

.2:
    mov [disp_pos], edi
    pop ebx
    pop eax
    pop edi
    pop esi
    pop ebp
    ret 

```


```

; ========================================================================
;                  void disp_color_str(char * info, int color);
; ========================================================================
disp_color_str:
	push	ebp
    push    esi
    push    edi
    push    eax
    push    ebx
	mov	ebp, esp

	mov	esi, [ebp + 24]	; pszInfo
	mov	edi, [disp_pos]
	mov	ah, [ebp + 12]	; color 因为两个形参，一个形参也压入堆栈
.1:
	lodsb
	test	al, al
	jz	.2
	cmp	al, 0Ah	; 是回车吗?
	jnz	.3
	push	eax
	mov	eax, edi
	mov	bl, 160
	div	bl
	and	eax, 0FFh
	inc	eax
	mov	bl, 160
	mul	bl
	mov	edi, eax
	pop	eax
	jmp	.1
.3:
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1

.2:
	mov	[disp_pos], edi

	pop	ebp
	ret

```