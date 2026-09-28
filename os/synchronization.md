# Synchronization

面试准备笔记：**English terminology + 中文解释**。重点是解释“为什么正确”，不只记住 API 名称。

前置知识：[Process & Thread](process-thread.md)、[Memory Management](memory.md)。本篇不重复 address translation、thread scheduling 和 cache hierarchy；涉及这些背景时引用对应笔记。

**Scope:** 主要讨论同机 shared-memory concurrency。Inter-process、async、database/distributed coordination 的边界在 Section 14 单独说明。

**Code:** 标为 Runnable Python 的例子可保存成独立 `.py` 文件运行，使用标准库。C++ fragments 用于说明 native memory-model 语义，不是完整 application；标为 incorrect 的 pseudocode 不要当成实现。Python examples 用于验证协议，不用于证明 CPU parallel speedup。

## 1. What Is Synchronization?

### 1.1 The Problem It Solves

**Synchronization** 为多个 execution flows 建立访问 shared state 和协调进度的规则，使结果满足约定的 correctness requirements。

不只是“防止两个 threads 同时写”：

- 两个 writers 可能需要 mutual exclusion。
- Reader 可能需要等 writer 发布完整 object。
- Producer 可能需要等 queue 有空位。
- 主线程可能需要等 workers 完成某个阶段。
- Shutdown 需要让所有 waiters 得知“不再有新工作”。

这些需求不同，因此没有一种 primitive 能自动解决全部问题。

### 1.2 Safety vs Liveness

| Property | 要回答的问题 | Example |
| --- | --- | --- |
| **Safety** | 不该发生的事情是否永远不会发生？ | Balance 不为负，item 不被重复消费 |
| **Liveness** | 在适当条件下，应该发生的事情是否最终能发生？ | 有 item 后 consumer 能继续 |
| **Fairness** | 机会如何在参与者之间分配？ | 某个 waiter 不被无限插队 |
| **Performance** | 做到这些需要多少时间和资源？ | Throughput、tail latency、CPU cost |

一个把所有 threads 永久 block 的设计，可能不会破坏 balance，却没有 liveness。一个结果极快但偶尔丢数据的设计，也不满足 correctness。

Liveness 通常依赖条件，例如 scheduler 给 runnable threads 执行机会、lock owner 最终完成、external operation 会返回。证明时应说明 assumptions。

### 1.3 Race Condition vs Data Race

基础 lost-update interleaving 见 [Process & Thread, Section 8.2](process-thread.md#82-race-condition-vs-data-race)。这里区分两个层级：

- **Race condition:** 结果依赖未妥善控制的执行顺序，并违反业务要求。即使所有单个 memory operations 都 atomic，仍可能存在。
- **Data race:** 在 C++ 等具体 memory model 中，未通过相应 ordering 协调的 conflicting accesses，至少一个非 atomic。普通读写场景中，冲突意味着至少一个写；object lifetime 的开始/结束也可能冲突。

C++ data race 导致 undefined behavior，不只是“最后结果随机”。单核也能通过 interleaving 暴露错误，不能靠“现在没有真正同时执行”证明安全。[C++ draft: data races](https://eel.is/c++draft/intro.races)

### 1.4 A New Example: Check Then Act

假设 stock = 1，两个 callers 想购买：

```text
A: checks stock > 0 → true
B: checks stock > 0 → true
A: deducts one, confirms sale
B: deducts one, confirms sale
```

问题不是某一个 subtraction 一定被撕裂，而是“检查 + 扣减 + 决定成功”没有被当成一个受保护的 operation。

需要保证的 invariant 是：

```text
Stock never becomes negative.
Each successful reservation consumes exactly one available unit.
```

先定义 invariant，才能决定 lock 的范围或 atomic protocol。

## 2. Critical Sections and Correctness Contracts

### 2.1 What Is a Critical Section?

**Critical section** 是必须按某种协调规则访问 shared state 的 code region。它的边界应由 invariant 决定，而不是“所有写入前后各随便加一个 lock”。

转账的业务要求可以是：

```text
Before: A.balance + B.balance = total
After:  A.balance + B.balance = total
And neither account becomes negative.
```

如果把“扣 A”和“加 B”分别置于不关联的 critical sections，中间 observer 可能看到总额暂时减少。是否允许这种观察，必须由 API contract 决定。

### 2.2 Three Classic Requirements

经典 critical-section problem 常要求：

1. **Mutual exclusion:** 不允许不兼容的 accesses 同时进入。
2. **Progress:** 没人在 critical section 时，选择下一个进入者不能无限拖延。
3. **Bounded waiting:** 某个参与者请求进入后，其他参与者不能无限次抢先进入。

这些是设计目标，不代表任意 mutex API 都保证 FIFO 或 bounded waiting。实际 fairness 和 scheduling 需要看实现。

### 2.3 Ownership Before Locks

对每一份 mutable state 问：

- 谁拥有它？
- 哪些 code paths 可以访问？
- 用哪个 lock 或 protocol 协调？
- Object 在 access 期间是否 alive？
- 谁负责关闭、取消或销毁？

```text
State: account balances
Rule: hold each account's lock when reading/writing its balance
Cross-account operation: acquire both locks in one global order
Snapshot of both balances: follow the same two-lock rule
```

“写入有 lock，但 reads 不加任何协调”通常不是完整协议。只要有不遵守规则的 access path，lock 本身不会替你拦住它。

### 2.4 Linearizability

一个 concurrent operation 若看起来像在 invocation 与 response 之间某一个瞬间完成，并保持不重叠 calls 的实际先后关系，就可以用 **linearizability** 来描述。

例如安全的 `try_reserve_one()`，每次应对应某个单一决策点：在那一点成功消耗一个 stock，或发现没有可用 stock。

**Linearization point** 可以是 mutex 内的一次决定性更新，也可以是成功 CAS。整个 operation 的实现可以包含很多 instructions。

一个 thread-safe `contains()` 加一个 thread-safe `insert()`，不自动成为一个 linearizable “insert if absent”；需要提供整个操作的 contract。

### 2.5 Thread Safety vs Reentrancy

- **Thread-safe:** 按 API contract 从多个 threads 调用时正确。
- **Reentrant:** 在前一次调用尚未结束时再次进入仍然安全，例如 recursive callback 或某些 interrupt/signal contexts。

一个 function 用普通 mutex 保护 global state，可能 thread-safe，却在同一 thread 的 callback 重入时 deadlock。Thread-safe 也不等于 async-signal-safe；signal handler 中允许调用什么要遵守平台的专门规则。

## 3. Mutexes and Locking Discipline

### 3.1 What a Mutex Provides

Mutex 提供 exclusive access；支持的 mutex memory semantics 还让前一个 owner 的 writes 能按规则被后一个 owner 观察到。

```text
Thread A: lock → update protected state → unlock
Thread B:                                  lock → observe/update
```

关键是“同一个 mutex”和“所有相关 accesses 都遵守协议”。不是 lock 名字相同、数量相同就足够。

概念上的 mutex 通常有 owner，owner 负责 unlock；具体 API 有差异，例如 Python primitive `Lock` 不跟踪 owner，而 `RLock` 会。不要把某个库的例外推广到所有 mutexes。

POSIX mutex 的 normal、recursive、error-checking、robust 等类型也有不同语义。[POSIX mutex operations](https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3p.html)

### 3.2 Exception-safe Lock Lifetime

优先使用 scope-based cleanup：

```cpp
// C++ fragment
std::mutex mutex;

void update() {
    std::lock_guard<std::mutex> guard(mutex);
    // Protected work.
}  // guard releases mutex, including ordinary exception unwinding
```

C++ `unique_lock` 支持更灵活的 ownership、unlock/relock 和 condition-variable waits；`scoped_lock` 可以管理多个 mutexes。Python 常用 `with lock:`。

这防止正常异常路径漏 unlock，但**不会自动 rollback 业务 state**。如果扣款后、入账前抛 exception，即使 locks 被释放，仍可能留下不一致数据。先准备可能失败的工作，或设计 rollback/transaction。

Process 被强制结束时，普通 stack unwinding 也不一定发生。

### 3.3 Runnable Python: Transfer with a Global Lock Order

示例假设 account IDs 唯一且不变，amount 为正整数，balance 只通过遵守规则的操作访问。

```python
from threading import Lock, Thread

class Account:
    def __init__(self, account_id, balance):
        self.account_id = account_id
        self.balance = balance
        self.lock = Lock()

def transfer(source, target, amount):
    if source is target:
        raise ValueError("Accounts must differ")
    if amount <= 0:
        raise ValueError("Amount must be positive")

    first, second = sorted(
        (source, target), key=lambda account: account.account_id
    )
    with first.lock:
        with second.lock:
            if source.balance < amount:
                return False
            source.balance -= amount
            target.balance += amount
            return True

def repeat_transfer(source, target):
    for _ in range(100):
        if not transfer(source, target, 1):
            raise RuntimeError("Unexpected insufficient balance")

left = Account(1, 1000)
right = Account(2, 1000)
workers = [
    Thread(target=repeat_transfer, args=(left, right)),
    Thread(target=repeat_transfer, args=(right, left)),
]
for worker in workers:
    worker.start()
for worker in workers:
    worker.join()

print(left.balance, right.balance)
assert (left.balance, right.balance) == (1000, 1000)
# 1000 1000
```

无论 transfer 方向，都先获取较小 ID 的 lock，避免 A→B 与 B→A 形成相反 lock order。最后 reads 发生在所有 workers join 后，不与 updates 并发。

若运行中要读取两个 balances 的一致 snapshot，也必须获取两把 locks，而不是靠某个 reader“只读所以安全”。

这个例子保证正常执行下的 in-memory operation，不是 crash-safe 或跨 database 的 financial transaction。

### 3.4 Recursive Locks

Recursive mutex / reentrant lock 允许同一 owner 再次 acquire，并记录 recursion count；通常要匹配相同次数的 release 才真正让其他 threads 进入。

它可以支持某些嵌套调用，但：

- 不解决两个 threads 的 lock-order deadlock。
- 不保证 callback 重入时 state 已满足 invariant。
- 可能掩盖不清晰的 layering。
- 普通 non-recursive lock 的重复 acquire 可能 self-deadlock。

不是“改成 recursive 就更安全”。

### 3.5 Coarse-grained vs Fine-grained Locks

| Strategy | Benefit | Cost |
| --- | --- | --- |
| **Coarse-grained** | 协议简单，容易保护跨对象 invariant | 无关操作也可能被串行化 |
| **Fine-grained** | 独立 data 可以 concurrent access | 多锁顺序、lifetime、snapshot 更复杂 |
| **Lock striping / sharding** | 以固定数量 locks 管理分区 | 不同 keys 可能落在同一 shard；跨 shard 操作要协调 |

先用能够清晰证明正确的方案，再根据 measured contention 拆分。Lock 越多不必然越快。

### 3.6 Try-lock, Timeout, and Fairness

`try_lock` 失败只说明当时未取得 lock，不说明 owner 已死或 resource 永久不可用。Timed acquisition 超时也不自动撤销 caller 之前做过的 updates。

不要写无限 tight retry loop 去模拟 blocking mutex；它可能变成高 CPU 的 busy wait。若使用 retries，应有清晰的重试、退避、取消和失败策略。

普通 mutex 通常不承诺严格 FIFO；公平锁可能降低吞吐，选择时要看 latency 与 starvation 要求。

## 4. Spinlocks, Blocking Locks, and Futexes

### 4.1 Busy Waiting vs Parking

**Spinlock** 在无法获得 lock 时继续执行检查，不把当前 execution 直接挂起。**Blocking lock** 可以把 waiter park，让 CPU 执行其他工作。

```text
Spinning: check → check → check → acquire
Blocking: wait registration → sleep → wake → compete → acquire
```

Spin 可能避免 sleep/wakeup 的成本，但等待期间持续消耗 CPU。若 owner 被抢占，waiters 再怎么 spin 也不能帮助 owner 完成。

### 4.2 When Spinning May Make Sense

通常需要满足：

- Critical section 极短。
- Owner 很可能正在其他 CPU 上执行并很快释放。
- 不执行可能长期 block 的工作。
- 有合适的 hardware/runtime support。

User-space business code 不应因为“spinlock 没有 context switch”就默认使用它。Oversubscription、单个可用 CPU、long I/O 或 GC pauses 都可能使 spin 更差。

Adaptive locks 可以先短暂 spin，再 park，因此 mutex 和 spin 并非所有实现中都截然分开。

### 4.3 How Futex-style Waiting Helps

Linux futex 是构建高层 synchronization 的底层机制，常见思路：

1. 无竞争时，在 user space 通过 atomic operation 更新 lock state。
2. 需要等待时，请 kernel 在 shared word 仍符合某个值的条件下挂起。
3. Unlock path 在需要时唤醒 waiter。

Kernel 的 compare-and-wait 处理有助于避免“检查后、睡下前已经发生更新”导致的漏唤醒。醒来仍要重查状态；futex wake 不是把业务 lock ownership 直接交给 waiter。[Linux futex](https://man7.org/linux/man-pages/man7/futex.7.html)

**Lock acquire 不一定 syscall；futex 也不是可以直接替代任意 application protocol 的万能 mutex。** 一般优先使用成熟 library abstractions。


## 5. Semaphores

### 5.1 Counting Permits

**Semaphore** 维护可用 permits 的数量。成功 acquire 消耗一个 permit，release 增加一个；没有 permits 时，acquire 可以等待或按 API 返回失败。

例如只有 3 个可同时使用的 connections：

```text
Initial permits = 3
Worker A acquires → 2
Worker B acquires → 1
Worker C acquires → 0
Worker D waits
A releases        → D may proceed
```

Semaphore 限制同时使用 resource 的人数，不自动保护进入者之间的所有 shared writes。3 个 admitted workers 仍可能需要 mutex 保护共同 metadata。

### 5.2 Binary Semaphore vs Mutex

Binary semaphore 的可用数量限定为 0/1，表面上类似 exclusive gate。但 semaphore 通常没有 owner 语义，可以由一个执行者 acquire、另一个 release；这适合 producer 发出“有一个新 item”的 permit。

Mutex 主要表达 ownership 和 critical-section access。两者的公平性、异常行为、priority inheritance 等保证可能不同，不能只因都取 0/1 就认为完全可互换。

并非把一般 counting semaphore 初始化为 1，就由 API 自动保证永远不超过 1。重复 release 可能增加 permits；是否检查取决于具体接口。

### 5.3 Permit Lifetime and Failure

正确模式是 acquire 成功后，保证每个退出路径恰好 release 一次：

```text
acquired = acquire()
if acquired:
    try:
        use resource
    finally:
        release()
```

如果 acquire 超时失败，不应 release 一个未取得的 permit。若取得 permit 后创建 resource 失败，也要按协议返还它。

Permit leak 会让容量逐渐下降，最终全部 workers 卡住；over-release 则可能突破资源上限。Bounded semaphore 可以帮助发现超出初始容量的 release，但不自动识别所有业务层错误。

### 5.4 Runnable Python: A Limit of Two Active Users

这里用 events 控制演示顺序，不依赖 `sleep` 猜测 thread timing：

```python
from threading import BoundedSemaphore, Event, Lock, Thread

permits = BoundedSemaphore(2)
state_lock = Lock()
two_inside = Event()
allow_finish = Event()
active = 0
peak = 0

def use_resource():
    global active, peak
    with permits:
        with state_lock:
            active += 1
            peak = max(peak, active)
            if active == 2:
                two_inside.set()

        try:
            allow_finish.wait()
        finally:
            with state_lock:
                active -= 1

workers = [Thread(target=use_resource) for _ in range(4)]
for worker in workers:
    worker.start()

two_inside.wait()   # 已有两个 workers 占用 permits
allow_finish.set()  # 允许它们完成，后面的 workers 才能进入

for worker in workers:
    worker.join()

print("Peak active:", peak)
assert peak == 2 and active == 0
# Peak active: 2
```

Semaphore 负责容量；`state_lock` 负责 `active/peak` 的复合更新；events 负责阶段协调。这三者的职责不能混为一谈。Event 的语义见 Section 8。

这里的容量是 **concurrent users**，不是“每秒请求数”。Rate limiting 还需要时间窗口、token refill 等额外规则。

## 6. Condition Variables and Predicate Waiting

### 6.1 Predicate, Mutex, and Notification

Condition variable 用于等待一个 **shared-state predicate**，不是用来永久保存通知。

三个组成部分：

| Component | Example |
| --- | --- |
| **Shared state** | Queue contents、closed flag |
| **Mutex** | 保护 state 与 predicate checks |
| **Condition variable** | 在条件暂不满足时等待，并接收重新检查的机会 |

```text
lock(mutex)
while predicate is false:
    condition.wait(mutex)
perform operation while holding mutex
unlock(mutex)
```

Wait 原子地协调“释放 mutex 与进入等待”；返回前重新取得 mutex。这里的 atomicity 是相对于符合相同 protocol 的相关操作，避免在检查条件后和真正等待之间漏掉更新。

### 6.2 Why the Loop Is Necessary

即使收到 notification，也不能假设 predicate 仍然 true：

1. 某些 APIs 允许 spurious wakeup。
2. 被唤醒后仍需重新竞争 mutex。
3. 另一个 consumer 可能先把 item 拿走。
4. 一组 waiters 可能等待不同 predicates。

所以用 `while` 或带 predicate 的 wait helper。若 predicate 不成立，继续等待，而不是访问不存在的 item。

**Wakeup 不等于把 mutex 直接交给 waiter，也不等于 reserved ownership of an item。** [Condition-variable wait](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3.html)

### 6.3 Lost Wakeup and the Role of State

Incorrect protocol：

```text
Consumer checks empty without proper coordination
Producer adds item and notifies; consumer not yet waiting
Consumer begins waiting and may sleep indefinitely
```

正确做法不是“让 producer 多通知几次”，而是让 predicate check、state update 和 wait 使用一致的 mutex protocol。

如果 producer 已经先更新 state，即使当时没有 waiter，之后的 consumer 会检查到 predicate true，根本不需要等待。**保存工作事实的是 state，不是过去那次 notification。**

### 6.4 notify_one vs notify_all

- **notify_one:** 让某个 waiter 有机会继续，通常用于只新增一个可消费 resource 等场景。
- **notify_all:** 让所有 waiters 重新检查，适合 shutdown、phase change，或无法确定哪类 waiter 可前进时。

若多个不同 predicates 共用同一个 condition，notify_one 可能唤醒不符合条件的一方，使真正能前进的一方继续睡眠。可以使用不同 condition variables 共用同一 mutex，或在正确性需要时 notify_all。

Notify_all 可能造成大量 waiters 同时竞争，再多数睡回去，形成 thundering herd。先保证协议正确，再优化 wakeup 数量。

调用 notify 是否必须持有 lock，依 API 而异。Python Condition 要求持有相应 lock；POSIX/C++ 规则不同，但 predicate updates 仍需正确同步。不能把一门语言的调用约束当成所有 platforms 的规则。

### 6.5 Runnable Python: A Closeable Bounded Buffer

设计 contract：

- `put`：满时等待；关闭后拒绝新 item。
- `get`：空且未关闭时等待；关闭后先 drain 已有 items。
- 已关闭且为空时，`get` 抛 `EOFError`。
- `close`：改变 state，并唤醒可能等待的 producers/consumers。

```python
from collections import deque
from threading import Condition, Thread

class BoundedBuffer:
    def __init__(self, capacity):
        if capacity <= 0:
            raise ValueError("Capacity must be positive")
        self.capacity = capacity
        self.items = deque()
        self.closed = False
        self.changed = Condition()

    def put(self, item):
        with self.changed:
            while len(self.items) == self.capacity and not self.closed:
                self.changed.wait()
            if self.closed:
                raise RuntimeError("Buffer is closed")
            self.items.append(item)
            self.changed.notify_all()

    def get(self):
        with self.changed:
            while not self.items and not self.closed:
                self.changed.wait()
            if not self.items:
                raise EOFError
            item = self.items.popleft()
            self.changed.notify_all()
            return item

    def close(self):
        with self.changed:
            self.closed = True
            self.changed.notify_all()

buffer = BoundedBuffer(2)
received = [[], []]

def consume(index):
    while True:
        try:
            item = buffer.get()
        except EOFError:
            return
        received[index].append(item)

workers = [Thread(target=consume, args=(index,)) for index in range(2)]
for worker in workers:
    worker.start()

for item in range(6):
    buffer.put(item)
buffer.close()

for worker in workers:
    worker.join()

combined = sorted(received[0] + received[1])
print(combined)
assert combined == list(range(6))
# [0, 1, 2, 3, 4, 5]
```

这个例子用一个 condition 表达多个 state changes，因此采用 notify_all。每个 consumer 只写自己的 result list，main 在 join 后汇总，避免引入另一个 shared-results locking 问题。

不要将 queue-empty 作为“所有工作结束”的唯一条件；暂时 empty 时，producer 可能还会继续产生 items。这里用 explicit closed state 表达 completion。

### 6.6 Timeouts and Cancellation

Timed wait 返回后，仍需在 lock 下检查 predicate 和终止条件。Timeout 可能与 producer update 同时发生，API 返回本身不是完整业务决策。

若循环使用 timeout，应计算基于 monotonic clock 的整体 deadline，再使用剩余时间，避免每次无效 wakeup 都重新等待完整时长。

Cancellation/shutdown 应进入 predicate：

```text
wait until: work_available OR closed OR cancelled
```

销毁 condition/mutex 前，先让相关 waiters 退出并完成必要 joins；不能释放它们仍在使用的 state。

## 7. Read-write Locks

### 7.1 Shared Readers, Exclusive Writer

Read-write lock 支持：

- 多个 compatible readers 同时访问。
- Writer 获得 exclusive access。
- Writer 持有期间不能有其他 reader/writer 访问被保护 state。

```text
Read phase:   R1 + R2 + R3
Write phase:  W1 only
```

“Reader”必须真的不修改受保护 state。某个看似查询的 API 如果更新 cache、lazy initialization 或 access statistics，就未必适合作为纯 reader。

### 7.2 Fairness Policies

| Policy | Behavior | Risk |
| --- | --- | --- |
| **Reader preference** | 已有 readers 时继续接纳新 readers | Writer 可能长期等不到空档 |
| **Writer preference** | 有 waiting writer 后限制新 readers | 持续 writers 可能延迟 readers |
| **Fair / queued policy** | 按某种队列或阶段顺序安排 | 更多管理成本；精确保证依 API |

“Read-write lock”这个名字不告诉你它是哪一种 policy，必须查实现。

### 7.3 Why Read-heavy Does Not Always Mean Faster

Read lock 也需要修改内部 synchronization metadata，可能在同一 cache line 造成竞争。对于极短 reads，一个普通 mutex 或 immutable snapshot 反而可能更简单、更快。

评估 read duration、reader 数、writer latency、cache traffic 和 object size，而不是只统计“90% 都是 reads”。

### 7.4 Upgrade and Downgrade

两个 readers 都持有 shared lock，再等待升级为 exclusive lock，可能互相阻塞。

若 API 没有明确支持 atomic upgrade，常见处理是：

1. 释放 read lock。
2. 获取 write lock。
3. **重新检查条件**，因为中间 state 可能已改变。
4. 再执行 update。

Downgrade 是否能保持连续保护，也取决于 API。不能自己组合两个 calls 就默认中间没有窗口。

## 8. Events, Barriers, Latches, Joins, and Futures

### 8.1 Match the Coordination Shape

| Primitive | 表达什么 | 常见场景 |
| --- | --- | --- |
| **Event** | 某个 flag/state 已被设置 | 启动 gate、停止请求、一次性准备完成 |
| **Barrier** | 一组参与者都到达当前 phase | 分阶段计算 |
| **Countdown latch** | 一组完成通知使 count 降到零 | 主协调者等待多个工作结束 |
| **Join** | 指定 thread 已终止 | 收尾和 lifetime 管理 |
| **Future / promise** | 某个 computation 的结果或失败 | 提交 task 后稍后取结果 |

这些工具不都提供 critical-section exclusion，也不是 semaphore 的不同名字。

### 8.2 Event State vs Notifications

例如 Python Event 维护 flag：

- `set()` 使 flag 为 true，唤醒 waiters。
- Flag 为 true 时，后来的 wait 也能通过。
- `clear()` 让后续等待重新受阻。

连续调用两次 set，不表示存了两个 permits。需要累计事件次数时，考虑 queue、counter + condition 或 semaphore。

多轮协议中，过早 clear 可能让部分参与者错过这一轮。可以用 generation number 或 barrier 等明确区分 phases。Section 5 的 event 只用于一次性 gate，没有 reset 的复杂度。

### 8.3 Runnable Python: Barrier Between Two Phases

```python
from threading import Barrier, Thread

participants = 3
phase_done = Barrier(participants)
partial = [0] * participants
totals = [0] * participants

def worker(index):
    partial[index] = index + 1
    phase_done.wait()
    totals[index] = sum(partial)

workers = [Thread(target=worker, args=(i,)) for i in range(participants)]
for worker in workers:
    worker.start()
for worker in workers:
    worker.join()

print(totals)
assert totals == [6, 6, 6]
# [6, 6, 6]
```

Barrier 前每个 worker 写自己的 slot；所有参与者到达后，才进入读取全部 partial results 的 phase。

若下一轮还要覆写 `partial`，必须再保证这一轮 readers 已结束，例如增加另一个 phase barrier。一个 barrier 只协调相应 phase，不能自动保护所有后续读写。

某个 participant 提前退出、抛 exception 或永远不到达，会破坏协议。实际代码应考虑 timeout、abort/broken barrier 和 error propagation。

### 8.4 Completion Is Also a Lifetime Boundary

Join/future completion 可以帮助保证 referenced objects 在 worker 使用期间仍然存在。异步 task 捕获 local reference 时，不能让 caller 先销毁被引用 object。

等待 future 时也不能一直持有 task 完成所需的 mutex，否则产生 dependency cycle。Future 会承载结果，不代表“等结果时一定不会 deadlock”。

不同 futures 对多次读取、多个 waiters、异常重抛等语义不同，需要按 language API 说明。


## 9. Atomic Operations and Compare-and-Exchange

### 9.1 Atomicity Has a Scope

Atomic operation 对它规定的那次 operation 提供不可分割的语义，不意味着：

- Function 中所有 instructions 合成一个 transaction。
- 多个 atomic variables 组成一致 snapshot。
- Object lifetime 自动安全。
- Operation 一定 lock-free。
- 周围普通 memory accesses 已建立所需 ordering。

Hardware instruction 原子性与 language-level atomic contract 也不是同一层。C++ code 应使用 `std::atomic` 等合法机制，不能只凭“这个 CPU 的普通 aligned load 看起来不会 torn”绕过 memory model。

### 9.2 Load, Store, and Read-modify-write

```cpp
// C++ fragment
std::atomic<int> count{0};

int old_value = count.load();  // one atomic load
count.store(7);               // one atomic store
int previous = count.fetch_add(1);  // one atomic read-modify-write
```

`load() + store()` 仍是两个 operations，其他线程可以在中间修改 state。要表达 increment，应使用合适的 atomic RMW，而不是自己拼接两个 atomic calls。

Atomicity 也不保证整型永不溢出或业务值合法；数据范围仍是程序 contract 的一部分。

### 9.3 CAS Semantics

**Compare-and-exchange / CAS** 可以概括为：

```text
If current == expected:
    write desired
    return success
Else:
    report failure and provide current value as specified by the API
```

C++ 的 `compare_exchange_weak` 失败时会更新 expected，且允许 spurious failure；因此通常放在 loop 中。`strong` 不允许这种纯粹的 spurious failure，但也可能因真正竞争失败。

### 9.4 C++ Fragment: Reserve One Unit

```cpp
#include <atomic>

std::atomic<int> stock{10};

bool try_reserve_one() {
    int observed = stock.load();  // default seq_cst

    while (observed > 0) {
        if (stock.compare_exchange_weak(observed, observed - 1)) {
            return true;
        }
        // On failure, observed is refreshed.
        // Re-check availability before trying again.
    }
    return false;
}
```

成功 CAS 是这次 reservation 的 linearization point。两个 callers 都看到 stock=1 时，只能有一个成功把对应当前值从 1 改成 0；另一个必须基于新的 observed state 再判断。

假设其他 access paths 也遵守 stock 的业务规则，且数值没有非法修改。这段代码只做 stock reservation，不代表同时完成 billing、inventory database update 等外部事务。

不要在可能反复执行的 CAS attempt 中做不可重复的 side effects，例如发送 email 或扣外部账户。通常先决定成功，再根据完整业务协议处理后续动作。

### 9.5 Weak vs Strong and Memory-order Arguments

Weak 适合已经有 retry loop 的算法；strong 适合某些不希望因为 spurious failure 额外重试的情况。不能一概说其中一种更快。

C++ CAS 有 success/failure memory orders；失败路径是 load，不执行成功的 write，因此有专门约束，不能为 failure 使用 release 或 acq_rel。初学示例采用默认 seq_cst，先证明算法正确，再基于具体 standard/version 与 contract 优化。[C++ compare-and-exchange](https://eel.is/c++draft/atomics.types.generic)

CAS 能帮助构建 optimistic protocols，但需要重新检查 observed state。若只 retry 写入、不重新验证 predicate，仍可能产生业务错误。

## 10. Memory Visibility, Ordering, and Happens-before

### 10.1 Three Separate Questions

| Concept | Question |
| --- | --- |
| **Atomicity** | 某个 operation 会不会被观察成部分完成，或丢失并发更新？ |
| **Visibility / ordering** | 其他 thread 在什么规则下观察到哪些 writes？ |
| **Lifetime** | 被访问的 object 是否仍然存在？ |

只满足其中一项不代表并发代码正确。CPU cache coherence 的背景见 [Memory, Section 12.7](memory.md#127-coherence-and-false-sharing)，它不能替代 language-level synchronization。

### 10.2 Why Source-code Order Is Not Enough

Compiler 和 CPU 可以在 language/hardware contract 允许的范围内优化与重排。不同 locations 的观察顺序，不必等于另一线程源码的书写顺序。

不要把问题简化成“每个 thread 有一份永远不更新的内存副本”。真实系统涉及 registers、caches、store buffers 和 language optimization rules；正确性应依赖规定的 synchronization edges，而不是某个硬件故事。

### 10.3 Happens-before Is a Relation, Not a Stopwatch

在本篇使用的 C++ 简化模型中：

- 同一 thread 中相关的 **sequenced-before** 关系提供局部顺序。
- 某些 synchronization operations 建立跨 thread 的 **synchronizes-with** 关系。
- 这些关系通过 transitivity 形成 **happens-before**。

```text
Thread A                         Thread B
write payload
     |
sequenced-before
     v
release / unlock ───────────> acquire / lock
                 synchronizes-with    |
                                sequenced-before
                                      v
                                 read payload
```

“现实中 A 恰好先执行了”不自动构成语言规定的 happens-before。Sleep、日志时间戳或一个 thread “通常更慢”都不是同步证明。[C++ happens-before rules](https://eel.is/c++draft/intro.races)

### 10.4 Mutex-based Publication

Writer 在 lock 内更新 object，unlock；reader 之后成功 acquire **同一个 mutex**，再读取。对应 synchronization 关系让之前受保护的 writes 按协议可见。

这不等于“unlock 把所有 cache 全部刷回 RAM”。语言保证通常由 compiler constraints、atomic instructions 和 coherence 等机制实现，具体指令依平台。

若 reader 获取另一把无关 lock，不会因为“也加了锁”就获得同样的保证。

### 10.5 Release/acquire Publication

以下 C++ fragment 是 **one-shot protocol**：payload 只由 producer 写一次，ready 发布后不再修改 payload，也不重置 ready。

```cpp
#include <atomic>

int payload = 0;
std::atomic<bool> ready{false};

void producer() {
    payload = 42;
    ready.store(true, std::memory_order_release);
}

int consumer() {
    while (!ready.load(std::memory_order_acquire)) {
        // Teaching example; a blocking wait may be better in practice.
    }
    return payload;
}
```

推理链：

1. Payload write 在 release store 之前。
2. Consumer 的 acquire load 观察到该 release store 发布的 true。
3. 该关联建立跨 thread synchronization。
4. Payload read 在 acquire 之后，因而有正确 ordering。

**不是任意 release 与任意 acquire 都会互相同步。** 必须满足标准规定的关联，例如 acquire 观察到对应 release 所发布的值；这里只使用最直接的例子，不展开 release sequence。

改成重复多轮 producer/consumer 后，还需要保证上一轮 reader 结束、下一轮 write 才能开始。把 ready 简单反复设 true/false 不能自动形成完整协议。[C++ atomic ordering](https://eel.is/c++draft/atomics.order)

### 10.6 Memory Orders at a Glance

| Order | Typical role | 不应推断 |
| --- | --- | --- |
| **relaxed** | Atomic access 与该 object 的 modification order | 其他普通 data 已被发布 |
| **release** | 发布之前的 effects，供匹配的 acquire 观察 | 无条件通知所有 readers |
| **acquire** | 在匹配观察后约束后续 accesses | 即使读到旧值也建立了所需 handoff |
| **acq_rel** | RMW 同时承担 acquire/release 角色 | 适用于所有普通 load/store |
| **seq_cst** | 对相应 SC operations/fences 提供额外总顺序约束 | 把任意多步算法自动变成 transaction |

默认 seq_cst 更容易作为起点，但不能弥补算法中缺失的 compound operation。也不要在没有证明时把所有 orders 改成 relaxed 来“优化”。

**Relaxed counter** 适合独立统计；如果 counter 的值意味着“所有 worker 已经发布了其他 results”，就必须另行证明所需 ordering。

### 10.7 A Safe Litmus Test to Reason About

假设 x、y 都是初值为 0 的 C++ atomics：

```text
Thread A                         Thread B
x.store(1, relaxed)              y.store(1, relaxed)
r1 = y.load(relaxed)             r2 = x.load(relaxed)
```

在 C++ relaxed model 中，r1=0 且 r2=0 是允许的 outcome；不是普通 variables 的 undefined-behavior 演示。

若四个 operations 都使用 seq_cst，则两个 reads 都为 0 会要求一个不可能的总顺序 cycle，因此该 outcome 被排除。

只把 stores 改 release、loads 改 acquire，仍不能保证排除双零：如果两个 loads 都读取初始值，就没有建立期待的 release/acquire handoff。

这是 language-model 推理，不保证某台机器上的一次或多次测试一定出现所有允许 outcomes。

### 10.8 volatile and Safe Initialization

C/C++ `volatile` 不提供通用 inter-thread atomicity 或 happens-before；它有其他用途，例如特定平台下对特殊 memory 的访问约束。

Java `volatile` 有相应 visibility/order semantics，但 `volatile count++` 仍是 compound action，不是 atomic increment。不同 languages 的同名 keyword 不能直接类推。[Java memory model](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html)

Lazy initialization 常见问题是“pointer 非空，但 object 没被正确发布”。优先使用成熟机制，例如 C++ `std::call_once` 或适用的 thread-safe local-static initialization，而不是手写未经证明的 double-checked locking。

## 11. Deadlock, Livelock, Starvation, and Priority Inversion

### 11.1 Build the Wait-for Graph

为每个 waiting execution flow 写出“谁需要谁才能继续”：

```text
A waits for resource owned by B
B waits for resource owned by C
C waits for resource owned by A

A → B → C → A
```

对于每个 resource 只有一个 exclusive instance 等简单模型，等待 cycle 可以直接揭示 deadlock。多实例资源模型更复杂，不能把任意画出来的 cycle 都当成充分证明。

Deadlock 也可以涉及 join、future、queue capacity，不限于 mutexes：

```text
Main holds M and waits for worker.join()
Worker needs M before it can finish
```

### 11.2 Four Conditions and Concrete Prevention

四个传统必要条件及基础示例见 [Process & Thread, Section 8.7](process-thread.md#87-deadlock-livelock-starvation-and-priority-inversion)。这里关注如何落地：

| Condition | 可考虑的破坏方式 | Trade-off |
| --- | --- | --- |
| **Mutual exclusion** | 对适合的数据改用 immutable/shared-read 设计 | 不是所有 resources 都可共享 |
| **Hold and wait** | 获取下一资源前释放已有资源，或合并 acquisition | 需要重查 state，可能增加 retries |
| **No preemption** | 对可回滚的资源设计安全撤销 | 不能随意抢走普通 mutex 并假装 state 一致 |
| **Circular wait** | 全局 lock order | 所有 paths 必须遵守，包括 callbacks |

Section 3 的 account IDs 给出全局顺序，因此不能形成 strictly increasing order 后又回到起点的 lock cycle。

### 11.3 Hidden Dependencies

常见危险 paths：

- 持有 lock 调用外部 callback，callback 再进入当前 object。
- 持有 lock 执行 I/O，而处理 I/O 结果的路径也需要该 lock。
- Thread pool 的所有 workers 等待同一 pool 内尚未运行的子任务。
- Producer 持有 queue mutex，然后等待“queue 有空位”的 semaphore；consumer 需要该 mutex 才能腾空。
- Shutdown thread 等 worker，worker 又等待 shutdown thread 尚未发送的 stop state。

Review 时追踪整个 dependency chain，而不只是函数内显眼的两把 locks。

### 11.4 Timeout Is Not Automatic Recovery

某个等待超时后，可能已经完成一半业务操作。需要明确：

- 已获取的 resources 如何释放？
- State 是否要 rollback？
- Caller 能否 retry，是否会重复执行？
- 被等待的 task 是否仍在继续？
- “超时”与“操作确定失败”是否被错误地等同？

Timeout 有助于限制 caller waiting 和发现问题，但不证明系统没有 deadlock，也不意味着后台 action 已取消。

### 11.5 Livelock and Starvation

**Livelock:** 参与者不断活动、重试或退让，但没有完成业务进展。两个 tasks 总是同时释放再同时争抢，就是可能的模式。Backoff、randomization 或改变协议可以减少这类同步碰撞，但仍需分析 liveness。

**Starvation:** 系统整体继续前进，某个参与者却长期拿不到机会。Unfair locks、持续 writer/read preference 或 scheduling 都可能造成。

公平性不是“所有 threads 恰好每隔一秒执行一次”，而是特定 protocol 对机会或等待的保证。

### 11.6 Priority Inversion

High-priority H 等待 low-priority L 持有的 mutex，medium-priority M 不断抢占 L，导致 H 间接被 M 阻挡。

Priority inheritance 可以让 L 临时继承需要的高 priority，以尽快完成 critical section。Priority ceiling 是另一类提前约束协议。两者依平台/API 支持，不是给 thread 设置高 priority 就自动解决。

它们也不允许忽略 lock-order deadlock 或无限长 critical sections。

## 12. Classical Synchronization Problems

### 12.1 Bounded Producer-consumer with Semaphores

Section 6 已给出 condition-variable implementation。这里用另一种机制说明 **容量 accounting**，不重复完整代码。

设 capacity=N：

```text
empty_slots = semaphore(N)
available_items = semaphore(0)
queue_mutex = mutex()

Producer:
    acquire(empty_slots)
    lock(queue_mutex)
    enqueue(item)
    unlock(queue_mutex)
    release(available_items)

Consumer:
    acquire(available_items)
    lock(queue_mutex)
    item = dequeue()
    unlock(queue_mutex)
    release(empty_slots)
```

必须先等 permit，再进入 queue critical section；否则可能持有 mutex 等别人改变 queue，而别人又需要 mutex。

不要简单断言运行中的任意瞬间都有 `empty_slots + available_items = N`。Producer/consumer 可能已经 reserve permit，但尚未完成 queue update。正确 reasoning 还需计入 **in-flight reservations**；在没有这些中间操作的稳定点，关系才可以简化。

Production implementation 还需要处理异常后的 permit restoration、close、cancellation、multiple consumers 和 resource cleanup。

### 12.2 Readers-writers Problem

目标是允许 compatible reads，同时让 writes exclusive。真正的面试重点通常是 policy：

- Waiting writer 是否阻止后来 readers？
- 连续 writers 是否可能饿死 readers？
- Reader 会不会偷偷修改 cache？
- Upgrade 是否安全？

Primitive 与 trade-offs 已集中在 Section 7。不要把“允许多读单写”当成 fairness 的完整证明。

### 12.3 Dining Philosophers

假设 N 个 philosophers，每人吃饭需要左右两个 exclusive forks。若全部先拿左边再等右边，会出现 circular wait。

可以比较：

| Approach | 为什么能避免对应 deadlock | 仍需考虑 |
| --- | --- | --- |
| **Global fork ordering** | 所有人按同一编号顺序取 forks，消除 cycle | Fairness 与等待时间 |
| **At most N-1 contenders** | 在经典 ring、先左后右模型下，避免所有人各占一把形成完整环 | Admission semaphore 必须正确释放 |
| **Waiter/arbitrator** | 协调者只在适当条件下发放一对 resources | 协调成本、policy、starvation |

这些解释都基于经典模型和有限持有时间等 assumptions。Deadlock-free 不自动等于 starvation-free。

### 12.4 A Reusable Phase Protocol

分阶段计算常有：

```text
Write own partial result
        ↓
Barrier 1: all results are ready
        ↓
Read / combine results
        ↓
Barrier 2: nobody still reads this generation
        ↓
Overwrite buffers for next generation
```

第二个 barrier 保护 reuse timing，而不是重复第一个。也可以使用 double buffering 或 generation counters，但需要明确哪个 generation 的 data 由谁读写。

这是将 “data ready” 与 “data no longer in use” 分开的典型例子，后者同样是 synchronization requirement。


## 13. Lock-free Algorithms, ABA, and Memory Reclamation

### 13.1 Progress Guarantees

在相关算法模型与 scheduling assumptions 下：

| Term | 保证层级 | 不能推断 |
| --- | --- | --- |
| **Blocking** | 进展可能依赖某个 owner/participant 继续执行 | 一定慢 |
| **Obstruction-free** | 单独运行足够长时可以完成 | 竞争下整体一定持续进展 |
| **Lock-free** | 整体持续取得进展，有 operations 完成 | 每个 thread 都不会 starvation |
| **Wait-free** | 每个 operation 有有界步骤的完成保证 | 固定 wall-clock latency 或必然更快 |

这里讨论算法步骤，不保证 OS 不抢占 thread、不发生 page fault。CAS loop 可能 lock-free，却不是 wait-free，因为某个 caller 可以持续输掉竞争。

算法调用的 allocator、reclamation 或 logging 也可能 block。证明一个内部 CAS loop lock-free，不等于证明整条 API path lock-free。

### 13.2 ABA Problem

CAS 只比较当前 representation 是否等于 expected，不记录中间历史：

```text
Thread A reads head = X
A pauses

Thread B changes head: X → Y → X

A resumes, sees head == X, CAS may succeed
```

如果算法认为“head 仍等于 X，所以结构没变”，这个推断可能错误。ABA 可以涉及 value cycles，也可能涉及 freed memory 被复用为相同 pointer address。

某些 counter algorithms 不关心中间历史，因此 value 回到旧值未必是 bug；是否有问题取决于 invariant。

### 13.3 Why Atomic Pointers Do Not Solve Lifetime

Thread A atomic-load 得到 node pointer 后，B 仍可能移除并释放 node。如果 A 接着 dereference，就可能 use-after-free。

Atomic pointer 保护的是 pointer value 的访问，不自动延长 pointee lifetime。这个问题与 ABA 相关，但不能完全等同：即使 address 从未回到原值，也可以已经发生 use-after-free。

### 13.4 Reclamation Strategies

| Strategy | Idea | Complexity / cost |
| --- | --- | --- |
| **Reference counting / shared ownership** | 活跃 owning references 维持 lifetime | Count contention、cycles、acquisition protocol |
| **Hazard pointers** | Readers 发布正在保护的 pointers，reclaimer 延迟释放 | 正确 publication/recheck、retired lists、扫描成本 |
| **Epoch-based reclamation** | Readers 标记参与 epoch，等待旧 readers 退出再释放 | 慢或停住的 participant 可能推迟回收 |
| **RCU-style design** | 发布新版本，经过适当 grace period 后回收旧版本 | Reader/update protocol 和 grace-period 管理 |

Hazard-pointer reader 通常需要发布保护后重新检查 shared source，再安全使用 pointer；不能先 dereference 再宣布“我在保护它”。

RCU 把 removal 与 reclamation 分开。典型情况下，updater 等待可能仍持有旧 reference 的既有 read-side critical sections 结束，再回收旧版本。它不代表可以忽略 writer-writer coordination。[Linux: What is RCU?](https://docs.kernel.org/RCU/whatisRCU.html)

Tagged pointers/version counters 可以检测部分 ABA，但存在 tag wraparound、representation 和硬件支持问题，也不自动解决 reclamation。

### 13.5 When to Use These Techniques

优先使用成熟、符合 workload 的 concurrent data structures。只有在 profiling 证明 locking 是瓶颈，且团队能维护完整 memory-model/lifetime proof 时，再考虑自写复杂 lock-free algorithm。

“没有显式 mutex”只是源码表象，不是 correctness、progress 或 performance 的证明。

## 14. Synchronization Scope and Alternative Designs

### 14.1 Thread vs Process Synchronization

两个 processes 都写：

```text
mutex = create_mutex()
lock(mutex)
modify shared file or memory
```

不代表它们使用了同一个 synchronization object。分别创建的 private locks 不能协调共同 resource。

Cross-process synchronization 需要适合该 scope 的 object 与 storage，例如支持 process-sharing 的 mutex、named semaphore 或经过约定的 IPC owner。

POSIX process-shared mutex 一般需要：

- Mutex storage 位于双方都能访问的 shared memory。
- 正确 initialization 与 `PTHREAD_PROCESS_SHARED` 等 attributes。
- 平台确实支持所需功能。
- 明确 initialization、destruction 和 owner failure 的协议。

只有“mutex 放进 shared memory”还不够；普通 C++ `std::mutex` 不能直接假设具备 portable inter-process semantics。[POSIX process-shared mutex attributes](https://man7.org/linux/man-pages/man3/pthread_mutexattr_setpshared.3.html)

### 14.2 Owner Failure and Robust Mutexes

Process 持有 lock 时异常退出，可能同时留下不完整的 shared data update。

某些 robust mutex APIs 会让下一位 owner 获得 lock，并报告 `EOWNERDEAD` 等状态，要求 caller 修复受保护的数据，再标记 consistent。

**Robust 不等于自动 rollback。** 修复需要 application 知道哪些 state 可能只完成了一半；无法恢复时要进入相应失败状态，不能简单忽略错误继续操作。[pthread_mutex_consistent](https://man7.org/linux/man-pages/man3/pthread_mutex_consistent.3.html)

### 14.3 Async Tasks Need Compatible Primitives

一个 event-loop thread 上的多个 tasks，也可能在 `await` points 之间发生 race：

```text
Task A: checks stock → awaits something
Task B: checks stock → updates
Task A: resumes → acts on stale assumption
```

“只有一个 OS thread”不等于整个 async operation 不会交错。

对 asyncio tasks 通常使用 `asyncio.Lock` 等 compatible primitives。等待它们时可以暂停 task，而不是把整个 event-loop thread block；这些 primitives 本身通常不用于跨 OS threads 的同步。[asyncio synchronization](https://docs.python.org/3/library/asyncio-sync.html)

不要用 OS-thread ownership 的 recursive lock 来隔离同一 thread 上的不同 tasks：它可能把所有 tasks 当成同一个 owner。即使用 async lock，也要避免持有它跨越不受控的长等待或循环依赖。

### 14.4 Single Owner and Message Passing

把 state 交给一个 owner，其他 participants 提交 messages：

```text
Workers → commands → owner → mutable state
                      |
                      └── results / acknowledgments
```

好处是 state mutation 的顺序更清晰，owner 内可能不需要多 writer locking。但：

- Queue 仍需要正确实现。
- Owner 可能成为 bottleneck。
- Caller 需要区分 request accepted 与 operation completed。
- Messages 中的 mutable references 需要 ownership-transfer 或 copy 协议。
- 一次 message handler 如果执行到中途 await 后允许其他 handlers 进入，仍可能有 interleaving。

Message passing 改变 coordination 方式，不是消除了所有 synchronization。

### 14.5 Database Transactions

如果 state 本来就在 database，应使用 database 能保证的 conditional operations、constraints 和合适 isolation level，而不是仅靠某个 application process 的 local mutex。

Conceptual SQL example：

```sql
UPDATE inventory
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
```

检查 affected-row count，并正确处理 transaction commit/failure，比 application 先无保护地 SELECT 再写回旧值更适合表达这次条件更新。还需要确保其他 writers 遵守相同业务约束。

Database 并发控制可能让 writers 等待，也可能在冲突时要求 retry；具体依 engine/isolation。PostgreSQL 文档说明了 concurrent updates 对目标 row 的等待与条件重新检查行为。[PostgreSQL transaction isolation](https://www.postgresql.org/docs/14/transaction-iso.html)

一次数据库更新不自动包含外部 payment、email 或另一个 service 的状态变化；这些要另行设计一致性与 retry semantics。

### 14.6 Distributed Coordination Boundary

多机器场景还会遇到 network delay、partitions、lease expiry、process pauses 等问题。一个 client 认为自己仍持有 lease，不意味着 resource server 必须接受它的 stale writes。

某些设计使用 fencing tokens 让受保护的 service 拒绝过期 owners。具体协议属于 distributed systems；不能把 local mutex 的 owner/failure assumptions 原样搬过去。

## 15. Performance, Testing, and Debugging

### 15.1 Reduce Coordination Demand

优化前先知道时间花在哪里。常见方向：

- **Partition state:** 独立 keys/regions 使用不同 owners 或 locks。
- **Local aggregation:** 每个 worker 累计，再合并。
- **Batching:** 减少 acquisition 次数，但控制单次 hold time。
- **Immutable snapshots:** Readers 读稳定版本，另行处理 publication/reclamation。
- **Shorter critical sections:** 把不依赖保护的准备工作移出去，再 lock 后 revalidate。
- **Bounded concurrency:** 降低过多 runnable waiters 和外部 resource pressure。

不要只看 lock 数量。一个很少竞争的 mutex 可能比不断 retries 的 atomic loop 更便宜。

### 15.2 What to Measure

| Metric | 可能揭示 |
| --- | --- |
| **Lock wait time** | Contention 与 owner delays |
| **Lock hold time** | Critical section 是否包含慢工作 |
| **Retry / CAS failure rate** | Optimistic design 是否竞争过强 |
| **Runnable threads / context switches** | Oversubscription、parking/wakeup 成本 |
| **CPU utilization** | 有用工作 vs spin/retry；低 usage 也可能在等待 |
| **Queue depth and oldest-item age** | Overload、backpressure 问题 |
| **Throughput and p95/p99 latency** | 增加 concurrency 是否损害慢请求 |
| **Allocation/reclamation cost** | “Lock-free”操作是否把成本转移到其他路径 |

Cache-line bouncing 与 false sharing 的硬件背景见 [Memory, Section 12.7](memory.md#127-coherence-and-false-sharing)。Shared atomic counter 也可能成为 cache-coherence bottleneck。

### 15.3 Deterministic Interleavings Beat Guessing with sleep

测试某个可能错误的窗口时，用 barriers、events 或 test hooks 控制执行阶段，例如：

```text
A completes the check
Test gate pauses A
B changes relevant state
Test gate resumes A
Verify A rechecks or rejects stale work
```

这样可以稳定验证某个 schedule，而不是加 sleep 祈祷它发生。但测试 hooks 自己的 synchronization 也会改变 execution，不能据此证明原 memory ordering 在所有情况下正确。

避免用永久挂住的测试展示 deadlock；使用有界的 external timeout、状态采集和明确 cleanup，让测试失败可诊断。

### 15.4 Testing Levels

1. **Sequential contract tests:** 输入、输出、关闭/异常边界是否符合 API。
2. **Invariant checks:** Transfer 总量、queue capacity、no duplicate/lost items。
3. **Controlled schedules:** 针对 check-act、close-wait、publish-read 的窗口。
4. **Stress and varied environments:** 多轮、不同 worker counts、不同平台；不是正确性证明。
5. **Race detection:** 检测实际 execution 中的未同步 accesses。
6. **History checking / model exploration:** 对关键 concurrent object 分析是否存在符合 contract 的合法历史。

ThreadSanitizer 可发现多类 data races，但不是所有业务 race、deadlock 或 missing progress 的自动证明器；没报告不等于所有 schedules 都正确。[Clang ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html)

AddressSanitizer 主要处理 memory safety 等问题，不能当作 ThreadSanitizer 的替代品。

### 15.5 Diagnose a Hang

先收集 thread/task stacks，再问：

- 哪些执行者是 runnable、blocked、spinning？
- 等的具体 object/predicate 是什么？
- 谁负责使 predicate 成立？
- Owner 是否仍存在，是否又在等待别人？
- 是否忘了 close/notify/release？
- 是否 queue/future dependencies 把所有 workers 占满？
- Wait-for graph 是否有 cycle？

不要只重启或增大 timeout；先明确是 deadlock、starvation、external stall，还是合法但过长的工作。

### 15.6 Review Checklist

- Invariants 与 ownership 写清楚了吗？
- Reads、writes、object destruction 都遵守同一规则吗？
- Multi-lock order 是否全局一致？
- Condition waits 是否在 loop 内检查 predicate？
- Notification、permit、event state 有没有混用？
- Timeout/cancellation/exception paths 是否维护 accounting？
- Publication 与 object lifetime 是否都被证明？
- Shutdown 能唤醒并结束所有需要结束的 waiters 吗？
- Optimizations 是否由 measurements 支持？

## 16. Common Interview Questions

详细解释已经在前面；这里用 question、answer checkpoints 和 links 进行 active recall。

### 16.1 Core Questions

| Question | Answer checkpoints | Reference |
| --- | --- | --- |
| **What does synchronization provide?** | Exclusion、ordering、coordination；区分 safety/liveness | [Section 1](#1-what-is-synchronization) |
| **Race condition vs data race?** | 业务执行顺序 vs 具体 memory-model 冲突 | [1.3](#13-race-condition-vs-data-race) |
| **Can single-core code have a race?** | Interleaving 不要求真正 parallel execution | [1.3](#13-race-condition-vs-data-race) |
| **What should a lock protect?** | State 与 invariant，而不只是某行 assignment | [Section 2](#2-critical-sections-and-correctness-contracts) |
| **What is linearizability?** | Invocation/response 内的逻辑瞬间，保持 real-time order | [2.4](#24-linearizability) |
| **Thread-safe vs reentrant?** | Concurrent callers 与未结束调用的重新进入 | [2.5](#25-thread-safety-vs-reentrancy) |
| **Why use RAII for locks?** | Cleanup 不遗漏，但不是业务 rollback | [3.2](#32-exception-safe-lock-lifetime) |
| **Does recursive locking prevent deadlock?** | 只处理特定同-owner重复 acquire，不消除 wait cycles | [3.4](#34-recursive-locks) |
| **Mutex vs spinlock?** | Parking vs busy waiting；短等待与 scheduling 条件 | [Section 4](#4-spinlocks-blocking-locks-and-futexes) |
| **Does every lock call enter the kernel?** | User-space fast path、contention slow path | [4.3](#43-how-futex-style-waiting-helps) |
| **Mutex vs semaphore?** | Ownership vs permit accounting | [5.2](#52-binary-semaphore-vs-mutex) |
| **Is a semaphore a rate limiter?** | Concurrent capacity 不等于 requests per second | [5.4](#54-runnable-python-a-limit-of-two-active-users) |
| **Why wait in a loop?** | Predicate、spurious wakeup、竞争后 state 改变 | [6.2](#62-why-the-loop-is-necessary) |
| **What causes a lost wakeup?** | Check/wait protocol 的窗口；notification 不保存状态 | [6.3](#63-lost-wakeup-and-the-role-of-state) |
| **notify_one or notify_all?** | Waiter classes、eligible predicates、shutdown、herd cost | [6.4](#64-notify_one-vs-notify_all) |
| **Why not always use a read-write lock?** | Metadata overhead、fairness、read duration | [Section 7](#7-read-write-locks) |
| **Barrier vs latch vs event?** | Phase rendezvous、completion count、persistent flag | [Section 8](#8-events-barriers-latches-joins-and-futures) |
| **Why can atomic code still be wrong?** | Compound invariant、lifetime、ordering | [9.1](#91-atomicity-has-a-scope) |
| **How does CAS work?** | Comparison、conditional update、failure refresh、retry | [9.3–9.4](#93-cas-semantics) |
| **Weak vs strong CAS?** | Spurious failure 与真实竞争 | [9.5](#95-weak-vs-strong-and-memory-order-arguments) |
| **What is happens-before?** | Local order + synchronization + transitivity | [10.3](#103-happens-before-is-a-relation-not-a-stopwatch) |
| **Does acquire always observe a release?** | 必须满足对应观察关系；读旧值不建立期望 handoff | [10.5](#105-releaseacquire-publication) |
| **Does volatile make code thread-safe?** | 先确定 language；不能代替 compound atomicity | [10.8](#108-volatile-and-safe-initialization) |
| **How do you prevent deadlock?** | Wait-for graph、全局 lock order、隐藏依赖 | [Section 11](#11-deadlock-livelock-starvation-and-priority-inversion) |
| **Deadlock-free vs starvation-free?** | 系统能前进，不代表每个参与者都能前进 | [11.5](#115-livelock-and-starvation) |
| **Lock-free vs wait-free?** | Overall progress vs per-operation bounded steps | [13.1](#131-progress-guarantees) |
| **What is ABA?** | 当前值相同不代表历史未变 | [13.2](#132-aba-problem) |
| **Does an atomic pointer protect the object?** | Pointer access 不自动延长 pointee lifetime | [13.3](#133-why-atomic-pointers-do-not-solve-lifetime) |
| **Can the same primitive work across processes?** | Storage、API scope、initialization、owner failure | [14.1](#141-thread-vs-process-synchronization) |
| **Can async code need locks?** | Await 允许 logical operations 交错；使用 compatible primitives | [14.3](#143-async-tasks-need-compatible-primitives) |

### 16.2 Scenario: Implement a Bounded Queue

**Prompt:** Multiple producers and consumers share a bounded queue. How would you synchronize it?

先定义 capacity、blocking behavior、close semantics，再选择 mutex + conditions 或合适 queue library。

**Sample answer:**

> I would protect the queue state with one mutex and wait on explicit predicates for space and available items. Waiters must recheck their predicate after waking. I would also define shutdown behavior so blocked producers and consumers can terminate. For production code, I would prefer a tested bounded queue unless the requirements justify a custom implementation.

Follow-up：close 时仍有 items 怎么办？多个 predicates 共用一个 condition 时 notify_one 是否足够？Producer failure 是否泄漏 capacity permit？

### 16.3 Scenario: Make Inventory Reservation Correct

**Prompt:** Each stock access is atomic, but the system still oversells. Why?

先判断是不是多个独立 operations 构成的 check-then-act，而不是立即加强 memory order。

**Sample answer:**

> Atomic reads and writes do not make the whole reservation atomic. Two callers can both observe available stock before either completes its deduction. I would combine the check and update under a mutex or use a correctly designed compare-and-exchange loop. External side effects and transaction boundaries still need their own consistency protocol.

Follow-up：CAS failure 后为什么重新检查 stock？成功 reservation 后 payment 失败怎么办？这是 application protocol 的后续问题，不能由 CAS 一步解决。

### 16.4 Scenario: Publish a Configuration Snapshot

**Prompt:** One thread builds a new configuration and others read it. What must be synchronized?

区分 publication、immutability、reader ownership 与 old-version reclamation。

**Sample answer:**

> I would publish a fully initialized immutable snapshot through a mechanism that provides the required ordering. Readers must also keep their snapshot alive for the duration of use. An atomic pointer alone is not enough if an updater can free the old object while a reader still holds it. A mutex or a suitable shared-ownership abstraction is often simpler than custom reclamation.

Follow-up：原子替换 pointer 后，何时可以删除旧 object？为什么 read-only pointee 仍需要 safe publication？

### 16.5 Scenario: A Lock-free Rewrite Is Slower

**Prompt:** A CAS-based implementation is slower than the mutex version. Is that possible?

**Sample answer:**

> Yes. Lock-free is a progress property, not a performance guarantee. Under contention, retries and cache-coherence traffic can dominate, and memory reclamation may add further cost. I would compare retry rates, CPU time, throughput, and tail latency. Reducing shared state or batching updates may help more than removing the mutex.

Follow-up：这个算法调用 allocator 时会不会 block？某个 thread 能不能 starvation？这些问题影响对整条 API 的判断。

### 16.6 Interleaving Exercises

1. **Two-account transfer:** 若去掉全局 lock order，写出 A→B 与 B→A 的 deadlock schedule；解释原版本为何排除该 cycle。
2. **Condition variable:** 两个 consumers 被唤醒，但只有一个 item；说明 `if` 与 `while` 的差别。
3. **Semaphore:** Capacity=2，两个 producers 已取得 empty permits、尚未 enqueue，此时 empty/items counts 为什么不能直接代表 queue 的完整 state？
4. **Barrier reuse:** 第一个 worker 已进入下一轮写 buffer，最后一个 worker 还在读取上一轮；指出缺失的 phase coordination。
5. **Relaxed atomics:** 对 Section 10.7 的双零 outcome，解释为什么没有 data race，却仍可能不符合预期。
6. **Timeout:** Caller 等 future 超时后重试，原 task 随后完成；说明为什么 timeout 不等于 cancellation。

### 16.7 How to Structure an Answer

用以下顺序回答：

1. **Invariant:** 到底什么不能被破坏？
2. **Participants and scope:** Threads、processes、async tasks，还是 machines？
3. **Ownership and lifetime:** 谁可访问，何时销毁？
4. **Synchronization edges:** 哪个 lock、permit、predicate 或 publication relation 建立顺序？
5. **Progress and failure:** 谁唤醒谁，exceptions、cancellation、shutdown 怎么处理？
6. **Cost:** 是否有 contention、retry、copying 或复杂 reclamation？
7. **Validation:** 哪些 tests、traces 或 measurements 能检查假设？

能说明一个最小正确设计，再根据测量讨论优化，比直接回答“用 atomic 就快了”更可靠。
