### 1. I/O 设备

<font class="i_r_b%30" id="硬件结构">

<img src="../img/io_structure.png" width=390>

#### (1). I/O 设备分类

-   块设备: 以块为单位寻址、传送，对应系统调用 `write()`，`read()`, `seek()` 等
-   字符设备: 以字符为单位，对应系统调用 `get()`, `put()` 等
-   网络设备: 网络 I/O 的性能和寻址特点不同于磁盘 I/O，对应 socket 系列的系统调用
-   其他，如时钟

#### (2). I/O 层次

</font>

<font class="i_r_b%30" id="io 层次">

<img src="../img/io_level.png" width=310>

-   设备硬件: 包含控制器和缓冲区，控制器由若干寄存器组成，设备通过控制器和中断机制与 cpu 通信
    -   OS 设置寄存器命令设备发送接受数据、开或关
    -   按下键盘产生中断，跳到键盘驱动注册的 handler
    -   磁盘需要缓冲区
-   设备驱动程序: 按 OS 给定的接口提供实现
    -   例如 linux 的 `file_operation`
    -   设备初始化时，要注册设备的中断 handler (例如 `require_irq()`)
-   设备无关层: 提供缓冲、调度、保护、命名等功能
-   用户进程发起 IO 请求

</font>

??? hint "为什么磁盘需要缓冲区"

    -   可以先缓冲一个或多个块，然后检查校验和，再传送到内存
    -   如果直接传送，则每个字节都需要请求总线，可能造成等待，那么下一个字节来了就需要缓冲

??? hint "linux fs.h &ensp; `file_operation`"

    ```c
    struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *, fl_owner_t id);
        int (*release) (struct inode *, struct file *);
        int (*lock) (struct file *, int, struct file_lock *);
        ...
    };
    ```

### 2. I/O 方式

#### (1). 轮询

内核程序 (cpu) 不断轮询 (polling) 设备的状态寄存器，直到其就绪，执行 I/O 过程

-   如果 I/O 需要时间很短，或 cpu 没有其他事要做 (如嵌入式系统)，轮询很合理
-   否则，I/O 耗时很长，cpu 需要等待并不断轮询耗费指令周期，影响完成其他任务

#### (2). 中断

遇到 I/O 指令后，挂起当前进程，待设备可用后发出中断，调用 handler 执行 I/O。以网卡为例:

-   网卡接受到数据包，通过中断通知内核
-   内核执行网卡驱动注册的 handler，拷贝数据包到内核缓冲区

中断 I/O 的缺点在于每个字符都支付一次中断的代价

#### (3). DMA

DMA 过程:

-   cpu 对 DMA 控制器发送 I/O 请求，随后挂起调度其他程序
-   DMA 执行指令，通知磁盘读数据到自己的缓冲区，再到内核缓冲区，磁盘传输完成后通知 DMA 控制器
-   DMA 向 cpu 发送一个中断，表明 I/O 已完成

DMA 的缺点:

-   DMA 占用内存总线时，cpu 无法访问内存 (但可以访问高速缓存)
-   DMA 执行指令慢于 cpu

因此使用 DMA 适用于大量的传输 (中断 I/O 性能很差)
