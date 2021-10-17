### 1. 虚拟内存 $\leftrightarrow$ 主存: 页式缓存

#### (1). 缓存结构设计

由于磁盘本身读取很慢，而且磁盘以扇区为单位读取，导致 DRAM 缓存不命中开销很大，使得

-   虚拟页很大
-   使用全相联结构，并使用页表查询缓存映射，每个虚拟页对应一个页表条目 (PTE)

<img src="../img/vm_cache_structure.png" width=500>

任意时刻，一个虚拟页可能处于以下三个状态之一:

-   未分配: 没有数据与之关联，不占实际的磁盘空间 ($\sf 0 + null$)
-   未缓存: 已与某个文件关联，但未缓存 ($\sf 0 + VA$)
-   已缓存: 已与某个文件关联，且缓存在主存中 ($\sf 1 + PA$)

因此，在请求虚拟页时，可直接根据 PTE 判断虚拟页处于哪个状态。

#### (2). 地址翻译

由于使用页表，虚拟地址不需要 $\rm tag$ 和 $\rm index$ 的概念，只有 $\rm page\ number + offset$。<br>

??? hint "页面命中时，CPU 硬件的工作流程"

    <img src="../img/pagehit.png" width=500>

    <font class="u_nn">

    -   ①. cpu 生成一个 VA，传给 MMU
    -   ②. MMU 根据 VA 和 PTBR 生成 PTE 的地址，向 高速缓存/主存 发出请求
    -   ③. 高速缓存/主存 向 MMU 返回 PTE 的内容
    -   ④. MMU 构造物理地址，传回给 高速缓存/主存
    -   ⑤. 高速主存/缓存 返回物理地址给 cpu

    </font>

??? hint "缺页时，CPU 硬件和内核的工作流程 "

    <img src="../img/pagefault.png" width=560>

    <font class="u_nn">

    -   ① ～ ③. 和页面命中时的流程相同
    -   ④. PTE 中有效位是 0，MMU 据此触发一个异常，传递控制给 OS 内核的异常处理程序
    -   ⑤ ～ ⑥. 处理程序确定主存中的牺牲页，若该页被修改，则将其换出到磁盘；然后，调入新页并更新 PTE
    -   ⑦. 缺页处理程序返回到原来的进程，重新执行导致缺页的指令

    </font>

#### (3). 多级页表

直接实现页表的问题:

-   以 32 位地址空间为例，每个 PTE 需要 $4\rm byte$，假设页面大小 $4\rm KB=2^{12} byte$，则内存需要常驻一个 $\rm 2^{32}/2^{12}\times 4 byte = 4MB$ 的页表。对于 64 位空间情况将更复杂
-   通常情况下，对于一个进程 $4\rm G$ 的地址空间可能有很多都是未分配的

通常用层次结构压缩页表，以 2 级页表为例:

&emsp;&emsp; <img src="../img/multi-level_pt.png" width=666>

一级页表的 PTE 映射一个 $\rm 4MB$ 的 chunk，若 chunk i 每个页都是未分配，PTE i 为空；否则 PTE i 指向一个二级页表的基址，二级页表的 PTE 负责映射一个 $4\rm KB$ 的虚拟页。多级页表因而节省了内存:

-   若一级页表未空，二级页表不需要存在
-   只有一级页表常驻主存，二级页表常用的部分缓存在主存中

### 2. 虚拟内存 $\leftrightarrow$ 磁盘: 内存映射

<font class="i_r_a%10&10_b%10&0" id="&emsp;&emsp; VM of a linux process">

<img src="../img/linux_vm.png" width=360>

#### (1). Linux 虚拟内存结构

Linux 为每个进程维护一个单独的虚拟地址空间，结构如图

-   虚拟内存是段的集合，段之间可以有间隙，段由已分配的虚拟页组成，间隙对应未分配的虚拟页，未分配的虚拟页不占用磁盘空间
-   内核虚拟内存的相同部分:
    -   内核代码、全局数据结构 被映射到所有进程共享的物理页面
    -   物理内存部分和主存是一一对应的，其存在是为了方便内核访问主存特定位置，例如访问页表
-   内核虚拟内存的不同部分: 保存进程的上下文，例如 PCB、内核栈、页表

</font>

#### (2). PCB

Linux 的 PCB 是结构体 `task_struct`，其中保存了运行该进程需要的所有信息，例如 PID、PC、寄存器值、优先级、IO 状态、可执行文件名字等。

&emsp;&emsp; <img src="../img/pcb.png" width=580>

`task_struct` 的一个条目指向 `mm_struct`，描述 VM 的状态；其中感兴趣的字段:

-   `pgd` 指向一级页表的基址
-   `mmap` 指向区域结构 `mm_area_structs` 的链表，`mmap_rb` 指向相同元素的红黑树 (系统级内存映射)
    -   `vm_start / vm_end` 段的开始和结束
    -   `vm_prot` 该段包含的所有页的读写权限
    -   `vm_flags` 该段是共享还是私有 (以及其他信息)
    -   `vm_file` 该段的内存映射的文件信息

??? hint "Linux 缺页判断逻辑"

    假设 MMU 试图翻译某个地址 $\rm A$ 时，触发了一个缺页，处理程序的执行逻辑为:

    -  判断 $\rm A$ 是否在某个 `vm_area_structs` 指定的区域内 (判断是否已分配?)，如果不在则触发段错误并终止这个进程
    -  判断是否有权限读或写该页，若不合法则触发一个保护异常
    -  此时，内核知道这个缺页是对合法虚拟地址进行合法操作造成的。内核选择一个牺牲页 (若修改过则写回磁盘)，CPU 重新执行导致缺页的指令

#### (3). 内存映射

内存映射把虚拟内存系统集成到传统的文件系统，提供了一种简单而高效 (节约主存) 的方式加载代码段和数据段的方式。
一个文件被映射到虚拟内存的一个区域:

-   作为共享对象，进程对虚存共享区的写操作，也会反映到原始文件中 (没找到例子)
-   作为私有对象，进程对私有区的写操作，对其他进程是不可见的，且不需要反映到原文件中，使用写时复制 (copy-on-write) 节省内存

??? hint "Linux 虚拟内存的共享机制"

    <img src="../img/shared_obj.png" width=700>

    -   进程 1 将一个共享对象映射到自己的虚拟内存区域中
    -   进程 2 将同一个共享对象映射到自己的虚存区域，由于每个对象有唯一的文件名，内核可以判定已经有进程共享了这个对象，主存中只需要存放一个副本即可 (猜测: 进程 2 若发现共享区域未缓存，会先去进程一的页表看看缓存没有，再决定是否触发缺页)
    -   暂时没找到需要修改共享对象的例子，书里也没提

    <img src="../img/private_obj.png" width=700>

    -   进程 1 和进程 2 映射了一个私有对象，由于还没写入，先暂时共享，并将页表条目都标记为只读 (为了触发异常)，将 `area_struct` 都标记为写时复制
    -   当有一个进程向私有区域执行一个写操作，会跳转到故障处理程序，创建一个被写的虚拟页的副本，然后写进程更新 PTE，最后故障处理程序返回，重新执行这个写操作

??? hint "段式内存管理"

    这里的段和程序分段不是一个概念，是完全的处理器概念，是 x86 架构独有的，Linux 没有考虑

    -   8086: &ensp; cpu 只有 16 位 而地址总线有 20 位，因此通过 cpu 和寄存器 16 位段基址 + 4 位段偏移寻址
    -   80386: &ensp; 引入了虚拟内存的概念，寻址模式变为 逻辑地址 -段表-> 线性地址 (VM) -页表-> 物理地址
    -   x86-64: &ensp; 废弃段式内存管理

    页式内存管理出现在段式之后，是更先进的概念，80386 的复杂设计没必要深究，猜测可能是为了兼容

### 总结

虚拟内存作为一种抽象，由两部分组成

-   通过页式管理 (缓存) 和主存关联 (硬件 + OS 异常，管理页面)
-   通过内存映射和文件系统关联 (OS，分配页面)

虚拟内存除了使用缓存节省了主存的空间外，在内存管理方面还有很多优点:

-   简化加载: loader 只需要实现类似内存映射的功能，分配虚拟页，并将文件的 section 映射到 `vm_area_structs` 上，该过程并没有从磁盘向主存拷贝任何内容
-   简化链接: 不同进程有相同布局的地址空间，使得链接器的实现不用考虑实际的物理地址
-   简化共享: 不同进程共享的部分，在物理内存中 (和磁盘上?) 只需要一个副本
-   简化内存分配: 和加载一样，只需分配连续的虚拟页然后做内存映射即可，不必物理内存如何工作 (例如不需要关心物理页是否连续)
-   简化权限控制: 在 PTE 上设置标记位，实现读写权限、进程在超级用户模式下才能访问该页等

<!--
为什么，判断地址是否有效要从 vm_area_structs 判断，直接页表不行吗，页表不一定是 Null?
-->