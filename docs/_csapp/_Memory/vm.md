### 虚拟内存的缓存机制

#### 缓存结构设计

由于磁盘本身读取很慢，而且磁盘以扇区为单位读取，导致 DRAM 缓存不命中开销很大，使得

-   虚拟页很大
-   使用全相联结构，并使用页表查询缓存映射，每个虚拟页对应一个页表条目 (PTE)

#### 页表工作机制

<img src="../img/vm_cache_structure.png" width=500>

任意时刻，一个虚拟页可能处于以下三个状态之一:

-   未分配: 没有数据与之关联，不占实际的磁盘空间 ($\sf 0 + null$)
-   未缓存: 已与某个文件关联，但未缓存 ($\sf 0 + VA$)
-   已缓存: 已与某个文件关联，且缓存在主存中 ($\sf 1 + PA$)

#### 地址翻译

虚拟地址相当于没有 $\rm tag$ 和 $\rm index$ 的概念，只有 $\rm page\ number + offset$。<br>
... here ... <br>
页面命中时，CPU 硬件的工作流程:

<img src="../img/pagehit.png" width=500>

<font class="u_nn">

-   ①. cpu 生成一个 VA，传给 MMU
-   ②. MMU 根据 VA 和 PTBR 生成 PTE 的地址，向 高速缓存/主存 发出请求
-   ③. 高速缓存/主存 向 MMU 返回 PTE 的内容
-   ④. MMU 构造物理地址，传回给 高速缓存/主存
-   ⑤. 高速主存/缓存 返回物理地址给 cpu

</font>

<img src="../img/pagefault.png" width=560>
