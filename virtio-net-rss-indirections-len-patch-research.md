# QEMU Patch 调研报告：virtio-net: validate RSS indirections_len in post_load

## 1. 基本信息

| 项目 | 内容 |
|------|------|
| **Patch 标题** | `[PATCH] virtio-net: validate RSS indirections_len in post_load` |
| **提交者** | Junjie Cao <junjie.cao@intel.com> |
| **提交日期** | 2026-03-23 (Mon, 23 Mar 2026 21:15:31 +0800) |
| **Message-ID** | `<20260323131531.1976-1-junjie.cao@intel.com>` |
| **邮件列表** | qemu-devel@nongnu.org |
| **CC** | jasowang@redhat.com (Jason Wang), mst@redhat.com (Michael S. Tsirkin), yuri.benditovich@daynix.com, akihiko.odaki@daynix.com, qemu-stable@nongnu.org |
| **Maintainers** | Michael S. Tsirkin, Jason Wang |
| **Fixes** | `e41b711485e5` ("virtio-net: add migration support for RSS and hash report") —— Yuri Benditovich 于 2020-05-08 提交 |
| **Patchew 链接** | https://patchew.org/QEMU/20260323131531.1976-1-junjie.cao@intel.com/ |
| **影响文件** | `hw/net/virtio-net.c` (+9 行) |
| **补丁状态** | 已应用到 patchew 测试树，邮件列表讨论中尚未合入主线 |

---

## 2. 问题背景与漏洞分析

### 2.1 RSS (Receive Side Scaling) 机制概述

RSS 是 virtio-net 设备的一项功能 (`VIRTIO_NET_F_RSS`)，用于将接收到的网络包根据哈希值分发到不同的虚拟队列（virtqueue），从而实现多队列负载均衡。核心数据结构包括：

- **`indirections_table`**: 间接表，将哈希值映射到具体的队列索引
- **`indirections_len`**: 间接表的长度，必须是 2 的幂次方（1, 2, 4, ..., 128）
- **`redirect`**: 是否启用 RSS 重定向

### 2.2 正常路径的验证逻辑

在正常的控制路径中（客户机通过 virtqueue 命令配置 RSS），`virtio_net_handle_rss()` 函数执行了严格的验证：

```c
// hw/net/virtio-net.c: 1415-1424
if (n->rss_data.indirections_len >= VIRTIO_NET_RSS_MAX_TABLE_LEN) {
    err_msg = "Too large indirection table";
    goto error;
}
n->rss_data.indirections_len++;
if (!is_power_of_2(n->rss_data.indirections_len)) {
    err_msg = "Invalid size of indirection table";
    goto error;
}
```

这确保了 `indirections_len` 始终是一个非零的 2 的幂次方值，且不超过 `VIRTIO_NET_RSS_MAX_TABLE_LEN`（128）。

### 2.3 漏洞：迁移路径缺少验证

在迁移（migration）路径中，VMState 框架直接从迁移数据流恢复 `indirections_len` 的值：

```c
// VMState 定义中直接反序列化
VMSTATE_UINT16(rss_data.indirections_len, VirtIONet),
VMSTATE_VARRAY_UINT16_ALLOC(rss_data.indirections_table, VirtIONet,
                            rss_data.indirections_len, 0,
                            vmstate_info_uint16, uint16_t),
```

而 `virtio_net_rss_post_load()` 函数在加载后只做了版本兼容性处理，**没有对 `indirections_len` 做任何有效性检查**：

```c
static int virtio_net_rss_post_load(void *opaque, int version_id)
{
    VirtIONet *n = VIRTIO_NET(opaque);
    if (version_id == 1) {
        n->rss_data.supported_hash_types = VIRTIO_NET_RSS_SUPPORTED_HASHES;
    }
    return 0;
}
```

### 2.4 崩溃触发链

攻击者可以通过构造恶意的迁移数据流，将 `indirections_len` 设置为 0，触发以下崩溃路径：

1. **迁移加载阶段**：恶意数据流设置 `indirections_len = 0`，同时可以清除 `redirect = false`
2. **特性恢复阶段**：`virtio_load()` 在加载设备 VMState（包括 RSS 子段及其 post_load）之后调用 `set_features_nocheck()`，根据已协商的客户机特性重新推导 `redirect`。当 `VIRTIO_NET_F_RSS` 已协商时，`redirect` 被重新设为 `true`，**无视迁移流中的值**
3. **接收路径**：当网络包到达时，`virtio_net_process_rss()` 执行：
   ```c
   new_index = hash & (n->rss_data.indirections_len - 1);
   // 当 indirections_len == 0 时：
   // 0 - 1 = 0xFFFF (uint16_t), 整数提升为 int 后变为 0xFFFFFFFF
   new_index = n->rss_data.indirections_table[new_index];
   // indirections_table 为 NULL（因为 VMState 加载器在元素数为 0 时不分配内存）
   // → NULL 指针解引用 → QEMU 崩溃
   ```

崩溃调用栈：
```
#0  virtio_net_process_rss    ../hw/net/virtio-net.c:1901
#1  virtio_net_receive_rcu    ../hw/net/virtio-net.c:1921
#2  virtio_net_do_receive     ../hw/net/virtio-net.c:2061
#3  nc_sendv_compat           ../net/net.c:823
#4  qemu_deliver_packet_iov   ../net/net.c:870
```

---

## 3. Patch 内容

Patch 本身非常简洁，仅添加了 9 行代码，在 `virtio_net_rss_post_load()` 函数中加入了与正常路径一致的验证逻辑：

```diff
 static int virtio_net_rss_post_load(void *opaque, int version_id)
 {
     VirtIONet *n = VIRTIO_NET(opaque);

     if (version_id == 1) {
         n->rss_data.supported_hash_types = VIRTIO_NET_RSS_SUPPORTED_HASHES;
     }

+    if (n->rss_data.indirections_len == 0 ||
+        n->rss_data.indirections_len > VIRTIO_NET_RSS_MAX_TABLE_LEN ||
+        !is_power_of_2(n->rss_data.indirections_len)) {
+        error_report("virtio-net: saved image has invalid RSS "
+                     "indirections_len: %u",
+                     n->rss_data.indirections_len);
+        return -EINVAL;
+    }
+
     return 0;
 }
```

验证条件：
1. `indirections_len == 0` → 阻止零长度表
2. `indirections_len > VIRTIO_NET_RSS_MAX_TABLE_LEN` → 阻止超出最大值 128
3. `!is_power_of_2(indirections_len)` → 确保是 2 的幂次方

---

## 4. 社区讨论分析

该 Patch 在邮件列表上引发了一场关于 QEMU **迁移数据流信任模型**的重要讨论，涉及多位核心维护者。

### 4.1 Michael S. Tsirkin (mst@redhat.com) —— virtio 维护者

**态度：支持 patch，提出改进建议**

两点代码审查意见：

1. **`== 0` 检查冗余**：指出 `is_power_of_2()` 对 0 返回 `false`，因此 `indirections_len == 0` 的检查是不需要的，被 `is_power_of_2` 已经涵盖。

2. **建议抽取公共函数**：去掉 `== 0` 后，验证逻辑与 `virtio_net_device_realize` 中的逻辑重复，建议将其提取为一个辅助函数（helper），避免代码重复。

Michael 还在后续讨论中确认，QEMU 过去确实为此类迁移数据流相关的安全问题分配过低优先级的 CVE。

### 4.2 Daniel P. Berrangé (berrange@redhat.com) —— QEMU 安全/迁移专家

**态度：质疑 patch 的安全性定位**

核心观点：**迁移数据流来自源端 QEMU，应被视为可信的（trusted）**。

具体论述：
- 迁移连接在建立时通过 SASL 显式认证或 x509 证书验证进行身份认证
- 数据流通过 TLS 或等效机制保护完整性
- VMState 数据期望反映当前 QEMU 配置，偏离将导致崩溃或更严重的后果
- 能够修改迁移数据流的攻击者意味着对整个客户机 RAM 具有任意读写能力，这意味着迁移后的客户机操作系统永远不可信——但实际上并非如此
- 源端和目标端 QEMU 使用相同的安全设施（非 root 运行、seccomp、SELinux/AppArmor、命名空间等），若攻击者无法在源端突破 QEMU，迁移不会使其能在目标端突破

他认为如果要接受此类 CVE，应该仅限于：
1. 恶意客户机驱动配置虚拟设备的某个方面，导致后续 VM 状态处理异常
2. 设备处理不受信任的数据时，导致设备状态异常，进而导致 VM 状态处理异常

他**不接受**外部攻击者修改迁移数据流或保存状态文件内容作为 CVE 的范围。

### 4.3 Peter Maydell (peter.maydell@linaro.org) —— QEMU 项目联合维护者

**态度：引用官方安全文档，支持将迁移流视为不可信**

关键引用了 QEMU 官方安全文档 (`docs/system/security.rst`)：

> "The following entities are untrusted, meaning that they may be buggy or malicious:
> - Network protocols (e.g. NBD, live migration)"

以及安全编码实践文档 (`docs/devel/secure-coding-practices.rst`)：

> "Live migration code must validate inputs when loading device state so an attacker cannot gain control by crafting invalid device states. Device state is therefore considered untrusted even though it is typically generated by QEMU itself."

Peter 的立场：
- 承认不信任迁移数据的威胁模型确实"极其谨慎"
- 但指出 QEMU 官方文档明确将其列为安全边界
- 提到过去确实有为此类问题分配 CVE 的先例（例如 CVE-2025-54566, CVE-2013-4536）
- 建议即使决定将迁移流视为可信，也应在 `security.rst` 文档的 "Sensitive configurations" 子节中明确记录建议

### 4.4 讨论焦点总结

讨论演变为一场关于 QEMU 安全边界的哲学性辩论：

| 观点 | 支持者 | 核心论据 |
|------|--------|----------|
| **迁移流不可信** | Peter Maydell, Michael S. Tsirkin | 官方安全文档明确列出；有 CVE 先例；纵深防御原则 |
| **迁移流可信** | Daniel P. Berrangé | TLS+认证保护；篡改意味着完全控制客户机 RAM；应是管理应用的 CVE |

Peter 最后建议寻求 Paolo Bonzini 和 Stefan Hajnoczi 的意见来解决这一争议，并建议无论最终决定如何，都应在安全文档中进行明确说明。

---

## 5. 技术深入分析

### 5.1 根本原因

该漏洞根源在于 **迁移路径与控制路径的验证不一致**。这是虚拟化代码中的常见模式：

- 正常路径（`virtio_net_handle_rss`）：经过完整的输入验证
- 迁移路径（`virtio_net_rss_post_load`）：跳过了验证，信任数据流

### 5.2 触发条件分析

漏洞触发需要以下条件同时满足：

1. 客户机协商了 `VIRTIO_NET_F_RSS` 特性
2. RSS 子段被加载（`rss_data.enabled == true`）
3. `indirections_len` 被设为无效值（如 0）
4. `virtio_load()` 中的 `set_features_nocheck()` 将 `redirect` 重新设为 `true`
5. 有网络包到达触发接收路径

### 5.3 影响范围

- **受影响版本**：自 commit `e41b711485e5`（2020-06-18 合入）以来的所有 QEMU 版本
- **影响类型**：QEMU 进程崩溃（DoS），因为是 NULL 指针解引用
- **攻击面**：取决于是否将迁移流视为安全边界

### 5.4 修复评估

Patch 本身质量良好，根据审查意见可做的改进：

1. 移除冗余的 `== 0` 检查（`is_power_of_2(0)` 返回 false）
2. 将验证逻辑提取为共享函数，供 `post_load` 和 `device_realize` 共同使用
3. Commit message 质量很高，详细描述了触发链和根因

---

## 6. 原始 Commit e41b711485e5 分析

该 Fixes 标签指向的原始 commit 由 Yuri Benditovich (yuri.benditovich@daynix.com) 于 2020 年 5 月提交，由 Jason Wang 合入，标题为 "virtio-net: add migration support for RSS and hash report"。

该 commit 添加了 RSS 数据的 VMState 迁移支持，定义了 `vmstate_virtio_net_rss` 结构，包含 `VMSTATE_VARRAY_UINT16_ALLOC` 用于迁移 `indirections_table`，但当时没有在 `post_load` 中加入对 `indirections_len` 的验证——这正是本次 Patch 要修复的遗留问题。

---

## 7. 关联上下文

### 7.1 Junjie Cao 的其他 QEMU 贡献

Junjie Cao 来自 Intel，具有 "QEMU/DPDK/Virtio-net/VHost stack" 方面的经验。在 2026 年 3 月，他还提交了另一个 v2 patch 系列：
- "[PATCH v2 1/3] io/channel: introduce qio_channel_pread{v,}_all() and preadv_all_eof()" — 修复 migration/file 中的类型不匹配和 NULL 解引用问题

### 7.2 历史 CVE 参考

讨论中提及的相关 CVE：
- **CVE-2025-54566**: QEMU PCIe SR-IOV 迁移状态不一致漏洞（CVSS 4.2 MEDIUM）
- **CVE-2013-4536**: 早期的 VMState 验证不足问题（Daniel 认为可能是错误签发的）

---

## 8. 结论与展望

### 当前状态
- Patch 已通过 patchew 自动化测试
- Maintainer (Michael S. Tsirkin) 基本认可修复方向，但要求代码改进
- 社区对迁移流安全性定位存在分歧，但按照现行文档标准，该修复是合理的

### 预期后续行动
1. Junjie Cao 需要提交 v2 版本：
   - 移除冗余的 `== 0` 检查
   - 将验证逻辑提取为辅助函数
2. 社区可能需要进一步讨论和更新 `security.rst` 文档中关于迁移流信任模型的表述
3. 如果被接受，Patch 将被标记 `Cc: qemu-stable`，回合到稳定版本分支
