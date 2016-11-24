---
date: 2016-11-22 13:57
title: Elf简介
tags: [linux]
categories: [android,linker]
---

ELF(Executable and Linkable Format)即可执行连接文件格式，是Linux默认的目标文件格式，分析elf文件有助于理解一些重要的系统概念，例如程序的编译和链接，程序的加载和运行等。
##ELF文件类型
a)可重定位文件:用户和其他目标文件一起创建可执行文件或者共享目标文件,例如lib*.a文件。
b)可执行文件：用于生成进程映像，载入内存执行,例如编译好的可执行文件a.out。
c)共享目标文件：用于和其他共享目标文件或者可重定位文件一起生成elf目标文件或者和执行文件一起创建进程映像，例如lib*.so文件。

##ELF文件的组织
ELF文件参与程序的连接(建立一个程序)和程序的执行(运行一个程序)，编译器和链接器将其视为节头表(section header table)描述的一些节(section)的集合，而加载器则将其视为程序头表(program header table)描述的段(segment)的集合，通常一个段可以包含多个节。可重定位文件都包含一个节头表，可执行文件都包含一个程序头表。共享文件两者都包含有。为此，ELF文件格式同时提供了两种看待文件内容的方式，反映了不同行为的不同要求。

###文件头(Elf header)
Elf头在程序的开始部位，作为引路表描述整个ELF的文件结构，其信息大致分为四部分：一是系统相关信息，二是目标文件类型，三是加载相关信息，四是链接相关信息。
其中系统相关信息包括elf文件魔数(标识elf文件)，平台位数，数据编码方式，elf头部版本，硬件平台e_machine，目标文件版本 e_version，处理器特定标志e_ftags：这些信息的引入极大增强了elf文件的可移植性，使交叉编译成为可能。目标文件类型用e_type的值表示，可重定位文件为1，可执行文件为2，共享文件为3;加载相关信息有：程序进入点e_entry．程序头表偏移量e_phoff，elf头部长度 e_ehsize，程序头表中一个条目的长度e_phentsize，程序头表条目数目e_phnum;链接相关信息有：节头表偏移量e_shoff，节头表中一个条目的长度e_shentsize，节头表条目个数e_shnum ，节头表字符索引e shstmdx。可使用命令"readelf -h filename"来察看文件头的内容。
```
typedef struct elf32_hdr{
unsigned char e_ident[EI_NIDENT];
Elf32_Half e_type;//目标文件类型
Elf32_Half e_machine;//硬件平台
Elf32_Word e_version;//elf头部版本
Elf32_Addr e_entry;//程序进入点
Elf32_Off e_phoff;//程序头表偏移量
Elf32_Off e_shoff;//节头表偏移量
Elf32_Word e_flags;/处理器特定标志
Elf32_Half e_ehsize;//elf头部长度
Elf32_Half e_phentsize;//程序头表中一个条目的长度
Elf32_Half e_phnum;//程序头表条目数目
Elf32_Half e_shentsize;//节头表中一个条目的长度
Elf32_Half e_shnum;//节头表条目个数
Elf32_Half e_shstrmdx;//节头表字符索引
}Elf32_Ehdr;

linker64的文件头：
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x6cb4
  Start of program headers:          64 (bytes into file)
  Start of section headers:          924600 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         8
  Size of section headers:           64 (bytes)
  Number of section headers:         24
  Section header string table index: 21
```

###程序头表(program header table)
程序头表告诉系统如何建立一个进程映像．它是从加载执行的角度来看待elf文件．从它的角度看．elf文件被分成许多段，elf文件中的代码、链接信息和注释都以段的形式存放。每个段都在程序头表中有一个表项描述，包含以下属性：段的类型，段的驻留位置相对于文件开始处的偏移，段在内存中的首字节地址，段的物理地址，段在文件映像中的字节数．段在内存映像中的字节数，段在内存和文件中的对齐标记。可用"readelf -l filename"察看程序头表中的内容。程序头表的结构如下：
```
typedef struct elf32_phdr{
Elf32_Word p_type; //段的类型
Elf32_Off p_offset; //段的位置相对于文件开始处的偏移
Elf32_Addr p_vaddr; //段在内存中的首字节地址
Elf32_Addr p_paddr;//段的物理地址
Elf32_Word p_filesz;//段在文件映像中的字节数
Elf32_Word p_memsz;//段在内存映像中的字节数
Elf32_Word p_flags;//段的标记
Elf32_Word p_align;，/段在内存中的对齐标记
)Elf32_Phdr;

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001c0 0x00000000000001c0  R      8
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000a7a9c 0x00000000000a7a9c  R E    1000
  LOAD           0x00000000000a8630 0x00000000000a9630 0x00000000000a9630
                 0x00000000000036b0 0x0000000000009f94  RW     1000
  DYNAMIC        0x00000000000aaa98 0x00000000000aba98 0x00000000000aba98
                 0x0000000000000150 0x0000000000000150  RW     8
  NOTE           0x0000000000000200 0x0000000000000200 0x0000000000000200
                 0x0000000000000020 0x0000000000000020  R      4
  GNU_EH_FRAME   0x00000000000a58d0 0x00000000000a58d0 0x00000000000a58d0
                 0x00000000000021cc 0x00000000000021cc  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0
  GNU_RELRO      0x00000000000a8630 0x00000000000a9630 0x00000000000a9630
                 0x00000000000029d0 0x00000000000029d0  RW     10
```

###节头表(section header table)
节头表描述程序节，为编译器和链接器服务。它把elf文件分成了许多节．每个节保存着用于不同目的的数据．这些数据可能被前面的程序头重复使用，完成一次任务所需的信息往往被分散到不同的节里。由于节中数据的用途不同，节被分成不同的类型，每种类型的节都有自己组织数据的方式。每一个节在节头表中都有一个表项描述该节的属性，节的属性包括小节名在字符表中的索引，类型，属性，运行时的虚拟地址，文件偏移，以字节为单位的大小，小节的对齐等信息，可使用"readelf -S filename"来察看节头表的内容。节头表的结构如下：
```
typedef struct{
Elf32_Word sh_name;//小节名在字符表中的索引
E1t32_Word sh_type;//小节的类型
Elf32_Word sh_flags;//小节属性
Elf32_Addr sh_addr; //小节在运行时的虚拟地址
Elf32_Off sh_offset;//小节的文件偏移
Elf32_Word sh_size;//小节的大小．以字节为单位
Elf32_Word sh_link;//链接的另外一小节的索引
Elf32 Word sh_info;//附加的小节信息
Elf32 Word sh_addralign;//小节的对齐
Elf32 Word sh_entsize; //一些sections保存着一张固定大小入口的表。就像符号表
}Elf32_Shdr;

linker64的section：
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.build-i NOTE             0000000000000200  00000200
       0000000000000020  0000000000000000   A       0     0     4
  [ 2] .dynsym           DYNSYM           0000000000000220  00000220
       0000000000000090  0000000000000018   A       3     1     8
  [ 3] .dynstr           STRTAB           00000000000002b0  000002b0
       0000000000000033  0000000000000000   A       0     0     1
  [ 4] .gnu.hash         GNU_HASH         00000000000002e8  000002e8
       0000000000000038  0000000000000000   A       2     0     8
  [ 5] .rela.dyn         RELA             0000000000000320  00000320
       0000000000006258  0000000000000018   A       2     0     8
  [ 6] .text             PROGBITS         0000000000006580  00006580
       0000000000082124  0000000000000000  AX       0     0     64
  [ 7] .rodata           PROGBITS         00000000000886b0  000886b0
       0000000000009770  0000000000000000   A       0     0     16
  [ 8] .gcc_except_table PROGBITS         0000000000091e20  00091e20
       0000000000009a20  0000000000000000   A       0     0     4
  [ 9] .eh_frame         PROGBITS         000000000009b840  0009b840
       000000000000a090  0000000000000000   A       0     0     8
  [10] .eh_frame_hdr     PROGBITS         00000000000a58d0  000a58d0
       00000000000021cc  0000000000000000   A       0     0     4
  [11] .data.rel.ro      PROGBITS         00000000000a9630  000a8630
       0000000000002430  0000000000000000  WA       0     0     16
  [12] .init_array       INIT_ARRAY       00000000000aba60  000aaa60
       0000000000000038  0000000000000000  WA       0     0     8
  [13] .dynamic          DYNAMIC          00000000000aba98  000aaa98
       0000000000000150  0000000000000010  WA       3     0     8
  [14] .got              PROGBITS         00000000000abbe8  000aabe8
       0000000000000400  0000000000000000  WA       0     0     8
  [15] .got.plt          PROGBITS         00000000000abfe8  000aafe8
       0000000000000018  0000000000000000  WA       0     0     8
  [16] .data             PROGBITS         00000000000ac000  000ab000
       0000000000000ce0  0000000000000000  WA       0     0     32
  [17] .bss              NOBITS           00000000000ad000  000abce0
       00000000000065c4  0000000000000000  WA       0     0     4096
  [18] .comment          PROGBITS         0000000000000000  000abce0
       0000000000000063  0000000000000001  MS       0     0     1
  [19] .note.gnu.gold-ve NOTE             0000000000000000  000abd44
       000000000000001c  0000000000000000           0     0     4
  [20] .gnu_debuglink    PROGBITS         0000000000000000  000abd60
       0000000000000010  0000000000000000           0     0     1
  [21] .shstrtab         STRTAB           0000000000000000  000abd70
       00000000000000f4  0000000000000000           0     0     1
  [22] .symtab           SYMTAB           0000000000000000  000abe68
       0000000000019ad0  0000000000000018          23   4377     8
  [23] .strtab           STRTAB           0000000000000000  000c5938
       000000000001c27b  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
####特殊节
![](~/18-29-04.jpg)

<font color=#FF0000>.bss</font>
构成程序的内存映像的未初始化数据。根据定义，系统在程序开始运行时会将数据初始化为零。如节类型 SHT_NOBITS 所指明的那样，此节不会占用任何文件空间。
.comment
注释信息，通常由编译系统的组件提供。此节可以由 mcs(1) 进行处理。
<font color=#FF0000>.data、.data1</font>
构成程序的内存映像的已初始化数据。
<font color=#FF0000>.dynamic</font>
动态链接信息。有关详细信息，请参见动态节。
.dynstr
进行动态链接所需的字符串，通常是表示与符号表各项关联的名称的字符串。
.dynsym
动态链接符号表。有关详细信息，请参见符号表节。
.eh_frame_hdr、.eh_frame
用于展开栈的调用帧信息。
.fini
可执行指令，用于构成包含此节的可执行文件或共享目标文件的单个终止函数。有关详细信息，请参见初始化和终止例程。
.fini_array
函数指针数组，用于构成包含此节的可执行文件或共享目标文件的单个终止数组。有关详细信息，请参见初始化和终止例程。
.got
全局偏移表。有关详细信息，请参见全局偏移表（特定于处理器）。
.hash
符号散列表。有关详细信息，请参见散列表节。
.init
可执行指令，用于构成包含此节的可执行文件或共享目标文件的单个初始化函数。有关详细信息，请参见初始化和终止例程。
.init_array
函数指针数组，用于构成包含此节的可执行文件或共享目标文件的单个初始化数组。有关详细信息，请参见初始化和终止例程。
.interp
程序的解释程序的路径名。有关详细信息，请参见程序的解释程序。
.lbss
特定于 x64 的未初始化的数据。此数据与 .bss 类似，但用于大小超过 2 GB 的节。
.ldata、.ldata1
特定于 x64 的已初始化数据。此数据与 .data 类似，但用于大小超过 2 GB 的节。
.lrodata、.lrodata1
特定于 x64 的只读数据。此数据与 .rodata 类似，但用于大小超过 2 GB 的节。
.note
注释节中说明了该格式的信息。
.plt
过程链接表。有关详细信息，请参见过程链接表（特定于处理器）。
.preinit_array
函数指针数组，用于构成包含此节的可执行文件或共享目标文件的单个预初始化数组。有关详细信息，请参见初始化和终止例程。
.rela
不适用于特定节的重定位。此节的用途之一是用于寄存器重定位。有关详细信息，请参见寄存器符号。
.relname、.relaname
重定位信息，如重定位节中所述。如果文件具有包括重定位的可装入段，则此节的属性将包括 SHF_ALLOC 位。否则，该位会处于禁用状态。通常，name 由应用重定位的节提供。因此，.text 的重定位节的名称通常为 .rel.text 或 .rela.text。
.rodata、.rodata1
通常构成进程映像中的非可写段的只读数据。有关详细信息，请参见程序头。
.shstrtab
节名称。
.strtab
字符串，通常是表示与符号表各项关联的名称的字符串。如果文件具有包括符号字符串表的可装入段，则此节的属性将包括 SHF_ALLOC 位。否则，该位会处于禁用状态。
.symtab
符号表，如符号表节中所述。如果文件具有包括符号表的可装入段，则此节的属性将包括 SHF_ALLOC 位。否则，该位会处于禁用状态。
.symtab_shndx
此节包含特殊符号表的节索引数组，如 .symtab 所述。如果关联的符号表节包括 SHF_ALLOC 位，则此节的属性也将包括该位。否则，该位会处于禁用状态。
.tbss
此节包含构成程序的内存映像的未初始化线程局部数据。根据定义，为每个新执行流实例化数据时，系统都会将数据初始化为零。如节类型 SHT_NOBITS 所指明的那样，此节不会占用任何文件空间。有关详细信息，请参见第 8 章。
.tdata、.tdata1
这些节包含已初始化的线程局部数据，这些数据构成程序的内存映像。对于每个新执行流，系统会对其内容的副本进行实例化。有关详细信息，请参见第 8 章。
.text
程序的文本或可执行指令。
.SUNW_bss
共享目标文件的部分初始化数据，这些数据构成程序的内存映像。数据会在运行时进行初始化。如节类型 SHT_NOBITS 所指明的那样，此节不会占用任何文件空间。
.SUNW_cap
功能要求。有关详细信息，请参见功能节。
.SUNW_capchain
功能链表。有关详细信息，请参见功能节。
.SUNW_capinfo
功能符号信息。有关详细信息，请参见功能节。
.SUNW_heap
从 dldump(3C) 中创建的动态可执行文件的堆。
.SUNW_dynsymsort
.SUNW_ldynsym – .dynsym 组合符号表中符号的索引数组。该索引进行排序，以按照地址递增的顺序引用符号。不表示变量或函数的符号不包括在内。对于冗余全局符号和弱符号，仅保留弱符号。有关详细信息，请参见符号排序节。
.SUNW_dyntlssort
.SUNW_ldynsym – .dynsym 组合符号表中线程局部存储符号的索引数组。该索引进行排序，以按照偏移递增的顺序引用符号。不表示 TLS 变量的符号不包括在内。对于冗余全局符号和弱符号，仅保留弱符号。有关详细信息，请参见符号排序节。
.SUNW_ldynsym
扩充 .dynsym 节。此节包含局部函数符号，以在完整的 .symtab 节不可用时在上下文中使用。链接编辑器始终将 .SUNW_ldynsym 节的数据放置在紧邻 .dynsym 节之前。这两个节始终使用相同的 .dynstr 字符串表节。这种放置和组织方式使两个符号表可以被视为一个更大的符号表。请参见符号表节。
.SUNW_move
部分初始化数据的附加信息。有关详细信息，请参见移动节。
.SUNW_reloc
重定位信息，如重定位节中所述。此节是多个重定位节的串联，用于为引用各个重定位记录提供更好的临近性。由于仅有重定位记录的偏移有意义，因此节的 sh_info 值为零。
.SUNW_syminfo
其他符号表信息。有关详细信息，请参见Syminfo 表节。
.SUNW_version
版本控制信息。有关详细信息，请参见版本控制节。
具有点 (.) 前缀的节名为系统而保留，但如果这些节的现有含义符合要求，则应用程序也可以使用这些节。应用程序可以使用不带前缀的名称，以避免与系统节产生冲突。使用目标文件格式，可以定义非保留的节。一个目标文件可以包含多个同名的节。
保留用于处理器体系结构的节名称通过在节名称前加上体系结构名称的缩写而构成。该名称应来自用于 e_machine 的体系结构名称。例如，.Foo.psect 是根据 FOO 体系结构定义的 psect 节。
现有扩展使用其历史名称。
