# QEMU GSoC 2026: Fast Snapshot Load — 深度调研报告

## 目录

1. [项目概览](#1-项目概览)
2. [技术背景：QEMU 快照架构](#2-技术背景qemu-快照架构)
3. [核心依赖技术深度分析](#3-核心依赖技术深度分析)
4. [项目实现方案分析](#4-项目实现方案分析)
5. [代码地图：关键文件与函数](#5-代码地图关键文件与函数)
6. [历史相关工作](#6-历史相关工作)
7. [申请策略与建议](#7-申请策略与建议)

---

## 1. 项目概览

### 基本信息

| 项目 | 详情 |
|------|------|
| **项目名称** | Fast Snapshot Load |
| **简介** | Instantly launch VMs from snapshots by loading RAM on demand |
| **难度** | Advanced |
| **时长** | 350 小时 |
| **语言** | C |
| **导师** | Peter Xu <peterx@redhat.com> (IRC: peterx on #qemu) |
| **申请窗口** | 2026年3月16日 – 3月31日 |
| **编码期** | 2026年5月25日 – 8月24日 |

### 核心目标

当前 QEMU 加载快照 (loadvm) 时，**必须将所有 VM 数据（RAM + 设备状态）完全加载到内存后才能启动 VM**。对于拥有数十 GB 甚至数百 GB RAM 的虚拟机，这个过程极其耗时。

本项目的目标是：**在加载完设备状态后立即启动 VM，RAM 按需加载 (on-demand)**。利用 Linux 的 `userfaultfd(2)` 系统调用，当 vCPU 访问尚未加载的内存页时，QEMU 捕获缺页异常并从快照文件中按需加载对应页面。

### 应用场景

- **频繁加载快照**：如 fuzzing 场景（反复从同一快照启动）
- **大内存 VM 的快速恢复**：数百 GB RAM 的 VM 不再需要等待全部加载
- **开发/调试**：快速回到特定 VM 状态

> 注意：对于简单的 VM suspend/resume 场景，此功能帮助有限，因为这些场景下 VM 本来就是停止的，等待完全加载通常可以接受。

---

## 2. 技术背景：QEMU 快照架构

### 2.1 快照的两种方式

QEMU 提供两种加载快照的方式：

1. **QMP `snapshot-load`** / HMP `loadvm`：传统方式，快照存储在 QCOW2 磁盘镜像中
2. **QMP `migrate_incoming` + `file:` URI**：现代方式，快照作为独立的 migration 文件

本项目**聚焦于第二种方式**（file: migration），因为这是 QEMU 推荐的现代快照方式。

### 2.2 快照 = Migration to/from 文件

QEMU 的快照机制**复用了 live migration 基础设施**。核心思想：

```
快照保存 = 将 VM 状态 migrate 到文件
快照加载 = 从文件 migrate_incoming
```

这意味着快照使用完全相同的 VMState 序列化框架、RAM 保存/加载机制和设备状态导出/导入逻辑。

### 2.3 Save/Load 的数据流

**保存流程 (savevm)：**

```
save_snapshot() [savevm.c:3206]
  ├── 验证所有块设备支持快照
  ├── 暂停 VM
  ├── global_state_store()
  ├── qemu_savevm_state()
  │   ├── qemu_savevm_state_prepare()     → save_prepare handlers
  │   ├── qemu_savevm_state_do_setup()    → save_setup handlers + 写文件头
  │   ├── qemu_savevm_state_iterate()     → save_live_iterate (脏页)
  │   ├── qemu_savevm_state_complete_precopy()
  │   │   ├── 写入 RAM (剩余页)
  │   │   └── 写入所有设备状态 (SECTION_END)
  │   └── qemu_savevm_state_end()         → EOF 标记
  └── 恢复 VM 运行
```

**加载流程 (loadvm) —— 当前的瓶颈所在：**

```
load_snapshot() [savevm.c:3402]  /  qemu_loadvm_state()
  ├── qemu_loadvm_state_header()         → 验证文件头 (magic: 0x5145564d)
  ├── qemu_loadvm_state_setup()          → load_setup handlers
  ├── qemu_loadvm_state_main()           → 主加载循环
  │   ├── 解析 section type (SECTION_START/PART/END/FULL)
  │   ├── 对每个 section 调用对应设备的 load_state handler
  │   ├── 【瓶颈】RAM 加载：逐页读取所有 RAM 数据   ← 这里是优化目标
  │   └── 设备状态加载
  └── Post-load 同步
      └── VM 启动
```

### 2.4 Migration Stream 格式

```
┌──────────────────────────────────────────────┐
│ File Header: Magic (0x5145564d) + Version(3) │
├──────────────────────────────────────────────┤
│ SECTION_START (0x01) - Configuration         │
│ SECTION_START (0x01) - RAM                   │
│   ├── RAMBlock headers                       │
│   └── RAM pages (逐页存储)                    │
│ SECTION_PART (0x02) - RAM (迭代更新)          │
│ SECTION_END (0x03) - RAM                     │
│ SECTION_FULL (0x04) - Device A state         │
│ SECTION_FULL (0x04) - Device B state         │
│ ...更多设备...                                │
│ EOF (0x00)                                   │
└──────────────────────────────────────────────┘
```

### 2.5 VMState 框架

设备通过 `VMStateDescription` 结构体声明其可序列化状态：

```c
// include/migration/vmstate.h
struct VMStateDescription {
    const char *name;
    int version_id;
    int minimum_version_id;
    MigrationPriority priority;       // 序列化优先级
    int (*pre_load)(void *opaque);    // 加载前回调
    int (*post_load)(void *opaque, int version_id);  // 加载后回调
    int (*pre_save)(void *opaque);
    const VMStateField *fields;       // 字段定义数组
    const VMStateDescription **subsections;
};
```

设备通过 `vmstate_register()` 注册，最终形成 `SaveStateEntry` 链表 (`savevm_state.handlers`)。

### 2.6 RAM 序列化

RAM 作为一种特殊的 `SaveStateEntry` 注册 (`is_ram=1`)，使用专门的 `SaveVMHandlers`：

```c
// migration/ram.c
static SaveVMHandlers savevm_ram_handlers = {
    .save_setup = ram_save_setup,
    .save_live_iterate = ram_save_live_iterate,
    .save_complete = ram_save_complete,
    .load_setup = ram_load_setup,
    .load_state = ram_load_precopy,     // ← 当前的加载函数（同步、阻塞）
    .load_cleanup = ram_load_cleanup,
    ...
};
```

**关键 RAM 保存标志：**

| 标志 | 值 | 含义 |
|------|-----|------|
| `RAM_SAVE_FLAG_ZERO` | 0x002 | 零填充页 |
| `RAM_SAVE_FLAG_PAGE` | 0x008 | 普通数据页 |
| `RAM_SAVE_FLAG_XBZRLE` | 0x040 | XOR 增量压缩 |
| `RAM_SAVE_FLAG_MULTIFD_FLUSH` | 0x200 | Multifd 同步 |

---

## 3. 核心依赖技术深度分析

### 3.1 Mapped-RAM 特性 ⭐ (最关键的依赖)

**这是 Fast Snapshot Load 的基石。** 传统 migration stream 中，RAM 页是顺序写入的，无法随机访问。Mapped-RAM 改变了这一点。

#### 核心思想

Mapped-RAM 确保**每个 RAM 页在文件中有固定的偏移量 (fixed offset)**，使得可以直接 seek 到任意页面的位置进行读取。

#### 文件布局对比

```
传统格式 (Sequential):
┌────────────┬──────────────┬──────────────┬─────┬──────────┐
│ RAMBlock   │ Page Stream  │ Page Stream  │ ... │ Section  │
│ Headers    │ (iteration 1)│ (iteration 2)│     │ End      │
└────────────┴──────────────┴──────────────┴─────┴──────────┘
  页面位置不确定，多次迭代中同一页可能出现多次

Mapped-RAM 格式:
┌──────────────────────────────────────────────────────┐
│ RAMBlock A Header + MappedRamHeader                  │
│   ├── Bitmap (哪些页被写入)                           │
│   ├── Bitmap size                                    │
│   └── Pages offset                                   │
├── [Padding to 1MB alignment]                         │
├── Page 0 (固定偏移)                                   │
├── Page 1 (固定偏移)                                   │
├── Page 2 (固定偏移)                                   │
├── ...                                                │
├──────────────────────────────────────────────────────┤
│ RAMBlock B Header + MappedRamHeader                  │
│   └── (同上)                                         │
├──────────────────────────────────────────────────────┤
│ Device States                                        │
│ EOF                                                  │
└──────────────────────────────────────────────────────┘
```

#### 关键代码

```c
// migration/ram.c

// 设置 mapped-ram RAMBlock（保存时）
static void mapped_ram_setup_ramblock(QEMUFile *file, RAMBlock *block) {
    // 写入 MappedRamHeader: bitmap + bitmap_size + pages_offset
    // 对齐到 1MB 边界（支持 O_DIRECT）
}

// 写入普通页到固定偏移
if (migrate_mapped_ram()) {
    qemu_put_buffer_at(file, buf, TARGET_PAGE_SIZE,
                       block->pages_offset + offset);  // ← 固定位置写入
}

// 加载时读取 mapped-ram
static bool read_ramblock_mapped_ram(QEMUFile *f, RAMBlock *block,
                                     long num_pages, unsigned long *bitmap,
                                     Error **errp);
```

#### 为什么是 Fast Snapshot Load 的必要条件

- **随机访问**：当 vCPU 访问某个未加载的页面时，QEMU 需要直接 seek 到该页在文件中的位置。传统顺序流做不到这一点。
- **Bitmap 追踪**：通过 bitmap 可以知道哪些页面存在于文件中（非零页），哪些是零页可以直接填零。
- **O_DIRECT 支持**：1MB 对齐允许绕过页缓存，提高 I/O 性能。

### 3.2 Postcopy Live Migration (参考实现)

Postcopy 是 Fast Snapshot Load 的**概念原型**。二者的核心思想完全一致：先启动 VM，再按需加载内存。

#### Postcopy 状态机 (目标端)

```
POSTCOPY_ADVISE     检查 OS 支持 (userfaultfd)，准备 RAM 映射
       ↓
POSTCOPY_DISCARD    禁用 hugepages
       ↓
POSTCOPY_LISTEN     启动监听线程接收页面，注册 userfaultfd
       ↓
POSTCOPY_RUN        启动 vCPU 和 IO 设备
       ↓
POSTCOPY_END        清理
```

#### userfaultfd 工作原理

```
1. QEMU 通过 userfaultfd(2) 创建一个文件描述符
2. 注册需要监控的内存区域 (UFFDIO_REGISTER)
3. 当 vCPU 访问未映射的页面 → 触发 page fault
4. 内核通过 userfaultfd 通知 QEMU (UFFD_EVENT_PAGEFAULT)
5. QEMU 的 fault handler 线程：
   a. 从源端/文件中获取对应页面数据
   b. 通过 UFFDIO_COPY 将页面数据复制到目标地址
   c. vCPU 恢复执行
```

#### 关键代码路径

```c
// migration/postcopy-ram.c

// 设置 userfaultfd 和注册内存区域
int postcopy_ram_incoming_setup(MigrationIncomingState *mis) {
    // 1. 创建 userfaultfd
    // 2. 对每个 RAMBlock 注册 UFFDIO_REGISTER
    // 3. 启动 fault handler 线程
}

// 处理缺页异常
// postcopy_ram_fault_thread() → 读取 userfaultfd 事件 → 请求页面

// 将页面放入目标地址
// qemu_ufd_copy_ioctl() → UFFDIO_COPY / UFFDIO_ZEROPAGE
```

### 3.3 File Migration (file: URI)

`file:` 是现代 QEMU 推荐的快照方式：

```bash
# 保存快照
{"execute": "migrate", "arguments": {
    "uri": "file:/path/to/snapshot,offset=0x11921"
}}

# 加载快照
{"execute": "migrate-incoming", "arguments": {
    "uri": "file:/path/to/snapshot,offset=0x11921"
}}
```

**关键特性：**
- 需要 seekable channel（`QIO_CHANNEL_FEATURE_SEEKABLE`）
- 支持 `mapped-ram` 和 `multifd`
- 文件描述符通过 `fdset` 传递
- 支持 `O_DIRECT` 模式

---

## 4. 项目实现方案分析

### 4.1 整体架构设计

```
当前 loadvm 流程:
─────────────────────────────────────────────
[加载设备状态] → [加载全部 RAM] → [启动 VM]
                  ^^^^^^^^^^^^^^^^
                  耗时瓶颈 (可能数分钟)

Fast Snapshot Load 流程:
─────────────────────────────────────────────
[加载设备状态] → [注册 userfaultfd] → [启动 VM] → [后台 + 按需加载 RAM]
                                       ^^^^^^^^^^
                                       几乎即时
```

### 4.2 详细实现步骤

#### 第一阶段：基础设施

1. **扩展 `migrate_incoming` 支持 postcopy-from-file 模式**
   - 添加新的 migration capability 或参数
   - 在 `file:` URI 的 incoming 路径中支持 postcopy 行为

2. **确保 mapped-ram 与 postcopy 的兼容性**
   - 当前 mapped-ram 和 postcopy 可能未协同工作
   - 需要确保 mapped-ram 的 bitmap 和偏移计算在 postcopy 模式下正确

#### 第二阶段：核心实现

3. **实现 "postcopy from file" 页面加载器**
   - 复用 `postcopy_ram_incoming_setup()` 注册 userfaultfd
   - 实现新的 fault handler：
     ```
     收到缺页事件 (fault_address)
       ↓
     计算对应的 RAMBlock + offset
       ↓
     通过 mapped-ram bitmap 检查页面是否存在
       ↓
     如果存在：seek 到文件偏移位置，读取页面，UFFDIO_COPY
     如果不存在（零页）：UFFDIO_ZEROPAGE
     ```

4. **后台预加载线程**
   - 不仅按需加载，还应该在后台预先加载页面
   - 类似 postcopy 中的 "background page sending"
   - 顺序扫描 bitmap，预加载未命中的页面

#### 第三阶段：优化与完善

5. **性能优化**
   - 批量读取：一次读取多个连续页面
   - 预读策略：根据访问模式预测下一个需要的页面
   - 多线程加载：利用 multifd 线程并行读取

6. **完成状态管理**
   - 所有 RAM 加载完成后，注销 userfaultfd
   - 正确处理错误和回退

### 4.3 关键技术挑战

#### 挑战 1：设备状态与 RAM 的依赖关系

设备的 `post_load` 回调可能访问 guest RAM。在 postcopy live migration 中，这通过将设备状态"打包"成 blob 来处理——先完整读取设备状态数据，再执行设备加载，期间的 RAM 访问通过 userfaultfd 处理。

**Fast Snapshot Load 需要相同的处理**：确保设备状态加载期间的 RAM 缺页能被正确拦截和解决。

#### 挑战 2：文件 I/O vs 网络 I/O

Postcopy 从网络接收页面，Fast Snapshot Load 从文件读取。差异：
- 文件 I/O 是同步可 seek 的（更简单）
- 不需要请求-响应协议（直接读取）
- 但需要处理 O_DIRECT 的对齐要求
- 文件 I/O 延迟通常远低于网络，但对于 NFS/远程存储可能不同

#### 挑战 3：BQL (Big QEMU Lock) 管理

Peter Xu 的 "Threadify loadvm" RFC 指出，loadvm 路径当前在主线程中以协程方式运行。Fast Snapshot Load 的 fault handler 需要在独立线程中运行，必须仔细处理 BQL。

**当前状态**：Peter Xu 已提交将 loadvm 从协程迁移到线程的 RFC，这个工作可能是 Fast Snapshot Load 的前置条件。

#### 挑战 4：Multifd 集成

当前 postcopy 和 multifd 不兼容。如果要利用多线程并行加载页面，需要解决这个问题或找到替代方案。

### 4.4 参考：Postcopy 页面加载的数据流

```c
// 这是 live migration postcopy 的流程，Fast Snapshot Load 需要类似实现

// 1. Fault handler 线程检测到缺页
postcopy_ram_fault_thread() {
    while (poll(userfaultfd)) {
        read(userfaultfd, &msg);  // 获取 fault 事件
        // msg.arg.pagefault.address → fault 地址

        // 2. 转换为 RAMBlock + offset
        rb = qemu_ram_block_from_host(fault_addr, &rb_offset);

        // 3. 请求页面（live migration 从源端请求，
        //    Fast Snapshot Load 直接从文件读取）
        // postcopy: 发送请求到源端
        // snapshot: seek + read 从 mapped-ram 文件

        // 4. 将页面放入内存
        qemu_ufd_copy_ioctl(userfaultfd, host_addr, page_data);
    }
}
```

---

## 5. 代码地图：关键文件与函数

### 5.1 快照核心

| 文件 | 行数 | 作用 |
|------|------|------|
| `migration/savevm.c` | ~3721 | 快照 save/load 主逻辑 |
| `migration/ram.c` | ~4785 | RAM 序列化/反序列化 |
| `migration/migration.c` | ~3933 | Migration 控制器 |
| `migration/vmstate.c` | ~712 | VMState 字段序列化 |
| `migration/postcopy-ram.c` | ~2200+ | Postcopy RAM 处理 |

### 5.2 关键函数

**快照入口：**
- `save_snapshot()` — savevm.c:3206
- `load_snapshot()` — savevm.c:3402
- `qemu_savevm_state()` — savevm.c
- `qemu_loadvm_state()` — savevm.c

**RAM 处理：**
- `ram_save_setup()` — ram.c
- `ram_save_live_iterate()` — ram.c
- `ram_load_precopy()` — ram.c (当前的 RAM 加载)
- `ram_load_postcopy()` — ram.c:3816 (postcopy 模式加载)

**Mapped-RAM：**
- `mapped_ram_setup_ramblock()` — ram.c:3026
- `mapped_ram_read_header()` — ram.c:3068
- `read_ramblock_mapped_ram()` — ram.c:4089
- `handle_zero_mapped_ram()` — ram.c:4056
- `ram_save_file_bmap()` — ram.c

**Postcopy：**
- `postcopy_ram_incoming_setup()` — postcopy-ram.c:1520
- `postcopy_ram_fault_thread()` — postcopy-ram.c
- `qemu_ufd_copy_ioctl()` — postcopy-ram.c

**File Migration：**
- `file_start_incoming_migration()` — migration/file.c
- `file_start_outgoing_migration()` — migration/file.c

### 5.3 关键数据结构

```c
// 设备状态注册
struct SaveStateEntry {
    char idstr[256];
    int version_id;
    int section_id;
    const SaveVMHandlers *ops;
    const VMStateDescription *vmsd;
    void *opaque;
    bool is_ram;
};

// RAM 状态
struct RAMState {
    PageSearchStatus pss[RAM_CHANNEL_MAX];
    // 脏页 bitmap、统计信息等
};

// RAMBlock
struct RAMBlock {
    uint8_t *host;          // 宿主内存地址
    ram_addr_t used_length;
    unsigned long *bmap;     // 脏页 bitmap
    unsigned long *file_bmap; // mapped-ram 文件 bitmap
    off_t pages_offset;      // mapped-ram 页面偏移
    // ...
};

// Migration 入口状态
struct MigrationIncomingState {
    QEMUFile *from_src_file;
    int userfault_fd;        // userfaultfd 描述符
    QEMUFile *postcopy_qemufile_dst;
    // ...
};
```

### 5.4 Mapped-RAM Header 结构

```c
// migration/ram.c:3024 附近
typedef struct MappedRamHeader {
    // bitmap: 记录哪些页面被写入文件
    // bitmap_size: bitmap 大小
    // pages_offset: 页面数据在文件中的起始偏移
} MappedRamHeader;
```

---

## 6. 历史相关工作

### 6.1 2016 年 RFC: Live Memory Snapshot Based on userfaultfd

**作者：** Hailiang Zhang (zhanghailiang)

这是最早将 userfaultfd 用于快照的尝试，包含两个方向：
- **快照保存优化**：使用 userfaultfd write-protect 实现写时复制
- **快照加载优化**：使用 userfaultfd 实现按需加载

对于快照加载，其方案是：
1. 保存时记录每个页面在文件中的位置（类似 mapped-ram 的功能）
2. 加载时只加载设备状态和页面位置数组
3. 将所有页面设为 MISS 状态
4. VM 启动后，像 postcopy incoming 一样处理缺页

**未被合入的原因**：当时缺少 mapped-ram 特性，无法高效地随机访问快照文件中的页面。

### 6.2 2021 年 RFC: External Snapshot Utility

提出将 migration stream 拆分为 RAM 页面和设备状态两部分，以支持异步快照恢复。同样因为缺少 mapped-ram 而受限。

### 6.3 2024-2025: Peter Xu 的 Threadify Loadvm RFC

**关键前置工作**：将 loadvm 从协程迁移到独立线程，为 Fast Snapshot Load 奠定基础。

主要变更：
- loadvm 不再在主线程的协程中运行
- 大部分 loadvm 路径不再需要 BQL
- 使用更细粒度的锁管理

### 6.4 Mapped-RAM 特性 (已合入)

mapped-ram 是近期合入的重要特性，使 RAM 页在文件中有固定偏移，这是 Fast Snapshot Load 的**必要前提条件**。

---

## 7. 申请策略与建议

### 7.1 申请前准备

#### 必须掌握的技术

1. **C 语言精通**：面试包含 30 分钟 C 编码测试
2. **Linux 系统编程**：
   - `userfaultfd(2)` 系统调用
   - `mmap`, `madvise`, `mprotect`
   - 文件 I/O (`pread`, `O_DIRECT`)
   - 线程 (`pthread`)
3. **QEMU 内部机制**：
   - Migration 子系统
   - VMState 框架
   - QEMUFile I/O 抽象
   - 主事件循环和 BQL

#### 推荐的学习路径

```
第 1 周：理解基础
├── 编译 QEMU，运行简单 VM
├── 使用 savevm/loadvm 体验快照
├── 使用 file: migration 方式做快照
└── 阅读 migration 文档

第 2 周：深入代码
├── 阅读 savevm.c 的 save/load 流程
├── 阅读 ram.c 的 RAM 序列化
├── 理解 mapped-ram 的实现
└── 阅读 postcopy-ram.c 的 userfaultfd 使用

第 3 周：动手实验
├── 写 userfaultfd 的小型 demo 程序
├── 用 GDB 跟踪 loadvm 流程
├── 测量当前 loadvm 的性能瓶颈
└── 向 QEMU 提交一个小补丁 (建立社区联系)

第 4 周：撰写 Proposal
├── 与 Peter Xu 邮件沟通项目细节
├── 在 #qemu-gsoc IRC 频道提问
└── 撰写详细的实现方案
```

### 7.2 Proposal 建议包含的内容

1. **问题陈述**：当前 loadvm 的性能瓶颈分析（最好有实际测量数据）
2. **方案设计**：
   - 如何复用 postcopy userfaultfd 基础设施
   - 如何与 mapped-ram 集成
   - Fault handler 的详细设计
   - 后台预加载策略
3. **实现计划**（按周分解）：
   - 第 1-3 周：基础设施和 mapped-ram + postcopy 兼容性
   - 第 4-7 周：核心 fault handler 和按需加载
   - 第 8-10 周：后台预加载和优化
   - 第 11-12 周：测试、文档和性能评估
4. **测试计划**：
   - 单元测试（qtest 框架）
   - 性能对比测试
   - 边界情况处理
5. **风险评估**：识别可能的阻碍和备选方案

### 7.3 加分项

- **提交过 QEMU 补丁**（哪怕很小，如修复 typo 或改进文档）
- **实际测量** loadvm 性能数据（不同 RAM 大小、不同存储后端）
- **Prototype**：哪怕是最简单的 PoC 也能大大增强 proposal 的说服力
- **与导师的沟通记录**：展示你已经与 Peter Xu 讨论过项目

### 7.4 联系方式

- **导师**：Peter Xu <peterx@redhat.com> (IRC: peterx on #qemu)
- **GSoC 协调人**：Stefan Hajnoczi (stefanha@gmail.com)
- **IRC 频道**：#qemu-gsoc (irc.oftc.net)
- **邮件列表**：qemu-devel@nongnu.org

### 7.5 关键时间节点

| 日期 | 事项 |
|------|------|
| 现在 – 3月15日 | 深入学习代码，联系导师，提交小补丁 |
| 3月16日 – 3月28日 | 撰写并迭代 proposal |
| 3月28日之前 | 提交 proposal（Google 建议至少提前 3 天） |
| 3月31日 18:00 UTC | 申请截止 |
| 4月15日 | 贡献任务截止 |
| 4月30日 | 公布录取结果 |
| 5月25日 | 编码期开始 |

---

## 附录：参考资料

### 官方文档
- [QEMU Migration Framework](https://www.qemu.org/docs/master/devel/migration/main.html)
- [Mapped-RAM Documentation](https://www.qemu.org/docs/master/devel/migration/mapped-ram.html)
- [Postcopy Documentation](https://www.qemu.org/docs/master/devel/migration/postcopy.html)
- [QEMU GSoC 2026 公告](https://www.qemu.org/2026/02/20/gsoc-2026/)
- [QEMU GSoC 2026 项目列表](https://wiki.qemu.org/Google_Summer_of_Code_2026)

### 深度技术文章
- [A Deep Dive into QEMU: Snapshot API](https://airbus-seclab.github.io/qemu_blog/snapshot.html)
- [QEMU Wiki: Snapshotting Improvements](https://wiki.qemu.org/Features/SnapshottingImprovements)
- [QEMU Wiki: Features/Snapshots](https://wiki.qemu.org/Features/Snapshots)

### 邮件列表讨论
- [Peter Xu: Fast Snapshot Load GSoC Idea](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161673.html)
- [Peter Xu: Threadify Loadvm RFC](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1138300.html)
- [Hailiang Zhang: Live Memory Snapshot RFC (2016)](https://lists.gnu.org/archive/html/qemu-devel/2016-01/msg00664.html)

### 源代码关键路径
- `migration/savevm.c` — 快照 save/load 逻辑
- `migration/ram.c` — RAM 序列化 + mapped-ram
- `migration/postcopy-ram.c` — Postcopy userfaultfd 处理
- `migration/migration.c` — Migration 状态机
- `migration/file.c` — File migration channel
- `include/migration/vmstate.h` — VMState 框架定义
- `migration/register.h` — SaveVMHandlers 定义
