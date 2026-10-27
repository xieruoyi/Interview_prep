# C++ Concurrency & Synchronization — Interview Exercises

**20 questions · 100 points · Suggested first pass: 4 hours**

英文 terminology，中文解释，练习时建议用英文口述 reasoning。本文根据提供的 quiz 整理，适用于 C++ SWE / systems / low-latency 面试准备；题目是模拟练习，不代表 HRT、Citadel、Jane Street 或其他公司的真实题库。

配套复习：[Synchronization](synchronization.md)、[Process & Thread](process-thread.md)、[Memory Management](memory.md)。

## How to Use

1. **Attempt first:** 前半部分只有题目；先写 invariant、counterexample / interleaving、修复方案与正确性解释。
2. **Review second:** 后半部分提供可折叠的参考答案、follow-ups 和每题 5 个评分点。开放设计题允许其他满足 contract 的方案。
3. **Code selectively:** 第二轮把 Q5、Q9、Q18、Q19 实现成完整 programs，编译、运行并补 edge-case tests。
4. **Revisit failures:** 记录错因和下一步验证，不把“看懂答案”等同于独立完成。

**C++ baseline:** 全篇以 C++20 为统一练习环境；Q5 的 queue class 只需要 C++17。题目中的错误 code 故意保留用于 code review，不能当作正确实现。所有 C++ blocks 都是 fragments 或 class definitions，通常没有 main，有些省略 headers / supporting types；text blocks 是 pseudocode / diagrams。文档整理时未进行 C++ compilation 或 sanitizer runs。

**Default assumptions:** 除题目另有说明，对象在并发访问前已完成构造并被安全发布，所有 users 退出后才析构。不要偷偷依赖“实际机器很快”“某个 thread 一定先跑”或 sleep 来证明正确性；需要的 memory-order、exception、fairness 和 lifetime assumptions 请明确写出来。

## Exercise Map

| Part | Focus | Questions | Suggested time |
| --- | --- | --- | --- |
| [Part A](#part-a--concurrency-correctness--locks) | Concurrency Correctness & Locks | Q1–Q4 | 40 min |
| [Part B](#part-b--condition-variables-semaphores--lifecycle) | Condition Variables, Semaphores & Lifecycle | Q5–Q8 | 60 min |
| [Part C](#part-c--atomic-cas--memory-ordering) | Atomic, CAS & Memory Ordering | Q9–Q12 | 45 min |
| [Part D](#part-d--lock-free-algorithms-aba--reclamation) | Lock-free Algorithms, ABA & Reclamation | Q13–Q15 | 35 min |
| [Part E](#part-e--performance-debugging--system-design) | Performance, Debugging & System Design | Q16–Q20 | 60 min |

4 hours 是首轮作答 budget，不包含把所有 fragments 写成完整 programs、benchmark 和复习的时间。

| Question | Difficulty | Focus |
| --- | --- | --- |
| [Q1. Thread-safe Doesn't Mean Atomic Transaction](#q1-thread-safe-doesnt-mean-atomic-transaction) | Medium | Race Condition · Linearizability · Mutex |
| [Q2. Reentrancy and Hidden Deadlock](#q2-reentrancy-and-hidden-deadlock) | Medium–Hard | Reentrancy · Recursive Mutex · Callbacks |
| [Q3. Fine-grained Locking and Bank Transfers](#q3-fine-grained-locking-and-bank-transfers) | Hard | Deadlock · Lock Ordering · Invariants |
| [Q4. Read-Write Lock Upgrade Race](#q4-read-write-lock-upgrade-race) | Hard | Shared Mutex · TOCTOU · Lazy Initialization |
| [Q5. Design a Closeable Bounded Blocking Queue](#q5-design-a-closeable-bounded-blocking-queue) | Hard · Coding | Condition Variable · Shutdown · RAII |
| [Q6. Lost Wakeup, Spurious Wakeup, and notify_one](#q6-lost-wakeup-spurious-wakeup-and-notify_one) | Hard | Controlled Interleavings · Predicate Protocol |
| [Q7. Semaphore Accounting and Deadlock](#q7-semaphore-accounting-and-deadlock) | Hard | Semaphore · Bounded Buffer · Deadlock |
| [Q8. Barrier, Latch, Join, and Future Lifetime](#q8-barrier-latch-join-and-future-lifetime) | Hard | Thread Lifecycle · Futures · Barrier |
| [Q9. Implement Reservation Using CAS](#q9-implement-reservation-using-cas) | Hard · Coding | CAS · Linearizability · Memory Order |
| [Q10. Memory Ordering Litmus Test](#q10-memory-ordering-litmus-test) | Hard | Relaxed · Acquire/Release · Seq_cst |
| [Q11. Safe Publication vs Atomic Flag](#q11-safe-publication-vs-atomic-flag) | Hard | Happens-before · Release/Acquire · Lifetime |
| [Q12. Memory Ordering for a Completion Counter](#q12-memory-ordering-for-a-completion-counter) | Very Hard | RMW · Acq_rel · Happens-before Chain |
| [Q13. Analyze a Lock-free Stack](#q13-analyze-a-lock-free-stack) | Very Hard | ABA · CAS · Data Structure Invariants |
| [Q14. Atomic Pointer vs Atomic Shared Ownership](#q14-atomic-pointer-vs-atomic-shared-ownership) | Hard | Object Lifetime · Ownership · Reclamation |
| [Q15. Lock-free Progress Guarantees](#q15-lock-free-progress-guarantees) | Hard | Obstruction-free · Lock-free · Wait-free |
| [Q16. Why Is a 32-thread Counter Slower?](#q16-why-is-a-32-thread-counter-slower) | Hard | False Sharing · Cache Coherence · Benchmarking |
| [Q17. Debug a Production Hang](#q17-debug-a-production-hang) | Very Hard · Debugging | Wait-for Graph · GDB · Thread Pool |
| [Q18. Deterministically Test a Concurrency Bug](#q18-deterministically-test-a-concurrency-bug) | Hard · Testing | Race Reproduction · Barriers · Sanitizers |
| [Q19. Design a High-Performance Configuration Service](#q19-design-a-high-performance-configuration-service) | Very Hard · System Design | RW Lock · Immutable Snapshot · Lifetime |
| [Q20. Full System Design — Concurrent Order Processing Engine](#q20-full-system-design--concurrent-order-processing-engine) | Very Hard · Final Interview | Correctness · Performance · Lifecycle · Design |

快速跳转：[Reference Answers](#reference-answers) · [Scorecard](#scorecard) · [Practice Plan](#practice-plan) · [Technical References](#technical-references)

## Part A — Concurrency Correctness & Locks

### Q1. Thread-safe Doesn't Mean Atomic Transaction

**Medium · 5 points**  
Race Condition · Linearizability · Mutex

一家商店有 10 件商品，两个线程可以同时处理用户订单。

以下两个函数本身都是 thread-safe 的：

```cpp
class Inventory {
    int stock = 10;
    std::mutex mtx;

public:
    int getStock() {
        std::lock_guard lock(mtx);
        return stock;
    }

    void decreaseStock() {
        std::lock_guard lock(mtx);
        --stock;
    }
};
```

订单处理：

```cpp
bool reserve(Inventory& inventory) {
    if (inventory.getStock() > 0) {
        inventory.decreaseStock();
        return true;
    }

    return false;
}
```

**Questions**

1. 这段代码有没有 C++ data race？
2. 能否出现超卖？写出两个线程的具体 interleaving。
3. 为什么 `getStock()` 和 `decreaseStock()` 都是 thread-safe，整体 `reserve()` 仍然不正确？
4. 修改代码，使 `reserve()` linearizable。
5. 如果还有发送邮件和付款操作，应该一起放到 mutex 里面吗？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 1](#answer-1)

### Q2. Reentrancy and Hidden Deadlock

**Medium–Hard · 5 points**  
Reentrancy · Recursive Mutex · Callbacks

你正在设计一个事件处理器：

```cpp
class EventManager {
    std::mutex mtx;
    int eventCount = 0;

public:
    void process(std::function<void()> callback) {
        std::lock_guard lock(mtx);

        ++eventCount;
        callback();
    }

    int getCount() {
        std::lock_guard lock(mtx);
        return eventCount;
    }
};
```

调用：

```text
EventManager manager;

manager.process([&] {
    std::cout << manager.getCount();
});
```

**Questions**

1. 这个程序可能发生什么？
2. 为什么同一个 thread 也可以 deadlock？
3. 将 `std::mutex` 改成 `std::recursive_mutex` 是否是一个完整解决方案？
4. 如何重新设计 callback 调用逻辑？
5. 如果 callback 需要读取本次事件对应的 state，怎么保证一致性？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 2](#answer-2)

### Q3. Fine-grained Locking and Bank Transfers

**Hard · 5 points**  
Deadlock · Lock Ordering · Invariants

有一个银行系统：

```cpp
struct Account {
    int id;
    long balance;
    std::mutex mtx;
};
```

转账函数：

```cpp
void transfer(Account& from, Account& to, long amount) {
    std::lock_guard lock1(from.mtx);
    std::lock_guard lock2(to.mtx);

    from.balance -= amount;
    to.balance += amount;
}
```

假设：

```text
Thread A: transfer(account1, account2, 100)
Thread B: transfer(account2, account1, 50)
```

**Questions**

1. 写出 Deadlock Interleaving。
2. 如何使用 `std::scoped_lock` 修复？
3. 如果 `from == to` 怎么办？
4. 如果要求同一账户不能透支，还需要修改什么？
5. 为什么即使每个 Account 都有自己的 mutex，读取所有账户余额的总和仍可能获得不一致 snapshot？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 3](#answer-3)

### Q4. Read-Write Lock Upgrade Race

**Hard · 5 points**  
Shared Mutex · TOCTOU · Lazy Initialization

你需要实现一个线程安全缓存：

```cpp
std::unordered_map<int, int> cache;
std::shared_mutex mtx;
```

要求：

- Cache hit 时允许多个线程并发读取。
- Cache miss 时计算并插入。
- 同一 key 不能被重复插入覆盖。
- `compute()` 可能耗时较长。

面试官让你审查：

```cpp
int get(int key) {
    std::shared_lock lock(mtx);

    if (cache.contains(key))
        return cache.at(key);

    lock.unlock();

    int value = compute(key);

    std::unique_lock writeLock(mtx);
    cache[key] = value;

    return value;
}
```

**Questions**

1. 这段代码有哪些 concurrency 问题？
2. 如何修复？
3. 可以直接在持有 shared lock 的同时 acquire exclusive lock 吗？
4. 如果要求对同一个 key 的 `compute()` 只能执行一次，如何进一步设计？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 4](#answer-4)

## Part B — Condition Variables, Semaphores & Lifecycle

### Q5. Design a Closeable Bounded Blocking Queue

**Hard · Coding · 5 points**  
Condition Variable · Shutdown · RAII

设计一个线程安全的 `BoundedQueue<T>`，要求：

- 最多存储 `capacity` 个元素。
- `push()` 在队列满时阻塞。
- `pop()` 在队列空时阻塞。
- `close()` 后拒绝新的 push。
- `close()` 后，consumer 仍然能消费已经入队的元素。
- 已关闭且清空后，`pop()` 返回结束状态。
- 多个 producer 和 consumer 可以同时运行。
- 不能因为 close 导致 waiter 永远卡住。

面试官还会问：

1. 为什么 `wait()` 必须释放 mutex？
2. 为什么要使用 predicate，而不是 `if`？
3. `close()` 应该使用 `notify_one` 还是 `notify_all`？
4. 如果 `push()` 等待过程中 queue 被关闭怎么办？
5. 如果 `T` 的 move constructor 可能抛异常怎么办？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 5](#answer-5)

### Q6. Lost Wakeup, Spurious Wakeup, and notify_one

**Hard · 5 points**  
Controlled Interleavings · Predicate Protocol

有一个 worker 等待任务：

```cpp
std::queue<int> tasks;
std::mutex mtx;
std::condition_variable cv;

void worker() {
    while (true) {
        std::unique_lock lock(mtx);

        if (tasks.empty()) {
            cv.wait(lock);
        }

        int task = tasks.front();
        tasks.pop();

        lock.unlock();
        process(task);
    }
}
```

Producer：

```cpp
void submit(int task) {
    {
        std::lock_guard lock(mtx);
        tasks.push(task);
    }

    cv.notify_one();
}
```

**Questions**

1. 为什么 Worker 这里的 `if` 不正确？
2. `cv.wait(lock)` 的内部流程是什么？
3. Producer 在 Worker 开始等待之前发送通知，会不会丢失任务？
4. 如果有三个消费者等待，Producer 只添加一个 item，应该使用 `notify_one` 还是 `notify_all`？
5. 如果你想实现 shutdown，应该增加什么 state？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 6](#answer-6)

### Q7. Semaphore Accounting and Deadlock

**Hard · 5 points**  
Semaphore · Bounded Buffer · Deadlock

假设队列容量为 2：

```text
emptySlots = semaphore(2)
itemsReady = semaphore(0)
queueMutex = mutex()
```

某个工程师写出了下面的 Producer：

```text
Producer:
    lock(queueMutex)
    acquire(emptySlots)
    queue.push(item)
    unlock(queueMutex)
    release(itemsReady)
```

Consumer：

```text
Consumer:
    acquire(itemsReady)
    lock(queueMutex)
    item = queue.pop()
    unlock(queueMutex)
    release(emptySlots)
```

**Questions**

1. Producer 的顺序有什么问题？
2. 写出产生 deadlock 的具体场景。
3. 正确的 acquire 顺序是什么？
4. 两个 Producers 都取得 `emptySlots` permits，但还未 enqueue，此时两个 semaphores 的 count 之和一定等于 capacity 吗？
5. 如果 Producer 在 acquire permit 后抛异常，应该怎么办？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 7](#answer-7)

### Q8. Barrier, Latch, Join, and Future Lifetime

**Hard · 5 points**  
Thread Lifecycle · Futures · Barrier

一个图像处理系统有四个 worker，每轮操作为：

1. Worker 计算自己的部分图像。
2. 所有 Worker 读取四块 partial results。
3. 所有 Worker 开始下一轮，覆写旧 buffer。

代码示意：

```cpp
std::array<Image, 4> partial;
std::barrier phaseBarrier(4);

void worker(int id) {
    for (int round = 0; round < 100; ++round) {
        partial[id] = calculate(id, round);

        phaseBarrier.arrive_and_wait();

        combine(partial);
    }
}
```

**Questions**

1. 一个 Barrier 够吗？为什么？
2. 如果某个 Worker 提前异常退出，会发生什么？
3. `std::latch` 和 `std::barrier` 哪个适合重复同步？
4. `join()` 与 `future.get()` 分别提供什么？
5. 如果某个异步 Task 捕获局部变量引用，如何避免 Use-after-free？

**Assumptions:** `combine(partial)` 对 partial 只读，并且每个 worker 的输出独立；否则需要另行同步 combine 的 output。

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 8](#answer-8)

## Part C — Atomic, CAS & Memory Ordering

### Q9. Implement Reservation Using CAS

**Hard · Coding · 5 points**  
CAS · Linearizability · Memory Order

当前库存为 100：

```cpp
std::atomic<int> stock{100};
```

要求实现：

```cpp
bool tryReserve(int quantity);
```

Contract：

- `quantity` 必须为正数。
- 只有库存足够时才能成功。
- 成功时原子扣除 quantity。
- 失败的调用本身不扣库存；其他 concurrent callers 仍可以改变库存。
- 多个线程同时调用不能超卖。
- 不允许使用 mutex。

**Questions**

1. 请实现完整 CAS Loop。
2. 为什么不能使用 atomic `load()` + `store()`？
3. 为什么每次 CAS 失败后需要重新验证库存？
4. 成功操作的 Linearization Point 在哪里？
5. 能否使用 `memory_order_relaxed`？
6. 如果两个线程持续竞争，某个线程是否可能永远 CAS 失败？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 9](#answer-9)

### Q10. Memory Ordering Litmus Test

**Hard · 5 points**  
Relaxed · Acquire/Release · Seq_cst

初始状态：

```cpp
std::atomic<int> x{0};
std::atomic<int> y{0};

int r1 = -1;
int r2 = -1;
```

两个线程分别执行：

```cpp
// Thread A
x.store(1, std::memory_order_relaxed);
r1 = y.load(std::memory_order_relaxed);
```

```cpp
// Thread B
y.store(1, std::memory_order_relaxed);
r2 = x.load(std::memory_order_relaxed);
```

假设 `r1` 和 `r2` 只由 Main 在两个线程 join 后读取，且没有其他访问。

**Questions**

1. `r1 = 0, r2 = 0` 是否允许？
2. 这个程序有 C++ data race 吗？
3. 如果四个操作都改为 `seq_cst`，双零是否仍然允许？
4. 如果 stores 改为 `release`，loads 改为 `acquire` 呢？
5. 为什么 acquire 并不意味着一定读取最新写入？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 10](#answer-10)

### Q11. Safe Publication vs Atomic Flag

**Hard · 5 points**  
Happens-before · Release/Acquire · Lifetime

审查下面的代码：

```cpp
struct Config {
    int port;
    int timeout;
};

Config config;
std::atomic<bool> initialized{false};

void init() {
    config.port = 8080;
    config.timeout = 100;

    initialized.store(
        true,
        std::memory_order_relaxed
    );
}

Config getConfig() {
    while (!initialized.load(
        std::memory_order_relaxed
    )) {}

    return config;
}
```

假设初始化仅执行一次，之后不再修改 config。

**Questions**

1. 为什么 initialized 是 atomic 仍然不够？
2. 是否存在 data race？
3. 如何修复 Memory Ordering？
4. 写出完整的 happens-before 推理。
5. 如果后来想动态更新 Config，继续反复切换 initialized 为 false/true 是否安全？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 11](#answer-11)

### Q12. Memory Ordering for a Completion Counter

**Very Hard · 5 points**  
RMW · Acq_rel · Happens-before Chain

假设有四个 Worker，每个线程只写自己的结果。

```cpp
int results[4] = {};
std::atomic<int> completed{0};
```

每个 Worker：

```cpp
void worker(int id) {
    results[id] = compute(id);

    int old = completed.fetch_add(
        1,
        std::memory_order_relaxed
    );

    if (old == 3) {
        int total = 0;

        for (int i = 0; i < 4; ++i)
            total += results[i];

        publish(total);
    }
}
```

**Questions**

1. 这里为什么有 Memory Ordering 问题？
2. 为什么使用 `relaxed` 计数虽然不会丢失 increment，却不够证明读取 results 安全？
3. 把 `fetch_add` 改成 `acq_rel` 后，为什么最后一个 Worker 可以读取之前的结果？
4. 有没有更简单的实现，例如用 `join()`？
5. 如果 Worker 可能抛 exception，完成计数协议还需要注意什么？

**Assumptions:** 每个 id 恰好由一个 worker 使用，每个 slot 只写一次；counter 不在本轮中 reset，没有其他 writers，sum 不 overflow，`publish` 自己满足其输出同步要求。

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 12](#answer-12)

## Part D — Lock-free Algorithms, ABA & Reclamation

### Q13. Analyze a Lock-free Stack

**Very Hard · 5 points**  
ABA · CAS · Data Structure Invariants

你正在审查一个 Treiber-style Lock-free Stack 的简化代码：

```cpp
struct Node {
    int value;
    Node* next;
};

std::atomic<Node*> head{nullptr};

Node* pop() {
    Node* oldHead = head.load();

    while (oldHead != nullptr) {
        Node* next = oldHead->next;

        if (head.compare_exchange_weak(
                oldHead, next
            )) {
            return oldHead;
        }
    }

    return nullptr;
}
```

这段代码使用 atomic head 和 CAS。

**Questions**

1. 算法想实现的 Linearization Point 是什么？
2. 为什么 `head` 是 atomic 不代表整个实现安全？
3. 请构造一个 ABA Interleaving，使得 CAS 成功却破坏 Stack。
4. 即使不存在 ABA，还可能发生什么 memory safety 问题？
5. 如何使用 Tagged Pointer、Hazard Pointer 或 Epoch-Based Reclamation 修复相关问题？
6. 为什么修复 lifetime 后仍需要单独分析 ABA？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 13](#answer-13)

### Q14. Atomic Pointer vs Atomic Shared Ownership

**Hard · 5 points**  
Object Lifetime · Ownership · Reclamation

某个系统的 Configuration 每隔几分钟就更新。

Reader 很多，Writer 很少。

最初设计：

```cpp
std::atomic<Config*> current;
```

Reader：

```cpp
void read() {
    Config* p = current.load();

    if (p)
        use(*p);
}
```

Writer：

```cpp
void update() {
    Config* next = new Config();

    initialize(*next);

    Config* old = current.exchange(next);

    delete old;
}
```

**Questions**

1. 为什么这段代码可能发生 Use-after-free？
2. 改成 acquire/release 是否能自动解决？
3. `std::atomic<std::shared_ptr<const Config>>` 可以提供什么？
4. Shared Pointer 解决了什么，没有解决什么？
5. 什么时候考虑 RCU 或 Epoch-Based Reclamation？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 14](#answer-14)

### Q15. Lock-free Progress Guarantees

**Hard · 5 points**  
Obstruction-free · Lock-free · Wait-free

假设一个算法使用 CAS Loop 修改共享状态。

在大量线程竞争时，某个线程 A 连续失败 100 万次，但其他线程不断 CAS 成功。

**Questions**

1. 这是否违反 Lock-free？
2. 这是否违反 Wait-free？
3. Lock-free 和 Obstruction-free 的主要区别是什么？
4. 如果 CAS 本身是 lock-free，但代码中还调用 `malloc()`，能否宣称整个 API lock-free？
5. Lock-free 一定比 Mutex 快吗？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 15](#answer-15)

## Part E — Performance, Debugging & System Design

### Q16. Why Is a 32-thread Counter Slower?

**Hard · 5 points**  
False Sharing · Cache Coherence · Benchmarking

你有一个 32-core Linux server。

最初代码：

```cpp
std::atomic<uint64_t> total{0};

void worker() {
    for (int i = 0; i < 1'000'000; ++i) {
        total.fetch_add(
            1,
            std::memory_order_relaxed
        );
    }
}
```

单线程表现不错，但增加到 32 个线程后，throughput 没有线性提升。

**Questions**

1. 为什么 relaxed atomic 仍然可能很慢？
2. 什么是 Cache-line bouncing？
3. 如何使用 Thread-local Aggregation 优化？
4. 什么是 False Sharing？和多个线程更新同一个 atomic 有何区别？
5. Benchmark 应该记录哪些指标？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 16](#answer-16)

### Q17. Debug a Production Hang

**Very Hard · Debugging · 5 points**  
Wait-for Graph · GDB · Thread Pool

假设你在一家低延迟交易公司工作。

某个 C++ Service 平时运行正常，但在高负载时偶尔停止处理请求。

Monitoring 显示：

```text
CPU utilization: 8%
Request queue depth: 15,000
Completed requests/sec: 0
Process is still alive
```

GDB 中观察到：

```text
Thread 1:
  std::future<void>::wait()
  RequestHandler::handle()

Thread 2:
  std::future<void>::wait()
  RequestHandler::handle()

Thread 3:
  std::future<void>::wait()
  RequestHandler::handle()

Thread 4:
  std::future<void>::wait()
  RequestHandler::handle()
```

Thread Pool 只有 4 个 Worker。

每个 RequestHandler 都会往同一个 Pool 提交一个子任务，然后同步等待这个子任务执行完成。

**Questions**

1. 你怀疑是什么问题？
2. 为什么 CPU utilization 很低？
3. 有没有必要涉及 mutex 才能 deadlock？
4. 如何通过 Thread Stacks 和 Queue State 验证？
5. 有哪些修复方案？
6. 为什么简单把 Thread Pool 从 4 扩到 8 不是根本解决方案？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 17](#answer-17)

### Q18. Deterministically Test a Concurrency Bug

**Hard · Testing · 5 points**  
Race Reproduction · Barriers · Sanitizers

你怀疑某个程序存在下面的竞态：

```cpp
if (!cache.contains(key)) {
    cache.insert(key, value);
}
```

每个操作都单独受锁保护，但整个 check-then-insert 并不是 atomic。

这个 Bug 在 production 中偶尔出现，但测试里很难复现。

**Questions**

1. 为什么添加 `sleep(100ms)` 不可靠？
2. 如何设计 Deterministic Interleaving？
3. 测试需要检查什么 Invariant？
4. ThreadSanitizer 能否保证找到问题？
5. Controlled Schedule 测试通过，能否证明所有执行顺序都正确？
6. 如何验证 Linearizability？

**Contract clarification:** 单独看到 contains + insert 还不能证明 bug。假设 wrapper 在先前 contains=false 后，无论底层 insert 是否实际成功都返回 success，或执行一次不应重复的 side effect。需要复现的是这个 composite contract 违规，而不是假设 unordered_map 一定容纳了 duplicate keys。

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 18](#answer-18)

### Q19. Design a High-Performance Configuration Service

**Very Hard · System Design · 5 points**  
RW Lock · Immutable Snapshot · Lifetime

设计一个线程安全的 Configuration Service。

要求：

- 64 个 Reader Threads。
- 1 个 Writer Thread。
- 每秒有大量读取。
- 配置每分钟更新一次。
- Reader 必须看到完整的一版配置，不能读到部分更新。
- 旧配置不能在仍有 Reader 使用时被销毁。
- Reader 的 latency 要尽可能低。
- 更新失败时，已有 Reader 仍可继续使用旧版本。

**Questions**

1. 先给出最简单正确的方案。
2. 如果 Read Lock 成为性能瓶颈，你会如何改进？
3. 如何保证 Safe Publication？
4. 如何保证旧 Snapshot 的 Lifetime？
5. 为什么 Atomic Raw Pointer 不够？
6. 如果换成 RCU，会带来哪些新的复杂度？
7. 如何 Benchmark 两种方案？

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 19](#answer-19)

### Q20. Full System Design — Concurrent Order Processing Engine

**Very Hard · Final Interview · 5 points**  
Correctness · Performance · Lifecycle · Design

最后这道是综合题。

假设你要设计一个简化版的交易订单处理引擎。

系统收到以下类型的请求：

```cpp
struct AddOrder {
    int orderId;
    int symbolId;
    int quantity;
    int price;
};

struct CancelOrder {
    int orderId;
};
```

系统要求：

- 多个网络线程并发接收 requests。
- 同一个订单不能重复创建。
- 订单不能被重复取消并产生错误状态。
- 不同 Symbol 的订单可以尽量并发处理。
- 同一个 Symbol 的状态更新必须保持一致。
- 需要支持 Graceful Shutdown。
- 不允许丢失已经确认完成的请求。
- 对性能要求较高，但正确性优先。

面试官会分阶段追加要求。

#### Stage A — 基础设计

1. 你选择 Global Mutex、Per-symbol Mutex、Lock-free Queue，还是 Single-owner Architecture？
2. 为什么？
3. 谁拥有 Order Book？
4. 如何避免 Duplicate Orders？
5. `submit()` 返回 success 表示消息已入队，还是订单已经处理完成？

#### Stage B — Concurrency

6. 同一个 Symbol 的 Add 和 Cancel 同时到达怎么办？
7. 两个请求操作不同 Symbols，能否并行？
8. 如果某个请求需要跨两个 Symbols 原子更新怎么办？
9. 如果 Consumer 线程在 Queue 为空时等待，如何实现通知？

#### Stage C — Failure & Shutdown

10. Shutdown 时还有 10,000 个 pending requests，怎么办？
11. 某个 Worker 处理订单时发生异常，怎么办？
12. 如何保证 Queue、Order Book 和 Worker 对象不会过早销毁？
13. 如果请求已经排队，但调用方 timeout，能否直接认为取消成功？

#### Stage D — Performance

14. 如果 Lock Contention 变高，优先做什么？
15. 如果 CAS-based Queue 比 mutex Queue 还慢，怎么排查？
16. 如何衡量系统是否满足性能要求？

**Clarify before designing:** `orderId` 是全局唯一还是仅在 symbol 内唯一？“不丢失已确认完成的请求”是否包含 process / machine crash？请先声明假设，再给 design；无需实现完整 matching engine。

<details>
<summary>Your attempt — 展开填写</summary>

- **Invariant / contract:** 
- **Counterexample / interleaving:** 
- **Fix / design:** 
- **Why it is correct:** 
- **Lifetime, exceptions, progress, tests:** 

</details>

[完成后查看 Answer 20](#answer-20)

## Reference Answers

以下是参考解法，不是唯一的标准实现。先判断自己的方案是否满足 assumptions / contract，再比较实现细节。

**Scoring:** 每题的 5 个 checkpoint 各计 1 分，总分 100。只说出关键词不算满足；应能通过 code、interleaving、happens-before proof 或明确的 design argument 解释。部分满足可记 0.5 分。分数只用于定位复习重点，不用于预测具体公司的录取结果。

### Answer 1

[返回 Q1](#q1-thread-safe-doesnt-mean-atomic-transaction)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

核心：单个 API thread-safe，不代表它们组合后的 check-then-act 是 atomic。

假设库存只剩 1：

| Step | Thread A            | Thread B             |
| ---- | ------------------- | -------------------- |
| 1    | getStock() → 1      |                      |
| 2    |                     | getStock() → 1       |
| 3    | decreaseStock() → 0 |                      |
| 4    |                     | decreaseStock() → -1 |
| 5    | return true         | return true          |

这里对 `stock` 的访问都受 mutex 保护，因此假设没有其他未同步访问，不存在 C++ data race。

但库存仍然可能变成 -1，属于逻辑 race condition。

正确设计是把检查与修改放进同一个 critical section：

```cpp
class Inventory {
    int stock = 10;
    std::mutex mtx;

public:
    bool tryReserve() {
        std::lock_guard lock(mtx);

        if (stock <= 0)
            return false;

        --stock;
        return true;
    }
};
```

成功操作的 linearization point 可以视为 `--stock`。

失败操作的 linearization point 可以视为锁内观察到 `stock <= 0` 的判断。

至于付款与发送邮件，不应该简单地在持锁期间执行网络调用，因为会延长 critical section，并且外部操作无法由普通 mutex 自动 rollback。

更合理的业务设计可能涉及 reservation state、数据库事务、幂等支付、补偿操作和失败恢复。

**Follow-up:** 如果 `stock` 改成 `std::atomic<int>`，然后仍然使用 atomic `load()` + `store()`，能解决吗？

不能。需要 mutex 或基于 CAS 的原子条件更新。

**Self-check — 1 point each**

- [ ] 区分 data race 与 logical race。
- [ ] 给出 stock=1 的超卖 interleaving。
- [ ] 把 check-and-decrement 放入同一 critical section。
- [ ] 指出成功与失败的 linearization points。
- [ ] 解释 external side effects 和 atomic load/store 的限制。

</details>

### Answer 2

[返回 Q2](#q2-reentrancy-and-hidden-deadlock)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

主要问题：持有 mutex 时执行外部 callback，callback 再次进入对象并尝试获取同一把 mutex。

调用关系：

```text
process()
  acquire mtx
  increment eventCount
  callback()
    getCount()
      acquire mtx again
```

普通 `std::mutex` 不支持同线程重复持有；再次 lock 已持有的 mutex 属于未定义行为，常见表现是 self-deadlock。

换成 `recursive_mutex` 可以允许同线程重复 acquire，但它不是完整的 reentrancy 保证，因为 callback 可能观察到尚未完成的内部状态更新。

更合理的设计：

```cpp
void process(std::function<void(int)> callback) {
    int snapshot;

    {
        std::lock_guard lock(mtx);
        snapshot = ++eventCount;
    }

    callback(snapshot);
}
```

这个方案：

- 锁内完成状态更新。
- 捕获需要传递给 callback 的值。
- 释放锁。
- 再调用 callback。

如果 callback 需要多个字段的一致 snapshot，就要在锁内一并复制，或者通过 immutable snapshot 提供稳定视图。

Callback 抛异常时，eventCount 的更新已经提交，不会自动 rollback。这个实现也不保证多个 callbacks 按 count 顺序执行；若 contract 要求严格 callback ordering，需要额外 serialization。

**Follow-up:** 释放锁再 callback 有没有新的问题？

有。如果 callback 需要保证“执行时状态仍是刚刚更新的版本”，解锁后其他线程可能修改状态。需要明确 API contract，可能使用 version、snapshot、serialization 或 single-owner event loop。

**Self-check — 1 point each**

- [ ] 定位 callback reentry / non-recursive mutex 问题。
- [ ] 准确说明 undefined behavior 与常见 self-deadlock。
- [ ] 解释 recursive_mutex 不保证 invariant / reentrancy。
- [ ] 给出 unlock-before-callback + snapshot。
- [ ] 说明 callback ordering、exception 与 snapshot contract。

</details>

### Answer 3

[返回 Q3](#q3-fine-grained-locking-and-bank-transfers)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

Deadlock Interleaving：

```text
A locks account1
B locks account2

A waits for account2
B waits for account1
```

出现 circular wait。

一种修复：

```cpp
#include <limits>

bool transfer(Account& from, Account& to, long amount) {
    if (amount < 0) return false;
    if (&from == &to) return true;

    std::scoped_lock lock(from.mtx, to.mtx);

    if (from.balance < amount ||
        to.balance > std::numeric_limits<long>::max() - amount)
        return false;

    from.balance -= amount;
    to.balance += amount;

    return true;
}
```

前提：账户余额保持 nonnegative，所有余额访问遵守相同 locking protocol。代码检查 receiver 的加法 overflow；`long` 的宽度取决于平台，不能默认是 64 bits。Self-transfer 定义为合法 nonnegative amount 的 no-op，不检查资金是否充足；若业务要求不同，需修改 contract。

`std::scoped_lock` 可以安全地协调获取多把 mutex，避免这种多锁获取方式造成的死锁。

跨账户总额统计还需要一致的 snapshot 协议。逐个账户单独锁住、读取、释放，可能读到一部分转账前状态和另一部分转账后状态。

可以通过固定顺序锁住全部相关账户、版本化 snapshot 或其他事务性机制解决，但每种方案都有性能成本。

**Follow-up:** 改成 atomic balance 就能实现 atomic transfer 吗？

不一定。两个独立的 atomic 更新不会自动组成一个跨账户事务。

**Self-check — 1 point each**

- [ ] 构造双向 transfer 的 wait cycle。
- [ ] 正确使用多锁获取并处理 self-transfer。
- [ ] 检查 amount、余额和 overflow。
- [ ] 解释 consistent aggregate snapshot。
- [ ] 区分 individual atomic balance 与 atomic transfer。

</details>

### Answer 4

[返回 Q4](#q4-read-write-lock-upgrade-race)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

主要逻辑问题是：

多个线程可能同时 miss，同步计算后直接覆盖彼此结果。

修复：

```cpp
int get(int key) {
    {
        std::shared_lock lock(mtx);
        auto it = cache.find(key);

        if (it != cache.end())
            return it->second;
    }

    int value = compute(key);

    std::unique_lock lock(mtx);

    auto [it, inserted] =
        cache.try_emplace(key, value);

    return it->second;
}
```

在 write lock 内重新检查，避免旧的 cache-miss 结论被直接使用。

这个实现假设 `compute()` 可以在锁外安全执行，也允许多个线程重复计算同一个 key。

如果要求 compute 只执行一次，可以使用：

- Per-key in-flight state。
- Promise/future 共享结果。
- Per-key locking。
- Single-flight 模式。

同时必须处理 compute exception、失败后是否允许 retry，以及等待线程如何被唤醒。

不能简单地在持有 `std::shared_mutex` 的 shared ownership 时再次 acquire exclusive ownership；C++ 标准的 `std::shared_mutex` 没有这样的原子 upgrade API。

**Follow-up:** 为什么 read-heavy workload 不一定适合 shared_mutex？

因为 shared lock 的 reader count/metadata 仍有同步和 cache-coherence 成本，且还要考虑 reader/writer fairness。

**Self-check — 1 point each**

- [ ] 指出 unlock 后旧 miss 结论可失效。
- [ ] 写出 exclusive lock 下 recheck / try_emplace。
- [ ] 说明标准 shared_mutex 没有直接 upgrade。
- [ ] 设计 single-flight 的 in-flight result。
- [ ] 处理 compute exception、retry 和 fairness / performance。

</details>

### Answer 5

[返回 Q5](#q5-design-a-closeable-bounded-blocking-queue)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

下面是 C++17 class fragment。显式要求 `T` nothrow-move-constructible / nothrow-destructible；调用方负责在 queue 析构前停止所有 users 并 join。使用默认 `std::queue<T>` / `std::deque<T>`；allocation failure 仍可能发生。

```cpp
#include <cstddef>
#include <type_traits>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>
#include <stdexcept>
#include <utility>

template<typename T>
class BoundedQueue {
    static_assert(std::is_nothrow_move_constructible_v<T>);
    static_assert(std::is_nothrow_destructible_v<T>);

    std::queue<T> queue_;
    std::size_t capacity_;
    bool closed_ = false;

    std::mutex mtx_;
    std::condition_variable notEmpty_;
    std::condition_variable notFull_;

public:
    explicit BoundedQueue(std::size_t capacity)
        : capacity_(capacity) {
        if (capacity == 0)
            throw std::invalid_argument("capacity");
    }

    bool push(T item) {
        std::unique_lock lock(mtx_);

        notFull_.wait(lock, [&] {
            return closed_ ||
                   queue_.size() < capacity_;
        });

        if (closed_)
            return false;

        try {
            queue_.push(std::move(item));
        } catch (...) {
            // The failed insertion leaves capacity available.
            // Pass the wakeup to another blocked producer.
            lock.unlock();
            notFull_.notify_one();
            throw;
        }

        lock.unlock();
        notEmpty_.notify_one();

        return true;
    }

    std::optional<T> pop() {
        std::unique_lock lock(mtx_);

        notEmpty_.wait(lock, [&] {
            return closed_ || !queue_.empty();
        });

        if (queue_.empty())
            return std::nullopt;

        T item = std::move(queue_.front());
        queue_.pop();

        lock.unlock();
        notFull_.notify_one();

        return item;
    }

    void close() {
        {
            std::lock_guard lock(mtx_);
            closed_ = true;
        }

        notEmpty_.notify_all();
        notFull_.notify_all();
    }
};
```

关键设计：

为什么使用两种 Condition Variables？

`notEmpty_` 负责唤醒等待 item 的 consumers。

`notFull_` 负责唤醒等待空间的 producers。

它们共享同一把 mutex，但等待不同 predicate。这样通常比所有线程共用一个 condition 然后频繁 `notify_all()` 更高效。

为什么 wait 用 predicate？

因为可能发生 spurious wakeup，或另一个线程先一步消耗了资源。

为什么 close 使用 notify_all？

因为关闭后所有等待线程都需要重新检查终止条件，而不是只有一个线程需要醒来。

为什么不能简单地把 `T` 的移动异常忽略？

如果 `pop()` 在从 `queue_.front()` 移动数据时抛异常，我们需要确保队列仍处于定义良好的状态，并决定是否可以 retry。生产级实现需要明确异常保证，可以限制 `T` 的类型要求，或者调整存储与返回策略。

**Follow-up:** `close()` 调用结束后，能否立即 delete Queue？

不能。其他线程可能刚被唤醒，还在使用 Queue 的 mutex 和 condition variables。必须先通过 join 或其他生命周期协议确保所有使用者退出。

**Exception and ownership contract:** Nothrow move 不代表 container allocation 不会失败。如果 producer 被唤醒后 insertion 抛异常，空位仍然存在；在异常路径通知另一个 producer，可避免其他 producers 因失去这次通知而继续睡眠。这个 reasoning 依赖上述 container/type 的 exception guarantee。

`push(T item)` 的 by-value parameter 可能在进入函数前已经从 caller 的 object 移走数据；返回 false 不代表 caller 原来的 object 未被移动。API 文档应明确这一点。

**Linearization points:** 成功 push / pop 在锁内完成 enqueue / dequeue；close 在锁内设置 closed_；被拒绝的 push 和 exhausted pop 在锁内观察终止条件。所有返回的 reference / value 都不能暗中依赖已释放的 queue storage。

**Self-check — 1 point each**

- [ ] 定义 bounded / closed / draining invariants。
- [ ] 写出两个正确 wait predicates。
- [ ] 正确实现 close 并唤醒所有 waiters。
- [ ] 处理 move、allocation exception 和 ownership contract。
- [ ] 说明 linearization 与 join-before-destruction。

</details>

### Answer 6

[返回 Q6](#q6-lost-wakeup-spurious-wakeup-and-notify_one)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

主要 bug 是使用 `if` 而不是 `while`。

```cpp
cv.wait(lock, [&] {
    return !tasks.empty();
});
```

Predicate overload 在逻辑上等价于反复检查条件并等待。

由于 `wait()` 返回前会重新获取 mutex，多个 consumers 被唤醒后，仍可能只有其中一个成功取走 item。

另外，某些 API 允许 spurious wakeup。

正确的基础模式：

```cpp
std::unique_lock lock(mtx);

cv.wait(lock, [&] {
    return !tasks.empty();
});

int task = tasks.front();
tasks.pop();
```

这个 Producer 和 Consumer 如果始终通过同一把 mutex 检查、修改队列，就不会因为 Producer 在 Consumer 检查之前提交任务而丢失任务：Consumer 会直接看到队列非空。

即使通知发生时没有 waiter，持久保存工作事实的是 `tasks`，不是 notification。

对于只新增一个 item、且所有 consumers 等待同一 predicate 的情况，`notify_one` 通常合适。

Shutdown 可以引入：

```cpp
bool closed = false;
```

并把等待条件改成：

```cpp
cv.wait(lock, [&] {
    return closed || !tasks.empty();
});
```

然后根据队列是否还有任务决定 drain 或退出。

**Follow-up:** 如果多个不同 predicates 共用同一 Condition Variable，为什么 notify_one 可能出问题？

因为它可能唤醒一个不满足当前 predicate 的 waiter，而真正能继续执行的 waiter 仍然在睡眠。

`wait(lock)` 把 unlock-and-block 作为一个协调步骤，醒来后重新 lock 才返回。Notifier 在同一 mutex 下修改 predicate，因此不会插进“检查为空”和“开始等待”之间制造无保护窗口。Notify 本身没有累计 permits；真正保留工作的是 queue state。参见 [C++ draft: condition_variable](https://eel.is/c++draft/thread.condition.condvar)。

**Self-check — 1 point each**

- [ ] 解释 spurious wakeup 与竞争消耗。
- [ ] 正确说明 unlock-and-block / relock。
- [ ] 区分 persistent predicate 与 notification。
- [ ] 说明 notify_one / notify_all 的适用条件。
- [ ] 给出 shutdown predicate 和多 predicate 风险。

</details>

### Answer 7

[返回 Q7](#q7-semaphore-accounting-and-deadlock)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

Producer 的问题是：

它持有 queueMutex，却可能无限等待 emptySlots。

假设 Queue 已满：

```text
Producer locks queueMutex
Producer waits for emptySlots

Consumer obtains itemsReady
Consumer waits for queueMutex
```

现在 Consumer 不能 pop，因此无法释放 emptySlots。

Producer 也无法继续，形成 deadlock。

正确顺序：

```text
Producer:
    acquire(emptySlots)
    lock(queueMutex)
    enqueue
    unlock(queueMutex)
    release(itemsReady)
```

Consumer 先 `acquire(itemsReady)`，再获取 queueMutex 并 dequeue。

关于 permit accounting：

假设初始 capacity=2，Queue 为空。

两个 Producers 都 acquire emptySlots，但尚未 enqueue：

```text
emptySlots = 0
itemsReady = 0
queue size = 0
```

因此：

```text
emptySlots + itemsReady = 0 != capacity (2)
```

因为两个空位已经被 Producers 预留，尚处于 in-flight 状态。

稳定点可以使用简化关系，真正的并发执行过程需要考虑已经取得但尚未提交的 permits。

如果获取 permit 后失败，必须根据实际是否已经 enqueue 或完成 ownership transfer，正确恢复 permit 或将操作完成。

不能随便在异常时无条件 release，因为可能导致重复计算容量许可。

**Self-check — 1 point each**

- [ ] 写出满队列时的 deadlock cycle。
- [ ] permit acquire 放在 queue mutex 之前。
- [ ] 给出 in-flight permits 的 accounting 例子。
- [ ] 按实际 ownership transfer 处理异常。
- [ ] 区分 semaphore permits 与 queue mutual exclusion。

</details>

### Answer 8

[返回 Q8](#q8-barrier-latch-join-and-future-lifetime)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

一个 Barrier 不够。

第一个 Barrier 只能保证四个 Worker 全部完成本轮 partial 写入。

但是执行完 `combine()` 的某个 Worker 可能提前开始下一轮，覆写其他线程还在读取的 partial。

正确阶段协议：

```cpp
std::barrier phaseBarrier(4);

void worker(int id) {
    for (int round = 0; round < 100; ++round) {
        partial[id] = calculate(id, round);

        phaseBarrier.arrive_and_wait();

        combine(partial);

        phaseBarrier.arrive_and_wait();
    }
}
```

第一个 Barrier 保证：

> All data is ready.

第二个 Barrier 保证：

> No worker is still reading this generation.

`std::barrier` 支持重复使用，而 `std::latch` 是一次性的倒计数同步原语。

`join()` 等待指定 thread 终止。

`future.get()` 等待异步结果 ready，并可以取得结果或重抛存储的异常；它并不普遍等同于等待某个具体线程终止。

如果异步任务捕获局部变量引用，则必须确保被引用对象在任务使用期间持续存活，例如通过明确 join、拥有型捕获或共享所有权。

如果 exception 逃出 `std::thread` 的 entry function，通常直接调用 `std::terminate`，不能描述成仅当前 worker 安静退出。如果 exception 在 worker 边界被捕获，但它没有再履行 barrier participation protocol，其他 workers 可能一直等待。

需要整体 error/cancellation protocol；不能在 catch 中随意 `arrive_and_drop()` 就认为完成修复。本例每轮有两个 phases，参与者数量、失败的 partial slot 和后续读取都需要协调。`combine` 必须只读 partial 且不产生其他未同步的共享写入。Barrier phase rules 参见 [C++ draft: barrier](https://eel.is/c++draft/thread.barrier.class)。

**Follow-up:** 如果 Main 持有某个 mutex 等待 `future.get()`，而 worker 要获取同一 mutex 才能完成 future，会发生什么？

会产生等待依赖环，也就是 Deadlock。

**Self-check — 1 point each**

- [ ] 指出下一轮 overwrite 与当前读取冲突。
- [ ] 给出两次 phase synchronization。
- [ ] 区分 barrier、latch、join、future.get。
- [ ] 区分 uncaught exception terminate 与参与者缺失。
- [ ] 说明 capture lifetime 和 cancellation protocol。

</details>

### Answer 9

[返回 Q9](#q9-implement-reservation-using-cas)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

```cpp
#include <atomic>

std::atomic<int> stock{100};

bool tryReserve(int quantity) {
    if (quantity <= 0)
        return false;

    int current = stock.load(
        std::memory_order_relaxed
    );

    while (current >= quantity) {
        if (stock.compare_exchange_weak(
                current,
                current - quantity,
                std::memory_order_relaxed,
                std::memory_order_relaxed
            )) {
            return true;
        }
    }

    return false;
}
```

注意这里的 `current` 是一个普通局部变量。

CAS 失败时，C++ 会更新它，使其包含本次 CAS 观察到的实际值。之后循环重新检查：

```text
current >= quantity
```

假设：

```text
stock = 10

A wants 8
B wants 5

A reads 10
B reserves 5 → stock becomes 5
A CAS(10 → 2) fails
A current becomes 5

A checks 5 >= 8 → false
```

A 必须失败，而不能继续按旧的库存 10 做决定。

如果所有对库存的操作遵守相同的计数规则，不需要通过 `stock` 发布其他数据，并且不存在无关的非法写入，那么这里的 relaxed 可以满足单个库存原子条件更新的需求。

成功 CAS 是 Linearization Point。

注意：

- CAS 保证当前更新的 atomicity。
- 不保证整个外部业务 transaction。
- 也不保证每个 caller 都能在有限尝试次数内成功。
- 内部 CAS loop 的无锁进展结论还依赖底层 atomic 操作和调度假设。

**Follow-up:** 能不能在 CAS Loop 中调用 `chargeCreditCard()`？

不应该。CAS 失败可能重试多次，不可重复的 side effect 可能被执行多次。

失败操作可在观察到不足库存的 atomic load / failed CAS observation 上定位；quantity invalid 则由参数检查决定。`current >= quantity > 0` 保证本例成功 subtraction 不产生 signed underflow，但补货等新增 API 仍需独立检查 overflow 和 invariant。

**Self-check — 1 point each**

- [ ] 正确处理 quantity / inventory invariant。
- [ ] 实现 expected 更新后的 CAS retry。
- [ ] 解释成功与失败的 linearization points。
- [ ] 限定 relaxed 成立的 publication assumptions。
- [ ] 说明 starvation、atomic implementation 与 side-effect 限制。

</details>

### Answer 10

[返回 Q10](#q10-memory-ordering-litmus-test)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

| Memory Order               | r1=0, r2=0 是否允许 |
| -------------------------- | --------------- |
| 全部 relaxed                 | 是               |
| Store release，Load acquire | 是               |
| 全部 seq_cst                 | 否               |

因为 relaxed 不要求两个不同 atomic variables 的这些操作符合单个统一的 SC 总顺序。

这里的 x、y 都是 atomic；r1、r2 由各自线程写入，Main 在 join 后读取，因此按题目假设没有 data race。

对于全部 seq_cst 的情况，双零会产生不可能的总顺序：

```text
X_store < Y_load < Y_store < X_load < X_store  // impossible cycle
```

所以双零不允许。

对于 release/acquire，如果两个 loads 都读到初始值 0，那么它们没有观察到对方的 release store，因此没有建立想要的同步 handoff。

关键面试结论：

> Acquire doesn't force a load to observe the latest wall-clock write. Synchronization depends on the value it actually reads and the memory model's rules.

**Self-check — 1 point each**

- [ ] 正确判断 relaxed 的双零结果。
- [ ] 证明题设没有 data race。
- [ ] 写出 seq_cst 禁止双零的 order cycle。
- [ ] 解释 acquire 未读到 release 时无 handoff。
- [ ] 避免 latest wall-clock / flush RAM 的错误解释。

</details>

### Answer 11

[返回 Q11](#q11-safe-publication-vs-atomic-flag)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

`relaxed` 只保证对 `initialized` 的 atomic 访问。

它不建立 `config` 写入与读取之间的跨线程同步关系。

在没有题目之外的 synchronization 的情况下，init 的 ordinary writes 与 getConfig 的 reads 没有 happens-before，构成 data race，导致 undefined behavior；不能只解释为“可能读到旧值”。

修复：

```cpp
void init() {
    config.port = 8080;
    config.timeout = 100;

    initialized.store(
        true,
        std::memory_order_release
    );
}

Config getConfig() {
    while (!initialized.load(
        std::memory_order_acquire
    )) {}

    return config;
}
```

在这个 one-shot protocol 中，推理链如下：

```text
Thread A:
write config fields
       ↓ sequenced-before
release store true
       ↓ synchronizes-with
acquire load reads true
       ↓ sequenced-before
Thread B reads config
```

因此：

```cpp
Config writes happens-before Config reads
```

但如果之后重新初始化或反复更新 config，仅仅把 flag 重新设为 false 再设为 true 并不足够，因为某个 Reader 可能仍在访问上一轮数据。

动态配置通常需要新的协议，例如：

- Mutex 保护更新与读取。
- Immutable Snapshot。
- Atomic shared ownership。
- 版本管理与安全 reclamation。

**Follow-up:** Release store 是否意味着把所有 CPU caches 刷回 RAM？

不是。C++ 提供的是语言级同步与顺序保证，具体硬件实现依平台而定。

**Self-check — 1 point each**

- [ ] 指出 config ordinary accesses 的 data race。
- [ ] 给出 release store / acquire load。
- [ ] 写出完整 happens-before chain。
- [ ] 限定 one-shot 与 object lifetime。
- [ ] 说明 repeated update 需要新的 reader protocol。

</details>

### Answer 12

[返回 Q12](#q12-memory-ordering-for-a-completion-counter)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

这里的 `completed.fetch_add(relaxed)` 保证：

- 每个 increment 原子进行。
- 不会丢失完成计数。

但它没有提供普通 `results` 写入之间所需要的跨线程 synchronization。

因此最后一个 Worker 对其他线程 results 的读取不能仅凭 relaxed counter 证明安全。

一种修复：

```cpp
int old = completed.fetch_add(
    1,
    std::memory_order_acq_rel
);
```

假设完成的 RMW 顺序是 A、B、C、D。

那么：

```text
A writes results[A]
A fetch_add(acq_rel)
           ↓
B fetch_add(acq_rel)
B releases prior effects
           ↓
C fetch_add(acq_rel)
           ↓
D fetch_add(acq_rel)
D reads all results
```

通过相应的 RMW 读取关系与 acquire/release 同步，之前 Worker 的结果可以被后续完成操作传递。

当然，这要求所有 Worker 都成功写完自己的结果后才增加 completed，而且之后不再修改这些 slots。

如果只需要 Main 汇总，一个更简单的方案通常是：

```cpp
for (auto& t : workers)
    t.join();

// Main can now read completed results.
```

Join 会提供相应的 completion synchronization，避免手写额外的完成计数协议。

如果 Worker 可能异常退出，需要确保失败状态能被正确传播，不能让协调线程误以为成功完成，或者永远等不到预期 completion count。

**Proof detail:** Atomic RMW 读取 modification order 中紧邻它之前的 value。所有 increments 都为 acq_rel，因此每个 RMW 的 acquire 接收前一个 release，形成 transitive happens-before chain。Last worker 的自身 slot 则由 sequenced-before 保证。Ordering rules 参见 [C++ draft: atomic ordering](https://eel.is/c++draft/atomics.order)。不能把 counter reset 后直接复用同一组 slots，除非已证明上一轮 readers 全部结束。

**Self-check — 1 point each**

- [ ] 区分 atomic completion count 与 result publication。
- [ ] 给出 acq_rel RMW 修复。
- [ ] 以 modification order / read-from 链证明。
- [ ] 声明 unique slots、one-shot、no later writes。
- [ ] 讨论 join alternative 与 worker error completion。

</details>

### Answer 13

[返回 Q13](#q13-analyze-a-lock-free-stack)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

成功的 `compare_exchange_weak` 是 pop 操作预期的 Linearization Point。

但原始代码不是一个完整安全的并发 Stack 实现。

主要有两个不同的问题。

问题一：ABA。

初始：

```text
head → A → B → C
```

Thread 1：

```text
oldHead = A
next = B
pause
```

Thread 2：

```text
pop A
pop B
push A

head → A → C
```

这是分析 pointer equality 的 logical ABA schedule。为把它作为 defined C++ execution 分析，还要假设额外的 test gates 等同步手段使 Thread 1 对 `A.next` 的读取 happens-before Thread 2 的改写；仅让 A、B 不被 delete 并不足够。原始 plain `Node::next` 在没有这种协议的 node reinsertion 中，还可能发生 data race。

在上述同步的示意 schedule 中，A、B 保持存活，A 重新插入前 `A.next` 改为 C，但 Thread 1 已经保存了旧 next=B。

Thread 1 恢复：

```text
CAS(head, A, B)
```

因为 head 再次为 A，CAS 可以成功。

但 B 已经不在 Stack 中。

此时 head 被错误更新为 B，破坏 Stack 结构。

问题二：Use-after-free。

假设 Thread 1：

```cpp
Node* oldHead = head.load();
```

然后暂停。

Thread 2 移除并删除 oldHead。

Thread 1 恢复：

```cpp
Node* next = oldHead->next;
```

此时正在 dereference freed memory。

即使 head 本身是 atomic，也不能保护节点的 lifetime。

修复方向：

- Hazard Pointers：保护仍可能被 Reader 使用的节点。
- Epoch-Based Reclamation：推迟回收到旧 Readers 安全退出。
- Tagged Pointer：为 head 的变化提供版本信息，检测某些 ABA。

注意，Tagged Pointer 本身不保证被指向的对象存活。

Hazard Pointer 解决 lifetime 也不自动证明逻辑 ABA 已消失。

实际方案需要同时满足：

1. 安全的 node lifetime。
2. 正确的 CAS invariants。
3. Memory ordering。
4. 完整的 push/pop 交错证明。
5. 进展保证。

**Follow-up:** 如果这个 Stack 从来不 delete 节点，是否就完全安全了？

不一定。可以避免某些 use-after-free，但逻辑 ABA 仍可能破坏数据结构。

**Reclamation follow-up:** Hazard pointer 常见流程是 load candidate → publish hazard → re-read/validate head → 在 protection 有效期间 dereference。这里只是协议轮廓，不能直接从该顺序推导任意 relaxed implementation 正确。EBR 需要 readers 正确 enter/exit epochs；stalled reader 可能延迟回收并造成 memory growth。

Tagged pointer 还需分析 tag wraparound、pointer+tag 是否能原子比较，以及该 atomic 是否 lock-free。Reclamation、逻辑 ABA 和 node fields 的 race freedom 必须分别证明。

**Self-check — 1 point each**

- [ ] 定位 intended CAS linearization point。
- [ ] 给出有明示前提的 logical ABA schedule。
- [ ] 独立指出 use-after-free 和 node-field data race。
- [ ] 区分 tagging 与 safe reclamation。
- [ ] 讨论 revalidation、tag wraparound、stalled readers / progress。

</details>

### Answer 14

[返回 Q14](#q14-atomic-pointer-vs-atomic-shared-ownership)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

Reader 可以加载 old pointer，然后 Writer 立即替换并 delete old object。

Reader 后续调用：

```text
use(*p);
```

可能访问已经被销毁的对象。

即使：

```cpp
current.load(std::memory_order_acquire);
current.exchange(next, std::memory_order_acq_rel);
```

也不能自动解决 lifetime 问题。

Acquire/release 解决的是必要的内存发布与顺序关系，不会延长裸指针指向对象的生命周期。

C++20 可以使用：

```cpp
#include <memory>
#include <atomic>

std::atomic<std::shared_ptr<const Config>> current;
```

Reader：

```cpp
void read() {
    auto snapshot = current.load();

    if (snapshot)
        use(*snapshot);
}
```

Writer：

```cpp
void update() {
    auto next = std::make_shared<const Config>(
        buildConfig()
    );

    current.store(next);
}
```

Reader 读取到的 `shared_ptr` 拥有有效共享所有权。

因此，即使 Writer 将 current 改成新版本，Reader 手中的旧 snapshot 也不会在它仍持有所有权时被删除。

不过：

- `shared_ptr` 的引用计数有成本。
- Atomic shared_ptr 不保证 lock-free，见 [C++ draft: atomic shared_ptr](https://eel.is/c++draft/util.smartptr.atomic.shared)。
- Snapshot 内部仍需满足 immutable 或正确同步的要求。
- 它不会解决跨多个独立 Config 对象的事务一致性。

如果读取频率极高、引用计数竞争成为瓶颈，并且团队有能力维护更复杂的生命周期协议，才值得考虑 RCU 或 Epoch-Based Reclamation。

**Self-check — 1 point each**

- [ ] 给出 load-old / delete-old / use-old schedule。
- [ ] 说明 acquire/release 不延长 lifetime。
- [ ] 以 owning atomic shared_ptr snapshot 修复。
- [ ] 说明 immutable data / object publication 前提。
- [ ] 讨论 refcount、reclamation 与非 lock-free 限制。

</details>

### Answer 15

[返回 Q15](#q15-lock-free-progress-guarantees)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

1. 不一定违反 Lock-free。

Lock-free 关注整个系统的进展。

只要在相应假设下，持续运行的操作中不断有操作完成，即使 A 不断失败，仍可能满足 lock-free。

2. 仅观察到 100 万次失败，不能单独证明违反 wait-free。

Wait-free 要求每个操作都有有界步骤的完成保证。

有限次 retry 的 trace 不能排除存在更大的 finite bound。但如果算法允许 A 在持续获得执行步骤时无限次 retry，就不满足 wait-free；普通竞争式 CAS loop 不能仅凭使用 atomic 就宣称有 per-operation bound。Wall-clock completion 还取决于 scheduling。

3. Obstruction-free 更弱。

它只保证当一个操作独自运行足够长、不受其他线程干扰时，可以完成。

但在持续竞争下，不一定有整体进展保证。

4. 整条 API 可能不是 Lock-free。

如果 `malloc()`、logging、reclamation 等路径可能阻塞，那么仅证明内部 CAS Loop lock-free 不足以证明整个 API lock-free。

5. Lock-free 不保证更快。

高竞争下可能产生：

- CAS retries。
- Cache-line bouncing。
- CPU spinning。
- Memory reclamation overhead。
- Unfair progress。

因此性能仍然需要 Benchmark。

**Follow-up:** 一个 lock-free 算法是否能保证没有 Starvation？

不能。某个线程可能一直输掉 CAS 竞争。

**Self-check — 1 point each**

- [ ] 区分 system-wide progress 与 per-operation progress。
- [ ] 有限失败次数不等于已证明非 wait-free。
- [ ] 给出 obstruction-free 的 isolation 条件。
- [ ] 检查 allocation / logging / reclamation 全路径。
- [ ] 说明 starvation、scheduling 和 benchmark。

</details>

### Answer 16

[返回 Q16](#q16-why-is-a-32-thread-counter-slower)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

`relaxed` 仍然是 atomic RMW。

每个线程都修改同一个 `total`，这会形成同一个 memory location 的竞争。

Cache coherence 需要协调不同 CPU cores 对同一条 cache line 的写入权限。

即使不使用 mutex，仍然可能有严重的 cache-line bouncing。

更好的方案：

```cpp
std::atomic<uint64_t> total{0};

void worker() {
    uint64_t local = 0;

    for (int i = 0; i < 1'000'000; ++i) {
        ++local;
    }

    total.fetch_add(
        local,
        std::memory_order_relaxed
    );
}
```

现在每个线程只在最后执行一次共享 atomic 更新。

如果只需要所有线程结束后的最终结果，甚至可以让每个 Worker 把自己的计数保存在独立 slot，Main 在 join 后汇总，避免热路径中的共享 atomic。

True sharing vs False sharing：

- True sharing：多个线程确实更新同一个变量。
- False sharing：多个线程修改不同变量，但变量恰好位于同一 cache line，因此产生不必要的 coherence traffic。

可以考虑 padding 或 `alignas`，但应根据实际布局和 profiling 决定。

Benchmark 需要比较：

- Throughput。
- CPU utilization。
- Context switches。
- Cache misses / coherence-related metrics（平台支持时）。
- Scaling 随线程数的变化。
- p95/p99 latency（若工作负载包含请求）。
- Correctness 和最终计数。

**Follow-up:** 如果把一个共享 counter 换成 32 个互不共享 cache line 的 counters，性能一定变好吗？

不一定。还要考虑内存布局、汇总成本、工作量以及其他 bottlenecks。

**Benchmark trap:** 上面的 local increment loop 可能被 optimizer 直接折叠成一个常量。这适合说明减少共享更新，但不能作为真实 workload 的公平 performance proof。用代表性 work、可验证 checksum、相同工作量和受控 measurement 对比，并观察 generated code；别用 volatile 代替合理 benchmark design。

**Self-check — 1 point each**

- [ ] 指出共享 RMW 的 true sharing / coherence cost。
- [ ] 给出 local aggregation 或 join 后汇总。
- [ ] 正确区分 false sharing。
- [ ] 设计公平 benchmark 并识别 loop folding。
- [ ] 比较 scaling、CPU、throughput、tail latency。

</details>

### Answer 17

[返回 Q17](#q17-debug-a-production-hang)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

这是典型的 Thread Pool Starvation Deadlock 或线程池资源依赖问题。

整个 Pool 只有四个 Workers。

四个 Workers 都在执行 RequestHandlers：

```text
Worker 1 waits for child task A
Worker 2 waits for child task B
Worker 3 waits for child task C
Worker 4 waits for child task D
```

但这些 child tasks 也提交到同一个 Pool。

现在没有空闲 Worker 执行这些 child tasks。

所以：

```text
All Workers waiting
        ↓
Child tasks queued
        ↓
No Worker available
        ↓
No child task completes
```

CPU usage 低，是因为主要线程在等待而不是执行 CPU-intensive work。

Request Queue 深度增长，表示系统不再正常消费请求。

排查时可以：

1. 收集全部 Thread Stacks。
2. 找出各 Worker 等待的 Future。
3. 追踪这些 Future 对应的 Task 所在执行队列。
4. 检查 Task 是否已经开始执行。
5. 绘制 Worker → Child Task → Worker Pool 的依赖关系。

修复方案包括：

- 使用 non-blocking continuations。
- 使用明确支持 waiting worker 执行依赖任务的 work helping / scheduler-aware wait。普通 work stealing 若所有 workers 都已阻塞，并不能凭空产生一个可执行 child 的 worker。
- 避免在 Worker 中同步等待同一 Pool 尚未执行的任务。
- 对不同依赖类型使用经过容量分析的独立执行资源。
- 重新设计任务依赖关系。

单纯增加 Worker 数量可能延后问题，但如果每个 Worker 都采用同样模式，仍可能再次耗尽线程池。

**Follow-up:** 如果某个 Worker 的 Future 永远不会 ready，但没有等待环，这一定叫 Deadlock 吗？

不一定。还可能是 lost notification、external stall、task failure 或不完整的 shutdown 协议，需要根据依赖关系和进展状态判断。

**Self-check — 1 point each**

- [ ] 识别 fixed-pool starvation dependency。
- [ ] 用 stacks + queued child tasks 验证。
- [ ] 解释 low CPU 和无 mutex 的等待环。
- [ ] 给出可执行 continuation / helping 修复。
- [ ] 说明普通 stealing 与加 workers 的局限。

</details>

### Answer 18

[返回 Q18](#q18-deterministically-test-a-concurrency-bug)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

设计一个测试 gate，让两个线程都完成 `contains()`，然后再允许它们继续执行 `insert()`。

概念流程：

```text
Thread A checks key absent
Thread A pauses at test gate

Thread B checks key absent
Thread B pauses at test gate

Test releases both threads

Both attempt insertion
```

这样可以稳定制造两个线程都基于旧状态做决定的执行顺序。

应该验证的 Invariant 取决于 API contract，例如：

> 对同一个 key，最多一个 insert-if-absent 操作可以报告自己成功插入。

如果底层容器已经保证插入唯一性，但上层两个 callers 都错误地返回 success，同样违反 composite operation 的 contract。

ThreadSanitizer 不一定找到这种 Bug。

因为单独的 `contains` 和 `insert` 可以都是线程安全的，不存在非法的普通内存并发访问。

错误可能纯粹是业务层面的逻辑竞态。

Controlled Schedule 可以证明某个错误执行顺序确实有问题，但无法穷举所有 schedule。

Linearizability Checking 可以记录：

- Operation invocation。
- Operation response。
- Return value。
- 关键状态变化。

然后检查是否存在符合实时先后约束的合法串行历史。

**Follow-up:** 测试中使用 Barrier 会不会改变原程序的 scheduling 或 memory ordering？

会。测试同步本身可能增加 happens-before 关系或改变交错机会，所以不能把测试通过视为原程序在所有 schedule 下正确的证明。

Test gate 应放在单次 contains 调用已经释放内部 mutex 之后。否则 Thread A 持锁等待 barrier，Thread B 无法完成 contains 到达 barrier，测试自身就会 deadlock。正确实现应以受同一同步保护的 insert-if-absent result 决定 success / side effects。

**Self-check — 1 point each**

- [ ] 明确 composite success / side-effect contract。
- [ ] 用 gate 控制两次 contains 均返回 absent。
- [ ] gate 不在 container 内部持锁区。
- [ ] 验证 invariant 与 history，不依赖 sleep / TSan。
- [ ] 说明测试同步会改变 ordering、不是普遍证明。

</details>

### Answer 19

[返回 Q19](#q19-design-a-high-performance-configuration-service)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

第一版：`std::shared_mutex`

Reader 获取 shared lock，读取配置。

Writer 获取 exclusive lock，更新配置。

优点是容易理解和验证。

但随着 Readers 增多，shared lock 的内部 metadata 和 cache-line contention 可能影响性能。

第二版：Immutable Snapshot + Atomic Shared Ownership

C++20：

```cpp
#include <atomic>
#include <memory>
#include <utility>

struct Config {
    int port;
    int timeout;
    // More immutable configuration data
};

class ConfigService {
    std::atomic<std::shared_ptr<const Config>>
        current_;

public:
    explicit ConfigService(Config initial) {
        current_.store(
            std::make_shared<const Config>(
                std::move(initial)
            )
        );
    }

    std::shared_ptr<const Config> get() const {
        return current_.load();
    }

    void update(Config next) {
        auto snapshot =
            std::make_shared<const Config>(
                std::move(next)
            );

        current_.store(snapshot);
    }
};
```

Reader：

```cpp
auto snapshot = service.get();

if (snapshot) {
    use(*snapshot);
}
```

这里的 `shared_ptr` 保证：

- Reader 获得一个稳定的 owning snapshot。
- Writer 可以发布新版本。
- 旧版本不会在 Reader 仍持有所有权时被销毁。

前提是 Config 内部确实遵守不可变读取的协议，而不是通过 `mutable`、内部裸指针或其他方式继续进行未经同步的修改。

Writer 在发布前完成新对象构建。如果构建失败，旧 snapshot 保持不变。

如果 atomic shared_ptr 的引用计数成本过高，可以考虑 RCU 或 Epoch-Based Reclamation，但需要处理 Reader participation、grace period、old-version cleanup 和 Writer coordination。

Benchmark 应该关注：

- Reader throughput。
- p50/p95/p99 latency。
- Writer update latency。
- 内存占用和旧版本回收延迟。
- 不同 Reader 数量下的 scaling。
- 长时间持有 snapshot 的 Reader。

**Follow-up:** `std::atomic<std::shared_ptr<T>>` 是否保证整个实现 lock-free？

不保证。

这里未写 memory order 的 atomic load/store 使用 seq_cst，已经足够完成 publication；若改为 acquire-load / release-store，要保留完整 ownership 和 immutable contract。构造完成后还必须安全发布 `ConfigService` 对象本身，不能让 reader 与其 constructor / destructor 并发。

Reader 必须在整个 `use(*snapshot)` 期间持有 owning snapshot，不能取出 raw pointer 后立即丢弃 shared_ptr。最后一个 owner 释放时可能在 reader thread 上执行 expensive destruction，影响 tail latency。

**Self-check — 1 point each**

- [ ] 先给出简单正确的 locking baseline。
- [ ] 构建 immutable snapshot 后 atomic publication。
- [ ] reader 全程持有 ownership。
- [ ] 解释 build failure、destruction 与 RCU tradeoffs。
- [ ] 用真实读负载和长 reader 测量 tail latency。

</details>

### Answer 20

[返回 Q20](#q20-full-system-design--concurrent-order-processing-engine)

<details>
<summary>展开参考答案、follow-ups 与评分要点</summary>

这是开放式设计题，不存在唯一标准实现。

一种可论证的起点是按 Symbol 分片的 single-owner architecture；global mutex 作为 correctness baseline 也可以，只要说明何时需要进一步分片。

一种可行的系统架构

下面只表示 ownership / routing，不包含完整 durability / recovery implementation：

```text
Network threads
      |
      v
Validation + routing + admission control
      |
      +--> bounded queue 0 --> owner 0 --> books for shard 0
      +--> bounded queue 1 --> owner 1 --> books for shard 1
      +--> bounded queue N --> owner N --> books for shard N
                                  |
                                  v
                          result / acknowledgment
```

#### A. Ownership and Ordering

每个 Symbol 被映射到某个 Shard。

一个 Shard 有自己的工作队列及单个状态修改 Owner。

优点是：

- 同一个 Shard 的修改可以串行处理。
- 不同 Shards 能并发执行。
- 更容易定义每个订单的状态转移。
- 减少对整个 Order Book 的全局锁竞争。

但如果 `CancelOrder` 只有 `orderId`，Router 必须能定位订单所属 Shard。

需要一个正确同步的 OrderId → SymbolId/Shard 映射，或者让请求包含能够安全验证的路由信息。

Duplicate Add 可以通过 OrderId 唯一性检查和状态机控制。

对于每个订单，需要定义：

```text
Nonexistent → Active → Cancelled
```

实际还可能需要 Filled、PartiallyFilled、Rejected 等状态。

关键是同一订单的状态修改必须遵守相同的序列化规则。

#### B. Message Acceptance vs Completion

这是面试里很重要的区别。

```text
submit(order)
```

返回 true 可能只代表：

> Request accepted into a queue.

但并不代表：

> Order was successfully processed.

如果 API 要求报告处理结果，可以使用 future、callback 或 response message。

而如果系统承诺“确认完成的订单在进程崩溃后仍不丢失”，还需要明确定义持久化和恢复保证；只放进进程内存队列是不够的。

#### C. Multi-symbol Operations

如果一个请求需要同时修改两个 Symbols，它不能仅依赖两个独立 Shard Owners 各自完成操作，就声称整个操作是 atomic。

需要跨 Shard 协议，例如有序获取资源、事务协调或重新定义业务 contract。

不能简单地说：

> Each shard is thread-safe, therefore the cross-shard operation is atomic.

#### D. Graceful Shutdown

一种常见方案：

1. 停止接收新请求。
2. 明确已经接受的请求如何处理。
3. 通知 Workers 进入 drain 或 cancellation 流程。
4. 唤醒可能在 Queue 上等待的 Workers。
5. 等待 in-flight requests 完成或被明确处理。
6. Join Workers。
7. 在最后销毁 Queue、Order Book 和其他共享对象。

还需要处理处理失败的任务如何报告结果，以及系统异常终止后的恢复策略。

#### E. Performance

初期优先使用简单、容易证明正确的数据结构。

之后根据测量决定是否优化：

| 观察                    | 可能措施                             |
| --------------------- | -------------------------------- |
| Router 成为 bottleneck  | 降低共享映射竞争，优化分片路由                  |
| 某个 Shard Queue 过长     | 分析热点 Symbol 和负载分布                |
| Mutex wait 很高         | 缩短 critical section 或调整 sharding |
| CAS retries 很高        | 减少共享状态或重新评估锁实现                   |
| p99 latency 很高        | 分析 Queue waiting、调度、I/O 和长任务     |
| Memory allocation 太频繁 | 评估 batching、pooling、对象生命周期       |

一个成熟的回答不应该承诺“Lock-free Queue 一定更快”。

应该先用 Benchmark 验证。

#### Final Follow-up: 如果面试官要求你解释整个系统的 Linearizability 呢？

你需要先定义哪个操作必须 linearizable。

例如，对单个 Symbol 的 Add/Cancel 状态更新，可以按照它们在 Owner 内的决定性状态转移点给出串行解释。

但跨 Shard 操作、异步确认、数据库提交和外部系统交互可能需要另外的 contract 与同步/事务协议。

不能因为每个 Shard 都有一个 Owner，就自动认为整个分布式或跨分区系统线性一致。

#### F. Global Identity, Ordering, and Recovery

如果 orderId 全局唯一，不能只在各 symbol book 内 independently check：同一个 orderId 带不同 symbolId 可能到达不同 owners。需要 global identity ownership / atomic reservation，或能验证的 ID-to-owner mapping；同时定义 enqueue 失败、duplicate retry 和 stale mapping 的处理。

Concurrent Add / Cancel 必须定义顺序与 NotFound / AlreadyCancelled semantics。可以按 owner 中的实际执行顺序线性化，不要把 network arrival timestamp 自动当成唯一合法顺序。

Queue 满时明确 reject / block / deadline behavior，保证没有无界 pending memory。Worker exception 必须对已接受 requests 给出 completion / error，并确保状态更新有明确的 exception guarantee；catch-and-continue 不能修复已经被破坏的 invariant。

若 ACK 承诺 crash durability，需先达到约定的 durable log / commit boundary 再 ACK，并通过 replay、dedup 和 recovery 重建状态。单纯 enqueue 或在 RAM 中完成，不满足该承诺。Timeout 表示 caller 不知道结果，不代表 operation 已取消。

#### G. Concrete Verification

- 同一 orderId 并发 Add，只出现一个按 contract 成功的 creation。
- Add / Cancel 的 history 可映射到允许的 state transitions。
- Queue 满与 shutdown 同时发生，所有 accepted requests 都有明确归宿。
- Inject worker failure，检查 promises 是否全部进入 terminal state。
- 若有 durability contract，测试 commit 前后 crash / restart / replay。
- Measure queue delay、service time、p99 latency、throughput 和 hotspot distribution。

**Self-check — 1 point each**

- [ ] 明确 identity scope、ownership 与 routing。
- [ ] 定义 Add/Cancel ordering 和 cross-shard 边界。
- [ ] 区分 accepted、completed、durable acknowledgments。
- [ ] 处理 bounded queues、shutdown、exceptions、timeout。
- [ ] 提出 verification、recovery 和可测性能指标。

</details>

## Scorecard

首次练习日期：________　复测日期：________　总分：____ / 100

| Question | First /5 | Retry /5 | Main mistake / next action |
| --- | --- | --- | --- |
| Q1. Thread-safe Doesn't Mean Atomic Transaction | — | — | |
| Q2. Reentrancy and Hidden Deadlock | — | — | |
| Q3. Fine-grained Locking and Bank Transfers | — | — | |
| Q4. Read-Write Lock Upgrade Race | — | — | |
| Q5. Design a Closeable Bounded Blocking Queue | — | — | |
| Q6. Lost Wakeup, Spurious Wakeup, and notify_one | — | — | |
| Q7. Semaphore Accounting and Deadlock | — | — | |
| Q8. Barrier, Latch, Join, and Future Lifetime | — | — | |
| Q9. Implement Reservation Using CAS | — | — | |
| Q10. Memory Ordering Litmus Test | — | — | |
| Q11. Safe Publication vs Atomic Flag | — | — | |
| Q12. Memory Ordering for a Completion Counter | — | — | |
| Q13. Analyze a Lock-free Stack | — | — | |
| Q14. Atomic Pointer vs Atomic Shared Ownership | — | — | |
| Q15. Lock-free Progress Guarantees | — | — | |
| Q16. Why Is a 32-thread Counter Slower? | — | — | |
| Q17. Debug a Production Hang | — | — | |
| Q18. Deterministically Test a Concurrency Bug | — | — | |
| Q19. Design a High-Performance Configuration Service | — | — | |
| Q20. Full System Design — Concurrent Order Processing Engine | — | — | |

建议把错因写成可执行的下一步，例如“Q12 没有证明 RMW read-from chain，重新画 A→B→C→D”，不要只写“memory ordering 不熟”。

## Practice Plan

| Round | Exercise | Deliverable |
| --- | --- | --- |
| 1 — Explain | 全部 20 题，不先看答案 | 每题先用 60–90 seconds 讲核心，再展开；写具体 interleaving / invariant |
| 2 — Implement | Q5、Q9、Q18、Q19 | 完整 C++ programs、明确 contracts、edge-case / controlled-schedule tests |
| 3 — Challenge | Q13、Q17、Q20 | 追问 failure、shutdown、ownership、progress、durability，检查原方案是否仍成立 |
| 4 — Revisit | 得分低于 4 的题 | 不看旧答案重新作答，记录第二次得分与仍未解决的问题 |

**Coding checks worth doing:**

- Q5：capacity=0；空 / 满队列上 close；多个 producers / consumers；drain 完后结束；重复 close；allocation failure 后仍能让其他 producer 前进。
- Q9：quantity≤0、库存不足、exact depletion、多个 callers 争抢最后库存；成功扣减总数与最终库存一致。
- Q18：用 gate 稳定复现两个 callers 都观察 absent；检查 success count / side effects，而不是只看容器最终 size。
- Q19：慢 reader 持有旧 snapshot、并发 update、build failure、reader release 后的 destruction；检查读取到的是完整版本。

Stress tests、ThreadSanitizer 和 controlled schedules 各有用途，但都不能单独证明所有 executions 正确。不要运行含 UB 的题目 code 来“测出标准允许的结果”；先修复或建立受控、defined 的模型。

## Technical References

以下 links 用于核对本练习涉及的 C++ rules；online working draft 可能随标准演进，具体项目还需匹配所选 language mode / library implementation。

- [C++ draft — Data races](https://eel.is/c++draft/intro.races)
- [C++ draft — Condition variables](https://eel.is/c++draft/thread.condition.condvar)
- [C++ draft — Atomic ordering](https://eel.is/c++draft/atomics.order)
- [C++ draft — Barriers](https://eel.is/c++draft/thread.barrier.class)
- [C++ draft — Atomic shared ownership](https://eel.is/c++draft/util.smartptr.atomic.shared)
