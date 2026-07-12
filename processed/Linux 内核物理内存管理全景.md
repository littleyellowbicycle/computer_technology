---
title: Linux 内核物理内存管理全景
source: raw_20260712_211329_cd3ec7a5
source_url: https://www.zhihu.com/question/332710035/answer/2770085703
created: '2026-07-12'
updated: '2026-07-12'
tags:
- 计算机基础
- Linux内核
- 内存管理
- NUMA架构
- 性能优化
status: processed
note_type: content
related:
- 增加内存为何无法提升CPU性能——Windows内存压缩的隐性开销
- Linux Slab 内存池设计与实现
- 内存为何不是二维寻址：DRAM 的多层抽象
- 一台主机TCP连接数量的真实限制
- 数据密集型应用系统设计 读书推荐
- 数据规模驱动的后端技术能力分层
- 让人惊叹的优雅代码：算法、数据结构与代码高尔夫
- GPU架构入门：从SIMD到SIMT的执行模型
---

# Linux 内核物理内存管理全景

## 核心结论
> Linux 内核通过 Node → Zone → page 的分层架构和 FLATMEM、DISCONTIGMEM、SPARSEMEM 三种内存模型协同管理物理内存。NUMA 架构下结合 [[水位线]] 机制、kswapd/kcompactd 后台进程以及 [[反向映射]] 等设计，在保证多进程隔离的同时兼顾了内存分配的扩展性与性能。整体设计在 [[虚拟内存]] 之上构筑了一套兼顾碎片治理、热插拔、冷热缓存与小对象分配的统一内存子体系。

## 详细内容
## 一、物理内存模型（CPU 视角）

内核以 [[页]]（4KB）为单位管理物理内存，每个物理页用 [[struct page]] 描述，并为其分配全局唯一的 PFN（Page Frame Number）索引。为应对不同的物理拓扑，内核演进出了三种内存模型：

- **[[FLATMEM]]**：物理内存连续时，用全局数组 `mem_map` 按 PFN 下标直接索引，page 与 PFN 互换只是数组偏移，最简单高效。
- **[[DISCONTIGMEM]]**：对内存空洞零容忍，按 node 划分连续段，每段内部仍采用 FLATMEM，避免为空洞分配多余 struct page。
- **[[SPARSEMEM]]**：以 section（128MB/512MB）为粒度对更小的连续块精细管理，支持 section 在线/离线切换，是当前主流模型，也是 [[内存热插拔]] 的基础。

## 二、物理内存架构（CPU 与内存的拓扑）

- **[[UMA]]**：所有 CPU 通过总线访问同一片内存，对称多处理器 SMP 结构简单但总线易成瓶颈。
- **[[NUMA]]**：内存按节点划分，CPU 访问本地节点最快，跨节点经 QPI 互联远程访问，延迟显著增加。内核将 UMA 视为「只有单节点的伪 NUMA」统一管理。

NUMA 下分配策略通过 `set_mempolicy` 设置：MPOL_BIND、MPOL_INTERLEAVE、MPOL_PREFERRED、MPOL_LOCAL（默认）。`numactl -H`、`numastat` 可观察节点距离与命中率，应用绑定 `--membind` 与 `--cpunodebind` 能显著降低跨节点访问开销。

## 三、节点管理：struct pglist_data

内核用 [[struct pglist_data]] 描述一个 NUMA 节点：
- 内核 2.4 之前用 pgdat_list 单链表串联，内核 2.4 之后改为全局数组 `node_data[]`，通过 `NODE_DATA(nid)` 索引。
- 关键字段：`node_mem_map`、`node_start_pfn`、`node_present_pages`（不含空洞）、`node_spanned_pages`（含空洞）、`kswapd` / `kcompactd` 内核线程指针。
- 节点状态通过 `node_states[]` 位图维护：N_POSSIBLE、N_ONLINE、N_NORMAL_MEMORY、N_HIGH_MEMORY、N_MEMORY、N_CPU。

## 四、内存区域：struct zone 与水位线

由于 ISA DMA 等硬件寻址限制，节点内的物理内存被进一步划分为：

- [[ZONE_DMA]]：供 16MB 内寻址的设备做 DMA。
- [[ZONE_DMA32]]：64 位系统下供 32 位设备 DMA。
- [[ZONE_NORMAL]]：常规可线性映射区。
- [[ZONE_HIGHMEM]]：32 位系统的高端内存，需要动态映射。
- [[ZONE_MOVABLE]]：逻辑区域，里面的页都可迁移，专门服务于防碎片与热插拔。
- [[ZONE_DEVICE]]：给非易失性内存使用。

struct zone 中的核心字段包括 `lock`、`zone_pgdat`、`zone_start_pfn`、`spanned_pages`、`present_pages`、`managed_pages`、伙伴系统核心的 `free_area[MAX_ORDER]`、统计信息 `vm_stat[]`，并通过 ZONE_PADDING 填充至不同 [[Cache Line]] 以降低 SMP 争用。

内存紧张时的应对路径：先后台回收（[[kswapd]] 回收不活跃页面），再内存规整（[[kcompactd]] 迁移可迁移页消除碎片），最后才会触发 [[OOM]]。回收对象分两类：[[文件页]]（回收代价低，可直接丢弃或回写）与 [[匿名页]]（必须通过 Swap 换出到磁盘）。`/proc/sys/vm/swappiness` 控制回收倾向。

回收触发点由三条 [[水位线]] 决定：
- WMARK_MIN：低于则同步直接回收 direct reclaim；
- WMARK_LOW：低于则唤醒 kswapd；
- WMARK_HIGH：kswapd 回收至此水位之上。

`min_free_kbytes = sqrt(lowmem_kbytes * 16)`，限定在 128~65536KB。`watermark_scale_factor`（默认 10，最大 3000）用于调节 WMARK_LOW 与 WMARK_MIN 的间距，应对突发流量避免直接回收抖动。`lowmem_reserve_ratio` 控制高位区域对低位区域的挤压比例。

## 五、物理页描述：struct page

`struct page` 是内核最热的「地段」，每个字段都精打细算，40 字节的体量靠大量 union 复用：

- 匿名页：`mapping` 低位置 1，指向 `anon_vma`；`index` 表示在 VMA 中的偏移。
- 文件页：`mapping` 低位置 0，指向 `address_space`（即 [[page cache]]）；`index` 表示在文件内的页偏移。
- 关键标志位：`PG_locked`、`PG_dirty`、`PG_writeback`、`PG_active`、`PG_lru`、`PG_referenced`、`PG_swapcache` 等，存于 `flags` 低位。
- 高位比特保存 section/node/zone 的定位信息，配合 `page_to_section`、`page_pgdat`、`page_zone` 反向定位。
- `_mapcount`、`_refcount` 记录映射数与引用次数；`lru` 链表头标识所属 LRU 链表。

### 匿名页反向映射

[[反向映射]] 用于在页面回收/迁移时快速找到所有引用该物理页的 VMA，链路为：

`struct page.mapping` → `anon_vma`（红黑树挂载所有 `anon_vma_chain`）→ `anon_vma_chain`（同时挂到 `vm_area_struct.anon_vma_chain` 链表）→ `vm_area_struct`。`do_anonymous_page` → `anon_vma_prepare` → `anon_vma_chain_link` → `page_add_new_anon_rmap` 完成整条链路建立。

### 五条 LRU 链表

为兼顾「最近最少」与「访问频率」并避免 Page Thrashing，内核把 page 分别挂到：
- `inactive_anon` / `active_anon`
- `inactive_file` / `active_file`
- `unevictable`（mlock 锁定页面）

文件页首次读入挂 inactive 头，再次访问升 active 尾；匿名页首次直接进 active 尾；回收优先级 inactive 尾 → inactive 头 → active 尾 → active 头。

## 六、冷热页：per_cpu_pageset

CPU 缓存中的页为热页，未缓存为冷页。早期内核用 `per_cpu_pageset` 数组的 hot/cold 两个 `per_cpu_pages` 分别管理，2.6.25 之后合并为单一 `lists[NR_PCP_LISTS]`，按迁移类型（可迁移/可回收/不可迁移）分别链表。`count`、`high`、`batch` 三个字段控制批量填充与回收。

## 七、复合页与 Slab

[[复合页]]（compound_page）：由若干连续物理页拼成大页（巨型页/hugepage），首页 `flags` 设 PG_head，其余尾页通过 `compound_head` 反指首页，`compound_order` 表示组合阶数，`compound_mapcount`、`compound_pincount` 保存在首页。优势是 TLB miss 减少、fork 时页表拷贝更快。

[[Slab]] 对象池：小对象（几十字节的 `anon_vma`、`vm_area_struct` 等）不适合按页申请，slab 从伙伴系统申请整页切成等大小块，`struct page` 内随之携带 slab 管理信息（`slab_list`、`slab_cache`、`freelist`、`s_mem`、`inuse`、`objects`、`frozen`）。

## 八、配套工具速查

- `cat /proc/zoneinfo`：查看 zone 水位线、LRU 数量与预留内存。
- `cat /proc/sys/vm/min_free_kbytes`、`/proc/sys/vm/watermark_scale_factor`、`/proc/sys/vm/swappiness`、`/proc/sys/vm/lowmem_reserve_ratio`：调参核心。
- `numactl -H/-s`、`numastat`：观察 NUMA 拓扑与命中率。
- `numactl --membind=... --cpunodebind=... <cmd>`：进程 NUMA 绑核绑内存。

## 九、整体层级回顾

```
Node（pglist_data） → Zone（struct zone，伙伴系统 free_area）→ Page（struct page，4KB）
        ↑
水资源：PFN 连续编号、Section 切分、WATERMARK 控制、kswapd 回收、kcompactd 规整、匿名反向映射、冷热页、Slab 小对象
```

## 相关笔记
- [[增加内存为何无法提升CPU性能——Windows内存压缩的隐性开销]]
- [[Linux Slab 内存池设计与实现]]
- [[内存为何不是二维寻址：DRAM 的多层抽象]]
- [[一台主机TCP连接数量的真实限制]]
- [[数据密集型应用系统设计 读书推荐]]
- [[数据规模驱动的后端技术能力分层]]
- [[让人惊叹的优雅代码：算法、数据结构与代码高尔夫]]
- [[GPU架构入门：从SIMD到SIMT的执行模型]]