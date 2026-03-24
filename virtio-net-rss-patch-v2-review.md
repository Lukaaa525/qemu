# [PATCH v2] virtio-net: validate RSS indirections_len in post_load — Review

## 总体评价

该 v2 补丁正确地回应了 Michael S. Tsirkin 在 v1 中提出的两个审查意见，质量有显著提升。补丁方向正确，修复了一个真实的 NULL 指针解引用崩溃路径。以下是逐项的专业审查。

---

## 1. Commit Message 审查

### 优点

- v2 将触发条件从 v1 的 "A crafted migration stream" 扩展为 **"A corrupted save file or crafted migration stream"**，这是对 Daniel Berrangé 在 v1 讨论中提出的 vmsave/vmload 场景的恰当回应，使得漏洞描述不再局限于"恶意迁移流"这一有争议的攻击面，同时覆盖了社区已达成共识的 save/restore 场景。
- 崩溃触发链的描述清晰完整，包含调用栈。
- 正确引用了 Fixes 标签和 `Cc: qemu-stable`。
- changelog（`v1 -> v2`）清晰记录了变更原因和出处（`[MST]`）。

### 一个小瑕疵

Commit message 中说 *"Factor the validation into virtio_net_rss_indirections_len_valid() and call it from both virtio_net_handle_rss() and virtio_net_rss_post_load()"*，但实际上 v2 还改变了 `virtio_net_handle_rss()` 中原有的两步检查逻辑（先检查 `>= MAX`，再 `++`，再检查 `is_power_of_2`）合并为一步。这个行为变更本身是正确的，但 commit message 没有明确提及这一点。严格来说这属于小问题，因为语义上 "replacing the two separate checks in the command path" 在 changelog 部分有说明。

**建议**：可以接受现状，不算阻塞性问题。

---

## 2. 代码审查

### 2.1 `virtio_net_rss_indirections_len_valid()` 辅助函数

```c
static bool virtio_net_rss_indirections_len_valid(uint16_t len)
{
    return is_power_of_2(len) && len <= VIRTIO_NET_RSS_MAX_TABLE_LEN;
}
```

**审查结论：正确。**

- `is_power_of_2(0)` 返回 `false`（已通过查看 `include/qemu/host-utils.h:713-716` 确认），因此 `len == 0` 被正确拒绝，无需额外的 `== 0` 检查。v1 中 MST 指出的冗余问题已解决。
- `len` 参数类型为 `uint16_t`，与 `rss_data.indirections_len` 的声明类型（`include/hw/virtio/virtio-net.h:153`）一致。
- `is_power_of_2()` 接受 `uint64_t` 参数，`uint16_t` 会被隐式零扩展，无符号性安全。
- `VIRTIO_NET_RSS_MAX_TABLE_LEN` 是 128（`0x80`），是 `uint16_t` 范围内的值，比较安全。
- 函数命名遵循 QEMU virtio-net 的命名惯例（`virtio_net_rss_` 前缀），返回 `bool`，语义清晰。
- 放置在 `virtio_net_handle_rss()` 之前（第 1374 行之后），保证了调用点的可见性。

**无修改意见。**

### 2.2 `virtio_net_handle_rss()` 中的替换

原代码（v1/主线）：
```c
    if (n->rss_data.indirections_len >= VIRTIO_NET_RSS_MAX_TABLE_LEN) {
        err_msg = "Too large indirection table";
        err_value = n->rss_data.indirections_len;
        goto error;
    }
    n->rss_data.indirections_len++;
    if (!is_power_of_2(n->rss_data.indirections_len)) {
        err_msg = "Invalid size of indirection table";
        err_value = n->rss_data.indirections_len;
        goto error;
    }
```

v2 替换为：
```c
    n->rss_data.indirections_len++;
    if (!virtio_net_rss_indirections_len_valid(n->rss_data.indirections_len)) {
        err_msg = "Invalid indirection table length";
        err_value = n->rss_data.indirections_len;
        goto error;
    }
```

**语义等价性分析**：需要验证两步检查合并为一步后是否存在行为差异。

原代码的逻辑是：
1. 首先检查 `mask >= 128`（在 `++` 之前），即拒绝 mask >= 128
2. 执行 `len = mask + 1`
3. 检查 `len` 是否为 2 的幂

v2 的逻辑是：
1. 执行 `len = mask + 1`
2. 检查 `len` 是否为 2 的幂 **且** `len <= 128`

考虑边界情况：
- `mask = 127` → `len = 128`：原代码中第一步 `127 >= 128` 为 false，通过；`128` 是 2 的幂，通过。v2 中 `is_power_of_2(128)` 为 true，`128 <= 128` 为 true，通过。**一致。**
- `mask = 128` → `len = 129`：原代码中第一步 `128 >= 128` 为 true，**拒绝**。v2 中 `is_power_of_2(129)` 为 false，**拒绝**。**一致。**
- `mask = 255` → `len = 256`：原代码第一步 `255 >= 128` 为 true，**拒绝**。v2 中 `is_power_of_2(256)` 为 true 但 `256 <= 128` 为 false，**拒绝**。**一致。**
- `mask = 65535`（`uint16_t` 最大值）→ `len = 0`（溢出回绕！）：原代码第一步 `65535 >= 128` 为 true，**拒绝**。v2 中会先溢出到 0，然后 `is_power_of_2(0)` 返回 false，**拒绝**。**一致。**
- `!do_rss` 路径下 `mask = 0` → `len = 1`：`is_power_of_2(1)` 为 true，`1 <= 128` 为 true，通过。原代码也通过。**一致。**

**结论：语义严格等价，溢出边界也安全。替换正确。**

但有一个注意点：原代码中 `>= VIRTIO_NET_RSS_MAX_TABLE_LEN` 的检查在 `++` 之前执行，意味着如果 `mask` 很大（比如 65535），`++` 操作不会执行，`indirections_len` 不会被修改。而 v2 中 **`++` 无条件执行**，即使验证失败，`n->rss_data.indirections_len` 已经被修改（增加了 1）。然后跳到 `error:` 标签。

让我们检查 `error:` 标签做了什么：

```c
error:
    trace_virtio_net_rss_error(n, err_msg, err_value);
    virtio_net_disable_rss(n);
    return 0;
```

`virtio_net_disable_rss(n)` 会将 `enabled` 设为 false 并调用 `virtio_net_commit_rss_config(n)`，所以被修改的 `indirections_len` 值不会在后续被使用。**因此这个差异实际上是无害的。** 但如果有人将来在 error 路径中依赖 `indirections_len` 的原始值（例如 trace 已经在用 `err_value` 而非 `indirections_len`），理论上存在微弱的可维护性风险。

**结论：可接受。**

### 2.3 `virtio_net_rss_post_load()` 中的新增验证

```c
    if (!virtio_net_rss_indirections_len_valid(n->rss_data.indirections_len)) {
        error_report("virtio-net: saved image has invalid RSS "
                     "indirections_len: %u",
                     n->rss_data.indirections_len);
        return -EINVAL;
    }
```

**审查结论：正确。**

- 放置位置正确：在 `version_id == 1` 的兼容性处理之后，在 `return 0` 之前。
- `error_report()` 是 `post_load` 回调中报告错误的标准方式（无 `Error **errp` 可用）。
- 返回 `-EINVAL` 将导致 VMState 加载失败，迁移/快照恢复被拒绝，这是正确的行为。
- 错误信息包含具体的 `indirections_len` 数值，有助于调试。

**无修改意见。**

---

## 3. 潜在问题与改进建议

### 3.1 [中等] `default_queue` 未在 `post_load` 中验证

`rss_data.default_queue` 同样从迁移流中反序列化，在正常路径中通过 `n->rss_data.default_queue >= n->max_queue_pairs` 进行验证（第 1428 行），但在 `post_load` 中没有验证。

在 `virtio_net_process_rss()` 中（第 1886 行）：
```c
return n->rss_data.redirect ? n->rss_data.default_queue : -1;
```

然后在 `virtio_net_receive_rcu()` 中（第 1924 行）：
```c
nc = qemu_get_subqueue(n->nic, index % n->curr_queue_pairs);
```

由于调用方使用了 `% n->curr_queue_pairs` 取模，`default_queue` 的越界值不会导致越界访问。所以这不是一个安全漏洞，但它确实意味着迁移后可能出现非预期的队列选择行为。

**建议**：这超出了本补丁的范围，可以作为后续工作。但如果作者愿意，可以在同一个 `post_load` 中顺带验证 `default_queue`。不作为本次 review 的阻塞项。

### 3.2 [低] `indirections_table` 元素值未验证

`indirections_table` 中的每个元素是一个队列索引。在正常路径中（`virtio_net_handle_rss()`），这些值从客户机提供的 buffer 中读取但并未进行范围验证。在接收路径中由 `% n->curr_queue_pairs` 保护（第 1924 行），所以不会导致越界访问。

这同样不在本补丁的范围内。

### 3.3 [低] 错误消息字符串差异

v2 将原来的两个不同错误消息（"Too large indirection table" 和 "Invalid size of indirection table"）合并为一个 "Invalid indirection table length"。这是合理的简化，但意味着 trace 中无法区分"太大"和"非 2 的幂"两种情况。由于 `err_value` 会记录实际值，运维人员仍可通过数值判断原因，所以这是可接受的。

---

## 4. 与 v1 讨论的一致性检查

| v1 审查意见 | v2 处理方式 | 状态 |
|---|---|---|
| MST: `== 0` 检查冗余，`is_power_of_2` 已覆盖 | 已移除 `== 0` 检查 | **已解决** |
| MST: 逻辑与 `virtio_net_device_realize` 重复，建议提取 helper | 提取了 `virtio_net_rss_indirections_len_valid()` | **已解决** |
| Daniel: 迁移流应被视为可信 | commit message 改为 "corrupted save file or crafted migration stream"，回避了争议 | **已妥善处理** |

注意：MST 原文说的是 *"the logic is duplicated from virtio_net_device_realize"*，但实际上在当前代码中 `virtio_net_device_realize()` 并不包含 `indirections_len` 的验证逻辑——相关检查只存在于 `virtio_net_handle_rss()` 中。所以 v2 在 `virtio_net_handle_rss()` 和 `virtio_net_rss_post_load()` 之间共享 helper 是对 MST 意图的正确理解。

---

## 5. 编译与测试考量

- 补丁仅修改 `hw/net/virtio-net.c`，不涉及头文件变更或新的外部依赖。
- `is_power_of_2` 来自 `include/qemu/host-utils.h`，已被 `virtio-net.c` 间接包含。
- `VIRTIO_NET_RSS_MAX_TABLE_LEN` 来自 `include/hw/virtio/virtio-net.h`，已包含。
- 没有添加新的测试。对于迁移相关的安全修复，理想情况下应该有一个 qtest 来验证恶意 vmstate 被正确拒绝，但这在 QEMU 社区中通常不是强制要求。

---

## 6. 最终结论

**该补丁可以接受（建议 Acked-by / Reviewed-by）**。

补丁质量良好，正确修复了一个可通过损坏的快照文件或恶意迁移流触发的 NULL 指针解引用崩溃。v2 恰当地回应了 v1 的所有审查意见，代码简洁，语义等价性经验证无误。commit message 措辞在社区争议（迁移流是否可信）之间取得了良好平衡。

如果要给出正式的 review tag：

```
Reviewed-by: <your-name> <your-email>
```

如果想提出非阻塞性建议，可以附带一句：

> Nit: might be worth also validating `default_queue` against
> `max_queue_pairs` in `post_load` in a follow-up, for the same
> class of issue.
