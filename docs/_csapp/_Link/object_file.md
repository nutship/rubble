### 1. 编译过程

=== "main.c"

    ```C
    #include <stdio.h>

    void hello(void);

    int main() {
        hello();
        return 0;
    }
    ```

=== "hello.c"

    ```C
    #include <stdio.h>

    void hello() {
        printf("hello world!");
    }
    ```

compiler driver 将源文件翻译为可执行文件的过程:

-   C 预处理器 (cpp) 将源程序 `main.c` 翻译为 ascii 中间文件 `main.i`
-   C 编译器 (cc1) 将 `main.i` 翻译为 ascii 汇编文件 `main.s`
-   汇编器 (as) 将 `main.s` 翻译为可重定位目标文件 `main.o`
-   链接器 ld 将所有 `.o` 和必要的系统目标文件结合，创建可执行文件

最后，shell 调用 loader，将可执行文件的代码和数据复制到内存，并将控制转移到它的开头

```console
$ cpp -o ./tmp/main.i main.c
$ /usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/cc1 -o ./tmp/main.s ./tmp/main.i
$ as -o ./tmp/main.o ./tmp/main.s
$ ld -o prog -dynamic-linker /lib64/ld-linux-x86-64.so.2  /usr/lib/crt1.o /usr/lib/crti.o /usr/lib/crtn.o  /usr/lib/libc.so ./tmp/main.o
$ ./prog
```

### 2. 目标文件

#### (1). 分类

<font class="u_n">

objected files come in three forms:

-   executable object file:
    -   当前流行的格式主要为 Windows 的 PE & Linux 的 ELF，可以直接加载到内存执行
-   relocatable object file (Windows: `.obj` & Linux: `.o`):
    -   用于链接成可执行文件和共享目标文件
-   shared object file (Windows: `.dll` & Linux: `.so`):
    -   可以与其他可重定位目标文件/共享目标文件链接，产生新的目标文件；也可以与可执行文件结合，作为进程映像的一部分运行

</font>

??? hint "可执行文件格式的历史"

    Unix 最早的可执行文件格式是 a.out，但由于其设计过于简单，无法适应共享库的概念；Unix System V Release 3 首先提出并使用了 COFF 格式，微软基于 COFF 提出了 PE 格式标准；System V Release 4 引入了 ELF 格式，也就是当前 Linux 的可执行文件格式。

#### (2). 格式

典型的 ELF 可重定位目标文件包含以下 section (代码区、数据区、辅助(链接/调试)区):

<font class="t_a%0&0_b%10_h%3&0">

| section                | description                                                                   |
| :--------------------- | :---------------------------------------------------------------------------- |
| `.text`                | 已编译程序的机器码                                                            |
| `.rodata`              | 只读数据，如 `printf` 的 format 串、`switch` 的跳转表                         |
| `.data`                | 已初始化的全局和静态变量                                                      |
| `.bss`<rspan>3</rspan> | 未初始化或初始化为 0 的全局和静态变量 (better save space) <br>                |
| &emsp;                 | 在可执行目标文件和可重定位目标文件中，`.bss` 均不占实际空间                   |
| &emsp;                 |                                                                               |
| `.symtab`              | 符号表，定义 & 引用的全局变量或函数的信息                                     |
| `.rel.text`            | relocation table of `.text`                                                   |
| `.rel.data`            | relocation table of `.data`                                                   |
| `.debug`               | (`-g`) 调试符号表，条目包括局部变量和类型定义、定义和引用的全局变量、C 源文件 |
| `.line`                | (`-g`) 原始 C 源程序中的行号和 `.text` 中机器指令中的映射                     |
| `.strtab`              | 字符串表，表中的内容会被 `.symtab`、`.debug`、section header table 引用       |
| `ABS`                  | 伪节，不该被重定位的符号                                                      |
| `COMMON `              | 伪节，其存在与符号解析方式有关，有的实现将弱符号 (未初始化的全局变量) 放在这  |
| `UNDEF`                | 伪节，未定义的符号 (本模块引用，其他地方定义)                                 |

</font>

将代码段和数据段分开存储的好处:

-   程序装载后，数据和指令映射到两个虚存区域，对于进程数据区可读写，指令区只读，方便权限控制
-   有利于程序局部性
-   当系统中运行同一程序的多个副本，所有只读区只需要保存一份，这是一个很重要的概念

### 3. ELF 文件结构

<!-- prettier-ignore-start -->

=== "&emsp;&emsp;&emsp; ELF Sections &emsp;&emsp;&emsp;"

    <font class="i_n_a%10&10_b%10&0" id="ELF sections">
    <img src="../img/sections.png">
    </font>

=== "&emsp;&emsp; Example: &ensp; SimpleSection.c &emsp;&emsp;"

    ```C
    int printf(const char* format, ...);

    int global_init_var = 84;
    int global_unit_var;

    void func1(int i) {
        printf("%d\n", i);
    }

    int main(void) {
        static int static_var = 85;
        static int static_var2;
        int a = 1;
        int b;
        func1(static_var + static_var2 + a + b);
        return a;
    }
    ```
<!-- prettier-ignore-end -->

#### (1). ELF Header

ELF Header 描述了整个文件的基本属性，例如 Magic Number、文件类型、程序入口、段表 Offset 等。`elf.h` 定义了自己的变量体系

=== "Elf64_Ehdr"

    ```C
    typedef struct {
        unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
        Elf64_Half	e_type;			/* Object file type */
        Elf64_Half	e_machine;		/* Architecture */
        Elf64_Word	e_version;		/* Object file version */
        Elf64_Addr	e_entry;		/* Entry point virtual address */
        Elf64_Off	e_phoff;		/* Program header table file offset */
        Elf64_Off	e_shoff;		/* Section header table file offset */
        Elf64_Word	e_flags;		/* Processor-specific flags */
        Elf64_Half	e_ehsize;		/* ELF header size in bytes */
        Elf64_Half	e_phentsize;		/* Program header table entry size */
        Elf64_Half	e_phnum;		/* Program header table entry count */
        Elf64_Half	e_shentsize;		/* Section header table entry size */
        Elf64_Half	e_shnum;		/* Section header table entry count */
        Elf64_Half	e_shstrndx;		/* Section header string table index */
    } Elf64_Ehdr;
    ```

=== "readelf -h"

    ```console
    $ readelf -h SimpleSection.o
        ELF 头：
        Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
        类别:                              ELF64
        数据:                              2 补码，小端序 (little endian)
        Version:                           1 (current)
        OS/ABI:                            UNIX - System V
        ABI 版本:                          0
        类型:                              REL (可重定位文件)
        系统架构:                          Advanced Micro Devices X86-64
        版本:                              0x1
        入口点地址：              0x0
        程序头起点：              0 (bytes into file)
        Start of section headers:          2352 (bytes into file)
        标志：             0x0
        Size of this header:               64 (bytes)
        Size of program headers:           0 (bytes)
        Number of program headers:         0
        Size of section headers:           64 (bytes)
        Number of section headers:         21
        Section header string table index: 20
    ```

ELF 文件有 32 位版本和 64 位版本，因此其文件头也有这两种版本。

#### (2). Section Header Table

段表描述各个段的信息，例如 段名、段长、在文件中的偏移 (索引各个段)、读写权限等，数组元素的结构：

```C
typedef struct {
    Elf64_Word	sh_name;		/* Section name (string tbl index) */
    Elf64_Word	sh_type;		/* Section type */
    Elf64_Xword	sh_flags;		/* Section flags */
    Elf64_Addr	sh_addr;		/* Section virtual addr at execution */
    Elf64_Off	sh_offset;		/* Section file offset */
    Elf64_Xword	sh_size;		/* Section size in bytes */
    Elf64_Word	sh_link;		/* Link to another section */
    Elf64_Word	sh_info;		/* Additional section information */
    Elf64_Xword	sh_addralign;		/* Section alignment */
    Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
} Elf64_Shdr;
```
