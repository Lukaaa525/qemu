# QEMU GSoC 2026: Fast Snapshot Load — 完整调研报告

## 一、项目概览

| 项目 | 详情 |
|------|------|
| **项目名称** | Fast Snapshot Load |
| **一句话概要** | "Instantly launch VMs from snapshots by loading RAM on demand" |
| **难度** | Advanced |
| **时长** | 350 小时 |
| **编程语言** | C |
| **导师** | Peter Xu (peterx@redhat.com) |
| **Wiki 页面** | https://wiki.qemu.org/Google_Summer_of_Code_2026 |
| **技术细节页** | https://wiki.qemu.org/ToDo/LiveMigration#Fast_load_snapshot |

---

## 二、项目描述

### 2.1 问题背景

当前 QEMU 的 `loadvm`（加载快照）过程是 **同步阻塞** 的：在 VM 启动之前，**所有** VM 数据（RAM 状态 + 设备状态）必须全部加载到 QEMU 实例中。对于拥有大内存的 VM（数十到数百 GiB），这个加载过程可能非常耗时。

### 2.2 核心思路

VM 启动 **并不需要** 先加载所有 guest 内存。类似于 postcopy live migration 的技术可以应用到 loadvm 过程中：

1. **先加载设备状态**（不加载 guest 内存）
2. **立即启动 VM**
3. **后台持续加载** RAM 数据
4. 当 vCPU 访问尚未加载的 guest 页面时，使用 **userfaultfd** 捕获缺页异常并按需解析

### 2.3 依赖特性

该项目依赖 **mapped-ram** 特性。mapped-ram 使得 RAM 页面在快照文件中具有 **固定偏移量**，从而可以在任意时间点随机访问快照中的任意页面，满足 vCPU 按需请求页面的需求。

### 2.4 适用场景

- **高频快照加载**：如 fuzzing 场景，一个快照被恢复数千次
- 对传统 VM suspend/resume 场景帮助有限（用户通常可以接受等待完整加载）

---

## 三、社区交流线程

### 3.1 GSoC 项目提案邮件线程

**线程**: ["Call for GSoC internship project ideas"](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161673.html)

**时间线**:

| 日期 | 发件人 | 内容 |
|------|--------|------|
| 2026-01-05 | Stefan Hajnoczi | 发起 GSoC 2026 项目征集，邀请 QEMU 贡献者在 1 月 30 日前提交项目提案 |
| 2026-01-13 | [Peter Xu](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161899.html) | 提出 "Fast Snapshot Load" 项目，CC 了 Marco Cavenati（fuzzing 用户视角） |
| 2026-01-14 | [Marco Cavenati](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161925.html) | 表达热情支持，指出 fuzzing 场景的强烈需求，提出可进一步使用 dirty tracking 优化同一快照的重复加载 |

**Peter Xu 的原始提案要点**:

> Load snapshot currently requires all VM data (RAM states and the rest device
> states) to be loaded into the QEMU instance before VM starts. It is not
> required, though, to load guest memory to start the VM.
>
> A similar technique (to postcopy live migration) can be used in a loadvm
> process to make loadvm very fast, starting the VM almost immediately right
> after the loadvm command. The idea is to start the VM right after loading
> device states (without loading the guest memory).
>
> In the background, the loadvm process should keep loading all the VM data
> atomically. Meanwhile, the vCPUs may access a missing guest page, and QEMU
> needs to trap these accesses with userfaultfd and resolve the page faults.
>
> This feature needs to depend on the mapped-ram feature, which allows
> offsetting into the snapshots to find whatever page being asked by the guest
> vCPUs at any point in time.

**Marco Cavenati 的回复要点**:
- 表示 Peter 提出的方案"不需要新的 QMP 命令或大量配置（只需 mapped-ram）"
- 提到可利用 dirty tracking 跳过两次相同快照加载间未被修改的页面
- 主要使用场景：fuzzing 短生命周期 VM，一个快照恢复数千次

### 3.2 前置工作：mapped-ram 与 snapshot 的兼容性

**线程**: ["migration: add FEATURE_SEEKABLE to QIOChannelBlock"](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1113922.html)

这是 Peter Xu 和 Marco Cavenati 之间关于快照性能优化的技术讨论：

- **Marco 的使用场景**: 开发 fuzzing 框架，创建一个快照后恢复数千次，VM 每次只运行少量函数
- **Marco 的实现**: 自定义 HMP 命令 (`loadvm_for_hotreload`, `hotreload`) 利用脏页追踪实现选择性恢复
- **Peter 的建议**: 使用 `memory-backend-file` + `share=off` + MAP_PRIVATE 实现 CoW 语义，加速快照恢复

**相关 Patch 系列**:
- [PATCH v2 0/2: migration: Add support for mapped-ram with snapshots](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1143987.html) — Marco Cavenati 提交
- [PULL 05/36: migration: mapped-ram: handle zero pages](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1151366.html) — 已合并

### 3.3 GSoC 2026 官方公告

**博客**: [Announcing QEMU Google Summer of Code 2026 internships](https://www.qemu.org/2026/02/20/gsoc-2026/)

- 申请窗口：**2026 年 3 月 16 日 — 3 月 31 日**
- 联系方式：Stefan Hajnoczi 或 IRC #qemu-gsoc (OFTC)
- 编码期：2026 年 5 月 25 日 — 8 月 24 日

---

## 四、关键技术分析（基于源码）

### 4.1 当前 snapshot load 流程

当前快照加载的完整调用链（`migration/savevm.c`）：

```
load_snapshot()                          [savevm.c:3402]
  ├─ migrate_can_snapshot()              // 检查迁移兼容性
  ├─ bdrv_all_can_snapshot()             // 检查所有块设备是否支持快照
  ├─ bdrv_all_has_snapshot()             // 确认快照存在
  ├─ bdrv_all_find_vmstate_bs()          // 定位存储 VM 状态的块设备
  ├─ bdrv_snapshot_find()                // 查找快照元数据
  ├─ replay_flush_events()               // 清空 record/replay 队列
  ├─ bdrv_drain_all_begin()              // ★ 阻塞：flush 所有 IO
  ├─ bdrv_all_goto_snapshot()            // ★ 耗时：恢复所有磁盘快照
  ├─ qemu_fopen_bdrv()                   // 打开 VM 状态文件
  ├─ qemu_system_reset()                 // 完全 VM 重置
  ├─ qemu_loadvm_state()                 // ★★★ 核心加载过程（见下）
  │   ├─ qemu_loadvm_state_header()      // 验证 magic number 和版本
  │   ├─ qemu_loadvm_state_setup()       // 设备 load_setup 回调
  │   ├─ cpu_synchronize_all_pre_loadvm() // CPU 状态同步
  │   ├─ qemu_loadvm_state_main()        // ★★ 主加载循环（处理所有 section）
  │   │   ├─ SECTION_START/FULL → 加载设备状态
  │   │   ├─ SECTION_PART/END → 处理迭代 section
  │   │   └─ RAM section → ram_load_precopy() ★★ 最大瓶颈
  │   ├─ qemu_loadvm_thread_pool_wait()  // 等待加载线程
  │   └─ cpu_synchronize_all_post_init() // 最终 CPU 同步
  ├─ migration_incoming_state_destroy()
  └─ bdrv_drain_all_end()
```

**关键瓶颈**: `ram_load_precopy()` 同步加载全部 RAM，这是整个过程最耗时的部分。

### 4.2 RAM 加载：Precopy vs Postcopy

**Precopy 模式** (`migration/ram.c:ram_load_precopy`):
- 按顺序从流中读取所有 RAM 页面
- 直接放置到 guest 内存中
- **同步阻塞**，VM 不能启动直到全部完成

**Postcopy 模式** (`migration/ram.c:ram_load_postcopy`):
- VM 先启动，RAM 按需加载
- 接收的页面先放入临时缓冲区（`PostcopyTmpPage`）
- 通过 UFFDIO_COPY/UFFDIO_ZEROPAGE **原子** 放置到内存中
- 缺页异常通过 userfaultfd 机制处理

**Fast Snapshot Load 正是要将 postcopy 的按需加载理念应用到 loadvm 中。**

### 4.3 Postcopy 的 userfaultfd 机制

**关键代码路径** (`migration/postcopy-ram.c`):

```
postcopy_ram_incoming_setup()            [postcopy-ram.c:1520]
  ├─ uffd_open(O_CLOEXEC | O_NONBLOCK)  // 打开 userfaultfd
  ├─ ufd_check_and_apply()               // API 握手
  ├─ 创建 eventfd                         // 线程通信
  ├─ 启动 postcopy_ram_fault_thread      // ★ 缺页处理线程
  └─ ram_block_enable_notify()            // UFFDIO_REGISTER 注册内存区域

postcopy_ram_fault_thread()              [postcopy-ram.c:1275]
  ├─ poll() 监控:
  │   ├─ userfault_fd (主 UFFD)
  │   ├─ userfault_event_fd (关闭信号)
  │   └─ 共享内存 UFDs
  ├─ 读取 struct uffd_msg                 // 从 kernel 获取缺页信息
  ├─ 验证地址在 guest RAM 范围内
  └─ postcopy_request_page()             // 请求缺失页面

qemu_ufd_copy_ioctl()                   [postcopy-ram.c:1587]
  ├─ uffd_copy_page()  → UFFDIO_COPY    // 非零页面
  └─ uffd_zero_page()  → UFFDIO_ZEROPAGE // 零页面
```

### 4.4 mapped-ram 文件格式

**文档**: https://www.qemu.org/docs/master/devel/migration/mapped-ram.html

mapped-ram 是该项目的 **必要前提**，因为它使得 RAM 页面在文件中有固定偏移量，支持随机访问：

```
+-------------------+
| RAM Block Header  |
+-------------------+
| Bitmap (脏页位图)  |  ← header->bitmap_offset
+-------------------+
| Padding (1MB对齐)  |
+-------------------+
| Page Data         |  ← header->pages_offset
| (固定偏移量页面)    |  ← 每个页面位置 = pages_offset + page_nr * page_size
+-------------------+
```

**关键数据结构** (`migration/ram.c`):
```c
struct MappedRamHeader {
    uint32_t version;        // Version 1
    uint64_t page_size;      // TARGET_PAGE_SIZE
    uint64_t bitmap_offset;  // 脏页位图位置
    uint64_t pages_offset;   // 页面数据起始位置
};
```

**核心优势**:
- **随机访问**: 页面在文件中有固定位置，可随时跳转读取
- **支持 O_DIRECT**: 对齐要求满足直接 IO
- **有界文件大小**: 同一页面多次写入只占固定位置
- **与 multifd 兼容**: 支持多线程并行读写

### 4.5 关键源码文件列表

| 文件 | 用途 |
|------|------|
| `migration/savevm.c` | 主要 savevm/loadvm 实现 (~3500 行) |
| `migration/savevm.h` | 格式常量（magic, version, section types） |
| `migration/vmstate.c` | VMState 序列化/反序列化引擎 |
| `migration/ram.c` | RAM 保存/加载，含 mapped-ram 格式 (~4700 行) |
| `migration/postcopy-ram.c` | Postcopy RAM 处理，userfaultfd 机制 (~2239 行) |
| `migration/postcopy-ram.h` | Postcopy API 和状态定义 |
| `migration/migration.h` | MigrationState, MigrationIncomingState 结构体 |
| `include/migration/snapshot.h` | 公共快照 API |
| `util/userfaultfd.c` | 底层 userfaultfd 系统调用封装 |
| `include/qemu/userfaultfd.h` | userfaultfd 公共 API |
| `block/snapshot.c` | 块层快照操作 |
| `block/qcow2-snapshot.c` | QCOW2 格式快照支持 |
| `migration/migration-hmp-cmds.c` | HMP 命令 (hmp_loadvm, hmp_savevm) |

---

## 五、实现方案分析

### 5.1 预计实现步骤

基于源码分析和项目描述，Fast Snapshot Load 的实现大致需要：

**阶段 1: 分离设备状态和 RAM 加载**
- 修改 `qemu_loadvm_state()` 流程，使其在加载完设备状态后即可返回
- RAM 部分不在此阶段同步加载

**阶段 2: userfaultfd 集成**
- 复用/适配 `postcopy_ram_incoming_setup()` 的 UFFD 注册逻辑
- 为所有 guest RAM 区域注册 userfaultfd 缺页通知
- 启动缺页处理线程

**阶段 3: 按需页面加载**
- 当 vCPU 触发缺页时，通过 fault thread 拦截
- 利用 mapped-ram 的固定偏移特性，从快照文件中定位并读取对应页面
- 通过 UFFDIO_COPY 原子放置到 guest 内存

**阶段 4: 后台批量加载**
- 启动后台线程持续加载剩余 RAM 页面
- 当所有页面加载完成后，注销 userfaultfd，完成整个加载过程

### 5.2 关键技术挑战

1. **设备状态与 RAM 的解耦**: 当前 `qemu_loadvm_state_main()` 按 section 顺序处理所有数据（设备 + RAM 混合），需要重构以支持 RAM 的延迟加载

2. **线程安全**: 后台加载线程、fault thread、vCPU 线程之间的同步

3. **BQL (Big QEMU Lock) 管理**:
   源码注释 (`savevm.c:2358`):
   > "if we can move migration loadvm out of main thread, then we won't block main thread from polling the accept() fds"

4. **RCU 临界区问题**:
   `ram.c` 中的 RAM 加载使用了非常长的 RCU 临界区：
   > "This RCU critical section can be very long running. When RCU reclaims... it will be necessary to reduce granularity"

5. **错误处理**: 缺页解析失败时如何优雅处理（VM 可能已在运行）

6. **与已有 postcopy 代码的复用 vs. 复制**: 需要评估 postcopy 代码的哪些部分可以直接复用

### 5.3 与 postcopy migration 的关键差异

| 方面 | Postcopy Migration | Fast Snapshot Load |
|------|--------------------|--------------------|
| 数据源 | 远程主机（网络） | 本地/远程存储（文件） |
| 数据格式 | 流式 | mapped-ram（随机访问） |
| 页面请求 | 向 source QEMU 请求 | 直接从文件偏移量读取 |
| multifd | 可选 | 初期可能不需要 |
| 网络延迟 | 存在 | 无（本地文件） |

---

## 六、GSoC 2026 其他 QEMU 项目（参考）

| 项目 | 难度 | 语言 | 导师 |
|------|------|------|------|
| Fast Snapshot Load | Advanced | C | Peter Xu |
| USB Device Redirection for qemu-rdp | Advanced | Rust | Marc-André Lureau |
| vhost-user Memory Isolation | Intermediate | C | Stefan Hajnoczi et al. |
| X86 PCID Support in COCONUT-SVSM | Intermediate | Rust | Joerg Roedel et al. |
| COCONUT-SVSM Observability Support | Intermediate | Rust/C | Stefano Garzarella et al. |
| SCSI TAPE Device Emulation | Intermediate | C | Helge Deller |

---

## 七、相关前置工作与 Patch 系列

### 7.1 mapped-ram 特性（已合并，前置依赖）

由 **Fabiano Rosas** 开发，使 RAM 页面在迁移文件中具有固定偏移量，支持随机访问。这是 Fast Snapshot Load 的 **必要前提**。

- 文档: https://www.qemu.org/docs/master/devel/migration/mapped-ram.html
- 配置方式: `migrate_set_capability mapped-ram on` + `migrate file:/path/to/file`

### 7.2 Threadify Loadvm Process RFC（2025年9月）

Peter Xu 提交了 9 个 patch 的 RFC 系列，将 loadvm 从协程模型转为 **线程模型**。这是直接相关的基础工作：
- 动机: 现代迁移特性（multifd, postcopy, mapped-ram, vfio）已使用线程，协程模型不一致
- 在关键设备状态加载点保持 BQL
- [RFC PATCH 0/9 migration: Threadify loadvm process](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1145648.html)

### 7.3 Marco Cavenati 的 mapped-ram snapshot 兼容 patches（已合并）

- [PATCH v2 0/2: Add support for mapped-ram with snapshots](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1143987.html)
- [PULL 05/36: mapped-ram: handle zero pages](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1151366.html)
- 解决了 mapped-ram 与 loadvm snapshot restore 的兼容性问题

### 7.4 历史相关工作：userfaultfd 快照（2016）

Hailiang Zhang 早期的 userfaultfd 快照相关工作：
- [RFC 00/13 Live memory snapshot based on userfaultfd](https://www.mail-archive.com/qemu-devel@nongnu.org/msg394897.html)

### 7.5 当前实现状态

截至目前，**尚无** Fast Snapshot Load 本身的 RFC 或 patch。项目仍处于 GSoC 提案阶段，等待 contributor 实现。

---

## 八、关键参考链接

### 官方资源
- [QEMU Snapshotting Improvements Wiki](https://wiki.qemu.org/Features/SnapshottingImprovements)
- [QEMU Snapshot API Deep Dive (Airbus SecLab)](https://airbus-seclab.github.io/qemu_blog/snapshot.html)
- [QEMU GSoC 2026 Wiki](https://wiki.qemu.org/Google_Summer_of_Code_2026)
- [QEMU GSoC 2026 Blog Announcement](https://www.qemu.org/2026/02/20/gsoc-2026/)
- [Fast Snapshot Load ToDo Page](https://wiki.qemu.org/ToDo/LiveMigration#Fast_load_snapshot)
- [Mapped-RAM Documentation](https://www.qemu.org/docs/master/devel/migration/mapped-ram.html)
- [Postcopy Documentation](https://www.qemu.org/docs/master/devel/migration/postcopy.html)

### 邮件列表讨论
- [Stefan Hajnoczi: Call for GSoC project ideas (2026-01-05)](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161673.html)
- [Peter Xu: Fast Snapshot Load proposal (2026-01-13)](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161899.html)
- [Marco Cavenati: Response with fuzzing use case (2026-01-14)](http://www.mail-archive.com/qemu-devel@nongnu.org/msg1161925.html)
- [Peter Xu & Marco: FEATURE_SEEKABLE 讨论](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1113922.html)
- [Marco: mapped-ram snapshot patches v2](https://www.mail-archive.com/qemu-devel@nongnu.org/msg1143987.html)

### 源码
- [migration/savevm.c](https://github.com/qemu/qemu/blob/master/migration/savevm.c) — 主要 loadvm 实现
- [migration/ram.c](https://github.com/qemu/qemu/blob/master/migration/ram.c) — RAM 加载 + mapped-ram
- [migration/postcopy-ram.c](https://github.com/qemu/qemu/blob/master/migration/postcopy-ram.c) — userfaultfd 机制

---

*报告生成日期: 2026-03-14*
