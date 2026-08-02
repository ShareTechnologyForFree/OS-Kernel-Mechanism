# Page 与 Folio 机制分析

> 适用内核：Android 15 / linux-6.6 GKI（源码树 `android-kernel/common`）
> 配套测试模块：`folio_page_test.c`（基础 API 演示）、`page_vs_folio_bug.c`（head/tail 陷阱演示）
> 本文按 **概念 → 案例 → 原理 → 工作流程 → 代码分析 → 对比总结** 六部分组织：先看 page 与 folio 是什么、怎么用（概念 + 案例），再深入原理与源码。

---

## 目录

1. [概念篇](#1-概念篇)
2. [案例篇：两个测试模块逐实验分析](#2-案例篇)
3. [原理篇：关键机制与内联函数源码分析](#3-原理篇)
4. [工作流程篇：分配 / 释放 / 转换](#4-工作流程篇)
5. [代码分析篇：folio-compat.c 桥接层](#5-代码分析篇)
6. [对比总结篇](#6-对比总结篇)
7. [附录：page_vs_folio_bug.ko 真实执行输出](#7-附录)

---

## 1. 概念篇

### 1.1 `struct page` —— 每一帧物理内存的描述符

物理内存被划分成固定大小的 **页帧（page frame）**（arm64 上通常 4KB），内核为**每一帧**分配一个 `struct page`（约 64B）作为元数据描述符。所有 `struct page` 连续排布在 **vmemmap** 中——物理内存有多大，vmemmap 就有多少个 `struct page`，**一个 page frame 有且只有一个 `struct page`**。

`struct page` 本身是一个大 union（`mm_types.h`），同一块存储根据页的用途被复用为：页缓存/匿名页字段（`lru / mapping / index / private`）、slab、页池、compound 元数据（`compound_head`）等。也正因为这种"一个结构多种身份"，才需要标志位来区分当前身份，也才埋下了 head/tail 陷阱。

### 1.2 `struct folio` —— 页组抽象（5.16 引入）

`struct folio` 表示**一组物理连续、大小是 2 的幂、且 2 的幂对齐的页**（大小 ≥ PAGE_SIZE）：

- 对 **order-0 单页**，folio 就是这个页本身；
- 对 **compound（复合）页**，folio 就是整个 compound page（head + 所有 tail）。

核心保证：**指向 folio 的指针永远指向 head 页——不存在 "tail folio"**。调用者拿到的 `struct folio *`，其 `mapping / index / private` 字段永远可信。

`struct folio` 布局（`include/linux/mm_types.h:319`）——**第一个 union 与 `struct page` 完全重叠**（所以 `&folio->page` 就是 head 页，`(struct folio *)page` 就是合法的强转）：

```c
struct folio {
	union {
		struct {
			unsigned long flags;
			union {
				struct list_head lru;
				struct {
					void *__filler;
					unsigned int mlock_count;
				};
			};
			struct address_space *mapping;
			pgoff_t index;
			union {
				void *private;
				swp_entry_t swap;
			};
			atomic_t _mapcount;
			atomic_t _refcount;
#ifdef CONFIG_MEMCG
			unsigned long memcg_data;
#endif
		};
		struct page page;   /* 与 struct page 重叠 */
	};
	union {
		struct {
			unsigned long _flags_1;   /* 存 order（低 8 位）等 */
			unsigned long _head_1;
			atomic_t _entire_mapcount;
			atomic_t _nr_pages_mapped;
			atomic_t _pincount;
#ifdef CONFIG_64BIT
			unsigned int __padding;
			unsigned int _folio_nr_pages;  /* 64 位下直接存页数 1<<order */
#endif
			/* ... */
		};
		/* ... */
	};
	/* ... */
};
```

### 1.3 二者关系

| 场景 | page 视角 | folio 视角 |
|---|---|---|
| `alloc_page()` | 1 个 `struct page` | `page_folio()` → order-0 folio，nr_pages=1 |
| `alloc_pages(..., 2)` | head + 3 个 tail | 1 个 large folio，nr_pages=4 |
| 语义 | 一页一描述符，页间无"组"概念 | 一组页一个对象，字段从 head 读 |

转换口诀：

- `page → folio`：`page_folio(p)`（O(1)，自动跳回 head）；
- `folio → page`：`&folio->page`（head 页）或 `folio_page(folio, n)`（第 n 个子页）。

### 1.4 为什么需要 folio：page 时代的三个坑

`alloc_pages(__GFP_COMP, order)` 返回的是 head 页指针，但**调用者拿到任意子页指针时，在编译期无法知道它是 head 还是 tail**。tail 页的字段被内核做了手脚，直接读会拿到假数据：

1. **`tail->mapping` 是毒值**（非空假指针）——用它判"是否挂 page cache"会误判，解引用即崩；
2. **`tail->index` 是垃圾残留值**——拿它当文件偏移会写错位置；
3. **`page_address(tail)` 起点偏 N 页**，且习惯上只按 4KB 处理——大块读写丢数据。

folio 用 `page_folio()` 在 API 层面**自动跳回 head**，从根上消除了这三点（详见第 2 章实测）。

---

## 2. 案例篇

### 2.1 `folio_page_test.c` —— 基础 API 演示（4 个实验）

#### Test 1：order-0 单页，page 与 folio 一一对应

流程：

1. `alloc_page(GFP_KERNEL)` 拿 1 个页（等价 `alloc_pages(gfp, 0)`）；
2. `page_folio(page)` 转换——order-0 无 compound，`_compound_head` 直接返回自身；
3. 打印验证：`folio_nr_pages=1`、`folio_order=0`、`folio_test_large=0`（非大页），`folio_address == page_address`；
4. `memset(addr, 0xAB, PAGE_SIZE)`：folio 与 page 的地址可互换操作；
5. 引用计数：`folio_get()` 后 `ref=2`，`folio_put()` 回到 1（同一个计数器）；
6. `__free_page(page)`：refcount 1→0 归还伙伴系统。

**验证点**：order-0 时 folio 就是该页本身，两种 API 完全等价、可互相转换。

#### Test 2：order-2 compound，一个 folio 管理 4 个 page

流程：

1. `alloc_pages(GFP_KERNEL | __GFP_COMP, 2)` 分配 4 个物理连续页；
2. `page_folio(page)` 得 large folio：`nr=4`、`order=2`、`large=1`；
3. 用 `folio_page(folio, i)` 逐个取子页，校验 `page_to_pfn(sub) == folio_pfn(folio) + i`，确认**物理连续**；
4. `memset(folio_address, 0x11, folio_nr_pages * PAGE_SIZE)` 把 16KB 当一整块写；
5. `__free_pages(page, 2)` 释放。

**验证点**：大页内部 pfn 连续，folio 粒度 = 整块，可一次性 memset 整块内存。

#### Test 3：`folio_alloc()` 直接以 folio 为粒度申请

流程：

1. `folio_alloc(GFP_KERNEL, 0)` —— 新代码风格，不经过 page 中间态（内部自动补 `__GFP_COMP`）；
2. `folio_address()` 取地址，`strcpy` 写入 "hello folio!" 并读回验证；
3. `folio_put(folio)` 释放（refcount 1→0，内部 `__folio_put()`）。

**验证点**：申请/释放全程不出现 `struct page *` 局部变量，folio 是自足的分配粒度。

#### Test 4：tail page 通过 `page_folio()` 找回 head folio

流程：

1. `alloc_pages(GFP_KERNEL | __GFP_COMP, 1)` 分 2 页，`folio = page_folio(page)`（head）；
2. `tail = folio_page(folio, 1)` 取 tail，`PageTail(tail)=1` 确认；
3. `head = page_folio(tail)` —— bit0 回跳，`head == folio` 成立；
4. 打印 `tail->mapping`（毒值）vs `folio->mapping`（有效值），`nr/order` 经 head 拿到。

**验证点**：从任意子页都能 O(1) 找回 head，tail 的裸字段不可信、folio 的字段可信。

### 2.2 `page_vs_folio_bug.c` —— head/tail 陷阱实战（4 个实验）

统一手法：每实验先 `alloc_pages(__GFP_COMP, order)` 得到 compound，然后**同一份数据分别走"旧式 page 写法"和"新式 folio 写法"**，对比结果。真实输出见附录。

#### Test 1：`tail->mapping` 毒值 → 误判页状态

- 分配 2 页；`folio = page_folio(page)`（head），`tail = folio_page(folio, 1)`；
- 读字段：
  - `head->mapping = 0`（真值：未挂 page cache）；
  - `tail->mapping = dead000000000000`（毒值，非空）；
- 旧式 `tail->mapping ? ... : ...` → **YES (WRONG)**：把毒值当"已挂 mapping"，后续解引用即崩；
- folio 式 `folio->mapping` → 读 head 真值 → **no (correct)**。

**结论**：同一个判空，page 读毒值、folio 读真值——差异即"是否自动跳回 head"。

#### Test 2：`tail->index` 不可信 → 拿错文件偏移

- 分配 2 页；模拟挂 page cache：`folio->index = 0x1234`（写 head）；
- `head->index = 0x1234`（真值）；`tail->index = 2`（残留垃圾，与 head 无任何固定关系）；
- 旧式 `off = tail->index` → **0x2**：当文件偏移就偏到第 2 块；
- folio 式 `folio->index + 1` → **0x1235**：块首 index 可信，第 n 子页偏移 = 块首 + n。

**结论**：偏移必须从块首（folio/index）出发；直接读 `tail->index` 是垃圾。

#### Test 3：`page_address(tail)` 起点偏 → 拿错虚拟地址

- 分配 4 页；`tail = folio_page(folio, 2)`；
- 地址对比：head=`ffffff800391c000`，tail=`ffffff800391e000`，**diff=0x2000**（2 页 × 4KB，恰为 tail 相对 head 的偏移）；`folio_address(folio) == page_address(head)` → yes；
- 旧式把 `page_address(tail)` 当块首 → 起点偏 8KB，且习惯只处理 `PAGE_SIZE`；
- folio 式：`folio_address`（块首）+ `folio_size=16384`（整块长度）。

**结论**：`page_address()` 只对"自己"准确，块首 + 整块长度只有 folio 语义保证。

#### Test 4：文件块写入 → page 版漏写，folio 版整块正确

- 分配 4 页，先 `memset(folio_address, 0, folio_size)` 清零 16KB；
- **旧式**：`p = folio_page(folio, 2)`（tail），`memset(page_address(p), 0x55, PAGE_SIZE)`：
  ```
  after old write: p0=00 p1=00 p2=55 p3=00   ← 只写了第 3 页，丢了 12KB
  ```
- **folio 式**：同样拿 tail[2]，但 `f = page_folio(p)` 跳回 head，`memset(folio_address(f), 0xaa, folio_size(f))`：
  ```
  after folio write: p0=aa p1=aa p2=aa p3=aa  ← 一次写满 16KB
  ```

**结论**：page 写法"起点偏 + 长度错"双重翻车；folio 写法一步到位。

---

## 3. 原理篇

案例里反复出现的 `page_folio()` 回跳、TAIL_MAPPING 毒值、"块首 + 整块大小"从哪来？原理篇逐个拆解内核实现。

### 3.1 compound page 布局：head + N 个 tail

申请 `order` 阶大页时，内核把它组织成 compound page：1 个 head + (2^order − 1) 个 tail。构造过程在 `mm/page_alloc.c:713`：

```c
void prep_compound_page(struct page *page, unsigned int order)
{
	int i;
	int nr_pages = 1 << order;

	__SetPageHead(page);                    /* ① 给 head 打 PG_head 标志 */
	for (i = 1; i < nr_pages; i++)
		prep_compound_tail(page, i);        /* ② 逐个初始化 tail */

	prep_compound_head(page, order);        /* ③ 登记 order、初始化大页计数 */
}
EXPORT_SYMBOL_GPL(prep_compound_page);
```

每个 tail 的初始化在 `mm/internal.h:709`，这里就是毒值的来源（对应案例 2.2 Test 1 的 `dead000000000000`）：

```c
static inline void prep_compound_tail(struct page *head, int tail_idx)
{
	struct page *p = head + tail_idx;

	p->mapping = TAIL_MAPPING;          /* 毒化 mapping，防误读 */
	set_compound_head(p, head);         /* 记下"我是 tail，head 在哪" */
	set_page_private(p, 0);
}
```

head 的初始化在 `mm/internal.h:697`：

```c
static inline void prep_compound_head(struct page *page, unsigned int order)
{
	struct folio *folio = (struct folio *)page;

	folio_set_order(folio, order);      /* 存 order，64 位下同时存 _folio_nr_pages */
	atomic_set(&folio->_entire_mapcount, -1);
	atomic_set(&folio->_nr_pages_mapped, 0);
	atomic_set(&folio->_pincount, 0);
	if (order > 1)
		INIT_LIST_HEAD(&folio->_deferred_list);
}
```

`folio_set_order`（`mm/internal.h:661`）：

```c
static inline void folio_set_order(struct folio *folio, unsigned int order)
{
	if (WARN_ON_ONCE(!order || !folio_test_large(folio)))
		return;

	folio->_flags_1 = (folio->_flags_1 & ~0xffUL) | order;   /* order 存低 8 位 */
#ifdef CONFIG_64BIT
	folio->_folio_nr_pages = 1U << order;                   /* 64 位直接存页数 */
#endif
}
```

### 3.2 head 定位魔法：`compound_head` 的 bit0

案例 2.1 Test 4 / 2.2 Test 4 里 `page_folio(tail)` 能一步跳回 head，秘密在 `set_compound_head`（`include/linux/page-flags.h:851`）——它故意把 head 指针**加 1** 再写入 tail 的 `compound_head` 字段：

```c
static __always_inline void set_compound_head(struct page *page, struct page *head)
{
	WRITE_ONCE(page->compound_head, (unsigned long)head + 1);
}
```

这样 bit0 = 1 既是"我是 tail"的标记，又能在回跳时被减去还原。反向解析（`page-flags.h:251`）：

```c
static inline unsigned long _compound_head(const struct page *page)
{
	unsigned long head = READ_ONCE(page->compound_head);

	if (unlikely(head & 1))            /* bit0 置位 → 我是 tail */
		return head - 1;               /* 减 1 即得 head 地址 */
	return (unsigned long)page_fixed_fake_head(page);
}

#define compound_head(page)	((typeof(page))_compound_head(page))
```

于是 `page_folio()`（`page-flags.h:275`，用 `_Generic` 保留 const 属性）：

```c
#define page_folio(p)		(_Generic((p),				\
	const struct page *:	(const struct folio *)_compound_head(p), \
	struct page *:		(struct folio *)_compound_head(p)))
```

- 传 head 进来：bit0=0，原样返回 → 返回自己；
- 传 tail 进来：bit0=1，减 1 跳到 head → 返回 head 的 folio。

**全程 O(1)、不遍历、不加锁**——这是 folio 一切安全性的地基。

### 3.3 tail 防御：TAIL_MAPPING 毒值

毒值定义在 `include/linux/poison.h:34`：

```c
#define TAIL_MAPPING	((void *) 0x400 + POISON_POINTER_DELTA)
```

`POISON_POINTER_DELTA` 在 64 位下取 `CONFIG_ILLEGAL_POINTER_VALUE`（`poison.h:13`），arm64 定义在 `arch/arm64/Kconfig:333`：

```c
config ILLEGAL_POINTER_VALUE
	hex
	default 0xdead000000000000
```

即本源码树 arm64 上 `TAIL_MAPPING = 0xdead000000000400`。注意：**实际设备运行时打印的是 `0xdead000000000000`**（见附录），说明运行内核的毒值定义可能不同——但关键是它**永远非空、且明显是非法地址**，任何"判空/解引用"都会立刻暴露错误（这正是案例 2.2 Test 1 的翻车点）。

### 3.4 页标志位（`include/linux/page-flags.h`）

```c
enum pageflags {
	...
	PG_lru,
	PG_head,		/* Must be in bit 6 */     /* L106-107 */
	...
};
```

与 head/tail 判定相关的谓词：

| 谓词 | 实现 | 含义 |
|---|---|---|
| `PageHead()`（L830） | `test_bit(PG_head, &page->flags) && !page_is_fake_head(page)` | 是 compound head |
| `PageTail()`（L290） | `READ_ONCE(page->compound_head) & 1 \|\| page_is_fake_head(page)` | 是 tail（bit0 标记） |
| `PageCompound()`（L295） | `test_bit(PG_head, ...) \|\| READ_ONCE(...) & 1` | 属于某个 compound 页 |
| `folio_test_large()`（L846） | `folio_test_head(folio)` | folio 大于一页（即带 PG_head） |

`compound_nr()` / `folio_nr_pages()` 都会先查 `PG_head`：不是大页直接返回 1；是大页在 64 位下直接读 `_folio_nr_pages`。

### 3.5 folio 常用内联函数（`include/linux/mm.h`）

案例里用到的 `folio_nr_pages` / `folio_order` / `folio_size` / `folio_address` 实现如下：

```c
/* folio_nr_pages —— 页组里有多少页（L2133） */
static inline long folio_nr_pages(struct folio *folio)
{
	if (!folio_test_large(folio))
		return 1;
#ifdef CONFIG_64BIT
	return folio->_folio_nr_pages;
#else
	return 1L << (folio->_flags_1 & 0xff);
#endif
}

/* folio_order —— 分配阶数，2^order 页（L1141） */
static inline unsigned int folio_order(struct folio *folio)
{
	if (!folio_test_large(folio))
		return 0;
	return folio->_flags_1 & 0xff;
}

/* folio_size —— 整块字节数（L2215） */
static inline size_t folio_size(struct folio *folio)
{
	return PAGE_SIZE << folio_order(folio);
}

/* folio_address —— 块首虚拟地址（L2380） */
static inline void *folio_address(const struct folio *folio)
{
	return page_address(&folio->page);   /* 就是 head 页的地址 */
}
```

以及子页访问（`page-flags.h:288`）：`#define folio_page(folio, n) nth_page(&(folio)->page, n)`——`nth_page` 按 pfn 递增取第 n 个物理连续的页。

---

## 4. 工作流程篇

### 4.1 分配流程（page 视角）

```c
alloc_pages(gfp, order)
  └─ alloc_pages_node(nid, gfp, order)          include/linux/gfp.h
       └─ __alloc_pages(gfp, order, nid, NULL)  mm/page_alloc.c:5072  ← 伙伴系统心脏
            ├─ prepare_alloc_pages(...)         构造 alloc_context、选 zone
            ├─ get_page_from_freelist(...)      从 pcp/伙伴树取空闲块
            └─ 成功后：
               post_alloc_hook(page, order, gfp)
               prep_new_page(page, order, gfp, alloc_flags)   mm/page_alloc.c:1729
                  ├─ post_alloc_hook(...)       初始化/清零/KASAN 等
                  └─ if (order && (gfp_flags & __GFP_COMP))
                         prep_compound_page(page, order)       ← 组装 compound
                             │  __SetPageHead(page)     头打 PG_head
                             │  循环 prep_compound_tail() 尾：mapping 毒化 + compound_head
                             └─ prep_compound_head()    存 order / _folio_nr_pages
```

关键点（`page_alloc.c:1729`）：

```c
void prep_new_page(struct page *page, unsigned int order, gfp_t gfp_flags,
						unsigned int alloc_flags)
{
	post_alloc_hook(page, order, gfp_flags);

	if (order && (gfp_flags & __GFP_COMP))    /* 只有带 __GFP_COMP 才组装 compound */
		prep_compound_page(page, order);
	...
}
```

即：**不带 `__GFP_COMP` 的 order>0 分配不是 compound**（没有 head/tail，tail 陷阱只出现在 `__GFP_COMP` 页上）；`alloc_page()` 展开为 `alloc_pages(gfp, 0)`，order=0 不触发 compound 逻辑。

### 4.2 分配流程（folio 视角）

`folio_alloc()`（`gfp.h:301`，非 NUMA）→ `__folio_alloc_node()`（`gfp.h:269`）→ `__folio_alloc()`（`page_alloc.c:5144`）：

```c
struct folio *__folio_alloc(gfp_t gfp, unsigned int order, int preferred_nid,
		nodemask_t *nodemask)
{
	struct page *page = __alloc_pages(gfp | __GFP_COMP, order,
					preferred_nid, nodemask);   /* 自动补 __GFP_COMP */
	return page_rmappable_folio(page);   /* 强转 + 置 LargeRmappable 标志 */
}
```

注意 folio 路径**自动补 `__GFP_COMP`**——保证大页一定是 compound，folio 语义才成立。`page_rmappable_folio`（`mm/internal.h:689`）：

```c
static inline struct folio *page_rmappable_folio(struct page *page)
{
	struct folio *folio = (struct folio *)page;

	folio_prep_large_rmappable(folio);
	return folio;
}
```

### 4.3 释放流程

**page 视角** `__free_pages()`（`page_alloc.c:5195`）：

```c
void __free_pages(struct page *page, unsigned int order)
{
	/* get PageHead before we drop reference */
	int head = PageHead(page);

	if (put_page_testzero(page))
		free_the_page(page, order);
	else if (!head)                     /* 非 head 且引用没到 0：逐块补还 */
		while (order-- > 0)
			free_the_page(page + (1 << order), order);
}
```

`free_the_page`（`page_alloc.c:693`）按阶数走 pcp 快速路径或直接回伙伴树：

```c
static inline void free_the_page(struct page *page, unsigned int order)
{
	if (pcp_allowed_order(order))		/* Via pcp? */
		free_unref_page(page, order);
	else
		__free_pages_ok(page, order, FPI_NONE);
}
```

**folio 视角** `folio_put()`（`mm.h:1552`）：

```c
static inline void folio_put(struct folio *folio)
{
	if (folio_put_testzero(folio))
		__folio_put(folio);
}
```

`__folio_put` 引用归零后经 `destroy_large_folio()`（`page_alloc.c:726`：hugetlb 特殊处理 / memcg uncharge / 解除 deferred split）再回到 `free_the_page(&folio->page, folio_order(folio))`。

### 4.4 page ↔ folio 转换流程

```c
  任意子页 p（可能是 tail）
        │  page_folio(p)
        │  └─ _compound_head(p)：读 compound_head，bit0 置位则 -1
        ▼
  head 对应的 struct folio *   ← 字段全部可信
        │  folio_page(folio, n) = nth_page(&folio->page, n)
        ▼
  第 n 个子页（head+n）
```

这就是 `page_vs_folio_bug.c` Test 4 里"从 tail 跳回整块"的机制来源。

---

## 5. 代码分析篇

### 5.1 `folio-compat.c`：旧 page API 的桥接层

folio 迁移不可能一步到位，内核用 `mm/folio-compat.c` 让旧 page 调用者**改一行注释量都为零**地继续工作——所有旧函数内部一律先 `page_folio(page)` 跳回 head，再调 folio 版本：

| 旧 page API | folio 实现（`folio-compat.c`） | 语义 |
|---|---|---|
| `page_mapping()` | `folio_mapping(page_folio(page))` | 拿 mapping |
| `unlock_page()` | `folio_unlock(page_folio(page))` | 解锁 |
| `set_page_dirty()` | `folio_mark_dirty(page_folio(page))` | 置脏 |
| `set_page_writeback()` | `folio_start_writeback(page_folio(page))` | 写回开始 |
| `end_page_writeback()` | `folio_end_writeback(page_folio(page))` | 写回结束 |
| `add_to_page_cache_lru()` | `filemap_add_folio(mapping, page_folio(page), ...)` | 入 page cache |
| `pagecache_get_page()` | `__filemap_get_folio(...)` + `folio_file_page()` | 取缓存页 |

典型实现（`folio-compat.c`）：

```c
struct address_space *page_mapping(struct page *page)
{
	return folio_mapping(page_folio(page));
}
EXPORT_SYMBOL(page_mapping);

int add_to_page_cache_lru(struct page *page, struct address_space *mapping,
		pgoff_t index, gfp_t gfp)
{
	return filemap_add_folio(mapping, page_folio(page), index, gfp);
}
EXPORT_SYMBOL(add_to_page_cache_lru);
```

**值得注意的例外**——`isolate_lru_page()` 对 tail 页直接拒收并告警：

```c
bool isolate_lru_page(struct page *page)
{
	if (WARN_RATELIMIT(PageTail(page), "trying to isolate tail page"))
		return false;
	return folio_isolate_lru((struct folio *)page);
}
```

这正说明：**有些操作"拿到 tail"本身就是 bug**，兼容层也要用 `PageTail()` 显式防御——印证了第 1.4 节的三个坑。

### 5.2 关键源码位置索引

| 内容 | 位置 |
|---|---|
| `struct folio` 定义 | `include/linux/mm_types.h:319` |
| `page_folio` / `compound_head` / `folio_page` / `PageTail` / `PageCompound` | `include/linux/page-flags.h:251-298` |
| `set_compound_head` | `include/linux/page-flags.h:851` |
| `folio_test_large` / `PageHead` | `include/linux/page-flags.h:830/846` |
| `TAIL_MAPPING` | `include/linux/poison.h:34` |
| `ILLEGAL_POINTER_VALUE`（arm64） | `arch/arm64/Kconfig:333` |
| `prep_compound_page` / `prep_new_page` / `__alloc_pages` / `__folio_alloc` / `__free_pages` / `free_the_page` | `mm/page_alloc.c:713/1729/5072/5144/5195/693` |
| `prep_compound_tail` / `prep_compound_head` / `folio_set_order` / `page_rmappable_folio` | `mm/internal.h:697-716/661/689` |
| `folio_nr_pages` / `folio_order` / `folio_size` / `folio_address` / `folio_put` | `include/linux/mm.h:2133/1141/2215/2380/1552` |
| `folio_alloc` / `__folio_alloc_node` | `include/linux/gfp.h:293-306/269` |
| page→folio 兼容层 | `mm/folio-compat.c` |

---

## 6. 对比总结篇

### 6.1 核心差异

| 维度 | `struct page` 时代 | `struct folio` 时代 |
|---|---|---|
| 对象粒度 | 1 页 = 1 描述符 | 1 个页组（1..N 页） |
| 指针保证 | 可能是 head/tail，编译期无法区分 | 永远指向 head |
| 读 mapping | tail 读到 TAIL_MAPPING 毒值 | 永远读 head 真值 |
| 读 index | tail 读到残留垃圾 | 永远读 head 真值 |
| 取地址 | `page_address(p)` 可能偏 N 页 | `folio_address()` 永远块首 |
| 知道整块大小 | 要自己传 order / `compound_order()` | `folio_size()` / `folio_nr_pages()` |
| 申请 API | `alloc_pages(gfp, order)` 需手动带 `__GFP_COMP` | `folio_alloc(gfp, order)` 自动补 |
| 释放 API | `__free_pages(page, order)` | `folio_put(folio)` |

### 6.2 API 迁移对照表

| 目的 | 旧写法 | 新写法 |
|---|---|---|
| 申请单页 | `alloc_page(gfp)` | `folio_alloc(gfp, 0)` |
| 申请大页 | `alloc_pages(gfp \| __GFP_COMP, order)` | `folio_alloc(gfp, order)` |
| page→folio | `(struct folio *)page`（危险） | `page_folio(page)` |
| folio→page | — | `&folio->page` / `folio_page(folio, n)` |
| 页数 | 无（自己算 `1<<order`） | `folio_nr_pages(folio)` |
| 字节数 | 无 | `folio_size(folio)` |
| 地址 | `page_address(page)` | `folio_address(folio)` |
| 释放 | `__free_pages(page, order)` | `folio_put(folio)` |
| 判大页 | `PageCompound(page)` | `folio_test_large(folio)` |

### 6.3 结论

- **folio 不是新内存，而是对既有 compound 页更安全的视图**：数据仍在原来的物理页上，变化的是 API 保证——`page_folio()` 的 bit0 回跳让"永远指向 head"成为语言层面的不变式；
- 三个历史坑（毒 mapping / 垃圾 index / 偏移起点）全部源于"调用者拿到的可能是 tail 却当成 head 用"，folio 在入口处统一解决；
- 旧代码迁移期由 `folio-compat.c` 桥接，新代码应直接使用 folio 系列 API。

---

## 7. 附录

### 7.1 `page_vs_folio_bug.ko` 真实执行输出（aarch64 目标机）

```bash
# insmod page_vs_folio_bug.ko
[  428.766903][  T110] [xxx_pf_bug] module loaded
[  428.768116][  T110] [xxx_pf_bug] == Test 1: tail->mapping poisoned ==
[  428.769647][  T110]   head->mapping=0000000000000000 (real)
[  428.770609][  T110]   tail->mapping=dead000000000000 (poison)
[  428.771386][  T110]   old(page): has_mapping=YES (WRONG)
[  428.772397][  T110]   new(folio): has_mapping=no (correct)
[  428.773887][  T110] [xxx_pf_bug] == Test 2: tail->index untrusted ==
[  428.775329][  T110]   sim: folio->index=0x1234 (page cache offset)
[  428.777460][  T110]   head->index=1234  tail->index=2 (garbage)
[  428.778877][  T110]   old(page): off=tail->index -> 0x2 (WRONG)
[  428.780130][  T110]   new(folio): off=folio->index+1 -> 0x1235 (correct)
[  428.782547][  T110] [xxx_pf_bug] == Test 3: page_address() wrong base ==
[  428.784545][  T110]   page_address(head)=ffffff800391c000
[  428.786375][  T110]   page_address(tail)=ffffff800391e000 (diff=2000)
[  428.788334][  T110]   folio_address(folio)=ffffff800391c000 == head? yes
[  428.790198][  T110]   old(page): addr=page_address(tail), size=PAGE_SIZE
[  428.792128][  T110]   new(folio): addr=folio_address, size=16384
[  428.794553][  T110] [xxx_pf_bug] == Test 4: file block write: page vs folio ==
[  428.796471][  T110]   old(page): p=page[2](tail), write 4KB
[  428.798476][  T110]     after old write: p0=00 p1=00 p2=55 p3=00
[  428.800174][  T110]   new(folio): folio=page_folio(p), write folio_size
[  428.803126][  T110]     after folio write: p0=aa p1=aa p2=aa p3=aa
[  428.805928][  T110] [xxx_pf_bug] all tests done
# rmmod page_vs_folio_bug.ko
[  431.810956][  T112] [xxx_pf_bug] module unloaded
```

> 注：本源码树 arm64 上 `TAIL_MAPPING = 0x400 + 0xdead000000000000 = 0xdead000000000400`，而实际设备打印 `dead000000000000`——运行内核的毒值定义不同，但"非空非法指针"的语义一致，不影响任何结论。
