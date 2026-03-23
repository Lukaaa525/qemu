# Patch Review: hw/net/virtio-net: Validate RSS configuration in post_load

## 一、复现情况

**当前环境没有 KVM（`/dev/kvm` 不存在）**，无法通过完整的 guest VM 进行端到端复现。
TCG 软件仿真启动 Alpine 需要 10-20 分钟以上，且需要 guest 内核协商 RSS 特性。

但 **从代码层面可以完全确认 bug 的真实性和可利用性**，崩溃路径如下：

### 崩溃路径完整推演

1. **迁移流加载**：`vmstate_virtio_net_rss` 从流中恢复 `rss_data`，其中：
   - `enabled = true`, `redirect = true`, `indirections_len = 0`
   - `VMSTATE_VARRAY_UINT16_ALLOC` 根据 `indirections_len = 0` 计算 `size = sizeof(uint16_t) * 0 = 0`
   - `vmstate_handle_alloc()` 中 `if (size)` 为 false，跳过 `g_malloc()` → `indirections_table` 保持 NULL

2. **`virtio_net_rss_post_load()`**（当前代码）：只做 version_id 检查，不验证任何字段，直接 `return 0`

3. **`virtio_net_post_load_device()`** 调用 `virtio_net_commit_rss_config(n)`（第 3246 行）：
   - `rss_data.enabled` 为 true
   - 如果 `populate_hash` 为 true：直接设置 `enabled_software_rss = true`
   - 如果 `populate_hash` 为 false：尝试 eBPF → `ebpf_rss_set_all()` 检测到 NULL table 返回 false → 非 vhost 场景回退 `enabled_software_rss = true`

4. **收包路径** `virtio_net_receive_rcu()`（第 1921 行）：检查 `n->rss_data.enabled && n->rss_data.enabled_software_rss` → true → 调用 `virtio_net_process_rss()`

5. **`virtio_net_process_rss()`**（第 1897-1899 行）：
   ```c
   new_index = hash & (n->rss_data.indirections_len - 1);  // 0 - 1 = 0xFFFF (uint16_t)
   new_index = n->rss_data.indirections_table[new_index];   // NULL[0xFFFF] → SEGV
   ```

**结论：bug 真实存在，可通过构造迁移流可靠触发。PoC 描述的崩溃路径与代码完全吻合。**

---

## 二、Patch 代码 Review

### 2.1 逻辑正确性：**通过**

```c
if (n->rss_data.redirect) {
    if (n->rss_data.indirections_len == 0 ||
        n->rss_data.indirections_len > VIRTIO_NET_RSS_MAX_TABLE_LEN ||
        !is_power_of_2(n->rss_data.indirections_len)) {
        return -1;
    }
}

if (n->rss_data.default_queue >= n->max_queue_pairs) {
    return -1;
}
```

- 命令路径 `virtio_net_handle_rss()` 先检查 `mask < 128`（即 `indirections_len < 128`），再 `++`，再检查 `is_power_of_2`。有效值集合为 `{1, 2, 4, 8, 16, 32, 64, 128}`。
- Patch 检查 `len == 0 || len > 128 || !is_power_of_2(len)`，允许的值集合同样为 `{1, 2, 4, 8, 16, 32, 64, 128}`。**等价，正确。**
- `default_queue < max_queue_pairs` 检查**与命令路径一致**（第 1428 行）。
- `redirect = false` 时不检查 `indirections_len` 是合理的，因为收包路径仅在 `redirect = true` 时索引该表。
- `default_queue` 检查放在 `redirect` 条件外面是正确的，与命令路径行为一致。
- `max_queue_pairs` 在设备 realize 阶段设置（`virtio_net_device_realize()`，第 3954 行），始终 >= 1，在 post_load 时已经有效。

### 2.2 问题一（严重）：缺少 error_report() — **与同文件惯例不一致**

**同一个文件 `hw/net/virtio-net.c` 中的所有其他迁移校验函数都使用了 `error_report()`：**

| 函数 | 做法 |
|------|------|
| `virtio_net_tx_waiting_pre_load()` (3333-3337) | `error_report("virtio-net: curr_queue_pairs %x > max_queue_pairs %x", ...)` + `return -EINVAL` |
| `virtio_net_ufo_post_load()` (3363-3365) | `error_report("virtio-net: saved image requires TUN_F_UFO support")` + `return -EINVAL` |
| `virtio_net_vnet_post_load()` (3397-3399) | `error_report("virtio-net: saved image requires vnet_hdr=on")` + `return -EINVAL` |

**当前 patch**：裸 `return -1`，没有任何日志输出。迁移失败时用户只看到一个通用的错误号，完全无法定位原因。

**这是 reviewer 几乎一定会要求修改的地方。**

### 2.3 问题二（中等）：返回 -1 而非 -EINVAL — **不符合主流惯例**

统计同类 post_load 返回值：

| 返回值 | 使用者 |
|--------|--------|
| `-EINVAL` | `mptsas_post_load`, `lsi_post_load`, `ide_drive_pio_post_load`, USB CVE-2013-4541, `virtio_net_ufo_post_load`, `virtio_net_vnet_post_load`, `virtio_net_tx_waiting_pre_load`, `configuration_post_load`, `global_state_post_load`, `nested_state_post_load`, `virtio_mem_mig_sanity_checks_post_load` |
| `-1` | `vmxnet3_post_load` |

**压倒性多数使用 `-EINVAL`。** 同文件内所有同类函数也都用 `-EINVAL`。虽然功能等价（`vmstate.c` 只检查 `ret < 0`），但 `-1` 是少数派，不符合项目规范。

### 2.4 其他细节

- **放置位置正确**：检查放在 `version_id == 1` 补丁之后、`return 0` 之前，逻辑清晰。
- **不需要额外检查 `indirections_table` 中的值**：命令路径也不逐条目校验（收包路径用 `% curr_queue_pairs` 截断），与现有行为一致。
- **不需要检查 `enabled_software_rss`**：这个字段不在 VMState 中，是 `virtio_net_commit_rss_config()` 动态计算的。

---

## 三、Commit Message Review

### 3.1 优点

- 清晰描述了 bug 的根因（post_load 缺少校验）
- 精确指出了 NULL 解引用的代码路径（`hash & (indirections_len - 1)` → 0xFFFF → 越界）
- 引用了 `vmstate_handle_alloc()` 解释为什么 len=0 导致 table 未分配
- `Fixes:` tag 正确指向 `e41b711485`
- `Cc: qemu-stable@nongnu.org` 符合安全修复流程

### 3.2 新版本 commit message 中加入 ASAN 栈回溯

新版本加入了 ASAN 崩溃栈回溯，这是一个**很好的改进**：

```
==PID==ERROR: AddressSanitizer: SEGV on unknown address (pc 0x... T0)
==PID==The signal is caused by a READ memory access.
    #0 in virtio_net_process_rss ../hw/net/virtio-net.c:1898
    ...
```

**但有一个细节**：ASAN 输出中的 `(pc 0x... T0)` 是匿名化的，实际提交时可以考虑保留真实地址或省略。目前的处理方式（用 `0x...` 占位）是可以接受的。

### 3.3 需要改进

commit message 总体质量好，但可以考虑：
- "crashes QEMU" 后面可以更明确写 "crashes the destination QEMU process, denying service to all VMs on the host"
- 最后的 "Mirror the command path constraints" 简洁准确

---

## 四、与已合入同类 Patch 的对比

### 4.1 标杆：CVE-2013-4541 (USB post_load)

```c
// hw/usb/bus.c - usb_device_post_load()
if (dev->setup_index < 0 ||
    dev->setup_len < 0 ||
    dev->setup_index >= sizeof(dev->data_buf) ||
    dev->setup_len >= sizeof(dev->data_buf)) {
    return -EINVAL;
}
```

- 用 `-EINVAL`
- 没有 `error_report()`（但这是 2014 年的代码，当时惯例不同）
- 获得了 CVE 编号

### 4.2 标杆：virtio-mem migration sanity checks (383ee44555)

- 70 行新增代码，专门做迁移流验证
- 用 `-EINVAL`
- 用 `error_report()` 输出每一个失败原因

### 4.3 标杆：vmxnet3_post_load + vmxnet3_validate_queues

```c
// hw/net/vmxnet3.c
static bool vmxnet3_validate_queues(VMXNET3State *s)
{
    if (s->txq_num > VMXNET3_DEVICE_MAX_TX_QUEUES) {
        qemu_log_mask(LOG_GUEST_ERROR, "vmxnet3: Bad TX queues number: %d\n",
                      s->txq_num);
        return false;
    }
    ...
}

// post_load 中：
if (!vmxnet3_validate_queues(s)) {
    return -1;
}
```

- 用 `qemu_log_mask(LOG_GUEST_ERROR, ...)` 输出原因
- 返回 `-1`（这是整个代码库中少数用 `-1` 的例子之一）

### 4.4 当前 patch 差距总结

| 方面 | 标杆做法 | 当前 patch | 差距 |
|------|---------|-----------|------|
| 返回值 | 绝大多数 `-EINVAL` | `-1` | 不符合主流惯例，尤其是**同文件内所有类似函数都用 `-EINVAL`** |
| 错误信息 | `error_report()` 或 `qemu_log_mask()` | 无 | **同文件内所有类似函数都有 `error_report()`**，这是最大的差距 |
| 逻辑完整性 | - | 完整镜像命令路径约束 | 无差距 |
| Commit message | 描述问题+修复 | 描述清晰，附带 ASAN 栈 | 无差距，甚至优于部分标杆 |

---

## 五、最终建议

### 必须修改（reviewer 几乎必然要求）

加 `error_report()`，改 `-EINVAL`。建议修改为：

```c
if (n->rss_data.redirect) {
    if (n->rss_data.indirections_len == 0 ||
        n->rss_data.indirections_len > VIRTIO_NET_RSS_MAX_TABLE_LEN ||
        !is_power_of_2(n->rss_data.indirections_len)) {
        error_report("virtio-net: saved image has invalid RSS "
                     "indirections_len: %u", n->rss_data.indirections_len);
        return -EINVAL;
    }
}

if (n->rss_data.default_queue >= n->max_queue_pairs) {
    error_report("virtio-net: saved image has invalid RSS default_queue: "
                 "%u (max: %u)", n->rss_data.default_queue,
                 n->max_queue_pairs);
    return -EINVAL;
}
```

### 可选改进

- Commit message 中 "crashes QEMU" 可以加上影响范围说明（如 "denying service to all VMs on the host"）
- 可以考虑请求 CVE 编号，这是一个可通过网络可达的 DoS（需要攻击者能控制迁移流）

---

## 六、结论

### 这是否真的完全有必要？

**是的。** 这不是"预防性代码美化"，而是修复一个确定可利用的 NULL 指针解引用 crash，攻击面是迁移流。QEMU 安全编码文档 (`docs/devel/secure-coding-practices.rst`) 明确规定迁移数据为不可信输入必须验证。

### 会不会被怀疑灌水？

**不会，前提是修复质量要跟上。** 目前的版本缺少 `error_report()` 和正确的返回值，可能被 reviewer 认为"做了但没做到位"。加上这两项改动后，patch 完全符合社区标准，有大量已合入的同类先例支撑。

### 是否有类似 patch 成功合入？

**大量先例：** CVE-2013-4541 (USB), virtio-mem 迁移校验 (383ee44555), mptsas_post_load, lsi_post_load, ide_drive_pio_post_load, vmxnet3_post_load, ahci post_load, 等等。这个类型的修复在 QEMU 中是常规安全维护。
