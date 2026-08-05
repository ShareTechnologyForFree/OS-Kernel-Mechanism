# Linux内核reclaim_empty_pte_page分析

> **分析对象**：原始 x86 patch 系列（Qi Zheng @ ByteDance，`[PATCH v3 0/9] / v4 0/11 synchronously scan and reclaim empty user PTE pages`，2024-11 ~ 2024-12）与 6.18 内核树（分支 `rept_6.18`，在 v6.18 基础上仅含一个 aarch64 适配提交）上的实际代码。
>
> **本文档范围**：在既有"patch 梳理 + aarch64 适配 + 调用链 + qemu 实测"分析的基础上，整合本课题全部技术讨论——机制背景与动机、mmu_gather 核心结构、x86 "IPI+同步"→"IPI+RCU" 演进、fast/slow 双路径与锁保护语义。
>
> **验证状态**：所有代码片段、Kconfig 项、调用路径均来自本项目源码。`mm/pt_reclaim.c` 的适配修改已经过实际交叉编译验证（`sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image` 成功生成 `arch/arm64/boot/Image`），并已在 qemu（virt 机型）完成适配前/适配后双内核运行对比验证（实测数据见第 8 章）。

---

[TOC]

# 1、机制背景与动机

## 1.1、问题现象：`MADV_DONTNEED` 之后 `VmPTE` 只增不减

jemalloc / tcmalloc 等用户态内存分配器在归还物理内存时，习惯对整块区域执行 `madvise(addr, len, MADV_DONTNEED)`，把已分配的物理页归还给内核。**但页表内存（PTE 页）从未被归还**：

- `MADV_DONTNEED` 只清空 PTE 表项（`pte` 置为 none）、释放数据页，**VMA 本身保留**，PTE 页继续留在页表层次中等待下次复用；
- 分配器反复"申请 → 释放 → 再申请"同一地址区间的典型工作负载下，PTE 页一旦建立就永久占用，`/proc/<pid>/status` 的 `VmPTE`（`mm->pgtables_bytes`）**只增不减**；
- 原始 patch 7/9 给出真实案例：VIRT 55T / RES 590G / **VmPTE 110G**——页表内存占到了物理内存的约 19%。

## 1.2、6.14 合入之前的页表释放现状（精确界定"缺失"）

需要澄清：**6.14 之前内核并非没有任何页表页释放机制**：

| 既有机制 | 路径 | 局限 |
|---|---|---|
| `free_pte_range()` / `pte_free_tlb` | `munmap`、`exit_mmap` | VMA 被销毁时才释放页表；DONTNEED 不清 VMA |
| `khugepaged` THP 折叠 | 后台线程，pmd 级 | 异步、不可控、仅 2MB 折叠时顺带回收 |
| `pmd_free_pte_page` 等 | 特殊场景 | 与用户态高频 DONTNEED 无关 |

**真正缺失的是**：在"`MADV_DONTNEED` 这类只清内容、保留 VMA、之后还会复用"的路径上，清空后的 PTE 页被**保留而不归还**。这就是 `VmPTE` 只增不减的根源。

## 1.3、作者、系列与合入历史

- **作者**：Qi Zheng（郑琦），字节跳动（`zhengqi.arch@bytedance.com`）；
- **系列演进**：RFC v1/v2（2024-08）→ PATCH v1/v2（2024-10）→ **v3 0/9（2024-11-14）** → **v4 0/11（2024-12-04）**，经 David Hildenbrand、Jann Horn、Hugh Dickins 等 review 后**合入 6.14**；
- **6.18 现状**：功能已合入并演进（详见 2.4），`CONFIG_PT_RECLAIM` 与 `CONFIG_ARCH_SUPPORTS_PT_RECLAIM` 为通用配置，x86_64 默认开启；
- 本项目适配分支 `rept_6.18` 在 v6.18 基础上仅含一个 aarch64 适配提交（第 6 章）。

## 1.4、作者的分步计划（答复 Andrew Morton）

1. **第一步（本系列）**：`madvise(MADV_DONTNEED)` 路径**同步**扫描回收空 PTE 页——"字节服务器上所有已知的页表内存暴涨案例都发生在 DONTNEED 路径"；且能先验证锁保护方案与基础设施，仅支持 x86_64；
2. **第二步**：`madvise(MADV_FREE)` 等场景**异步**回收——先给 VMA 打标记，加入全局链表，在内存回收流程中异步扫描；
3. **第三步**：回收所有"PTE 全部映射 zero page"的页（对内存 balloon 场景有益）。

## 1.5、为什么第一步选 `MADV_DONTNEED` 而不是 `MADV_FREE`

- `MADV_DONTNEED` 语义是"内容作废、立即丢弃"，与**同步摘表回收**天然匹配——清空后页表内容没有保留价值；
- `MADV_FREE` 语义是"懒回收"，内容在真正回收前仍可被读回（进程可撤销），不适合同步拆页表；
- 因此作者把 DONTNEED 的同步回收作为第一步，FREE 留给异步方案。

---

# 2、原始 patch 系列梳理（x86 架构）

## 2.1、系列总览

目标：**在 `madvise(MADV_DONTNEED)` 路径中同步扫描并回收"空的用户 PTE 页"**。设计核心：在 `zap_pte_range()` 中检测"整 PMD 范围内 512 个 PTE 全部为 none"的 PTE 页，通过 `mmu_gather` 将其释放回系统；通过新增的 `zap_details.reclaim_pt` 标志把回收范围严格限定在 `MADV_DONTNEED` 路径（`munmap`/`exit_mmap` 之外）。v3 共 9 个 patch、15 个文件、+397/-113 行，并新建 `mm/pt_reclaim.c`（71 行）。

## 2.2、Patch 清单与各文件变更说明

| Patch | 标题 | 修改文件 | 变更要点 |
|---|---|---|---|
| 0/9 | cover letter | - | 系列设计说明与演进记录 |
| 1/9 | mm: khugepaged: recheck pmd state in retract_page_tables() | `mm/khugepaged.c` | 重构 `retract_page_tables()`：持有 pmd 锁后引入 `check_pmd_state()` 重新校验 pmd 状态，不再使用 `pmd_same()`。原因：`new_folio` 的锁会阻塞 page fault，防止 PTE 被重新填充，只需重查 pmd 状态即可安全摘除空的 PTE 页（Jann Horn 建议） |
| 2/9 | mm: userfaultfd: recheck dst_pmd entry in move_pages_pte() | `mm/userfaultfd.c` | 引入 `is_pte_pages_stable()`，在 `move_present_pte`/`move_swap_pte`/`move_zeropage_pte` 中追加 `pmd_same(dst_pmdval, pmdp_get_lockless(dst_pmd))` 二次校验。原因：`dst_pte` 必须为 none，`pte_same()` 无法阻止 dst 的 PTE 页被并发回收 |
| 3/9 | mm: introduce zap_nonpresent_ptes() | `mm/memory.c` | 非 present PTE（swap/device/migration/uffd-wp marker/guard/hwpoison）处理从 `zap_pte_range()` 提取为独立函数，纯重构 |
| 4/9 | mm: introduce skip_none_ptes() | `mm/memory.c` | 批量跳过连续 none PTE，优化 `need_resched()`/`force_break`/逐条递增开销（David Hildenbrand 建议） |
| 5/9 | mm: introduce do_zap_pte_range() | `mm/memory.c` | 按 `pte_present()` 分流到 `zap_present_ptes()`/`zap_nonpresent_ptes()`，纯重构，为二次校验做准备 |
| 6/9 | mm: make zap_pte_range() handle full within-PMD range | `mm/memory.c` | 增加 `retry:` 标签与 `goto retry`，使函数能完整处理一个 PMD 覆盖的整段范围（中途被打断后重入），为"整页为空即可回收"提供前提 |
| 7/9 | mm: pgtable: try to reclaim empty PTE page in madvise(MADV_DONTNEED) | `include/linux/mm.h`、`mm/Kconfig`、`mm/Makefile`、`mm/internal.h`、`mm/madvise.c`、`mm/memory.c`、`mm/pt_reclaim.c`（新建） | **核心 patch**。新增 `zap_details.reclaim_pt`；新增 `CONFIG_PT_RECLAIM`（`default y`、`depends on ARCH_SUPPORTS_PT_RECLAIM && MMU && SMP`、`select MMU_GATHER_RCU_TABLE_FREE`）与 `CONFIG_ARCH_SUPPORTS_PT_RECLAIM`；`madvise_dontneed_single_vma()` 置 `reclaim_pt=true`；`zap_pte_range()` 集成 fast/slow 双路径回收 |
| 8/9 | x86: mm: free page table pages by RCU instead of semi RCU | `arch/x86/include/asm/tlb.h`、`arch/x86/kernel/paravirt.c`、`arch/x86/mm/pgtable.c`、`include/linux/mm_types.h`、`mm/mmu_gather.c` | x86 将 `MMU_GATHER_RCU_TABLE_FREE` 下"batch 用 RCU、单表用 IPI+同步"的 **semi-RCU 改为全部 RCU 释放**。原因：IPI 无法与 `pte_offset_map{_lock}()` 内的 `rcu_read_lock()` 同步，只有 RCU 延迟释放才能保证非 munmap/exit_mmap 路径安全（详见第 4 章） |
| 9/9 | x86: select ARCH_SUPPORTS_PT_RECLAIM if X86_64 | `arch/x86/Kconfig` | `select ARCH_SUPPORTS_PT_RECLAIM if X86_64`。理由：回收 PTE 页仅在 64 位系统上收益明显 |

## 2.3、patch 8/9 详解（semi-RCU → 全 RCU，为第 4 章铺垫）

- 修改前（semi-RCU）：批量攒批的表页走 `call_rcu` 延迟释放；但 batch 页分配失败时的**单表兜底**走 `tlb_remove_table_sync_one()`（IPI 广播 + 同步等待）后立即释放；
- 修改后（全 RCU）：`__tlb_remove_table_one()` 改为 `call_rcu(&ptdesc->pt_rcu_head, ...)`，借用 `ptdesc->pt_rcu_head`（与 `pt_list`/`pmd_huge_pte` union，页面回收时这些字段已闲置，无冲突）；
- 配套：`paravirt_tlb_remove_table`/`native_tlb_remove_table` 在 `CONFIG_PT_RECLAIM` 下改走 `tlb_remove_table`；`mmu_gather.c` 的 `__tlb_remove_table_one` 改为可被架构覆写（`#ifndef __tlb_remove_table_one`）。

## 2.4、6.18 mainline 中的演进

对比原始 v3 patch，6.18 中：

- `__tlb_remove_table_one` 的 RCU 版本已被**泛化到 generic 代码**（`#ifdef CONFIG_PT_RECLAIM` 内，[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L322-L344)），x86 不再需要自行定义；
- x86 的 `___pte_free_tlb()` 简化为 `paravirt_release_pte()` + `tlb_remove_ptdesc()`（[x86 pgtable.c](file:~/linux_dir/linux_old1/arch/x86/mm/pgtable.c#L21-L25)）；
- `skip_none_ptes()`/`do_zap_pte_range()` 合并为一个函数，并新增 `any_skipped`（遇到 uffd-wp marker 等被跳过条目时关闭回收）。

---

# 3、核心基础设施一：mmu_gather 结构体

## 3.1、设计目的：三条不变式

mmu_gather 是 mm 代码实现"正确且高效"的 TLB 失效 + 页面释放顺序的核心数据结构（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L31-L46) 顶部注释明确）：

```
 1) unhook page          （从页表中摘除映射）
 2) TLB invalidate page  （失效对应翻译）
 3) free page            （归还物理内存）
```

**绝不能在失效 TLB 之前释放页面**，否则其他 CPU 可能通过残留翻译观察到（甚至改写）已被复用的页。mmu_gather 的全部机制——范围累积、batch 攒批、`freed_tables` 标志、RCU 延迟释放——都是为了让这三步在正确顺序下尽可能批量化。

## 3.2、`struct mmu_gather` 定义

[asm-generic/tlb.h:321-378](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L321-L378)

```c
struct mmu_gather {
	struct mm_struct	*mm;                    // 目标进程

#ifdef CONFIG_MMU_GATHER_TABLE_FREE
	struct mmu_table_batch	*batch;                 // 表页 batch（RCU 延迟释放队列）
#endif

	unsigned long		start;                  // 待失效范围起点
	unsigned long		end;                    // 待失效范围终点
	unsigned int		fullmm : 1;             // 整 mm 退出（exit_mmap）
	unsigned int		need_flush_all : 1;     // 需要全 TLB 失效（如 x86-PAE 顶层表变更）
	unsigned int		freed_tables : 1;       // 已释放页表页（驱动架构全级别失效）
	unsigned int		delayed_rmap : 1;       // 有延迟的 rmap 删除待处理
	unsigned int		cleared_ptes : 1;       // 在哪些层级清过表项
	unsigned int		cleared_pmds : 1;
	unsigned int		cleared_puds : 1;
	unsigned int		cleared_p4ds : 1;
	unsigned int		vma_exec : 1;           // tlb_start_vma 记录 VMA 属性
	unsigned int		vma_huge : 1;
	unsigned int		vma_pfn  : 1;
	unsigned int		batch_count;            // 数据页 batch 数量（防 soft lockup）

#ifndef CONFIG_MMU_GATHER_NO_GATHER
	struct mmu_gather_batch *active;                // 数据页当前 batch
	struct mmu_gather_batch	local;                  // 栈上首个 batch（免分配）
	struct page		*__pages[MMU_GATHER_BUNDLE]; // 栈上兜底页槽
#ifdef CONFIG_MMU_GATHER_PAGE_SIZE
	unsigned int page_size;                         // 当前最小 unmap 粒度（本工程未启用）
#endif
#endif
};
```

## 3.3、成员含义与设计用途

| 成员 | 含义 | 设计用途 |
|---|---|---|
| `mm` | 目标进程 mm | 一切 mm 级记账与 TLB 操作的目标 |
| `batch` | 表页 batch（`struct mmu_table_batch`） | 把待 RCU 释放的页表页攒批，摊薄宽限期开销；懒分配，内存压力下可退化为单表路径 |
| `start`/`end` | 待失效范围 | 由 `__tlb_adjust_range()`（[tlb.h:382-388](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L382-L388)）随 zap 持续 `min/max` 累积；`__tlb_reset_range()`（[tlb.h:390-408](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L390-L408)）在 flush 后重置。`end==0` 表示尚无有效范围 |
| `fullmm` | 整 mm 退出 | 允许"忽略范围、只刷一次全 TLB"以及 ASID 切换延迟失效等优化 |
| `need_flush_all` | 需要全量失效 | 架构可置位强制 `flush_tlb_mm()` |
| `freed_tables` | 有页表页被释放 | **核心标志**。arm64 `tlb_flush` 据此 `last_level=false` → `vae1is` 全级别失效（含 walk cache）；x86 据此全范围失效 |
| `cleared_ptes/puds/...` | 各级清过表项 | 决定 `tlb_flush_mmu_tlbonly` 是否值得执行（全 0 则 return）；`cleared_pmds` 还参与 `tlb_get_unmap_shift()` 的粒度计算 |
| `delayed_rmap` | 延迟 rmap 删除 | 配合 `tlb_remove_tlb_entries` 在 TLB 失效后统一删 rmap |
| `vma_exec/huge/pfn` | VMA 属性快照 | 供 generic `tlb_flush` 构造 flush 用 vma |
| `active`/`local`/`__pages` | 数据页 batch | 数据页攒批后用 `free_pages_and_swap_cache` 批量归还；`local` 在栈上、免首次分配 |

## 3.4、辅助结构

```c
struct mmu_table_batch {          // tlb.h:204-213 —— 表页（页表页）延迟释放队列
#ifdef CONFIG_MMU_GATHER_RCU_TABLE_FREE
	struct rcu_head rcu;        // call_rcu 挂接点
#endif
	unsigned int nr;
	void *tables[];             // 攒批的 ptdesc 指针
};
#define MAX_TABLE_BATCH ((PAGE_SIZE - sizeof(struct mmu_table_batch)) / sizeof(void *))

struct mmu_gather_batch {         // tlb.h:271-279 —— 数据页释放队列
	struct mmu_gather_batch *next;
	unsigned int nr, max;
	struct encoded_page *encoded_pages[];
};
```

## 3.5、生命周期三阶段

1. **收集期**：`tlb_gather_mmu()`（[mmu_gather.c:441-444](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L441-L444)）→ `__tlb_gather_mmu`（[mmu_gather.c:407-431](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L407-L431)）：`fullmm=false`、`batch=NULL`、`__tlb_reset_range`、`inc_tlb_flush_pending`。zap 过程中：清 PTE 记录范围（`tlb_flush_pte_range`）、数据页入 page batch、表页入 table batch。
2. **失效 + 释放期**：`tlb_flush_mmu()`（[mmu_gather.c:401-405](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L401-L405)）**顺序固定：先 `tlb_flush_mmu_tlbonly`（架构 TLB 失效）→ 后 `tlb_flush_mmu_free`（表页 RCU + 数据页同步释放）**。
3. **RCU 期**：表页 batch 经 `call_rcu` 延迟；grace period 结束后回调 `pagetable_dtor_free` 真正归还（第 7 章 7.10）。

## 3.6、在 TLB shootdown 中的功能

- **范围合并**：多次 zap 的范围在 `start/end` 上取并集，避免逐页失效；
- **标志驱动失效语义**：`freed_tables=1` 通知架构"页表页被释放了"——x86 需 IPI 到**所有** CPU（含 lazy TLB 模式，防止投机访问曾是页表的页面，[tlb.c:1361-1372](file:~/linux_dir/linux_old1/arch/x86/mm/tlb.c#L1361-L1372)）；arm64 需全级别 TLBI；
- **失效与释放的顺序保证**：`tlb_flush_mmu` 内先失效后释放，是三条不变式的载体。

## 3.7、如何支撑页表回收（PT_RECLAIM 依赖链）

```c
PT_RECLAIM → select MMU_GATHER_RCU_TABLE_FREE
free_pte() → pte_free_tlb()
              ├─ tlb_flush_pmd_range()   → 范围并入 + cleared_pmds=1
              ├─ tlb->freed_tables = 1
              └─ __pte_free_tlb() → tlb_remove_ptdesc() → tlb_remove_table()
                                                            ├─ batch 路径 → call_rcu(&batch->rcu)
                                                            └─ 单表兜底   → call_rcu(&ptdesc->pt_rcu_head)
```

mmu_gather 为回收提供了"收集 → 统一失效 → 延迟释放"的完整框架，使回收空 PTE 页与常规 munmap 共用同一套安全语义。

---

# 4、核心基础设施二：x86 "IPI+同步" → "IPI+RCU" 演进

## 4.1、概念辨析：两条独立的链

| 链 | 做什么 | 手段 | PT_RECLAIM 是否改动 |
|---|---|---|---|
| **链① TLB 失效**（杀翻译） | 清除 TLB 中残留的虚拟地址→物理地址翻译 | x86：`flush_tlb_mm_range` → IPI 广播 + 同步等待 | **不改**（翻译即时生效，必须立刻全网清除） |
| **链② 页表页释放**（还内存） | 把页表页归还 buddy | 原 semi-RCU；PT_RECLAIM 后全 RCU | **改**（见 4.5/4.8） |

## 4.2、IPI+同步：实现流程与作用场景

**链①（TLB 失效）**，[x86 tlb.h:13-24](file:~/linux_dir/linux_old1/arch/x86/include/asm/tlb.h#L13-L24) → [flush_tlb_mm_range](file:~/linux_dir/linux_old1/arch/x86/mm/tlb.c#L1448-L1483) → `native_flush_tlb_multi`（[tlb.c:1346-1376](file:~/linux_dir/linux_old1/arch/x86/mm/tlb.c#L1346-L1376)）：

```c
if (info->freed_tables || mm_in_asid_transition(info->mm))
	on_each_cpu_mask(cpumask, flush_tlb_func, (void *)info, true);  // wait=true
else
	on_each_cpu_cond_mask(should_flush_tlb, flush_tlb_func, (void *)info, 1, cpumask);
```

- `freed_tables=1` 时 IPI 发往**所有** CPU（包括 lazy TLB），等全部执行完 `flush_tlb_func`（`invlpg`/`__flush_tlb_one`）才返回；
- 作用场景：一切解除映射但 TLB 仍可能残留翻译的路径——munmap、mprotect、THP 拆分/折叠、以及本机制的 PTE 页回收。

**链②（semi-RCU 的单表兜底）**，[mmu_gather.c:270-286](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L270-L286)：

```c
static void tlb_remove_table_smp_sync(void *arg) { /* Simply deliver the interrupt */ }

void tlb_remove_table_sync_one(void)
{
	smp_call_function(tlb_remove_table_smp_sync, NULL, 1);   // 空 handler，wait=1
}
```

- 空 handler 不干任何事，纯粹"确认你收到了中断"；`smp_call_function(..., wait=1)` **同步等待**所有 CPU 执行完；
- 作用场景：`tlb_remove_table` 分配 batch 页失败（`GFP_NOWAIT` 内存压力）时的单表兜底释放。

## 4.3、IPI+同步为什么对"关中断的遍历者"安全（示例）

> CPU A = 回收者，CPU B = 用 `gup_fast` 遍历 PTE 页（**全程关中断**）。

```
CPU A                              CPU B
                                    t0: local_irq_save()，开始遍历
t1: 发 TLB flush IPI / sync IPI
t2: 阻塞等待
                                    t3: 中断被关着 → IPI 挂起，进不来
                                    t4: 遍历结束，开中断 → 此刻才处理 IPI
                                        空 handler 执行，返回 ACK
t5: 收到 ACK → B 已退出遍历
t6: 释放 PTE 页                       安全 ✔
```

**"B 回复 ACK" ⟺ "B 已退出遍历"**——因为 IPI 是中断，B 关着中断就无法处理 IPI，ACK 只能在遍历结束后发出。

## 4.4、RCU 核心原理与在 x86 页表管理中的应用

- **读端**：`rcu_read_lock()/rcu_read_unlock()` 近乎零开销，无锁无原子；
- **写端**：`call_rcu(&head, cb)` **不阻塞**，把回调挂队；RCU 内核线程在确认"所有早于 call_rcu 进入的读端临界区都已退出"（宽限期结束）后才执行 cb；
- **安全保证**：cb 执行时，不可能再有读者引用待释放对象——读者要么已退出，要么在宽限期后才进入（看不到即将释放的对象）。

在页表管理中的落地（两处 `call_rcu`）：

```c
// batch 级：多个表页攒批后整体 RCU
call_rcu(&batch->rcu, tlb_remove_table_rcu);                       // mmu_gather.c:295
// 单表级：batch 分配失败兜底，借用 ptdesc 的 union 字段
call_rcu(&ptdesc->pt_rcu_head, __tlb_remove_table_one_rcu);        // mmu_gather.c:336
```

`ptdesc->pt_rcu_head` 复用 `struct ptdesc` 的 union（[mm_types.h:549-556](file:~/linux_dir/linux_old1/include/linux/mm_types.h#L549-L556)）：页面回收时其 PTE 已全清空，`pt_list`/`pmd_huge_pte` 均已闲置，借用为 `rcu_head` **无冲突、零额外内存**。

## 4.5、为什么 IPI 无法与 `pte_offset_map` 的 RCU 读端同步（竞态示例）

`pte_offset_map{_lock}()` 在 `___pte_offset_map` 内 `rcu_read_lock()`（[pgtable-generic.c:286](file:~/linux_dir/linux_old1/mm/pgtable-generic.c#L286)、[L353-357](file:~/linux_dir/linux_old1/mm/pgtable-generic.c#L353-L357) 注释："it does take rcu_read_lock()"）——**只持 RCU 读锁、不关中断**。

```
CPU A（回收者）                      CPU B（pte_offset_map 读者）
                                      t0: rcu_read_lock()，开始遍历（中断开着）
t1: 发 sync IPI（wait=1）
                                      t2: 中断开着 → IPI 立即处理，空 handler 秒回 ACK
t3: 收到 ACK ← 误以为 B 安全了
t4: 释放 PTE 页 → 归还 buddy
                                      页面可能立刻被重新分配
                                      t5: B 遍历其实没结束（刚被打断又继续）
                                      t6: 💥 读到的已是别人的数据 → UAF
```

**IPI 能打断 RCU 临界区**（中断开着），所以 "B 回了 ACK" ≠ "B 已退出临界区"。semi-RCU 注释自己也承认（[mmu_gather.c:278-285](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L278-L285)）：

> "This isn't an RCU grace period... It is however sufficient for software page-table walkers that **rely on IRQ disabling**."

——IPI+同步**只对关中断的遍历者成立**。现代内核大量遍历路径（page fault、uprobe、numa、huge_pmd_unshare 等）已改用 `pte_offset_map` 的 RCU 读端语义，不再关中断。因此 patch 8/9 把单表兜底也改成 RCU，实现**全 RCU 释放**，才能让 `madvise(MADV_DONTNEED)` 在任何时刻安全回收 PTE 页。

## 4.6、新方案：职责分工（保留 IPI 杀翻译 + RCU 延迟释放）

| 职责 | 手段 | 原因 |
|---|---|---|
| TLB 旧翻译失效 | **IPI + 同步**（保留） | 翻译即时生效，必须立刻全网清除；`freed_tables=1` 时强制发所有 CPU |
| 页表页释放 | **RCU**（全路径：batch + 单表兜底） | 页面有 RCU 读者，必须等宽限期 |

一句话：**IPI 负责"让世界看不见旧翻译"，RCU 负责"让世界不再引用这块内存"，各管一段，才凑成一个安全的回收。**

## 4.7、两种机制优缺点对比

| 维度 | IPI+同步 | RCU 延迟释放 |
|---|---|---|
| 释放延迟 | 低（同步等待后立即释放） | 高（等宽限期，内存滞留时间长） |
| 发起者阻塞 | **同步阻塞**，等所有 CPU ACK（多核放大） | **异步**，`call_rcu` 立即返回 |
| 对遍历者的要求 | 遍历必须关中断（否则竞态） | 只要求遍历在 RCU 临界区内 |
| 读者开销 | 无（靠关中断保护，但关中断有代价） | 极小（`rcu_read_lock/unlock` 近免费） |
| 内存压力下 | 立即归还，好 | 延迟归还，短暂放大 RSS |
| 可扩展性 | 差（每次释放都全网 IPI） | 好（攒批 + 无 CPU 间同步） |
| 极端场景兜底 | batch 分配失败可立即释放 | 兜底也走 RCU（依赖 `ptdesc->pt_rcu_head`） |

## 4.8、6.18 泛化后的代码形态

[mmu_gather.c:322-344](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L322-L344)：

```c
#ifdef CONFIG_PT_RECLAIM
static inline void __tlb_remove_table_one(void *table)
{
	struct ptdesc *ptdesc = table;
	call_rcu(&ptdesc->pt_rcu_head, __tlb_remove_table_one_rcu);   // 全 RCU：挂队列、不阻塞
}
#else
static inline void __tlb_remove_table_one(void *table)
{
	tlb_remove_table_sync_one();       // 旧 IPI+同步（semi-RCU 兜底）
	__tlb_remove_table(table);
}
#endif
```

注意方向：**不是"RCU 主路径 + IPI 兜底"，而是"IPI 兜底被 RCU 兜底替换"**。开启 PT_RECLAIM 后，页表页释放无论正常还是兜底全部走 RCU；IPI+同步只在链① TLB 失效中继续存在。

---

# 5、机制核心：识别、清除、同步

## 5.1、识别：`reclaim_pt` 标志与判定

- 置位点：`madvise_dontneed_single_vma()` 的 `zap_details.reclaim_pt = true`（[madvise.c:860-873](file:~/linux_dir/linux_old1/mm/madvise.c#L860-L873)）；
- 判定函数 `reclaim_pt_is_enabled()`（[pt_reclaim.c:8-12](file:~/linux_dir/linux_old1/mm/pt_reclaim.c#L8-L12)）：`details->reclaim_pt && (end - start >= PMD_SIZE)`——**单次释放范围必须整 2MB**（arm64 4K 页下 = 512 个 PTE）；
- 附加关闭条件：`any_skipped`（本 PMD 内遇到 uffd-wp marker 等被跳过条目 → 无法确认"全部 none" → 放弃回收，[memory.c:1857-1858](file:~/linux_dir/linux_old1/mm/memory.c#L1857-L1858)）。

## 5.2、fast path / slow path 双路径流程

两条路径都嵌在 [zap_pte_range()](file:~/linux_dir/linux_old1/mm/memory.c#L1821-L1911)，目标相同：**确认 PTE 页 512 项全空后，摘表交给 mmu_gather**。区别只在"怎么确认全空"。

**分流逻辑**：

```c
zap_pte_range() 进入，持 ptl，逐 PTE 清空
        │
        │  do-while 循环内：
        │    · need_resched()           → direct_reclaim = false; break   (L1850-1853)
        │    · force_break（batch 满）    → direct_reclaim = false; break  (L1859-1863)
        │    · 正常跑完 addr == end     → direct_reclaim 保持 true
        ▼
  [fast path] can_reclaim && direct_reclaim && addr == end ?   ← ptl 还在手上 (L1874-1875)
        │                     │是                            │否
        │           try_get_and_clear_pmd()                 跳过快路径
        ▼                     ▼
   (ptl 仍持有)     ptl 释放后，若 addr!=end → goto retry 重跑 (L1896-1901)
        │                      │
        ▼                      ▼
   direct_reclaim=true → free_pte()      direct_reclaim=false → try_to_free_pte()
        (L1904-1905)                       slow path (L1907)
```

**fast path（`try_get_and_clear_pmd` + `free_pte`）**，[pt_reclaim.c:14-26](file:~/linux_dir/linux_old1/mm/pt_reclaim.c#L14-L26)：

- 走它要满足：`can_reclaim_pt && direct_reclaim && addr == end`——**ptl 自始至终未释放**，整段 PMD 一次跑完；
- **为什么可以免复查**：清空每个 PTE 都是在持有 ptl 下完成的，而任何线程要往该 PTE 页填充新表项（page fault）**也必须拿 ptl**。ptl 一直在我们手里 → 别人没有机会塞回 present 项 → "512 项全 none"是**构造出来的必然**；
- 动作：`spin_trylock(pml)`（pmd 锁，抢不到→返回 false 降级 slow）→ `pmdval = pmdp_get_lockless(pmd)` → `pmd_clear(pmd)` 摘表 → 释放 pml。

**slow path（`try_to_free_pte`）**，[pt_reclaim.c:35-71](file:~/linux_dir/linux_old1/mm/pt_reclaim.c#L35-L71)：

- 走它的条件（任一）：中途 `need_resched`/`force_break`（ptl 被释放过）；`goto retry` 重跑过；fast path 抢 pmd 锁失败；
- **为什么必须重查**：ptl 中途释放过，窗口期别人可能塞回表项——第一遍"全 none"不代表现在还是；
- 动作：`pmd_lock(mm, pmd)` 先拿 pml → `pte_offset_map_rw_nolock`（校验 pmd 仍指向该页）→ `spin_lock_nested(ptl, SINGLE_DEPTH_NESTING)` 拿 ptl → **逐项复查 `PTRS_PER_PTE` 项**，任一项非 none 就放弃 → 全空才 `pmd_clear` → 释放锁 → `free_pte`。

| | Fast path | Slow path |
|---|---|---|
| 触发时机 | ptl 一直持有、一次跑完 | ptl 中途释放 / retry / pmd 锁抢不到 |
| 安全依据 | 持 ptl 期间别人无法塞表项 → 全空是必然 | 重新持 pml+ptl 逐个复查 512 项 |
| 关键操作 | `try_get_and_clear_pmd`（trylock + pmd_clear） | `pmd_lock` + `pte_offset_map_rw_nolock` + 全表扫描 |
| 失败处理 | trylock 失败 → 降级 slow | 发现非 none / pmd 已变 → 放弃回收（不误伤） |
| 收尾 | 统一 `free_pte()` → `pte_free_tlb` + `mm_dec_nr_ptes` | 相同 |

## 5.3、锁的作用：pml 与 ptl 两层保护

回收需要两个动作，各需要一把锁：

| 锁 | 保护的对象 | 管的是"什么" |
|---|---|---|
| **pml（pmd 锁）** | pmd 槽位本身 | 这个 pmd **指向哪张表**（摘表/换表/变大页）——结构层面 |
| **ptl（pte 锁）** | PTE 页的内容 | 这张表**每一项是啥**（填充/清空表项）——内容层面 |

- **不拿 ptl 会怎样**：page fault 线程往我们正在"验空/摘表"的页面塞新表项 → 释放的页被并发写入 → UAF。ptl 让"填表"与"验空/摘表"互斥；
- **不拿 pml 会怎样**：pmd 槽位是页表层次的入口，"谁装上去、谁摘下来"必须串行。THP 折叠/拆分、其他线程在 pmd 装大页或新 PTE 页都与我们的 `pmd_clear` 竞争——ptl 管不住 pmd 槽位，两层谁也替代不了谁；
- **锁顺序（防死锁）**：
  - fast path：先 `pte_offset_map_lock` 拿 **ptl**（[memory.c:1841](file:~/linux_dir/linux_old1/mm/memory.c#L1841)）→ 再 `spin_trylock` 拿 **pml**（trylock：抢不到说明 pmd 正被占，放弃免检资格降级，绝不忙等）；
  - slow path：先 `pmd_lock` 拿 **pml** → 再 `spin_lock_nested(ptl, SINGLE_DEPTH_NESTING)` 拿 **ptl**（嵌套标记告知锁校验器这是正确顺序，防误报死锁）；
- **摘表之后为何等 ptl 释放才 free_pte**：`pmd_clear` 摘表后旧页结构上不可达，`free_pte` 进 mmu_gather 走 RCU 延迟释放已不再需要锁；把释放放在 `pte_unmap_unlock`（[memory.c:1885](file:~/linux_dir/linux_old1/mm/memory.c#L1885)）之后，缩小临界区、避免持锁做慢操作。

## 5.4、清除：`free_pte()` 与 `pte_free_tlb` 宏

```c
void free_pte(struct mm_struct *mm, unsigned long addr, struct mmu_gather *tlb,
	      pmd_t pmdval)                                    // pt_reclaim.c L28-33
{
	pte_free_tlb(tlb, pmd_pgtable(pmdval), addr);   // ← 宏，展开三步
	mm_dec_nr_ptes(mm);                              // VmPTE - PTRS_PER_PTE*sizeof(pte_t)
}
```

`pte_free_tlb` 宏（[asm-generic/tlb.h:726-733](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L726-L733)）展开为三步：

```c
#define pte_free_tlb(tlb, ptep, address)			\
	do {							\
		tlb_flush_pmd_range(tlb, address, PAGE_SIZE);	\   // ① 范围并入 + cleared_pmds=1
		tlb->freed_tables = 1;				\   // ② 告知架构"有表页被释放"
		__pte_free_tlb(tlb, ptep, address);		\   // ③ 实际释放入表页 batch
	} while (0)
```

1. **① 范围记录**：`tlb_flush_pmd_range`（[tlb.h:606-611](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L606-L611)）→ `__tlb_adjust_range` 把被回收的 PMD 并入 `start/end` + `cleared_pmds=1`——保证后续 TLB 失效**覆盖被回收的 PMD**；
2. **② `freed_tables=1`**：arm64 `tlb_flush` 据此 `last_level=false` → `vae1is` 全级别失效（含 pmd 级 table 缓存 / walk cache）；为 0 则只做 leaf 级 `vale1is`；
3. **③ `__pte_free_tlb`**（arm64：[asm/tlb.h:75-81](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlb.h#L75-L81)，`tlb_remove_ptdesc` → `tlb_remove_table`）把 ptdesc 送入表页 batch；
4. **`mm_dec_nr_ptes`**（[mm.h:2907-2910](file:~/linux_dir/linux_old1/include/linux/mm.h#L2907-L2910)）：`/proc/<pid>/status` 的 `VmPTE` 减 4KB。

## 5.5、同步：`tlb_remove_table` 与 RCU 延迟释放

`tlb_remove_table`（[mmu_gather.c:362-379](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L362-L379)）：

- **batch 路径**：懒分配 `mmu_table_batch`，表页入 batch；batch 满或 `tlb_finish_mmu` 时 `tlb_table_flush`（[mmu_gather.c:351-360](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L351-L360)）→ `tlb_table_invalidate`（[mmu_gather.c:310-320](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L310-L320)，arm64 再执行一次 `tlb_flush_mmu_tlbonly` 确保表页缓存失效）→ `call_rcu(&batch->rcu, tlb_remove_table_rcu)`（[mmu_gather.c:288-296](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L288-L296)）；
- **单表兜底**：batch 分配失败 → `tlb_remove_table_one` → `__tlb_remove_table_one`——`CONFIG_PT_RECLAIM=y` 时 `call_rcu(&ptdesc->pt_rcu_head, ...)`（4.8 节）。

> **为什么必须 RCU 延迟释放**：`zap_pte_range` 的 `pte_offset_map_lock` 在 `rcu_read_lock` 临界区内映射 PTE 页；同步释放的话另一线程的读者可能仍映射着已归还（甚至被复用）的页。RCU 保证"读端临界区结束后页面才真正归还"。arm64 用硬件广播 TLBI（非 IPI 同步 walker），天然需要 `MMU_GATHER_RCU_TABLE_FREE`——这也是 Kconfig 依赖链 `PT_RECLAIM → select MMU_GATHER_RCU_TABLE_FREE` 的原因。

---

# 6、aarch64 架构适配修改分析

## 6.1、适配提交

- 内核版本：v6.18（`git log` 显示基于上游 v6.18 打了一个提交）；
- 提交：`rept_6.18` 分支，提交信息 "feat: enable reclaim empty pte page for aarch64."；
- 修改文件（最终状态）：2 个——`arch/arm64/Kconfig`（+1 行声明架构支持）、`mm/pt_reclaim.c`（+1 行 include / 去掉错误的 `#ifdef CONFIG_ARM64` 分支，恢复与 x86 完全一致的 `pte_free_tlb` 逻辑）。

## 6.2、修改点 1：arch/arm64/Kconfig

在现有 `select MMU_GATHER_RCU_TABLE_FREE`（第 249 行）之后新增：

```c
	select MMU_GATHER_RCU_TABLE_FREE
	select ARCH_SUPPORTS_PT_RECLAIM
```

与 `mm/Kconfig` 联动：

```c
config ARCH_SUPPORTS_PT_RECLAIM
	def_bool n

config PT_RECLAIM
	bool "reclaim empty user page table pages"
	default y
	depends on ARCH_SUPPORTS_PT_RECLAIM && MMU && SMP
	select MMU_GATHER_RCU_TABLE_FREE
```

本项目 `.config` 已确认：`CONFIG_ARCH_SUPPORTS_PT_RECLAIM=y`、`CONFIG_PT_RECLAIM=y`、`CONFIG_MMU_GATHER_RCU_TABLE_FREE=y`。

## 6.3、修改点 2：mm/pt_reclaim.c（include 修正——编译根因）

**现象**：用户实测 `error: implicit declaration of function '__pte_free_tlb'`（展开自 `pte_free_tlb` 宏第三步）。

**根因**：arm64 的 `__pte_free_tlb` 定义在 `arch/arm64/include/asm/tlb.h`（[asm/tlb.h:75-81](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlb.h#L75-L81)），**不在** `asm-generic/tlb.h`、也**不在** `arch/arm64/include/asm/pgalloc.h`。原 include 列表只含 `<asm-generic/tlb.h>` + `<asm/pgalloc.h>` → 宏展开时 `__pte_free_tlb` 不可见。

**修复**：`#include <asm-generic/tlb.h>` → `#include <asm/tlb.h>`（第 4 行保留 `<asm/pgalloc.h>`）：

- arm64：`asm/tlb.h` 第 17 行内部 `#include <asm-generic/tlb.h>`（include guard 保证不重复），随后提供 `__pte_free_tlb`，宏可完整展开；
- x86_64：[asm/tlb.h:8](file:~/linux_dir/linux_old1/arch/x86/include/asm/tlb.h#L8) 同样内部包含 generic，且 x86 的 `__pte_free_tlb` 定义在 `asm/pgalloc.h`（第 4 行仍保留），同样可展开；
- 其他架构：`<asm/tlb.h>` 是所有 MMU 架构必选的架构抽象头文件（`mm/memory.c:86`、`mm/mmu_gather.c:14` 等 16 个 mm 通用文件均如此包含），且当前只有 x86_64 与 arm64 两个架构 `select ARCH_SUPPORTS_PT_RECLAIM`，不受影响。

## 6.4、与 mainline 的关系

mainline 6.18 中 `free_pte()` 只有 `pte_free_tlb()` 一行，且只有 x86_64 通过 `select ARCH_SUPPORTS_PT_RECLAIM if X86_64` 开启。本适配在 mainline 基础上增加 arm64 声明，并将 include 修正为 `<asm/tlb.h>`，使 `pte_free_tlb` 在 arm64 上可用——**最终代码与 x86 完全一致，无任何架构分支**。

---

# 7、运行时调用链完整追踪（MADV_DONTNEED 全路径）

> 本章以"一次 `madvise(addr, 2MB+, MADV_DONTNEED)` 系统调用，整块匿名映射、PTE 全部已清空"为场景，从系统调用入口逐层追踪到物理页归还。所有行号均与 6.18 源码核对。标注"（架构差异）"处为 x86/arm64 不同点，其余为两架构共用路径。

## 7.0、调用链总览

```c
用户态 madvise(addr, len, MADV_DONTNEED)
  │ syscall
  ▼
[阶段1] SYSCALL_DEFINE3(madvise) ──────────────── mm/madvise.c:1985
  ▼
[阶段2] do_madvise() ──────────────────────────── mm/madvise.c:1962
  ├─ madvise_lock()                 VMA 读锁模式选择（MADVISE_VMA_READ_LOCK）
  ├─ madvise_init_tlb()             tlb_gather_mmu() —— mmu_gather 生命周期开始
  ├─ madvise_do_behavior()
  │    └─ madvise_walk_vmas()       VMA 遍历
  │         ├─ try_vma_read_lock()  lock_vma_under_rcu() 每 VMA 读锁
  │         └─ madvise_vma_behavior()
  │              └─ madvise_dontneed_free()
  │                   └─ madvise_dontneed_single_vma()  置 reclaim_pt=true
  │                        └─ zap_page_range_single_batched()
  │                             └─ unmap_single_vma() / unmap_page_range()
  │                                  └─ tlb_start_vma()
  │                                       └─ zap_p4d_range → zap_pud_range
  │                                            └─ zap_pmd_range
  │                                                 └─ zap_pte_range()      mm/memory.c:1821
  │                                                      ├─ reclaim_pt_is_enabled()   判定能否回收
  │                                                      ├─ do_zap_pte_range()        逐 PTE zap
  │                                                      │    ├─ pte_none 批量跳过
  │                                                      │    ├─ zap_present_ptes()    present 页
  │                                                      │    │    └─ tlb_remove_tlb_entries() 记录 TLB 范围
  │                                                      │    └─ zap_nonpresent_ptes() non-present（可能置 any_skipped）
  │                                                      ├─ try_get_and_clear_pmd()  fast path（ptl 持有中）
  │                                                      ├─ free_pte() / try_to_free_pte()   slow path
  │                                                      │    └─ pte_free_tlb() 宏展开（7.7 节）
  │                                                      └─ tlb_flush_mmu_tlbonly()  中途强制 flush
  └─ madvise_finish_tlb()           tlb_finish_mmu() —— mmu_gather 生命周期收尾
       └─ tlb_flush_mmu()
            ├─ tlb_flush_mmu_tlbonly()  架构 TLB 失效（arm64: __flush_tlb_range）
            └─ tlb_flush_mmu_free()
                 ├─ tlb_table_flush()   表页 batch → RCU 延迟释放
                 └─ tlb_batch_pages_flush()  数据页 batch 同步释放
                      └─ [RCU 回调] __tlb_remove_table_free()
                           └─ pagetable_dtor_free()   PTE 页归还 buddy
```

## 7.1、阶段 1：系统调用入口

```c
SYSCALL_DEFINE3(madvise, unsigned long, start, size_t, len_in, int, behavior)  // madvise.c:1985
{
	return do_madvise(current->mm, start, len_in, behavior);               // madvise.c:1987
}
```

- 仅做参数透传，`current->mm` 作为目标 mm；
- `behavior = MADV_DONTNEED` 是唯一能触发 PTE 页回收的行为。

## 7.2、阶段 2：do_madvise() —— 锁模式选择与 mmu_gather 生命周期

```c
int do_madvise(struct mm_struct *mm, unsigned long start, size_t len_in, int behavior)  // madvise.c:1962
{
	struct mmu_gather tlb;                    // 栈上 mmu_gather
	struct madvise_behavior madv_behavior = { .mm = mm, .behavior = behavior, .tlb = &tlb };

	if (madvise_should_skip(start, len_in, behavior, &error))  // 校验（页对齐、范围合法性）
		return error;
	error = madvise_lock(&madv_behavior);      // ①
	if (error) return error;
	madvise_init_tlb(&madv_behavior);         // ② tlb_gather_mmu()
	error = madvise_do_behavior(start, len_in, &madv_behavior);  // ③ 核心
	madvise_finish_tlb(&madv_behavior);       // ④ tlb_finish_mmu()
	madvise_unlock(&madv_behavior);           // ⑤
	return error;
}
```

**① 锁模式**：`get_lock_mode()`（[madvise.c](file:~/linux_dir/linux_old1/mm/madvise.c#L1699-L1722)）对 `MADV_DONTNEED` 返回 `MADVISE_VMA_READ_LOCK`。`madvise_lock()`（[madvise.c](file:~/linux_dir/linux_old1/mm/madvise.c#L1724-L1747)）对该模式**不取 mmap 锁**（注释：每 VMA 在 `madvise_walk_vmas()` 中单独获取）。这是与 `munmap`（写锁）的本质区别——回收全程只有 VMA 读锁，页表并发安全必须依赖 pmd 锁/pte 锁 + RCU（见 5.2/5.3）。

**② `madvise_init_tlb`**（[madvise.c](file:~/linux_dir/linux_old1/mm/madvise.c#L1781-L1785)）：`madvise_batch_tlb_flush(MADV_DONTNEED)` 为 true → `tlb_gather_mmu(tlb, mm)`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L441-L444)）→ `__tlb_gather_mmu`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L407-L431)）：

- `tlb->fullmm = false`（非整体退出）
- `tlb->batch = NULL`（表页 batch 待懒分配）、`tlb->active = &tlb->local`（数据页 batch 用栈上 `local`）
- **`__tlb_reset_range(tlb)`**（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L390-L408)）：`tlb->start = TASK_SIZE; tlb->end = 0;` 以及 `freed_tables/cleared_*` 全部清零 —— 这是后续"范围是否有效"的判据（`tlb->end == 0` 表示尚无任何 zap 记录）
- `inc_tlb_flush_pending(mm)`

**④ `madvise_finish_tlb`**（[madvise.c](file:~/linux_dir/linux_old1/mm/madvise.c#L1787-L1791)）：`tlb_finish_mmu(tlb)`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L469-L503)），见 7.9 节。

**③ `madvise_do_behavior`**（[madvise.c](file:~/linux_dir/linux_old1/mm/madvise.c#L1865-L1888)）：`blk_start_plug`（合并块设备 I/O）→ `madvise_walk_vmas()` → `blk_finish_plug`。

## 7.3、阶段 3：VMA 遍历与 per-VMA 读锁

```c
static int madvise_walk_vmas(struct madvise_behavior *madv_behavior)  // madvise.c:1618
{
	...
	if (madv_behavior->lock_mode == MADVISE_VMA_READ_LOCK &&
	    try_vma_read_lock(madv_behavior)) {          // ① 单 VMA 读锁路径
		error = madvise_vma_behavior(madv_behavior);
		vma_end_read(madv_behavior->vma);
		return error;
	}
	// 否则走 mmap 读锁 + find_vma_prev 遍历
}
```

**① `try_vma_read_lock`**（[madvise.c](file:~/linux_dir/linux_old1/mm/madvise.c#L1583-L1607)）→ `lock_vma_under_rcu(mm, range->start)`（[mmap_lock.c](file:~/linux_dir/linux_old1/mm/mmap_lock.c#L223-L258)）：
- `rcu_read_lock()` 下 `mas_walk(&mas)` 在 maple tree 中定位 VMA；
- `vma_start_read(mm, vma)`（VMA 读写锁的读端）成功 → 拿到**稳定的单 VMA**；失败（VMA 被隔离/替换）返回 NULL；
- 要求范围落在单个 VMA 内、且无 userfaultfd，否则回退 `mmap_read_lock` 全局读锁。

**关键点**：`lock_vma_under_rcu` 返回的 VMA 读锁只保护 VMA 结构本身（vma 范围/标志），**不保护页表内容**。PTE 回收的并发安全完全由 `zap_pte_range()` 内部的 pmd/pte 锁与 RCU 页表映射保证。

## 7.4、阶段 4：zap 入口 —— mmu_notifier 与四级页表下钻

```c
static void madvise_dontneed_single_vma(struct madvise_behavior *madv_behavior)  // madvise.c:860
{
	struct zap_details details = {
		.reclaim_pt = true,      // ★ 本机制开关
		.even_cows = true,
	};
	zap_page_range_single_batched(madv_behavior->tlb, madv_behavior->vma,
				      range->start, range->end - range->start, &details);
}
```

- `zap_page_range_single_batched`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L2124-L2153)）：
  1. `mmu_notifier_range_init` + `mmu_notifier_invalidate_range_start`（通知 KVM 等外部映射者）；
  2. `unmap_single_vma(tlb, vma, address, end, details, false)`；
  3. `mmu_notifier_invalidate_range_end`。
- `unmap_single_vma`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L2024-L2063)）：非 hugetlb → `unmap_page_range`。
- `unmap_page_range`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L2003-L2021)）：
  - `tlb_start_vma(tlb, vma)`（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L548-L557)）：记录 `vma_exec/vma_huge/vma_pfn` 标志 + `flush_cache_range`；
  - `pgd_offset` → `zap_p4d_range`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1984-L2001)）→ `zap_pud_range`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1954)）→ `zap_pmd_range`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1912-L1952)）逐级下钻，每级 `xxx_none_or_clear_bad()` 快速跳过空项；
  - `zap_pmd_range` 对 THP/swap pmd 走 `__split_huge_pmd`/`zap_huge_pmd`；普通 pmd → `zap_pte_range(tlb, vma, pmd, addr, next, details)`；每轮 `cond_resched()`；
  - `tlb_end_vma(tlb, vma)`（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L559-L571)）：非 fullmm 且非 MERGE_VMAS 时执行 `tlb_flush_mmu_tlbonly()`（VMA 边界处提前 flush）。

## 7.5、阶段 5：zap_pte_range() 核心循环

```c
static unsigned long zap_pte_range(struct mmu_gather *tlb,
		struct vm_area_struct *vma, pmd_t *pmd,
		unsigned long addr, unsigned long end,
		struct zap_details *details)                       // memory.c:1821
{
	bool force_flush = false, force_break = false;
	spinlock_t *ptl;
	pte_t *start_pte;
	pmd_t pmdval;
	unsigned long start = addr;
	bool can_reclaim_pt = reclaim_pt_is_enabled(start, end, details);   // ①
	bool direct_reclaim = true;

retry:
	tlb_change_page_size(tlb, PAGE_SIZE);                   // ②
	start_pte = pte = pte_offset_map_lock(mm, pmd, addr, &ptl);   // ③
	if (!pte) return addr;

	flush_tlb_batched_pending(mm);
	arch_enter_lazy_mmu_mode();
	do {
		bool any_skipped = false;
		if (need_resched()) { direct_reclaim = false; break; }

		nr = do_zap_pte_range(tlb, vma, pte, addr, end, details, rss,
				      &force_flush, &force_break, &any_skipped);  // ④
		if (any_skipped) can_reclaim_pt = false;           // ⑤
		if (unlikely(force_break)) { addr += nr*PAGE_SIZE; direct_reclaim = false; break; }
	} while (pte += nr, addr += PAGE_SIZE*nr, addr != end);

	if (can_reclaim_pt && direct_reclaim && addr == end)      // ⑥ fast path
		direct_reclaim = try_get_and_clear_pmd(mm, pmd, &pmdval);

	add_mm_rss_vec(mm, rss);
	arch_leave_lazy_mmu_mode();

	if (force_flush) {                                        // ⑦ 中途 flush
		tlb_flush_mmu_tlbonly(tlb);
		tlb_flush_rmaps(tlb, vma);
	}
	pte_unmap_unlock(start_pte, ptl);                          // ⑧ 释放 ptl + rcu
	if (force_flush) tlb_flush_mmu(tlb);

	if (addr != end) { cond_resched(); ... goto retry; }       // ⑨ 重入

	if (can_reclaim_pt) {                                      // ⑩ 回收决策
		if (direct_reclaim) free_pte(mm, start, tlb, pmdval);
		else               try_to_free_pte(mm, pmd, start, tlb);
	}
	return addr;
}
```

① `reclaim_pt_is_enabled(start, end, details)`（[pt_reclaim.c](file:~/linux_dir/linux_old1/mm/pt_reclaim.c#L8-L12)）：`details && details->reclaim_pt && (end - start >= PMD_SIZE)`。**只有范围整 2MB（arm64 4K 页）时才可能回收**。
② `tlb_change_page_size`：本项目 arm64 未启用 `CONFIG_MMU_GATHER_PAGE_SIZE`，为 no-op。
③ `pte_offset_map_lock`：`rcu_read_lock` + 映射 PTE 页 + 取 pte 锁 `ptl`（[pgtable-generic](file:~/linux_dir/linux_old1/mm/pgtable-generic.c#L17) 体系）。`rcu_read_lock` 区间的意义：PTE 页通过 RCU 延迟释放（7.7/7.8），读者在该区间内页面不会被释放。
④ `do_zap_pte_range`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1785-L1819)）：
   - **none 批量跳过**：开头对连续 `pte_none` 条目直接跳过（`skip_none_ptes` 的 6.18 合并版），并累计计数；
   - **present** → `zap_present_ptes` → `zap_present_folio_ptes`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1618-L1661)）：`clear_full_ptes`（anon）/`get_and_clear_full_ptes`（file）清空 PTE；`rss[MM_ANONPAGES] -= nr`；`tlb_remove_tlb_entries(tlb, pte, nr, addr)`（→ `tlb_flush_pte_range`：`__tlb_adjust_range` 并入范围 + `cleared_ptes=1`）；`__tlb_remove_folio_pages` 把数据页入 batch，**batch 满返回 true → `force_flush=force_break=true`**；
   - **non-present** → `zap_nonpresent_ptes`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1715-L1783)）：swap/device/migration/uffd-wp marker/guard/hwpoison 各自处理；其中 uffd-wp marker 若被跳过或需要保留，置 `*any_skipped = true`。
⑤ `any_skipped` → `can_reclaim_pt = false`：本 PMD 内存在无法确认"全部 none"的条目，放弃回收（安全优先）。
⑥ **fast path**（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L1873-L1875)）：`can_reclaim_pt && direct_reclaim && addr == end`（整段 zap 完成、未被 resched/force_break 打断）→ `try_get_and_clear_pmd(mm, pmd, &pmdval)`。注意此时 **ptl 仍被持有**（`pte_unmap_unlock` 在 ⑧）——ptl 持有期间并发 page fault 无法向该 PTE 页填充条目，这是 fast path 只做"假设性检查"仍安全的前提。
⑦ `force_flush`（batch 满或 dirty folio 延迟 rmap）：先 `tlb_flush_mmu_tlbonly`（失效已记录范围）再 `tlb_flush_rmaps`，**仍持 ptl**。
⑧ `pte_unmap_unlock`：释放 ptl、退出 RCU 区。
⑨ `addr != end`：本 PMD 未处理完（force_break），`cond_resched()` 后 `goto retry`（重新持锁）。
⑩ **回收决策**：`can_reclaim_pt` 为真才回收；fast path 成功则直接 `free_pte`，否则 `try_to_free_pte` 兜底。

## 7.6、阶段 6：空 PTE 页检测与摘除（fast / slow 双路径）

**[pt_reclaim.c](file:~/linux_dir/linux_old1/mm/pt_reclaim.c)（现已与 x86 完全一致）**

```c
/* fast path：在 ptl 仍持有时，尝试抢占 pmd 锁并直接摘除 */
bool try_get_and_clear_pmd(struct mm_struct *mm, pmd_t *pmd, pmd_t *pmdval)   // L14-26
{
	spinlock_t *pml = pmd_lockptr(mm, pmd);

	if (!spin_trylock(pml))     // 抢不到 pmd 锁就放弃 → 由 slow path 复查
		return false;

	*pmdval = pmdp_get_lockless(pmd);   // 读回 pmd 值（保留给 free_pte 用）
	pmd_clear(pmd);                     // 摘除 PTE 页
	spin_unlock(pml);
	return true;
}

/* slow path：持 pmd 锁 + pte 锁，逐个复查 PTRS_PER_PTE 项，确认全空才摘除 */
void try_to_free_pte(struct mm_struct *mm, pmd_t *pmd, unsigned long addr,
		     struct mmu_gather *tlb)                                  // L35-71
{
	pml = pmd_lock(mm, pmd);
	start_pte = pte_offset_map_rw_nolock(mm, pmd, addr, &pmdval, &ptl);
	if (!start_pte) goto out_ptl;
	if (ptl != pml) spin_lock_nested(ptl, SINGLE_DEPTH_NESTING);

	for (i = 0, pte = start_pte; i < PTRS_PER_PTE; i++, pte++) {
		if (!pte_none(ptep_get(pte)))
			goto out_ptl;      // 任一项非 none → 不回收
	}
	pte_unmap(start_pte);
	pmd_clear(pmd);                    // 确认全空后摘除
	if (ptl != pml) spin_unlock(ptl);
	spin_unlock(pml);
	free_pte(mm, addr, tlb, pmdval);   // 与 fast path 汇合
	return;
out_ptl:
	...
}
```

- **fast path 成立条件**：① `reclaim_pt_is_enabled`（范围≥PMD_SIZE）；② 全程未被 `need_resched`/`force_break` 打断；③ 未被 any_skipped 关闭；④ `spin_trylock(pml)` 成功。它"相信" 7.5 ④ 中 none 计数与 zap 已覆盖全部 512 项。
- **slow path 触发条件**：fast path 失败（pmd 锁被占）或中途被打断（`direct_reclaim=false`）。它重新持锁、**逐一验证**，杜绝并发线程在回收窗口内重新填充 PTE 的竞态。

## 7.7、阶段 7：free_pte() 与 pte_free_tlb 宏展开（x86/arm64 完全一致）

```c
void free_pte(struct mm_struct *mm, unsigned long addr, struct mmu_gather *tlb,
	      pmd_t pmdval)                                     // pt_reclaim.c L28-33
{
	pte_free_tlb(tlb, pmd_pgtable(pmdval), addr);   // ← 宏，展开见下
	mm_dec_nr_ptes(mm);                              // VmPTE - PTRS_PER_PTE*sizeof(pte_t)
}
```

`pte_free_tlb` 宏（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L726-L733)）展开为**三步**（加上 `free_pte()` 内的显式记账，共四件事），这是回收语义完整性的关键：

```c
#define pte_free_tlb(tlb, ptep, address)			\
	do {							\
		tlb_flush_pmd_range(tlb, address, PAGE_SIZE);	\   // ①
		tlb->freed_tables = 1;				\   // ②
		__pte_free_tlb(tlb, ptep, address);		\   // ③
	} while (0)
```

1. **① 范围记录**：`tlb_flush_pmd_range`（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L606-L611)）→ `__tlb_adjust_range`（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L382-L388)，`tlb->start = min(...)`、`tlb->end = max(...)`）并把 `addr` 按 PMD 粒度并入，同时置 `tlb->cleared_pmds = 1`。→ 保证后续 TLB 失效范围**覆盖被回收的 PMD**。
2. **② `freed_tables = 1`**：告知架构 TLB 失效时"有页表页被释放"。arm64 `tlb_flush()` 据此选择 `vae1is`（全级别失效，含 pmd 级 table 缓存/walk cache）；若为 0 则只做 leaf 级 `vale1is`。
3. **③ 实际释放**：`__pte_free_tlb`（arm64：[asm/tlb.h](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlb.h#L75-L81)，`tlb_remove_ptdesc` → `tlb_remove_table`）把 ptdesc 送入表页 batch（7.8 节）。
4. **`mm_dec_nr_ptes`**：`atomic_long_sub(PTRS_PER_PTE * sizeof(pte_t), &mm->pgtables_bytes)`（[mm.h](file:~/linux_dir/linux_old1/include/linux/mm.h#L2907-L2910)）——`/proc/<pid>/status` 的 `VmPTE` 减 4KB。

> 对比：`munmap` 正常路径的 `free_pte_range()`（[memory.c](file:~/linux_dir/linux_old1/mm/memory.c#L189-L215)）同样走 `pte_free_tlb`。PT_RECLAIM 的 `free_pte()` 与之一致，因此回收路径与内核常规路径的 TLB 语义完全相同。

## 7.8、阶段 8：tlb_remove_table —— 表页入 batch 与 RCU 延迟释放

`__pte_free_tlb` → `tlb_remove_ptdesc`（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L506-L509)）→ `tlb_remove_table(tlb, pt)`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L362-L379)）：

```c
void tlb_remove_table(struct mmu_gather *tlb, void *table)
{
	if (*batch == NULL) {                     // 首次：懒分配 batch 页
		*batch = __get_free_page(GFP_NOWAIT);
		if (*batch == NULL) {             // 分配失败 → 单表立即处理
			tlb_table_invalidate(tlb);
			tlb_remove_table_one(table);   // → call_rcu 单表延迟释放
			return;
		}
		(*batch)->nr = 0;
	}
	(*batch)->tables[(*batch)->nr++] = table;  // 入 batch
	if ((*batch)->nr == MAX_TABLE_BATCH)
		tlb_table_flush(tlb);              // batch 满 → 立即 flush+RCU
}
```

- **batch 满或 tlb_finish_mmu 时**：`tlb_table_flush`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L351-L360)）→ `tlb_table_invalidate`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L310-L320)，arm64 使用 generic 默认 `tlb_needs_table_invalidate()==true`，即再执行一次 `tlb_flush_mmu_tlbonly`，确保表页缓存已失效）→ `tlb_remove_table_free` → `call_rcu(&batch->rcu, tlb_remove_table_rcu)`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L288-L296)）。
- **单表路径**：`tlb_remove_table_one` → `__tlb_remove_table_one`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L322-L344)）。在 `CONFIG_PT_RECLAIM=y` 时（本项目两架构均为 y）使用 `call_rcu(&ptdesc->pt_rcu_head, __tlb_remove_table_one_rcu)`——**与 patch 8/9 泛化后的 generic 逻辑一致**（第 4 章 4.8）。

> 为什么必须 RCU 延迟释放（patch 8/9 的核心）：`zap_pte_range` 的 `pte_offset_map{_lock}()`（7.5 ③）在 `rcu_read_lock` 临界区内映射 PTE 页；若同步释放，另一线程的读者可能仍映射着已归还（甚至被复用作他途）的页。RCU 延迟释放保证"读端临界区结束后页面才真正归还"。arm64 使用 IPI 广播 TLBI（非 IPI 同步 walker），因此天然需要 `MMU_GATHER_RCU_TABLE_FREE`——这也是 Kconfig 依赖链中 `PT_RECLAIM → select MMU_GATHER_RCU_TABLE_FREE` 的原因。

## 7.9、阶段 9：tlb_finish_mmu —— 收尾（并发兜底 + 最终 TLB 失效 + 释放）

`madvise_finish_tlb` → `tlb_finish_mmu`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L469-L503)）：

```c
void tlb_finish_mmu(struct mmu_gather *tlb)
{
	if (mm_tlb_flush_nested(tlb->mm)) {       // ① 并发 batched flush 兜底
		tlb->fullmm = 1;                   //    源码注释明言 aarch64 场景
		__tlb_reset_range(tlb);
		tlb->freed_tables = 1;
	}
	tlb_flush_mmu(tlb);                        // ②
	tlb_batch_list_free(tlb);                  // ③
	dec_tlb_flush_pending(tlb->mm);
}
```

① `mm_tlb_flush_nested`：若检测到并行线程在同一范围做 deferred TLB flush，强制 `fullmm=1` + `freed_tables=1` → 最终 `flush_tlb_mm()` 全 TLB 失效（注释明确：aarch64 下 fullmm 可避免多 CPU 同时刷 TLBI 广播）。这是并发兜底，单线程回收路径不触发。
② `tlb_flush_mmu`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L401-L405)）顺序固定：**先失效、后释放**：
   - `tlb_flush_mmu_tlbonly`（[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L480-L492)）：
     - 判据：`freed_tables || cleared_ptes || cleared_pmds || cleared_puds || cleared_p4ds` 任一非 0 才继续，否则直接 return；
     - 调架构 `tlb_flush(tlb)` → **arm64 语义**（[asm/tlb.h](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlb.h#L53-L73)）：
       - `last_level = !tlb->freed_tables`：本机制回收路径中 `free_pte` 已置 `freed_tables=1` → `last_level=false` → `vae1is` 全级别失效；
       - `stride = tlb_get_unmap_size(tlb)`（`cleared_pmds` 优先于 `cleared_ptes` 时返回 `PMD_SHIFT`）；
       - `tlb_level = tlb_get_level(tlb)`：`freed_tables=1` → `TLBI_TTL_UNKNOWN`（TLT hint 未知）；
       - `__flush_tlb_range(&vma, start, end, stride, last_level, tlb_level)`（[tlbflush.h](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlbflush.h#L465-L473)）→ `__flush_tlb_range_nosync`（[tlbflush.h](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlbflush.h#L436-L463)）：`pages = (end - start) >> PAGE_SHIFT`；`__flush_tlb_range_limit_excess`（[tlbflush.h](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlbflush.h#L419-L434)）判定超限 → `flush_tlb_mm()` 全量回退；否则 `last_level=false` → `__flush_tlb_range_op(vae1is, ...)`（[tlbflush.h](file:~/linux_dir/linux_old1/arch/arm64/include/asm/tlbflush.h#L379-L414)）发射 TLBI 指令 + `dsb(ish)`；
     - **x86 对照**：`tlb_flush`（[x86 tlb.h](file:~/linux_dir/linux_old1/arch/x86/include/asm/tlb.h#L13-L24)）→ `flush_tlb_mm_range(mm, start, end, stride_shift, freed_tables)`，`freed_tables=1` 时全范围失效。
     - 之后 `__tlb_reset_range(tlb)` 清空 start/end 与 cleared_*（下一次 VMA 用新范围）。
   - `tlb_flush_mmu_free`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L393-L399)）：
     - `tlb_table_flush`：表页 batch → RCU（7.8 节）；
     - `tlb_batch_pages_flush`（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L144-L151)）：数据页 batch → `__tlb_batch_free_encoded_pages` → `free_pages_and_swap_cache` 同步归还。
     ③ 释放 batch 链表页。

## 7.10、阶段 10：RCU 回调 —— PTE 页物理归还与记账

grace period 到期后，RCU 回调执行（[mmu_gather.c](file:~/linux_dir/linux_old1/mm/mmu_gather.c#L222-L230)、[asm-generic/tlb.h](file:~/linux_dir/linux_old1/include/asm-generic/tlb.h#L215-L222)）：

```c
tlb_remove_table_rcu → __tlb_remove_table_free
  → __tlb_remove_table(ptdesc)                       // generic 版（arm64 未覆写）
      → pagetable_dtor_free(ptdesc)                  // include/linux/mm.h:3103
          ├─ pagetable_dtor(ptdesc)                  // mm.h:3094
          │    ├─ ptlock_free(ptdesc)                // arm64 4K 页：no-op（锁内嵌 ptdesc->ptl）
          │    ├─ __ClearPageTable(ptdesc_page)      // 清 PG_table 标志
          │    └─ mod_node_page_state(NR_PAGETABLE, -1)  // node 统计递减
          └─ pagetable_free(ptdesc)                  // 页归还 buddy 系统
```

- **同步路径（数据页）**：zap 阶段入 page batch 的普通页，在 `tlb_finish_mmu` 内随 `tlb_batch_pages_flush` 立即归还（本机制不涉及数据页，此处为同路径普通 zap）。
- **异步路径（PTE 表页）**：经 RCU 延迟至 grace period 后归还 —— 保证 7.5 ③ 的 `rcu_read_lock` 读者安全。

## 7.11、关键状态量随调用链的变化（一次典型回收）

| 状态量 | 初始（tlb_gather_mmu） | zap present PTE 后 | free_pte 后 | tlb_finish_mmu 后 |
|---|---|---|---|---|
| `tlb->start` | `TASK_SIZE` | `min(..., addr)`（被 zap 地址） | 并入回收 PMD 地址 | `TASK_SIZE`（reset） |
| `tlb->end` | `0` | 被 zap 地址 + 长度 | +PAGE_SIZE 覆盖回收 PMD | `0`（reset） |
| `cleared_ptes` | 0 | 1（有 present PTE 被清） | 不变 | 0 |
| `cleared_pmds` | 0 | 0（除非 THP） | **1**（tlb_flush_pmd_range） | 0 |
| `freed_tables` | 0 | 0 | **1** | 0 |
| `mm->pgtables_bytes` | 初始 | 不变 | **-4KB**（mm_dec_nr_ptes） | 不变 |
| `NR_PAGETABLE`（node） | 初始 | 不变 | 不变 | RCU 回调后 **-1** |

> 注：`cleared_pmds=1` + `freed_tables=1` 的组合是 arm64 `tlb_get_level()` 返回 `TLBI_TTL_UNKNOWN`、`tlb_get_unmap_shift()` 返回 `PMD_SHIFT` 的依据；`tlb_flush_mmu_tlbonly` 的判据正是依赖这些位被置位后才执行架构 flush。

---

# 8、qemu 实测验证（aarch64 适配前/后对比）

## 8.1、测试用例

`rept_test`（[rept_test.c](file:~/linux_dir/test_dir/rept_test/rept_test.c#L1-142)）：`mmap 1GB（MADV_NOHUGEPAGE 强制 4KB 基页）` → `while(1){ 逐 2MB memset touch → madvise(MADV_DONTNEED) }`，每轮结束打印进程 `VmPTE`（/proc/self/status）与系统 `PageTables`（/proc/meminfo）。

> ⚠️ 关键前提：若内核 `CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS=y`（默认全局开启 THP），2MB touch 走 PMD 大页映射、不产生 4KB PTE 页，PT_RECLAIM 无对象可回收（实测两内核 VmPTE 均稳定 236 kB 无差异）。必须 `madvise(MADV_NOHUGEPAGE)` 强制 4KB 基页后效果才可见。

## 8.2、测试用例源码（完整）

完整源码：[rept_test.c](file:~/linux_dir/test_dir/rept_test/rept_test.c#L1-142)（共 142 行），交叉编译后置于 qemu initramfs 中运行：

```c
/*
 * rept_test.c - reclaim empty PTE page 机制验证测试用例
 *
 * 复现 lkml patch 7/9 (mm: pgtable: try to reclaim empty PTE page
 * in madvise(MADV_DONTNEED)) 提交信息中的测试逻辑：
 *
 *     mmap 50G
 *     while (1) {
 *         for (i = 0; i < 1024 * 25; i++) {   // 25600 次 × 2MB = 50GB
 *             touch 2M memory
 *             madvise MADV_DONTNEED 2M
 *         }
 *     }
 *
 * 预期效果：反复 touch + MADV_DONTNEED 后，空 PTE 页被 PT_RECLAIM 机制
 * 回收，VmPTE / PageTables 大幅下降（上游示例：102640 KB -> 240 KB）。
 *
 * 编译（aarch64 静态链接，供 qemu 使用）：
 *     aarch64-linux-gnu-gcc -O2 -static -o rept_test rept_test.c
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>

#define MMAP_SIZE	(1ULL * 1024ULL * 1024ULL * 1024ULL)	/* 默认 1 GB */
#define CHUNK_SIZE	(2 * 1024 * 1024)			/* 2 MB */

/* 读取 /proc/self/status 中 VmPTE 字段，单位 kB */
static unsigned long get_vmpte_kb(void)
{
	FILE *f;
	char line[256];
	unsigned long v = 0;

	f = fopen("/proc/self/status", "r");
	if (!f) {
		perror("open /proc/self/status");
		return 0;
	}
	while (fgets(line, sizeof(line), f)) {
		if (sscanf(line, "VmPTE: %lu kB", &v) == 1)
			break;
	}
	fclose(f);
	return v;
}

/* 读取 /proc/meminfo 中 PageTables 字段，单位 kB */
static unsigned long get_pagetables_kb(void)
{
	FILE *f;
	char line[256];
	unsigned long v = 0;

	f = fopen("/proc/meminfo", "r");
	if (!f) {
		perror("open /proc/meminfo");
		return 0;
	}
	while (fgets(line, sizeof(line), f)) {
		if (sscanf(line, "PageTables: %lu kB", &v) == 1)
			break;
	}
	fclose(f);
	return v;
}

static void dump(const char *tag)
{
	printf("[%s] VmPTE=%lu kB, PageTables=%lu kB\n",
	       tag, get_vmpte_kb(), get_pagetables_kb());
	fflush(stdout);
}

int main(int argc, char *argv[])
{
	char *base;
	unsigned long long i, rounds;
	size_t mmap_size = MMAP_SIZE;
	size_t nchunks;

	/* 可选参数：./rept_test [size_mb]，默认 1GB。
	 * 注意：qemu 纯软件模拟下缺页很慢，50GB 无 THP 时一轮需 ~1300 万次缺页，
	 * 建议 1GB~4GB 即可清楚验证 VmPTE 回收效果。 */
	if (argc > 1)
		mmap_size = strtoull(argv[1], NULL, 0) * 1024ULL * 1024ULL;
	nchunks = mmap_size / CHUNK_SIZE;

	/* stdout 无缓冲，qemu 串口上实时可见 */
	setvbuf(stdout, NULL, _IONBF, 0);

	printf("=== reclaim empty PTE page test (%llu MB, CHUNK=2M) ===\n",
	       (unsigned long long)(mmap_size / 1024 / 1024));

	/* 1) mmap 之前 */
	dump("before mmap");

	base = mmap(NULL, mmap_size, PROT_READ | PROT_WRITE,
		    MAP_PRIVATE | MAP_ANONYMOUS | MAP_NORESERVE, -1, 0);
	if (base == MAP_FAILED) {
		perror("mmap");
		return 1;
	}

	/*
	 * 强制 4KB 基页：禁用该 VMA 的透明大页（THP）。
	 *
	 * 若内核 CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS=y（默认全局开启），
	 * 2MB 的 touch 会走 PMD 大页映射，根本不分配 4KB PTE 页，
	 * PT_RECLAIM 就没有空 PTE 页可回收，适配前后看不出差异。
	 * MADV_NOHUGEPAGE 后，touch 按 4KB 小页分配 PTE 页，
	 * 回收效果（VmPTE 大幅下降）才能体现。
	 */
	if (madvise(base, mmap_size, MADV_NOHUGEPAGE)) {
		perror("madvise MADV_NOHUGEPAGE");
		return 1;
	}

	/* 2) mmap 之后（仅建 VMA，未 touch，PTE 页应尚未分配） */
	dump("after mmap");

	/* 3) while(1)：每轮遍历全部映射，逐 2MB touch + MADV_DONTNEED */
	rounds = 0;
	while (1) {
		for (i = 0; i < nchunks; i++) {
			char *p = base + i * CHUNK_SIZE;

			memset(p, 0x5a, CHUNK_SIZE);	/* touch 2M：触发缺页，分配物理页 + PTE 页 */
			madvise(p, CHUNK_SIZE, MADV_DONTNEED);	/* 释放物理页，空 PTE 页被回收 */
		}
		/* 每一轮结束时输出 */
		dump("after round done");
		printf("round=%llu, touched=%llu MB\n",
		       rounds, (unsigned long long)(nchunks * CHUNK_SIZE / 1024 / 1024));
		rounds++;
	}

	munmap(base, mmap_size);	/* 不可达 */
	return 0;
}
```

> 说明：测试程序已在 qemu（virt 机型，`-smp 2 -m 512M`）中实际运行，输出见 8.3 节。`round` 循环无限执行，按 `Ctrl-A x` 退出 qemu。

## 8.3、实测数据

适配前（无 PT_RECLAIM）：

```bash
/tmp # zcat /proc/config.gz | grep PT_RECLAIM     ← 无输出
/tmp # ./rept_test
=== reclaim empty PTE page test (1024 MB, CHUNK=2M) ===
[before mmap] VmPTE=36 kB, PageTables=280 kB
[after mmap]  VmPTE=36 kB, PageTables=280 kB
[after round done] VmPTE=2088 kB, PageTables=2300 kB   ← 512 个空 PTE 页全部保留
round=0, touched=1024 MB
[after round done] VmPTE=2088 kB, PageTables=2300 kB   ← 每轮稳定，只增不减
round=1, touched=1024 MB
...
```

适配后（有 PT_RECLAIM）：

```bash
/tmp # zcat /proc/config.gz | grep PT_RECLAIM
CONFIG_ARCH_SUPPORTS_PT_RECLAIM=y
CONFIG_PT_RECLAIM=y
/tmp # ./rept_test
=== reclaim empty PTE page test (1024 MB, CHUNK=2M) ===
[before mmap] VmPTE=36 kB, PageTables=212 kB
[after mmap]  VmPTE=36 kB, PageTables=212 kB
[after round done] VmPTE=40 kB, PageTables=260 kB     ← 空 PTE 页每轮全部回收
round=0, touched=1024 MB
[after round done] VmPTE=40 kB, PageTables=256 kB
round=1, touched=1024 MB
...
```

## 8.4、结果分析

| 指标 | 适配前 | 适配后 | 说明 |
|---|---|---|---|
| 每轮结束 `VmPTE` | 2088 kB（稳定） | 40 kB（稳定） | ↓98%；2088 kB = 36 基础 + 512×4KB（2048 kB，1GB/2MB）+ PGD/PUD 页，与理论吻合 |
| 每轮结束 `PageTables` | 2300 kB | 256 kB | ↓89%，系统级统计同步下降 |
| 多轮趋势 | 只增不减（空 PTE 页保留） | 每轮回落到低位（空 PTE 页回收） | 印证 patch 提交信息中的病态与机制效果 |

与上游 x86 示例（`VmPTE 102640 → 240 kB`，50GB，约 427×）相比，本次 1GB 场景约 52×，回收率本质一致（空 PTE 页 100% 回收），证明 arm64 上 `free_pte → pte_free_tlb`（`tlb_flush_pmd_range` + `freed_tables=1` + RCU 释放 + `mm_dec_nr_ptes`）语义与 x86 完全等价。

## 8.5、编译与运行验证步骤

```bash
# 1. 内核配置（arm64 defconfig 基础上确认/追加）
make ARCH=arm64 defconfig
grep -E "PT_RECLAIM|MMU_GATHER_RCU_TABLE_FREE" .config
# CONFIG_ARCH_SUPPORTS_PT_RECLAIM=y
# CONFIG_PT_RECLAIM=y
# CONFIG_MMU_GATHER_RCU_TABLE_FREE=y

# 2. 交叉编译（本项目已实际执行，成功）
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image -j$(nproc)
#  ... CC      mm/pt_reclaim.o     ← 无报错（原 implicit declaration 已消除）
#  ... OBJCOPY arch/arm64/boot/Image

# 3. qemu 启动（virt 机型，initramfs 内置 busybox，同一内核两版对比：适配前/适配后）
qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 2 -m 512M \
  -kernel arch/arm64/boot/Image \
  -initrd /path/to/initramfs.cpio.gz \
  -nographic -append "console=ttyAMA0 rdinit=/init"
```

---

# 9、总结

1. **机制本质**：`madvise(MADV_DONTNEED)` 释放 ≥2MB 整块映射时，`zap_pte_range()` 通过新增的 `reclaim_pt` 标志同步扫描并回收"全部 PTE 均为 none"的空 PTE 页，从根源上解决 jemalloc/tcmalloc 场景下 `VmPTE` 只增不减的问题。该功能由字节跳动 Qi Zheng 于 2024-11 提交系列 patch，2025 年合入 6.14。

2. **识别（fast/slow 双路径）**：fast path 依赖"ptl 全程未释放 → 全空是构造出的必然"，`try_get_and_clear_pmd` 用 `spin_trylock(pml)` 抢占 pmd 锁直接摘表；slow path 在 ptl 释放过 / retry / 抢锁失败时，重新持 pml+ptl **逐项复查 512 个 PTE** 后才 `pmd_clear`。两把锁各管一层：pml 管"pmd 指向哪张表"（结构），ptl 管"表里每一项是啥"（内容），缺一不可，拿锁顺序相反且 slow path 用嵌套标记防死锁。

3. **清除**：两条路径统一汇入 `free_pte()` → `pte_free_tlb` 宏（范围并入 + `freed_tables=1` + `__pte_free_tlb`）+ `mm_dec_nr_ptes` 记账。

4. **同步（x86 "IPI+同步" → "IPI+RCU"）**：页表页释放从 semi-RCU（批量 RCU、单表 IPI+同步）改为全 RCU（批量与单表兜底都 `call_rcu`），因为 IPI 无法与 `pte_offset_map` 的 RCU 读端同步（中断可打断 RCU 临界区，ACK 不代表读端退出）；TLB 翻译失效仍保留 IPI+广播同步。**IPI 负责"让世界看不见旧翻译"，RCU 负责"让世界不再引用这块内存"**。

5. **运行时调用链（核心结论）**：一次回收从 `madvise` 系统调用到物理页归还共 10 个阶段——入口 → 框架（VMA 读锁 + `tlb_gather_mmu`）→ 遍历 → 下钻 → 核心 zap（fast/slow 回收）→ 摘除 → `pte_free_tlb` 宏展开 → `tlb_remove_table` 入 batch → `tlb_finish_mmu`（arm64 `vae1is` 全级别 TLB 失效 + RCU 释放）→ RCU 回调归还。

6. **aarch64 适配的最终形态**：与 x86 **完全一致、无架构分支**。核心修改只有两点——`arch/arm64/Kconfig` 增加 `select ARCH_SUPPORTS_PT_RECLAIM`；`mm/pt_reclaim.c` 的 include 从 `<asm-generic/tlb.h>` 修正为 `<asm/tlb.h>`（使 `pte_free_tlb` 宏在 arm64 上可完整展开，`__pte_free_tlb` 可见）。曾尝试的 `tlb_remove_page` 方案因丢失 TLB 范围记录与 `freed_tables` 语义而被弃用。

7. **安全保证**：回收路径与常规 `munmap`/`free_pte_range` 共享同一套 mmu_gather 基础设施（`pte_free_tlb` → RCU 延迟释放 → 先失效后释放），并发安全依赖 `MMU_GATHER_RCU_TABLE_FREE` 与配套补丁（khugepaged/userfaultfd 二次校验 pmd 状态）。

8. **验证状态**：`mm/pt_reclaim.c` 已实际交叉编译通过；`.config` 确认 `CONFIG_ARCH_SUPPORTS_PT_RECLAIM=y`、`CONFIG_PT_RECLAIM=y`、`CONFIG_MMU_GATHER_RCU_TABLE_FREE=y`；qemu 双内核对比实测：适配前 `VmPTE` 2088 kB 只增不减 → 适配后 40 kB 每轮回落（约 52×），证明机制在 aarch64 上完整生效。

---

> **参考**：LWN《synchronously scan and reclaim empty user PTE pages》(v3: lwn.net/Articles/998169)；LKML PATCH v4 0/11（lkml.iu.edu/2412.0/04492.html）；作者对分步计划的答复（lkml.iu.edu/2412.0/06012.html）。