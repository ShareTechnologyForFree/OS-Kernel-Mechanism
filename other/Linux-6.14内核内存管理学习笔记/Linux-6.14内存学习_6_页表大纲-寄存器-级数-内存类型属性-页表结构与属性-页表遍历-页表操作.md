# 1、页表大纲

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222141344658.png" alt="image-20260222141344658" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222141403065.png" alt="image-20260222141403065" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222141414450.png" alt="image-20260222141414450" style="zoom:50%;" />

# 2、页表寄存器

## 2.1、总览

内核中VA通过两个寄存器进行转换：

* 高位地址使用 TTBR1_EL1 寄存器存储页表所在基地址
* 低位地址使用 TTBR0_EL1 寄存器存储页表所在基地址

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222141749783.png" alt="image-20260222141749783" style="zoom:50%;" />

arm v8的架构手册：https://developer.arm.com/documentation/ddi0487/latest/ （这是latest版本的地址，每个版本内容有所不同）

## 2.2、TTBR0_EL1

访问低位（linux内核表示用户空间）虚拟地址的页表基地址。

![image-20260222144233348](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222144233348.png)



## 2.3、TTBR1_EL1

访问高位（linux内核表示内核空间）虚拟地址的页表基地址。

![image-20260222144420968](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222144420968.png)



## 2.4、SCTLR_EL1

系统控制相关，其中几个寄存器控制mmu使能，cache开关。

![image-20260222145549234](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222145549234.png)

![image-20260222145555026](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222145555026.png)

## 2.5、TCR_EL1

转换表控制，粒度，虚拟空间大小。

![image-20260222150045099](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222150045099.png)

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222145901396.png" alt="image-20260222145901396" style="zoom:50%;" />

# 3、页表级数

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222151758892.png" alt="image-20260222151758892" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222151910190.png" alt="image-20260222151910190" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222152026163.png" alt="image-20260222152026163" style="zoom:50%;" />

其中页表基地址使用TTBR0还是TTBR1取决于虚拟地址的最高位bit[63]是0还是1：

* 0：使用TTRB0
* 1：使用TTBR1

# 4、内存类型与属性

## 4.1、介绍

```c
// arch/arm64/include/asm/memory.h
/*
 * Memory types available.
 *
 * IMPORTANT: MT_NORMAL must be index 0 since vm_get_page_prot() may 'or' in
 *	      the MT_NORMAL_TAGGED memory type for PROT_MTE mappings. Note
 *	      that protection_map[] only contains MT_NORMAL attributes.
 */
#define MT_NORMAL		0
#define MT_NORMAL_TAGGED	1
#define MT_NORMAL_NC		2
#define MT_DEVICE_nGnRnE	3
#define MT_DEVICE_nGnRE		4
```

Normal Memory：普通内存，如DDR，SRAM

Device Memory：设备内存，如内存映射IO寄存器

内存类型在arm v8的架构手册：https://developer.arm.com/documentation/ddi0487/latest/ 

![image-20260222152651922](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222152651922.png)

## 4.2、Normal Memory

![image-20260222153155468](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222153155468.png)

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222153207960.png" alt="image-20260222153207960" style="zoom:67%;" />

### 4.2.1、共享属性

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222153251992.png" alt="image-20260222153251992" style="zoom:67%;" />

### 4.2.2、缓存属性

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222153338196.png" alt="image-20260222153338196" style="zoom:67%;" />

## 4.3、Device Memory

![image-20260222153708250](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222153708250.png)

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222153746713.png" alt="image-20260222153746713" style="zoom:67%;" />

# 5、页表结构与属性

## 5.1、页表结构

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222154249724.png" alt="image-20260222154249724" style="zoom:50%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222154343335.png" alt="image-20260222154343335" style="zoom:67%;" />

转换表格式：

![image-20260222154849367](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222154849367.png)

## 5.2、页表属性

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222155416904.png" alt="image-20260222155416904" style="zoom: 67%;" />

### 5.2.1、Upper attributes

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222155714209.png" alt="image-20260222155714209" style="zoom: 67%;" />

### 5.2.2、Lower attributes

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222155910836.png" alt="image-20260222155910836" style="zoom: 67%;" />

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222155943203.png" alt="image-20260222155943203" style="zoom:67%;" />

# 6、页表遍历

## 6.1、TTBR寄存器选择过程

![image-20260222161225631](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222161225631.png)

这张图来自：https://developer.arm.com/documentation/ddi0487/fb/?lang=en（不是latest版本，是F-b版本，只是排版不同，内容一致）

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222161434452.png" alt="image-20260222161434452" style="zoom:50%;" />

## 6.2、遍历过程（48位4级页表）

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222161538695.png" alt="image-20260222161538695" style="zoom:50%;" />

## 6.3、缺页异常页表遍历

软件做的工作是填充页表；有时候软件也需要遍历页表，比如在缺页异常中，最终能不能找到页表项，所以也需要遍历的过程。

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222161829931.png" alt="image-20260222161829931" style="zoom:50%;" />

# 7、页表操作

## 7.1、页表项数据类型定义

```c
// arch/arm64/include/asm/pgtable-types.h
typedef u64 pteval_t;	// PTE原始类型
typedef u64 pmdval_t;	// PMD原始类型
typedef u64 pudval_t;	// PUD原始类型
typedef u64 p4dval_t;	// P4D原始类型
typedef u64 pgdval_t;	// PGD原始类型
```

## 7.2、类型转换定义

```c
// arch/arm64/include/asm/pgtable-types.h
/*
 * These are used to make use of C type-checking..
 */
typedef struct { pteval_t pte; } pte_t;
#define pte_val(x)	((x).pte)
#define __pte(x)	((pte_t) { (x) } )

#if CONFIG_PGTABLE_LEVELS > 2
typedef struct { pmdval_t pmd; } pmd_t;
#define pmd_val(x)	((x).pmd)
#define __pmd(x)	((pmd_t) { (x) } )
#endif

#if CONFIG_PGTABLE_LEVELS > 3
typedef struct { pudval_t pud; } pud_t;
#define pud_val(x)	((x).pud)
#define __pud(x)	((pud_t) { (x) } )
#endif

#if CONFIG_PGTABLE_LEVELS > 4
typedef struct { p4dval_t p4d; } p4d_t;
#define p4d_val(x)	((x).p4d)
#define __p4d(x)	((p4d_t) { (x) } )
#endif

typedef struct { pgdval_t pgd; } pgd_t;
#define pgd_val(x)	((x).pgd)
#define __pgd(x)	((pgd_t) { (x) } )
```

## 7.3、页表项大小定义

```c
// arch/arm64/include/asm/pgtable-hwdef.h
/*
 * Size mapped by an entry at level n ( -1 <= n <= 3)
 * We map (PAGE_SHIFT - 3) at all translation levels and PAGE_SHIFT bits
 * in the final page. The maximum number of translation levels supported by
 * the architecture is 5. Hence, starting at level n, we have further
 * ((4 - n) - 1) levels of translation excluding the offset within the page.
 * So, the total number of bits mapped by an entry at level n is :
 *
 *  ((4 - n) - 1) * (PAGE_SHIFT - 3) + PAGE_SHIFT
 *
 * Rearranging it a bit we get :
 *   (4 - n) * (PAGE_SHIFT - 3) + 3
 */
#define ARM64_HW_PGTABLE_LEVEL_SHIFT(n)	((PAGE_SHIFT - 3) * (4 - (n)) + 3)

#define PTRS_PER_PTE		(1 << (PAGE_SHIFT - 3))

/*
 * PMD_SHIFT determines the size a level 2 page table entry can map.
 */
#if CONFIG_PGTABLE_LEVELS > 2
#define PMD_SHIFT		ARM64_HW_PGTABLE_LEVEL_SHIFT(2)
#define PMD_SIZE		(_AC(1, UL) << PMD_SHIFT)
#define PMD_MASK		(~(PMD_SIZE-1))
#define PTRS_PER_PMD		(1 << (PAGE_SHIFT - 3))
#endif

/*
 * PUD_SHIFT determines the size a level 1 page table entry can map.
 */
#if CONFIG_PGTABLE_LEVELS > 3
#define PUD_SHIFT		ARM64_HW_PGTABLE_LEVEL_SHIFT(1)
#define PUD_SIZE		(_AC(1, UL) << PUD_SHIFT)
#define PUD_MASK		(~(PUD_SIZE-1))
#define PTRS_PER_PUD		(1 << (PAGE_SHIFT - 3))
#endif

#if CONFIG_PGTABLE_LEVELS > 4
#define P4D_SHIFT		ARM64_HW_PGTABLE_LEVEL_SHIFT(0)
#define P4D_SIZE		(_AC(1, UL) << P4D_SHIFT)
#define P4D_MASK		(~(P4D_SIZE-1))
#define PTRS_PER_P4D		(1 << (PAGE_SHIFT - 3))
#endif

/*
 * PGDIR_SHIFT determines the size a top-level page table entry can map
 * (depending on the configuration, this level can be -1, 0, 1 or 2).
 */
#define PGDIR_SHIFT		ARM64_HW_PGTABLE_LEVEL_SHIFT(4 - CONFIG_PGTABLE_LEVELS)
#define PGDIR_SIZE		(_AC(1, UL) << PGDIR_SHIFT)
#define PGDIR_MASK		(~(PGDIR_SIZE-1))
#define PTRS_PER_PGD		(1 << (VA_BITS - PGDIR_SHIFT))
```

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222162955169.png" alt="image-20260222162955169" style="zoom:50%;" />

## 7.4、获取表项索引

```c
// include/linux/pgtable.h
/*
 * A page table page can be thought of an array like this: pXd_t[PTRS_PER_PxD]
 *
 * The pXx_index() functions return the index of the entry in the page
 * table page which would control the given virtual address
 *
 * As these functions may be used by the same code for different levels of
 * the page table folding, they are always available, regardless of
 * CONFIG_PGTABLE_LEVELS value. For the folded levels they simply return 0
 * because in such cases PTRS_PER_PxD equals 1.
 */
// 获取PTE索引：取地址低9位（除去内偏移）
static inline unsigned long pte_index(unsigned long address)
{
	return (address >> PAGE_SHIFT) & (PTRS_PER_PTE - 1);
}

// 获取PMD索引：取地址中间9位
#ifndef pmd_index
static inline unsigned long pmd_index(unsigned long address)
{
	return (address >> PMD_SHIFT) & (PTRS_PER_PMD - 1);
}
#define pmd_index pmd_index
#endif

// 获取PUD索引：取地址中间9位
#ifndef pud_index
static inline unsigned long pud_index(unsigned long address)
{
	return (address >> PUD_SHIFT) & (PTRS_PER_PUD - 1);
}
#define pud_index pud_index
#endif

// 获取PGD索引：取地址高9位
#ifndef pgd_index
/* Must be a compile-time constant, so implement it as a macro */
#define pgd_index(a)  (((a) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))
#endif

```

举例说明：

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222163728527.png" alt="image-20260222163728527" style="zoom:67%;" />

## 7.5、获取表项地址

```c
// include/linux/pgtable.h
#ifndef pte_offset_kernel
static inline pte_t *pte_offset_kernel(pmd_t *pmd, unsigned long address)
{
	return (pte_t *)pmd_page_vaddr(*pmd) + pte_index(address);
}
#define pte_offset_kernel pte_offset_kernel

/* Find an entry in the second-level page table.. */
#ifndef pmd_offset
static inline pmd_t *pmd_offset(pud_t *pud, unsigned long address)
{
	return pud_pgtable(*pud) + pmd_index(address);
}
#define pmd_offset pmd_offset
#endif

#ifndef pud_offset
static inline pud_t *pud_offset(p4d_t *p4d, unsigned long address)
{
	return p4d_pgtable(*p4d) + pud_index(address);
}
#define pud_offset pud_offset
#endif

static inline pgd_t *pgd_offset_pgd(pgd_t *pgd, unsigned long address)
{
	return (pgd + pgd_index(address));
};

/*
 * a shortcut to get a pgd_t in a given mm
 */
#ifndef pgd_offset
#define pgd_offset(mm, address)		pgd_offset_pgd((mm)->pgd, (address))
#endif

/*
 * a shortcut which implies the use of the kernel's pgd, instead
 * of a process's
 */
#define pgd_offset_k(address)		pgd_offset(&init_mm, (address))
```

## 7.6、表项状态判断

```c
// arch/arm64/include/asm/pgtable-hwdef.h
/*
 * Hardware page table definitions.
 *
 * Level -1 descriptor (PGD).
 */
#define PGD_TYPE_TABLE		(_AT(pgdval_t, 3) << 0)
#define PGD_TABLE_BIT		(_AT(pgdval_t, 1) << 1)
#define PGD_TYPE_MASK		(_AT(pgdval_t, 3) << 0)
#define PGD_TABLE_AF		(_AT(pgdval_t, 1) << 10)	/* Ignored if no FEAT_HAFT */
#define PGD_TABLE_PXN		(_AT(pgdval_t, 1) << 59)
#define PGD_TABLE_UXN		(_AT(pgdval_t, 1) << 60)

/*
 * Level 0 descriptor (P4D).
 */
#define P4D_TYPE_TABLE		(_AT(p4dval_t, 3) << 0)
#define P4D_TABLE_BIT		(_AT(p4dval_t, 1) << 1)
#define P4D_TYPE_MASK		(_AT(p4dval_t, 3) << 0)
#define P4D_TYPE_SECT		(_AT(p4dval_t, 1) << 0)
#define P4D_SECT_RDONLY		(_AT(p4dval_t, 1) << 7)		/* AP[2] */
#define P4D_TABLE_AF		(_AT(p4dval_t, 1) << 10)	/* Ignored if no FEAT_HAFT */
#define P4D_TABLE_PXN		(_AT(p4dval_t, 1) << 59)
#define P4D_TABLE_UXN		(_AT(p4dval_t, 1) << 60)

/*
 * Level 1 descriptor (PUD).
 */
#define PUD_TYPE_TABLE		(_AT(pudval_t, 3) << 0)
#define PUD_TABLE_BIT		(_AT(pudval_t, 1) << 1)
#define PUD_TYPE_MASK		(_AT(pudval_t, 3) << 0)
#define PUD_TYPE_SECT		(_AT(pudval_t, 1) << 0)
#define PUD_SECT_RDONLY		(_AT(pudval_t, 1) << 7)		/* AP[2] */
#define PUD_TABLE_AF		(_AT(pudval_t, 1) << 10)	/* Ignored if no FEAT_HAFT */
#define PUD_TABLE_PXN		(_AT(pudval_t, 1) << 59)
#define PUD_TABLE_UXN		(_AT(pudval_t, 1) << 60)

/*
 * Level 2 descriptor (PMD).
 */
#define PMD_TYPE_MASK		(_AT(pmdval_t, 3) << 0)
#define PMD_TYPE_TABLE		(_AT(pmdval_t, 3) << 0)
#define PMD_TYPE_SECT		(_AT(pmdval_t, 1) << 0)
#define PMD_TABLE_BIT		(_AT(pmdval_t, 1) << 1)
#define PMD_TABLE_AF		(_AT(pmdval_t, 1) << 10)	/* Ignored if no FEAT_HAFT */
```

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222164307819.png" alt="image-20260222164307819" style="zoom: 67%;" />

## 7.7、表项设置

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222164407441.png" alt="image-20260222164407441" style="zoom:67%;" />

## 7.8、页表分配与释放

<img src="C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260222164446411.png" alt="image-20260222164446411" style="zoom:67%;" />
