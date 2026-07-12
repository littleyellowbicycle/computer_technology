---
title: Linux Slab 内存池设计与实现
source: raw_20260712_211200_558f13c2
source_url: https://www.zhihu.com/question/25527491/answer/2969738461
created: '2026-07-12'
updated: '2026-07-12'
tags:
- 性能优化
- 内存管理
- Linux 内核
- slab
- 对象池
status: processed
note_type: content
related:
- 增加内存为何无法提升CPU性能——Windows内存压缩的隐性开销
- 内存为何不是二维寻址：DRAM 的多层抽象
- 数据密集型应用系统设计 读书推荐
- Linux 内核物理内存管理全景
---

# Linux Slab 内存池设计与实现

## 核心结论
> slab 内存池是 Linux 内核在 [[伙伴系统]] 之上构建的小内存分配层，通过将连续物理页按对象尺寸切分为等大内存块并池化管理，专为高频创建/销毁的核心对象（如 task_struct、struct file、socket 等）优化分配路径。整体架构采用三级缓存（per-CPU 本地缓存 → NUMA 节点缓存 → 伙伴系统），结合 freelist、freepointer、red zone、POISON 等内存布局技巧，既减少内存碎片、避免缓存污染，又实现无锁快速分配。

## 详细内容
## 产生背景

[[伙伴系统]] 的最小分配单位是物理页 page，对于几十到几百字节的小内存请求（实际内核运行中最常见的场景），直接分配整页会造成严重的内存浪费。此外，伙伴系统的分配/释放链路较长（涉及 swap、compact 等），频繁调用还会污染 CPU L1Cache 的指令与数据缓存。

因此 Linux 引入了 [[slab 内存池]]：先向伙伴系统一次性申请一个或多个物理页，再按对象尺寸 object size 切分成若干等大的内存块，组成专属对象池。这是典型的 [[池化思想]]，与用户态的 Netty Recycler 对象池设计一脉相承。

## 三种实现：slab / slub / slob

- **slab**：最早由 Sun 公司的 Jeff Bonwick 在 Solaris 2.4 中设计，Linux 2.0（1996）引入；管理队列多、元数据臃肿、对 NUMA 支持繁杂。
- **slub**：Christoph Lameter 在 Linux 2.6.22（2007）引入，简化了 slab 的复杂队列，对多处理器和 NUMA 做了优化，弃用了着色机制，是当前服务器 Linux 的主流实现。
- **slob**：Linux 2.6.16（2006）引入，针对嵌入式小内存机器的精简实现。

内核对外 API 统一以 slab 命名，可通过配置切换内部实现；本文讨论的 slab 均指 slub。

## Slab 对象的内存布局

slab 在物理页中按对象的真实占用 `size` 切分内存块，`size` 的构成并不是简单的 object size，而是叠加了多重考量：

1. **内存对齐**：CPU 按 word size（64 位下为 8 字节）读写，未对齐访问需两次访存，因此 object size 需对齐到 word size，未对齐时填充 padding。
2. **red zone（红色警戒区）**：在对象内存两侧填充 0xbb，用于检测内存读写越界；若已有 padding 可复用，则只在一侧额外插入。
3. **freepointer**：利用对象未被使用时的内存空间存放指向下一个空闲对象的指针，避免额外分配串联链表的存储。
4. **SLAB_POISON**：用 0x6b 和 0xa5 填充对象内存表示特殊状态，但这会覆盖 freepointer，此时需为 freepointer 额外开辟一个 word size 空间。
5. **track 区域**：SLAB_STORE_USER 开启时，在对象内存中划出两块 `sizeof(struct track)`，分别记录对象的分配与释放信息。

> 一个 slab 在内核中由 [[复合页]] `struct page` 表示，相关字段（如 `freelist`、`frozen`、`inuse`、`objects`）描述 slab 的基本状态。Linux 5.17 起将 slab 元数据迁出到独立的 `struct slab`，目前 5.4 内核仍混在 page 内。

## 总体架构

slab 内存池的整体架构演进为三级缓存：

- **第一级：`struct kmem_cache_cpu`（per-CPU 本地缓存）**——每个 CPU 一份，无锁分配热点所在。包含 `freelist`（指向空闲对象）、`page`（被本 CPU 缓存的 slab）、`tid`（事务 ID，用于检测 CPU 迁移）、`partial`（备用的 partial slab 列表）。
- **第二级：`struct kmem_cache_node`（NUMA 节点缓存）**——每个 NUMA 节点一个仓库，含 `partial` 链表与上限控制字段。
- **第三级：[[伙伴系统]]**——终极供货商，按 `oo` 指定的内存页数批量申请 slab，内存紧张时降级为 `min`（至少容纳一个对象）。

`struct kmem_cache` 是整个对象池的管理中枢，关键字段包括：`flags`（HWCACHE_ALIGN / POISON / RED_ZONE / DMA / STORE_USER 等）、`size` / `object_size` / `offset`（freepointer 偏移）、`oo` / `max` / `min`、`cpu_partial`（控制 per-CPU partial 列表中空闲对象总数）、`min_partial`（控制 NUMA 节点 partial 列表中 slab 个数）。

## 内存分配路径

slab 分配分快速路径与慢速路径，按余量从近到远逐级回退：

1. **per-CPU 直接分配**：`kmem_cache_cpu->page` 的 slab 中若有空闲对象，freelist 出对象即可，无锁最快。
2. **per-CPU partial 列表**：本 CPU slab 耗尽时，从 `partial` 链表摘下一个 slab 提升为新的 page，分配后补充 `partial`。
3. **NUMA 节点缓存**：从 `kmem_cache_node->partial` 链表头部摘下 slab 提升为 page，并将后续 slab 转移至 per-CPU partial（最多 `cpu_partial / 2` 个）。
4. **向伙伴系统申请**：三级缓存全空时（多见于刚创建的 slab cache），按 `oo` 向伙伴系统批量申请 slab，失败则降级为 `min`。

## 内存释放路径

释放与分配互逆，按对象所在 slab 的位置和释放后 slab 状态变化分四种场景：

1. **对象在 per-CPU page**：fastpath，直接归还并更新 freelist。
2. **对象在 per-CPU partial**：归还后修正 page->freelist。
3. **full → partial**：被频繁使用的活跃 slab 释放后会晋升到 per-CPU partial 头部；若 `cpu_partial` 超限，则先批量转移到 NUMA 节点 partial，再插入——这是权衡"频繁分配场景下减少 slab 流动"的设计。
4. **partial → empty**：非活跃 slab 降级回 NUMA 节点 partial；若 `nr_partial` 已超 `min_partial`，则直接将整页释放回伙伴系统。

## 优势小结

- 利用 CPU 高速缓存：刚释放的"热"对象留在 cache，下次分配零成本命中。
- 相比伙伴系统多按整页分配，slab 切小对象粒度更细，显著减少对 cache line 和 TLB 的占用。
- 减少对伙伴系统的调用次数，避免 L1Cache 指令/数据缓存污染。
- 同一偏移对象分散在不同页中放置，降低对同一 cache line 的争用（false sharing）。

## 运维观测

- `cat /proc/slabinfo`：查看所有 slab cache 的 active_objs / num_objs / objsize / objperslab / pagesperslab 等。
- `cat /proc/meminfo`：查看 Slab/Reclaimable/Unreclaimable 等汇总内存占用。
- `slabtop`：动态观察占用内存最高的 slab cache，便于快速定位内存飙升的对象类型。

## 相关双链

- [[伙伴系统]]：slab 的底层物理页来源。
- [[NUMA]]：决定 slab 第二级缓存的拓扑结构。
- [[CPU 高速缓存]]：slab 设计中 cache line 对齐、热对象保留等优化的物理基础。
- [[内存对齐]]：slab 对象布局中 padding 计算的依据。
- [[复合页]]：跨页 slab 在内核中的表示方式。
- [[内存碎片]]：slab 解决的核心问题之一。
- [[池化思想]]：设计哲学层面的归纳。
- [[对象池]]：用户态对应实现（Netty Recycler）。

## 相关笔记
- [[增加内存为何无法提升CPU性能——Windows内存压缩的隐性开销]]
- [[内存为何不是二维寻址：DRAM 的多层抽象]]
- [[数据密集型应用系统设计 读书推荐]]
- [[Linux 内核物理内存管理全景]]