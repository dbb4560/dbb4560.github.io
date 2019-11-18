---
layout:     post
title:      Orange'S：一个操作系统的实现
subtitle:   Dos可以识别的引导盘
date:       2019-11-02
header-img: img/post-first-blog-web.jpg
catalog: true
tags:
    - x86
    - 操作系统
    - 保护模式
    - dos
    - 引导盘

---

### boot.asm

引导扇区需要有BPB 等头信息才能被识别，需要加上去！

代码如下：

```

1 
2 	;%define	_BOOT_DEBUG_	; 做 Boot Sector 时一定将此行注释掉!将此行打开后用 nasm Boot.asm -o Boot.com 做成一个.COM文件易于调试
3 
4 	%ifdef	_BOOT_DEBUG_
5 		org  0100h			; 调试状态, 做成 .COM 文件, 可调试
6 	%else
7 		org  07c00h			; Boot 状态, Bios 将把 Boot Sector 加载到 0:7C00 处并开始执行
8 	%endif
9 
10		jmp short LABEL_START		; Start to boot.
11		nop				; 这个 nop 不可少
12
13		; 下面是 FAT12 磁盘的头
14		BS_OEMName	DB 'ForrestY'	; OEM String, 必须 8 个字节
15		BPB_BytsPerSec	DW 512		; 每扇区字节数
16		BPB_SecPerClus	DB 1		; 每簇多少扇区
17		BPB_RsvdSecCnt	DW 1		; Boot 记录占用多少扇区
18		BPB_NumFATs	DB 2		; 共有多少 FAT 表
19		BPB_RootEntCnt	DW 224		; 根目录文件数最大值
20		BPB_TotSec16	DW 2880		; 逻辑扇区总数
21		BPB_Media	DB 0xF0		; 媒体描述符
22		BPB_FATSz16	DW 9		; 每FAT扇区数
23		BPB_SecPerTrk	DW 18		; 每磁道扇区数
24		BPB_NumHeads	DW 2		; 磁头数(面数)
25		BPB_HiddSec	DD 0		; 隐藏扇区数
26		BPB_TotSec32	DD 0		; wTotalSectorCount为0时这个值记录扇区数
27		BS_DrvNum	DB 0		; 中断 13 的驱动器号
28		BS_Reserved1	DB 0		; 未使用
29		BS_BootSig	DB 29h		; 扩展引导标记 (29h)
30		BS_VolID	DD 0		; 卷序列号
31		BS_VolLab	DB 'OrangeS0.02'; 卷标, 必须 11 个字节
32		BS_FileSysType	DB 'FAT12   '	; 文件系统类型, 必须 8个字节  
33
34	LABEL_START:
35		mov	ax, cs
36		mov	ds, ax
37		mov	es, ax
38		Call	DispStr			; 调用显示字符串例程
39		jmp	$			; 无限循环
40	DispStr:
41		mov	ax, BootMessage
42		mov	bp, ax			; ES:BP = 串地址
43		mov	cx, 16			; CX = 串长度
44		mov	ax, 01301h		; AH = 13,  AL = 01h
45		mov	bx, 000ch		; 页号为0(BH = 0) 黑底红字(BL = 0Ch,高亮)
46		mov	dl, 0
47		int	10h			; int 10h
48		ret
49	BootMessage:		db	"Hello, OS world!"
50	times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
51	dw 	0xaa55				; 结束标志
52

```

可以看出代码只有52行。

第10行开始到第34行是BPB的区间！

编译生成的bin写入磁盘，运行就显示 hello OS world。


### 一个最简单的loader

新建一个loader.asm，代码如下所示：

```

1 	org	0100h
2 
3 		mov	ax, 0B800h
4 		mov	gs, ax
5 		mov	ah, 0Fh				; 0000: 黑底    1111: 白字
6 		mov	al, 'L'
7 		mov	[gs:((80 * 0 + 39) * 2)], ax	; 屏幕第 0 行, 第 39 列
8 
9 		jmp	$				; 到此停住
10

```

这个代码是编译成com文件，直接在dos， 显示界面是屏幕第一行显示L然后进入死循环！

`nasm loader.asm -o loader.bin`

### 加载loader 入内存

要加载一个文件入内存，需要读取硬盘，要用到BIOS中断int13h
用法如图所示：

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191117085701.png)

中断需要的参数不是原来提到的从第0扇区开始的扇区号，而是柱面号、磁头号以及当前柱面上的扇区号3个分量，所以需要我们自己来转换一下。对于1。44MB的软件来说，总共两面（磁头号0和1），每个面80个磁道（磁道号0~79），每个磁道有18个扇区（扇区号1-18）。下面的公司就是软盘容量的由来：

 2 X 80 X 18 X 512 = 1.44MB

于是，磁头号、柱面（磁道）号和起始扇区号可以如图所示计算

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191117123616.png)

读取软盘扇区的代码：

```

155	;----------------------------------------------------------------------------
156	; 函数名: ReadSector
157	;----------------------------------------------------------------------------
158	; 作用:
159	;	从第 ax 个 Sector 开始, 将 cl 个 Sector 读入 es:bx 中
160	ReadSector:
161		; -----------------------------------------------------------------------
162		; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
163		; -----------------------------------------------------------------------
164		; 设扇区号为 x
165		;                           ┌ 柱面号 = y >> 1
166		;       x           ┌ 商 y ┤
167		; -------------- => ┤      └ 磁头号 = y & 1
168		;  每磁道扇区数     │
169		;                   └ 余 z => 起始扇区号 = z + 1
170		push	bp  ;会用到bp，所以要先push bp
171		mov	bp, sp
172		sub	esp, 2 ; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]
173
174		mov	byte [bp-2], cl  ；把要读的扇区数放在byte[bp-2]的堆栈空间
175		push	bx			; 保存 bx 因为要用到bl作为除数，所以要先push保护起来
176		mov	bl, [BPB_SecPerTrk]	; bl: 除数 18
177		div	bl			; y 在 al 中, z 在 ah 中 ax/bl ,商在al中，余数在ah中
178		inc	ah			; z ++ 相当于余数加1，得到柱面的开始扇区号
179		mov	cl, ah			; cl <- 起始扇区号
180		mov	dh, al			; dh <- y
181		shr	al, 1			; y >> 1 (y/BPB_NumHeads) 把商al右移1得到柱面号
182		mov	ch, al			; ch <- 柱面号
183		and	dh, 1			; dh & 1 = 磁头号 ；商and1，得到磁头号，磁头号本来就要赋给dh
184		pop	bx			; 恢复 bx
185		; 至此, "柱面号, 起始扇区, 磁头号" 全部得到
186		mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
这样， cl 起始扇区号，ch 柱面号，dl 驱动器号，dh磁头号 就全部有值了，这样就把ax中的扇区数转化过来了，ax中的值也没用了。

；至此int 13中还有ah和al没有赋值，请看下面
187	.GoOnReading:
188		mov	ah, 2			; 读 ah=2表示要读
189		mov	al, byte [bp-2]		; 读 al 个扇区
190		int	13h                  ；所有的参数都有值了。可以调用
191		jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 
192						; 这时就不停地读, 直到正确为止
193		add	esp, 2
194		pop	bp
195
196		ret

```

由于代码中用到了堆栈，还需要初始化堆栈

```

14      BaseOfStack		equ	07c00h	; 堆栈基地址(栈底, 从这个位置向低地址生长)

...

48 		mov	ax, cs
49 		mov	ds, ax
50 		mov	es, ax
51 		mov	ss, ax
52 		mov	sp, BaseOfStack


```

读扇区的函数写好了，下面就开始在软盘中寻找Loader.bin

```

		
54 		xor	ah, ah	; `.
55 		xor	dl, dl	;  |  软驱复位
56 		int	13h	; /执行软驱复位
57 		
58 	; 下面在 A 盘的根目录寻找 LOADER.BIN
59 		mov	word [wSectorNo], SectorNoOfRootDirectory ;  SectorNoOfRootDirectory=19，根目录开始的扇区，初值等于19。
60 	LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
61 		cmp	word [wRootDirSizeForLoop], 0	;  `. 判断根目录区是不是已经读完,初始值是14个扇区，前面有针对这个知识点说明！
62 		jz	LABEL_NO_LOADERBIN		;  /  如果读完表示没有找到 LOADER.BIN
63 		dec	word [wRootDirSizeForLoop]	; 根目录扇区总数减1
64 		mov	ax, BaseOfLoader
65 		mov	es, ax			; es <- BaseOfLoader
66 		mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader
67 		mov	ax, [wSectorNo]		; ax <- Root Directory 中的某 Sector 号 初始值是第19个
68 		mov	cl, 1
69 		call	ReadSector
70 
71 		mov	si, LoaderFileName	; ds:si -> "LOADER  BIN" LoaderFileName其实就是字符串
72 		mov	di, OffsetOfLoader	; es:di -> BaseOfLoader:0100
73 		cld
74 		mov	dx, 10h              ; 因为一个扇区最多有512/32=16个根目录
75 	LABEL_SEARCH_FOR_LOADERBIN:
76 		cmp	dx, 0				   ; `. 循环次数控制,
77 		jz	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR ;  / 如果已经读完了一个 Sector,
78 		dec	dx				   ; /  就跳到下一个 Sector
79 		mov	cx, 11             ; 因为根目录的DIR_Name有11个字节
80 	LABEL_CMP_FILENAME:
81 		cmp	cx, 0
82 		jz	LABEL_FILENAME_FOUND	; 如果比较了 11 个字符都相等, 表示找到
83 		dec	cx
84 		lodsb				; ds:si -> al
85 		cmp	al, byte [es:di]
86 		jz	LABEL_GO_ON
87 		jmp	LABEL_DIFFERENT		; 只要发现不一样的字符就表明本 DirectoryEntry
88 						; 不是我们要找的 LOADER.BIN
89 	LABEL_GO_ON:
90 		inc	di
91 		jmp	LABEL_CMP_FILENAME	; 继续循环
92 
93 	LABEL_DIFFERENT:
94 		and	di, 0FFE0h		; else `. di &= E0 为了让它指向本条目开头, FFE0h = 1111111111100000(低5位=32=目录条目大小),低5位清零
95 		add	di, 20h			;       |di += 20h  下一个目录条目 20h=32个字节
96 		mov	si, LoaderFileName	;       | di += 20h  下一个目录条目
97 		jmp	LABEL_SEARCH_FOR_LOADERBIN;    /
98 
99 	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
100		add	word [wSectorNo], 1
101		jmp	LABEL_SEARCH_IN_ROOT_DIR_BEGIN
102
103	LABEL_NO_LOADERBIN:
104		mov	dh, 2			; "No LOADER."
105		call	DispStr			; 显示字符串
106	%ifdef	_BOOT_DEBUG_
107		mov	ax, 4c00h		; `.
108		int	21h			; /  没有找到 LOADER.BIN, 回到 DOS
109	%else
110		jmp	$			; 没有找到 LOADER.BIN, 死循环在这里
111	%endif
112
113	LABEL_FILENAME_FOUND:			; 找到 LOADER.BIN 后便来到这里继续
114		jmp	$			; 代码暂时停在这里
115

```


这段代码就是遍历根目录区所有的扇区，将每一个扇区加载入内存，然后从中寻找文件名为Loader.bin的条目，直到找到为止。找到的那一刻，es：di是指向条目中字母N后面的那个字符！ 我在注释已经写清楚了！


```

17 	BaseOfLoader		equ	09000h	; LOADER.BIN 被加载到的位置 ----  段地址
18 	OffsetOfLoader		equ	0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址
19 	RootDirSectors		equ	14	; 根目录占用空间
20 	SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号

```

变量和字符串定义

```

118	;============================================================================
119	;变量
120	wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数，
121							; 在循环中会递减至零
122	wSectorNo		dw	0		; 要读取的扇区号
123	bOdd			db	0		; 奇数还是偶数
124
125	;字符串
126	LoaderFileName		db	"LOADER  BIN", 0 ; LOADER.BIN 之文件名
127	; 为简化代码, 下面每个字符串的长度均为 MessageLength
128	MessageLength		equ	9
129	BootMessage:		db	"Booting  " ; 9字节, 不够则用空格补齐. 序号 0
130	Message1		db	"Ready.   " ; 9字节, 不够则用空格补齐. 序号 1
131	Message2		db	"No LOADER" ; 9字节, 不够则用空格补齐. 序号 2
132	;============================================================================
133

```

代码定义的字符串都是设定为9个字节，不够用空格补齐！相当于一个二维数组，定位的时候通过数字就可以。

显示字符串的函数名 DispStr

```

135	;----------------------------------------------------------------------------
136	; 函数名: DispStr
137	;----------------------------------------------------------------------------
138	; 作用:
139	;	显示一个字符串, 函数开始时 dh 中应该是字符串序号(0-based)
140	DispStr:
141		mov	ax, MessageLength
142		mul	dh
143		add	ax, BootMessage
144		mov	bp, ax			; `.
145		mov	ax, ds			;  | ES:BP = 串地址
146		mov	es, ax			; /
147		mov	cx, MessageLength	; CX = 串长度
148		mov	ax, 01301h		; AH = 13,  AL = 01h
149		mov	bx, 0007h		; 页号为0(BH = 0) 黑底白字(BL = 07h)
150		mov	dl, 0
151		int	10h			; int 10h
152		ret
153
154

```



![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191117151202.png)


书上说打断点是在第114行，也就是
```

LABEL_FILENAME_FOUND:			; 找到 LOADER.BIN 后便来到这里继续
	jmp	$			; 代码暂时停在这里

```
通过bochs 命令sreg 和r 指令看出来es:di是0x9000:0x12b,也就是内存0x9012b处

确定能找到文件，就开始下面的工作，把loader.bin加载到内存里面去！

现在我们有Loader.bin的起始扇区号，我们需要用这个扇区号来做两件事：一件是把起始扇区装入内存，另外一件则是通过过它找到FAT中的项，从而找到Loader占用的其余所有的扇区。

这里我们把loader装入内存的BaseOfLoader:offsetofLoader处，前面章节已经知道如何装入扇区了，现在要从FAT 中找到一个项！

如果loader占用多个扇区的话，我们可能需要重复从FAT中找相应的项目，所以写一个函数来实现，函数的输入就是扇区号，输出是对应的FAT项的值。

```

21 	SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号

23 	DeltaSectorNo		equ	17	; DeltaSectorNo = BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) - 2
24 						; 文件的开始Sector号 = DirEntry中的开始Sector号 + 根目录占用Sector数目 + DeltaSectorNo

...

258	;----------------------------------------------------------------------------
259	; 函数名: GetFATEntry
260	;----------------------------------------------------------------------------
261	; 作用:
262	;	找到序号为 ax 的 Sector 在 FAT 中的条目, 结果放在 ax 中
263	;	需要注意的是, 中间需要读 FAT 的扇区到 es:bx 处, 所以函数一开始保存了 es 和 bx

也就是参数存放在ax寄存器中，表示一个簇号。输出结果也是该簇号在FAT表的FATENTRY，它的内容仍然是一个簇号，表示文件下一部分所在簇的簇号。“簇”表示一个或多个扇区的集合。FAT12中，一个簇就是一个扇区，512字节。簇号，从软盘的数据区开始，从2开始编号（0，1为系统使用），本质上是一个基于FAT表的索引。
264	GetFATEntry:
265		push	es
266		push	bx
；函数中使用到了es和bx所以要进行保存，保存在栈上。

；es:bx会被函数ReadSector所使用，是int 13h所要求的，指向了一个缓冲区，保存从软盘读进来的数据。

267		push	ax
268		mov	ax, BaseOfLoader; `.
269		sub	ax, 0100h	;  | 在 BaseOfLoader 后面留出 4K 空间用于存放 FAT ；因为这是个段基址，0100h还要乘上个16。
270		mov	es, ax		; /
271		pop	ax
272		mov	byte [bOdd], 0
273		mov	bx, 3
274		mul	bx			; dx:ax = ax * 3
275		mov	bx, 2
276		div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
277		cmp	dx, 0
278		jz	LABEL_EVEN
279		mov	byte [bOdd], 1
；以上部分是整个函数的难点所在，用户计算簇号在FAT表中所对应的FATENTRY相对于FAT首地址的偏移。

；从书上可以得知，FAT12中每个FATENTRY是12位的。所以如下：

7654 | 3210(byte1)    7654|3210(byte2)    7654|3210(byte3)

byte1和byte2的低4位表示一个Entry；根据Big-Endian，Entry内容为：3210(byte2)76543210(byte1)

byte3和byte2的高4位表示一个Entry；根据Big-Endian，Entry内容为：76543210(byte3)7654(byte2)

所以这里存在一个奇偶数的问题，以字节为单位。以上为例，Entry0偏移为0，Entry1偏移为1，Entry2偏移为3。

以INT[“簇号”*1.5]的方式增加。这也就是为什么上面先乘3再除2来计算。

根据DIV指令规定，商保存在ax中，余数在dx中。所以此时ax就是FATENTRY在FAT中以字节为边界的偏移量。
280	LABEL_EVEN:;偶数
281		; 现在 ax 中是 FATEntry 在 FAT 中的偏移量,下面来
282		; 计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
283		xor	dx, dx			
284		mov	bx, [BPB_BytsPerSec]
285		div	bx ; dx:ax / BPB_BytsPerSec
286			   ;  ax <- 商 (FATEntry 所在的扇区相对于 FAT 的扇区号)
287			   ;  dx <- 余数 (FATEntry 在扇区内的偏移)
         ；dx:ax/BPB_BytsPerSec，所以ax现在是扇区的个数，dx是在最后一个扇区中的偏移。
288		push	dx
289		mov	bx, 0 ; bx <- 0 于是, es:bx = (BaseOfLoader - 100):00
；es:bx指向了目标空间的首地址，就是函数开始在BaseOfLoader之后留出的4KB空间。
290		add	ax, SectorNoOfFAT1 ; 此句之后的 ax 就是 FATEntry 所在的扇区号
；SectorNoOfFAT1是FAT1的相对扇区号，物理上就是0面0道1扇区。FAT12有两个完全一样的FAT表，FAT1和FAT2。
291		mov	cl, 2
；cl总保存ReadSector函数的参数，表示要连续读入2个扇区。另一个参数ax在前面已经给出。
292		call	ReadSector ; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界
293				   ; 发生错误, 因为一个 FATEntry 可能跨越两个扇区
294		pop	dx
295		add	bx, dx
；在add之前，bx为目标空间的首地址，其中内容是包含FATEntry的扇区。bx+dx就定位到了该FATEntry。
296		mov	ax, [es:bx]
；ax中为es:bx指向的空间的内容。
297		cmp	byte [bOdd], 1
；是奇数的簇号还是偶数的簇号？？
298		jnz	LABEL_EVEN_2
299		shr	ax, 4
；7654 | 3210(byte1)    7654|3210(byte2)    7654|3210(byte3)

根据上面这个例子，奇数的簇号，1，偏移为1，所以读要ax中的时候，因为ax是16位，所以根据Big-Endian，ax的内容为7654|3210(byte3)|7654|3210(byte2)，所以右移4位就可以了。

偶数簇号，0，偏移为0，读入ax后，ax的内容为7654|3210(byte2)7654 | 3210(byte1)，我们只需要低12位即可，

所以and ax,0FFFh。
300	LABEL_EVEN_2:
301		and	ax, 0FFFh
302
303	LABEL_GET_FAT_ENRY_OK:
304
305		pop	bx
306		pop	es
307		ret

```

我在代码弄了详细的注释！

针对奇偶性，

FAT12 entry location:
FAT表项的偏移扇区号 = 保留扇区数 + N*1.5 / 每扇区字节数  <- 取商值
   ThisFATSecNum = BPB_ResvdSecCnt + ((N + (N / 2)) / BPB_BytsPerSec);
   
 FAT表项在扇区中的偏移位置 = N*1.5 % 每扇区字节数   <- 取余数
   ThisFATEntOffset = (N + (N / 2)) % BPB_BytsPerSec;

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191117235035.png)

从代码可以看出我们新增了宏SecotorNoOfFAT1，它与前面提到的RootDirSectors、SectorNoOfRootDirectory等宏一起，与FAT12有关的几个数字都定义成宏，而不是在程序中进行计算。

一方面，这是为缩小引导扇区代码考虑；另一个方面，这些数字一般情况下不会变的！

取得FAT 项的这段代码中有个问题，就是要区别对待扇区号是奇数还是偶数！

由于FAT项目可能会跨越两个扇区，所以在代码中一次总是读2个扇区，以避免在边界发生错误！

现在都准备好了，扇区都可以读取了，FAT 表项也可以读取了。开始加载Loader吧

代码如下：

```

20 	RootDirSectors		equ	14	; 根目录占用空间

....


127	LABEL_FILENAME_FOUND:			; 找到 LOADER.BIN 后便来到这里继续
128		mov	ax, RootDirSectors    ;mov ax,14   RootDirSectors 的值是14
129		and	di, 0FFE0h		; di -> 当前条目的开始 前面章节提到过，低五位清零 di是32的整数倍了
130		add	di, 01Ah		; di -> 首 Sector   +26字节偏移就是DIR_FstClus 也就是此条目对应开始的簇号
131		mov	cx, word [es:di]  ;mov cx,3 通过bochs调试本例中是3
132		push	cx			; 保存此 Sector 在 FAT 中的序号
133		add	cx, ax           ; 3+14 = 17
134		add	cx, DeltaSectorNo	; cl <- LOADER.BIN的起始扇区号(0-based) 3+14+17=34，34是loader.bin的起始扇区
135		mov	ax, BaseOfLoader
136		mov	es, ax			; es <- BaseOfLoader
137		mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader
138		mov	ax, cx			; ax <- Sector 号  ax=34
139
140	LABEL_GOON_LOADING_FILE:
141		push	ax			; `.
142		push	bx			;  |  bx=0100h 因为之前就是 bx <- OffsetOfLoader 
143		mov	ah, 0Eh			;  | 每读一个扇区就在 "Booting  " 后面
144		mov	al, '.'			;  | 打一个点, 形成这样的效果:
145		mov	bl, 0Fh			;  | Booting ......
146		int	10h			;  |
147		pop	bx			;  |
148		pop	ax			; /
149
150		mov	cl, 1  ;ax=0022h=34，cx=0001h=1，dx=000eh=14，bx=0100h
151		call	ReadSector
152		pop	ax			;   取出此 Sector 在 FAT 中的序号     ax=0003h， 把ss：0x00007bfe的值0003赋给ax
153		call	GetFATEntry
154		cmp	ax, 0FFFh
155		jz	LABEL_FILE_LOADED
156		push	ax			; 保存 Sector 在 FAT 中的序号
157		mov	dx, RootDirSectors
158		add	ax, dx
159		add	ax, DeltaSectorNo
160		add	bx, [BPB_BytsPerSec]
161		jmp	LABEL_GOON_LOADING_FILE
162	LABEL_FILE_LOADED:
163
164		mov	dh, 1			; "Ready."
165		call	DispStr			; 显示字符串
166
167	; *****************************************************************************************************
168		jmp	BaseOfLoader:OffsetOfLoader	; 这一句正式跳转到已加载到内
169							; 存中的 LOADER.BIN 的开始处，
170							; 开始执行 LOADER.BIN 的代码。
171							; Boot Sector 的使命到此结束
172	; *****************************************************************************************************
173

```


下一步要清屏显示，要不然看不清楚！

清屏显示字符串代码：

```

58 		; 清屏
59 		mov	ax, 0600h		; AH = 6,  AL = 0h
60 		mov	bx, 0700h		; 黑底白字(BL = 07h)
61 		mov	cx, 0			; 左上角: (0, 0)
62 		mov	dx, 0184fh		; 右下角: (80, 50)
63 		int	10h			; int 10h
64 
65 		mov	dh, 0			; "Booting  "
66 		call	DispStr			; 显示字符串
67 		

```


执行效果如图所示，可以看出显示L

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191119003240.png)


