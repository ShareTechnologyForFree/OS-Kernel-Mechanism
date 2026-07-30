# 1、Linux内核内存概述

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201154902351.png" alt="image-20260201154902351" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201155054224.png" alt="image-20260201155054224" style="zoom:50%;" />

# 2、Node介绍

## 2.1、Node结构体

```c
// include/linux/mmzone.h
typedef struct pglist_data {
	/*
	 * node_zones contains just the zones for THIS node. Not all of the
	 * zones may be populated, but it is the full list. It is referenced by
	 * this node's node_zonelists as well as other node's node_zonelists.
	 */
	struct zone node_zones[MAX_NR_ZONES];

	/*
	 * node_zonelists contains references to all zones in all nodes.
	 * Generally the first zones will be references to this node's
	 * node_zones.
	 */
	struct zonelist node_zonelists[MAX_ZONELISTS];

	int nr_zones; /* number of populated zones in this node */
#ifdef CONFIG_FLATMEM	/* means !SPARSEMEM */
	struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#if defined(CONFIG_MEMORY_HOTPLUG) || defined(CONFIG_DEFERRED_STRUCT_PAGE_INIT)
	/*
	 * Must be held any time you expect node_start_pfn,
	 * node_present_pages, node_spanned_pages or nr_zones to stay constant.
	 * Also synchronizes pgdat->first_deferred_pfn during deferred page
	 * init.
	 *
	 * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
	 * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG
	 * or CONFIG_DEFERRED_STRUCT_PAGE_INIT.
	 *
	 * Nests above zone->lock and zone->span_seqlock
	 */
	spinlock_t node_size_lock;
#endif
	unsigned long node_start_pfn;
	unsigned long node_present_pages; /* total number of physical pages */
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	int node_id;
	wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;

	/* workqueues for throttling reclaim for different reasons. */
	wait_queue_head_t reclaim_wait[NR_VMSCAN_THROTTLE];

	atomic_t nr_writeback_throttled;/* nr of writeback-throttled tasks */
	unsigned long nr_reclaim_start;	/* nr pages written while throttled
					 * when throttling started. */
#ifdef CONFIG_MEMORY_HOTPLUG
	struct mutex kswapd_lock;
#endif
	struct task_struct *kswapd;	/* Protected by kswapd_lock */
	int kswapd_order;
	enum zone_type kswapd_highest_zoneidx;

	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */

#ifdef CONFIG_COMPACTION
	int kcompactd_max_order;
	enum zone_type kcompactd_highest_zoneidx;
	wait_queue_head_t kcompactd_wait;
	struct task_struct *kcompactd;
	bool proactive_compact_trigger;
#endif
	/*
	 * This is a per-node reserve of pages that are not available
	 * to userspace allocations.
	 */
	unsigned long		totalreserve_pages;

#ifdef CONFIG_NUMA
	/*
	 * node reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* Write-intensive fields used by page reclaim */
	CACHELINE_PADDING(_pad1_);

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
	/*
	 * If memory initialisation on large machines is deferred then this
	 * is the first PFN that needs to be initialised.
	 */
	unsigned long first_deferred_pfn;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	struct deferred_split deferred_split_queue;
#endif

#ifdef CONFIG_NUMA_BALANCING
	/* start time in ms of current promote rate limit period */
	unsigned int nbp_rl_start;
	/* number of promote candidate pages at start time of current rate limit period */
	unsigned long nbp_rl_nr_cand;
	/* promote threshold in ms */
	unsigned int nbp_threshold;
	/* start time in ms of current promote threshold adjustment period */
	unsigned int nbp_th_start;
	/*
	 * number of promote candidate pages at start time of current promote
	 * threshold adjustment period
	 */
	unsigned long nbp_th_nr_cand;
#endif
	/* Fields commonly accessed by the page reclaim scanner */

	/*
	 * NOTE: THIS IS UNUSED IF MEMCG IS ENABLED.
	 *
	 * Use mem_cgroup_lruvec() to look up lruvecs.
	 */
	struct lruvec		__lruvec;

	unsigned long		flags;

#ifdef CONFIG_LRU_GEN
	/* kswap mm walk data */
	struct lru_gen_mm_walk mm_walk;
	/* lru_gen_folio list */
	struct lru_gen_memcg memcg_lru;
#endif

	CACHELINE_PADDING(_pad2_);

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
#ifdef CONFIG_NUMA
	struct memory_tier __rcu *memtier;
#endif
#ifdef CONFIG_MEMORY_FAILURE
	struct memory_failure_stats mf_stats;
#endif
} pg_data_t;
```

# 3、Zone介绍

## 3.1、Zone的分类

```c
// include/linux/mmzone.h
enum zone_type {
#ifdef CONFIG_ZONE_DMA
	ZONE_DMA,
#endif
    
#ifdef CONFIG_ZONE_DMA32
	ZONE_DMA32,
#endif
    
	ZONE_NORMAL,
    
#ifdef CONFIG_HIGHMEM
	ZONE_HIGHMEM,
#endif

	ZONE_MOVABLE,
    
#ifdef CONFIG_ZONE_DEVICE
	ZONE_DEVICE,
#endif
    
	__MAX_NR_ZONES

};
```

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201153321081.png" alt="image-20260201153321081" style="zoom:50%;" />

## 3.2、每个Zone类型的深度解析

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201153528731.png" alt="image-20260201153528731" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201153620363.png" alt="image-20260201153620363" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201153635401.png" alt="image-20260201153635401" style="zoom:50%;" />

## 3.3、Zone在机器上的查看

```bash
~ # cat /proc/zoneinfo 
Node 0, zone      DMA
  per-node stats
      nr_inactive_anon 12
      nr_active_anon 1
      nr_inactive_file 89
      nr_active_file 362
      nr_unevictable 0
      nr_slab_reclaimable 447
      nr_slab_unreclaimable 1166
      nr_isolated_anon 0
      nr_isolated_file 0
      workingset_nodes 0
      workingset_refault_anon 0
      workingset_refault_file 0
      workingset_activate_anon 0
      workingset_activate_file 0
      workingset_restore_anon 0
      workingset_restore_file 0
      workingset_nodereclaim 0
      nr_anon_pages 15
      nr_mapped    345
      nr_file_pages 464
      nr_dirty     0
      nr_writeback 0
      nr_writeback_temp 0
      nr_shmem     0
      nr_shmem_hugepages 0
      nr_shmem_pmdmapped 0
      nr_file_hugepages 0
      nr_file_pmdmapped 0
      nr_anon_transparent_hugepages 0
      nr_vmscan_write 0
      nr_vmscan_immediate_reclaim 0
      nr_dirtied   4105
      nr_written   4105
      nr_throttled_written 0
      nr_kernel_misc_reclaimable 0
      nr_foll_pin_acquired 0
      nr_foll_pin_released 0
      nr_kernel_stack 1144
      nr_page_table_pages 19
      nr_sec_page_table_pages 0
      nr_iommu_pages 0
      nr_swapcached 0
      pgpromote_success 0
      pgpromote_candidate 0
      pgdemote_kswapd 0
      pgdemote_direct 0
      pgdemote_khugepaged 0
      nr_hugetlb   0
  pages free     235154
        boost    0
        min      5512
        low      6890
        high     8268
        promo    9646
        spanned  262144
        present  262144
        managed  248725
        cma      0
        protection: (0, 0, 0, 0)
      nr_free_pages 235154
      nr_zone_inactive_anon 12
      nr_zone_active_anon 1
      nr_zone_inactive_file 89
      nr_zone_active_file 362
      nr_zone_unevictable 0
      nr_zone_write_pending 0
      nr_mlock     0
      nr_bounce    0
      nr_free_cma  0
      numa_hit     11594
      numa_miss    0
      numa_foreign 0
      numa_interleave 8852
      numa_local   11593
      numa_other   1
  pagesets
    cpu: 0
              count:    1722
              high:     1722
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 1
              count:    781
              high:     1722
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 2
              count:    2135
              high:     2135
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 3
              count:    511
              high:     1722
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 4
              count:    0
              high:     1722
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 5
              count:    0
              high:     0
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 6
              count:    0
              high:     0
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
    cpu: 7
              count:    4
              high:     1722
              batch:    63
              high_min: 1722
              high_max: 7678
  vm stats threshold: 32
  node_unreclaimable:  0
  start_pfn:           262144
Node 0, zone    DMA32
  pages free     0
        boost    0
        min      0
        low      0
        high     0
        promo    0
        spanned  0
        present  0
        managed  0
        cma      0
        protection: (0, 0, 0, 0)
Node 0, zone   Normal
  pages free     0
        boost    0
        min      0
        low      0
        high     0
        promo    0
        spanned  0
        present  0
        managed  0
        cma      0
        protection: (0, 0, 0, 0)
Node 0, zone  Movable
  pages free     0
        boost    0
        min      32
        low      32
        high     32
        promo    32
        spanned  0
        present  0
        managed  0
        cma      0
        protection: (0, 0, 0, 0)
Node 1, zone      DMA
  per-node stats
      nr_inactive_anon 17
      nr_active_anon 1
      nr_inactive_file 5
      nr_active_file 63
      nr_unevictable 0
      nr_slab_reclaimable 813
      nr_slab_unreclaimable 1203
      nr_isolated_anon 0
      nr_isolated_file 0
      workingset_nodes 0
      workingset_refault_anon 0
      workingset_refault_file 0
      workingset_activate_anon 0
      workingset_activate_file 0
      workingset_restore_anon 0
      workingset_restore_file 0
      workingset_nodereclaim 0
      nr_anon_pages 23
      nr_mapped    30
      nr_file_pages 70
      nr_dirty     1
      nr_writeback 0
      nr_writeback_temp 0
      nr_shmem     0
      nr_shmem_hugepages 0
      nr_shmem_pmdmapped 0
      nr_file_hugepages 0
      nr_file_pmdmapped 0
      nr_anon_transparent_hugepages 0
      nr_vmscan_write 0
      nr_vmscan_immediate_reclaim 0
      nr_dirtied   4110
      nr_written   4109
      nr_throttled_written 0
      nr_kernel_misc_reclaimable 0
      nr_foll_pin_acquired 0
      nr_foll_pin_released 0
      nr_kernel_stack 536
      nr_page_table_pages 13
      nr_sec_page_table_pages 0
      nr_iommu_pages 0
      nr_swapcached 0
      pgpromote_success 0
      pgpromote_candidate 0
      pgdemote_kswapd 0
      pgdemote_direct 0
      pgdemote_khugepaged 0
      nr_hugetlb   0
  pages free     243278
        boost    0
        min      5751
        low      7188
        high     8625
        promo    10062
        spanned  262144
        present  262144
        managed  256381
        cma      8192
        protection: (0, 0, 0, 0)
      nr_free_pages 243278
      nr_zone_inactive_anon 17
      nr_zone_active_anon 1
      nr_zone_inactive_file 5
      nr_zone_active_file 63
      nr_zone_unevictable 0
      nr_zone_write_pending 1
      nr_mlock     0
      nr_bounce    0
      nr_free_cma  8000
      numa_hit     10902
      numa_miss    0
      numa_foreign 0
      numa_interleave 8985
      numa_local   833
      numa_other   10069
  pagesets
    cpu: 0
              count:    755
              high:     1797
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 1
              count:    0
              high:     1797
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 2
              count:    1797
              high:     1797
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 3
              count:    78
              high:     1797
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 4
              count:    991
              high:     1797
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 5
              count:    1117
              high:     1797
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 6
              count:    285
              high:     1848
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
    cpu: 7
              count:    532
              high:     1860
              batch:    63
              high_min: 1797
              high_max: 8011
  vm stats threshold: 32
  node_unreclaimable:  0
  start_pfn:           524288
Node 1, zone    DMA32
  pages free     0
        boost    0
        min      0
        low      0
        high     0
        promo    0
        spanned  0
        present  0
        managed  0
        cma      0
        protection: (0, 0, 0, 0)
Node 1, zone   Normal
  pages free     0
        boost    0
        min      0
        low      0
        high     0
        promo    0
        spanned  0
        present  0
        managed  0
        cma      0
        protection: (0, 0, 0, 0)
Node 1, zone  Movable
  pages free     0
        boost    0
        min      32
        low      32
        high     32
        promo    32
        spanned  0
        present  0
        managed  0
        cma      0
        protection: (0, 0, 0, 0)
~ # 
```

## 3.4、Linux内核中的Zone结构体

这个结构体结合3中机器上的 /proc/zoneinfo对照查看。

```c
// include/linux/mmzone.h
struct zone {
	/* Read-mostly fields */

	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long _watermark[NR_WMARK];
	unsigned long watermark_boost;

	unsigned long nr_reserved_highatomic;
	unsigned long nr_free_highatomic;

	long lowmem_reserve[MAX_NR_ZONES];

#ifdef CONFIG_NUMA
	int node;
#endif
	struct pglist_data	*zone_pgdat;	// 这就是Zone所在的Node节点结构体
	struct per_cpu_pages	__percpu *per_cpu_pageset;
	struct per_cpu_zonestat	__percpu *per_cpu_zonestats;
	/*
	 * the high and batch values are copied to individual pagesets for
	 * faster access
	 */
	int pageset_high_min;
	int pageset_high_max;
	int pageset_batch;

#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */

	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;

	atomic_long_t		managed_pages;
	unsigned long		spanned_pages;
	unsigned long		present_pages;
#if defined(CONFIG_MEMORY_HOTPLUG)
	unsigned long		present_early_pages;
#endif
#ifdef CONFIG_CMA
	unsigned long		cma_pages;
#endif

	const char		*name;

#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Number of isolated pageblock. It is used to solve incorrect
	 * freepage counting problem due to racy retrieving migratetype
	 * of pageblock. Protected by zone->lock.
	 */
	unsigned long		nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif

	int initialized;

	/* Write-intensive fields used from the page allocator */
	CACHELINE_PADDING(_pad1_);

	/* free areas of different sizes */
	struct free_area	free_area[NR_PAGE_ORDERS];

#ifdef CONFIG_UNACCEPTED_MEMORY
	/* Pages to be accepted. All pages on the list are MAX_PAGE_ORDER */
	struct list_head	unaccepted_pages;
#endif

	/* zone flags, see below */
	unsigned long		flags;

	/* Primarily protects free_area */
	spinlock_t		lock;

	/* Write-intensive fields used by compaction and vmstats. */
	CACHELINE_PADDING(_pad2_);

	/*
	 * When free pages are below this point, additional steps are taken
	 * when reading the number of free pages to avoid per-cpu counter
	 * drift allowing watermarks to be breached
	 */
	unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* pfn where compaction free scanner should start */
	unsigned long		compact_cached_free_pfn;
	/* pfn where compaction migration scanner should start */
	unsigned long		compact_cached_migrate_pfn[ASYNC_AND_SYNC];
	unsigned long		compact_init_migrate_pfn;
	unsigned long		compact_init_free_pfn;
#endif

#ifdef CONFIG_COMPACTION
	/*
	 * On compaction failure, 1<<compact_defer_shift compactions
	 * are skipped before trying again. The number attempted since
	 * last failure is tracked with compact_considered.
	 * compact_order_failed is the minimum compaction failed order.
	 */
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
	int			compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* Set to true when the PG_migrate_skip bits should be cleared */
	bool			compact_blockskip_flush;
#endif

	bool			contiguous;

	CACHELINE_PADDING(_pad3_);
	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	atomic_long_t		vm_numa_event[NR_VM_NUMA_EVENT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```

# 4、Page介绍

## 4.1、基本介绍

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201155516611.png" alt="image-20260201155516611" style="zoom:50%;" />

配置方式：

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201155717520.png" alt="image-20260201155717520" style="zoom:50%;" />

```bash
 Kernel Features  --->
	Page size (4KB)  ---> 
        (X) 4KB
        ( ) 16KB
        ( ) 64KB   
```

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201155840126.png" alt="image-20260201155840126" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160107500.png" alt="image-20260201160107500" style="zoom:50%;" />

## 4.2、内核结构体

```c
// include/linux/mm_types.h
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	/*
	 * Five words (20/40 bytes) are available in this union.
	 * WARNING: bit 0 of the first word is used for PageTail(). That
	 * means the other users of this union MUST NOT use the bit to
	 * avoid collision and false-positive PageTail().
	 */
	union {
		struct {	/* Page cache and anonymous pages */
			/**
			 * @lru: Pageout list, eg. active_list protected by
			 * lruvec->lru_lock.  Sometimes used as a generic list
			 * by the page owner.
			 */
			union {
				struct list_head lru;

				/* Or, for the Unevictable "LRU list" slot */
				struct {
					/* Always even, to negate PageTail */
					void *__filler;
					/* Count page's or folio's mlocks */
					unsigned int mlock_count;
				};

				/* Or, free page */
				struct list_head buddy_list;
				struct list_head pcp_list;
			};
			/* See page-flags.h for PAGE_MAPPING_FLAGS */
			struct address_space *mapping;
			union {
				pgoff_t index;		/* Our offset within mapping. */
				unsigned long share;	/* share count for fsdax */
			};
			/**
			 * @private: Mapping-private opaque data.
			 * Usually used for buffer_heads if PagePrivate.
			 * Used for swp_entry_t if swapcache flag set.
			 * Indicates order in the buddy system if PageBuddy.
			 */
			unsigned long private;
		};
		struct {	/* page_pool used by netstack */
			/**
			 * @pp_magic: magic value to avoid recycling non
			 * page_pool allocated pages.
			 */
			unsigned long pp_magic;
			struct page_pool *pp;
			unsigned long _pp_mapping_pad;
			unsigned long dma_addr;
			atomic_long_t pp_ref_count;
		};
		struct {	/* Tail pages of compound page */
			unsigned long compound_head;	/* Bit zero is set */
		};
		struct {	/* ZONE_DEVICE pages */
			/** @pgmap: Points to the hosting device page map. */
			struct dev_pagemap *pgmap;
			void *zone_device_data;
			/*
			 * ZONE_DEVICE private pages are counted as being
			 * mapped so the next 3 words hold the mapping, index,
			 * and private fields from the source anonymous or
			 * page cache page while the page is migrated to device
			 * private memory.
			 * ZONE_DEVICE MEMORY_DEVICE_FS_DAX pages also
			 * use the mapping, index, and private fields when
			 * pmem backed DAX files are mapped.
			 */
		};

		/** @rcu_head: You can use this to free a page by RCU. */
		struct rcu_head rcu_head;
	};

	union {		/* This union is 4 bytes in size. */
		/*
		 * For head pages of typed folios, the value stored here
		 * allows for determining what this page is used for. The
		 * tail pages of typed folios will not store a type
		 * (page_type == _mapcount == -1).
		 *
		 * See page-flags.h for a list of page types which are currently
		 * stored here.
		 *
		 * Owners of typed folios may reuse the lower 16 bit of the
		 * head page page_type field after setting the page type,
		 * but must reset these 16 bit to -1 before clearing the
		 * page type.
		 */
		unsigned int page_type;

		/*
		 * For pages that are part of non-typed folios for which mappings
		 * are tracked via the RMAP, encodes the number of times this page
		 * is directly referenced by a page table.
		 *
		 * Note that the mapcount is always initialized to -1, so that
		 * transitions both from it and to it can be tracked, using
		 * atomic_inc_and_test() and atomic_add_negative(-1).
		 */
		atomic_t _mapcount;
	};

	/* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
	atomic_t _refcount;

#ifdef CONFIG_MEMCG
	unsigned long memcg_data;
#elif defined(CONFIG_SLAB_OBJ_EXT)
	unsigned long _unused_slab_obj_exts;
#endif

	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
	int _last_cpupid;
#endif

#ifdef CONFIG_KMSAN
	/*
	 * KMSAN metadata for this page:
	 *  - shadow page: every bit indicates whether the corresponding
	 *    bit of the original page is initialized (0) or not (1);
	 *  - origin page: every 4 bytes contain an id of the stack trace
	 *    where the uninitialized value was created.
	 */
	struct page *kmsan_shadow;
	struct page *kmsan_origin;
#endif
} _struct_page_alignment;
```

### 4.2.1、struct page结构体的空间优化设计

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160404907.png" alt="image-20260201160404907" style="zoom:50%;" />

### 4.2.2、多页面类型的统一管理

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160439854.png" alt="image-20260201160439854" style="zoom:50%;" />

### 4.2.3、引用计数和生命周期管理

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160601892.png" alt="image-20260201160601892" style="zoom:50%;" />

### 4.2.4、复合页面的设计

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160644450.png" alt="image-20260201160644450" style="zoom:50%;" />

### 5.2.5、内存回收和LRU管理

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160730241.png" alt="image-20260201160730241" style="zoom:50%;" />

### 5.2.6、平台相关优化

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201160811315.png" alt="image-20260201160811315" style="zoom:50%;" />

### 5.2.7、性能优化

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260201161004509.png" alt="image-20260201161004509" style="zoom:50%;" />

























