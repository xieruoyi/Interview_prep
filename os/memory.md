# Memory Management

面试准备笔记：**English terminology + 中文解释**。通用概念以支持 MMU 的 paged virtual memory 系统为基础；Linux/POSIX、Windows 或特定 language 的行为会单独标注。

已有的 process/thread 定义、stack/heap 基础见 [process-thread.md](process-thread.md)。本篇重点是 memory 如何被映射、分配、访问和回收；locks、atomics、memory ordering 的详细协议留给 [synchronization.md](synchronization.md)。

**Examples:** `text` blocks 是示意图或 pseudocode；标为 Runnable Python 的示例可以保存为独立 `.py` 文件运行，不需要大规模分配 memory。C/C++ fragments 用来说明语义，不能把展示错误的代码当成可运行练习。

**Units:** KiB = 2^10 bytes，MiB = 2^20 bytes，GiB = 2^30 bytes。所有 page size、address width、timing 数值都会注明假设，不能当成所有机器的固定配置。

## 1. Why Do We Need Memory Management?

### 1.1 Four Responsibilities

如果 application 直接操作全局 physical addresses，需要自己解决地址冲突、硬件容量变化、权限和碎片等问题。Memory management 把这些职责分层处理：

| Responsibility | 解决什么问题 |
| --- | --- |
| **Allocation** | 谁可以使用哪些 memory，以及何时回收 |
| **Address translation** | Program 使用的 address 如何对应实际 storage |
| **Isolation and protection** | 防止越权访问，限制 read/write/execute |
| **Sharing and efficiency** | 有控制地共享 pages，减少重复数据，按需使用 RAM |

例如，两个 applications 都可以把各自的 object 放在 virtual address `0x1000`，不需要互相协商 physical RAM 的位置；OS 为它们建立不同 mappings。

### 1.2 Three Layers of Allocation

```text
Application
  asks for an object / buffer
            ↓
Language runtime / user-space allocator
  manages objects, blocks, free lists, lifetime
            ↓
OS virtual-memory subsystem
  manages mappings, permissions, page residency
            ↓
Physical memory management + hardware
  manages frames; translates and executes accesses
```

这几个层次的 allocation 不是同一件事：

- `malloc(100)` 是请求一个至少满足 100-byte 使用需求的 allocation。
- Allocator 可能从已经持有的 memory 中切出一个 block。
- 只有需要更多空间时，才可能向 OS 请求更多 mappings 或扩大已有区域。
- 即使 mapping 已成功建立，physical page 也可能要到第一次访问时才准备好。

**Object、allocation block、virtual page、physical frame 是不同层级。** 一个 page 可以存放多个 objects，一个 object 也可以跨多个 pages。

### 1.3 Questions to Keep Separate

讨论 memory 时先问清：

1. Address space 中有没有这段合法 region？
2. 相关 pages 是否已经 resident？
3. 当前 access 是否被 permissions 允许？
4. Language-level object 是否仍然 alive？
5. 多个执行者之间的访问是否满足 synchronization 要求？

OS mapping 合法，不代表 C++ object lifetime 合法；memory 有效，也不代表 concurrent access 正确。不同层级的错误不能用同一种机制解决。

## 2. Process Memory Layout

### 2.1 A Logical Layout

```text
Process virtual address space
+------------------------------------+
| Executable code / read-only data   |
| Initialized data / BSS             |
| Heap and allocator-managed regions|
| Mapped files / shared libraries   |
| Thread user stacks                |
| Unmapped or reserved gaps         |
+------------------------------------+
```

上图只表示组成，不表示地址顺序、固定大小或增长方向。真实 layout 受 architecture、OS、executable format、loader、allocator 和 ASLR 影响。

| Region | 常见内容 | 注意事项 |
| --- | --- | --- |
| **Code / text** | Machine instructions | 通常 read/execute，不允许普通写入 |
| **Read-only data** | 常量 tables、部分 literals | C/C++ `const` 不保证所有对象都在此区 |
| **Initialized data** | 带显式非零初始化的部分 global/static objects | Loader 从 executable 等建立初始内容 |
| **BSS** | 常见 native layout 中的 zero-initialized global/static storage | 不需要在 executable 中逐字节存储同样大小的零 |
| **Heap / allocation regions** | Dynamic objects、allocator metadata | 不一定是一块连续区域 |
| **Mappings** | Files、libraries、anonymous memory、shared memory | 不限于 heap 之外某个固定位置 |
| **Thread stacks** | Function-call execution context | 每个 thread 分别维护；仍属于共享 address space |

“未显式初始化的 global/static variable”与“未初始化的 automatic local variable”不同。在 C/C++ 中，前者通常经历 zero-initialization；不能据此推断 local variable 也自动为零。

### 2.2 Storage Duration Is Not a Physical Location

```cpp
// C++ fragment
int global_count = 7;
static int zero_count;

void example() {
    int local = 3;
    int* buffer = new int[100];
    delete[] buffer;
}
```

典型实现中：

- `global_count` 与 `zero_count` 有 static storage duration。
- `local`、`buffer` 是 automatic variables，可能在 stack、registers，或被优化掉。
- `buffer` 的 pointee 是 dynamic allocation；pointer 自己的 lifetime 与 allocation lifetime 分开。
- `delete[]` 结束数组 allocation 的使用，不会把 local pointer variable 从源码 scope 中移除，也不会自动把它改为 null。

Memory layout 是实现模型；language standard 对 lifetime/storage duration 的要求，不等于强制某个 object 位于某个固定 segment。

### 2.3 Mapped, Unmapped, and Protected Regions

Address 数值在 process 可表达的范围内，不代表一定可访问。Region 可能：

- 没有 mapping。
- 预留但未具备可访问 backing。
- 有 mapping，但不允许当前 read/write/execute。
- 合法可访问，但尚未 resident，访问时需要 fault handling。

Guard pages 常用于在 region 边界保留不可访问空间，以发现某些越界或参与 stack growth。它们只能在触及受保护边界时帮助检测，不能发现同一 accessible page 内所有 object-level 越界。

### 2.4 ASLR

**Address Space Layout Randomization** 让 executable、libraries、stack 或其他 regions 的位置更难预测；具体随机化范围取决于平台与配置。

ASLR 改变 placement，不改变 pointer 的基本语义。它降低某些 attacks 的可预测性，但不会修复 buffer overflow，也不提供 object-level memory safety。

## 3. Virtual Memory

### 3.1 Virtual vs Physical Address

Virtual address 属于某个 address space；physical address 对应系统物理地址空间中的位置。这里关注 RAM mappings，devices 还可能使用 memory-mapped I/O，不能把所有 physical addresses 都等同于普通 RAM。

```text
Process A: VA 0x1000 ── mapping A ──> Frame X
Process B: VA 0x1000 ── mapping B ──> Frame Y

Process A: VA 0x5000 ──┐
                      ├── shared mapping ──> Frame Z
Process B: VA 0x9000 ──┘
```

这同时解释了 isolation 与 sharing：独立 address spaces 可以映射到不同 frames，也可以主动共享某些 frames。

### 3.2 Why Virtual Memory Is More Than Swap

Virtual memory 的核心是 **address abstraction、translation、protection 和 controlled sharing**。Demand paging 让 memory 按需 resident；swap 只是部分系统保存某些非 resident data 的一种机制。

即使机器没有启用 swap，它仍然可以完整使用 virtual addressing、page tables、file mappings 和 memory protection。[Linux memory-management concepts](https://docs.kernel.org/admin-guide/mm/concepts.html)

### 3.3 Address Space, Commitment, and Residency

| Concept | 问的问题 | 不代表什么 |
| --- | --- | --- |
| **Address-space reservation** | 是否占住一段 virtual address range？ | 不代表已占用同样多 RAM |
| **Commitment** | OS 按策略为该 allocation 承担何种 backing/accounting 责任？ | 不代表每个 page 当前 resident |
| **Residency** | 当前哪些 pages 实际位于 physical RAM？ | 不代表它们永远不会被回收或换出 |

Windows 明确区分 free、reserved、committed page states。Reserve 保留地址范围；commit 建立相应 backing 承诺/accounting，physical storage 仍可能按访问建立。Committed 与 resident 不能画等号。[Windows page states](https://learn.microsoft.com/en-us/windows/win32/memory/page-state)

Linux 不完全采用相同 API 语义，并有 overcommit policy。回答跨 OS 问题时，应先说明具体度量和接口，而不是把 Windows commit terminology 当成通用精确标准。

**例子：** Program 保留 1 GiB virtual range，只访问其中一小部分。Virtual size 可以明显增长，而 resident memory 只增长一部分；metadata、allocator 和平台策略会影响实际测量。

### 3.4 Protection and Permissions

Pages 通常带有 access-control attributes，例如 read/write、execute disable、user/supervisor。实际 bits 和权限组合由 architecture 决定，不一定每种权限都能独立组合。

Application 不能随意改自己的 page tables 来获得 kernel memory access。OS 控制 mappings，hardware 在执行 access 时检查当前权限。

同一 physical page 经不同 mappings 暴露时，可以使用不同 permissions；共享 storage 不意味着每个 mapping 都可写。

### 3.5 Cost and Limits

Virtual memory 也有成本：

- Page tables 占用 physical memory。
- Translation 与 TLB misses 有开销。
- Page faults 可能需要 OS work，甚至 storage I/O。
- Reclaim、writeback、swap 会引入 latency。
- Address-space width、resource limits、commit policy、RAM 都可能成为约束。

64-bit pointer 不保证平台使用完整 64-bit virtual address range，也不意味着 application 可无限 allocation。

## 4. Paging and Address Translation

### 4.1 Pages and Frames

**Page** 通常指 virtual memory 的固定大小单元；**frame** 通常指对应大小的 physical memory 单元。实际文档也可能把 physical frame 称为 physical page，要看上下文。

```text
Virtual pages                  Physical frames
Page 0 ──────────────────────> Frame 8
Page 1 ──────────────────────> Frame 2
Page 2 ──────────────────────> Frame 15
```

连续的 virtual pages 不要求 physical frames 连续。Program 可以把一个 buffer 当连续数组使用，而 OS 为其安排分散的 frames。

### 4.2 Split an Address

假设 page size = 4 KiB = 4096 bytes = 2^12 bytes：

```text
Virtual address = [Virtual Page Number | 12-bit offset]
Physical address = [Physical Frame Number | same offset]
```

Formulas：

```text
VPN    = virtual_address // page_size
offset = virtual_address % page_size

physical_address = PFN * page_size + offset
```

Translation 改变 page/frame 部分，**page 内 offset 保持不变**。一个 page 的 base 必须按该 page size 对齐。

### 4.3 Worked Example

假设 VA = `0x12345`，page size = `0x1000`，page table 把 VPN `0x12` 映射到 PFN `0xAB`：

```text
VPN    = 0x12
offset = 0x345
PA     = 0xAB * 0x1000 + 0x345
       = 0xAB345
```

**Runnable Python — 模拟 address translation，不读取真实 process page tables：**

```python
PAGE_SIZE = 4096
page_table = {0x12: 0xAB}

virtual_address = 0x12345
vpn, offset = divmod(virtual_address, PAGE_SIZE)
pfn = page_table[vpn]
physical_address = pfn * PAGE_SIZE + offset

print(f"VPN={vpn:#x}, offset={offset:#x}")
print(f"PA={physical_address:#x}")
assert physical_address == 0xAB345
```

如果 access 跨 page boundary，例如从某 page 最后两个 bytes 开始读取四个 bytes，就涉及两个 pages；第二个 page 可能有不同 permissions 或尚未 resident。

### 4.4 PTE: Mapping Plus Attributes

Page table entry 不只是一个 address。概念上还需要描述：

| Attribute | 用途 |
| --- | --- |
| **Frame reference** | 在有效 resident mapping 中定位 physical frame |
| **Present / valid information** | 表示 hardware 是否可以直接使用当前 mapping；具体语义依平台 |
| **Access permissions** | 读写、执行、user/supervisor 等 |
| **Accessed / reference information** | 帮助判断是否被访问，用于部分 replacement 策略 |
| **Dirty information** | 记录相应粒度上的修改，辅助 writeback/reclaim |

并不是所有 platforms 都由同样的 hardware bits 完成这些工作。Non-present entries 的 OS-specific 编码还可以记录 swap 等信息。

### 4.5 Conceptual Translation Path

```text
CPU issues a virtual-address access
                 ↓
Lookup translation and permissions
        ┌────────┴─────────┐
        │                  │
Usable mapping       Cannot complete access
        │                  │
Access target        Fault / exception
physical location    OS investigates reason
```

多数普通 accesses 不需要进入 OS；hardware 根据已有 mappings 完成。TLB 是此路径的优化，见 Section 6；fault handling 见 Section 7。

示意图是逻辑依赖，不是要求实际 CPU 严格逐步串行执行。部分 cache lookup 和 translation 工作可以重叠。

### 4.6 Page Size and Segmentation

Small pages 可以提高 allocation/protection granularity，减少某些浪费，但需要更多 mappings。Large pages 可以减少 translation overhead，却更难精细分配；huge pages 在 Section 5 单独展开。

**Segmentation** 按 code、data 等逻辑 segments 描述可变大小 regions，通常涉及 base/limit。**Paging** 按固定大小 pages 映射。二者可以组合，不能说系统用了 segmentation 就不能 paging。

现代主流 64-bit application memory management 主要围绕 paging；具体 architecture 仍可能保留部分 segmentation 机制。面试掌握区别即可，不要与 executable 中的 text/data “segments”完全混为一谈。


## 5. Page Tables

### 5.1 Why Separate Address Spaces Need Separate Mapping Contexts

不同 processes 对同一 VA 可以有不同解释，因此 hardware 必须知道当前使用哪套 mapping context。OS 管理 page-table structures，并在需要时选择对应 root/context。

同一 process 的普通 threads 共享 address space，因此通常使用同一套 process mappings。不能理解成“每个 thread 都需要复制整个 page table”。

Page tables 本身位于 memory 中；CPU 使用相应 root information 开始查询。它们不是把全部 mappings 永久放在 registers 里。

### 5.2 The Cost of a Flat Page Table

**假设 32-bit virtual addresses、4 KiB pages、每个 PTE 为 4 bytes：**

```text
Virtual pages = 2^32 / 2^12 = 2^20
Flat page-table size = 2^20 * 4 bytes = 4 MiB
```

即使 application 只使用几页，完整 flat table 也可能需要为整个 range 保留 entries。

再假设 48-bit virtual range、4 KiB pages、8-byte entries：

```text
2^(48 - 12) * 8 bytes = 2^39 bytes = 512 GiB
```

这个计算说明 flat representation 的问题，不代表现代 OS 真为每个 process 分配 512 GiB page table。

### 5.3 Multi-level Page Tables

把巨大 flat array 拆成 hierarchy，只在需要映射的区域分配 lower-level tables：

```text
Root table
   ├── absent → whole range unmapped
   ├── next-level table
   │       ├── absent
   │       └── leaf table → data frames
   └── absent
```

Sparse address space 中大片 holes 可以由 upper-level 的 absent entry 表示，不需要为其中每个 virtual page 都建 leaf entry。

Trade-off：保存空间，但 TLB miss 时的 page walk 可能要查询多个 levels。对于非常密集的 mappings，hierarchy 本身仍有管理成本。

Linux 使用可适配硬件的多级 page-table abstraction；没有用到的 levels 可以折叠。Huge-page entries 可以在较高 level 直接结束 walk。[Linux page tables](https://docs.kernel.org/mm/page_tables.html)

### 5.4 Two-level Worked Example

继续采用教学模型：32-bit VA、4 KiB pages、4-byte PTE，并让每张 table 为 4 KiB：

```text
Entries per table = 4096 / 4 = 1024 = 2^10

Virtual address:
[10-bit directory index | 10-bit table index | 12-bit offset]
```

Walk：

1. 用 directory index 找到相应 second-level table。
2. 用 table index 找到 final PTE。
3. 获取 PFN、确认权限。
4. 拼上 offset 得到 physical address。

一个 leaf table 覆盖 `1024 × 4 KiB = 4 MiB` 的 virtual range。

如果只在一个 4 MiB range 中使用少量 pages，可以仅分配 root 和对应 leaf table，总计约 8 KiB 的 table storage，另加实际 data frames 与其他 bookkeeping。

**这不是所有 32-bit 或 64-bit CPUs 的真实 layout。** Address bits 如何分级取决于 architecture 和配置。

### 5.5 Sharing a Page Does Not Share Every Mapping

两个 page-table contexts 可以指向同一 frame：

```text
A: PTE for VA 0x4000 → Frame 20, read-only
B: PTE for VA 0x9000 → Frame 20, read-only
```

共享的只是指向的 storage，其他 entries、regions 和 permissions 仍可不同。取消 A 的 mapping，不必取消 B 的 mapping。

### 5.6 Huge Pages

在某些 x86 configurations 中，常见的 large mappings 包括 2 MiB、1 GiB；其他 architectures/configurations 不一定相同。

**Benefits：**

- 一个 translation 覆盖更多 bytes，提高 TLB reach。
- 减少 leaf entries 和部分 page walks。
- 对大型、长期访问的 memory regions 可能有收益。

**Costs：**

- 对齐和 physical contiguity 要求更难满足。
- 稀疏访问时可能浪费更多 memory。
- Allocation/compaction 可能引入 latency。
- Protection、reclaim、COW 等操作的 granularity 和 split 行为更复杂。

Linux 的 explicit HugeTLB 与 Transparent Huge Pages 是不同机制。THP 尝试自动使用适当的大页，但不是“开启就让任何 workload 都更快”。关注实际 page sizes、fault latency 与 memory pressure，而不是只看配置开关。

## 6. Translation Lookaside Buffer (TLB)

### 6.1 What It Caches

**TLB 缓存 address translations 及相关 access attributes，不是 application 的实际数据。**

```text
TLB:        "This virtual page maps to that physical frame."
Data cache: "These bytes currently have these values."
```

Page table 是完整 mapping structures 的一部分；TLB 只保留当前有用的一小部分 translations。TLB entry 被淘汰，不代表 page 被从 RAM 中移走。

### 6.2 Hit, Miss, and Page Walk

- **TLB hit:** 找到适用于当前 context 和 access 的 entry，可以避免完整 page-table walk；权限仍需允许此次访问。
- **TLB miss:** Translation 不在相应 TLB 中，需要查询 page tables，或在某些 architecture 上走 software-managed refill。
- **Page fault:** Access 因 mapping 状态或 permissions 等原因无法直接完成，需要 fault handling。

因此：

```text
TLB miss
   ├── Page-table mapping usable → refill translation, continue
   └── Mapping cannot satisfy access → fault handling may be required
```

**TLB miss 不等于 page fault，也不等于 disk I/O。** 此外，即使 TLB hit，写入一个不允许直接写的 mapping 也可能产生 protection fault。

Hardware page walks 还可能命中 CPU caches 或专门的 page-walk caches，所以“4 levels = 每次 4 次 DRAM access”也不准确。

### 6.3 TLB Reach

简化模型：

```text
TLB reach ≈ number of entries * page size
```

假设同一组 TLB entries 数量为 64：

| Page size | Idealized reach |
| --- | --- |
| 4 KiB | 256 KiB |
| 2 MiB | 128 MiB |

实际 CPU 可能为不同 page sizes 使用不同结构，还受 associativity 和 access pattern 影响。上表只展示“大 page 为什么有助于覆盖更多 memory”，不是硬件规格预测。

### 6.4 Context Switching, ASID, and PCID

不同 processes 的相同 VPN 可以映射到不同 PFN。如果硬件不区分 address spaces，复用旧 entry 就会访问错误位置。

ASID/PCID 等 tags 可以区分 translations 所属 context，让不同 address spaces 的 entries 有机会共存。是否保留、何时 flush、如何处理 tag reuse 由 architecture 和 OS 决定。

**不要说每次 process switch 必须清空整个 TLB。** PCID 可以减少某些 page-table switches 的 flush 成本。[Linux x86 PTI / PCID](https://docs.kernel.org/arch/x86/pti.html)

### 6.5 Invalidation and TLB Shootdown

若 OS 改变 mapping 或撤销权限，已有 TLB entries 不能继续无限使用旧结果。

假设两个 CPUs 正在使用同一个 address space：

1. CPU A 对 page table 做修改。
2. CPU B 可能仍缓存旧 translation。
3. OS 必须协调相关 CPUs，使过期 translations 不再可用。
4. 安全失效后，旧 frame 才能在适当条件下用于别的用途。

跨 CPU 协调 invalidation 常称 **TLB shootdown**，可能涉及 inter-processor notifications 和等待。它是频繁 mapping changes 在多核环境中的潜在成本；具体实现不一定对每次修改都广播给所有 CPUs。

## 7. Page Faults and Demand Paging

### 7.1 A Fault Is a Request for Resolution, Not Always a Bug

CPU 遇到无法直接完成的 memory access，产生相应 exception，交给 OS 检查。

常见情况：

| Cause | 可能的处理 |
| --- | --- |
| 合法 anonymous region 第一次访问 | 准备 zero-filled backing / mapping |
| File-backed page 尚未建立可用 resident mapping | 从 page cache 建立 mapping，或读取 file |
| Page 已换出 | 从相应 backing 恢复 |
| 写入 COW-protected page | 建立可写 private version |
| Address 非法或 permissions 不允许 | 向程序报告 fault，常导致终止 |
| Mapping 的 backing 出现问题 | 可能报告其他 fault，例如部分 Unix 场景的 SIGBUS |

**COW write fault 说明“protection fault”也不必然是错误。** Hardware 只知道不能按当前 entry 直接写；OS 知道该 region 是否允许通过 COW 完成写入。

### 7.2 Demand Allocation vs Demand Loading

- **Demand allocation:** 合法区域先不分配所有 private physical storage，实际访问时才准备。
- **Demand loading:** 需要使用某个 page 时，再从 executable/file 或其他 backing 提供内容。

例如，新 anonymous region 第一次读取时，OS 可以在适当实现中使用共享 zero page；第一次写入才需要独立可写 frame。具体路径可能随 OS、page size 和 flags 改变。

因此“大 allocation 成功”和“已经把所有 RAM 准备好”是不同事件。

### 7.3 Typical Fault Handling

```text
Access cannot complete
          ↓
Enter OS fault handler
          ↓
Is this access valid for the region?
     ┌────┴─────┐
     No        Yes
     ↓          ↓
Report       Prepare backing:
error        zero page / COW / file / swap
                 ↓
            Update mapping and permissions
                 ↓
            Retry / resume the faulting access
```

如果需要 storage I/O，faulting thread 可能 blocked；其他 ready threads 仍可被调度。Fault handling 也可能涉及 allocation、reclaim 或等待，所以不能把所有 non-I/O faults 视为零成本。

### 7.4 Minor vs Major Faults

Linux `getrusage` 的常见定义：

- **Minor fault:** 没有要求从 storage 加载 page 的 I/O，例如从已有 page cache 建立 mapping，或某些 demand-zero/COW 情况。
- **Major fault:** Fault handling 需要加载 page 的 I/O。

Minor/major 是 OS 的统计分类，不是“轻微 bug / 严重 bug”。Windows 常见 hard/soft fault terminology 也要按对应工具解释。[Linux getrusage](https://man7.org/linux/man-pages/man2/getrusage.2.html)

**没有 swap 也可能有 major faults**：首次访问未缓存的 file-backed pages 就可能需要 file I/O。

### 7.5 Page Fault vs Segmentation Fault

Page fault 是硬件/OS 层的机制；segmentation fault 通常指 Unix-like OS 报告的 SIGSEGV，以及其常见终止结果。

一个可恢复的 demand-page fault 不需要向 application 报 SIGSEGV。反过来，语言层 illegal access 不一定都被硬件发现，例如 array 越界到同一 writable page 内，可能悄悄破坏别的 object。

“没有 crash”不能证明 memory access 合法。

### 7.6 Why the First Access May Be Slower

第一次访问可能触发：

- Physical storage 的准备与 zeroing。
- Page-table allocation 或 mapping 更新。
- File I/O 或 COW。
- Cold TLB / CPU caches。

这些是不同成本来源。第二次快了，并不能单凭现象判断究竟是哪个层的 cache 生效。

教学计算：若普通 access 为 100 ns，某类 fault 的总服务时间为 1 ms，发生概率 p = 0.001：

```text
Average ≈ (1 - p) * 100 ns + p * 1,000,000 ns
        = 1099.9 ns
```

这个极简模型说明少量昂贵 faults 就能影响平均延迟；数字是示例，不是机器 benchmark，也忽略 parallel overlap 和 queueing。

## 8. Physical Memory Pressure and Page Replacement

### 8.1 What Can the OS Reclaim?

| Memory kind | 常见回收方式 | 代价或限制 |
| --- | --- | --- |
| **Clean file-backed page** | 丢弃 RAM copy，需要时从 file 重读 | 未来可能发生 I/O |
| **Dirty file-backed page** | 通常需完成适当 writeback 后再回收 | Storage bandwidth 和写入 latency |
| **Anonymous private page** | 保留、换出到 swap，或按其他机制处理 | 没有原 file 可直接重建其任意内容 |
| **Reconstructible caches** | 丢弃并在以后重建 | 重建成本 |
| **Pinned / unreclaimable memory** | 不能按普通 page eviction 处理 | 直到相应 owner 解锁/释放等 |

Dirty 表示相对 backing 发生修改，不是“数据损坏”。一块原本来自 file 的 private mapping 在 COW 后，修改后的 page 也不能简单当成 clean file page 丢弃。

### 8.2 Swap vs File-backed Reload

```text
Anonymous data:
RAM contents → swap backing → later restored

Clean file-backed data:
RAM copy discarded → later reloaded from original file
```

Swap 不提供与 RAM 相同的访问速度。即使使用 SSD，fault、I/O 和 scheduling latency 也不同；部分系统还可能在 RAM 中压缩 pages，具体性能更依实现。

没有 swap 不代表没有 reclaim，也不代表所有 memory pressure 都立即变成 OOM。但 arbitrary anonymous contents 少了一条常见保留路径。

### 8.3 Working Set and Thrashing

**Working set** 是 workload 在某个观察窗口内实际需要频繁访问的 pages。它不等于 allocation 总量，也不是 permanent constant。

如果可用 memory 容量无法容纳活跃 working sets，刚换出的 page 很快又被访问，系统可能反复 fault 和回收：

```text
Need A → evict B
Need B → evict C
Need C → evict A
Need A → ...
```

大量时间用于搬运 pages，而不是有用计算，这叫 **thrashing**。

可尝试减少同时活跃的 workers、缩小 working set、改变 locality、增加可用 RAM，或调整 workload placement。增加线程通常不能解决这种容量矛盾。

### 8.4 Replacement Algorithms

以下是教学模型；真实 OS 通常使用更复杂的近似、分类和反馈机制。

| Algorithm | 决策规则 | 优点 / 问题 |
| --- | --- | --- |
| **FIFO** | 淘汰最早进入的 page | 简单，但可能淘汰热点；可出现 Belady's anomaly |
| **LRU** | 淘汰最久没有被访问的 page | 利用 temporal locality；精确维护成本高 |
| **Clock / second chance** | 扫描环形候选集合，参考 accessed 信息 | 近似 recency，通常比精确 LRU 便宜 |
| **Optimal / MIN** | 淘汰下一次使用最晚的 page | 需要未来信息，主要用于理论比较 |

Clock 的基础思路：hand 指向 candidate；若 reference bit 为 1，清零并给第二次机会；若为 0，则可作为 victim。真实实现还可能考虑 dirty state、类别等。

**Belady's anomaly：** 对某些 algorithms，增加 frames 反而会增加 faults。它不是“RAM 越多总会越慢”，而是具体 replacement policy 的现象；精确 LRU 不存在这种经典异常。

**Runnable Python — FIFO simulation：**

```python
from collections import deque

def fifo_faults(references, capacity):
    resident = set()
    order = deque()
    faults = 0

    for page in references:
        if page in resident:
            continue
        faults += 1
        if len(resident) == capacity:
            resident.remove(order.popleft())
        resident.add(page)
        order.append(page)

    return faults

references = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]
for frames in (3, 4):
    print(f"{frames} frames: {fifo_faults(references, frames)} faults")
# 3 frames: 9 faults
# 4 frames: 10 faults
```

这是 page replacement 模拟，不是在改变 OS 的 memory policy。

### 8.5 Overcommit, Failure, and OOM

Linux 的 overcommit policy 会影响是否接受某些 memory commitments。接受 allocation 请求，不应被理解为保证未来每次 fault 都能成功提供 private physical backing。

失败可能出现在不同阶段：

- Virtual address range 或 resource limit 不够。
- Allocator 无法满足请求。
- Commit/accounting policy 拒绝。
- 之后的 page allocation/reclaim 无法完成。
- 系统或 container 的 memory limit 触发 OOM handling。

C `malloc` 失败返回 null；普通 C++ throwing `new` 失败通常抛 `std::bad_alloc`。OS 杀死 process 则不是普通 application allocation error return。

**OOM killer 不保证杀死“引起最后一次 allocation 的 process”，也不应简单背成“必杀占内存最大的 process”。** 实际选择依 OS、policy、limits 和评分等。

### 8.6 Physical Fragmentation and Compaction

普通小 page mappings 可以让 virtual range 连续而 physical frames 分散，但 huge pages 或某些 device requirements 仍可能需要较大连续 physical ranges。

总 free memory 足够，不代表能马上找到所需的连续 physical range。OS 可以尝试 compaction，移动可迁移 pages 来整理空间；这也可能增加 latency。

这个问题与 Section 9 的 user-space allocator fragmentation 是不同层级，不能把两者完全等同。


## 9. Dynamic Memory Allocation

### 9.1 Allocation Request vs OS Mapping

Allocator 经常一次向 OS 获取较大 region，再在其中管理较小 blocks：

```text
OS mapping
+------------------------------------------------+
| allocator metadata | used | free | used | free |
+------------------------------------------------+
                         ↑
                    malloc request
```

一个 small allocation 可以由现有 free block 满足，不需要 syscall。一个 free block 也可能被分割成 used block 和剩余 free block。

Linux user-space allocator 可能使用 `brk/sbrk` 或 `mmap` 等机制取得更多空间；具体选择、thresholds、arenas 和 caches 属于 allocator implementation，不能用一个固定大小规则代表所有系统。[malloc documentation](https://man7.org/linux/man-pages/man3/malloc.3.html)

### 9.2 malloc, calloc, realloc, new, and delete

| API | 关键语义 |
| --- | --- |
| **malloc(n)** | 请求 raw storage，内容未初始化；失败返回 null |
| **calloc(count, size)** | 请求相应 storage 并把 bytes 清零；不等于调用 C++ constructors |
| **realloc(p, n)** | 调整 allocation，可能移动；保留新旧 size 较小范围内的内容 |
| **free(p)** | 释放对应 allocation；不能继续使用旧对象 |
| **new T(...)** | C++ 中通常先取得 storage，再构造 object |
| **delete p** | C++ 中销毁相应 object，再释放 storage |
| **new T[n] / delete[]** | 数组形式必须正确配对 |

补充：

- 对 nonzero size 的 `realloc`，失败返回 null 时原 allocation 仍有效；不要直接覆盖唯一旧 pointer。
- `realloc` 成功后，旧 allocation 的 pointers 不应继续使用，即使返回的数值看起来相同。
- `malloc` 与 `delete`、`new` 与 `free` 混用不合法。
- Ordinary throwing `new` 与 `new (std::nothrow)` 的失败方式不同。
- Over-aligned types 可能需要专用 alignment-aware APIs。
- Zero-size allocation/reallocation 的细节有语言版本和 implementation 差异，实际代码应避免依赖模糊边界行为。

**C fragment — retain old allocation on realloc failure：**

```c
/* Assume p is a valid allocation and new_size > 0. */
void *replacement = realloc(p, new_size);
if (replacement != NULL) {
    p = replacement;
} else {
    /* p is still valid; handle allocation failure. */
}
```

### 9.3 Free Lists, Size Classes, and Coalescing

Allocator 可以维护不同大小的 free blocks：

- **Free list:** 跟踪哪些 blocks 可复用。
- **Size classes:** 把 requests 向某些 block sizes 舍入，加速查找与复用。
- **Splitting:** 把较大 free block 分成所需 block 和剩余部分。
- **Coalescing:** 合并相邻的 free blocks，帮助满足更大 requests。
- **Per-thread caches / arenas:** 减少 allocation contention，但可能增加保留空间。

实际 allocator 不一定采用所有策略。“释放”在这个层面可能只是把 block 标记为可复用。

### 9.4 Internal vs External Fragmentation

**Internal fragmentation：** 已分配 block 内部，application 没用到的部分。

例如请求 33 bytes，被放进 48-byte size class，仅从 payload capacity 看浪费 15 bytes；headers、alignment 还可能增加额外成本。

**External fragmentation：** 可用空间分散成不合适的 holes，无法满足某个需要足够连续空间的 request。

```text
One allocator region:
[free 100][used 100][free 100][used 100][free 100]
```

总共 free 300 bytes，但如果 allocator 必须从这一区域提供单个连续 250-byte block，就无法直接满足。它可能申请新 region，但现有 holes 仍未被有效利用。

Paging 降低了普通 virtual allocation 对连续 physical RAM 的要求，**并没有消除 allocator-level fragmentation 或 huge-page physical fragmentation**。

若单独为 5000 bytes 保留完整 4 KiB pages，需要 2 pages，即 8192 bytes，尾部未用 3192 bytes。但实际 allocator 可以在 pages 中容纳其他 blocks，所以不能把这个计算直接当成 `malloc(5000)` 的真实浪费。

### 9.5 Alignment

Alignment 要求某个 address 是规定数值的倍数，例如 8-byte alignment：

```text
address % 8 == 0
```

原因包括类型/ABI 要求、硬件访问限制，以及性能。Struct 为使 members 满足 alignment，可能插入 padding；因此 `sizeof(struct)` 不一定等于 members sizes 简单相加。

Padding 也可能用于减少 false sharing，但会增加 footprint。应区分“满足正确性所需 alignment”和“为性能选择的数据布局”。

### 9.6 Why free Does Not Necessarily Reduce RSS

可能的执行链：

```text
Object no longer needed
       ↓
Destructor / runtime cleanup
       ↓
Block returned to allocator
       ↓
Allocator may keep it for reuse
       ↓
Only later, some pages may be released/discarded to OS
```

RSS 可能不下降，因为：

- Free blocks 仍在 allocator 的 regions/caches 中。
- Page 上还有其他 live objects，不能整页释放。
- Fragmentation 使可回收整页很少。
- Runtime/allocator 有自己的保留策略。

某些 allocators 提供主动 trim 接口，例如 glibc `malloc_trim` 尝试释放合适的 free memory，但这是特定实现的调优手段，不是所有 `free` 的默认行为，也不是修复真实 leak 的办法。[malloc_trim](https://man7.org/linux/man-pages/man3/malloc_trim.3.html)

## 10. Memory Mapping and Shared Memory

### 10.1 What mmap Does

POSIX `mmap` 建立 virtual address range 与 backing 的关系。之后 application 可以通过 load/store 访问 region，而不必每次显式调用 `read` 或 `write`。

Mapping 创建不等于全部内容已 resident，也不等于已经把整个 file 复制到 process heap。

### 10.2 Two Independent Choices

**Backing 是什么？**

- **Anonymous:** 不以普通 file contents 作为初始 backing，适合 dynamic memory 等。
- **File-backed:** 对应 file 的某段内容。

**Updates 如何传播？**

- **Private:** Writes 不通过该 mapping 修改其他参与者看到的原 file/shared data；常使用 COW。
- **Shared:** 同一 backing 的 shared mappings 可以看到 updates；file-backed writes 还涉及后续 writeback。

这两个维度可以组合。Private 不意味着一开始就已经复制全部 pages；shared 也不意味着各 process 使用相同 VA。[mmap](https://man7.org/linux/man-pages/man2/mmap.2.html)

### 10.3 Mapped I/O vs read/write

| Aspect | read/write | mmap |
| --- | --- | --- |
| Application interface | 显式传入 buffers、offset/count 等 | 通过 memory access 访问 mapped range |
| Data movement | 常见 buffered read 路径会从 page cache 拷贝到 user buffer | 可以避免这一步显式 copy，具体路径依平台 |
| Error handling | 通常在 API return 检查错误 | 某些错误延迟到 access 时表现为 fault |
| Access pattern | Streaming、明确控制 I/O 很自然 | Random access、file-like arrays 可能方便 |
| Costs | Syscalls、copy 等 | Mapping 管理、faults、translation 等 |

两者都可能受益于 OS page cache。**mmap 不保证比 read 更快**，也不能把它叫“完全没有 I/O 的访问”。

File 被另一个进程 truncate 后，访问超过有效 backing 的 mapping 可能 fault。维护 file/mapping lifetime 和 size 是 application 的责任。

### 10.4 Runnable Python: Private Copy vs Shared File Update

下面只创建一个很小的 temporary file，可以在 Windows 上运行：

```python
import mmap
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as directory:
    path = Path(directory) / "sample.bin"
    path.write_bytes(b"hello")

    with path.open("r+b") as file:
        with mmap.mmap(file.fileno(), 0, access=mmap.ACCESS_COPY) as view:
            view[0:1] = b"H"
            print("Private view:", view[:].decode())

    print("File after private:", path.read_bytes().decode())

    with path.open("r+b") as file:
        with mmap.mmap(file.fileno(), 0, access=mmap.ACCESS_WRITE) as view:
            view[0:1] = b"J"
            view.flush()

    print("File after shared:", path.read_bytes().decode())

# Private view: Hello
# File after private: hello
# File after shared: Jello
```

Python 的 `ACCESS_COPY` 提供 copy-on-write behavior，`ACCESS_WRITE` 的修改可写回 file。[Python mmap](https://docs.python.org/3/library/mmap.html)

### 10.5 Visibility, Writeback, and Durability Are Different

- **Visibility:** 另一个 reader 能否正确观察到这次更新。
- **Writeback:** Dirty file-backed data 是否已提交回 backing filesystem/storage 路径。
- **Durability:** 在预期故障模型下，data 是否可以恢复。

共享 mapping 本身不替代 locks/atomics 等 coordination。Reader 看到了数据，也不意味着掉电后仍能恢复。Flush/writeback 不等于建立 application transaction。

POSIX `msync` 可以控制 mapped file updates 的同步；实际持久性还取决于相应 API、filesystem 和存储保证。[msync](https://man7.org/linux/man-pages/man2/msync.2.html)

### 10.6 Why Shared-memory Structures Often Use Offsets

假设 A 把 shared region 映射在 `0x100000`，B 映射在 `0x700000`。某个 record 位于 region 内 offset `0x80`：

```text
A's pointer = 0x100000 + 0x80 = 0x100080
B's pointer = 0x700000 + 0x80 = 0x700080
```

A 把 raw pointer `0x100080` 存进 shared region，B 直接使用通常无效。可以存 region-relative offset，让每个 process 用自己的 base 计算地址。

跨 process data structure 还需要一致的 layout、alignment、版本、lifetime，以及适合 process-shared 使用的 synchronization。不能直接把含普通 heap pointers 的复杂 object 拷进去就认为可以共享。

### 10.7 Mapping Lifetime

Mapping、FD/handle 和 file path 是不同 references。在 POSIX 中，`mmap` 成功后可以关闭原 FD，mapping 仍然有效；`munmap` 移除的是当前 address space 的 mapping。

任何依赖该 mapping 的 pointer/view 都不能在 unmap 后继续使用。其他 process 的独立 mapping 是否仍有效，要看 backing 和操作语义，不能从“我关闭了 reference”推断“所有人的 storage 都消失了”。

## 11. Copy-on-Write (COW)

### 11.1 Motivation

如果两个使用者当前需要相同内容，但未来可能分别修改，立刻复制所有数据可能浪费。COW 先共享内容，在需要独立写入时再分离。

Linux `fork` 是常见应用：为 child 建立独立 address-space context，但许多 private data pages 可以暂时共享。[Linux fork](https://man7.org/linux/man-pages/man2/fork.2.html)

### 11.2 Before and After a Write

```text
After fork, before writing:

Parent PTE ── read-only ──┐
                         ├── Frame X: value = 10
Child PTE  ── read-only ──┘

Child writes 20:

Parent PTE ────────────────> Frame X: value = 10
Child PTE  ────────────────> Frame Y: value = 20
```

典型步骤：

1. 原本允许 private writes 的 region 使用暂时不可直接写的 mappings。
2. Child 尝试写入，hardware 触发 protection fault。
3. OS 判断这是合法的 COW write，而非越权。
4. 如果仍需保持其他 owners 的旧内容，分配新 frame 并复制相关 page data。
5. 更新 child mapping 和必要 translation state。
6. 重试 write；parent 继续看到旧内容。

若该 page 已经不需要与其他 owner 分享，OS 在某些情况下可以直接恢复可写，而不实际复制。**COW fault 不保证每次都执行一次 memory copy。**

### 11.3 Copy Granularity

假设采用 4 KiB base-page COW，修改一个 byte 也可能引起复制整个 4 KiB page。若大量小 objects 位于同一 page，写其中一个 object 就可能触及该 page 的 COW 成本。

Huge pages 可能涉及不同 copy/split 策略，所以不能无条件说 COW 永远只复制 4 KiB。

### 11.4 Benefits and Hidden Costs

**Benefits：**

- 避免复制从不修改的 data。
- `fork` 后很快 `exec` 时，减少不必要的 private page copies。
- Read-mostly 内容有机会长期共享。

**Costs：**

- First-write faults。
- Page copies 和 memory bandwidth。
- Page tables 等 metadata 仍然有成本。
- Parent/child 逐渐写入许多 pages 后，实际 memory usage 可能显著增长。

某些 runtime 的引用计数或其他 metadata updates 也可能让看似“读取 objects”的程序触及 writes。分析 COW 时要看实际 memory writes，不只看业务逻辑是否修改字段。

### 11.5 COW vs Writable Shared Memory

| COW private data | Writable shared memory |
| --- | --- |
| 修改时保持彼此独立的内容 | 修改目标就是共同的底层内容 |
| 其他 owners 通常继续看到旧值 | 其他参与者可按同步协议看到新值 |
| 用于延迟复制、隔离 writes | 用于 IPC、协作访问 |

COW 不是多个 writers 更新同一 shared object 的 synchronization 方案。


## 12. CPU Caches and Locality

### 12.1 Memory Hierarchy

```text
CPU registers
      ↓
L1 cache
      ↓
L2 cache
      ↓
L3 / last-level cache, if present
      ↓
DRAM
      ↓
Backing storage, accessed through OS/I/O
```

越靠近执行单元，通常容量更小、访问成本更低。具体 cache levels、sharing、latency 取决于硬件，不应背一个平台无关的纳秒表。

Storage 不像 L1/L2 一样由普通 cache hierarchy 对任意 load 自动直接查询。访问非 resident virtual page 通常通过 fault 和 OS backing 管理处理。

Registers 保存当前计算直接使用的 values；cache 保存 memory contents 的部分副本。Program 的 variable 也可能被 compiler 一直保留在 register 中，不是每条源码表达式都产生 RAM access。

### 12.2 Cache Lines

CPU caches 通常以 **cache line** 为单位存储和传输数据。64-byte line 很常见，但不是所有 hardware 的通用保证。

假设 line size = 64 bytes，array elements 为 4-byte integers，理想对齐下一个 line 可容纳 16 个 elements。

访问一个 element 时，附近 elements 可能一起进入 cache。之后访问它们就可能命中，不必每个 element 都重新访问 DRAM。

需要区分：

- **Cache line:** Cache/data movement granularity。
- **Page:** Mapping/protection/residency granularity。
- **Object:** Language/application 层的 data unit。

一个 4 KiB page 可以包含 64 个 64-byte cache lines，但这是基于特定尺寸的示例。

### 12.3 Temporal and Spatial Locality

**Temporal locality:** 刚访问的数据很快又访问，例如 loop 中反复使用一个小 lookup table。

**Spatial locality:** 即将访问的数据靠近刚访问的数据，例如顺序扫描 contiguous array。

```text
Sequential:
a[0] → a[1] → a[2] → a[3]

Pointer chasing:
node at X → node at P → node at K → node at Z
```

Linked list 不必然比 array 慢，但典型 scattered nodes 会带来较差 spatial locality，且下一个 address 依赖当前 node，可能限制 prefetch 和并行 memory requests。

### 12.4 Row-major Traversal Example

对于 C/C++ built-in row-major 2D array：

```cpp
// C++ fragments; assume a has R rows and C columns.

// Adjacent columns within a row are contiguous.
for (int row = 0; row < R; ++row) {
    for (int col = 0; col < C; ++col) {
        consume(a[row][col]);
    }
}

// Large stride when C is large.
for (int col = 0; col < C; ++col) {
    for (int row = 0; row < R; ++row) {
        consume(a[row][col]);
    }
}
```

第一种通常更符合 spatial locality。但 array 大小、cache capacity、compiler transformations 和 access stride 都影响结果。

这个结论不能原样套到 Python list-of-lists、column-major arrays 或任意 dataframe 内存表示；先确认实际 layout。

### 12.5 Cache Misses and Performance

常见分析分类：

- **Compulsory miss:** 首次使用，相关内容尚未装入。
- **Capacity miss:** 活跃数据超过 cache capacity。
- **Conflict miss:** 某些 addresses 竞争有限的 cache sets/ways。
- **Coherence-related miss:** 其他 cores 的操作使相应 line 状态失效等。

Cache miss 可以由下一层 cache 满足，不一定到 DRAM。DRAM 访问慢也不代表出现 page fault；page 可以 resident，却不在 CPU cache。

还要区分：

- **Latency-bound:** 依赖链上的单次 memory access 很慢。
- **Bandwidth-bound:** 同时搬运的数据量达到可用 bandwidth。

增加 workers 对后者可能无益，甚至加剧竞争。

### 12.6 Cache vs TLB vs Page Cache

| Structure | 缓存什么 | 常见 miss 后果 |
| --- | --- | --- |
| **CPU data/instruction cache** | Memory 中的数据或 instructions | 从更低层 cache 或 memory 获取 |
| **TLB** | Address translations 和 permissions 等 | Page-table lookup/refill |
| **OS page cache** | File-backed contents 的 RAM copies | 可能需要 file I/O，取决于具体 operation |

这三种 cache 的容量、管理者和使用路径不同。不要从“cache miss”这个词直接猜测需要 disk I/O。

### 12.7 Coherence and False Sharing

Cache coherence 协调多个 cores 对同一 memory location 的 cached copies，使它们遵守硬件 coherence 规则。它不是 application-level transaction，也不保证多个变量间的程序顺序符合预期。

两个 threads 更新不同 variables，但它们位于同一 cache line 时，可能反复争取写权限，造成 **false sharing**：

```text
One cache line:
[ counter_A written by Core A | counter_B written by Core B ]
```

它可以是纯性能问题，而没有 data race。可以考虑 per-worker layout、适当 padding、批量 local updates；先测量，避免无谓增加 memory footprint。

Atomic operations、happens-before 与 memory ordering 的正确性规则留给 synchronization notes。

### 12.8 NUMA Awareness

在 NUMA 系统中，不同 CPUs 访问某些 memory nodes 的成本不同。即使全部 data 都 resident，remote memory access 也可能更慢。

Placement、first-touch policies、thread migration 和 workload pinning 都可能影响 locality，但具体 policy 依 OS/configuration。面试知道“RAM access cost 不一定对所有 CPUs 一样”即可；调优需要测量而非盲目绑定。

## 13. Memory Safety and Object Lifetime

### 13.1 Typical Bugs

| Bug | 发生什么 | 例子 |
| --- | --- | --- |
| **Memory leak** | 不再需要的 allocation 未被释放或仍被无意保留 | Request object 长期留在 global container |
| **Dangling pointer/reference** | Reference 的 target lifetime 已结束 | 保存了 function 内 local variable 的 address |
| **Use-after-free** | 通过旧 reference 访问已释放 allocation | free 后继续读 object |
| **Use-after-return** | Function 返回后继续用已失效的 local storage | 返回 local buffer address |
| **Double free** | 同一 allocation 被重复释放 | 两个 owners 都认为自己负责 cleanup |
| **Out-of-bounds access** | 超出 object/array 有效范围 | 写 array[size] |
| **Uninitialized read** | 在有效初始化前读取 value | 使用未初始化 local integer |
| **Stack overflow** | Stack 空间耗尽或越界 | 过深 recursion、巨大 automatic buffer |
| **Heap/resource exhaustion** | Allocation 无法继续满足 | Live set 太大、leak 或限制过小 |

这些问题可能 crash，也可能没有立即可见症状。在 C/C++ 中许多情况属于 undefined behavior，不能把“当前机器读到旧值”当成规定结果。

### 13.2 A Valid Mapping Does Not Prove a Valid Object

```cpp
// Deliberately invalid C++ fragment: do not execute.
int* p = new int(7);
delete p;
int value = *p;  // use-after-free
```

Free 后 allocator 可能仍保留该 page，hardware mapping 也可能仍允许 read。这不让 `*p` 变合法：object lifetime 已结束，memory 也可能马上被另一个 allocation 复用。

同理，array 越界后如果落在同一个 mapped page，MMU 未必能检测。OS 按 pages 保护，language objects 通常比 pages 小得多。

### 13.3 Ownership and RAII

Ownership 回答“谁负责何时销毁这个 resource”。清晰的 ownership 可以减少 leak 和重复释放。

C++ RAII 把 cleanup 绑定到 object lifetime，例如 scope 退出时自动析构：

```cpp
// C++ fragment
#include <memory>

void example() {
    auto value = std::make_unique<int>(42);
    // Scope exit releases the owned allocation.
}
```

- **unique_ptr:** 表达独占 ownership，可移动转交。
- **shared_ptr:** 通过 shared ownership 保持 pointee lifetime。
- **weak_ptr:** 不增加 owning reference，可用于避免部分 cycles。
- **Borrowed pointer/reference:** 不负责延长 target lifetime，caller 必须保证有效性。

两个 objects 相互持有 owning `shared_ptr` 可能形成 cycle，让 reference counts 无法归零。Smart pointers 也不自动提供 pointee 的 thread safety。

### 13.4 Garbage Collection

Tracing GC 通常从 roots 出发识别 reachable objects，回收 unreachable objects。Reference counting 跟踪 owning references，可能需要额外机制处理 cycles。

GC 解决部分 allocation lifetime 管理，但不知道哪些 reachable objects 在业务上“已经没用了”：

```text
Global root → cache → old request → large payload
```

只要这条引用链仍存在，GC 就有理由保留 payload。无上限 cache、忘记注销的 listeners、长期保存的 task results，都可能造成 logical memory leak。

GC 也不自动及时关闭 files/sockets，resource cleanup 通常仍应显式管理。Memory safety 也不等于 race-free 或业务正确。

### 13.5 Runnable Python: Reachability Keeps an Object Alive

```python
import gc
import weakref

class Payload:
    pass

cache = {}
item = Payload()
reference = weakref.ref(item)
cache["old_request"] = item

del item
gc.collect()
print("Retained by cache:", reference() is not None)

cache.clear()
gc.collect()
print("Collected after cleanup:", reference() is None)

# Retained by cache: True
# Collected after cleanup: True
```

Weak reference 用于观察 object 是否仍存在，不拥有它。这个示例说明 global/container reachability 的影响，不是测试 RSS 是否下降；object 回收与 allocator 归还 pages 是不同层次。

Python 的 GC 接口与 implementation-specific behavior 可参考 [gc documentation](https://docs.python.org/3/library/gc.html)。

### 13.6 Prevention and Detection

- 明确 ownership、borrowed references 和 scope。
- 使用 bounds-aware containers / APIs。
- 在检查 overflow 后再计算 allocation sizes，避免 `count * size` 溢出导致少分配。
- 将错误 paths 与正常 paths 的 cleanup 统一管理。
- 对复杂 native code 使用 sanitizer，并配合 targeted tests。

AddressSanitizer 可帮助发现多类越界和 use-after-free 等问题，但测试没有报错不证明所有 execution paths 都安全。它也不替代 race detector。[Clang AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)

## 14. Memory Metrics and Troubleshooting

### 14.1 Know What a Metric Measures

| Metric | 含义 | 常见误解 |
| --- | --- | --- |
| **Virtual size / VSZ** | Process virtual mappings 等占用的 address-space size，按工具定义 | 不是独占 RAM |
| **RSS** | 当前 resident mappings 的 memory 统计 | 不是全部私有，也不是 application live heap |
| **PSS** | 将共享 resident pages 的成本按 sharers 分摊 | 不是所有系统都提供此指标 |
| **Private resident memory** | 按工具定义归为 private 的 resident pages | 不等于全体已 commit private memory |
| **Peak RSS / high-water mark** | 曾达到的 resident 高点 | Free 后不会因为当前用量下降而回退 |
| **Allocator/runtime live bytes** | 对应 allocator/runtime 仍认为 live 的 allocations | 可能不覆盖 native libraries、stacks、mappings |
| **Committed memory** | 按 OS policy 承担 backing/accounting 的量 | 不等于当前 RAM 使用量 |

Linux `/proc/PID/smaps` 提供 mapping-level RSS/PSS 等统计；`status` 中某些 aggregate counters 为近似值，不能要求不同采样接口在并发变化时完全一致。[smaps](https://man7.org/linux/man-pages/man5/proc_pid_smaps.5.html)、[status](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html)

Windows 的 **working set** 是 process 当前 resident 的一组 pages；它与 Section 8 中按访问窗口定义的理论 working set 相关，但不是完全相同的统计定义。[Windows working set](https://learn.microsoft.com/en-us/windows/win32/memory/working-set)

### 14.2 Shared Pages and Double Counting

假设一个 process 有 10 MiB private resident pages，同时与另外一个 process 共享 20 MiB resident pages。

简化统计：

```text
RSS = 10 + 20 = 30 MiB
PSS = 10 + 20 / 2 = 20 MiB
```

若两个 processes 各有上述 10 MiB private，加起来：

- Sum of RSS = 60 MiB。
- 实际这组 pages 的唯一 physical footprint = 40 MiB。
- Sum of PSS = 40 MiB。

示例忽略其他 mappings 与 kernel overhead，只用于说明“把所有 process RSS 相加会重复计算共享部分”。

### 14.3 Diagnose Growing Memory

先观察相同 workload 下、多个周期的趋势，再判断：

| Observation | Hypothesis | Next check |
| --- | --- | --- |
| Live objects 和 RSS 持续增长 | Leak、无上限 cache、积压任务 | Allocation profiles、retaining references、queue length |
| Live objects 回落，RSS 高位稳定 | Allocator retention、fragmentation | Heap/allocator stats、mapping breakdown |
| Virtual size 增长但 RSS 变化小 | Reserved ranges、stacks、sparse mappings | Map sizes 与实际 touched pages |
| fork 后私有 RSS/PSS 逐渐增长 | COW pages 被写入 | Private dirty pages 与 worker行为 |
| Process footprint 正常但系统 pressure 高 | 其他 processes、kernel memory、container limit | 系统与 cgroup 层指标 |

“Memory 一直涨”必须先说清哪个 metric、时间窗口、请求量，以及是否最终稳定。

### 14.4 Diagnose Sudden Slowness

推荐分层排查：

1. **Page faults:** Minor/major rate 是否变化？是否涉及 cold file data？
2. **Reclaim and swap:** 是否有持续 swap I/O、direct reclaim 或 memory pressure？
3. **Working set:** 是否增大了 data 或同时活跃的 workers？
4. **CPU caches/TLB:** 是否出现新的 random access、large stride、translation pressure？
5. **Allocator/runtime:** 是否有 GC pauses、allocation churn、fragmentation 或 contention？
6. **Limits and NUMA:** 是否触及 container limits，或使用 remote memory？
7. **Workload controls:** 比较 cold/warm runs，并控制 input 和后台任务。

单凭“CPU 不高”不能证明程序没有 bottleneck；它可能在等待 faults、I/O 或 coordination。

### 14.5 Tools by Layer

以下是选择方向，不要求全部使用；commands 是参考，不会由笔记自动执行。

| Layer | Possible tools | 观察什么 |
| --- | --- | --- |
| Linux mappings/accounting | `/proc/PID/maps`、`smaps`、`smaps_rollup`、`pmap` | Regions、RSS/PSS、private/shared |
| Linux system pressure | `vmstat`、`free`、pressure metrics | 系统可用 memory、swap 与 stalls |
| Windows process/system | Task Manager、Resource Monitor、VMMap | Working set、commit、region types |
| Native memory bugs | AddressSanitizer、适用平台上的 Valgrind | Invalid accesses、leaks 等 |
| Language allocations | Python tracemalloc、runtime heap profilers | Allocation sites、live object growth |
| Hardware behavior | 平台支持的 performance counters/profilers | Cache/TLB misses、bandwidth 等 |

Python `tracemalloc` 追踪 Python allocator 所记录的 allocations，不能把它的数字当成整个 process RSS。Native libraries 的所有 allocations 不一定都被记录。[tracemalloc documentation](https://docs.python.org/3/library/tracemalloc.html)

### 14.6 A Useful Measurement Protocol

- 先记录 baseline 和 workload 参数。
- 每轮执行同一批工作，区分 warm-up 与 steady state。
- 记录 live allocations、resident memory、faults、throughput/latency。
- 检查对象是否仍被引用，再决定是否属于 leak。
- 一次改变一个主要因素，观察趋势。
- 只有确认瓶颈后，再尝试 allocator knobs、huge pages 或 NUMA tuning。

**不要用“强制 GC 后 RSS 没降”单独证明有 leak，也不要用“RSS 降了”单独证明所有 object lifetimes 都正确。**

## 15. Common Interview Questions

这部分用于 active recall。先口头回答，再用 reference 检查机制；不重复正文的完整解释。

### 15.1 Core Questions

| Question | Answer checkpoints | Reference |
| --- | --- | --- |
| **What is virtual memory?** | Abstraction、translation、protection、sharing；不是 swap 的同义词 | [Section 3](#3-virtual-memory) |
| **Can two processes use the same virtual address?** | Context-dependent mappings | [3.1](#31-virtual-vs-physical-address) |
| **What is the difference between a page and a frame?** | Virtual unit vs physical unit；注明 page size | [4.1](#41-pages-and-frames) |
| **How is an address translated?** | VPN、offset、PTE、PFN、permissions | [Section 4](#4-paging-and-address-translation) |
| **Why does the offset stay the same?** | Translation maps page-sized aligned regions | [4.2](#42-split-an-address) |
| **Why use multi-level page tables?** | Sparse ranges、按需分配 lower tables、walk trade-off | [Section 5](#5-page-tables) |
| **What does the TLB cache?** | Translations，不是 application contents | [6.1](#61-what-it-caches) |
| **Does a TLB miss mean disk I/O?** | 区分 refill、fault、backing I/O | [6.2](#62-hit-miss-and-page-walk) |
| **What is TLB shootdown?** | 修改 mappings 后协调相关 CPUs 的 stale entries | [6.5](#65-invalidation-and-tlb-shootdown) |
| **What happens on a page fault?** | 检查合法性、准备 backing、更新 mapping、resume 或 error | [Section 7](#7-page-faults-and-demand-paging) |
| **Does every page fault indicate a bug?** | Demand paging / COW 与非法访问不同 | [7.1](#71-a-fault-is-a-request-for-resolution-not-always-a-bug) |
| **Minor fault vs major fault?** | 是否需要 page-loading I/O，注明 OS 定义 | [7.4](#74-minor-vs-major-faults) |
| **What if physical memory runs out?** | Reclaimability、dirty/file/anonymous、swap、limits、OOM | [Section 8](#8-physical-memory-pressure-and-page-replacement) |
| **What is thrashing?** | Working set 与容量失衡，反复回收/恢复 | [8.3](#83-working-set-and-thrashing) |
| **LRU vs FIFO vs Clock?** | 决策规则、维护成本、Belady anomaly | [8.4](#84-replacement-algorithms) |
| **Does malloc always call the OS?** | Allocator reuse vs mapping acquisition | [9.1](#91-allocation-request-vs-os-mapping) |
| **Internal vs external fragmentation?** | Allocated block 内浪费 vs free holes 分散 | [9.4](#94-internal-vs-external-fragmentation) |
| **Why can RSS remain high after free?** | Reuse、partial pages、fragmentation、retention | [9.6](#96-why-free-does-not-necessarily-reduce-rss) |
| **Is mmap always faster than read?** | Pattern、copy、fault、mapping overhead | [10.3](#103-mapped-io-vs-readwrite) |
| **Can raw pointers be stored in shared memory?** | 不同 bases、offset representation、layout/lifetime | [10.6](#106-why-shared-memory-structures-often-use-offsets) |
| **How does copy-on-write work?** | 共享、write protection、fault、必要时复制 | [Section 11](#11-copy-on-write-cow) |
| **Why are sequential accesses often faster?** | Spatial locality、prefetch、dependency chains | [12.3](#123-temporal-and-spatial-locality) |
| **Cache vs TLB vs page cache?** | Data、translation、file contents | [12.6](#126-cache-vs-tlb-vs-page-cache) |
| **Can a GC language have a memory leak?** | Reachability 不等于业务需要 | [13.4](#134-garbage-collection) |
| **RSS vs PSS?** | Shared resident pages 的重复计算与分摊 | [14.2](#142-shared-pages-and-double-counting) |

### 15.2 Calculation Practice

先遮住答案计算，并说明每个 assumption：

| Problem | Answer |
| --- | --- |
| Page size 8 KiB，需要多少 offset bits？ | 13 bits，因为 8192 = 2^13 |
| 32-bit VA、4 KiB pages，一共有多少 virtual pages？ | 2^20 |
| 上题若每个 PTE 8 bytes，flat table 多大？ | 8 MiB |
| Page size 4 KiB，VA 0x23456，VPN 对应 PFN 0x7A，PA 是多少？ | VPN 0x23，offset 0x456，PA 0x7A456 |
| 理想 128-entry TLB、4 KiB pages，reach 多大？ | 512 KiB |
| 单独以完整 4 KiB pages 承载 9000 bytes，需要几页，尾部剩多少？ | 3 pages，12288 - 9000 = 3288 bytes |
| 12 MiB private RSS + 24 MiB 被 3 个 processes 共享，简化 RSS/PSS？ | RSS 36 MiB；PSS 20 MiB |

不要漏掉“每个 PTE 多大”“pages 是否必须整块独占”“共享者有几个”等题设。

### 15.3 Scenario: A Large Allocation Succeeds but Memory Usage Barely Changes

**Prompt:** A program allocates a large buffer, but RSS hardly increases. Is the allocation fake?

思考顺序：先确认 allocator/API 和 OS，再区分 address-space allocation、commit policy、physical residency，以及 pages 是否被实际 touched。

**Sample answer:**

> The allocation may have established usable virtual storage without making every page resident. The allocator may also be reusing existing mappings. Physical backing can be prepared on demand, so I would compare virtual size, commitment where applicable, resident memory, and the effect of touching the pages. Allocation success is not the same as immediate residency of the entire buffer.

Follow-up：如果随后写遍每页，为什么可能出现 faults 和 memory growth？如果只读取，zero-page optimization 是否可能改变结果？

### 15.4 Scenario: Memory Remains High After Cleanup

**Prompt:** All request objects were released, but the process still reports high memory usage. Is there a leak?

先确认“released”的证据、metric 是否为 peak、allocator 是否保留 free blocks，以及 workload 是否继续生成对象。

**Sample answer:**

> I would distinguish live objects from allocator-retained memory and resident pages. High RSS alone does not prove a leak. I would compare allocation profiles and mapping statistics across repeated workload cycles, check retaining references, and see whether memory stabilizes or keeps growing. Fragmentation and reuse caches can keep pages resident after objects are freed.

Follow-up：为什么强制 trim 可能降低 RSS，却增加下一轮 allocation/fault 成本？

### 15.5 Scenario: A Read-heavy Service Uses More Memory After fork

**Prompt:** Workers mostly read data, but memory grows after forking. Why?

检查 runtime metadata updates、实际 writes、page-level granularity，以及 private/shared accounting。

**Sample answer:**

> Copy-on-write avoids eager copying, but any actual writes can make pages private. A read-heavy workload may still modify reference counts, allocator metadata, or caches. I would inspect private dirty memory and the write paths rather than assume that application-level reads preserve all sharing. Summed RSS can also overcount shared pages, so PSS may provide a clearer view.

### 15.6 Scenario: More RAM Does Not Fix a Slow Loop

**Prompt:** A loop over resident data is slow, even though the machine has plenty of free RAM.

考虑 CPU caches、TLB reach、pointer-chasing dependency、memory bandwidth、NUMA 和 data layout，而不是立即增加 memory capacity。

**Sample answer:**

> Free RAM tells me about capacity, not the cost of each access. The loop may be limited by cache misses, address translation, dependent loads, or memory bandwidth. I would profile the access pattern and hardware counters, then consider layout and locality changes. I would first confirm that paging is not the bottleneck.

### 15.7 Common Statements to Challenge

| Statement | Missing distinction |
| --- | --- |
| “Virtual memory is disk used as RAM.” | Address abstraction vs backing policy |
| “A TLB miss is a page fault.” | Missing cached translation vs access that needs OS resolution |
| “A page fault always reads from disk.” | Minor faults、COW、demand-zero |
| “Continuous virtual memory requires continuous physical memory.” | Page-by-page mapping |
| “Every malloc performs a syscall.” | User-space allocator reuse |
| “Free immediately returns RAM to the OS.” | Object lifetime、allocator ownership、residency |
| “An accessible address means a valid C++ object.” | Page protection vs language-level lifetime |
| “Shared memory can use any pointer from either process.” | Different virtual bases |
| “mmap writes are immediately durable.” | Visibility、writeback、durability |
| “GC means leaks are impossible.” | Reachability vs usefulness |
| “More RAM eliminates cache misses.” | Capacity at different hierarchy levels |
| “Total physical usage equals the sum of all RSS values.” | Shared-page double counting |

### 15.8 Interview Answer Structure

推荐按以下顺序组织：

1. **State the layer:** 是 language object、allocator block、virtual page、physical frame，还是 CPU cache line？
2. **Explain the mechanism:** 谁维护 state，hardware/OS/runtime 各负责什么？
3. **Give a concrete example:** 一次 address translation、fault 或 allocation。
4. **Discuss cost:** Memory footprint、latency、copying、I/O 或 fragmentation。
5. **Qualify the platform:** 区分通用模型和具体 OS/language behavior。

能够把层次和条件说清楚，比只背“heap 慢、stack 快”“page fault 很严重”更适合应对 follow-up。
