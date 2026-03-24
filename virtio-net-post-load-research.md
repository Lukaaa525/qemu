# Junjie Cao (junjie.cao@intel.com) QEMU 社区 virtio-net post_load Patch 调研报告

## 1. 概述

Junjie Cao 于 2026 年 3 月 23 日向 QEMU 社区提交了一个 patch：**"[PATCH] virtio-net: validate RSS indirections_len in post_load"**，旨在修复 virtio-net 设备在迁移流（migration stream）加载后 RSS（Receive Side Scaling）`indirections_len` 字段未经验证导致的空指针解引用（NULL pointer dereference）崩溃问题。

该 patch 引发了 QEMU 核心维护者之间关于**迁移流（migration stream）是否应被视为可信（trusted）**这一安全模型根本性问题的深入讨论，参与者包括 Michael S. Tsirkin、Daniel P. Berrangé、Peter Maydell 等社区重量级人物。

---

## 2. Patch 技术分析

### 2.1 问题背景

QEMU 的 virtio-net 设备实现了 RSS（Receive Side Scaling）功能，用于将接收的网络包分散到多个队列中以提高多核系统的网络性能。RSS 的核心数据结构之一是**间接表（indirections_table）**，其长度由 `indirections_len` 字段决定。

在正常的 guest 命令路径中，`virtio_net_handle_rss()` 函数对 `indirections_len` 实施了严格的验证：

```c
// hw/net/virtio-net.c, 约 1410-1424 行
n->rss_data.indirections_len =
    virtio_lduw_p(vdev, &cfg.indirection_table_mask);
// ...
if (n->rss_data.indirections_len >= VIRTIO_NET_RSS_MAX_TABLE_LEN) {  // 上限为 128
    err_msg = "Too large indirection table";
    goto error;
}
n->rss_data.indirections_len++;  // mask + 1 = 长度
if (!is_power_of_2(n->rss_data.indirections_len)) {  // 必须为 2 的幂
    err_msg = "Invalid size of indirection table";
    goto error;
}
```

合法值范围：{1, 2, 4, 8, 16, 32, 64, 128}

### 2.2 漏洞机制

然而，`virtio_net_rss_post_load()` 函数在从迁移流恢复 RSS 状态时，**没有对 `indirections_len` 实施任何验证**：

```c
// hw/net/virtio-net.c, 约 3424-3433 行（修复前）
static int virtio_net_rss_post_load(void *opaque, int version_id)
{
    VirtIONet *n = VIRTIO_NET(opaque);
    if (version_id == 1) {
        n->rss_data.supported_hash_types = VIRTIO_NET_RSS_SUPPORTED_HASHES;
    }
    return 0;  // 没有任何 indirections_len 的验证！
}
```

**攻击链分析：**

1. 恶意迁移流将 `indirections_len` 设为 0，同时将 `redirect` 设为 false
2. VMState 加载器根据 `VMSTATE_VARRAY_UINT16_ALLOC` 宏按 `indirections_len=0` 分配内存 —— 即**不分配任何内存**（`indirections_table` 为 NULL）
3. `virtio_net_rss_post_load()` 不做任何验证即返回成功
4. **关键步骤**：`virtio_load()` 在 vmstate（包含 RSS 子段及其 post_load）加载**之后**调用 `set_features_nocheck()`，该函数从协商的 guest features 重新派生 `redirect`
5. 当 `VIRTIO_NET_F_RSS` 已协商时，`redirect` 被重新设为 `true`（无视迁移流中的 false 值）
6. 数据包接收路径执行：
   ```c
   // hw/net/virtio-net.c, 约 1897-1899 行
   if (n->rss_data.redirect) {
       new_index = hash & (n->rss_data.indirections_len - 1);
       // 当 indirections_len == 0 时：
       // 0 - 1 = 0xFFFF (uint16_t) -> 整数提升为 0xFFFFFFFF
       new_index = n->rss_data.indirections_table[new_index];
       // indirections_table 为 NULL -> 空指针解引用 -> QEMU 崩溃！
   }
   ```

**崩溃栈：**

```
#0  virtio_net_process_rss    ../hw/net/virtio-net.c:1901
#1  virtio_net_receive_rcu    ../hw/net/virtio-net.c:1921
#2  virtio_net_do_receive     ../hw/net/virtio-net.c:2061
#3  nc_sendv_compat           ../net/net.c:823
#4  qemu_deliver_packet_iov   ../net/net.c:870
```

### 2.3 修复方案

Patch 在 `virtio_net_rss_post_load()` 中添加了与命令路径一致的验证逻辑：

```c
static int virtio_net_rss_post_load(void *opaque, int version_id)
{
    VirtIONet *n = VIRTIO_NET(opaque);
    if (version_id == 1) {
        n->rss_data.supported_hash_types = VIRTIO_NET_RSS_SUPPORTED_HASHES;
    }

    if (n->rss_data.indirections_len == 0 ||
        n->rss_data.indirections_len > VIRTIO_NET_RSS_MAX_TABLE_LEN ||
        !is_power_of_2(n->rss_data.indirections_len)) {
        error_report("virtio-net: saved image has invalid RSS "
                     "indirections_len: %u",
                     n->rss_data.indirections_len);
        return -EINVAL;
    }

    return 0;
}
```

**Fixes 标签**：`e41b711485e5 ("virtio-net: add migration support for RSS and hash report")`

**抄送**：`qemu-stable@nongnu.org`（表明作者认为此修复应回溯到稳定分支）

---

## 3. 社区讨论深度分析

### 3.1 Michael S. Tsirkin（virtio 维护者）的代码审查

**时间**：patch 提交后约 1 小时

**主要反馈：**

1. **`== 0` 检查冗余**：`is_power_of_2()` 宏对 0 已经返回 false，因此 `indirections_len == 0` 的显式检查是不必要的
2. **建议代码重构**：指出验证逻辑与 `virtio_net_device_realize` 中的逻辑重复，建议**提取为公共辅助函数（helper function）**以避免代码重复

> "with == 0 removed, the logic is duplicated from virtio_net_device_realize. How about factoring it out to a helper?"

### 3.2 Daniel P. Berrangé（安全/迁移维护者）的根本性质疑

**时间**：与 Tsirkin 审查几乎同时

Daniel 提出了一个**颠覆性观点**：

> "The migration stream originating from the source QEMU is trusted."
>
> "Is there a problem you can demonstrate with regular QEMU commands / versions, not crafting a malicious migration stream."

他质疑 patch 的前提——即迁移流是否需要被防御性地对待。在他看来，迁移流应通过以下机制保证可信度：
- **连接认证**：SASL 认证或 x509 证书验证
- **数据完整性**：TLS 加密保护或等效机制

### 3.3 Peter Maydell（QEMU 项目联合负责人）引用官方安全文档反驳

Peter Maydell 迅速引用了 QEMU 官方安全文档（`docs/master/system/security.html`）进行反驳：

> ```
> The following entities are untrusted, meaning that they may be buggy or malicious:
>   * Guest
>   * User-facing interfaces (e.g. VNC, SPICE, WebSocket)
>   * Network protocols (e.g. NBD, live migration)      <-- 明确列出
>   * User-supplied files (e.g. disk images, kernels, device trees)
>   * Passthrough devices (e.g. PCI, USB)
> ```

他指出，QEMU 的官方安全文档**明确将 live migration 列为不可信实体**，并表示过去确实接受过添加迁移数据健全性检查的 patch。

同时，QEMU 的安全编码实践文档（`docs/master/devel/secure-coding-practices.html`）更加明确：

> "Device state can be saved to disk image files and shared with other users. Live migration code must validate inputs when loading device state so an attacker cannot gain control by crafting invalid device states. **Device state is therefore considered untrusted even though it is typically generated by QEMU itself.**"

### 3.4 Michael S. Tsirkin 补充 CVE 先例

Tsirkin 进一步补充道：

> "And we even assigned a low priority CVEs to these."

这表明 QEMU 项目历史上确实曾为迁移流相关的安全问题分配过 CVE 编号。

### 3.5 Daniel Berrangé 的深层安全模型论述

Daniel 对此展开了更深层次的分析，试图从威胁模型角度论证迁移流可信论的合理性：

1. **源 QEMU 被攻陷场景**：唯一能发送恶意 vmstate 数据的实体是源 QEMU。如果源 QEMU 已被攻陷，但攻击者无法突破源主机操作系统（由 seccomp、SELinux/AppArmor、命名空间等安全设施保护），那么通过迁移数据突破目标主机也同样困难，因为两端使用相同的安全设施。

2. **快照文件场景**：保存的快照文件同样包含 guest RAM 数据（本质上不可信），如果攻击者有能力修改快照文件，则信任已完全丧失。

3. **他认为 CVE 范围应限于**：
   - 恶意 guest OS 驱动配置虚拟设备的某些方面，导致后续 VM state 处理出错
   - 虚拟设备处理的不可信数据导致设备进入异常状态

4. **他不接受的 CVE 范围**：
   > "I don't accept scope for CVEs in QEMU for an external attacker modifying the migration data stream or saved state file contents though. AFAICS, such possibilities imply gross misconfiguration of QEMU by the mgmt app, and should be CVEs in the mgmt app instead."

### 3.6 Peter Maydell 的反驳与折中

Peter 对 Daniel 的论点进行了多维度反驳：

1. **CVE 先例**：引用了 `CVE-2025-54566` 和 `CVE-2013-4536` 作为 QEMU 迁移流安全 CVE 的实际例证

2. **云厂商视角**：
   > "If I'm a cloud vendor then I don't trust the guest OS in the first place. QEMU itself should consider everything in guest RAM untrusted data, because it's guest-controlled."

3. **用户可修改迁移流的场景**：用户可能被授权 vmsave/vmload 到其拥有的文件，就像直接写入 VM 磁盘镜像一样，不构成逃逸向量

4. **关于政策变更的关键认定**：
   > "I don't object to our deciding we want to call the migration-stream trusted -- but I do think this is a policy change from our current stance."

5. **建议将此安全模型讨论结果更新到 `security.rst` 文档**中

### 3.7 Daniel 的最终回应与 CVE 区分

Daniel 仔细分析了 Peter 引用的两个 CVE：

- **CVE-2025-54566**：他认为有效，因为涉及 guest OS 控制的设置导致的不正确处理，没有外部实体声称修改了迁移流
- **CVE-2013-4536**：他认为可能是"错误分配的 CVE"

他进一步区分了两种不同情况：
- 由 `cad9aa6fbdccd95e56e10cfa57c354a20a333717` 修复的 CVE-2025-54566 / CVE-2025-54567 组合同时修复了 (a) guest OS 设置恶意值 和 (b) post_load 中缺少验证两个问题。Daniel 的立场是只有 (a) 需要修复。

---

## 4. Junjie Cao 的社区背景

### 4.1 个人背景

- 新加坡国立大学（NUS）毕业
- 目前就职于 Intel
- 专注于网络设备虚拟化领域：QEMU/DPDK/Virtio-net/VHost 栈

### 4.2 QEMU 社区活动时间线

| 日期 | 活动 |
|------|------|
| 2026-03-15 | 在 qemu-devel 邮件列表表达参与 GSoC 2026 "Fast Snapshot Load" 项目的兴趣 |
| 2026-03-16 | 提交第一个 patch (v1)：修复 `multifd_file_recv_data()` 中的类型不匹配和空指针解引用 |
| 2026-03-18 | 提交 v2 版本（3 patch 系列）：根据 Peter Xu 和 Daniel 的建议引入 `qio_channel_pread{v,}_all()` 辅助函数 |
| 2026-03-23 | 提交 virtio-net RSS `indirections_len` 验证 patch（本报告的核心主题） |

### 4.3 GSoC 2026 Fast Snapshot Load 项目

Junjie 正在准备参与 QEMU 的 GSoC 2026 项目——"Fast Snapshot Load"，其核心思路：
- 将 loadvm 拆分为"设备状态加载"和"RAM 加载"两个阶段，通过 userfaultfd 桥接
- 利用 mapped-ram 的固定偏移格式，支持 RAM 页面的按需 pread()
- 后台预加载线程 + 缺页处理线程并行填充 RAM

Peter Xu（项目导师）对此方案给予了积极反馈，建议复用 postcopy-ram + mapped-ram 机制以避免新增 feature bit。

### 4.4 其他 Patch 贡献

**migration/file 修复系列（v1→v2）**：
- **v1**（2026-03-16）：修复 `multifd_file_recv_data()` 中 `ssize_t`/`size_t` 类型混用导致错误消息显示 SIZE_MAX，以及截断迁移文件导致的空指针解引用
- **v2**（2026-03-18）：根据社区反馈重新设计方案，引入 `qio_channel_pread{v,}_all()` 和 `preadv_all_eof()` 辅助函数（遵循既有的 `read_all` 模式），替换了 v1 中简单的错误处理拆分，并添加了 5 个单元测试

---

## 5. 讨论总结与影响

### 5.1 技术层面

Patch 本身的修复是正确且必要的——无论最终安全策略如何定义，在 `post_load` 中添加对 `indirections_len` 的验证都能防止 QEMU 崩溃，增强了代码的健壮性。

Michael S. Tsirkin 的审查建议（去除冗余检查、提取公共辅助函数）是合理的代码质量改进，预期 Junjie 会提交 v2 版本。

### 5.2 安全策略层面

此 patch 引发的讨论触及了 QEMU 项目的一个长期未解分歧：

| 立场 | 代表人物 | 核心论点 |
|------|----------|----------|
| **迁移流不可信** | Peter Maydell, Michael S. Tsirkin | 官方安全文档明确列出；有 CVE 先例；安全编码实践文档要求验证 |
| **迁移流可信** | Daniel P. Berrangé | 应通过 TLS/SASL 保护通道完整性；修改迁移流意味着攻击者已拥有对 guest RAM 的完全控制；CVE 应分配给管理应用的安全配置缺陷 |

Peter Maydell 建议将讨论结论更新到 `security.rst` 文档中，无论最终决策如何。他还指出，如果社区决定将迁移流视为可信，这将是**对当前立场的政策变更**。

### 5.3 Patch 当前状态

截至调研时间（2026-03-24），该 patch：
- 已获得 Tsirkin 的初步代码审查（要求小幅修改）
- 在安全模型讨论中未被否决
- 预计 Junjie 会根据 Tsirkin 的反馈提交 v2 版本
- 安全模型的根本性讨论可能需要更多社区成员参与才能达成共识

---

## 6. 相关引用

- **Patch 原始链接**：https://patchew.org/QEMU/20260323131531.1976-1-junjie.cao@intel.com/
- **QEMU 安全文档**：https://www.qemu.org/docs/master/system/security.html
- **QEMU 安全编码实践**：https://www.qemu.org/docs/master/devel/secure-coding-practices.html
- **修复对应的原始 commit**：`e41b711485e5 ("virtio-net: add migration support for RSS and hash report")`
- **相关 CVE 引用**：CVE-2025-54566, CVE-2025-54567, CVE-2013-4536
- **Junjie 的 migration/file 修复**：https://patchew.org/QEMU/20260318140113.434-1-junjie.cao@intel.com/
- **Junjie 的 GSoC 讨论**：https://www.mail-archive.com/qemu-devel@nongnu.org/msg1178239.html
