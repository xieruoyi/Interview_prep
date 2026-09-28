# Process & Thread

面试准备笔记：标题和 terminology 保留英文，解释使用中文。通用 OS 概念与 Linux/POSIX、Windows、language runtime 的具体行为会分别注明。

阅读顺序按原大纲。每个机制只在对应章节展开；第 10 节用于口头练习，答案索引指向正文，不重复整段讲解。标为 **Runnable Python** 的例子可以单独保存为 `.py` 运行；`text` code blocks 是示意图或 pseudocode。

## 1. What is a Process?

### 1.1 Definition and Program vs Process

A **process** is an instance of a program in execution.

Program 是 disk 上的 instructions 和相关数据；process 是 OS 为执行它建立的运行环境。它包含 virtual address space、resource references、security context，以及执行程序的 thread(s)。

```text
One program on disk
        |
        +── Process A: PID 100, private state A
        +── Process B: PID 101, private state B
```

同一个 program 可以有多个 instances，各自处理不同 input，执行进度也可以不同。一个 application 也可以由多个 processes 组成，不能把 application、executable 和 process 当成一一对应的关系。

Process 不必始终占用 CPU。等待输入、等待 network response 时，它仍然存在，只是相关 execution 暂时没有运行。

### 1.2 Virtual Address Space and Isolation

**Virtual address 是 process 所使用的 address，不是直接指向 RAM 的全局编号。**

每个 process 通常有自己的 address mapping。CPU 的 MMU 根据当前 address-space context，把 virtual address 转换成 physical address，并检查访问权限。

```text
Process A: virtual 0x1000 ── A's mapping ──> physical location X
Process B: virtual 0x1000 ── B's mapping ──> physical location Y
```

两栋楼都可以有 Room 101；“101”必须结合是哪栋楼才有意义。相同地，A 和 B 可以使用相同的 virtual address 数值，而访问不同的 physical memory。

这带来三个结果：

- A 写自己的 `0x1000`，通常不会修改 B 的数据。
- 把 B 的 pointer 数值直接交给 A，A 使用它时仍按 A 的 mapping 访问。
- 如果 address 未映射或权限不允许，访问会触发 fault；OS 能否处理，以及最终是否终止程序，取决于 fault 的原因。

Page table 用于记录 mappings 和权限。Virtual address space 可以包含尚未实际占用 RAM 的区域；“保留了一段 virtual address range”不等于已经分配相同大小的 physical RAM。

**Isolation 不意味着所有 physical pages 都不共享。** OS 可以共享 read-only code pages，programs 也可以显式建立 shared memory mappings。共享关系由 mapping 决定，不要求两边 virtual address 相同。

### 1.3 Typical Memory Layout

以下是逻辑组成示意，不表示固定地址顺序或增长方向：

```text
Process virtual address space
+------------------------------------+
| Code / text                        |
| Global and static data             |
| Heap / dynamically allocated data  |
| Mapped files and shared libraries  |
| User stack(s)                      |
+------------------------------------+
```

| Region | 用途 |
| --- | --- |
| **Code / text** | 可执行 instructions，通常设置为不可写。 |
| **Global / static data** | 具有 static storage duration 的数据；初始化形式会影响具体 segment。 |
| **Heap** | Dynamic allocation 使用的 memory；allocator 可能管理多个区域，不一定是一块连续空间。 |
| **Memory mappings** | 映射的 files、libraries、shared memory 等。 |
| **User stacks** | 保存各个 thread 的 function-call context；细节见 [Section 5](#5-what-resources-are-private-to-each-thread)。 |

这是常见实现模型，不是所有 languages 都必须采用的固定布局。ASLR、architecture、OS 和 runtime 都会影响实际 layout。

### 1.4 Resources, Identity, and OS Bookkeeping

Process 持有的通常是 OS-managed resources 的 **references**，不代表独占底层 resource。

| Item | 作用与例子 |
| --- | --- |
| **PID** | 在对应 PID namespace 中识别当前 process；退出后可能被重用。 |
| **Credentials / security context** | OS 判断它能否访问 files、devices 等；例如 user/group IDs。 |
| **File descriptors / handles** | 用于请求 read、write、close 等操作的 resource references。 |
| **Sockets** | 本机或跨机器通信的 endpoints，由 OS 管理相关 buffers 和状态。 |
| **Resource accounting / limits** | 跟踪 CPU time、memory usage 等，并施加限制。 |
| **Scheduling-related metadata** | 管理 priority 等策略；现代 OS 中很多具体 scheduling state 属于 thread。 |

在 Unix-like systems 中，FD 是 process 内部使用的整数，按惯例 `0/1/2` 对应 stdin/stdout/stderr。它还可以指向 pipe 或 socket。Windows 常用 handles 引用 OS objects。

**两个 processes 都持有 FD 3，不代表它们打开了同一个 file**；FD 数值也是各自 descriptor table 内的编号。反过来，不同 FD 也可能指向同一个底层 open file description。

OS 教科书常用 **PCB (Process Control Block)** 描述 process 管理信息，用 **TCB (Thread Control Block)** 描述 thread 管理信息。实际 kernel 未必有这两个名字的独立 structures，也可能采用共享或组合的数据结构。

Windows 的 process/resource 与 thread/execution 划分可参考 [Microsoft: Processes and Threads](https://learn.microsoft.com/en-us/windows/win32/procthread/about-processes-and-threads)。

### 1.5 Creation, Exit, and Cleanup

一般流程是：建立 process identity 和 address space，准备 executable、resources 和 initial thread，然后使其可以被调度。

**Linux/POSIX 常见面试追问：**

- **`fork()`:** 创建 child process。Parent 得到 child PID，child 得到返回值 0；失败时 parent 得到 -1。两者有独立的 address spaces。Linux 通常用 copy-on-write，避免立即复制所有 private memory pages。
- **Copy-on-write (COW):** Parent 和 child 起初可以共享 physical pages；对需要私有化的 page 执行写入时，OS 才建立独立副本。这与主动共享可写 memory 的目的不同。
- **`exec()` family:** 用新的 program image 替换当前 process 的 program image；成功时不会返回原来的 code，PID 通常保持不变。它本身不创建新 process。
- **`wait()/waitpid()`:** Parent 获取 child 的退出状态，并回收其残留的 exit bookkeeping。
- **Zombie:** Child 已退出，但 exit status 等少量记录尚未被回收；不是仍在运行或持有完整 address space。
- **Orphan:** Parent 先退出的 child。Linux 会由合适的 subreaper 或 init 接管；它不一定已经终止。

```text
Parent ── fork() ──> Child ── exec() ──> New program
   |                                      |
   +──────────── waitpid() <──────────── exit
```

Multithreaded process 调用 `fork()` 时，child 只保留调用方 thread；其他 threads 曾持有的 locks 可能留下不可用状态。因此不能把它当成“完整复制所有正在运行的 threads”。

Windows 常使用 `CreateProcess`，不能把 POSIX 的 `fork/exec` 模型直接套到所有 OS。

退出通常会释放 private mappings、关闭 resource references。底层 object 若仍被其他 process 引用，不一定立即销毁。

参考：[fork](https://man7.org/linux/man-pages/man2/fork.2.html)、[execve](https://man7.org/linux/man-pages/man2/execve.2.html)、[wait](https://man7.org/linux/man-pages/man2/wait.2.html)。

### 1.6 IPC: How Processes Communicate

| Mechanism | 典型接口或流程 | Message boundary | 适用场景 |
| --- | --- | --- | --- |
| **Pipe** | writer `write` → reader `read` | 普通 byte-stream pipe 不保留任意应用 messages 的边界 | Parent/child、command pipelines |
| **Socket** | 建立 endpoint 后 send/receive | TCP 无边界；UDP 保留 datagram 边界 | 本机 services 或网络通信 |
| **Shared memory** | 映射共同区域后直接 load/store | 由 application 自己定义 layout | 大 buffers、大量重复访问 |
| **Message queue** | send/put → receive/get | 提供离散 messages | Task dispatch、producer/consumer |

**Pipe example — Runnable Python：**

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", 'print("Hello from child")'],
    stdout=subprocess.PIPE,
    text=True,
    check=True,
)
print(result.stdout.strip())  # Hello from child
```

Child 的 stdout 接到 pipe，parent 读取后得到自己的 string。这不是共享同一个 Python object。[Python subprocess documentation](https://docs.python.org/3/library/subprocess.html)

**TCP socket 的典型流程：**

```text
Server                         Client
socket()
bind(address, port)
listen()                       socket()
accept() <──────────────────── connect()
  |
  +── connected socket <─────> send / receive
  |
  +── listening socket continues accepting
```

`accept()` 返回新的 connected socket，原 listening socket 继续服务后续 connections。TCP 是 byte stream；一次 send 不对应一次 receive。Application 可以用 length prefix 或 delimiter 划分 messages；大数据也可能需要循环读取。[Python socket documentation](https://docs.python.org/3/library/socket.html)

**Shared memory example — Runnable Python：**

```python
from multiprocessing import Process, shared_memory

def change_byte(name):
    block = shared_memory.SharedMemory(name=name)
    try:
        block.buf[0] = 99
    finally:
        block.close()

if __name__ == "__main__":
    block = shared_memory.SharedMemory(create=True, size=1)
    try:
        block.buf[0] = 10
        child = Process(target=change_byte, args=(block.name,))
        child.start()
        child.join()  # 完成后才读取，避免与 writer 同时访问
        assert child.exitcode == 0
        print(block.buf[0])  # 99
    finally:
        block.close()
        block.unlink()
```

Name 用于找到 shared memory，不是在传递 raw pointer。两个 processes 都映射相同底层 memory；这里用 completion ordering 协调访问。真正并发修改时，通常还需要 process-shared synchronization，详见 Section 8。

这个例子应保存为文件运行。Windows 在所有相关 handles 关闭后删除 shared memory；`unlink()` 在 Windows 上没有实际效果。[Python SharedMemory documentation](https://docs.python.org/3/library/multiprocessing.shared_memory.html)

Message queues 会在 Section 8 的 ownership 和 producer/consumer 场景继续应用；跨 process 的 queue 通常需要 serialization，不能默认传递的是同一个 object。

## 2. What is a Thread?

### 2.1 Definition and Execution Flow

A **thread** is a unit of execution within a process.

同一个 function 可以被不同 threads 同时调用，也可以由不同 threads 执行不同 functions。它们不是“每个 thread 都有一份完整 program copy”，而是多个 execution flows 使用同一个 process 环境。

```text
Process
  +── Thread A → handle request A
  +── Thread B → handle request B
  +── Thread C → background maintenance
```

Thread 需要各自的 execution state 才能独立前进；shared/private resources 分别在 Sections 4 和 5 展开。以下 scheduling 讨论默认普通 application threads，而非 interrupts 或 kernel 内部特殊执行机制。

### 2.2 Kernel Thread, Core, and Logical CPU

这里的 **kernel thread** 指由 kernel 管理、可被 OS scheduler 调度的 thread。它可以执行 application 的 user-mode code。Linux 语境下，“kernel thread”也可能专指只执行 kernel 工作的 kthread；面试时先明确讨论的是哪一种含义。

| Term | 含义 |
| --- | --- |
| **Physical core** | 实际的 CPU core。 |
| **Hardware thread / logical CPU** | 硬件提供、可供 OS 调度的 execution context。 |
| **Kernel-scheduled thread / OS thread** | OS 调度的软件 execution unit。 |
| **Task** | 一份待完成的工作，不一定对应一个 thread。 |

“8 cores / 16 threads”中的 threads 通常指 16 个 hardware threads，不是系统只能创建 16 个 OS threads。SMT 让一个 physical core 提供多个 logical CPUs，它们共享部分执行资源，所以不等于增加了同样数量的完整 cores。

```text
Thousands of software threads
              |
         OS scheduler
              |
       Limited logical CPUs
```

一个 logical CPU 同一时刻运行一个被调度的 OS thread。其他 threads 可以 ready 或 blocked。Thread 也不永久绑定一个 core，scheduler 可以让它迁移；CPU affinity 限定允许执行的位置，但不自动独占这些 CPUs。

### 2.3 What Is a Runtime?

**Runtime system** 是支持程序执行的软件，例如负责 allocation、garbage collection、task scheduling 或 I/O integration。具体职责因 language 和 implementation 而异；不是每个 runtime 都具备这些功能。

“Runtime error”中的 runtime 只是“执行期间”；这里讨论的是一套 software。

对于 runtime-managed execution：

```text
Application creates user-level tasks
                  |
Runtime chooses which task runs on an OS thread
                  |
OS chooses which OS thread runs on a logical CPU
                  |
CPU executes instructions
```

Runtime 本身也通过 CPU 执行，不是 OS 与 CPU 之间新增的一层硬件。

### 2.4 User-level Threads and Mapping Models

User-level threads 由 runtime/library 管理，最终通过 OS threads 执行。

| Model | Mapping | Parallel execution | Blocking 风险 |
| --- | --- | --- | --- |
| **Many-to-one** | 多个 user-level threads → 一个 OS thread | 这些 tasks 不能彼此同时在多个 logical CPUs 上执行 | 底层 thread 真正 blocked 后，所有依赖它的 tasks 都停下 |
| **One-to-one** | 一个 application thread → 一个 OS thread | 可由 OS 分配到不同 logical CPUs | 一个 thread blocked，不必阻塞其他 ready threads |
| **Many-to-many** | 多个 user-level threads → 多个 OS threads | 受可用 OS threads、CPU capacity 和 runtime policy 限制 | Runtime 可调度到其他 workers，但仍受具体 blocking 行为影响 |

```text
User-level threads: U1 U2 U3 U4 U5 U6
                          |
                   Runtime scheduler
                       /       \
OS threads:           K1       K2
                       \       /
                      OS scheduler
                       /       \
Logical CPUs:         L0       L1
```

若只有 K1、K2 用来执行这组 tasks，那么这组 tasks 同一时刻最多两个执行；存在两个 workers 也不保证它们一直有 CPU 可用。

调用 user-space thread library 不等于创建了 user-level scheduled thread。例如 library 可以只是包装 OS thread API。

Goroutines 是 runtime 把轻量 execution units multiplex 到 OS threads 的一个实际例子，但它们的具体设计不能代表所有 user-level thread implementations。[Go FAQ: goroutines](https://go.dev/doc/faq#goroutines)

### 2.5 Waiting, Blocking, and Scheduling Style

**Runtime-aware waiting：** U1 等待 network data，runtime 暂停 U1，让同一个 K1 去执行 U2。

**Blocking the underlying OS thread：** U1 执行某个实际阻塞 K1 的操作，K1 就暂时无法执行其他 user-level tasks。Runtime 是否用其他 workers 或补充 workers，需要看实现。

因此，“一个 task blocked 会不会拖住其他 tasks”必须先问 mapping 和 blocking operation 的类型。

Runtime scheduling 还可以区分：

- **Cooperative:** Task 主动 yield 或到达 runtime 的等待点。一个长期计算、不 yield 的 task 可能占住 worker。
- **Preemptive:** 有机制暂停当前 task，让其他 tasks 前进；何时可抢占仍由实现决定。

OS 的 preemption 与 runtime 的 preemption 是两层不同的事情。OS 抢占 K1，不等于 runtime 已经把 K1 上的 U1 换成 U2。

### 2.6 Creation, Join, and Lifetime

创建 thread 时，需要准备入口函数、参数、execution context，以及相应管理资源。**调用 thread creation API 后，不能假定 creator 一定先于新 thread 执行下一行。**

`join()` 让 caller 等待 target thread 完成；它不是“让两个 threads 合并”，也不等同于开始执行 target。

**Runnable Python：**

```python
from threading import Thread

def work():
    print("Worker finished")

worker = Thread(target=work)
worker.start()
worker.join()
print("Main continues after worker completion")
```

没有 `join()` 等完成协调，就不能假定下一行执行时 worker 已经完成。

POSIX joinable threads 通常需要 join 或 detach 来回收相关 thread resources；detach 不是停止 thread，也不会延长它引用的 objects 的 lifetime。Language 的具体退出语义要单独确认，不能把“main function return”一概等同于“只退出 main thread”。

## 3. Process vs Thread

### 3.1 Comparison at the Right Level

| Dimension | Separate processes | Threads within one process |
| --- | --- | --- |
| Isolation | 通常有独立 address spaces 和 security boundaries | 共享环境，缺少彼此间的 memory protection |
| Communication | 需要 IPC，或显式建立共享区域 | 可直接传递 references，仍需 lifetime 与 synchronization 协议 |
| Failure impact | 较容易把 fault 限制在单个 process | Memory corruption 可能影响整个 process |
| Creation / management | 通常需要更多资源管理工作 | 通常较轻，但仍有 stack、metadata 等成本 |
| Scaling | 可扩展到多个 machines，但要设计网络协议 | 普通 threads 属于同一个 process，不能直接跨机器 |
| Resource access | 更适合权限隔离、独立生命周期 | 更方便复用同一份 cache、connections、objects |
| Switching | 跨 process 通常还涉及 address-space context | 同一 process 内通常可复用 address-space context |

这张表比较的是常见取舍，不是绝对速度排名。COW、shared memory、runtime、workload 都会改变实际成本。Context-switch 细节集中在 Sections 6–7。

### 3.2 How to Choose

- **Untrusted plugin / fault containment:** 优先考虑独立 process，加上合适的 permissions/sandbox；仅创建 process 不自动限制所有危险操作。
- **频繁访问同一份 in-memory data:** Threads 方便，但先设计 ownership 和 synchronization。
- **独立 CPU-heavy jobs:** 两种方案都可行；比较 runtime parallelism、serialization、failure isolation 和 deployment。
- **大量等待 I/O 的 requests:** Thread pool、async/event loop 或组合架构都可考虑，不能仅凭“I/O 多”就无限创建 threads。
- **需要跨机器:** 使用 network communication；不能继续依赖 raw pointers 或本机 shared memory。

实际系统经常采用混合模式：多个 worker processes，每个 process 内使用 threads 或 async tasks。

### 3.3 Failure Is Not the Same as an Exception

Thread 里的 ordinary exception 是否只结束当前 task/thread，由 language 和处理方式决定。不要把它与 native invalid memory access 混为一谈。

例如，一个 worker 返回 error，application 可以处理；一个 thread 破坏 shared heap，则可能让其他 threads 随后崩溃。Process isolation 也无法消除服务依赖带来的间接故障，只能提供更强的边界。


## 4. What Resources Do Threads Share?

### 4.1 Shared Resource Map

这里讨论的是**同一 process 内的普通 threads**。具体 OS 的特殊机制另行说明。

| Resource | Shared 的含义 | 常见后果 |
| --- | --- | --- |
| **Virtual address space** | 使用同一套 process memory mappings | 有有效 pointer 时可访问同一 object |
| **Code / text** | 执行同一份映射 code | 同时执行 function 不要求复制 instructions |
| **Global / static variables** | 通常是同一个 storage instance，TLS 除外 | Mutable state 需要协调访问 |
| **Heap objects** | 在同一 address space 中可达 | Allocation 不自动提供 object-level thread safety |
| **File descriptors / handles** | 普通 process-level resource references 共享 | 一个 thread 的 close 或重定向会影响其他使用方 |
| **Process identity and settings** | 例如 PID，以及许多 process-wide settings | 改动可能作用于所有 threads |

POSIX 还区分共享的 signal dispositions 与各 thread 的 signal mask；前者定义如何处理 signal，后者控制当前 thread 屏蔽哪些 signals。不要笼统地说“signal 的所有状态都共享”。[POSIX threads overview](https://man7.org/linux/man-pages/man7/pthreads.7.html)

Credentials 的细节也依赖 OS，例如 Windows thread impersonation 可以提供 thread-specific security context，因此“所有 security state 都完全相同”不是通用规则。

### 4.2 Shared Memory Is About Reachability, Not Automatic Use

```text
Thread A reference ──┐
                     ├──> One heap object
Thread B reference ──┘
```

两个 threads 能访问同一 object，不表示它们一定都在使用它。可以把一个 object 约定只交给一个 owner thread。

同样，**local pointer 不代表 pointee 是 private**：

```cpp
// C++ fragment: global shared_value has one shared instance.
int shared_value = 0;

void worker() {
    int* p = &shared_value;  // p 是这次调用的 local variable
                            // *p 仍指向 shared_value
}
```

必须区分：

1. Variable 自己的 storage 在哪里。
2. 它引用的 object 在哪里。
3. 谁能够访问 object。
4. Object 是否会被修改，以及访问顺序是否协调。

### 4.3 Sharing Files and Sockets

假设两个 threads 使用同一个 FD：

- 它们访问的是同一底层打开状态，可能共同推进 file offset。
- 不同 write 的业务内容可能交错；OS 对某个 syscall 的保证，不等于多次 calls 合成的逻辑操作是 atomic。
- 一个 thread 执行 close，另一个不能继续把旧 FD 当成稳定、有效的 resource identity；FD 数值还可能被重用。
- 多个 threads 同时从一个 socket 读取时，数据通常由其中某个 reader 消费，不会自动给每个 reader 各复制一份。

常见设计是由一个 owner 管理 connection，其他 workers 通过 queue 与它交互，或明确规定 reads/writes 的同步协议。

共享一个 FD 和“分别 open 同一个 path”也不同：后者通常产生独立的 open file descriptions 和各自的 offsets。

### 4.4 Thread-safe API Does Not Make a Workflow Atomic

一个 container 的单个 method 可以是 thread-safe，但下面两个步骤不一定组成一个 atomic operation：

```text
if key does not exist:
    insert key
```

两个 threads 都可能先观察到“不存在”，然后各自尝试 insert。解决方案可能是：

- 使用提供完整语义的 atomic `put_if_absent` operation。
- 用同一个 lock 保护 check 和 insert。
- 改成 single-owner design。

这里需要保护的是 **invariant**，例如“同一个 key 只能初始化一次”，而不只是某一行代码。具体 synchronization 见 Section 8。

## 5. What Resources Are Private to Each Thread?

### 5.1 Per-thread State

这里的 private 表示“各自拥有一份 state”，**不保证这块 memory 受硬件保护、其他 threads 无法访问**。

| Item | 作用 |
| --- | --- |
| **Program counter (PC)** | 记录恢复后从哪里继续执行；源码行与 machine instruction 不一一对应 |
| **Register state** | 当前 operands、addresses、flags，以及必要的 floating-point/SIMD state 等 |
| **Stack pointer (SP)** | 标记当前 stack 的位置 |
| **User stack** | 管理当前 function-call chain |
| **Kernel-side execution context** | OS 通常也维护对应的 kernel stack/context；它与 user stack 不同，不能由 application 任意访问 |
| **TLS** | 每个 thread 独立的 variable instances |
| **Identity and scheduling state** | TID、ready/blocked state、priority 等 |
| **OS-specific state** | 例如 POSIX signal mask、thread-local errno；具体取决于平台 |

真正运行时，register values 位于 CPU execution context；暂停后，必要 state 保存在 OS/runtime 管理的 memory 中。每个 thread 并不永久拥有专属 physical registers。

### 5.2 Stack Frames and Heap Lifetime

典型 native function call 会使用 stack frame，保存部分 parameters、local values、return information 和必要的 registers。实际 calling convention 可能通过 registers 传参，compiler 也可以优化掉整个 frame。

```text
Thread A calls: main → handle_request → parse
Thread B calls: worker → compress

A stack                         B stack
+------------------+           +------------------+
| parse frame      |           | compress frame   |
| handle_request   |           | worker frame     |
| main frame       |           +------------------+
+------------------+
```

| Aspect | Stack allocation | Heap allocation |
| --- | --- | --- |
| Lifetime | 常跟随 scope/function execution | 由 ownership、显式释放或 GC 等决定 |
| Management | 通常通过 call/return 自动维护 | 通过 allocator/runtime |
| Typical cost | 通常很低 | 取决于 allocator、size 和 contention |
| Common problem | Deep recursion、过大的 local storage、悬空 references | Leak、use-after-free、fragmentation |
| Sharing | 可以把地址传出，但必须保证 lifetime | 多个 threads 可以持有 reference，但仍需访问协议 |

C++ 示意：

```cpp
void example() {
    int local = 10;          // 自动 storage duration，通常在 stack/register
    int* p = new int(20);    // p 是 local；动态分配的 int 在 heap
    delete p;               // 释放 pointee，不是删除 local pointer variable
}
```

实际 C++ 通常优先采用 RAII 和合适的 smart pointers。Smart pointer 管理 lifetime，不自动使 pointee 的所有操作 thread-safe。

**Stack overflow 与 heap exhaustion 是不同问题。** Stack 的默认大小、是否可增长，以及 allocation 失败如何表现，都依赖 OS/runtime，不能背一个通用固定数值。

### 5.3 Can One Thread Access Another Thread's Stack?

可以，在 memory model 和 object lifetime 允许的前提下。以下是完整 C++ example：

```cpp
#include <iostream>
#include <thread>

int main() {
    int result = 0;
    std::thread worker([&result] {
        result = 42;
    });
    worker.join();
    std::cout << result << '\n';  // 42
}
```

这里 worker 访问 main 的 local object：

- `result` 在 worker 完成前仍然存在。
- Main 在 join 完成后才读取；join 提供 completion synchronization。
- Worker 是唯一 writer，没有与 main 同时读写。

如果 main 提前离开 scope，worker 再访问 `result` 就可能 use-after-lifetime；如果 main 在 join 前无同步地读取，则可能产生 data race。**位置不是唯一问题，lifetime 和 ordering 同样重要。**

### 5.4 Thread-local Storage

C++ 示例：

```cpp
thread_local int request_count = 0;

void handle_request() {
    ++request_count;  // 每个 thread 更新自己的 instance
}
```

TLS 适合 per-thread cache、统计和 context。它避免多个 threads 为这个 variable 的同一 instance 争抢，但不会自动汇总各线程的 counts。

Thread pool 会复用 threads，因此 TLS 通常跟随 **thread lifetime**，不是 request lifetime。前一个 request 留下的 state 可能影响后一个 request，需要明确清理。

如果同一个 logical task 会迁移到不同 workers，thread-local state 也不一定等于 task-local state。

## 6. Context Switch

### 6.1 What Is a Context Switch?

**Context switch** 是执行环境从一个 schedulable thread/task 切换到另一个，使后者可以从此前的位置继续执行。

在现代 OS 中，“process switch”通常可以理解为：切换到了属于另一个 process 的 thread。切换执行 state 与切换 address-space context 是相关但不同的工作。

### 6.2 States and Transitions

本笔记只在这里完整列出 execution-state model。教科书也常称它为 process-state model；在 multithreaded OS 中，execution states 通常要逐个 thread 看。

| State | 含义 |
| --- | --- |
| **New** | 正在建立执行所需的 context |
| **Ready / Runnable** | 可以执行，等待 CPU |
| **Running** | 当前正在执行 |
| **Blocked / Waiting** | 等待 I/O、lock、timer、join target 或其他条件 |
| **Terminated** | 执行结束；部分 bookkeeping 可能尚待回收 |

```text
New → Ready → Running → Terminated
        ↑        |
        |        +── preemption ─────────> Ready
        |        |
        |        +── wait for event ─────> Blocked
        |                                      |
        +────────── event completes ───────────+
```

Ready 缺的是 CPU；Blocked 缺的是继续执行所需的条件。**高 priority 也不能让仍 blocked 的 thread 直接继续执行。**

I/O 完成通常让 thread 变为 ready，不保证它立即 running。一个 process 内可以同时有 running、ready 和 blocked 的 threads。

### 6.3 Typical Switching Steps

以下是概念步骤，具体保存顺序和哪些 state 已在 kernel entry 时保存，依 architecture/OS 而异：

1. 由于 preemption、blocking、yield 或 exit，进入相应 scheduler 路径。
2. 保存当前 thread 必要的 execution state，记录其新状态。
3. Scheduler 从符合条件的 ready threads 中选择下一个。
4. 如果跨 address space，切换相应 translation context。
5. 切换 stack/context，恢复下一个 thread 的必要 state。
6. 继续执行目标 thread。

```text
CPU runs A
   ↓ save A / update state
Scheduler chooses B
   ↓ load B's context
CPU continues B
```

这里不复制整个 heap，不重新启动 B，也不把 A 的所有 memory 写回 disk。Swap/page-out 是另外的 memory-management 机制。

常见触发原因包括 time slice 用完、更高 priority 的 thread ready、等待 I/O、竞争 lock、主动 yield。Timer interrupt 发生后，scheduler 也可能继续运行原 thread，不一定每次都换人。[Microsoft: Context Switches](https://learn.microsoft.com/en-us/windows/win32/procthread/context-switches)

### 6.4 Mode Switch Is Not Necessarily a Thread Switch

**Mode switch** 是 CPU 在 user mode 与 kernel mode 之间转换权限级别。System call 通常需要进入 kernel，但可能由同一个 thread 完成并返回。

```text
Same thread:
User code → syscall → Kernel handles request → User code
```

如果 syscall 需要等待，才可能出现：

```text
Thread A → syscall → blocks
                       |
                    schedule B
```

因此：

- System call 不一定引发 thread context switch。
- Interrupt 不一定引发 thread context switch。
- User-level runtime 可以在 user mode 切换 tasks，不一定请求 OS 切换 thread。

### 6.5 Blocking, Sleeping, Spinning, and Yielding

| Operation | CPU behavior | 不保证什么 |
| --- | --- | --- |
| **Blocking wait** | 不能继续执行时通常让出 CPU，等待 event | 被唤醒后立即运行 |
| **Sleep** | 在一段时间内不运行，之后重新可调度 | 精确时间点恢复，或其他 thread 已完成 |
| **Spin / busy wait** | 保持执行，不断检查条件 | 低 CPU usage、公平性 |
| **Yield** | 给 scheduler/runtime 一个调度机会 | 指定的另一个 thread 会运行 |

不能用 `sleep(1)` 代替 completion synchronization。对方可能超过一秒，也可能根本没有成功执行；应该用 join、event、condition 或 future 等明确机制。

## 7. Why Are Thread Context Switches Usually Cheaper?

### 7.1 State That Must Still Change

这个问题比较的是 **same-process thread switch** 与 **cross-process thread switch**。

两种都需要切换必要的 execution state。Same-process switch 仍然要处理 registers、stack context 和 scheduler bookkeeping，并不是“共享 memory 所以不用保存状态”。

### 7.2 Address-space Context and the TLB

**TLB (Translation Lookaside Buffer)** 缓存 virtual-to-physical address translations，减少反复 page-table walks。

| Work | Same-process thread switch | Cross-process switch |
| --- | --- | --- |
| Thread execution context | 需要切换 | 需要切换 |
| Address-space mapping context | 通常可以继续使用 | 通常需要切换 |
| Address translations | 较容易复用已有 translations | 复用取决于 address-space tags、硬件与 OS |
| Code/data locality | 可能复用同一 working set | 也可能共享部分 pages，但 working set 可能不同 |

不同 address spaces 中相同 virtual address 可能映射到不同 physical pages，因此 translation 必须对应正确的 context。

**不要说“每次 process switch 都清空全部 TLB”。** 现代硬件可用 ASID/PCID 等 tags 区分 address spaces，保留部分 entries；何时需要 invalidation 取决于硬件和 mapping 变化。TLB invalidation 与 refill 本身也有成本。[Linux kernel: TLB](https://docs.kernel.org/arch/x86/tlb.html)、[PCID and page-table switching](https://docs.kernel.org/arch/x86/pti.html)

### 7.3 Direct Cost vs Indirect Cost

- **Direct cost:** 保存/恢复 state、scheduler 工作、必要的 address-space context 操作。
- **Indirect cost:** 切换后访问的数据不在 cache、需要补充 translations、branch prediction 状态不适合新代码等。

Cache 通常不会因为普通 context switch 就全部清空。但新 workload 可能使用不同数据，造成 misses；thread 迁移到其他 core 也可能失去部分 locality。

Same-process threads 如果访问完全不同的大数据集，也不一定有良好的 cache reuse。因此“通常更便宜”是条件性的，不能背固定倍数或固定纳秒数。

### 7.4 User-level Task Switches

若 runtime 在同一个 OS thread 内切换 user-level tasks，可能避免 OS scheduler 和 kernel transition，只保存 runtime 所需的 context。

但它仍可能有 stack/state、queue management 和 cache 成本，也不因此获得新的 CPU parallelism。并且 **user-level task switching 的轻量** 与 **kernel thread switching 的相对轻量** 是两种不同的比较。


## 8. Multithreading

### 8.1 Goals and Workload

Multithreading 可以帮助 application 保持 responsiveness、重叠 I/O waiting，或在可用硬件/runtime 上并行处理工作。它不是默认的加速按钮；concurrency 与 parallelism 的区别及 speedup 上限见 Section 9。

| Workload | 主要时间花在哪里 | 首先考虑 |
| --- | --- | --- |
| **CPU-bound** | 真正执行计算 | 可用 CPU capacity、任务拆分、runtime parallelism |
| **I/O-bound** | 等待 network、disk 或外部服务 | 覆盖等待、控制 outstanding requests |
| **Memory-bound** | Cache misses、memory bandwidth/latency | Locality、数据布局；增加 threads 可能更拥挤 |
| **Lock-bound** | 等待 shared-state coordination | 减少共享、缩短 critical section |

先明确目标是降低单个 task latency、提高 throughput，还是保持 UI responsiveness。这些目标不总是一起改善。

### 8.2 Race Condition vs Data Race

**Race condition**：业务正确性依赖不受控制的执行顺序。它可能涉及多个操作，即使每个单独操作都是 atomic。

**Data race**：在 C++ memory model 中，存在未通过合适 ordering 协调的 conflicting concurrent accesses，其中至少一个是写，且至少一个 access 非 atomic。发生 data race 会导致 undefined behavior。[C++ draft: data races](https://eel.is/c++draft/intro.races)

#### Lost Update

`counter = counter + 1` 在逻辑上包含 read、compute、write：

```text
Initial counter = 10

Thread A                     Thread B
read 10
                             read 10
compute 11
                             compute 11
write 11
                             write 11

Final = 11, expected = 12
```

这是帮助理解丢失更新的 interleaving，不是对有 data race 的 C++ code 行为作保证。即使某次测试结果正好是 12，也不能证明 code 正确。

#### Atomic Operations Can Still Form a Race Condition

```text
if atomic_stock.load() > 0:
    atomic_stock.fetch_sub(1)
    sell_one_item()
```

若初始 stock 为 1，两个 callers 都可能先看到 1，再各减一次，导致 oversell。各次 atomic access 没有消除整个 check-and-act workflow 的 race。

需要保护完整 invariant，例如用 mutex 把“检查 + 扣减”合在一起，或设计正确的 compare-and-exchange loop。

### 8.3 Mutex and Critical Sections

Mutex 提供 mutual exclusion。必须由相关参与方采用一致的 locking protocol，保护对 shared state 的访问以及需要保持的 invariant。

**Runnable Python — protected counter：**

```python
from threading import Lock, Thread

counter = 0
lock = Lock()

def increment():
    global counter
    for _ in range(10_000):
        with lock:
            counter += 1

threads = [Thread(target=increment) for _ in range(2)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join()

print(counter)  # 20000
```

这里每次 read-modify-write 都由同一个 lock 保护；最终 print 在 workers 都结束后执行。示例用于说明 correctness，不是高效的计数方案。

**Lock 本身不会扫描你的 code，也不会禁止其他 thread 绕过它访问 variable。** 只锁 writer、不协调可能并发访问的 reader，通常仍不够。

实践原则：

- 明确“哪个 lock 保护哪些 state/invariants”。
- Critical section 尽量短。
- 避免持有 lock 时做慢速 I/O、调用不受控 callbacks。
- 用 scope-based cleanup，例如 Python `with` 或 C++ RAII，避免异常路径漏 unlock。
- 不要为了降低等待随意缩小范围，导致 invariant 被拆开。

### 8.4 Mutex vs Spinlock vs Semaphore

| Primitive | 核心语义 | 适合什么 | 风险 |
| --- | --- | --- | --- |
| **Mutex** | Exclusive ownership | 保护一组共享状态 | Contention、deadlock |
| **Spinlock** | 等待时持续检查 lock | 极短等待、特定低层场景 | 消耗 CPU；owner 未获调度时尤其差 |
| **Semaphore** | 计数 permits，acquire/release | 限制同时使用资源的数量、通知 | Permit 泄漏、不能自动保护业务 invariant |
| **Read-write lock** | 多个 readers 或一个 writer | 特定读多写少 workload | 额外管理成本、公平性或 writer starvation |

Mutex 常有 user-space fast path，也可能短暂 spin，竞争严重时再让 OS 挂起等待者。不能把所有 mutex implementations 都描述为“每次 lock 都 syscall”。Linux futex 常用于实现这类 waiting/wakeup 支持。[Linux futex](https://man7.org/linux/man-pages/man7/futex.7.html)

Semaphore permits 为 3，可以限制最多 3 个 workers 同时使用某个 external resource。它不会自动保证这三个 workers 对 shared object 的读写安全。

Mutex 通常有 owner 语义；semaphore 可以由一个参与者 acquire、另一个 release。具体 library 也可能提供无 owner 的 primitive lock，例如 Python `threading.Lock`，因此面试讲概念时应注明接口差异。

### 8.5 Condition Variables: Wait for a Predicate

Condition variable 用于等待 shared state 满足某个 **predicate**，例如 queue 非空。

正确的概念模式：

```text
Consumer:
    lock(m)
    while queue is empty:
        condition.wait(m)   # 原子地释放 m 并进入等待
                            # 返回前重新获得 m
    item = queue.pop()
    unlock(m)

Producer:
    lock(m)
    queue.push(item)
    unlock(m)
    condition.notify_one()
```

必须用 `while` 而不是只检查一次的 `if`：

- 可能发生 spurious wakeup。
- 被唤醒后需要重新竞争 mutex。
- 另一个 consumer 可能已经取走 item。

Notify 不等于把 lock 直接交给 waiter，也不表示 predicate 永久为真。没有 waiter 时的 notification 通常不会像 semaphore permit 一样被存下来；**持久的事实应该记录在受保护的 state 中**。

Wait 把“释放 lock + 开始等待”协调起来，否则可能在检查完 predicate、真正睡下之前漏掉通知。[Condition variable wait](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3.html)

### 8.6 Atomics, Visibility, and Ordering

**Atomicity** 保证某个 operation 不被观察为部分完成；**visibility/ordering** 决定其他 threads 何时、按什么约束观察更新。两者相关，但不是同一个概念。

C++ counter fragment：

```cpp
#include <atomic>

std::atomic<int> completed{0};

void mark_completed() {
    completed.fetch_add(1, std::memory_order_relaxed);
}
```

对一个只用于独立计数、不承担发布其他 data 职责的 counter，relaxed increment 可以避免 lost updates。但不能从“counter 更新了”直接推导任意其他 data 都已经安全可读。

**Release/acquire publication — C++ fragment：**

```cpp
#include <atomic>

int payload = 0;
std::atomic<bool> ready{false};

void producer() {
    payload = 42;
    ready.store(true, std::memory_order_release);
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {
        // 教学用 busy wait，生产环境可能更适合 event/condition
    }
    int value = payload;  // 此一次性协议中可观察到 42
    (void)value;
}
```

假设只有这个 producer 写 payload，初始化发生在线程启动前，且不再次修改或重用该协议：consumer 的 acquire load 读到 release store 写的 true，就建立了所需 ordering，使之前的 payload 写入对后续读取可见。[C++ draft: atomic ordering](https://eel.is/c++draft/atomics.order)

不要从这个简化例子直接扩展成可反复使用的 lock-free queue。重用时还涉及 ownership、generation、object lifetime 等。

常见误区：

- **Atomic 不等于 lock-free。** 某些类型或平台上的 atomic implementation 可能使用 locks。
- **Lock-free 不等于更快。** Retry、contention、cache coherence 仍有成本。
- **Lock-free 不等于 wait-free。** 前者保证系统整体进展，某个 thread 仍可能长时间失败；后者要求每个 operation 有有界步骤保证。
- **C/C++ `volatile` 不提供 thread synchronization。** Java `volatile` 有不同的 memory-model 语义，但 `volatile count++` 仍不是一个 atomic increment，不能跨语言类推。[Java memory model](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html)

### 8.7 Deadlock, Livelock, Starvation, and Priority Inversion

#### Deadlock

```text
Thread A: holds Lock 1 → waits for Lock 2
Thread B: holds Lock 2 → waits for Lock 1
```

传统 resource deadlock 的四个必要条件：

1. **Mutual exclusion:** 某些 resources 不能同时共享。
2. **Hold and wait:** 持有一些 resources，同时等待其他 resources。
3. **No preemption:** Resource 不能被系统随意强制收回。
4. **Circular wait:** 存在循环等待关系。

预防可以破坏其中一个条件，例如所有 paths 遵循全局 lock ordering，或使用支持多个 locks 的安全 acquisition API。若采用 try-lock/retry，需要释放已持有资源，并考虑 rollback、backoff 和 fairness。

Timeout 能帮助检测或脱离等待，但不会自动恢复已更新一半的 state，也不等于根治 deadlock。

#### Livelock

Threads 不断运行、重试或相互礼让，但业务没有进展。例如双方总是同时退让后再同时重试。适当 backoff 或改变协调协议可能有帮助。

#### Starvation

其他 threads 持续进展，但某个 thread 长期拿不到 CPU 或 lock。它不要求存在循环等待，可能与不公平的 scheduling/locking 有关。

#### Priority Inversion

Low-priority thread 持有 high-priority thread 需要的 lock，而 medium-priority threads 抢占 low-priority thread，间接拖住 high-priority thread。

Priority inheritance 可以临时提升 lock owner 的 priority，帮助它完成并释放 lock。它是特定 scheduling/locking 支持，不是普通 mutex 的通用默认保证。

### 8.8 Reduce Contention Before Adding Complexity

多个 workers 都修改同一 counter，就需要某种协调。换成 database 只会把协调移到 database，不会让同一数据的 conflicting updates 自动并行。

| Approach | 如何减少冲突 | Trade-off |
| --- | --- | --- |
| **Partitioning / sharding** | 每个 worker 负责不同 data | 跨分区操作更复杂 |
| **Local aggregation** | 先 local 计算，最后合并 | 中间 global view 可能不及时 |
| **Batching** | 一次 lock 处理多项更新 | Batch 太大会增加等待和 latency |
| **Single writer + queue** | 只有 owner 修改 state | Queueing 和 owner bottleneck |
| **Immutable snapshots** | Readers 读取稳定版本 | Publication、复制和回收成本 |
| **Atomic operations** | 对简单操作减少显式 locking | 不自动保护多个 fields 的 invariant |
| **Database transactions** | 交给 database 保证持久化和一致性 | 仍有协调、I/O、network 或 transaction overhead |

例如统计多个 files 的行数，让每个 worker 返回 local count，最后求和，通常比每处理一行就抢同一个 global counter 更合理。

Shared memory 可以减少传递大 buffers 的复制，但并不自动是所有场景最快方案；要计入 synchronization、cache coherence、layout 和 lifetime 管理。

#### False Sharing

即使 Thread A 只写 `counterA`，Thread B 只写 `counterB`，如果它们落在同一 cache line，不同 cores 的写入仍可能反复触发 coherence traffic。

```text
One cache line
+------------------+------------------+
| counterA         | counterB         |
| written by A     | written by B     |
+------------------+------------------+
```

这可以是 performance problem，而不是 data race。可考虑 per-worker layout、padding/alignment 或减少写频率，但应基于 profiling；cache-line size 与硬件有关，不要无条件硬编码某个值。

### 8.9 Thread Pools, Task Queues, and Backpressure

Thread pool 复用有限数量的 workers，减少每个 task 创建/销毁 thread 的成本。

```text
Incoming tasks → Bounded queue → Worker 1
                              → Worker 2
                              → Worker 3
```

Task 是工作项，worker thread 是执行者。一个 worker 可以先后处理很多 tasks；task 数量不等于 thread 数量。

设计时同时考虑：

- **Worker count:** CPU-heavy work 可从可用 CPU capacity 附近开始测量，不能只看机器标称 core 数。
- **Queue capacity:** Unbounded queue 可能把 overload 变成 memory exhaustion 和极高 latency。
- **Backpressure:** Queue 满时阻塞 producer、拒绝请求或降低接收速度。
- **Failure handling:** 保存 task 的 error，不要让 failure 静默丢失。
- **Shutdown:** 停止接收新任务，按策略完成或取消 pending tasks，最后 join workers。
- **Cancellation:** 优先 cooperative cancellation；强行中断可能留下 locks 或不完整 state。

I/O-heavy work 可以需要更多 workers 来覆盖等待，但也受 external connection limits、rate limits 和 memory 限制。不能套一个固定“最佳 thread 数”。

**Runnable Python — bounded producer/consumer：**

```python
from queue import Queue
from threading import Thread

tasks = Queue(maxsize=2)
results = Queue()
STOP = object()

def worker():
    while True:
        item = tasks.get()
        try:
            if item is STOP:
                return
            results.put(item * item)
        finally:
            tasks.task_done()

workers = [Thread(target=worker) for _ in range(2)]
for thread in workers:
    thread.start()

for number in range(5):
    tasks.put(number)  # Queue 满时 producer 等待，形成 backpressure

for _ in workers:
    tasks.put(STOP)    # 每个 worker 一个 stop message

tasks.join()          # 等所有已提交 items 都被 task_done()
for thread in workers:
    thread.join()     # 等 worker 真正退出

print(sorted(results.get() for _ in range(5)))
# [0, 1, 4, 9, 16]
```

输出排序是为了不依赖 workers 的完成顺序。这个示例的 task 是不会预期抛异常的简单乘法；实际系统应捕获 task exceptions 并把失败传给调用方。

`queue.Queue` 用于同一 process 的 threads；跨 process 可以考虑 `multiprocessing.Queue`，其中的 object 通常需要 serialization。这两种 queue 不应混用。[Python Queue documentation](https://docs.python.org/3/library/queue.html)

**Thread pool 也可能 deadlock：** 若所有 workers 都在等待同一 pool 内尚未执行的子 tasks，就没有空闲 worker 执行这些子 tasks。避免这种 nested blocking dependency，或采用支持相应 task coordination 的设计。

### 8.10 Language Runtime Caveats

OS 能 parallel schedule threads，不等于 language code 一定能 parallel execute。

标准的 GIL-enabled CPython 中，同一 interpreter 通常只有一个 thread 执行 Python bytecode。等待 I/O 时经常释放 GIL，某些 native extensions 也会释放它；free-threaded builds 的行为不同。因此回答 Python threads 的 CPU parallelism 时，应先说明 implementation/build，而不是说“Python 永远不能并行”。

GIL 也不是 application-level locking protocol。多步骤业务操作、I/O、extensions 和其他 builds 都可能让“依赖 GIL 恰好保护我的逻辑”失效。[Python threading documentation](https://docs.python.org/3/library/threading.html)

### 8.11 How to Investigate a Slow Multithreaded Program

先测量，再调整 thread 数或改成 lock-free：

1. **CPU utilization:** CPUs 忙于有效计算，还是忙于 spin/retry？
2. **Wait time:** 在等 I/O、lock、queue，还是外部服务？
3. **Task granularity:** Task 是否太小，dispatch 成本超过计算本身？
4. **Queue length / latency:** 输入是否长期超过处理能力？
5. **Context switches / migrations:** 是否有大量 runnable threads 或差的 locality？
6. **Memory behavior:** 是否达到 bandwidth 上限、出现 false sharing？
7. **Scaling curve:** 从 1、2、4 等 workers 逐步比较 throughput 与 tail latency。

测试可能暴露 race，但“跑很多次没出错”不能证明不存在。需要检查 ownership、invariants、lifetime 和 happens-before；language 支持时可用 race detectors 等工具辅助。


## 9. Concurrency vs Parallelism

### 9.1 Definitions and Timelines

**Concurrency:** 多个 tasks 的执行期间重叠，可以交替取得进展，不要求同一瞬间执行。

**Parallelism:** 多个 tasks 在同一瞬间执行。

```text
Concurrency on one logical CPU:
Time →  [A][B][A][B][A][B]

Parallel execution:
L0   →  [A][A][A][A]
L1   →  [B][B][B][B]
```

Concurrency 不是只允许交替；parallel execution 也可以是 concurrent execution。两者强调的维度不同：前者讨论重叠的工作组织，后者讨论实际同时执行。

| Scenario | Concurrent? | Parallel? |
| --- | --- | --- |
| 一个 thread 完成 A 后才开始 B | 否，针对这两个 tasks | 否 |
| 一个 logical CPU 交替执行 A/B | 是 | 否 |
| 两个 logical CPUs 同时执行 A/B | 是 | 是 |
| 单 thread event loop 管理多个 network requests | 是 | Callbacks 在此 thread 上通常不并行 |

使用多个 physical cores 通常可以 parallel execute；SMT 也能提供并发硬件 execution contexts，但共享 core resources，不能等同于同数量独立 cores。

### 9.2 Concurrency Does Not Require Multiple Threads

一个 event loop 可以管理多个尚未完成的 I/O operations：

```text
Start request A → A waits for network
Start request B → B waits for network
Handle B's completion
Handle A's completion
```

当 A 等待时，不让唯一的 event-loop thread 一直 blocked 在 A 上，而是注册等待并处理其他 ready work。

但如果在这个 thread 上运行一个很长的 CPU loop，或调用没有被妥善处理的 blocking API，其他 callbacks 也可能无法及时执行。Async syntax 本身不会把任意 blocking code 自动变为 non-blocking。

### 9.3 Blocking vs Async Is a Different Axis

- **Blocking API:** Caller 等待 operation 完成后才返回。
- **Non-blocking API:** 暂时无法完成时也可立即返回，例如表示 would-block。
- **Async API:** 先启动 operation，稍后通过 callback、future、completion event 等获得结果。

实现 async 的底层可能是 OS async I/O、non-blocking polling/event notification，也可能是 worker threads。**Async 不自动意味着没有 threads，也不自动意味着 CPU parallelism。**

例如 `await` 一个 runtime-aware operation，可以暂停 logical task，而不把 execution worker 一直占住；具体 behavior 仍取决于 awaited operation 和 runtime。

### 9.4 Throughput vs Latency

- **Latency:** 一个 request 从开始到完成经历多久。
- **Throughput:** 单位时间完成多少 requests。
- **Tail latency:** 较慢部分 requests 的延迟，例如 p95、p99。

启动更多 workers 可能提高 throughput，却因为 queueing、contention 或 CPU competition，让单个 request 更慢。不能只报告“总共处理更多了”，还应观察 latency 和资源成本。

如果每个 request 都等待 network，concurrency 可以重叠等待；若每个 request 都在争抢一个完全串行的 critical section，则增加 workers 很难提高该部分的处理能力。

### 9.5 Amdahl's Law

假设某项固定总工作中，可并行部分比例为 `p`，使用 `N` 个理想 workers，忽略 coordination overhead：

```text
Speedup(N) = 1 / ((1 - p) + p / N)
```

例如 90% 可以并行、10% 必须串行：

- N = 4：speedup = 1 / (0.1 + 0.9 / 4) ≈ 3.08。
- N 趋于无穷：speedup 上限为 1 / 0.1 = 10。

这里的 N 指模型中能够真正同时执行等量工作的理想 workers，不能把 100 个 software threads 自动当成 N = 100。

真实系统还会有 dispatch、synchronization、load imbalance、memory bandwidth 等开销，所以该式是理想模型，不是实际性能保证。

### 9.6 Choosing an Execution Strategy

先明确瓶颈，再选择：

```text
Mostly waiting?
    → Consider bounded threads or async I/O.

Mostly independent computation?
    → Partition work across effective CPU capacity.

Mostly shared-state contention?
    → Reduce sharing or coordination frequency first.

Needs fault/security isolation?
    → Consider separate processes alongside the above.
```

Processes、threads、async 都不是互斥选项。一个 service 可以用多个 processes 隔离故障，每个 process 用 event loop 接入请求，再把 CPU-heavy tasks 交给 bounded workers。

## 10. Common Interview Questions

这一节用来 **active recall**。先不看正文，用 30–60 秒回答，再按链接检查遗漏。详细机制已在前面展开，这里只给回答要求和 follow-up。

### 10.1 Core Questions and Answer Checkpoints

| Question | 回答必须覆盖 | Follow-up / reference |
| --- | --- | --- |
| **What is a process?** | Running instance；address space；resources；execution via threads | Program vs process，[1.1](#11-definition-and-program-vs-process) |
| **What is a thread?** | Execution unit；属于 process；有独立 execution state | Scheduler 看到哪一层，[2.1](#21-definition-and-execution-flow) |
| **Can two processes use the same virtual address?** | Address 数值是各自 address space 内的标识；mapping 可不同 | Pointer 为什么不能直接传过去，[1.2](#12-virtual-address-space-and-isolation) |
| **How do processes communicate?** | 给出 IPC 选项，并按一个具体 workload 做选择 | Byte stream vs messages，[1.6](#16-ipc-how-processes-communicate) |
| **What do threads share?** | Memory 与 process-level resources，指出实际后果 | 一个 thread close FD 会怎样，[Section 4](#4-what-resources-do-threads-share) |
| **What is private to a thread?** | PC、register state、stack、TLS、thread metadata | Private 是否代表 memory protection，[Section 5](#5-what-resources-are-private-to-each-thread) |
| **Can a thread access another thread's local variable?** | 可以在 lifetime 与 synchronization 满足时访问 | Dangling reference 与 data race 是两类问题，[5.3](#53-can-one-thread-access-another-threads-stack) |
| **Does one kernel thread correspond to one core?** | 软件 thread 数与 logical CPU 数独立；scheduler multiplexes | SMT 与 affinity，[2.2](#22-kernel-thread-core-and-logical-cpu) |
| **What does a runtime do?** | 给出具体职责；区分 runtime scheduler 与 OS scheduler | 并非每个 runtime 都管理 user-level threads，[2.3](#23-what-is-a-runtime) |
| **What happens when a user-level thread blocks?** | 先问 mapping 和 blocking 类型 | Runtime-aware wait vs blocked worker，[2.4–2.5](#24-user-level-threads-and-mapping-models) |
| **What is a context switch?** | 保存/恢复 execution state；选择 ready work | 不复制 heap，[6.3](#63-typical-switching-steps) |
| **Does every system call cause a context switch?** | 区分 mode transition 与改变 running thread | Blocking syscall 的情况，[6.4](#64-mode-switch-is-not-necessarily-a-thread-switch) |
| **Why are thread switches usually cheaper?** | 明确 same-process 前提；address space/TLB/locality | 不承诺固定倍数，[Section 7](#7-why-are-thread-context-switches-usually-cheaper) |
| **Is incrementing a counter thread-safe?** | 先确认语言与类型；普通 increment 是复合操作 | Mutex 或 atomic RMW，[8.2–8.6](#82-race-condition-vs-data-race) |
| **Can atomic code still have a race condition?** | 单个 atomic operation 与业务 invariant 不同 | Stock check-and-act，[8.2](#82-race-condition-vs-data-race) |
| **Mutex or semaphore?** | Exclusive ownership vs permits；根据 resource model 选 | 具体 API 差异，[8.4](#84-mutex-vs-spinlock-vs-semaphore) |
| **Why wait on a condition variable in a loop?** | Predicate、spurious wakeup、重新竞争 lock | Notify 不存储业务事实，[8.5](#85-condition-variables-wait-for-a-predicate) |
| **How do you prevent deadlock?** | 展示 wait cycle，再提出可执行的 lock-order 协议 | Timeout 不自动恢复 state，[8.7](#87-deadlock-livelock-starvation-and-priority-inversion) |
| **Would a database remove contention?** | 转移 coordination，不消除 conflicting writes | 从 correctness、persistence、overhead 取舍，[8.8](#88-reduce-contention-before-adding-complexity) |
| **Why use a thread pool?** | Reuse、bounded concurrency、queue/backpressure | Nested task waiting 的风险，[8.9](#89-thread-pools-task-queues-and-backpressure) |
| **Does more threading always improve performance?** | Workload、serial fraction、contention、hardware/runtime | 用数据定位 bottleneck，[8.11](#811-how-to-investigate-a-slow-multithreaded-program) |
| **Concurrency vs parallelism?** | 重叠推进 vs 实际同时执行 | 单 thread event loop，[Section 9](#9-concurrency-vs-parallelism) |
| **What is the difference between fork and exec?** | 创建 child vs 替换当前 program image | COW、multithreaded fork，[1.5](#15-creation-exit-and-cleanup) |
| **What is a zombie process?** | 已退出，等待回收 exit bookkeeping | 与 orphan 区别，[1.5](#15-creation-exit-and-cleanup) |

### 10.2 Scenario: Design a Request-processing Service

**Prompt:** A service handles many network requests, and some requests require expensive computation. Would you use processes, threads, or async?

先澄清：

1. CPU time 与 I/O wait 各占多少？
2. Requests 是否共享 mutable state？
3. 是否需要隔离 untrusted code 或 worker crashes？
4. External services 有什么 concurrency limits？
5. 目标是 throughput、latency，还是两者？

**Sample spoken answer:**

> I would separate request handling from expensive computation. For network-heavy work, I would consider asynchronous I/O or a bounded thread pool. CPU-heavy tasks should run with concurrency matched to the effective CPU capacity and runtime. If fault isolation is important, I would use worker processes. I would also bound the work queue and measure throughput and tail latency before increasing concurrency.

Follow-up 可以追问 queue 满时怎么办、worker 失败如何重试、重复执行是否安全。后两项属于 service design，不应仅靠“多开 threads”回答。

### 10.3 Scenario: The Program Gets Slower with More Threads

**Prompt:** A job takes 10 seconds with four threads, but 15 seconds with sixteen. Why?

不要立即断言“context switches 太多”。先提出可验证的 hypotheses：

- Shared lock 是不是把工作串行化了？
- Memory bandwidth 是否已经饱和？
- Tasks 是否太小或 load 不均衡？
- 是否出现 false sharing、oversubscription 或 runtime 限制？
- Benchmark 是否控制了 input、warm-up 和 background load？

**Sample spoken answer:**

> More threads only help if additional work can make useful progress. I would profile CPU usage, lock wait time, memory behavior, and scheduling overhead, then compare scaling at several worker counts. Depending on the bottleneck, I might reduce workers, partition shared state, aggregate locally, or increase task size. I would not assume that replacing a mutex with atomics will automatically improve performance.

### 10.4 Scenario: Choose a Safe Ownership Model

**Prompt:** Many workers update a shared cache. Should every access use one global mutex?

先问 cache 的 correctness requirements：read/write ratio、eviction、object lifetime、跨 key invariant，以及是否允许读旧 snapshot。

可以比较 global mutex、per-shard locks、single owner、immutable snapshots；不能只追求 lock 数量少，忽略删除 object 时仍有 reader 的 lifetime 问题。

**Sample spoken answer:**

> I would first define the invariants and ownership rules. A single mutex is a reasonable starting point if contention is low. If profiling shows it is a bottleneck, I would consider sharding or immutable snapshots, while preserving safe object lifetimes and any cross-key guarantees. The design should make both reads and writes correct, not just protect updates.

### 10.5 Tricky Statements to Challenge

以下都不能无条件当成事实。练习说明缺少了哪个前提，并回到对应章节核对：

| Statement | 检查位置 |
| --- | --- |
| “A local variable is always private and safe.” | Sections 4.2、5.3 |
| “The stack is isolated from other threads.” | Section 5 |
| “Every lock acquisition enters the kernel.” | Section 8.4 |
| “Atomic operations make the whole algorithm correct.” | Sections 8.2、8.6 |
| “A notification guarantees the condition is still true.” | Section 8.5 |
| “A process switch clears every cache.” | Section 7 |
| “A thread pool cannot deadlock.” | Section 8.9 |
| “Async means parallel execution.” | Section 9 |
| “Thread count should always equal physical core count.” | Sections 2.2、8.9 |
| “If a race did not show up in testing, the code is safe.” | Section 8.11 |

### 10.6 How to Structure an Interview Answer

用以下顺序组织口头回答：

1. **Definition:** 一句说明是什么。
2. **Mechanism:** 解释谁维护 state、谁 scheduling、数据如何访问。
3. **Example:** 给一个小而具体的 execution sequence。
4. **Trade-off:** 说清 correctness、isolation 或 overhead。
5. **Qualification:** 指出 OS/runtime/language 前提。

遇到没有给定 platform 的问题，先回答通用模型，再说明具体 OS 或 language 可能不同。比起背“永远”“一定”，能准确说出条件通常更有说服力。
