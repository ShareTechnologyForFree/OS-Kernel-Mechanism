# Linux内核workqueue机制分析

[TOC]

# 1、workqueue 解决什么问题

内核里很多代码在**中断上下文**中触发（网卡收包、定时器），但后续处理需要**进程上下文**（分配内存可能睡眠、拿信号量）。中断上下文不能睡眠，直接做会炸。

workqueue 的做法：把工作**推迟到内核线程**里执行。线程是进程上下文，可以睡眠。

workqueue 自 Linux 2.6 引入 CMWQ（Concurrency Managed Workqueue）后，所有工作队列共享一组 per-CPU 的 worker thread pool，内核自动管理 worker 数量，极大减少了线程数和上下文切换开销。

linux-6.6内核源码：https://github.com/torvalds/linux/tree/v6.6

# 2、核心概念

## 2.1、四个角色

| 概念 | 通俗理解 | 对应结构体 | 谁创建 |
|------|---------|-----------|--------|
| **work** | 一封信，写好了要干什么 | `struct work_struct` | 你（调用者） |
| **workqueue** | 收件箱，装信的容器 | `struct workqueue_struct` | 你（`alloc_workqueue`） |
| **worker** | 干活的员工 | `struct worker` | 内核自动 |
| **worker_pool** | 办公室，员工集中干活的地方 | `struct worker_pool` | 内核启动时 |

**这种设计的动机：**

老内核（2.6 之前）是每个 workqueue 一个专属内核线程。问题：几十上百个 workqueue 各占一个线程，绝大多数在睡觉，白白占用 PID、栈空间和调度开销。

CMWQ 的改进：**池化 + 共享**。N 个 workqueue 共用 M 个 worker（M << N），中间用 `pool_workqueue` 做闸门控制每个 wq 的并发数。

```bash
      你的收件箱              你的收件箱             你的收件箱
     (my_wq_A)              (my_wq_B)             (my_wq_C)
          │                      │                     │
          │  每个 CPU 一个        │                     │
          ▼                      ▼                     ▼
     pool_workqueue         pool_workqueue        pool_workqueue
     (max_active=2)         (max_active=4)        (max_active=1)
          │                      │                     │
          └──────────────────────┼─────────────────────┘
                                 ▼
                          worker_pool (CPU0)
                         worklist: 所有人的信混在一起
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
               kworker/0:1  kworker/0:2  kworker/0:3
               (拿信→干活)  (拿信→干活)  (睡觉待命)
```

## 2.2、一张图看懂运行流程

假设 `my_wq` 设了 `max_active=2`，你要投 3 个 work。从投递到执行完毕全流程如下：

```bash
═══════════════════════════════════════════════════════════════════
                          阶段 A：投递
═══════════════════════════════════════════════════════════════════

  queue_work(my_wq, &work1)                     queue_work(my_wq, &work2)
        │                                              │
        ▼                                              ▼
  ┌──────────────┐                            ┌──────────────┐
  │ PENDING = 1  │ 原子操作防重入               │ PENDING = 1  │
  │ pick CPU0    │ 选当前 CPU 的 pwq            │ pick CPU0    │
  │              │                            │              │
  │ pwq:         │                            │ pwq:         │
  │  max_active=2│                            │  max_active=2│
  │  nr_active=0 │                            │  nr_active=1 │
  │  0<2 → 放行! │                            │  1<2 → 放行! │
  └──────┬───────┘                            └──────┬───────┘
         │                                           │
         ▼                                           ▼
  insert_work(&pool->worklist)              insert_work(&pool->worklist)
  pool->worklist: [work1]                   pool->worklist: [work1]→[work2]
         │                                           │
         ▼                                           ▼
  kick_pool(pool)                            kick_pool(pool)
  唤醒 kworker/0:1                           唤醒 kworker/0:2


  queue_work(my_wq, &work3)
        │
        ▼
  ┌──────────────┐
  │ pwq:         │
  │  nr_active=2 │
  │  max_active=2│
  │  2==2 → 满了!│  ← 不放行! 放入 pwq->inactive_works
  └──────────────┘    work3 标记 WORK_STRUCT_INACTIVE，不进入 pool->worklist


═══════════════════════════════════════════════════════════════════
                          阶段 B：执行
═══════════════════════════════════════════════════════════════════

worker_thread() 主循环 (kernel/workqueue.c:2729):

  kworker/0:1 醒来                          kworker/0:2 醒来
     │                                         │
     ├─ 持锁 pool->lock                        ├─ 持锁 pool->lock
     ├─ 从 idle_list 摘除                      ├─ 从 idle_list 摘除
     ├─ 从 pool->worklist 取 work1             ├─ 从 pool->worklist 取 work2
     ├─ hash_add(busy_hash)                    ├─ hash_add(busy_hash)
     ├─ 清除 PENDING                           ├─ 清除 PENDING
     │                                         │
     ├─ ★ 释放 pool->lock ★                    ├─ ★ 释放 pool->lock ★
     │  (执行期间不持锁,别人可以继续投信)        │  (执行期间不持锁)
     │                                         │
     ├─ work1->func() ← 执行回调               ├─ work2->func() ← 执行回调
     │  (进程上下文, 可睡眠, 可持续任意时间)      │
     │                                         │
     ├─ 重新持锁 pool->lock                    ├─ 重新持锁 pool->lock
     ├─ hash_del(busy_hash)                    ├─ hash_del(busy_hash)
     ├─ pwq->nr_active: 2→1                   ├─ pwq->nr_active: 1→0
     │                                         │
     │  ┌────────────────────────────┐         │
     ├─→│ pwq->inactive_works 有东西? │         │  pwq->inactive_works 为空
     │  │ 有! work3 在等              │         │
     │  │ 取出 work3                  │         │
     │  │ 移入 pool->worklist         │         │
     │  │ pwq->nr_active: 1→2        │         │
     │  │ kick_pool(pool)             │         │
     │  └────────────────────────────┘         │
     │                                         │
     ├─ keep_working? 没有了                   ├─ keep_working? 没有了
     ├─ 回到 idle_list 睡觉                    ├─ 回到 idle_list 睡觉
     └─ schedule()                            └─ schedule()

  kworker/0:1 被 kick 唤醒:
     ├─ ...拿 work3...
     ├─ 执行 work3->func()
     ├─ pwq->nr_active: 2→1
     ├─ inactive_works 空了
     └─ 回 idle_list 睡觉
```

**理解要点：**

1. **pool->worklist 是公共的**：work1、work2、work3 和其他 wq 的 work 混在一起。worker 从上面拿，不管信是谁家的
2. **pool_workqueue 是闸门**：只管"你家的信同时在处理几封"，超了就扣在 inactive_works
3. **执行时不持锁**：`work->func()` 跑的时候锁是放掉的，别人可以继续 queue_work
4. **PENDING 位防重入**：`test_and_set_bit(PENDING, &work->data)` 保证同一封信不会重复投递

# 3、核心数据结构

## 3.1、`struct work_struct`（工作项）

定义于 [include/linux/workqueue.h:98](file:///home/xxx/linux_old1/include/linux/workqueue.h#L98)。

```c
struct work_struct {
    atomic_long_t data;           // 低位是 flag 位，高位指向 pwq
    struct list_head entry;       // 链表节点，挂入 worklist 或 inactive_works
    work_func_t func;             // 实际执行的回调函数
};
```

`data` 字段的精妙设计：低位置标志位，高位存指向 `pool_workqueue` 的指针。`test_and_set_bit(PENDING)` 一个原子操作同时完成状态检查和指针访问。

```c
// 标志位定义 (include/linux/workqueue.h:30)
enum {
    WORK_STRUCT_PENDING_BIT   = 0,   // work 已排队等待执行
    WORK_STRUCT_INACTIVE_BIT  = 1,   // work 被搁置在 inactive_works
    WORK_STRUCT_PWQ_BIT       = 2,   // data 指向 pwq
    WORK_STRUCT_LINKED_BIT    = 3,   // 下一个 work 链接到当前
    WORK_STRUCT_COLOR_SHIFT   = 4,   // flush 颜色位 (4 bits, 支持 16 种颜色)
};
```

辅助结构：
- `struct delayed_work`：包含 `work_struct` + `timer_list`，用于延迟执行。
- `struct rcu_work`：包含 `work_struct` + `rcu_head`，用于 RCU 回调后执行。

## 3.2、`struct worker`（执行体）

定义于 [kernel/workqueue_internal.h:24](file:///home/xxx/linux_old1/kernel/workqueue_internal.h#L24)。

```c
struct worker {
    union {
        struct list_head  entry;   // 空闲时挂在 pool->idle_list 上
        struct hlist_node hentry;  // 忙碌时挂入 pool->busy_hash 哈希表
    };
    struct work_struct    *current_work;    // 正在处理的 work
    work_func_t            current_func;    // 当前执行的函数
    struct pool_workqueue  *current_pwq;    // 当前关联的 pwq
    struct list_head       scheduled;       // 已调度的工作列表
    struct task_struct     *task;           // 对应的内核线程
    struct worker_pool     *pool;           // 所属的 worker_pool
    struct list_head       node;            // pool->workers 链表节点
    int                    id;              // worker ID（用于进程名 kworker/u:ID）
    unsigned int           flags;           // WORKER_DIE, WORKER_IDLE 等状态
    char                   desc[WORKER_DESC_LEN];  // 当前工作描述
    struct workqueue_struct *rescue_wq;     // rescuer 专用的目标 wq
};
```

worker 空闲/忙碌用 union 存储：空闲时挂在 idle_list 用 `entry`，忙碌时挂入 busy_hash 用 `hentry`。一个 worker 不可能同时空闲又忙碌。

## 3.3、`struct worker_pool`（工人池）

定义于 [kernel/workqueue.c:153](file:///home/xxx/linux_old1/kernel/workqueue.c#L153)。

每个 CPU 有 2 个标准的 **bound pool**（`NR_STD_WORKER_POOLS = 2`）：一个普通优先级（nice=0），一个高优先级（nice=-20）。此外还有动态创建的 unbound pool。

```c
struct worker_pool {
    raw_spinlock_t    lock;             // 池的自旋锁
    int               cpu;              // 关联的 CPU（-1 表示 unbound）
    int               node;             // NUMA 节点 ID
    int               id;               // 池的唯一 ID
    unsigned int      flags;            // POOL_MANAGER_ACTIVE | POOL_DISASSOCIATED
    int               nr_running;       // 正在运行的 worker 数
    struct list_head  worklist;         // ★ 待处理的工作项链表（核心队列）
    int               nr_workers;       // worker 总数
    int               nr_idle;          // 空闲 worker 数
    struct list_head  idle_list;        // 空闲 worker 链表
    struct timer_list idle_timer;       // 空闲超时定时器 (300s)
    struct timer_list mayday_timer;     // SOS 求救定时器 (10ms 后求援)
    DECLARE_HASHTABLE(busy_hash, ...);  // 忙碌 worker 哈希表
    struct worker     *manager;         // 当前 manager
    struct list_head  workers;          // 所有附属 worker 链表
    struct ida        worker_ida;       // worker ID 分配器
    struct workqueue_attrs *attrs;      // worker 属性 (nice值、cpumask 等)
    int               refcnt;           // unbound pool 的引用计数
};
```

## 3.4、`struct pool_workqueue`（每池工作队列）

定义于 [kernel/workqueue.c:227](file:///home/xxx/linux_old1/kernel/workqueue.c#L227)。

连接一个 `workqueue_struct` 到一个 `worker_pool`，是并发控制的关键结构。

```c
struct pool_workqueue {
    struct worker_pool    *pool;            // 关联的 worker_pool
    struct workqueue_struct *wq;            // 所属的 workqueue
    int    work_color;                      // 当前工作颜色
    int    flush_color;                     // flush 颜色
    int    nr_in_flight[WORK_NR_COLORS];    // 飞行中的工作数量（按颜色）
    int    nr_active;                       // ★ 活跃工作数（并发控制核心）
    int    max_active;                      // 最大活跃工作数
    struct list_head inactive_works;        // 因并发限制暂未激活的工作
    struct list_head pwqs_node;             // wq->pwqs 链表节点
    struct list_head mayday_node;           // wq->maydays 链表节点（求救）
};
```

**并发控制原理**：当 `pwq->nr_active >= max_active` 时，新 work 被放入 `inactive_works` 链表（标记 `WORK_STRUCT_INACTIVE`），而不是 `pool->worklist`。当一个活跃 work 执行完毕后，`pwq_activate_first_inactive()` 从 `inactive_works` 中激活下一个。

## 3.5、`struct workqueue_struct`（工作队列）

定义于 [kernel/workqueue.c:285](file:///home/xxx/linux_old1/kernel/workqueue.c#L285)。

```c
struct workqueue_struct {
    struct list_head    pwqs;              // 此 wq 的所有 pwq 链表
    struct list_head    list;              // 全局 workqueues 链表
    struct mutex        mutex;             // 保护此 wq 的互斥锁
    int    work_color;                     // 当前工作颜色 (flush 机制用)
    int    flush_color;                    // 当前 flush 颜色
    struct list_head    flusher_queue;     // flush 等待者队列
    struct list_head    flusher_overflow;  // flush 溢出列表
    struct list_head    maydays;           // 请求 rescuer 救援的 pwq 列表
    struct worker       *rescuer;          // rescuer worker (WQ_MEM_RECLAIM 时)
    int    saved_max_active;               // 保存的 max_active
    struct workqueue_attrs *unbound_attrs; // unbound wq 的属性
    struct pool_workqueue *dfl_pwq;        // unbound wq 的默认 pwq
    char   name[WQ_NAME_LEN];              // workqueue 名称
    unsigned int        flags;             // WQ_* 标志
    struct pool_workqueue __percpu __rcu **cpu_pwq; // per-CPU 的 pwq 指针
};
```

### Workqueue Flags（创建标志）

| Flag | 含义 |
|---|---|
| `WQ_UNBOUND` | 不绑定 CPU，worker 可在任意 CPU 上运行 |
| `WQ_FREEZABLE` | 系统挂起时冻结 worker |
| `WQ_MEM_RECLAIM` | 内存回收路径可用，创建 rescuer 线程 |
| `WQ_HIGHPRI` | 高优先级 worker（nice=-20） |
| `WQ_CPU_INTENSIVE` | CPU 密集型，不受并发管理限制 |
| `WQ_POWER_EFFICIENT` | 节能模式，per-CPU 模式下尽量 bound，否则自动转 unbound |
| `__WQ_ORDERED` | 严格按排队顺序串行执行 |

## 3.6、系统预定义 Workqueue

定义于 [kernel/workqueue.c:6596-6608](file:///home/xxx/linux_old1/kernel/workqueue.c#L6596)。

| 变量名 | 名称 | 用途 |
|---|---|---|
| `system_wq` | "events" | 通用系统 workqueue |
| `system_highpri_wq` | "events_highpri" | 高优先级，`WQ_HIGHPRI` |
| `system_long_wq` | "events_long" | 可能长时间运行的任务 |
| `system_unbound_wq` | "events_unbound" | 不绑核，`WQ_UNBOUND` |
| `system_freezable_wq` | "events_freezable" | 可冻结，`WQ_FREEZABLE` |
| `system_power_efficient_wq` | "events_power_efficient" | 节能，`WQ_POWER_EFFICIENT` |
| `system_freezable_power_efficient_wq` | "events_freezable_power_efficient" | 可冻结+节能 |

# 4、workqueue 完整生命周期函数调用链

> 基于 Linux 内核源码逐函数追踪

---

## 4.0、阶段 0：系统启动 — workqueue 框架初始化

workqueue 子系统采用 **三阶段初始化**，因为 work 执行需要内核线程，而线程创建需要在调度器就绪之后。

```c
start_kernel()                                                // init/main.c
  │
  ├─ workqueue_init_early()                                  // kernel/workqueue.c:6524 ★ 第一阶段
  │     │
  │     │  [1] 初始化 WQ_AFFN_SYSTEM 亲和性 pod
  │     ├─ 设置 wq_unbound_cpumask (从 housekeeping_cpumask 过滤)
  │     │
  │     │  [2] 为每个 CPU 创建并初始化 2 个 worker_pool
  │     └─ for_each_possible_cpu(cpu):
  │           for_each_cpu_worker_pool(pool, cpu):
  │             ├─ init_worker_pool(pool)                     // workqueue.c:3747
  │             │     ├─ raw_spin_lock_init(&pool->lock)
  │             │     ├─ INIT_LIST_HEAD(&pool->worklist)
  │             │     ├─ INIT_LIST_HEAD(&pool->idle_list)
  │             │     ├─ INIT_LIST_HEAD(&pool->workers)
  │             │     ├─ timer_setup(&pool->idle_timer, idle_worker_timeout)
  │             │     ├─ timer_setup(&pool->mayday_timer, pool_mayday_timeout)
  │             │     ├─ ida_init(&pool->worker_ida)
  │             │     ├─ hash_init(pool->busy_hash)
  │             │     └─ pool->attrs = alloc_workqueue_attrs()
  │             ├─ pool->cpu = cpu                            绑定到指定 CPU
  │             ├─ pool->attrs->nice = std_nice[i]            i=0: nice=0, i=1: nice=-20
  │             └─ worker_pool_assign_id(pool)                分配 pool ID
  │
  │     │  [3] 创建 unbound 和 ordered 的默认 attrs
  │     ├─ 各创建 NR_STD_WORKER_POOLS(2) 组 attrs
  │     │
  │     │  [4] 预分配所有系统级 workqueue
  │     ├─ system_wq = alloc_workqueue("events", 0, 0)
  │     ├─ system_highpri_wq = alloc_workqueue("events_highpri", WQ_HIGHPRI, 0)
  │     ├─ system_long_wq = alloc_workqueue("events_long", 0, 0)
  │     ├─ system_unbound_wq = alloc_workqueue("events_unbound", WQ_UNBOUND, WQ_MAX_ACTIVE)
  │     ├─ system_freezable_wq = alloc_workqueue("events_freezable", WQ_FREEZABLE, 0)
  │     ├─ system_power_efficient_wq = alloc_workqueue(..., WQ_POWER_EFFICIENT, 0)
  │     └─ system_freezable_power_efficient_wq = alloc_workqueue(...)
  │
  ├─ ... (其他子系统初始化，调度器就绪) ...
  │
  ├─ workqueue_init()                                         // kernel/workqueue.c:6662 ★ 第二阶段
  │     │
  │     │  [1] 初始化 CPU intensive 阈值检测
  │     ├─ wq_cpu_intensive_thresh_init()
  │     │     ├─ pwq_release_worker = kthread_create_worker(...)  创建 pwq 释放用 kthread_worker
  │     │     └─ wq_cpu_intensive_thresh_us = 10ms (根据 BogoMIPS 缩放)
  │     │
  │     │  [2] 为 WQ_MEM_RECLAIM 的 workqueue 创建 rescuer
  │     └─ list_for_each_entry(wq, &workqueues, list):
  │           └─ init_rescuer(wq)                             // workqueue.c:4862
  │                 └─ kthread_create(rescuer_thread, rescuer, "kworker/R-%s", wq->name)
  │
  │     │  [3] 为每个 worker_pool 创建初始 worker ★关键步骤★
  │     ├─ for_each_online_cpu(cpu):
  │     │     for_each_cpu_worker_pool(pool, cpu):
  │     │       ├─ pool->flags &= ~POOL_DISASSOCIATED         解除 offline 状态
  │     │       └─ create_worker(pool)                        创建第一个 kworker
  │     │
  │     └─ hash_for_each(unbound_pool_hash, ...):
  │           └─ create_worker(pool)
  │
  │     └─ wq_online = true                                    标记 workqueue 系统完全上线
  │
  └─ workqueue_init_topology()                                // kernel/workqueue.c:6774 ★ 第三阶段
        │
        │  [1] 初始化 NUMA/Cache/SMT 等亲和性 pod
        ├─ init_pod_type(&wq_pod_types[WQ_AFFN_CPU], ...)
        ├─ init_pod_type(&wq_pod_types[WQ_AFFN_SMT], ...)
        ├─ init_pod_type(&wq_pod_types[WQ_AFFN_CACHE], ...)
        ├─ init_pod_type(&wq_pod_types[WQ_AFFN_NUMA], ...)
        │
        │  [2] 应用 pod 拓扑并更新 unbound workqueue
        └─ apply_wqattrs_lock()
```

---

## 4.1、阶段 1：创建 workqueue — `alloc_workqueue()`

```c
alloc_workqueue(fmt, flags, max_active, ...)                  // kernel/workqueue.c:4672
  │
  │  [1] 参数校验和标志处理
  ├─ if (flags & WQ_UNBOUND && max_active == 1)
  │     flags |= __WQ_ORDERED            // 向后兼容：unbound + max_active=1 → ordered
  ├─ if (flags & WQ_POWER_EFFICIENT)
  │     flags |= WQ_UNBOUND              // 节能模式下自动转 unbound
  │
  │  [2] 分配和初始化 workqueue_struct
  ├─ wq = kzalloc(sizeof(*wq), GFP_KERNEL)
  ├─ vsnprintf(wq->name, WQ_NAME_LEN, fmt, args)
  ├─ wq->flags = flags
  ├─ wq->saved_max_active = max_active
  ├─ mutex_init(&wq->mutex)
  ├─ INIT_LIST_HEAD(&wq->pwqs)
  ├─ INIT_LIST_HEAD(&wq->list)
  │
  │  [3] 分配 per-CPU pool_workqueue 并链接 ★核心★
  └─ alloc_and_link_pwqs(wq)                                 // workqueue.c:4080
        │
        │  [3a] 初始化 wq->cpu_pwq (per-CPU 指针数组)
        ├─ wq->cpu_pwq = alloc_percpu(struct pool_workqueue *)
        │
        │  [3b] 对于 STANDARD (bound to CPU) workqueue:
        └─ for_each_possible_cpu(cpu):
              ├─ pool = cpu_worker_pools[cpu][highpri ? 1 : 0]  选择普通/高优先 pool
              ├─ pwq = kmem_cache_alloc_node(pwq_cache, ...)    分配 pool_workqueue
              ├─ init_pwq(pwq, wq, pool)                        初始化 pwq
              │     ├─ pwq->pool = pool
              │     ├─ pwq->wq = wq
              │     ├─ pwq->max_active = max_active
              │     └─ INIT_LIST_HEAD(&pwq->inactive_works)
              ├─ link_pwq(pwq)                                  挂入 wq->pwqs 链表
              └─ rcu_assign_pointer(per_cpu_ptr(wq->cpu_pwq, cpu), pwq)
        │
        │  [3c] 对于 UNBOUND workqueue:
        └─ apply_workqueue_attrs(wq, unbound_std_wq_attrs[highpri])
              └─ apply_wqattrs_prepare() → 创建 unbound pool + pwq
              └─ apply_wqattrs_commit()  → 更新 cpu_pwq 指针
  │
  │  [4] 如果设置了 WQ_MEM_RECLAIM，创建 rescuer 线程
  ├─ if (flags & WQ_MEM_RECLAIM):
  │     └─ init_rescuer(wq)                                   // workqueue.c:4862
  │           └─ rescuer->task = kthread_create(rescuer_thread, rescuer, ...)
  │
  │  [5] 加入全局 workqueues 链表
  └─ list_add_tail_rcu(&wq->list, &workqueues)
```

---

## 4.2、阶段 2：创建 worker — `create_worker()`

```c
create_worker(pool)                                           // kernel/workqueue.c:2165
  │
  │  [1] 分配 worker ID
  ├─ id = ida_alloc(&pool->worker_ida, GFP_KERNEL)
  │
  │  [2] 分配 worker 结构体
  ├─ worker = alloc_worker(pool->node)                        // workqueue.c:1980
  │     └─ kzalloc_node(sizeof(*worker), GFP_KERNEL, node)
  │
  │  [3] 创建内核线程 ★核心★
  ├─ worker->task = kthread_create_on_node(
  │       worker_thread,             // 线程主函数
  │       worker,                    // 参数 (worker 结构体)
  │       pool->node,
  │       "kworker/%s", id_buf)      // 命名: kworker/CPU:ID 或 kworker/uPOOL:ID
  │
  │  [4] 设置调度属性
  ├─ set_user_nice(worker->task, pool->attrs->nice)           设置 nice 值
  └─ kthread_bind_mask(worker->task, pool_allowed_cpus(pool))  绑定 CPU 亲和性
  │
  │  [5] 将 worker 挂入 pool ★关键步骤★
  ├─ worker_attach_to_pool(worker, pool)                      // workqueue.c:1893
  │     ├─ mutex_lock(&wq_pool_attach_mutex)
  │     ├─ worker->pool = pool                                关联 pool
  │     ├─ list_add_tail(&worker->node, &pool->workers)       挂入 workers 链表
  │     └─ set_cpus_allowed_ptr(worker->task, pool->attrs->cpumask)
  │
  │  [6] 唤醒 worker 线程
  ├─ raw_spin_lock_irq(&pool->lock)
  ├─ pool->nr_workers++
  ├─ worker_enter_idle(worker)                                // workqueue.c:2125
  │     ├─ worker->flags |= WORKER_IDLE
  │     ├─ list_add(&worker->entry, &pool->idle_list)         挂入空闲链表
  │     └─ pool->nr_idle++
  ├─ kick_pool(pool)                                          唤醒 pool
  ├─ wake_up_process(worker->task)                            确保线程被唤醒
  └─ raw_spin_unlock_irq(&pool->lock)
```

---

## 4.3、阶段 3：排队工作 — `queue_work()`

```c
用户调用: schedule_work(&my_work)  或  queue_work(my_wq, &my_work)
  │
  ▼
queue_work_on(cpu, wq, work)                                  // kernel/workqueue.c:1825
  │
  │  [1] 原子地设置 PENDING 标志 ★防重入★
  ├─ if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, &work->data))
  │     │  ← 如果 work 已经被排队 (PENDING=1)，直接返回 false
  │     │
  │     └─ __queue_work(cpu, wq, work)                        // workqueue.c:1702 ★核心★
  │           │
  │           │  [2] 确定目标 CPU → 获取目标 pwq
  │           ├─ pwq = rcu_dereference(*per_cpu_ptr(wq->cpu_pwq, cpu))
  │           ├─ pool = pwq->pool
  │           │
  │           │  [3] 检查是否需要排回原 pool ★保证非重入★
  │           ├─ last_pool = get_work_pool(work)              // 从 data 中提取上次执行的 pool
  │           ├─ if (last_pool && last_pool != pool):
  │           │     └─ if 原 pool 上有 worker 正在执行该 work:
  │           │           使用原 pool (保证同一 work 不会同时在两个 CPU 上执行)
  │           │
  │           │  [4] 活动计数 + 颜色标记
  │           ├─ pwq->nr_in_flight[pwq->work_color]++
  │           ├─ work_flags = work_color_to_flags(pwq->work_color)
  │           │
  │           │  [5] 并发控制判断 ★关键分支★
  │           ├─ if (pwq->nr_active < pwq->max_active):        ← 未达到并发上限
  │           │     ├─ pwq->nr_active++                        活跃计数 +1
  │           │     ├─ insert_work(pwq, work, &pool->worklist) 直接插入 pool 的待处理列表
  │           │     └─ kick_pool(pool)                         唤醒 pool 中的 worker
  │           │
  │           └─ else:                                         ← 达到并发上限
  │                 ├─ work_flags |= WORK_STRUCT_INACTIVE      标记为 inactive
  │                 └─ insert_work(pwq, work, &pwq->inactive_works) 放入 inactive 链表
  │
  └─ return true/false
```

---

## 4.4、阶段 4：执行工作 — `worker_thread()` → `process_one_work()`

### 4.4.1、worker 主循环

```c
worker_thread(__worker)                                       // kernel/workqueue.c:2729
  │
  │  [1] 标记当前进程为 workqueue worker
  ├─ set_pf_worker(true)                                      // current->flags |= PF_WQ_WORKER
  │
woke_up:
  ├─ raw_spin_lock_irq(&pool->lock)                           持锁检查
  │
  │  [2] 检查是否需要退出
  ├─ if (worker->flags & WORKER_DIE):                         ← 收到死亡信号
  │     ├─ raw_spin_unlock_irq(&pool->lock)
  │     ├─ set_pf_worker(false)
  │     ├─ set_task_comm(worker->task, "kworker/dying")       改名表示正在死亡
  │     ├─ ida_free(&pool->worker_ida, worker->id)            释放 ID
  │     ├─ worker_detach_from_pool(worker)                    从 pool 摘除
  │     └─ kfree(worker)                                      释放自身结构体
  │
  │  [3] 离开空闲状态
  ├─ worker_leave_idle(worker)                                // workqueue.c:2145
  │     ├─ pool->nr_idle--
  │     ├─ list_del_init(&worker->entry)                      从 idle_list 摘除
  │     └─ worker->flags &= ~WORKER_IDLE
  │
  │  [4] 检查是否需要更多 worker
recheck:
  ├─ if (!need_more_worker(pool))                              ← 不需要我了
  │     goto sleep
  │
  │  [5] 检查是否需要承担 manager 角色 ★并发管理★
  ├─ if (!may_start_working(pool) && manage_workers(worker))
  │     goto recheck          ← manager 创建了新 worker，重新检查
  │
  │  [6] 清除 PREP 和 REBOUND 标志，开始处理 work
  ├─ worker_clr_flags(worker, WORKER_PREP | WORKER_REBOUND)
  │
  │  [7] 循环处理 worklist 中的工作 ★核心执行循环★
  └─ do {
        ├─ work = list_first_entry(&pool->worklist, ...)      取出第一个 work
        ├─ if (assign_work(work, worker, NULL))               分配给当前 worker
        │     └─ process_scheduled_works(worker)              处理已安排的 work
        │           │
        │           └─ while (!list_empty(&worker->scheduled)):
        │                 ├─ work = list_first_entry(&worker->scheduled, ...)
        │                 └─ process_one_work(worker, work)   ★处理单个 work
        │
    } while (keep_working(pool))                              继续直到无事可做
  │
  │  [8] 无事可做 → 进入睡眠
sleep:
  ├─ worker_enter_idle(worker)                                重回 idle_list
  ├─ __set_current_state(TASK_IDLE)
  ├─ raw_spin_unlock_irq(&pool->lock)
  ├─ schedule()                                               ← 睡眠，等待唤醒
  └─ goto woke_up                                             ← 被唤醒后回到检查点
```

### 4.4.2、处理单个 work

```c
process_one_work(worker, work)                                // kernel/workqueue.c:2536
  │
  │  [1] 认领 work: 从 worklist 摘除 + 加入 busy_hash
  ├─ debug_work_deactivate(work)
  ├─ hash_add(pool->busy_hash, &worker->hentry, (unsigned long)work)
  ├─ worker->current_work = work                              记录当前 work
  ├─ worker->current_func = work->func                        记录当前函数
  ├─ worker->current_pwq = pwq                                记录所属 pwq
  ├─ worker->current_color = get_work_color(work_data)        记录颜色
  ├─ strscpy(worker->desc, pwq->wq->name, WORKER_DESC_LEN)   设置进程描述
  └─ list_del_init(&work->entry)                              从 worklist 摘除
  │
  │  [2] CPU intensive 处理
  ├─ if (wq->flags & WQ_CPU_INTENSIVE):
  │     └─ worker_set_flags(worker, WORKER_CPU_INTENSIVE)     不受并发管理
  │
  │  [3] 链式执行 kick（唤醒其他 worker 处理队列中剩余的 work）
  ├─ kick_pool(pool)                                           // workqueue.c:1310
  │
  │  [4] 清除 PENDING 标志 + 记录最后执行的 pool
  ├─ set_work_pool_and_clear_pending(work, pool->id)
  │     └─ work->data = pool->id << WORK_OFFQ_POOL_SHIFT      // 记录 pool id + 清除 flag
  │
  │  [5] 统计计数 + 释放锁 ★执行回调前释放锁★
  ├─ pwq->stats[PWQ_STAT_STARTED]++
  ├─ raw_spin_unlock_irq(&pool->lock)                         ← 释放锁，允许新的 work 排队
  │
  │  [6] 执行用户回调 ★实际工作在这里★
  ├─ trace_workqueue_execute_start(work)
  ├─ worker->current_func(work)                               ← 调用 work->func(work)
  ├─ trace_workqueue_execute_end(work, worker->current_func)
  ├─ pwq->stats[PWQ_STAT_COMPLETED]++
  │
  │  [7] 检漏：确保没在 atomic 或持有锁的上下文中返回
  ├─ if (unlikely(in_atomic() || lockdep_depth(current) > 0))
  │      pr_err("BUG: workqueue leaked lock or atomic: ...")
  │
  │  [8] 主动让出 CPU（防止非抢占内核上饥饿）
  ├─ cond_resched()
  │
  │  [9] 重新持锁，执行完成后处理 ★关键★
  ├─ raw_spin_lock_irq(&pool->lock)
  ├─ worker->current_work = NULL                               清除 current_work
  ├─ worker->current_func = NULL
  ├─ worker->current_pwq = NULL
  ├─ worker->current_color = INT_MIN
  ├─ hash_del(&worker->hentry)                                 从 busy_hash 摘除
  │
  │  [10] 激活 inactive work ★并发控制解除★
  ├─ pwq->nr_active--                                          活跃计数 -1
  ├─ if (!list_empty(&pwq->inactive_works))                    有等待中的 work
  │     └─ pwq_activate_first_inactive(pwq)                    激活第一个 inactive work
  │           ├─ work = list_first_entry(&pwq->inactive_works, ...)
  │           ├─ pwq->nr_active++                              活跃计数 +1
  │           ├─ move_linked_works(work, &pool->worklist, NULL) 移到 pool 的 worklist
  │           └─ kick_pool(pool)                               唤醒 worker 处理
  │
  │  [11] 递减 nr_in_flight（用于 flush 机制）
  ├─ pwq->nr_in_flight[pwq->work_color]--
  │
  │  [12] 更新颜色（所有 work 处理完毕时）
  └─ pwq_adjust_max_active(pwq)
```

---

## 4.5、阶段 5：并发管理（Manager 机制）

```c
manage_workers(worker)                                        // kernel/workqueue.c:2505
  │
  │  [1] 检查是否已经是 manager（同一时刻只有一个）
  ├─ if (pool->flags & POOL_MANAGER_ACTIVE)
  │     return false           ← 已经有 manager，我不能当
  │
  │  [2] 标记为 manager
  ├─ pool->flags |= POOL_MANAGER_ACTIVE
  ├─ pool->manager = worker
  │
  │  [3] 尝试创建 worker
  └─ maybe_create_worker(pool)                                // workqueue.c:2450
        │
        │  [3a] 设置 mayday 定时器（如果 10ms 内没创建成功，求救）
        ├─ mod_timer(&pool->mayday_timer, jiffies + MAYDAY_INITIAL_TIMEOUT)
        │     └─ 超时后 → pool_mayday_timeout()               // workqueue.c:2407
        │           └─ 遍历 pool->worklist 中的每个 work:
        │                 └─ send_mayday(work)                 发送求救信号
        │                       └─ 如果 work 所属 wq 有 rescuer:
        │                             ├─ list_add(&pwq->mayday_node, &wq->maydays)
        │                             └─ wake_up_process(wq->rescuer->task)
        │
        │  [3b] 循环尝试创建 worker (直到不需要或创建成功)
        └─ while (true):
              ├─ if (create_worker(pool)) break;              创建成功 → 退出
              ├─ if (!need_to_create_worker(pool)) break;     不需要了 → 退出
              └─ schedule_timeout_interruptible(CREATE_COOLDOWN) 休息 1s 再试
        │
        │  [3c] 解除 manager 角色
        ├─ pool->flags &= ~POOL_MANAGER_ACTIVE
        └─ pool->manager = NULL
```

**mayday 机制**：当 worker 创建失败（通常是因为 GFP_KERNEL 分配失败），pool 会通过 mayday 定时器向所有 `WQ_MEM_RECLAIM` 的 workqueue 的 rescuer 线程求救，让 rescuer 来执行紧急工作，从而释放内存。

---

## 4.6、阶段 6：rescuer 救援线程

```c
rescuer_thread(__rescuer)                                     // kernel/workqueue.c:2824
  │
  │  [1] 设置高优先级 (nice=-20)
  ├─ set_user_nice(current, RESCUER_NICE_LEVEL)
  │
repeat:
  ├─ set_current_state(TASK_IDLE)
  │
  │  [2] 轮询 mayday 链表
  ├─ while (!list_empty(&wq->maydays)):
  │     ├─ pwq = list_first_entry(&wq->maydays, ...)
  │     ├─ pool = pwq->pool
  │     ├─ list_del_init(&pwq->mayday_node)                   从 mayday 链表摘除
  │     │
  │     │  [3] 将 rescuer attach 到求救的 pool
  │     ├─ worker_attach_to_pool(rescuer, pool)
  │     │
  │     │  [4] 从 pool->worklist 中捞出属于求救 wq 的所有 work
  │     └─ list_for_each_entry_safe(work, n, &pool->worklist, entry):
  │           └─ if (get_work_pwq(work) == pwq):
  │                 assign_work(work, rescuer, &n)
  │                 pwq->stats[PWQ_STAT_RESCUED]++
  │     │
  │     │  [5] 批量执行捞出的 work
  │     └─ process_scheduled_works(rescuer)
  │           │
  │           │  [6] 检查是否因为执行过程中产生了新的 work
  │           └─ if (pwq->nr_active && need_to_create_worker(pool)):
  │                 └─ list_add_tail(&pwq->mayday_node, &wq->maydays)  重新排入 mayday
  │
  ├─ if (kthread_should_stop()) return 0                      收到停止信号 → 退出
  └─ schedule() → goto repeat
```

---

## 4.7、阶段 7：销毁 workqueue — `destroy_workqueue()`

```c
destroy_workqueue(wq)                                         // kernel/workqueue.c:4786
  │
  │  [1] 标记为 DESTROYING（阻止新的 queue_work）
  ├─ wq->flags |= __WQ_DESTROYING
  │
  │  [2] 排空所有正在执行和等待的 work
  ├─ drain_workqueue(wq)                                      // workqueue.c:3174
  │     └─ 循环等待所有 pwq 的 nr_in_flight 和 nr_active 归零
  │
  │  [3] 清除 flush 相关状态
  ├─ flush_workqueue(wq)
  │
  │  [4] 释放 rescuer
  ├─ if (wq->rescuer):
  │     ├─ kthread_stop(wq->rescuer->task)                    通知 rescuer 线程退出
  │     ├─ wq->rescuer = NULL
  │     └─ kfree(rescuer)
  │
  │  [5] 释放所有 pwq
  ├─ for_each_pwq(pwq, wq):
  │     └─ put_pwq(pwq)                                      递减引用计数
  │           └─ if refcnt == 0:
  │                 └─ pwq_release_workfn()                   延迟释放（RCU 安全）
  │
  │  [6] 解除 sysfs 注册 + free_percpu(cpu_pwq)
  ├─ free_percpu(wq->cpu_pwq)
  │
  │  [7] 从全局链表摘除 + RCU 延迟释放
  ├─ list_del_rcu(&wq->list)
  └─ call_rcu(&wq->rcu, rcu_free_wq)                          RCU 保护下释放
```

---

# 5、总结：完整调用链速查表

| 阶段 | 操作 | 内核入口函数 | 核心函数 | 所在文件 |
|:-----|:-----|:------------|:---------|:---------|
| 0. 初始化 (early) | 系统启动 | `workqueue_init_early()` | `init_worker_pool()` → `alloc_workqueue()` (创建 7 个系统 wq) | workqueue.c:6524 |
| 0. 初始化 (late) | 调度器就绪后 | `workqueue_init()` | `init_rescuer()` → `create_worker()` | workqueue.c:6662 |
| 0. 初始化 (topo) | 拓扑就绪后 | `workqueue_init_topology()` | `init_pod_type()` → `apply_wqattrs_lock()` | workqueue.c:6774 |
| 1. 创建 workqueue | `alloc_workqueue("name", flags, max_active)` | 同左 | `alloc_and_link_pwqs()` → `init_pwq()` → `link_pwq()` | workqueue.c:4672 |
| 2. 创建 worker | (manager 触发) | `create_worker()` | `kthread_create_on_node(worker_thread)` → `worker_attach_to_pool()` → `wake_up_process()` | workqueue.c:2165 |
| 3. 排队 work | `queue_work(wq, &work)` | `queue_work_on()` | `test_and_set_bit(PENDING)` → `__queue_work()` → `insert_work()` → `kick_pool()` | workqueue.c:1825 |
| 4. 执行 work | (kworker 自动) | `worker_thread()` | `process_scheduled_works()` → `process_one_work()` → `worker->current_func(work)` | workqueue.c:2729 |
| 5. 并发管理 | (kworker 自动) | `manage_workers()` | `maybe_create_worker()` → `create_worker()` / `send_mayday()` | workqueue.c:2505 |
| 6. rescuer 救援 | (mayday 触发) | `rescuer_thread()` | `process_scheduled_works(rescuer)` → `process_one_work()` | workqueue.c:2824 |
| 7. 销毁 | `destroy_workqueue(wq)` | 同左 | `drain_workqueue()` → `flush_workqueue()` → `put_pwq()` → `call_rcu()` | workqueue.c:4786 |

**关键设计要点：**

1. **PENDING 标志 + 原子操作** — 防止同一个 work 被重复排队
2. **last_pool 机制** — 保证同一个 work 不会同时在两个 CPU 上执行（非重入）
3. **nr_active vs max_active** — 控制每个 workqueue 在每个 pool 上的并发数
4. **WORK_STRUCT_INACTIVE** — 被并发限制搁置的 work，存在 inactive_works
5. **执行时释放锁** — `work->func()` 执行期间不持 pool->lock，新的 work 可以继续排队
6. **work_color / flush_color** — 实现 `flush_workqueue()` 的屏障语义
7. **mayday 定时器 + rescuer** — 内存分配死锁场景下的救援机制
8. **idle_timer (300s)** — 超时回收多余的 idle worker
9. **manager 机制** — 保证同一时刻只有一个 worker 负责创建新 worker
