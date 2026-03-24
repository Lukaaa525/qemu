# virtio-net: validate RSS indirections_len in post_load — 完整邮件时间线

按时间顺序整理，共 **11 封邮件**。

---

## Message #1 — 原始 Patch 提交

- **发件人**: Junjie Cao <junjie.cao@intel.com>
- **时间**: Mon, 23 Mar 2026 21:15:31 +0800 (北京时间)
- **Message-ID**: msg06572

### 原文

```
virtio_net_handle_rss() enforces that indirections_len is a non-zero
power of two no larger than VIRTIO_NET_RSS_MAX_TABLE_LEN, but
virtio_net_rss_post_load() applies none of these checks to values
restored from the migration stream.

A crafted migration stream can set indirections_len to 0.  Even if it
also clears redirect, virtio_load() calls set_features_nocheck() after
the device vmstate (including the RSS subsection and its post_load) has
already been loaded, re-deriving redirect from the negotiated guest
features.  When VIRTIO_NET_F_RSS was negotiated, redirect is set back
to true regardless of the migration stream value.  The receive path
then computes

    hash & (indirections_len - 1)   /* wraps to 0xFFFFFFFF via int promotion */

and uses the result to index into indirections_table, which was not
allocated by the VMState loader when the element count is zero (see
vmstate_handle_alloc()), resulting in a NULL pointer dereference that
crashes QEMU:

  #0  virtio_net_process_rss    ../hw/net/virtio-net.c:1901
  #1  virtio_net_receive_rcu    ../hw/net/virtio-net.c:1921
  #2  virtio_net_do_receive     ../hw/net/virtio-net.c:2061
  #3  nc_sendv_compat           ../net/net.c:823
  #4  qemu_deliver_packet_iov   ../net/net.c:870

The RSS subsection is only loaded when rss_data.enabled is true (via
virtio_net_rss_needed()), and the command path always produces
indirections_len in {1, 2, 4, …, 128}, so an unconditional check
cannot reject a legitimate migration stream.

Fixes: e41b711485e5 ("virtio-net: add migration support for RSS and hash report")
Cc: qemu-stable@nongnu.org
Signed-off-by: Junjie Cao <junjie.cao@intel.com>
---
 hw/net/virtio-net.c | 9 +++++++++
 1 file changed, 9 insertions(+)

diff --git a/hw/net/virtio-net.c b/hw/net/virtio-net.c
index 2a5d642a64..3038836098 100644
--- a/hw/net/virtio-net.c
+++ b/hw/net/virtio-net.c
@@ -3427,6 +3427,15 @@ static int virtio_net_rss_post_load(void *opaque, int version_id)
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

### 中文翻译

`virtio_net_handle_rss()` 函数强制要求 `indirections_len` 是一个非零的、不超过 `VIRTIO_NET_RSS_MAX_TABLE_LEN` 的 2 的幂次方值，但 `virtio_net_rss_post_load()` 对从迁移流中恢复的值不执行任何这些检查。

一个精心构造的迁移流可以将 `indirections_len` 设置为 0。即使它同时清除了 `redirect`，`virtio_load()` 也会在设备 vmstate（包括 RSS 子段及其 `post_load`）已经加载完成之后调用 `set_features_nocheck()`，从已协商的客户机特性中重新推导 `redirect`。当 `VIRTIO_NET_F_RSS` 已被协商时，`redirect` 会被重新设置为 `true`，无视迁移流中的值。接收路径随后计算：

    hash & (indirections_len - 1)   /* 通过整数提升回绕到 0xFFFFFFFF */

并将结果用作 `indirections_table` 的索引，但当元素计数为零时，VMState 加载器不会分配该表（参见 `vmstate_handle_alloc()`），导致 NULL 指针解引用使 QEMU 崩溃。

RSS 子段仅在 `rss_data.enabled` 为 true 时加载（通过 `virtio_net_rss_needed()`），且命令路径始终产生 `indirections_len` 为 {1, 2, 4, …, 128} 之一，因此无条件检查不会拒绝合法的迁移流。

---

## Message #2 — Michael S. Tsirkin 的代码审查

- **发件人**: Michael S. Tsirkin <mst@redhat.com>（virtio 子系统维护者）
- **时间**: Mon, 23 Mar 2026, 约 13:40 UTC
- **Message-ID**: msg06609
- **回复**: Message #1

### 原文

```
On Mon, Mar 23, 2026 at 09:15:31PM +0800, Junjie Cao wrote:
> [引用完整的 commit message...]
>
> Signed-off-by: Junjie Cao <junjie.cao@intel.com>

Thanks for the patch!
Yet something to improve:

> +    if (n->rss_data.indirections_len == 0 ||

not needed since it's included in is_power_of_2 check below?

> +        n->rss_data.indirections_len > VIRTIO_NET_RSS_MAX_TABLE_LEN ||
> +        !is_power_of_2(n->rss_data.indirections_len)) {

with == 0 removed, the logic is duplicated from
virtio_net_device_realize.
How about factoring it out to a helper?

> +        error_report("virtio-net: saved image has invalid RSS "
> +                     "indirections_len: %u",
> +                     n->rss_data.indirections_len);
> +        return -EINVAL;
> +    }
```

### 中文翻译

感谢这个补丁！但还有一些地方可以改进：

> `if (n->rss_data.indirections_len == 0 ||`

这个不需要吧，因为下面的 `is_power_of_2` 检查已经包含了它？

> `!is_power_of_2(n->rss_data.indirections_len))`

去掉 `== 0` 之后，这段逻辑与 `virtio_net_device_realize` 中的是重复的。考虑把它提取到一个辅助函数（helper）中如何？

---

## Message #3 — Daniel P. Berrangé 质疑迁移流的不可信假设

- **发件人**: Daniel P. Berrangé <berrange@redhat.com>（QEMU 安全/迁移专家）
- **时间**: Mon, 23 Mar 2026 13:43 UTC
- **Message-ID**: msg06598
- **回复**: Message #1

### 原文

```
On Mon, Mar 23, 2026 at 09:15:31PM +0800, Junjie Cao wrote:
> virtio_net_handle_rss() enforces that indirections_len is a non-zero
> power of two no larger than VIRTIO_NET_RSS_MAX_TABLE_LEN, but
> virtio_net_rss_post_load() applies none of these checks to values
> restored from the migration stream.
>
> A crafted migration stream can set indirections_len to 0.  Even if it

The migration stream originating from the source QEMU is trusted.

Is there a problem you can demonstrate with regular QEMU commands
/ versions, not crafting a malicious migration stream.

> [后续内容...]
>
> Signed-off-by: Junjie Cao <junjie.cao@intel.com>

With regards,
Daniel
```

### 中文翻译

> 一个精心构造的迁移流可以将 `indirections_len` 设置为 0。即使它

来自源端 QEMU 的迁移流是**可信的**。

你是否能用常规的 QEMU 命令/版本来演示这个问题，而不是构造一个恶意的迁移流？

---

## Message #4 — Peter Maydell 引用安全文档反驳

- **发件人**: Peter Maydell <peter.maydell@linaro.org>（QEMU 项目联合维护者）
- **时间**: Mon, 23 Mar 2026 13:53:53 UTC
- **Message-ID**: msg06604
- **回复**: Message #3（Daniel 的质疑）

### 原文

```
On Mon, 23 Mar 2026 at 13:43, Daniel P. Berrangé <berrange@redhat.com> wrote:
>
> The migration stream originating from the source QEMU is trusted.

Is it? In https://www.qemu.org/docs/master/system/security.html we say:

# The following entities are untrusted, meaning that they may be buggy
# or malicious:

#  * Guest
#  * User-facing interfaces (e.g. VNC, SPICE, WebSocket)
#  * Network protocols (e.g. NBD, live migration)
#  * User-supplied files (e.g. disk images, kernels, device trees)
#  * Passthrough devices (e.g. PCI, USB)

which explicitly lists "live migration" as an untrusted entity.

I would definitely be extremely cautious about having a threat
model where I had to distrust inbound migration data, but the
above does suggest we aim to handle that, and we have I think
in the past taken patches which add sanity-checking to the
migration data.

thanks
-- PMM
```

### 中文翻译

> 来自源端 QEMU 的迁移流是可信的。

是吗？在 https://www.qemu.org/docs/master/system/security.html 中我们说：

> 以下实体是不可信的，意味着它们可能有 bug 或者是恶意的：
> - 客户机
> - 面向用户的接口（如 VNC、SPICE、WebSocket）
> - 网络协议（如 NBD、**热迁移**）
> - 用户提供的文件（如磁盘镜像、内核、设备树）
> - 直通设备（如 PCI、USB）

这明确将 "热迁移" 列为不可信实体。

如果我必须在威胁模型中不信任入站迁移数据，我肯定会非常谨慎，但上述内容确实表明我们的目标是处理这种情况，而且我认为我们过去也接受过给迁移数据添加合理性检查的补丁。

---

## Message #5 — Michael S. Tsirkin 确认有 CVE 先例

- **发件人**: Michael S. Tsirkin <mst@redhat.com>
- **时间**: Mon, 23 Mar 2026 09:57:24 -0400 (约 13:57 UTC)
- **Message-ID**: msg06610
- **回复**: Message #4（Peter 的回复）

### 原文

```
On Mon, Mar 23, 2026 at 01:53:53PM +0000, Peter Maydell wrote:
> [引用 Peter 关于安全文档的回复...]
>
> which explicitly lists "live migration" as an untrusted entity.
>
> I would definitely be extremely cautious about having a threat
> model where I had to distrust inbound migration data, but the
> above does suggest we aim to handle that, and we have I think
> in the past taken patches which add sanity-checking to the
> migration data.

And we even assigned a low priority CVEs to these.
```

### 中文翻译

而且我们甚至为这类问题分配过低优先级的 CVE。

---

## Message #6 — Daniel 阐述迁移流可信的立场

- **发件人**: Daniel P. Berrangé <berrange@redhat.com>
- **时间**: Mon, 23 Mar 2026 14:12 UTC
- **Message-ID**: msg06647
- **回复**: Message #4（Peter 的回复）

### 原文

```
On Mon, Mar 23, 2026 at 01:53:53PM +0000, Peter Maydell wrote:
> Is it? In https://www.qemu.org/docs/master/system/security.html we say:
> [引用安全文档...]
>
> which explicitly lists "live migration" as an untrusted entity.
>
> I would definitely be extremely cautious about having a threat
> model where I had to distrust inbound migration data, but the
> above does suggest we aim to handle that, and we have I think
> in the past taken patches which add sanity-checking to the
> migration data.

My view of the migration stream is that we authenticate the client
at the point of connection (either explicitly with SASL, or implicitly
with a x509 certificate validation), and protect the data stream
integrity with TLS, or equivalent.

For the vmstate data, we simply expect that to reflect the current
QEMU configuration, and variation of that is liable to lead to
a crash or worse.

With regards,
Daniel
```

### 中文翻译

我对迁移流的看法是：我们在连接建立时对客户端进行认证（通过 SASL 显式认证，或通过 x509 证书验证隐式认证），并通过 TLS 或等效机制保护数据流的完整性。

对于 vmstate 数据，我们只是期望它反映当前的 QEMU 配置，偏离这一点可能导致崩溃或更严重的后果。

---

## Message #7 — Peter 建议更新安全文档

- **发件人**: Peter Maydell <peter.maydell@linaro.org>
- **时间**: Mon, 23 Mar 2026 15:33:37 UTC
- **Message-ID**: msg06653
- **回复**: Message #6（Daniel 的立场阐述）

### 原文

```
On Mon, 23 Mar 2026 at 14:12, Daniel P. Berrangé <berrange@redhat.com> wrote:
>
> My view of the migration stream is that we authenticate the client
> at the point of connection (either explicitly with SASL, or implicitly
> with a x509 certificate validation), and protect the data stream
> integrity with TLS, or equivalent.

This seems like it would be useful clarifying information to
add to the security.rst document under the "Sensitive configurations"
subsection as part of documenting what we recommend, even if we
decide that we want to continue treating malicious migration-data
as part of our threat model / security boundary.

-- PMM
```

### 中文翻译

这似乎是很有用的澄清信息，可以添加到 `security.rst` 文档的 "敏感配置" 子节中，作为我们推荐做法的文档记录，即使我们决定继续将恶意迁移数据视为我们威胁模型/安全边界的一部分。

---

## Message #8 — Daniel 要求提供 CVE 示例

- **发件人**: Daniel P. Berrangé <berrange@redhat.com>
- **时间**: Mon, 23 Mar 2026 14:56 UTC
- **Message-ID**: msg06649
- **回复**: Message #5（Michael 提到 CVE）

### 原文

```
On Mon, Mar 23, 2026 at 09:57:24AM -0400, Michael S. Tsirkin wrote:
> And we even assigned a low priority CVEs to these.

Do you have an example of that ?

With regards,
Daniel
```

### 中文翻译

你有这方面的例子吗？

---

## Message #9 — Peter 提供 CVE 示例

- **发件人**: Peter Maydell <peter.maydell@linaro.org>
- **时间**: Mon, 23 Mar 2026 15:25:10 UTC
- **Message-ID**: msg06652
- **回复**: Message #8（Daniel 要求示例）

### 原文

```
On Mon, 23 Mar 2026 at 14:56, Daniel P. Berrangé <berrange@redhat.com> wrote:
>
> Do you have an example of that ?

Here's one from last year:
https://access.redhat.com/security/cve/cve-2025-54566

I vaguely recall we had a set of them some years ago too
(probably somebody specifically looking for flaws in the
category). https://access.redhat.com/security/cve/cve-2013-4536
might be an example of that.

-- PMM
```

### 中文翻译

这是去年的一个：
https://access.redhat.com/security/cve/cve-2025-54566

我依稀记得几年前我们也有一批这样的（可能是有人专门在这个类别中寻找缺陷）。https://access.redhat.com/security/cve/cve-2013-4536 可能就是其中一个例子。

---

## Message #10 — Daniel 的长篇深入回复（立场转变的关键邮件）

- **发件人**: Daniel P. Berrangé <berrange@redhat.com>
- **时间**: Mon, 23 Mar 2026 15:58 UTC
- **Message-ID**: msg06665
- **回复**: Message #9（Peter 的 CVE 示例）

### 原文

```
On Mon, Mar 23, 2026 at 03:25:10PM +0000, Peter Maydell wrote:
> Here's one from last year:
> https://access.redhat.com/security/cve/cve-2025-54566

I think this one is valid, because it involves incorrect handling
of settings that are controlled by the guest OS. There's no
external party claimed to be modifying the migration stream IIUC

> I vaguely recall we had a set of them some years ago too
> (probably somebody specifically looking for flaws in the
> category). https://access.redhat.com/security/cve/cve-2013-4536
> might be an example of that.

I think this one is probably issued in error.

If we're considering that the migration data stream is modifiable
by an attacker, the implication is that the attacker has arbitrary
read and write over the entire of guest RAM. That in turn implies
no guest OS can ever be trusted after a migration has been performed,

That is certainly not the case though.

We establish trust in the guest RAM after migration by protecting
the live migration data stream with TLS (or an equivalent mechanism
external to QEMU), and including some mechanism for authenticating
the connections. We prove the incoming connection is from the
expected source QEMU. That protection also applies to the VMstate
data.

So the only entity would can give us malicious vmstate data would
be the source QEMU.

The risk would appear to be that the source QEMU has been compromised,
but has been unable to break out into the source host OS. The migration
data would thus be leveraged as a way to break out into the target host
OS instead.

This feels dubious though. The source QEMU and target QEMU would be
assumed to be using the same security facilities (running non-root,
using seccomp, using selinux/apparmor, using namespaces, etc). So if
it could not break out of QEMU on the source host, I can't see that
migration would make it possible todo that on the target host either.


The save/restore of VM state to/from disk, for the purpose of VM
snapshotting is slightly different as  we don't have a network
channel involved.

None the less, I'd claim that the saved snapshot file must be considered
trusted, because it again contains data (guest RAM) that we inherently
must trust. If an attacker has the ability to modify a saved state file,
then we've already lost all trust.


I can see scope for CVEs in QEMU wrt vmstate, if:

 * a malicious guest OS driver is able to configure some aspect
   of a virtual device, such that it then causes misbehaviour in
   later handling of the VM state.

   IIUC the recent cve-2025-54566 is an example of that


 * untrusted data being processed by the virtual device, causes
   the device to get itself into a state which then causes
   misbehaviour in handling of VM state.


I don't accept scope for CVEs in QEMU for an external attacker modifying
the migration data stream or saved state file contents though. AFAICS,
such possibilities imply gross misconfiguration of QEMU by the mgmt app,
and should be CVEs in the mgmt app instead.

With regards,
Daniel
```

### 中文翻译

> 这是去年的一个：CVE-2025-54566

我认为这个是有效的，因为它涉及对由客户机操作系统控制的设置的不正确处理。据我理解，没有外部方被声称在修改迁移流。

> CVE-2013-4536 可能就是其中一个例子。

我认为这个可能是错误签发的。

如果我们认为迁移数据流可以被攻击者修改，那意味着攻击者对整个客户机 RAM 拥有任意读写权限。这反过来意味着在迁移完成后，没有任何客户机操作系统可以被信任。

但事实肯定不是这样的。

我们通过用 TLS（或 QEMU 外部的等效机制）保护热迁移数据流，并包含某种连接认证机制来建立迁移后对客户机 RAM 的信任。我们证明传入连接来自预期的源端 QEMU。该保护同样适用于 VMstate 数据。

因此，唯一能给我们提供恶意 vmstate 数据的实体是源端 QEMU。

风险似乎在于源端 QEMU 已被入侵，但无法突破到源端宿主机操作系统。迁移数据因此可能被利用作为突破到目标宿主机操作系统的途径。

但这感觉很牵强。源端 QEMU 和目标端 QEMU 应该使用相同的安全设施（非 root 运行、使用 seccomp、使用 SELinux/AppArmor、使用命名空间等）。如果它无法在源端宿主机上突破 QEMU，我看不出迁移如何使其能在目标宿主机上做到这一点。

VM 状态的磁盘保存/恢复（用于虚拟机快照目的）略有不同，因为不涉及网络通道。

尽管如此，我认为保存的快照文件必须被视为可信的，因为它同样包含我们必须内在信任的数据（客户机 RAM）。如果攻击者有能力修改保存的状态文件，那么我们已经失去了所有信任。

我能看到 QEMU 在 vmstate 方面 CVE 的范围，如果：

* 恶意客户机操作系统驱动能够配置虚拟设备的某些方面，使其在后续的 VM 状态处理中导致异常行为。据我理解，最近的 CVE-2025-54566 就是一个例子。

* 虚拟设备处理的不受信任数据，导致设备自身进入一种状态，随后导致 VM 状态处理异常行为。

但对于外部攻击者修改迁移数据流或保存状态文件内容的 CVE 范围，我**不予接受**。在我看来，这种可能性意味着管理应用对 QEMU 的严重错误配置，应该是管理应用的 CVE。

---

## Message #11 — Peter 回复，呼吁 Paolo 和 Stefan 参与讨论

- **发件人**: Peter Maydell <peter.maydell@linaro.org>
- **时间**: Mon, 23 Mar 2026 16:15:56 UTC
- **Message-ID**: msg06667
- **回复**: Message #10（Daniel 的长篇回复）

### 原文

```
On Mon, 23 Mar 2026 at 15:58, Daniel P. Berrangé <berrange@redhat.com> wrote:
>
> I think this one is valid, because it involves incorrect handling
> of settings that are controlled by the guest OS. There's no
> external party claimed to be modifying the migration stream IIUC

I think the "guest OS sets things wrongly" is CVE-2025-54567,
and 54566 is specifically for the "migration stream is malicious"
case. The commit fixing them:
https://gitlab.com/qemu-project/qemu/-/commit/cad9aa6fbdccd95e56e10cfa57c354a20a333717
fixes both at the same time, by providing a validation function
and calling it both (a) when the guest OS writes the settings
and (b) in post_load. If we trust migration data then we don't need
to validate it in post-load, we could just say "make sure your
source QEMU has the fix for (a) and that you've restarted the guest".

> I think this one is probably issued in error.
>
> If we're considering that the migration data stream is modifiable
> by an attacker, the implication is that the attacker has arbitrary
> read and write over the entire of guest RAM. That in turn implies
> no guest OS can ever be trusted after a migration has been performed,

> That is certainly not the case though.

Trusted by whom? If I'm a cloud vendor then I don't trust the
guest OS in the first place. QEMU itself should consider everything
in guest RAM untrusted data, because it's guest-controlled.
You can probably have setups where the user who owns the VM might
be able to modify the migration stream but no third party can
(e.g. if you give them the ability to vmsave/vmload to a file
that they own), in the same way you can give the VM owner the
ability to directly write to the disk images the VM is using
without that being a way for them to escape to the host.

> I don't accept scope for CVEs in QEMU for an external attacker modifying
> the migration data stream or saved state file contents though. AFAICS,
> such possibilities imply gross misconfiguration of QEMU by the mgmt app,
> and should be CVEs in the mgmt app instead.

I agree that you have a pretty weird threat/operating model if
you allow the migration-stream to be considered untrusted, because
it's reasonable and sensible to secure it. But I have a vague
recollection that this language is in the doc precisely because
at least one person/party has argued in the past in favour of
treating the migration stream as on the security boundary. I don't
object to our deciding we want to call the migration-stream trusted --
but I do think this is a policy change from our current stance.

Paolo, Stefan, do you happen to remember anything about this ?

thanks
-- PMM
```

### 中文翻译

> 我认为这个是有效的，因为它涉及由客户机操作系统控制的设置的不正确处理。

我认为 "客户机操作系统错误设置" 是 CVE-2025-54567，而 54566 专门针对 "迁移流是恶意的" 这种情况。修复它们的提交 https://gitlab.com/qemu-project/qemu/-/commit/cad9aa6fbdccd95e56e10cfa57c354a20a333717 同时修复了两者，通过提供一个验证函数并在以下两个地方调用它：(a) 当客户机操作系统写入设置时，以及 (b) 在 `post_load` 中。如果我们信任迁移数据，那么就不需要在 `post_load` 中验证它，我们可以直接说 "确保你的源端 QEMU 有 (a) 的修复并且已经重启了客户机"。

> 这肯定不是这样的。

被谁信任？如果我是一个云供应商，那我首先就不信任客户机操作系统。QEMU 本身应该将客户机 RAM 中的一切都视为不受信任的数据，因为它是客户机控制的。你可能有这样的配置场景：拥有 VM 的用户可以修改迁移流但第三方不能（例如，如果你允许他们将 vmsave/vmload 到他们拥有的文件），就像你可以让 VM 所有者直接写入 VM 正在使用的磁盘镜像，而这不会成为他们逃逸到宿主机的途径。

> 对于外部攻击者修改迁移数据流或保存状态文件内容的 CVE 范围，我不予接受。

我同意，如果你允许迁移流被视为不可信，那你的威胁/操作模型确实很奇特，因为保护它是合理且明智的。但我依稀记得文档中的这段语言正是因为过去至少有一个人/组织主张将迁移流视为安全边界。我不反对我们决定将迁移流称为可信的——但我确实认为这是对我们当前立场的**政策变更**。

**Paolo, Stefan, 你们碰巧记得关于这件事的什么吗？**

---

## Message #12 — Daniel 的最终回复（立场部分软化）

- **发件人**: Daniel P. Berrangé <berrange@redhat.com>
- **时间**: Mon, 23 Mar 2026 16:45:06 UTC
- **Message-ID**: msg06673
- **回复**: Message #11（Peter 呼吁更多人参与）

### 原文

```
On Mon, Mar 23, 2026 at 04:15:56PM +0000, Peter Maydell wrote:
>
> I think the "guest OS sets things wrongly" is CVE-2025-54567,
> and 54566 is specifically for the "migration stream is malicious"
> case. The commit fixing them:
> https://gitlab.com/qemu-project/qemu/-/commit/cad9aa6fbdccd95e56e10cfa57c354a20a333717
> fixes both at the same time, by providing a validation function
> and calling it both (a) when the guest OS writes the settings
> and (b) in post_load. If we trust migration data then we don't need
> to validate it in post-load, we could just say "make sure your
> source QEMU has the fix for (a) and that you've restarted the guest".

Ah, yes, the pairing of CVEs is key here.

Our users expect their workloads to execute without interruption.
Thus our story for fixing CVEs bugs without guest impact is to
live migrate the guest onto the fixed CVE.

So in this case, I think we do need the fix in post_load as a
way to mitigate the root cause CVE in existing workloads that
cannot be restarted.

> Trusted by whom? If I'm a cloud vendor then I don't trust the
> guest OS in the first place. QEMU itself should consider everything
> in guest RAM untrusted data, because it's guest-controlled.
> You can probably have setups where the user who owns the VM might
> be able to modify the migration stream but no third party can
> (e.g. if you give them the ability to vmsave/vmload to a file
> that they own), in the same way you can give the VM owner the
> ability to directly write to the disk images the VM is using
> without that being a way for them to escape to the host.

Hmm, yes, load/save into a file that the guest owner controls
is a good point. I'm fairly sure I recall that OpenStack allows
for exactly that scenario. So the host owner would need to
treat the vmstate as potentially hostile in that case.

IOW, bugs that affect handling of vm state that merely result
in an assert are merely self-inflicted DoS by the guest owner.
Bugs that could be leveraged to exploit QEMU itself should be
CVE worthy.

So even if the migration stream can be considerd trusted, we
can't trust vmstate in general due to the load/save scenario.

This is probably a case where we ought to have had a host
controlled digital signature over vmstate data to validate
its integrity.

> I agree that you have a pretty weird threat/operating model if
> you allow the migration-stream to be considered untrusted, because
> it's reasonable and sensible to secure it. But I have a vague
> recollection that this language is in the doc precisely because
> at least one person/party has argued in the past in favour of
> treating the migration stream as on the security boundary. I don't
> object to our deciding we want to call the migration-stream trusted --
> but I do think this is a policy change from our current stance.

Possibly we should clarify the language to say that the vmstate
format is untrusted due to the load/save possibility to guest
owner controlled storage.

With regards,
Daniel
```

### 中文翻译

啊，对的，CVE 的配对是这里的关键。

我们的用户期望他们的工作负载不间断运行。因此我们修复 CVE 漏洞而不影响客户机的方式就是将客户机热迁移到已修复 CVE 的版本上。

所以在这种情况下，我认为我们**确实需要在 `post_load` 中进行修复**，作为在无法重启的现有工作负载中缓解根因 CVE 的方式。

> 如果你允许他们将 vmsave/vmload 到他们拥有的文件……

嗯，对，加载/保存到客户机所有者控制的文件中是一个很好的观点。我相当确定 OpenStack 就允许这种场景。所以在这种情况下，宿主机所有者需要将 vmstate 视为潜在的敌意数据。

换句话说，影响 VM 状态处理但仅仅导致断言失败（assert）的 bug，只是客户机所有者对自己的拒绝服务。但可以被利用来攻击 QEMU 本身的 bug 应该是值得分配 CVE 的。

**所以即使迁移流可以被视为可信，由于加载/保存场景，我们也无法在一般意义上信任 vmstate。**

这可能是一个我们本应对 vmstate 数据设置宿主机控制的数字签名来验证其完整性的场景。

> 但我确实认为这是对我们当前立场的政策变更。

也许我们应该澄清措辞，说明 **vmstate 格式由于加载/保存到客户机所有者控制的存储的可能性而不可信**。

---

## 讨论线程结构图

```
#1  Junjie Cao — [PATCH] 原始补丁提交
├── #2  Michael S. Tsirkin — 代码审查（冗余检查、建议提取 helper）
├── #3  Daniel P. Berrangé — 质疑："迁移流来自源端 QEMU，是可信的"
│   └── #4  Peter Maydell — 反驳：引用安全文档，迁移被列为不可信
│       ├── #5  Michael S. Tsirkin — "我们甚至为这类问题分配过低优先级 CVE"
│       ├── #6  Daniel — 阐述立场：TLS+认证保护，VMstate 应可信
│       │   └── #7  Peter — 建议更新 security.rst 文档
│       ├── #8  Daniel — "有例子吗？"
│       │   └── #9  Peter — CVE-2025-54566, CVE-2013-4536
│       │       └── #10 Daniel — 长篇回复，区分客户机触发 vs 外部篡改
│       │           └── #11 Peter — 区分 CVE-54566 vs 54567，
│       │                          引入 vmsave/vmload 场景，呼吁 Paolo/Stefan
│       │               └── #12 Daniel — **立场软化**：承认 load/save 场景使
│       │                                vmstate 不可信，同意需要 post_load 修复
```
