# 1、内存碎片

## 1.1、外部碎片

解决方案如伙伴系统。

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260218155717688.png" alt="image-20260218155717688" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260218155933248.png" alt="image-20260218155933248" style="zoom:50%;" />

调试信息：

```c
~ # cat /proc/buddyinfo 
Node 0, zone      DMA      9      8      7      6      1      4      3      6      3      5    225 
Node 1, zone      DMA      7     11      8      4      5      5      7      1      4      3    234 
~ # 
```

从左到右依次是order=0开始的page数量。

## 1.2、内部碎片

解决方法例如slab分配器

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260218160040473.png" alt="image-20260218160040473" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260218160206383.png" alt="image-20260218160206383" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260218160300088.png" alt="image-20260218160300088" style="zoom:50%;" />

调试信息：

```c
~ # cat /proc/slabinfo 
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
ext4_groupinfo_4k     22     22    184   22    1 : tunables    0    0    0 : slabdata      1      1      0
p9_req_t               0      0    160   25    1 : tunables    0    0    0 : slabdata      0      0      0
isp1760_qtd            0      0     72   56    1 : tunables    0    0    0 : slabdata      0      0      0
isp1760_urb_listitem      0      0     24  170    1 : tunables    0    0    0 : slabdata      0      0      0
...
kmalloc-cg-8k          0      0   8192    4    8 : tunables    0    0    0 : slabdata      0      0      0
kmalloc-cg-4k         24     24   4096    8    8 : tunables    0    0    0 : slabdata      3      3      0
...
```

其中：

* num_objs 表示此slab缓存的数量（即给调用者使用的数量）
* objsize 表示一个slab缓存的大小（即给调用者使用的大小）

# 2、伙伴系统

## 2.1、伙伴系统的定义

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219152049290.png" alt="image-20260219152049290" style="zoom:50%;" />

每个Node当中分成多个不同类型的Zone，每个Zone中使用伙伴系统管理对应的物理页pages

```c
// include/linux/mmzone.h
struct zone {
...
    /*
	 * managed_pages is present pages managed by the buddy system, which
	 * is calculated as (reserved_pages includes pages allocated by the
	 * bootmem allocator):
	 *	managed_pages = present_pages - reserved_pages;
	 */
	atomic_long_t		managed_pages;
...
	/* free areas of different sizes */
	struct free_area	free_area[NR_PAGE_ORDERS];
...
} ____cacheline_internodealigned_in_smp;
```

其中 free_area数组中的 NR_PAGE_ORDERS 11，即[0~10]：

```c
// include/linux/mmzone.h

/* Free memory management - zoned buddy allocator.  */
#ifndef CONFIG_ARCH_FORCE_MAX_ORDER
#define MAX_PAGE_ORDER 10
#else
#define MAX_PAGE_ORDER CONFIG_ARCH_FORCE_MAX_ORDER
#endif
#define MAX_ORDER_NR_PAGES (1 << MAX_PAGE_ORDER)

#define NR_PAGE_ORDERS (MAX_PAGE_ORDER + 1)
```

```c
// .config 内核顶层目录配置config之后使用的文件
// 这个是我的arm64，qemu的配置
CONFIG_ARCH_FORCE_MAX_ORDER=10
```

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219153436433.png" alt="image-20260219153436433" style="zoom:50%;" />

从上图中可以看出，能够申请的最大连续内存，取决于ORDER的大小。

我使用的qemu中最大为 11，即 2^10个page。

## 2.2、伙伴系统的实现

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219153922352.png" alt="image-20260219153922352" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219153943420.png" alt="image-20260219153943420" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219154022858.png" alt="image-20260219154022858" style="zoom:50%;" />

page的迁移类型：

```c
// include/linux/mmzone.h
enum migratetype {
	MIGRATE_UNMOVABLE,		// 不可移动
	MIGRATE_MOVABLE,		// 可移动
	MIGRATE_RECLAIMABLE,	// 可回收
	MIGRATE_PCPTYPES,		// 属于CPU高速缓存中的类型，PCP是 per_cpu_pageset 的缩写
	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,	// 紧急内存
#ifdef CONFIG_CMA
	/*
	 * MIGRATE_CMA migration type is designed to mimic the way
	 * ZONE_MOVABLE works.  Only movable pages can be allocated
	 * from MIGRATE_CMA pageblocks and page allocator never
	 * implicitly change migration type of MIGRATE_CMA pageblock.
	 *
	 * The way to use it is to change migratetype of a range of
	 * pageblocks to MIGRATE_CMA which can be done by
	 * __free_pageblock_cma() function.
	 */
	MIGRATE_CMA,			// 预留的CMA内存
#endif
#ifdef CONFIG_MEMORY_ISOLATION
	MIGRATE_ISOLATE,	/* can't allocate from here */
#endif
	MIGRATE_TYPES			// 迁移类型的数量，做标记使用
};

```

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219154446416.png" alt="image-20260219154446416" style="zoom:50%;" />

## 2.3、伙伴系统的API

### 2.3.1、申请

```c
// include/linux/gfp.h
// 这里NUMA和UMA分为不同的实现，具体看代码即可
#ifdef CONFIG_NUMA
struct page *alloc_pages_noprof(gfp_t gfp, unsigned int order);
#else
static inline struct page *alloc_pages_noprof(gfp_t gfp_mask, unsigned int order)
{
	return alloc_pages_node_noprof(numa_node_id(), gfp_mask, order);
}
#endif /* CONFIG_NUMA */

#define alloc_pages(...)		alloc_hooks(alloc_pages_noprof(__VA_ARGS__))

#define alloc_page(gfp_mask)	alloc_pages(gfp_mask, 0)
```

```c
// include/linux/gfp.h
// 在 mm/page_alloc.c 中实现
extern unsigned long get_free_pages_noprof(gfp_t gfp_mask, unsigned int order);

#define __get_free_pages(...)	alloc_hooks(get_free_pages_noprof(__VA_ARGS__))

#define __get_free_page(gfp_mask)	__get_free_pages((gfp_mask), 0)
```

Linux内核中的GFP（Get Free Pages）掩码标志可以分为三种主要类型：

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219155858696.png" alt="image-20260219155858696" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219155953492.png" alt="image-20260219155953492" style="zoom:50%;" />

申请内存示意图：

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219160115540.png" alt="image-20260219160115540" style="zoom:50%;" />

伙伴系统最核心的思想就是分治和归并，将搜索page的时间降低到O(logN)。将合并的时间降低到O(1)。

这里的申请就是分治的过程。

### 2.3.1、释放

```c
// include/linux/gfp.h

// 在 mm/page_alloc.c 中定义
extern void __free_pages(struct page *page, unsigned int order);
// 这个最终也是调用 __free_pages 函数
extern void free_pages(unsigned long addr, unsigned int order);

#define __free_page(page) __free_pages((page), 0)
#define free_page(addr) free_pages((addr), 0)
```

![image-20260219160709184](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219160709184.png)

上图展示了释放的过程：先从低阶的链表中查找伙伴，之后一步步阶数更大的链表中查找，直到无法查找伙伴，并插入的最大的链表中（即归并的过程）。

### 2.3.3、测试

1）编写源文件

/home/xxx/test/buddy_test.c

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/gfp.h>
#include <linux/mm.h>

static int __init buddy_test_init(void)
{
    unsigned long addr1, addr2;
    struct page *page1, *page2;
    int order = 2;

    printk(KERN_INFO "Buddy system test module loaded.\n");

    // 测试 __get_free_pages
    addr1 = __get_free_pages(GFP_KERNEL, order);
    if (!addr1) {
        printk(KERN_ERR "Failed to allocate pages with __get_free_pages.\n");
        return -ENOMEM;
    }
    printk(KERN_INFO "Allocated %lu pages with __get_free_pages at address %lx.\n", 1UL << order, addr1);

    // 测试 __get_free_page
    addr2 = __get_free_page(GFP_KERNEL);
    if (!addr2) {
        printk(KERN_ERR "Failed to allocate page with __get_free_page.\n");
        free_pages(addr1, order);
        return -ENOMEM;
    }
    printk(KERN_INFO "Allocated a page with __get_free_page at address %lx.\n", addr2);

    // 测试 alloc_pages
    page1 = alloc_pages(GFP_KERNEL, order);
    if (!page1) {
        printk(KERN_ERR "Failed to allocate pages with alloc_pages.\n");
        free_page(addr2);
        free_pages(addr1, order);
        return -ENOMEM;
    }
    printk(KERN_INFO "Allocated pages with alloc_pages, page pointer %p.\n", page1);

    // 测试 alloc_page
    page2 = alloc_page(GFP_KERNEL);
    if (!page2) {
        printk(KERN_ERR "Failed to allocate page with alloc_page.\n");
        __free_pages(page1, order);
        free_page(addr2);
        free_pages(addr1, order);
        return -ENOMEM;
    }
    printk(KERN_INFO "Allocated a page with alloc_page, page pointer %p.\n", page2);

    // 释放所有内存
    __free_page(page2);
    __free_pages(page1, order);
    free_page(addr2);
    free_pages(addr1, order);

    printk(KERN_INFO "All memory freed. Test completed successfully.\n");
    return 0;
}

static void __exit buddy_test_exit(void)
{
    printk(KERN_INFO "Buddy system test module unloaded.\n");
}

module_init(buddy_test_init);
module_exit(buddy_test_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Test");
MODULE_DESCRIPTION("Test case for buddy system memory allocation");
```

2）编写Makefile

/home/xxx/test/Makefile

```c
KERNEL_DIR := /home/xxx/linux_old1
obj-m := buddy_test.o

ARCH := arm64
CROSS_COMPILE := aarch64-linux-gnu-

all:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean
	rm -rf *.ko *.o *.mod* *.cmd modules.order Module.symvers .tmp_versions
```

3）cd /home/xxx/test/

执行make 生成ko文件：(其中执行出错直接AI纠错即可)

```c
xxx@DESKTOP-3QNUG9S ~/test
$ ls
Makefile  buddy_test.c

xxx@DESKTOP-3QNUG9S ~/test
$ make
make -C /home/xxx/linux_old1 M=/home/xxx/test ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
make[1]: Entering directory '/home/xxx/linux_old1'
make[2]: Entering directory '/home/xxx/test'
  CC [M]  buddy_test.o
  MODPOST Module.symvers
  CC [M]  buddy_test.mod.o
  CC [M]  .module-common.o
  LD [M]  buddy_test.ko
make[2]: Leaving directory '/home/xxx/test'
make[1]: Leaving directory '/home/xxx/linux_old1'

xxx@DESKTOP-3QNUG9S ~/test
$ ls
Makefile  Module.symvers  buddy_test.c  buddy_test.ko  buddy_test.mod  buddy_test.mod.c  buddy_test.mod.o  buddy_test.o  modules.order

xxx@DESKTOP-3QNUG9S ~/test
$ 

```

4）将ko文件copy到文件系统的目录中并制作新的文件系统

```c
xxx@DESKTOP-3QNUG9S ~/test
$ cp buddy_test.ko ~/toolchain/busybox-1.36.1/_install/tmp/

xxx@DESKTOP-3QNUG9S ~/test
$ 
```

```c
xxx@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ sudo ./build_img.sh
32+0 records in
32+0 records out
33554432 bytes (34 MB, 32 MiB) copied, 0.0458109 s, 732 MB/s
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done                            
Creating filesystem with 8192 4k blocks and 8192 inodes

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done


xxx@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ 
```

5）启动qemu，并加载ko文件

```c
// 开启NUMA，两个Node，不调试
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m 2G \
  -cpu cortex-a57 \
  -smp cores=8,threads=1 \
  -object memory-backend-ram,id=mem0,size=1G \
  -object memory-backend-ram,id=mem1,size=1G \
  -numa node,memdev=mem0,cpus=0-3,nodeid=0 \
  -numa node,memdev=mem1,cpus=4-7,nodeid=1 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz \
  -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8"
```

```c
...
...
[    1.986716]     TERM=linux
[    2.184877] EXT4-fs (ram0): re-mounted dffc0903-eb36-4f0e-bf38-a01b489b61f2 r/w. Quota mode: none.
/etc/init.d/rcS: line 5: can't create /proc/sys/kernel/hotplug: nonexistent directory

Please press Enter to activate this console. 
~ # 
~ # cd tmp/
/tmp # 
/tmp # ls
buddy_test.ko               xxx
fair.c                      xxx2
hello_world_aarch64_static
/tmp # 
/tmp # 
/tmp # insmod buddy_test.ko 
[   16.512354] buddy_test: loading out-of-tree module taints kernel.
[   16.529396] Buddy system test module loaded.
[   16.530248] Allocated 4 pages with __get_free_pages at address ffff273dc40dc000.
[   16.531030] Allocated a page with __get_free_page at address ffff273dc42de000.
[   16.531708] Allocated pages with alloc_pages, page pointer 000000009e0480dc.
[   16.532497] Allocated a page with alloc_page, page pointer 00000000a5dbbc58.
[   16.533296] All memory freed. Test completed successfully.
/tmp # 
/tmp # 
/tmp # rmmod buddy_test.ko 
[  660.936048] Buddy system test module unloaded.
/tmp # 
```

### 2.3.4、调试

```
~ # 
~ # cat /proc/buddyinfo 
Node 0, zone      DMA      6      1      5      4      3      4      2      3      4      3    222 
Node 1, zone      DMA     16      7      8     10     11      9      4      4      5      3    229 
~ # 
~ # cat /proc/pagetypeinfo 
Page block order: 9
Pages per block:  512

Free pages count per migrate type at order       0      1      2      3      4      5      6      7      8      9     10 
Node    0, zone      DMA, type    Unmovable      9      6      5      6      4      4      2      2      4      2      2 
Node    0, zone      DMA, type      Movable      0      0      1      0      1      1      0      1      0      1    220 
Node    0, zone      DMA, type  Reclaimable      1      0      1      0      1      1      0      0      0      0      0 
Node    0, zone      DMA, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone      DMA, type          CMA      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone      DMA, type      Isolate      0      0      0      0      0      0      0      0      0      0      0 

Number of blocks type     Unmovable      Movable  Reclaimable   HighAtomic          CMA      Isolate 
Node 0, zone      DMA           30          474            8            0            0            0 
Page block order: 9
Pages per block:  512

Free pages count per migrate type at order       0      1      2      3      4      5      6      7      8      9     10 
Node    1, zone      DMA, type    Unmovable      7      5      8     20     22      8      2      3      4      1      0 
Node    1, zone      DMA, type      Movable      8      3      0      1      0      0      1      1      0      1    222 
Node    1, zone      DMA, type  Reclaimable      2      2      1      2      3      4      1      0      1      0      0 
Node    1, zone      DMA, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 
Node    1, zone      DMA, type          CMA      0      0      0      0      0      0      1      0      1      1      7 
Node    1, zone      DMA, type      Isolate      0      0      0      0      0      0      0      0      0      0      0 

Number of blocks type     Unmovable      Movable  Reclaimable   HighAtomic          CMA      Isolate 
Node 1, zone      DMA           30          458            8            0           16            0 
~ # 
```

# 3、slab分配器

## 3.1、简介

![image-20260219170553087](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219170553087.png)

slub分配器的config配置：

![image-20260219170659878](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219170659878.png)

## 3.2、slab基础

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219171447498.png" alt="image-20260219171447498" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219171510325.png" alt="image-20260219171510325" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219171539295.png" alt="image-20260219171539295" style="zoom:50%;" />

## 3.3、专用slab和通用slab

![image-20260219171635684](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219171635684.png)

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219171730753.png" alt="image-20260219171730753"  />

![image-20260219171936307](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219171936307.png)

## 3.4、slab着色

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219172625497.png" alt="image-20260219172625497" style="zoom:50%;" />

![image-20260219172646012](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219172646012.png)

![image-20260219172706635](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219172706635.png)

## 3.5、关键结构体

![image-20260219173249904](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219173249904.png)

### 3.5.1、struct kmem_cache

```c
// mm/slab.h
/*
 * Slab cache management.
 */
struct kmem_cache {
#ifndef CONFIG_SLUB_TINY
    // 每个CPU拥有一个本地缓存，用于无锁快速分配释放对象
	struct kmem_cache_cpu __percpu *cpu_slab;
#endif
...
    // 每个Node一个，UMA只有一个Node
	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

### 3.5.2、struct kmem_cache_cpu

```c
// mm/slub.c
struct kmem_cache_cpu {
	union {
		struct {
            // 指向CPU本地缓存的slab中第一个空闲的对象
			void **freelist;	/* Pointer to next available object */
            // 保证进程在slab cache中获取到的cpu本地缓存kmem_cache_cpu与当前执行进程的cpu一致
			unsigned long tid;	/* Globally unique transaction id */
		};
		freelist_aba_t freelist_tid;
	};
    // 这个就是实际分配的page被拆分之后的slab（原先是直接使用struct page结构体表示）
	struct slab *slab;	/* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
    // cpu cache缓存的备用的slab列表，同样使用slab结构体表示
    // 当本地cpu缓存的slab中没有空闲对象时，内核会从partial列表中的slab查找空闲对象
	struct slab *partial;	/* Partially allocated slabs */
#endif
	local_lock_t lock;	/* Protects the fields above */
#ifdef CONFIG_SLUB_STATS
    // 记录slab分配对象的一些状态信息
	unsigned int stat[NR_SLUB_STAT_ITEMS];
#endif
};
```

### 3.5.3、struct slab

```c
// mm/slab.h
/* Reuses the bits in struct page */
struct slab {
	unsigned long __page_flags;

	struct kmem_cache *slab_cache;
	union {
		struct {
			union {
				struct list_head slab_list;
#ifdef CONFIG_SLUB_CPU_PARTIAL
				struct {
					struct slab *next;
					int slabs;	/* Nr of slabs left */
				};
#endif
			};
			/* Double-word boundary */
			union {
				struct {
					void *freelist;		/* first free object */
					union {
						unsigned long counters;
						struct {
							unsigned inuse:16;
							unsigned objects:15;
							/*
							 * If slab debugging is enabled then the
							 * frozen bit can be reused to indicate
							 * that the slab was corrupted
							 */
							unsigned frozen:1;
						};
					};
				};
#ifdef system_has_freelist_aba
				freelist_aba_t freelist_counter;
#endif
			};
		};
		struct rcu_head rcu_head;
	};

	unsigned int __page_type;
	atomic_t __page_refcount;
#ifdef CONFIG_SLAB_OBJ_EXT
	unsigned long obj_exts;
#endif
};
```

struct slab被设计为与struct page共享相同的内存区域。这是通过union和位字段重用实现的：

```c
/* mm/slab.h */
struct slab {
    unsigned long __page_flags;      // 重用page->flags
    // ... 其他slab特定字段
    unsigned int __page_type;        // 重用page->page_type
    atomic_t __page_refcount;        // 重用page->_refcount
    // ...
};
```

内核提供了专门的转换函数来实现`struct page`和`struct slab`之间的转换：

```c
// mm/slab.h
/**
 * folio_slab - Converts from folio to slab.
 * @folio: The folio.
 *
 * Currently struct slab is a different representation of a folio where
 * folio_test_slab() is true.
 *
 * Return: The slab which contains this folio.
 */
#define folio_slab(folio)	(_Generic((folio),			\
	const struct folio *:	(const struct slab *)(folio),		\
	struct folio *:		(struct slab *)(folio)))

/**
 * slab_folio - The folio allocated for a slab
 * @s: The slab.
 *
 * Slabs are allocated as folios that contain the individual objects and are
 * using some fields in the first struct page of the folio - those fields are
 * now accessed by struct slab. It is occasionally necessary to convert back to
 * a folio in order to communicate with the rest of the mm.  Please use this
 * helper function instead of casting yourself, as the implementation may change
 * in the future.
 */
#define slab_folio(s)		(_Generic((s),				\
	const struct slab *:	(const struct folio *)s,		\
	struct slab *:		(struct folio *)s))

/**
 * page_slab - Converts from first struct page to slab.
 * @p: The first (either head of compound or single) page of slab.
 *
 * A temporary wrapper to convert struct page to struct slab in situations where
 * we know the page is the compound head, or single order-0 page.
 *
 * Long-term ideally everything would work with struct slab directly or go
 * through folio to struct slab.
 *
 * Return: The slab which contains this page
 */
#define page_slab(p)		(_Generic((p),				\
	const struct page *:	(const struct slab *)(p),		\
	struct page *:		(struct slab *)(p)))

/**
 * slab_page - The first struct page allocated for a slab
 * @s: The slab.
 *
 * A convenience wrapper for converting slab to the first struct page of the
 * underlying folio, to communicate with code not yet converted to folio or
 * struct slab.
 */
#define slab_page(s) folio_page(slab_folio(s), 0)
```

### 3.5.4、struct kmem_cache_node

```c
// mm/slub.c
/*
 * The slab lists for all objects.
 */
struct kmem_cache_node {
	spinlock_t list_lock;
    // 该 node 节点中缓存的salb个数
	unsigned long nr_partial;
    // 该链表用于组织串联node节点中缓存的slabs
    // partial链表中缓存的slab为部分空闲的（slab中的对象部分被分配出去）
	struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG	// slab_debug调试功能
    // slab的个数
	atomic_long_t nr_slabs;
    // 该node节点中缓存的所有slab包含的对象总和
	atomic_long_t total_objects;
    // full链表中包含的slab全部是已经被分配完毕的full slab
	struct list_head full;
#endif
};
```

## 3.6、slab api

创建kmem_cache、创建slab、申请slab内存object、释放slab内存object：

![image-20260219180040042](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219180040042.png)

创建kmem_cache时传入的flag：

![image-20260219180112551](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260219180112551.png)

## 3.7、测试

1）源文件

/home/xxx/test/slab_test.c

创建了kmem_cache之后的内存分配流程：

```bash
调用 kmem_cache_alloc()
    ↓
1. 检查本地CPU缓存（per-CPU cache）中是否有空闲对象
    ↓
2. 如果有，直接返回（最快路径）
    ↓
3. 如果没有，检查SLAB部分空闲链表
    ↓
4. 如果还没有，从伙伴系统分配新页面创建新SLAB
    ↓
5. 返回对象地址
```

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/slab.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/string.h>

#define CACHE_NAME "my_custom_slub_cache"
#define OBJ_COUNT 10

// 自定义结构体
struct my_data {
    int id;
    char name[32];
    unsigned long timestamp;
    void *buffer;
    struct list_head list;
};

static struct kmem_cache *my_cache = NULL;
static struct my_data *objects[OBJ_COUNT];

// 显示缓存信息到/proc
static int my_cache_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Custom SLUB Cache Information:\n");
    seq_printf(m, "==============================\n");
    
    if (my_cache) {
        seq_printf(m, "Cache name: %s\n", CACHE_NAME);
        seq_printf(m, "Object size: %u bytes\n", kmem_cache_size(my_cache));
        seq_printf(m, "Cache size: %u bytes\n", kmem_cache_size(my_cache) * OBJ_COUNT);
        seq_printf(m, "Objects allocated: %d\n", OBJ_COUNT);
        seq_printf(m, "\n");
        
        // 显示已分配对象信息
        for (int i = 0; i < OBJ_COUNT; i++) {
            if (objects[i]) {
                seq_printf(m, "Object %d: addr=%p, id=%d, name=%s, timestamp=%lu\n",
                          i, objects[i], objects[i]->id, 
                          objects[i]->name, objects[i]->timestamp);
            }
        }
    } else {
        seq_printf(m, "Cache not initialized\n");
    }
    
    return 0;
}

static int my_cache_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, my_cache_proc_show, NULL);
}

static const struct proc_ops my_cache_proc_fops = {
    .proc_open = my_cache_proc_open,
    .proc_read = seq_read,
    .proc_lseek = seq_lseek,
    .proc_release = single_release,
};

static int __init custom_slub_init(void)
{
    int i;
    struct proc_dir_entry *proc_entry;
    
    printk(KERN_INFO "Custom SLUB Cache Module Loading...\n");
    
    // 1. 创建SLUB缓存
    my_cache = kmem_cache_create(
        CACHE_NAME,                     // 缓存名称
        sizeof(struct my_data),         // 对象大小
        0,                              // 对齐方式（默认）
        SLAB_HWCACHE_ALIGN |            // 硬件缓存对齐
        SLAB_POISON |                   // 毒化，便于调试
        SLAB_RED_ZONE |                 // 红区检测
        SLAB_TRACE,                     // 跟踪分配
        NULL                            // 构造函数
    );
    
    if (!my_cache) {
        printk(KERN_ERR "Failed to create SLUB cache: %s\n", CACHE_NAME);
        return -ENOMEM;
    }
    
    printk(KERN_INFO "Successfully created SLUB cache: %s\n", CACHE_NAME);
    printk(KERN_INFO "Object size: %u bytes\n", kmem_cache_size(my_cache));
    
    // 2. 分配多个对象
    for (i = 0; i < OBJ_COUNT; i++) {
        objects[i] = kmem_cache_alloc(my_cache, GFP_KERNEL);
        if (!objects[i]) {
            printk(KERN_ERR "Failed to allocate object %d\n", i);
            // 释放之前分配的对象
            for (int j = 0; j < i; j++) {
                kmem_cache_free(my_cache, objects[j]);
            }
            kmem_cache_destroy(my_cache);
            return -ENOMEM;
        }
        
        // 初始化对象
        objects[i]->id = i;
        snprintf(objects[i]->name, sizeof(objects[i]->name), "obj_%03d", i);
        objects[i]->timestamp = jiffies;
        objects[i]->buffer = kmalloc(64, GFP_KERNEL);  // 分配额外内存
        INIT_LIST_HEAD(&objects[i]->list);
        
        printk(KERN_DEBUG "Allocated object %d at %p (cache %s)\n", 
               i, objects[i], CACHE_NAME);
    }
    
    printk(KERN_INFO "Allocated %d objects from cache: %s\n", OBJ_COUNT, CACHE_NAME);
    
    // 3. 创建proc文件系统条目
    proc_entry = proc_create("my_slub_cache", 0444, NULL, &my_cache_proc_fops);
    if (!proc_entry) {
        printk(KERN_WARNING "Failed to create /proc entry\n");
    } else {
        printk(KERN_INFO "Created /proc/my_slub_cache\n");
    }
    
    // 4. 显示/proc/slabinfo查看提示
    printk(KERN_INFO "\n");
    printk(KERN_INFO "=============================================\n");
    printk(KERN_INFO "To check in /proc/slabinfo, run:\n");
    printk(KERN_INFO "  sudo cat /proc/slabinfo | grep %s\n", CACHE_NAME);
    printk(KERN_INFO "=============================================\n");
    
    return 0;
}

static void __exit custom_slub_exit(void)
{
    int i;
    
    printk(KERN_INFO "Custom SLUB Cache Module Unloading...\n");
    
    // 1. 释放所有对象
    for (i = 0; i < OBJ_COUNT; i++) {
        if (objects[i]) {
            kfree(objects[i]->buffer);  // 释放额外内存
            kmem_cache_free(my_cache, objects[i]);
            printk(KERN_DEBUG "Freed object %d\n", i);
        }
    }
    
    // 2. 删除proc条目
    remove_proc_entry("my_slub_cache", NULL);
    
    // 3. 销毁缓存
    if (my_cache) {
        kmem_cache_destroy(my_cache);
        printk(KERN_INFO "Destroyed SLUB cache: %s\n", CACHE_NAME);
    }
    
    printk(KERN_INFO "Module unloaded. Check /proc/slabinfo again.\n");
}

module_init(custom_slub_init);
module_exit(custom_slub_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Test Developer");
MODULE_DESCRIPTION("Custom SLUB cache test module");
MODULE_VERSION("1.0");
```

2）编写Makefile

/home/xxx/test/Makefile

```makefile
KERNEL_DIR := /home/xxx/linux_old1
# 编译两个模块
obj-m := buddy_test.o slab_test.o

ARCH := arm64
CROSS_COMPILE := aarch64-linux-gnu-

all:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean
	rm -rf *.ko *.o *.mod* *.cmd modules.order Module.symvers .tmp_versions

# 单独编译slab_test的快捷方式
slab:
	$(MAKE) obj-m=slab_test.o

buddy:
	$(MAKE) obj-m=buddy_test.o
```

3）cd /home/xxx/test/

同2.3.3

4）将ko文件copy到文件系统的目录中并制作新的文件系统

同2.3.3

5）启动qemu，并加载ko文件

同2.3.3

执行insmod之前：/proc/slabinfo中没有自定义的slab缓存

```bash
/tmp #
/tmp # ls -l
total 1084
-rw-r--r--    1 0        0            47640 Feb 19 10:11 buddy_test.ko
-rwxr-xr-x    1 0        0           354264 Feb 19 10:11 fair.c
-rwxr-xr-x    1 0        0           633736 Feb 19 10:11 hello_world_aarch64_static
-rw-r--r--    1 0        0            52208 Feb 19 10:11 slab_test.ko
drwxr-xr-x    2 0        0             4096 Feb 19 10:11 xxx
drwxr-xr-x    2 0        0             4096 Feb 19 10:11 xxx2
/tmp # 
/tmp # cat /proc/slabinfo | grep my_custom_slub_cache
/tmp # 
```

执行insmod之后：/proc/slabinfo中出现自定义的slab缓存

```bash
...
[   57.747191] Allocated object 9 at 00000000e5356be2 (cache my_custom_slub_cache)
[   57.755265] Allocated 10 objects from cache: my_custom_slub_cache
[   57.755706] Created /proc/my_slub_cache
[   57.755906] 
[   57.756013] =============================================
[   57.756463] To check in /proc/slabinfo, run:
[   57.756666]   sudo cat /proc/slabinfo | grep my_custom_slub_cache
[   57.757018] =============================================
/tmp # 
/tmp # cat /proc/slabinfo 
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
my_custom_slub_cache     10     21    192   21    1 : tunables    0    0    0 : slabdata      1      1      0
ext4_groupinfo_4k     22     22    184   22    1 : tunables    0    0    0 : slabdata      1      1      0
...
```

# 4、kmalloc分配器

## 4.1、实现方案

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220154151286.png" alt="image-20260220154151286" style="zoom: 50%;" />

```c
// include/linux/slab.h
static __always_inline __alloc_size(1) void *kmalloc_noprof(size_t size, gfp_t flags)
{
	/* 1 编译时优化：如果size是编译时常量，使用快速路径 */
	if (__builtin_constant_p(size) && size) {
		unsigned int index;

		/* 1.1 如果请求大小超过最大缓存大小，使用buddy分配器的大内存分配路径 */
		if (size > KMALLOC_MAX_CACHE_SIZE)
			return __kmalloc_large_noprof(size, flags);

		/* 1.2 获取适合该大小的缓存数组索引（对2的幂进行对齐） */
		index = kmalloc_index(size);
		/* 1.3 从对应类型和大小的缓存中分配内存 */
        // kmalloc_type(flag); 根据flag获取缓存类型索引
        // kmalloc_caches[type][index]；获取对应的slab缓存
        // kmem_cache_alloc_trace；从缓存分配器获取内存并跟踪
		return __kmalloc_cache_noprof(
				kmalloc_caches[kmalloc_type(flags, _RET_IP_)][index],
				flags, size);
	}
	/* 2 运行时路径：处理非常量大小的分配 */
	return __kmalloc_noprof(size, flags);
}
#define kmalloc(...)				alloc_hooks(kmalloc_noprof(__VA_ARGS__))
```

## 4.2、kmalloc尺寸

```bash
~ # 
~ # cat /proc/slabinfo | grep kmalloc*
kmalloc-cg-8k          0      0   8192    4    8 : tunables    0    0    0 : slabdata      0      0      0
kmalloc-cg-4k         24     24   4096    8    8 : tunables    0    0    0 : slabdata      3      3      0
...
kmalloc-cg-192       462    462    192   21    1 : tunables    0    0    0 : slabdata     22     22      0
kmalloc-cg-96        116    126     96   42    1 : tunables    0    0    0 : slabdata      3      3      0
...
dma-kmalloc-8          0      0      8  512    1 : tunables    0    0    0 : slabdata      0      0      0
dma-kmalloc-192        0      0    192   21    1 : tunables    0    0    0 : slabdata      0      0      0
dma-kmalloc-96         0      0     96   42    1 : tunables    0    0    0 : slabdata      0      0      0
...
kmalloc-rcl-8          0      0      8  512    1 : tunables    0    0    0 : slabdata      0      0      0
kmalloc-rcl-192        0      0    192   21    1 : tunables    0    0    0 : slabdata      0      0      0
kmalloc-rcl-96         0      0     96   42    1 : tunables    0    0    0 : slabdata      0      0      0
kmalloc-16          2929   3072     16  256    1 : tunables    0    0    0 : slabdata     12     12      0
...
kmalloc-8           5468   6144      8  512    1 : tunables    0    0    0 : slabdata     12     12      0
kmalloc-192         1779   1785    192   21    1 : tunables    0    0    0 : slabdata     85     85      0
kmalloc-96           671    882     96   42    1 : tunables    0    0    0 : slabdata     21     21      0
~ # 
```

![image-20260220160246051](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220160246051.png)

```c
// mm/slab.h
/* A table of kmalloc cache names and sizes */
extern const struct kmalloc_info_struct {
    // size 用于指定该slab cache中所管理的通用内存块尺寸
	const char *name[NR_KMALLOC_TYPES];
    // name为该通用slab cache的名称，形式为 kmalloc-name（单位是字节）
	unsigned int size;
} kmalloc_info[];
```

/proc/slabinfo当中的这些kmalloc-xxx定位在：

```c
// mm/slab_common.c

#ifdef CONFIG_ZONE_DMA
#define KMALLOC_DMA_NAME(sz)	.name[KMALLOC_DMA] = "dma-kmalloc-" #sz,
#else
#define KMALLOC_DMA_NAME(sz)
#endif

#ifdef CONFIG_MEMCG
#define KMALLOC_CGROUP_NAME(sz)	.name[KMALLOC_CGROUP] = "kmalloc-cg-" #sz,
#else
#define KMALLOC_CGROUP_NAME(sz)
#endif

#ifndef CONFIG_SLUB_TINY
#define KMALLOC_RCL_NAME(sz)	.name[KMALLOC_RECLAIM] = "kmalloc-rcl-" #sz,
#else
#define KMALLOC_RCL_NAME(sz)
#endif

#define INIT_KMALLOC_INFO(__size, __short_size)			\
{								\
	.name[KMALLOC_NORMAL]  = "kmalloc-" #__short_size,	\
	KMALLOC_RCL_NAME(__short_size)				\
	KMALLOC_CGROUP_NAME(__short_size)			\
	KMALLOC_DMA_NAME(__short_size)				\
	KMALLOC_RANDOM_NAME(RANDOM_KMALLOC_CACHES_NR, __short_size)	\
	.size = __size,						\
}

/*
 * kmalloc_info[] is to make slab_debug=,kmalloc-xx option work at boot time.
 * kmalloc_index() supports up to 2^21=2MB, so the final entry of the table is
 * kmalloc-2M.
 */
const struct kmalloc_info_struct kmalloc_info[] __initconst = {
	INIT_KMALLOC_INFO(0, 0),
	INIT_KMALLOC_INFO(96, 96),
	INIT_KMALLOC_INFO(192, 192),
	INIT_KMALLOC_INFO(8, 8),
	INIT_KMALLOC_INFO(16, 16),
	INIT_KMALLOC_INFO(32, 32),
	INIT_KMALLOC_INFO(64, 64),
	INIT_KMALLOC_INFO(128, 128),
	INIT_KMALLOC_INFO(256, 256),
	INIT_KMALLOC_INFO(512, 512),
	INIT_KMALLOC_INFO(1024, 1k),
	INIT_KMALLOC_INFO(2048, 2k),
	INIT_KMALLOC_INFO(4096, 4k),
	INIT_KMALLOC_INFO(8192, 8k),
	INIT_KMALLOC_INFO(16384, 16k),
	INIT_KMALLOC_INFO(32768, 32k),
	INIT_KMALLOC_INFO(65536, 64k),
	INIT_KMALLOC_INFO(131072, 128k),
	INIT_KMALLOC_INFO(262144, 256k),
	INIT_KMALLOC_INFO(524288, 512k),
	INIT_KMALLOC_INFO(1048576, 1M),
	INIT_KMALLOC_INFO(2097152, 2M)
};
```

其中：

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220161530850.png" alt="image-20260220161530850" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220161657934.png" alt="image-20260220161657934" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220161923138.png" alt="image-20260220161923138" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220162027143.png" alt="image-20260220162027143" style="zoom:50%;" />

## 4.3、来自哪些Zone

linux内核中定义的Zone类型如下：

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

在我的qemu中有如下Zone：

```bash
~ # 
~ # cat /proc/zoneinfo | grep Node
Node 0, zone      DMA
Node 0, zone    DMA32
Node 0, zone   Normal
Node 0, zone  Movable
Node 1, zone      DMA
Node 1, zone    DMA32
Node 1, zone   Normal
Node 1, zone  Movable
~ # 
~ # 
```

即开了 ZONE_DMA、ZONE_DMA32；

ZONE_NORMAL、ZONE_MOVABLE是默认的。

![image-20260220162710965](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220162710965.png)

```c
// include/linux/slab.h
    *
 * Whenever changing this, take care of that kmalloc_type() and
 * create_kmalloc_caches() still work as intended.
 *
 * KMALLOC_NORMAL can contain only unaccounted objects whereas KMALLOC_CGROUP
 * is for accounted but unreclaimable and non-dma objects. All the other
 * kmem caches can have both accounted and unaccounted objects.
 */
enum kmalloc_cache_type {
	KMALLOC_NORMAL = 0,
#ifndef CONFIG_ZONE_DMA
	KMALLOC_DMA = KMALLOC_NORMAL,
#endif
#ifndef CONFIG_MEMCG
	KMALLOC_CGROUP = KMALLOC_NORMAL,
#endif
	KMALLOC_RANDOM_START = KMALLOC_NORMAL,
	KMALLOC_RANDOM_END = KMALLOC_RANDOM_START + RANDOM_KMALLOC_CACHES_NR,
#ifdef CONFIG_SLUB_TINY
	KMALLOC_RECLAIM = KMALLOC_NORMAL,
#else
	KMALLOC_RECLAIM,
#endif
#ifdef CONFIG_ZONE_DMA
	KMALLOC_DMA,
#endif
#ifdef CONFIG_MEMCG
	KMALLOC_CGROUP,
#endif
	NR_KMALLOC_TYPES
};
```

```c
// include/linux/slab.h
typedef struct kmem_cache * kmem_buckets[KMALLOC_SHIFT_HIGH + 1];

// mm/slab_common.c
kmem_buckets kmalloc_caches[NR_KMALLOC_TYPES] __ro_after_init =
{ /* initialization for https://llvm.org/pr42570 */ };
EXPORT_SYMBOL(kmalloc_caches);
```

数组展开之后为：

```c
struct kmem_cache *kmalloc_caches[NR_KMALLOC_TYPES][KMALLOC_SHIFT_HIGH + 1];
```

![image-20260220163012613](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260220163012613.png)

## 4.4、测试

1）源文件

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/slab.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/string.h>
#include <linux/time.h>

#define MODULE_NAME "kmalloc_test"
#define TEST_COUNT 8

// 测试配置结构体
struct kmalloc_test_cfg {
    size_t size;
    gfp_t flags;
    const char *flag_name;
    const char *desc;
};

// 测试结果结构体
struct kmalloc_result {
    void *addr;
    size_t size;
    gfp_t flags;
    phys_addr_t phys_addr;
    struct timespec64 alloc_time;
};

// 测试配置数组
static struct kmalloc_test_cfg test_configs[TEST_COUNT] = {
    { 32,      GFP_KERNEL,                     "GFP_KERNEL",                     "32 bytes, kernel context" },
    { 128,     GFP_KERNEL | __GFP_ZERO,        "GFP_KERNEL|__GFP_ZERO",          "128 bytes, zeroed" },
    { 512,     GFP_KERNEL,                     "GFP_KERNEL",                     "512 bytes, normal allocation" },
    { 2048,    GFP_KERNEL | GFP_ATOMIC,        "GFP_KERNEL|GFP_ATOMIC",          "2KB, atomic allocation" },
    { 4096,    GFP_KERNEL,                     "GFP_KERNEL",                     "4KB, single page" },
    { 8192,    GFP_KERNEL | __GFP_ZERO,        "GFP_KERNEL|__GFP_ZERO",          "8KB, multiple pages" },
    { 16384,   GFP_KERNEL,                     "GFP_KERNEL",                     "16KB, large allocation" },
    { 64,      GFP_KERNEL | GFP_DMA,           "GFP_KERNEL|GFP_DMA",             "64 bytes, DMA memory" },
};

// 修复：添加变量名 test_results
static struct kmalloc_result test_results[TEST_COUNT];
static int test_passed = 0;
static int test_failed = 0;

// 显示测试结果到 /proc
static int kmalloc_test_proc_show(struct seq_file *m, void *v)
{
    int i;
    
    seq_printf(m, "============================================\n");
    seq_printf(m, "   kmalloc() Test Results\n");
    seq_printf(m, "============================================\n");
    seq_printf(m, "Module: %s\n", MODULE_NAME);
    seq_printf(m, "Test Count: %d\n", TEST_COUNT);
    seq_printf(m, "Passed: %d\n", test_passed);
    seq_printf(m, "Failed: %d\n", test_failed);
    seq_printf(m, "============================================\n\n");
    
    seq_printf(m, "Detailed Test Results:\n");
    seq_printf(m, "----------------------\n");
    
    for (i = 0; i < TEST_COUNT; i++) {
        struct kmalloc_result *res = &test_results[i];
        struct kmalloc_test_cfg *cfg = &test_configs[i];
        
        seq_printf(m, "\nTest %d: %s\n", i + 1, cfg->desc);
        seq_printf(m, "  Size: %zu bytes\n", cfg->size);
        seq_printf(m, "  Flags: %s\n", cfg->flag_name);
        
        if (res->addr) {
            seq_printf(m, "  Status: SUCCESS ✓\n");
            seq_printf(m, "  Virtual Addr: %p\n", res->addr);
            seq_printf(m, "  Physical Addr: 0x%llx\n", (unsigned long long)res->phys_addr);
            seq_printf(m, "  Allocation Time: %lld.%09ld\n", 
                       (long long)res->alloc_time.tv_sec, res->alloc_time.tv_nsec);
            
            // 显示内存内容（前32字节）
            seq_printf(m, "  Memory Content (first 32 bytes): %*ph\n", 
                       32, res->addr);
        } else {
            seq_printf(m, "  Status: FAILED ✗\n");
        }
    }
    
    seq_printf(m, "\n============================================\n");
    seq_printf(m, "Memory Usage Summary\n");
    seq_printf(m, "============================================\n");
    
    size_t total_allocated = 0;
    for (i = 0; i < TEST_COUNT; i++) {
        if (test_results[i].addr) {
            total_allocated += test_results[i].size;
        }
    }
    
    // 修复：避免浮点运算，使用整数除法
    unsigned long total_kb = total_allocated / 1024;
    unsigned long remainder = total_allocated % 1024;
    
    seq_printf(m, "Total Memory Allocated: %zu bytes (%lu.%03lu KB)\n", 
               total_allocated, total_kb, (remainder * 1000) / 1024);
    
    return 0;
}

static int kmalloc_test_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, kmalloc_test_proc_show, NULL);
}

static const struct proc_ops kmalloc_test_proc_fops = {
    .proc_open = kmalloc_test_proc_open,
    .proc_read = seq_read,
    .proc_lseek = seq_lseek,
    .proc_release = single_release,
};

// 测试内存写入和读取
static int test_memory_access(void *addr, size_t size)
{
    char *ptr = (char *)addr;
    size_t i;
    
    // 写入测试模式
    for (i = 0; i < size && i < 256; i++) {
        ptr[i] = (char)(i & 0xFF);
    }
    
    // 验证读取
    for (i = 0; i < size && i < 256; i++) {
        if (ptr[i] != (char)(i & 0xFF)) {
            return -1;
        }
    }
    
    return 0;
}

static int __init kmalloc_test_init(void)
{
    int i;
    struct proc_dir_entry *proc_entry;
    
    printk(KERN_INFO "============================================\n");
    printk(KERN_INFO "kmalloc() Test Module Loading...\n");
    printk(KERN_INFO "============================================\n");
    
    // 运行测试
    for (i = 0; i < TEST_COUNT; i++) {
        struct kmalloc_test_cfg *cfg = &test_configs[i];
        struct kmalloc_result *res = &test_results[i];
        
        printk(KERN_INFO "\nTest %d: %s\n", i + 1, cfg->desc);
        printk(KERN_INFO "  Requested size: %zu bytes\n", cfg->size);
        printk(KERN_INFO "  Flags: %s\n", cfg->flag_name);
        
        // 记录开始时间
        ktime_get_real_ts64(&res->alloc_time);
        
        // 执行分配
        res->addr = kmalloc(cfg->size, cfg->flags);
        res->size = cfg->size;
        res->flags = cfg->flags;
        
        if (!res->addr) {
            printk(KERN_ERR "  ✗ FAILED: kmalloc returned NULL\n");
            test_failed++;
            continue;
        }
        
        // 获取物理地址
        res->phys_addr = virt_to_phys(res->addr);
        
        printk(KERN_INFO "  ✓ SUCCESS: Allocated at %p (phys: 0x%llx)\n",
               res->addr, (unsigned long long)res->phys_addr);
        
        // 如果是__GFP_ZERO标志，验证内存是否清零
        if (cfg->flags & __GFP_ZERO) {
            char *ptr = (char *)res->addr;
            int zeroed = 1;
            size_t j;
            
            for (j = 0; j < cfg->size && j < 64; j++) {
                if (ptr[j] != 0) {
                    zeroed = 0;
                    break;
                }
            }
            
            if (zeroed) {
                printk(KERN_INFO "  ✓ Memory is zeroed as expected\n");
            } else {
                printk(KERN_WARNING "  ⚠ Memory is not zeroed (first 64 bytes)\n");
            }
        }
        
        // 测试内存访问
        if (test_memory_access(res->addr, cfg->size) == 0) {
            printk(KERN_INFO "  ✓ Memory read/write test passed\n");
        } else {
            printk(KERN_ERR "  ✗ Memory test failed\n");
        }
        
        test_passed++;
    }
    
    // 创建 /proc 接口
    proc_entry = proc_create("kmalloc_test", 0444, NULL, &kmalloc_test_proc_fops);
    if (!proc_entry) {
        printk(KERN_WARNING "Failed to create /proc/kmalloc_test\n");
    } else {
        printk(KERN_INFO "\nCreated /proc/kmalloc_test\n");
    }
    
    // 显示总结
    printk(KERN_INFO "\n============================================\n");
    printk(KERN_INFO "kmalloc() Test Summary:\n");
    printk(KERN_INFO "  Total tests: %d\n", TEST_COUNT);
    printk(KERN_INFO "  Passed: %d\n", test_passed);
    printk(KERN_INFO "  Failed: %d\n", test_failed);
    
    // 计算总内存使用量（整数运算）
    size_t total_allocated = 0;
    for (i = 0; i < TEST_COUNT; i++) {
        if (test_results[i].addr) {
            total_allocated += test_results[i].size;
        }
    }
    
    unsigned long total_kb = total_allocated / 1024;
    unsigned long remainder = total_allocated % 1024;
    
    printk(KERN_INFO "Total memory allocated: %zu bytes (%lu.%03lu KB)\n",
           total_allocated, total_kb, (remainder * 1000) / 1024);
    
    printk(KERN_INFO "============================================\n");
    printk(KERN_INFO "To view detailed results: cat /proc/kmalloc_test\n");
    
    return 0;
}

static void __exit kmalloc_test_exit(void)
{
    int i;
    size_t total_freed = 0;
    
    printk(KERN_INFO "\n============================================\n");
    printk(KERN_INFO "kmalloc() Test Module Unloading...\n");
    printk(KERN_INFO "============================================\n");
    
    // 释放所有分配的内存
    for (i = 0; i < TEST_COUNT; i++) {
        if (test_results[i].addr) {
            printk(KERN_INFO "Freeing test %d: %p (%zu bytes)\n",
                   i + 1, test_results[i].addr, test_results[i].size);
            kfree(test_results[i].addr);
            total_freed += test_results[i].size;
            test_results[i].addr = NULL;
        }
    }
    
    // 修复：避免浮点运算，使用整数运算
    unsigned long total_kb = total_freed / 1024;
    unsigned long remainder = total_freed % 1024;
    
    printk(KERN_INFO "Total memory freed: %zu bytes (%lu.%03lu KB)\n",
           total_freed, total_kb, (remainder * 1000) / 1024);
    
    // 删除 proc 条目
    remove_proc_entry("kmalloc_test", NULL);
    
    printk(KERN_INFO "Module unloaded successfully\n");
}

module_init(kmalloc_test_init);
module_exit(kmalloc_test_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Test Developer");
MODULE_DESCRIPTION("kmalloc() test module for Linux kernel memory allocation");
MODULE_VERSION("1.0");

```

2）Makefile

```makefile
KERNEL_DIR := /home/xxx/linux_old1

# 编译所有测试模块
obj-m := buddy_test.o slab_test.o kmalloc_test.o

ARCH := arm64
CROSS_COMPILE := aarch64-linux-gnu-

all:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean
	rm -rf *.ko *.o *.mod* *.cmd modules.order Module.symvers .tmp_versions

# 快捷方式
buddy:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) obj-m=buddy_test.o

slab:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) obj-m=slab_test.o

kmalloc:
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) obj-m=kmalloc_test.o

help:
	@echo "Usage:"
	@echo "  make all     - Build all test modules"
	@echo "  make buddy   - Build buddy_test.ko only"
	@echo "  make slab    - Build slab_test.ko only"
	@echo "  make kmalloc - Build kmalloc_test.ko only"
	@echo "  make clean   - Clean all build files"

```

3）编译ko、制作文件系统

同2.3.3

4）执行

```bash
/tmp # ls
buddy_test.ko               slab_test.ko
fair.c                      xxx
hello_world_aarch64_static  xxx2
kmalloc_test.ko
/tmp # 
/tmp # insmod kmalloc_test.ko 
[   14.536711] kmalloc_test: loading out-of-tree module taints kernel.
[   14.546252] ============================================
[   14.547517] kmalloc() Test Module Loading...
[   14.548104] ============================================
[   14.548622] 
[   14.548622] Test 1: 32 bytes, kernel context
[   14.548988]   Requested size: 32 bytes
[   14.549270]   Flags: GFP_KERNEL
[   14.549640]   ✓ SUCCESS: Allocated at 00000000e834e9bb (phys: 0x804b20a0)
[   14.550536]   ✓ Memory read/write test passed
[   14.551431] 
[   14.551431] Test 2: 128 bytes, zeroed
[   14.552349]   Requested size: 128 bytes
[   14.552579]   Flags: GFP_KERNEL|__GFP_ZERO
[   14.552756]   ✓ SUCCESS: Allocated at 0000000089ba772e (phys: 0x8041d780)
[   14.553368]   ✓ Memory is zeroed as expected
[   14.553767]   ✓ Memory read/write test passed
[   14.554178] 
[   14.554178] Test 3: 512 bytes, normal allocation
[   14.554875]   Requested size: 512 bytes
[   14.555187]   Flags: GFP_KERNEL
[   14.555534]   ✓ SUCCESS: Allocated at 00000000a1c09460 (phys: 0x8127ca00)
[   14.556131]   ✓ Memory read/write test passed
[   14.556498] 
[   14.556498] Test 4: 2KB, atomic allocation
[   14.557144]   Requested size: 2048 bytes
[   14.557372]   Flags: GFP_KERNEL|GFP_ATOMIC
[   14.557660]   ✓ SUCCESS: Allocated at 00000000523c35f6 (phys: 0x8151b800)
[   14.558192]   ✓ Memory read/write test passed
[   14.558501] 
[   14.558501] Test 5: 4KB, single page
[   14.559215]   Requested size: 4096 bytes
[   14.559866]   Flags: GFP_KERNEL
[   14.560502]   ✓ SUCCESS: Allocated at 0000000079823f4f (phys: 0x804c4000)
[   14.561098]   ✓ Memory read/write test passed
[   14.561434] 
[   14.561434] Test 6: 8KB, multiple pages
[   14.561759]   Requested size: 8192 bytes
[   14.561943]   Flags: GFP_KERNEL|__GFP_ZERO
[   14.562161]   ✓ SUCCESS: Allocated at 00000000c807d619 (phys: 0x44918000)
[   14.562552]   ✓ Memory is zeroed as expected
[   14.563183]   ✓ Memory read/write test passed
[   14.563463] 
[   14.563463] Test 7: 16KB, large allocation
[   14.563813]   Requested size: 16384 bytes
[   14.564054]   Flags: GFP_KERNEL
[   14.564400]   ✓ SUCCESS: Allocated at 000000005440d430 (phys: 0x44328000)
[   14.565099]   ✓ Memory read/write test passed
[   14.565624] 
[   14.565624] Test 8: 64 bytes, DMA memory
[   14.566078]   Requested size: 64 bytes
[   14.566478]   Flags: GFP_KERNEL|GFP_DMA
[   14.567318]   ✓ SUCCESS: Allocated at 000000002b219480 (phys: 0x440a2000)
[   14.568758]   ✓ Memory read/write test passed
[   14.569208] 
[   14.569208] Created /proc/kmalloc_test
[   14.569705] 
[   14.569705] ============================================
[   14.570098] kmalloc() Test Summary:
[   14.570610]   Total tests: 8
[   14.571272]   Passed: 8
[   14.571549]   Failed: 0
[   14.571788] Total memory allocated: 31456 bytes (30.718 KB)
[   14.572242] ============================================
[   14.572672] To view detailed results: cat /proc/kmalloc_test
/tmp # 
/tmp # rmmod kmalloc_test.ko 
[  354.909420] 
[  354.909420] ============================================
[  354.909955] kmalloc() Test Module Unloading...
[  354.910424] ============================================
[  354.911397] Freeing test 1: 00000000e834e9bb (32 bytes)
[  354.911902] Freeing test 2: 0000000089ba772e (128 bytes)
[  354.912140] Freeing test 3: 00000000a1c09460 (512 bytes)
[  354.912685] Freeing test 4: 00000000523c35f6 (2048 bytes)
[  354.913118] Freeing test 5: 0000000079823f4f (4096 bytes)
[  354.913502] Freeing test 6: 00000000c807d619 (8192 bytes)
[  354.913858] Freeing test 7: 000000005440d430 (16384 bytes)
[  354.914491] Freeing test 8: 000000002b219480 (64 bytes)
[  354.914873] Total memory freed: 31456 bytes (30.718 KB)
[  354.915544] Module unloaded successfully
/tmp # 
```

