---
date: 2023-03-05T09:28:11+08:00
title: "Golang 运行时探究"
description: "从源码入手，了解 golang 运行时工作流程"
tags: ["golang"]
series: []
---

> 版本与原型环境：Golang: 1.19.6; WSL2 Ubuntu-20.04

本文尽量不放过多源码，只注明源码位置及大致逻辑，开发者还需要自己阅读源码才能对 runtime 有更深刻的体会。runtime 代码都在 [src/runtime](https://github.com/golang/go/tree/master/src/runtime) 目录下。

## 启动阶段

Debug 运行一段最简单的代码，观察一下程序刚启动后的调用堆栈：
```go
func main() {
	fmt.Println("hello world")  // 程序停在这
}
```

观察程序的 Call Stack，可以发现启动了五个协程，根据调用栈可以找到这五个协程对应的入口:
- `runtime.main`，它调用了我们定义的 main 函数，是用户程序的入口
- `runtime.forcegchelper`，gc 辅助协程，它在 proc.go 的 `runtime.init` 中被调用
- `runtime.bgsweep`，gc 标记，在 `runtime.gcenable` 中被启动
- `runtime.bgscavenge`，gc 清除，也在 `runtime.gcenable` 中被启动；`runtime.gcenable` 在 `runtime.main`中被调用
- `runtime.runfinq`，它是运行所有 `finalizers` 的协程，它只被启动一次，会在第一次调用 `runtime.SetFinalizer` 时被启动。用户自定义的所有 Finalizer 都会在这个写成立串行执行，所以要在 Finalizer 里执行耗时操作，最好启动新的协程。

以上只是在 go 的层面观察到的启动顺序。Go 程序启动的最初入口在汇编代码层面，主要是在 `runtime.rt0_go` 汇编函数中[^1]：
```s
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
	// ...
nocgo:
	// 创建 m0 和 g0 并相互引用
	// save m->g0 = g0
	MOVD	g, m_g0(R0)
	// save m0 to g0->m
	MOVD	R0, g_m(g)

	// 初始化：执行文件路径，内存页大小
	BL	runtime·args(SB)   // runtime1.go/args
    // CPU 个数，大页内存页大小
	BL	runtime·osinit(SB)  // os_linux.go/osinit
    // 命令行参数、环境变量、gc、栈空间、内存管理、所有P实例、HASH算法等
	BL	runtime·schedinit(SB)  // proc.go/schedinit

	// create a new goroutine to start program
    // 创建一个新的 goroutine，绑定 runtime.main，放在 P 本地队列，等待调度
	MOVD	$runtime·mainPC(SB), R0		// entry
	SUB	$16, RSP
	MOVD	R0, 8(RSP) // arg
	MOVD	$0, 0(RSP) // dummy LR
	BL	runtime·newproc(SB)
	ADD	$16, RSP

	// 启动 M，开始调度 goroutine
	BL	runtime·mstart(SB)
```

初始化过程做了很多事情，包括对于 android、ios 以及其他窗口应用的一些前置逻辑。用 `schedinit` 函数的注释说，启动顺序是：
- call osinit
- call schedinit
- make & queue new G, The new G calls runtime·main.
- call runtime.mstart


我们主要关注 `schedinit` 和 `runtime.main`。

`schedinit` 的几个主要的初始化动作：
- 初始化系统全局锁的 lock rank：系统层面的锁有很多，有些操作需要获取多个锁，对锁按照 rank 排序，防止死锁。
- 设置 `sched.maxmcount = 10000`，即 `G-M-P` 模型中 `M` 的最大数量。
- `mallocinit`，包含若干内存管理过程
  - `mheap_.init()` 用于初始化 `span` 分配器、`cache` 分配器及其他特殊分配器，以及初始化 136 个对应 `spanClass` 的中央缓存 `mheap.central[136]`，最后是向操作系统申请内存；这些内存分配器都是 `fixalloc` 的实例，`fixalloc` 用于固定大小对象的空闲列表分配器。
  - 使用 `cache` 分配器初始化 `mcache0`，用于初始化时候分配内存，后续会绑定到第一个 `P`
  - 初始化 `mheap_.arenaHints`，用于设置申请更多 `heap areanas` 后的地址（go runtime 定义的内存地址）
- `cpuinit` 读取 `GODEBUG` 环境变量并初始化 cpu 信息
- `alginit` 随机初始化 `hash` 算法种子
- `fastrandinit` 初始化随机算法种子
- `modulesinit` 从所有加载的 `module` 中创建 `activeModules` 切片
- `typelinksinit` 加载 `activeModules` 定义的类型
- `goargs` 初始化命令行 `args`
- `goenv` 初始化 `env`
- `gcinit` 初始化 `gc work` 状态，以及对应锁的 `rank`
- `procresize` 设置 `P` 的数量，创建 `P` 并初始化 `P`（如果 `P` 数量变少，则销毁对应数量的 `P`），第一个创建的 `P` 的 `P.mcache = mcache0`，后续的 `P.mcache = allocmcache()`，即从 `mheap` 中申请新的 `mcache`；给当前 `g.m` 绑定一个 P；`mcache0 = nil` 全局 `mcache0` 不再持有该内存指针；`p.m.set(mget())` 尝试给 `P` 绑定一个 `M`；
- `worldStarted` `P` 可以运行了

`runtime.main` 的主要流程：
- 设置最大栈空间大小：64 位系统 1GB，32 位系统 250M，
- 创建 `sysmon` 线程，辅助垃圾清扫，协程调度
- `doInit(runtime_inittask)` 调用 `runtime init` 函数
- `gcenable` 启动 gc 协程 `bgsweep` `bgscavenge`
- `doInit(main_inittask)` 调用用户程序 `init` 函数
- `fn := main_main; fn()` 调用 `main` 函数
- `main` 函数返回后，函数退出

总的来说，初始化阶段可以分为两部分，第一部分是系统环境初始化：系统锁顺序、内存分配器、堆内存、最初的 `mcache0`、`hash` 与 随机算法、环境参数、`process` 等，第二部分是运行环境初始化：运行环境所需的 `sysmon` 线程（不绑定 `P`，不在 `G-M-P` 模型内调度）、垃圾标记与清扫协程、`runtime` 与用户程序的 `init`，以及最后启动 `main` 函数。

## 运行阶段

### 内存模型

Go 使用 `mheap` 管理所有堆内存。向操作系统申请的最小内存单元是 `page`，go 程序每次向操作系统申请若干 `page`，并使用 `heapArena` 管理 `mheap` 中 `page` 的状态。从 `mheap` 中申请内存的最小单位是 `mspan`，它由若干 `page` 组成。

`mspan` 根据存储的单个 `object` 的内存大小划分为不同的 `spanClass`，比如有的用于存储小对象，有的用于储存较大的对象；这种划分划分方式既能提高可用内存的查找效率，又能提高空间利用率，避免过多空间碎片。

每个 `mspan` 都在一个双向链表中，可能是 `mheap` 的 `busy list`，或者是某个 `mcentral` 的 `span list`。

一些关键对象字段：
```go
type mheap struct {
	pages pageAlloc // page allocation data structure
	allspans []*mspan // all spans out there
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	// 中心缓存
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}
	// ...
}
type mcentral struct {
	spanclass spanClass
	partial [2]spanSet // list of spans with a free object
	full    [2]spanSet // list of spans with no free objects
	// ...
}

var mheap_ mheap

type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.
	startAddr uintptr // address of first byte of span aka s.base()
	npages    uintptr // number of pages in span
	freeindex uintptr
	nelems uintptr // number of object in the span.
	// ...
}
``` 


### 栈空间管理

每个 `G` 有自己的栈空间。调用 `proc.go/newproc()` 创建新的协程，程序会尝试从本地 `P.gFree` 和全局 `sched.gFree` 获取一个 `dead G`（被回收的 `G`），则直接复用它的栈。如果没有 `dead G`，则调用 `malg(stackMin)` 创建一个拥有最小栈空间 2048 的 `G`。

编译器会为函数调用插入 `runtime.morestack` 检查，如果栈空间不足，需要扩容，则会调用 `runtime.newstack` 扩容。`newsize := oldsize * 2`，每次容量增加一倍；程序会检查 `newsize` 是否符合新栈空间要求，如果不够，继续增大 `newsize`。调用 `copystack(gp *g, newsize uintptr)` 申请新的栈空间，并复制栈内容，把新栈指针设置到 `g`，`stackfree(old)` 回收旧栈。

`stack.go/stackalloc(n uint32)` 用于申请新的栈内存，主要逻辑为：
- 如果 GODEBUG 设置 efence != 0，或 stackFromSystem !=0，则直接从系统内存中申请一段内存
- 如果 n 小于 `_StackCacheSize = 32768`，且小于 `_FixedStack<<_NumStackOrders`（随系统变化），则尝试从本地或去全局 `stack` 缓存获取：
  - 如果当前 g.m 没有 p，或 p 不使用 `stackcache`，则调用 `stackpoolalloc` 从全局 `stackpool` 中获取，`stackpool` 没有可用缓存时，调用 `mheap._allocManual` 从 `mheap` 中申请。
  - 否则，从当前 P 的 `mcache.stackcache` 申请，如果 `mcache.stackcache` 为空，则调用 `stackpoolalloc` 从全局 `stackpool` 获取，直到 `mcache.stackcache` 填充一半（`size >= _StackCacheSize/2`）
- 先尝试从全局 `stackLarge` 缓存中获取，如果没有，则直接从 mheap 中调用 `mheap_.allocManual` 申请包含若干 `page` 的 `span`。

在 gc 扫描栈的过程中，可能发生栈缩容，栈缩容与扩容类似，计算目标栈空间大小，调用 `copystack(gp *g, newsize uintptr)` 申请新的栈空间，并复制栈内容，把新栈指针设置到 g，`stackfree(old)` 回收旧栈。

`stackfree` 的逻辑与 `stackalloc` 类似：
- 若 n 小于 `_StackCacheSize = 32768`，且小于 `_FixedStack<<_NumStackOrders`，则尝试回收到全局 `stackpool` 或者 `p.mcache.stackcache`，回收过程也会根据各级缓存是否已满，则回收到上级缓存，直到最后回收到 `mheap`
  - 只有在 `_GCoff` 阶段才能向 `mheap` 直接回收栈空间
- 若 n 比较大，则根据 GC 阶段判断回收到哪里：
  - `gcphase == _GCoff`：GC 没有运行，后台清扫中，没有开启写屏障，则直接回收到 `mheap`
  - GC 运行时，为了避免 `span` 回收到 `mheap` 后，又被申请用作堆 `span`，可能与 gc 发生冲突，所以直接回收到 `stackLarge` 缓存。

总结一下，较小的栈存在两级缓存：本地 `P` 缓存 `P.mcache.stackcache`、全局 `stackpool`；大的栈空间存在一级缓存：`stackLarge`；申请缓存时根据条件分别尝试从各级缓存中申请，从全局或 mheap 申请/释放时否需要加锁，所以会优先尝试本地 `P.mcache.stackcache`。释放缓存时，避免在 GC 时，栈 `span` 回收到 `mheap` 后又被申请为堆 `span`，导致状态冲突，所以 GC 时不能向 `mheap` 回收栈内存，最多只回收到全局栈缓存。

此外，各级缓存都会根据栈空间大小（扩容次数），或栈的 `page` 数量（large stack），划分为不同的链表（例如类型为：`stackcache[size or page count] stackfreelist`）进行管理，方便快速找到对应大小的占空间。

## 堆内存申请

以下情况，对象的内存会被分配到堆上[^3]：
- 内存原因：`interface{}` 动态类型，编译期无法知道内存大小；栈空间不足，比如创建一个超过系统栈空间大小的数组
- 作用域原因：闭包或协程访问外部作用域；函数内创建的指针返回到外部；指针被堆上的对象引用

堆内存申请的入口在 `malloc.go/mallocgc(size uintptr, typ *_type, needzero bool)`，根据此函数注释可以了解到：
- 小对象从本地 `P` 的缓存中申请
- 大对象（> 32kB）直接从 `mheap` 中申请

内存申请的主要逻辑为：
- 如果申请内存大小 `size` 为 0，则直接返回一个固定的 `zeroAddr`
- 获取当前线程 m 的乐观锁，`m.mallocing = 1`
- 如果类型 typ 内没有指针，且小于 `maxTinySize` 16 字节，则尝试从当前 `P.mcache` 的 `tinyAllocs` 申请内存（`tinyAllocs` 内存申请只需要计算在 tiny 缓存区的偏移量，性能很高）；没有 tiny 缓存，则依次尝试向 `mcache.alloc[tinySpanclass]` 及其上级缓存申请
- 如果 `size <= maxSmallSize` 32kB，则计算 size 对应的 `spanClass`，并一次尝试向 `mcache.alloc[tinySpanclass]` 及其上级缓存申请，具体逻辑为：
  - 先调用 `span = c.alloc[spanClass]; v = nextFreeFast(span)` 从 span 双向链表的当前 span 节点中，使用 `allocCache` 标志位快速查找是否存在未使用的缓存，如果有，则直接用此缓存
  - 否则，调用本地缓存 `mcache.nextFree(spanClass)`，此方法先尝试从 `span.freeIndex` 之后查找可用缓存，如果没有，说明 span 已满；则调用 `mcache.refill(spanClass)` 从全局缓存 `mheap.central[spanClass]` 中申请一块有剩余缓存的 span，并调用新 `span.nextFreeIndex` 得到内存指针。如果需要从全局缓存获取，则 `shouldhelpgc` 设置为 true，后续会触发 GC。`central[spanClass]` 用尽时，会调用 `mheap_.alloc(npages, spanclass)` 从 mheap 中申请一个有 `npages pages` 的 `span`（span 根据 spanClass 有不同的大小，多个 pages 组成一个 span）。
- 如果 `size > maxSmallSize` 32kB，则通过 `mcache.allocLarge` 方法，调用 `mheap_.alloc(npages, spanclass)` 直接向 `mheap` 中申请若干 `pages`，组成 `span` 并返回。`shouldhelpgc` 设置为 `true`，后续会触发 GC。

总结一下，申请无指针的小对象时有 `tinyAlloc` 优化措施，中等对象优先尝试从当前 `P.mcache` 中申请，没有则依次向全局缓存 `mheap.central mheap` 中申请，大对象直接从 `mheap` 中申请。申请过程中，如果 `P.mcache` 用尽或申请了大对象，都会触发 GC。内存的释放在 GC 过程。


## 垃圾回收

GC 开始的入口在 `mgc.go/gcStart(gcTrigger)`，查看此函数的所有调用可以了解到 GC 触发的场景有:
- 堆内存申请时 `P.mcache` 用尽，或申请了大于 32kB 的对象
- 用户程序调用 `runtime.GC()`
- `forcegchelper` 协程定时触发 GC

Go 的三色标记法的大致流程：短暂 STW，为每个 `P` 上创建一个 `G`，用于垃圾回收的标记阶段；标记阶段完成后，触发清扫阶段，清扫阶段需要 STW。标记阶段采用三色标记法：**先把所有的根对象标记为灰色并放到任务队列，然后持续从任务队列取出灰色对象，并标记为黑色，把它的指针指向的对象标记为灰色，入工作队列，持续此过程直到工作队列为空**；为了防止标记协程与用户协程出现并发问题，使用混合写屏障[^4]将被覆盖的对象标记成灰色，并在当前栈没有扫描时将新对象也标记成灰色；在 `mallocgc` 函数里，把新创建的对象标记为黑色。

垃圾回收入口 `gcStart` 的主要流程：
- 启动与 `P` 数量相同的协程用于并发标记，每个 `P` 上跑一个标记协程 `gcBgMarkWorker`
- `stopTheWorldWithSema` 暂停世界
- 处理一些 GC 前置的工作，例如 GC、CPU 限制，GC 状态修改，初始化堆标记 work（标记的工作队列），标记 `TinyObjects` 等
- `startTheWorldWithSema` 继续调度协程，`gcBgMarkWorker` 被调度后会进行标记动作

标记协程 `gcBgMarkWorker` 流程：
- 通过 `gcController.findRunnableGCWorker` 被唤醒调度
- 调用 `gcDrain`，先扫所有根对象 `nDataRoots, nBSSRoots, nSpanRoots, nStackRoots`，然后循环调用 `scanobject` 标记工作队列里的对象；工作队列分为本地工作队列 `P.gcw` 和全局工作队列，标记过程中会对本地和全局工作队列进行平衡，比如本地队列满了，则会拿出一部分放到全局队列
- 检测到标记结束后，会调用 `gcMarkDone`，暂停世界，并调用 `gcMarkTermination -> gcSweep -> sweepone` 进行清扫，最后通过调用 `mheap.freeSpan(span)` 把空 `span` 放回堆里；`sweep` 结束后，唤醒 `scavenger`，把空余内存和不使用的页还给操作系统
- 启动世界，继续调度

并发标记能够解决垃圾回收时暂停世界时间过长的问题，Go 的垃圾回收主要在清扫阶段暂停世界。标记任务通过本地队列、全局队列进行两级存储，本地队列无锁性能高，全局队列可以平衡不同协程之间的工作量。


## 协程创建与调度

`go` 关键字会在编译时转换为对 `proc.go/newproc(fn)` 的调用，它会调用 `newproc1(fn, currentG, currentCallerPc)` 创建协程，并通过 `runqput(currentP, newg, true)` 加到运行队列：
- `runqput` 第三个参数为 true 表示会把 `newg` 加到本地 `P` 的 `next slot`，也即本地 `P` 下次调度的 `G` 就是刚创建的 `newg`（如果过程中创建了新的 `G`，则又会被新的 `G` 抢占 `next slot`）
- `runqput` 先尝试把 `G` 加到容量为 256 的本地队列（无锁），如果本地队列已满，则把 `newg` 和本地队列的一半 `G` 放到全局队列

`newproc1` 创建协程的主要流程：
- 调用 `gfget` 尝试从 `P` 本地 `freelist` 获取一个被回收的 `G`，本地如果没有，则从全局 `freelist` 获取 32 个 `G` 到本地；得到 `G` 后会判断是否有栈，没有的话会调用 `stackalloc` 申请栈空间
- 如果 `freelist` 没有可用 `G`，则调用 `malg` 创建新的 `G`，并申请栈空间
- 设置栈指针 `SP`
- 设置 `sched.pc` 为 `goexit`，这样每个协程最后退出时都会执行 `goexit`
- 设置 `startpc` 为用户协程函数 `fn`
- 修改 `G` 的状态为 `_Grunnable`

触发调度的场景有很多，调度的最终入口都在 `proc.go/schedule()`。触发调度的情况有：
- **主动挂起**：`gopark` -> `park_m`，状态修改为 `_Gwaiting`，通常发生在 `channel` 阻塞，等待 `time`、`netpoll` 等；`pack_m` 会把 `G-M` 解绑，并等待调用 `goready` 后，修改为 `_Grunnable` 并调用 `runqput` 加入到运行队列
- **系统调用**：`cgocall` -> `entersyscall/exitsyscall`，系统调用开始与退出。`entersyscall` 进入系统调用，保存 `SP/PC`，`M-P` 解绑并记录 `P` 为 `m.oldp`，状态修改为 `_Gsyscall`，P 的状态也为 `_Psyscall`；`exitsyscall` 已退出系统调用，状态修改为 `_Grunnable`，并把 `G` 加入到 `oldp` 运行队列 
- **协作式调度**：`GC` 阶段，信号触发，以及用户程序都可以调用 `Gosched`
- **系统监控**：`sysmon` -> `retake`，如果当前 `G` 已运行（或 `P` 陷入系统调用）超过 10ms，则调用 `preemptone` 通过信号 `sigPreempt` 发送抢占请求

`schedule` 先调用 `findRunable` 找到一个等待运行的 `G`，然后调用 `execute` 执行它。

`findRunable` 查找一个可运行 `G` 的顺序：
- 以 1/61 的概率从全局待运行队列里查找一个
- 从 `P` 的本地 `runq` 中查找
- 本地没有，则从全局 `runq` 中查找，并拿一部分 `G` 到本地
- 调用 `netpoll` 从网络轮询器中查找是否有就绪的 `G`
- 从其他 `P` 中偷取
- 会再次尝试全局队列及网络轮询器
- 重复上述过程，直到找到一个就绪的 `G`

`execute` 会把当前 `G-M` 解绑，把要运行的 `G` 与 `M` 绑定，最后调用 `gogo(&g.sched)` 恢复 `G` 的执行上下文，并继续执行。

此流程里并没有把当前的 `G` 重新入待运行队列的操作，因为 P 的本地队列是一个 `[256]uintptr` 数组，以及两个指针 `head tail`，当前 `G` 本身就处于 `head` 和 `tail` 之间。`P` 中关于 `G` 的一些字段如下：

```go
type p struct {
	//...
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	runnext guintptr
	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}
}
```

Go 协程退出时会调用汇编程序里的 `goexit`，最终调用 `goexit0(g)` 来结束协程。它把 `G` 的状态设置为 `_Gdead`，并回收 `G` 的资源，最后触发调度。回收资源包括：
- 把栈加入到 `GC` 扫描栈，用于释放栈引用的堆内存；
- 解绑 `M`；
- 如果栈空间大于初始的 2kB，回收整个栈；
- 把 `G` 加入到本地 `P.gFree`，如果 `P.gFree.number > 64`，则把一半的 `dead G` 加入到全局 `gFree`


## 总结

Go 运行时初始化阶段进行了运行环境读取（操作系统内核数、页大小、环境变量、参数等）及一些初始化工作（各种内存分配器初始化、随机/hash 算法初始化，`P` 及应用程序的 init 函数）。

Go 的垃圾回收经过多个版本的迭代，已经非常成熟，通过在各个 P 上并发标记，大大减少了 STW 的时间。

运行时的设计充分体现了多级缓存的思想：堆、栈内存的分配设立了本地/全局缓存，堆对象的内存申请也很多对象大小进行分级，使用 `spanClass` 把对象进行分类，不仅加快分配速度，还减少了空间碎片；协程调度、协程创建与回收，也使用本地/全局缓存，降低锁争用的发生。这种多级缓存、按类分配的思想能给我们在做系统设计时提供启发。



[^1]: 探索 golang 启动过程 https://cbsheng.github.io/posts/%E6%8E%A2%E7%B4%A2golang%E7%A8%8B%E5%BA%8F%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/

[^2] draveness Go 语言设计与实现，栈扩容 https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/#%E6%A0%88%E6%89%A9%E5%AE%B9

[^3] 极客兔兔 Go 语言高性能编程，内存逃逸 https://geektutu.com/post/hpg-escape-analysis.html

[^4] draveness Go 语言设计与实现，混合写屏障 https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#%E6%B7%B7%E5%90%88%E5%86%99%E5%B1%8F%E9%9A%9C