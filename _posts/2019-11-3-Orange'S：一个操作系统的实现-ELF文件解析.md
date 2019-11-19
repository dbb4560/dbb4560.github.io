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

