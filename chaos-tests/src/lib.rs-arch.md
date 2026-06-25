# `chaos-tests/src/lib.rs` 架构文档

> 本文档对 chaos-tests 测试库的核心文件 `lib.rs` 进行全面的结构解析。  
> 该文件约 6400 行，实现了一个完整的**模拟操作系统内核**，覆盖进程管理、内存管理、文件系统、IPC、同步原语等 OS 核心子系统。

---

## 目录

1. [整体架构](#1-整体架构)
2. [常量定义](#2-常量定义)
3. [基础数据结构](#3-基础数据结构)
4. [同步原语](#4-同步原语)
5. [内存管理](#5-内存管理)
6. [虚拟内存映射](#6-虚拟内存映射-vmmap)
7. [ELF 加载与网络辅助](#7-elf-加载与网络辅助)
8. [文件系统与文件抽象](#8-文件系统与文件抽象)
9. [通道与页面缓存](#9-通道与页面缓存)
10. [设备与 IO 子系统](#10-设备与-io-子系统)
11. [进程间通信 IPC](#11-进程间通信-ipc)
12. [进程与线程](#12-进程与线程)
13. [内核主体 Kernel](#13-内核主体-kernel)
14. [地址空间与 COW](#14-地址空间与-cow)
15. [进程组与资源限制](#15-进程组与资源限制)
16. [伙伴分配器 BuddyAllocator](#16-伙伴分配器-buddyallocator)
17. [工具函数](#17-工具函数)
18. [数据流全景图](#18-数据流全景图)

---

## 1. 整体架构

`lib.rs` 中的代码组织遵循**自底向上**的层级结构：

```
┌──────────────────────────────────────────────────────┐
│ Layer 6: 工具函数                                     │
│   hash / CRC / bit-ops / BuddyAllocator              │
├──────────────────────────────────────────────────────┤
│ Layer 5: 内核主体 (Kernel)                            │
│   dispatch_syscall / schedule / fork / exec          │
├──────────────────────────────────────────────────────┤
│ Layer 4: 进程子系统                                   │
│   Task / TaskTable / RunQueue / TrapCtl / Context    │
├──────────────────────────────────────────────────────┤
│ Layer 3: 文件/IO 子系统                               │
│   FHandle / PipeNode / FLike / EpInst / Disk / IoQ  │
├──────────────────────────────────────────────────────┤
│ Layer 2: 内存 & 同步                                  │
│   VmRegion / FramePool / KernLock / Spin / Sema     │
├──────────────────────────────────────────────────────┤
│ Layer 1: 常量 & 基础结构                              │
│   PAGE_SZ / SYS_* / 枚举 / 配置常量                    │
└──────────────────────────────────────────────────────┘
```

---

## 2. 常量定义

文件开头（行 13–168）定义了内核所有子系统使用的核心常量。

### 2.1 内存布局常量

| 常量 | 值 | 含义 |
|------|------|------|
| `PAGE_SZ` | 4096 | 页帧大小（4KB） |
| `N_PROC` | 256 | 最大进程数 |
| `N_FRAMES` | 65536 | 物理帧总数 |
| `KERN_BASE` | `0xFFFF_FFFF_8000_0000` | 内核虚拟地址基址 |
| `PHYS_OFF` | `0xFFFF_FFFF_0000_0000` | 物理内存偏移基址 |
| `MEM_OFF` | `0x8000_0000` | 物理内存起始 \\
| `KHEAP_SZ` | `0x800000` (8MB) | 内核堆大小 |
| `USR_STK_OFF` | `0x7FFF_0000` | 用户栈顶 |
| `USR_STK_SZ` | `0x10000` (64KB) | 用户栈大小 |

### 2.2 系统调用号

```
SYS_READ(0)  SYS_WRITE(1)   SYS_OPEN(2)    SYS_CLOSE(3)
SYS_MMAP(9)  SYS_MUNMAP(11) SYS_BRK(12)    SYS_IOCTL(16)
SYS_PIPE(22) SYS_DUP(32)    SYS_DUP2(33)   SYS_FORK(57)
SYS_EXEC(59) SYS_EXIT(60)   SYS_WAIT4(61)  SYS_KILL(62)
SYS_FCNTL(72)               SYS_FUTEX(202) SYS_EPOLL_*(213/232/233)
SYS_CLOCK_GETTIME(228)      SYS_SIGACTION(13) SYS_SIGPROCMASK(14)
```

### 2.3 其他关键常量

- **MMAP 标志**: `VM_READ(0x01)`, `VM_WRITE(0x02)`, `VM_EXEC(0x04)`, `VM_SHARED(0x08)`, `VM_GROWSDOWN(0x10)`
- **调度**: `SCHED_NORMAL(0)`, `SCHED_FIFO(1)`, `SCHED_RR(2)`, `SCHED_BATCH(3)`
- **信号**: `SIGKILL(9)`, `SIGSTOP(19)`, `SIGCHLD(17)`, `SIGUSR1(10)`, `SIGUSR2(12)`
- **Capability**: `CAP_CHOWN(0)`, `CAP_SYS_ADMIN(21)`, `CAP_SYS_PTRACE(19)` 等

---

## 3. 基础数据结构

### 3.1 `VmRegion` — 虚拟内存区域

```rust
pub struct VmRegion {
    pub base: usize,           // 起始虚拟地址
    pub len: usize,            // 区域长度
    pub flags: u32,            // 访问标志 (VM_READ|VM_WRITE|...)
    pub offset: usize,         // 文件偏移（mmap 场景）
    pub tag: u16,              // 区域标签
    pub ref_count: AtomicUsize, // 原子引用计数
}
```

**核心方法**：
- `contains(addr)` / `overlaps(other)` — 地址/重叠检测
- `split_at(addr)` — 在指定地址处分裂区域
- `merge_with(other)` — 尝试合并相邻且属性相同的区域
- `end()` — 计算区域的结束地址

### 3.2 `CapSet` — 权限集（Linux Capabilities）

```
bits: 允许的能力位图
effective: 当前生效的能力位图
ambient: 环境能力位图（ambient capabilities）
```

关键操作：`check(cap)` 权限检查、`grant(cap)` 授予、`drop_cap(cap)` 移除、`inherit(parent)` 从父进程继承。

### 3.3 `SigSet` / `SigAction` — 信号子系统

```rust
pub struct SigSet {
    pub pending: u64,          // 挂起信号位图
    pub blocked: u64,          // 阻塞信号位图
    pub actions: Vec<SigAction>, // 每信号的处理动作
}
```

- `sig_raise(signo)` — 向自身发送信号
- `deliverable()` — 寻找第一个可递交的信号
- `sig_block(mask)` / `sig_unblock(mask)` — 阻塞/解除阻塞
- `coalesce_pending()` — 合并挂起信号（同一信号只保留一次）

### 3.4 事件系统 — `EvBus` / `EvFlag`

```rust
pub struct EvBus {
    pub ev: u32,                          // 当前事件位掩码
    pub cbs: Vec<Box<dyn Fn(u32) -> bool + Send>>, // 回调链
}
```

事件类型：
- `READABLE` / `WRITABLE` / `ERROR` / `CLOSED` — IO 就绪事件
- `PROC_QUIT` — 进程退出
- `CHILD_QUIT` — 子进程退出
- `RECV_SIG` — 接收到信号
- `SEM_RM` / `SEM_ACQ` — 信号量移除/可获取

---

## 4. 同步原语

### 4.1 `KernLock` — 内核大锁（BKL 模拟）

```
flag:  AtomicBool    — 锁状态
holder: AtomicUsize  — 持有者 ID
depth:  AtomicUsize  — 递归深度（允许嵌套持有）
```

- `enter(id)` — 进入临界区，支持递归重入
- `leave()` — 离开临界区，深度归零后释放
- `try_enter(id)` — 非阻塞尝试

### 4.2 `Spin` — 自旋锁

基于 CAS 的标准自旋锁实现：

```rust
pub fn acquire(&self) {
    while self.v.compare_exchange(false, true, ...).is_err() {
        core::hint::spin_loop();  // PAUSE 指令等效
    }
}
```

### 4.3 `SyncQueue` — 等待队列

内核中**使用最广泛**的同步机制：

| 方法 | 功能 |
|------|------|
| `park_on(g, pred)` | 条件不满足时阻塞当前线程 |
| `signal()` | 唤醒一个等待者 |
| `broadcast()` | 唤醒所有等待者 |
| `signal_n(n)` | 唤醒至多 n 个等待者 |
| `wait_ev(g, cond)` | 等待事件条件满足 |
| `wait_timeout(g, timeout)` | 带超时的等待 |
| `reg_epoll(task_id, epfd, fd)` | epoll 注册 |

### 4.4 `Sema` — 信号量

基于 CAS 自旋的信号量实现（非基于 SyncQueue）：

- `new(count)` — 创建计数信号量
- `try_acquire()` — 非阻塞 P 操作
- `acquire_spin()` — 自旋等待 P 操作
- `release()` — V 操作
- `remove()` — 标记信号量为已删除

### 4.5 `FutexBucket` / `FutexTable` — Futex 支持

两套 futex 实现：
- **`FutexBucket`** — 分桶哈希表实现，支持 `wait`/`wake`/`requeue`
- **`FutexTable`** — 基于全局双端队列的简化实现

---

## 5. 内存管理

### 5.1 地址转换

```rust
p2v(pa) : 物理地址 → 虚拟地址  (PHYS_OFF + pa)
v2p(va) : 虚拟地址 → 物理地址  (va - PHYS_OFF)
k_off(va): 内核地址 → 偏移量  (va - KERN_BASE)
```

### 5.2 `PgFrame` — 物理页帧描述符

```rust
pub struct PgFrame { pub rc: AtomicUsize }
```

纯原子引用计数的物理页帧管理：
- `up()` / `down()` — 原子增减引用计数
- `inc_if_nonzero()` — 仅当计数非零时增加（用于 COW 快照场景）
- `cas(expected, desired)` — CAS 比较并交换

### 5.3 `FramePool` — 物理帧池

```rust
pub struct FramePool {
    slots: Mutex<Vec<bool>>,  // 位图：true=空闲
    cap: usize,               // 总帧数
}
```

分配策略：
| 方法 | 策略 |
|------|------|
| `get()` | 顺序扫描空闲帧（带全局锁） |
| `get_contig(sz, align)` | 连续帧分配 |
| `get_zone_aware(zone)` | 按内存区域分配（zone-aware） |
| `batch_alloc(count)` | 批量分配 |
| `put(idx)` | 单帧归还 |

### 5.4 `ZoneInfo` — 内存区域

模拟 Linux 的 DMA/NORMAL/HIGH 内存区域：
- `zone_can_alloc()` — 检查是否可分配（高于 low watermark）
- `zone_pressure()` — 计算内存压力百分比
- `reclaim_target()` — 回收目标页数

### 5.5 `SharedPage` — COW 共享页

```rust
pub struct SharedPage {
    pub frame: AtomicUsize,  // 物理帧号
    pub w: AtomicBool,       // 是否已私有化（writable）
    pub pending: AtomicBool, // 是否仍处于 COW 状态
}
```

`fault(pool, src)` — 处理写时复制缺页：分配新帧，降低原帧引用计数。

---

## 6. 虚拟内存映射 — `VmMap`

```rust
pub struct VmMap {
    pub regions: Vec<VmRegion>,  // 按地址排序的区域列表
    pub brk: usize,              // 程序段尾（brk）
    pub mmap_base: usize,        // mmap 分配基址
}
```

### 核心算法

| 方法 | 算法/策略 |
|------|----------|
| `insert(region)` | 有序插入，检测重叠 |
| `find(addr)` | **二分查找**定位地址所在区域 |
| `remove_range(base, len)` | 删除范围内所有区域 |
| `find_free(len, align)` | 顺序扫描寻找空闲地址区间 |
| `clone_regions()` | 深拷贝（fork 时使用） |

---

## 7. ELF 加载与网络辅助

### 7.1 `validate_elf_header(data)` — ELF64 验证

```
1. 检查魔数 0x7f 'E' 'L' 'F' (行 1414)
2. 验证 EI_CLASS=2  (64位)  (行 1418)
3. 验证 EI_DATA=1   (小端)  (行 1420)
4. 检查 e_type=2/3  (可执行/共享) (行 1424)
5. 解析程序头表 (行 1447-1458):
   - p_type=1 (PT_LOAD) → 加载段计数
   - p_type=3 (PT_INTERP) → 解释器标记
6. 返回 e_entry (入口地址)
```

返回值条件：
- `load_count == 0` → 返回错误 `"no_load"`
- `load_count > 0` → 返回 `Ok(entry_point)`

### 7.2 TCP/IP 辅助

- `tcp_checksum(src_ip, dst_ip, payload)` — TCP 伪首部+负载校验和
- `parse_ipv4_header(pkt)` — 解析源/目标 IP、协议、总长度
- `compute_inet_checksum(data)` — Internet 16位校验和

---

## 8. 文件系统与文件抽象

### 8.1 `FHandle` — 文件句柄

```rust
pub struct FHandle {
    pub path: String,                      // 文件路径
    pub data: Arc<Mutex<Vec<u8>>>,         // 文件内容（共享所有权）
    desc: Arc<RwLock<FdState>>,            // 状态：偏移量、选项、锁
    pub pipe: bool,                        // 是否为管道
    pub cloexec: bool,                     // close-on-exec 标志
}
```

**读写模式**：

```
read(off, buf)           — 从当前偏移读取并前进
read_at(off, buf)        — 从指定偏移读取（不改变偏移）
write(off, buf)          — 从当前偏移写入
write_at(off, buf)       — 从指定偏移写入

FdOpt.ap=true (O_APPEND) — 写入前将偏移定位到文件末尾
FdOpt.nb=true (O_NONBLOCK)— 非阻塞模式
```

**高级操作**：
- `transfer(dir, offset, buf_rd, buf_wr)` — 统一读写传输
- `splice_to(dst, count)` — 零拷贝数据传输
- `seek(FSeek)` — 支持 Start/End/Cur 三种定位
- `set_len(len)` — 截断/扩展文件
- `fallocate(offset, len)` — 预分配文件空间

### 8.2 `PipeNode` / `PipeBuf` — 匿名管道

```rust
pub struct PipeBuf {
    pub buf: VecDeque<u8>,  // 数据缓冲区
    pub bus: EvBus,          // 事件总线（通知可读）
    pub ends: i32,           // 存活端数（2=两端都在，1=一端关闭）
}
```

- `PipeNode::pair()` → 创建 (读端, 写端) 对
- `write_at` → 写入数据并触发 `READABLE` 事件
- `read_at` → 读取数据；若缓冲区空且写端关闭，返回错误
- `Drop` → 减少 `ends` 计数，广播 `CLOSED` 事件

### 8.3 `FLike` — 统一文件描述符类型

```rust
pub enum FLike {
    File(FHandle),    // 普通文件
    Pipe(PipeNode),   // 管道端
    Ep(EpInst),       // epoll 实例
}
```

实现了多态派发：
| 方法 | File | Pipe | Ep |
|------|------|------|-----|
| `read()` | 按偏移读取 | 从管道缓冲区读取 | ENOSYS |
| `write()` | 按偏移写入（支持 APPEND） | 写入管道并通知 | ENOSYS |
| `io_ctl()` | 委派给 FHandle | 支持 FIONBIO | ENOSYS |
| `poll()` | 基于 opt.rd/opt.wr | 基于缓冲区状态 | 基于 ready 集合 |
| `dup()` | 克隆引用 | 创建新端共享数据 | 克隆引用 |

### 8.4 Epoll 实现

```rust
pub struct EpInst {
    pub events: BTreeMap<usize, EpEvent>,    // fd → 监控事件
    pub ready: Arc<Mutex<BTreeSet<usize>>>,  // 就绪 fd 集合
    pub new_ctl: Arc<Mutex<BTreeSet<usize>>>,// 新注册的 fd
}
```

- `control(op, fd, ev)` — ADD(1)/MOD(3)/DEL(2) 三种操作
- `EpEvent` 标志位：`IN(0x001)`, `OUT(0x004)`, `ERR(0x008)`, `HUP(0x010)`, `ET(1<<31)`, `ONESHOT(1<<30)` 等

### 8.5 `TrmIO` / `WinSz` — 终端 IO 结构

```rust
pub struct TrmIO {
    pub iflag: u32,  // 输入标志（BRKINT, ICRNL 等）
    pub oflag: u32,  // 输出标志
    pub cflag: u32,  // 控制标志（波特率等）
    pub lflag: u32,  // 行规程标志（ICANON, ECHO 等）
    pub cc: [u8; 32], // 控制字符（VINTR, VEOF 等）
    pub ispeed/ospeed: u32,
}
```

---

## 9. 通道与页面缓存

### 9.1 `Channel` — 阻塞通信通道

```rust
pub struct Channel {
    pub buf: Mutex<CircBuf>,   // 环形缓冲区
    pub guard: Spin,           // 互斥自旋锁
    pub wq: SyncQueue,         // 等待队列
    pub shut: AtomicBool,      // 关闭标志
}
```

- `send(v)` → 写入数据并唤醒一个接收者
- `recv()` → 阻塞接收；缓冲区空时 park 等待
- `try_recv()` → 非阻塞接收
- `send_batch(data)` → 批量写入
- `drain_all()` → 清空所有数据
- `close()` → 关闭通道，唤醒所有等待者

### 9.2 `PageCache` — LRU 页面缓存

```rust
pub struct PageCache {
    pub entries: HashMap<usize, PageCacheEntry>,
    pub lru_order: VecDeque<usize>,   // LRU 链表
    pub hits/misses/evictions: AtomicUsize,
}
```

- `lookup(page_id)` — 查找并更新访问时间
- `insert(page_id, data)` — 插入，满时触发 LRU 淘汰
- `evict_lru()` — 淘汰最久未使用且未 pin 的页面
- `pin(page_id)` / `unpin(page_id)` — 增加/减少 pin 计数
- `writeback_all()` — 写回所有脏页

### 9.3 `BlockCache` / `CacheChain` — 分链块缓存

```rust
pub struct BlockCache {
    pub chains: Vec<CacheChain>,  // N 条独立链
    pub width: usize,             // 分链数量 (N_CHAINS=64)
}
```

- `fetch(key, latency)` — 读取块（带模拟延迟），未命中时生成伪数据
- `sync_all(id)` — 全局同步所有修改块
- `evict_cold(max_age)` — 淘汰冷块

---

## 10. 设备与 IO 子系统

### 10.1 `Disk` — 模拟磁盘

```rust
pub struct Disk {
    pub errs: AtomicUsize,    // 错误计数（>0 时模拟 IO 错误）
    pub ops: AtomicUsize,     // 操作计数
    pub label: String,        // 磁盘标签
    pub journal: Option<Arc<Disk>>, // 日志设备
}
```

**错误注入机制**：
- `errs > 0` → 模拟 IO 错误
- `errs == usize::MAX` → 持久错误
- 每次失败后 `errs.fetch_sub(1)`，错误用尽后恢复正常

### 10.2 `IoQueue` — IO 调度队列

```
调度算法：逐级电梯算法（SCAN/C-LOOK 变体）
- 维护 current_head 和方向标志
- 在 pending 队列中选择距离最近的请求
- 当同方向无请求时翻转方向
```

- `submit(blk, write, priority)` — 提交 IO 请求
- `submit_batch(requests)` — 批量提交，深度超限时合并相邻请求
- `dispatch()` — 调度一个请求（SCAN 算法）
- `merge_adjacent()` — 合并相邻且同方向的请求

### 10.3 `MountTable` — 挂载表

```rust
pub struct MountTable {
    pub entries: RwLock<Vec<MountEntry>>, // 按前缀长度降序排列
}
```

- `bind(prefix, target)` — 绑定/挂载
- `resolve(path)` → **最长前缀匹配**递归解析
- `unmount(prefix)` — 卸载
- `find_mount(path)` — 查找匹配的挂载点

### 10.4 `KObjRegistry` — 内核对象注册表

带树形引用的对象追踪系统：
- `register(type_tag, owner_pid)` — 注册对象
- `register_child(type_tag, owner_pid, parent)` — 注册带父引用的对象
- `dump_graph()` — 导出父子关系图
- `gc_sweep()` — 回收 ref_count=0 的对象

---

## 11. 进程间通信 IPC

### 11.1 System V 信号量

```rust
pub struct SemArr {
    pub ds: Mutex<SemDs>,   // 信号量集描述符
    pub sems: Vec<Sema>,    // 信号量数组
}
```

- `get_or_create(key, nsems, flags, store)` — 全局获取或创建
- `set_ds(new)` — 更新描述符
- 信号量数组存储在 `Kernel.sem_store`（全局 Weak 引用表）中

### 11.2 每进程信号量上下文 `SemCtx`

```rust
pub struct SemCtx {
    pub arrays: BTreeMap<SemId, Arc<SemArr>>,  // 持有的信号量集
    pub undos: BTreeMap<(SemId, SemNum), SemOp>, // undo 记录
}
```

- `Drop` 时自动执行 undo 操作（释放信号量）
- `Clone` 时不复制 undo 记录

### 11.3 System V 共享内存

```rust
pub fn shm_get_or_create(key, npages, store) -> Arc<Mutex<Vec<usize>>>
```

- 存储在 `Kernel.shm_store` 全局 Weak 引用表中
- `ShmCtx.ids` 维护每个进程的共享内存段

---

## 12. 进程与线程

### 12.1 `Context` — CPU 上下文

```rust
pub struct Context {
    pub r: [u64; N_REGS],  // 16 个通用寄存器
    pub ip: u64,           // 指令指针
    pub flags: u64,        // 标志寄存器
}
```

核心操作：
- `capture(src)` — 从寄存器快照创建上下文
- `syscall_args()` — 提取 syscall 6 个参数
- `set_ret(r0)` / `set_sp(rsp)` / `set_ip(rip)` / `set_tls(fs)`
- `transform(op, val)` — 按操作码修改特定寄存器
- `diff(other)` — 比较两个上下文的差异
- `hash()` — FNV-1a 哈希（用于上下文校验）

### 12.2 `TrapCtl` — 中断/异常控制器

```rust
pub struct TrapCtl {
    pub hw_mask: AtomicU32,  // 硬件中断掩码
    pub sw_mask: AtomicU32,  // 软件中断掩码
    pub nest: AtomicUsize,   // 嵌套深度
    pub frame: Mutex<Option<Context>>,  // 当前异常帧
}
```

- `dispatch(ctx)` — 保存当前上下文并进入异常处理
- `dispatch_vector(vector, ctx)` — 按向量号分发：
  - 0-7: 硬件中断
  - 8-15: 软件中断
  - 14: **缺页异常**特殊处理
- `on_pgfault(va)` — 缺页处理
- `handle_irq(ctx)` — IRQ 处理（嵌套保护）

### 12.3 `SchedulePolicy` / `RunQueue` — CFS 调度

```rust
pub struct SchedulePolicy {
    pub policy: u8,        // 调度策略
    pub prio: i32,         // 静态优先级
    pub nice: i32,         // nice 值
    pub vruntime: u64,     // 虚拟运行时间
}
```

**调度算法**：
- 按 `prio * 1000 + vruntime - weight` 排序
- `weight()` 函数根据 nice 值映射权重：
  - nice -10: 88761, nice 0: 1024, nice 10: 335
- `rebalance()` — 更新所有任务的 vruntime
- `preempt_enable()` / `preempt_disable()` — 抢占控制
- `boost_priority(task_id, amount)` — 优先级提升

### 12.4 `Task` — 进程/线程核心结构

```rust
pub struct Task {
    pub info: Mutex<TaskInfo>,          // 基本信息
    pub parent: Mutex<Option<Arc<Task>>>,// 父进程
    pub subtasks: Mutex<Vec<Arc<Task>>>,// 子进程列表
    pub files: Mutex<BTreeMap<usize, FLike>>, // 文件描述符表
    pub cwd: Mutex<String>,             // 当前工作目录
    pub exec_path: Mutex<String>,       // 可执行文件路径
    pub futexes: Mutex<BTreeMap<usize, Arc<FutexBucket>>>,// futex 表
    pub sem_ctx: Mutex<SemCtx>,         // 信号量上下文
    pub shm_ctx: Mutex<ShmCtx>,         // 共享内存上下文
    pub pid: Mutex<Pid>,                // 进程 ID
    pub pgid: Mutex<Pgid>,              // 进程组 ID
    pub threads: Mutex<Vec<Tid>>,       // 线程列表
    pub ev: Arc<Mutex<EvBus>>,          // 事件总线
    pub exit_code: Mutex<usize>,        // 退出码
    pub sig_queue: Mutex<VecDeque<(i32, isize)>>, // 信号队列
    pub sig_mask: Mutex<u64>,           // 信号掩码
    pub ep_inst: Mutex<BTreeMap<usize, EpInst>>, // epoll 实例
    pub kstk: Mutex<Option<KStk>>,      // 内核栈
    pub thd_ctx: Mutex<Option<ThdCtx>>, // 线程上下文
    pub vm_token: AtomicUsize,          // 地址空间令牌
}
```

**关键操作**：
- `exit_proc(code)` — 关闭所有 FD、通知父进程、广播 PROC_QUIT
- `begin_run()` / `end_run(cx)` — 线程上下文切换
- `send_sig(signo, sender_tid)` — 向进程发送信号
- `has_sig()` — 检查是否有可递交的未阻塞信号

### 12.5 `TaskTable` — 全局任务表

```rust
pub struct TaskTable {
    pub map: RwLock<BTreeMap<usize, Arc<Task>>>,
    pub seq: AtomicUsize,     // 自增 ID
    pub root: Mutex<Option<Arc<Task>>>,
}
```

- `spawn_root()` → 创建 PID=1 的 init 进程
- `fork_task(src)` → 深拷贝父进程的 FD 表、信号量上下文、共享内存上下文
- `clone_thread(src, sp, tls, clear_tid)` → 创建线程
- `reap(id)` → 回收僵尸进程（孤儿进程归 init 所有）
- `send_signal_group(pgid, signo)` → 向进程组广播信号

---

## 13. 内核主体 `Kernel`

```rust
pub struct Kernel {
    pub tasks: TaskTable,               // 进程表
    pub cache: BlockCache,              // 块缓存
    pub pool: FramePool,                // 物理帧池
    pub cpus: Mutex<[Option<Arc<Task>>; MAX_CPU]>, // 每 CPU 当前任务
    pub mnt: MountTable,                // 挂载表
    pub sem_store: RwLock<BTreeMap<u32, Weak<SemArr>>>, // 全局信号量
    pub shm_store: RwLock<BTreeMap<usize, Weak<Mutex<Vec<usize>>>>>, // 全局共享内存
    pub tty_buf: Mutex<VecDeque<u8>>,   // TTY 输入缓冲区
}
```

### 13.1 `dispatch_syscall` — 系统调用分发器

这是整个文件最核心的方法（约 1000 行），处理所有系统调用：

```
dispatch_syscall(nr, a0..a5) → Result<usize, &'static str>
```

| 系统调用 | 处理逻辑 |
|----------|---------|
| `SYS_READ` | 检查 buf 可访问性、页跨度、块缓存命中 |
| `SYS_WRITE` | 页内对齐处理、块缓存脏标记 |
| `SYS_OPEN` | 路径解析、flags 解析、创建文件、分配 FD |
| `SYS_CLOSE` | 块缓存清理、引用释放 |
| `SYS_MMAP` | 保护检查、空闲地址寻找、帧分配 |
| `SYS_MUNMAP` | 逐页解除映射 |
| `SYS_BRK` | 程序段边界扩充/收缩 |
| `SYS_IOCTL` | termios / TIOC / FION 命令 |
| `SYS_PIPE` | 创建管道对、分配两个 FD |
| `SYS_DUP/DUP2` | 文件描述符复制 |
| `SYS_FORK` | 内存压力检查、新 PID 分配 |
| `SYS_EXEC` | ELF 验证、路径检查 |
| `SYS_EXIT` | 进程退出、孤儿进程收养（归 init） |
| `SYS_WAIT4` | 等待子进程（支持 -1/0/>0/<-pgid） |
| `SYS_KILL` | 信号发送（支持 0/-1/>0/<-pgid） |
| `SYS_FCNTL` | F_DUPFD / F_GETFD / F_SETFD / F_GETFL / F_SETFL / F_GETLK / F_SETLK |
| `SYS_EPOLL_*` | epoll_create/ctl/wait |
| `SYS_FUTEX` | wait/wake/requeue/cmp_requeue |
| `SYS_SIGACTION` | 信号处理设置 |
| `SYS_SIGPROCMASK` | BLOCK/UNBLOCK/SETMASK |

### 13.2 `schedule_tick` — 调度周期性处理

- 每个 tick 更新当前任务的时间片
- 判断是否需要抢占（时间片耗尽时寻找下一个可运行任务）
- 统计内核态时间消耗

### 13.3 其他高级方法

- `balance_load()` — 多 CPU 负载均衡
- `reclaim_zombies()` — 回收僵尸进程资源
- `do_fork(parent_id)` — 执行完整的 fork 流程
- `do_exec(task_id, path, args, envs)` — 完整 exec 流程
- `handle_pgfault(addr)` — 缺页异常处理

---

## 14. 地址空间与 COW

### `AddrSpace` — 完整地址空间

```rust
pub struct AddrSpace {
    pub vm_map: VmMap,
    pub page_table_root: usize,
    pub asid: u16,                       // 地址空间 ID
    pub ref_count: AtomicUsize,
    pub cow_pages: Mutex<BTreeMap<usize, PgFrame>>, // COW 页面表
}
```

### COW Fork 流程

```
fork_from(parent, new_asid):
  1. 复制 vm_map（brk、mmap_base）
  2. 遍历父进程的所有 VmRegion：
     - 含 VM_WRITE 标志的页 → 标记为 COW（ref_count++）
     - 其他页 → 直接复制
  3. 拷贝 COW 页面表并增加引用计数
```

### COW 缺页处理

```
handle_cow_fault(addr, pool):
  1. 查找 VmRegion（必须含 VM_WRITE）
  2. 检查 COW 引用计数：
     - count <= 1 → 无需复制，直接返回
     - count > 1  → 分配新帧，降低旧帧引用
  3. 更新 COW 页面表
```

---

## 15. 进程组与资源限制

### `ProcessGroup`

```rust
pub struct ProcessGroup {
    pub pgid: Pgid,
    pub leader: usize,                // 组长 PID
    pub members: Mutex<Vec<usize>>,   // 成员列表
    pub session_id: usize,            // 会话 ID
    pub foreground: AtomicBool,       // 是否为前台进程组
}
```

- 支持动态添加/移除成员
- `broadcast_signal(signo, tasks)` — 向全组广播信号
- `is_leader(pid)` — 判断是否为组长

### `WaitQueue` — 通用等待队列

```rust
pub struct WaitQueue {
    pub inner: Mutex<VecDeque<(usize, thread::Thread, u32)>>,
    pub wake_count: AtomicUsize,
}
```

- `sleep(key, flags)` — 阻塞等待
- `wake_all(key)` — 唤醒所有匹配 key 的等待者
- `wake_filtered(pred)` — 条件唤醒
- `reorder_by_priority()` — 按优先级重排

### `ResourceLimits` — rlimit

```rust
pub struct ResourceLimits {
    pub max_fds: usize,           // RLIMIT_NOFILE
    pub max_threads: usize,       // 最大线程数
    pub max_stack_size: usize,    // RLIMIT_STACK
    pub max_data_size: usize,     // RLIMIT_DATA
    pub max_file_size: usize,     // RLIMIT_FSIZE
    pub max_mappings: usize,      // RLIMIT_MEMLOCK
    pub cpu_time_limit: usize,    // RLIMIT_CPU
}
```

---

## 16. 伙伴分配器 `BuddyAllocator`

```rust
pub struct BuddyAllocator {
    pub free_lists: Vec<Vec<usize>>,  // 按 order 分组的空闲块链表
    pub max_order: usize,
    pub base_addr: usize,
    pub total_pages: usize,
    pub allocated: AtomicUsize,
}
```

### 分配流程 `alloc_order(order)`

```
1. 从目标 order 向上搜索:
   - 若当前 order 有空闲块 → 取出
   - 若块过大 → 向低 order 分裂，buddy 归还
2. 更新 allocated 计数
```

### 释放流程 `free_order(addr, order)`

```
1. 从当前 order 向上尝试合并:
   - 计算 buddy_addr = addr ^ block_size
   - 若 buddy 空闲 → 合并，order++
   - 否则 → 停止合并
2. 将块插入对应 order 的空闲链表
3. 更新 allocated 计数
```

### 碎片化评估

```
fragmentation_score():
  free_ratio = (total_free - largest_block) * 100 / total_free
```

---

## 17. 工具函数

### 位操作

| 函数 | 功能 |
|------|------|
| `popcount64(v)` | 64位种群计数（Hamming Weight） |
| `clz64(v)` | 前导零计数（Count Leading Zeros） |
| `ffs64(v)` | 查找第一个置位（Find First Set） |
| `bitwise_merge(a, b, mask)` | 按掩码合并 `(a & ~mask) | (b & mask)` |
| `rotate_bits(value, amount, width)` | 定宽循环移位 |

### 对齐工具

```rust
align_up(addr, align)   — 向上对齐
align_down(addr, align) — 向下对齐
is_power_of_two(v)      — 判断是否为 2 的幂
log2_floor(v)           — 计算 floor(log2(v))
```

### 哈希函数

```rust
hash_combine(seed, value)       — boost::hash_combine 等效
murmurhash3_finalize(h)         — MurmurHash3 混合最终化
```

### 编码/校验

```rust
compute_crc32(data)                  — CRC32 校验
encode_varint(v, out)                — varint 编码
decode_varint(data) → (value, len)  — varint 解码
```

### 内存扫描

```rust
mem_scan_pattern(data, pattern, max_matches) → Vec<usize>
```

使用 **KMP 算法**（预处理失败函数）实现高效模式匹配。

### 其他

```rust
validate_access(mode, addr, len, pid) — 地址空间访问验证（三种模式）
compute_load_balance(counts, prios, blocked) — 多核负载均衡决策
audit_fd_table(files) → Vec<usize> — FD 泄露审计
rehash_mount_cache(entries) → BTreeMap — 挂载缓存哈希重建
defragment_frame_pool(slots) → usize — 帧池碎片整理
```

---

## 18. 数据流全景图

```
用户态 (User Mode)
    │
    │ syscall(number, args...)
    ▼
┌──────────────────────────────────────────┐
│          Kernel::dispatch_syscall         │
│                                          │
│  ┌─ SYS_READ  ──► FLike::read()         │
│  ├─ SYS_WRITE ──► FLike::write()         │
│  ├─ SYS_OPEN  ──► FHandle::new() + add_file │
│  ├─ SYS_CLOSE ──► files.remove() + cache.invalidate │
│  ├─ SYS_FORK  ──► TaskTable::fork_task() │
│  ├─ SYS_EXEC  ──► validate_elf + ProcInit │
│  ├─ SYS_EXIT  ──► Task::exit_proc()      │
│  ├─ SYS_MMAP  ──► VmMap::find_free()     │
│  ├─ SYS_PIPE  ──► PipeNode::pair()       │
│  └─ SYS_FUTEX ──► FutexBucket::wait/wake │
└──────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌─────────────┐      ┌──────────────┐
│  TaskTable  │◄────►│  FramePool   │
│  (进程管理)  │      │  (物理内存)   │
└─────────────┘      └──────────────┘
         │                    │
         ▼                    ▼
┌─────────────┐      ┌──────────────┐
│  VmMap /    │      │  PageCache / │
│  AddrSpace  │      │  BlockCache  │
│  (虚拟内存)  │      │  (缓存层)     │
└─────────────┘      └──────────────┘
```

### 信号处理流程

```
Task::send_sig(signo, sender)
  → sig_queue.push((signo, sender_tid))
  → ev.set(RECV_SIG)
  → Task::has_sig() 检测
  → TaskTable::send_signal_group(pgid, signo)
```

### Fork 流程

```
Kernel::do_fork(parent_id)
  → TaskTable::fork_task(parent)
    → 深拷贝 cwd, exec_path
    → 遍历 files 复制每个 FD
    → 复制 sem_ctx, shm_ctx, sig_mask
    → link_parent / link_child
  → child.vm_token = parent.vm_token  // COW 共享地址空间
```

### Exec 流程

```
Kernel::do_exec(task_id, path, args, envs)
  → 验证 ELF header
  → 关闭带 O_CLOEXEC 的 FD
  → ProcInit::push_at() 布局栈
  → 设置新 IP = 0x0040_0000
```

---

## 附录：关键设计决策

1. **单一大锁 (KernLock)**：模拟传统 Unix 的 BKL，内核全局使用互斥锁而非细粒度锁。
2. **COW 页面管理**：fork 时通过 `ref_count` 共享物理页，写时触发 `SharedPage::fault()` 分配新帧。
3. **模拟硬件**：`Disk::errs` 支持 IO 错误注入，`BlockCache::fetch` 支持访问延迟模拟。
4. **伪 ELF**：ELF 验证使用硬编码的伪二进制数据，仅验证头部结构。
5. **调度器**：混合 CFS（公平调度）和优先级调度，vruntime 驱动任务选择。
6. **全局对象表**：sem_store/shm_store 使用 `Weak` 引用，允许惰性回收。
