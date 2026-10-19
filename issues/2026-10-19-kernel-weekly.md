# 🔍 内核邮件列表周刊 — 2026-10-19

> 自动爬取 lore.kernel.org 邮件列表，AI 分析生成
> 覆盖时间：2026-10-12 至 2026-10-19
> 数据源：linux-kernel, netdev, linux-mm, linux-block, linux-pm, linux-security-module, stable, regressions

---

## 📊 本周概览

| 指标 | 数量 |
|------|------|
| 邮件总数 | 32,410 |
| 原始帖子（非回复） | 17,788 |
| 回复邮件 | 14,622 |
| 补丁邮件 | 16,051 |
| RFC 提案 | 627 |
| GIT PULL 请求 | 96 |
| CVE 相关 | 31（14 个唯一 CVE） |
| syzbot 报告 | 205 |
| 回归报告 | ~20 |

### 各邮件列表分布

| 邮件列表 | 总邮件数 |
|----------|---------|
| linux-kernel | 18,459 |
| stable | 7,410 |
| netdev | 3,552 |
| linux-mm | 1,492 |
| linux-pm | 633 |
| linux-block | 407 |
| linux-security-module | 320 |
| regressions | 137 |

### 子系统补丁分布（估算）

| 子系统 | 补丁数 |
|--------|--------|
| 网络 (net/wifi/bluetooth/can/netfilter) | ~2,551 |
| 设备树绑定 (dt-bindings) | ~971 |
| ARM64 / ARM | ~924 |
| 内存管理 (mm) | ~598 |
| SCSI | ~561 |
| KVM | ~527 |
| ASoC / ALSA | ~883 |
| HID / Input | ~409 |
| BPF | ~402 |
| perf | ~384 |
| USB | ~320 |
| PCI | ~302 |
| 媒体 (media) | ~427 |
| XFS / Btrfs / F2FS | ~911 |
| Rust | ~198 |
| DRM/AMD | ~174 |

### 本周重大事件

🎉 **Linux 7.3 正式发布**（2026-10-18，Linus Torvalds 宣布）。v7.4 合并窗口随即开启，本周已收到大量子系统 GIT PULL 请求。

### 本周稳定版发布

| 版本 | 候选 | 补丁数 |
|------|------|--------|
| 6.6.160 | -rc1 | 1,674 |
| 6.12.113 | -rc1 / -rc2 | 497 / 505 |
| 6.12.114 | -rc1 | 869 |
| 6.18.57 | -rc1 / -rc2 | 621 / 632 |
| 6.18.58 | -rc1 | 52 |
| 7.2.11 | -rc1 / -rc2 | 281 / 284 |
| 7.2.12 | -rc1 | 10 |
| 6.1.190-rt70 | PREEMPT_RT | — |
| 6.6.159-rt81 | PREEMPT_RT | — |

> 本周稳定版维护活跃，4 个稳定分支共发布 RC，合计约 3,965 个补丁被回 port。6.6.160-rc1 以 1,674 个补丁居首。

---

## 🔴 严重问题

> 本周发现 7 个严重问题，涉及安全漏洞、内核崩溃和内存安全。

### 1. AppArmor cred UAF：begin_current_label_crit_section() 可导致凭证释放后使用

- **邮件列表**: linux-security-module / stable
- **发件人**: Jann Horn (Google) / Artem Dinaburg (Trail of Bits，6.6.y 回 port)
- **类型**: 安全漏洞 / use-after-free
- **严重程度**: 🔴 严重
- **影响子系统**: security/apparmor
- **描述**: AppArmor 的 `begin_current_label_crit_section()` 在大量 LSM 钩子（特别是 VFS/socket 相关）中被调用。当标签标记为 `FLAG_STALE` 时，它尝试通过 `aa_replace_current_label()` 替换当前凭证。这存在两个问题：(1) 如果内核其他代码在安全钩子调用后仍持有旧 cred 指针，直接替换 cred 会导致 UAF；(2) 当使用 `override_creds()` 覆盖凭证时，`aa_replace_current_label()` 的检测不可靠，可导致 `task->real_cred` 指向已释放的凭证。Jann Horn 构造了实际可复现的 KASAN UAF 测试用例。
- **修复方案**: 将 cred 替换从直接调用改为通过 `task_work` 在当前系统调用结束时异步执行，避免在 LSM 钩子中直接修改 cred。已回 port 到 6.6.y 稳定分支。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/linux-security-module/20261014193517.24940-1-artem@trailofbits.com/)

### 2. SELinux avtab 损坏导致通用保护错误（syzkaller 发现）

- **邮件列表**: linux-security-module
- **发件人**: 1sh1ro
- **类型**: bug / 内核崩溃
- **严重程度**: 🔴 严重
- **影响子系统**: security/selinux
- **描述**: 在 7.3.0-rc5 KASAN 内核（SELinux 启用，permissive 模式）上使用 syzkaller fuzzing 时，`avtab_search_node()` 在 AVC 查找路径中触发通用保护错误。策略的 avtab 似乎包含垃圾数据：首次错误读取非规范地址，随后引发数千个进一步 Oops（本次运行 8850 个）。崩溃发生在 `systemd-udevd` 执行 `setxattr()` 时，路径遍历跟随符号链接触发 AVC 决策命中损坏的 avtab。该问题在三次独立 fuzzing 运行中均可复现。
- **修复方案**: 问题仍在调查中，需要确定 avtab 损坏的根因（可能是策略加载时的竞态或内存损坏）。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/linux-security-module/20261012223824.41804-2-selinux-avtab@yangguikun19.qq.com/)

### 3. blk-crypto NULL 解引用和 UAF：dm-inlinecrypt 初始化失败路径

- **邮件列表**: linux-block
- **发件人**: Baokun Li (阿里巴巴)
- **类型**: bug / NULL deref + UAF
- **严重程度**: 🔴 严重
- **影响子系统**: block/blk-crypto
- **描述**: 当 `blk_crypto_fallback_init()` 失败时，`kfree(profile)` 释放了 profile 但未置 NULL。随后 `inlinecrypt_dtr()` 调用 `blk_crypto_evict_key()` → `__blk_crypto_evict_key()`，访问已释放的 `profile->num_slots`，触发 NULL 解引用（实际是 UAF）。可通过 dmsetup 创建 inlinecrypt 设备复现，在 7.3.0-rc3 KASAN 内核上触发 `general protection fault`。
- **修复方案**: 在 `blk_crypto_fallback_start_using_key()` 失败时正确清理 profile 指针，并增加 NULL 检查。补丁已发布 v2 版本。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/linux-block/20261013095106.881914-1-libaokun@linux.alibaba.com/)

### 4. x86/kaslr：vmemmap 可与 CPU 入口区重叠导致 amdgpu Oops

- **邮件列表**: linux-mm
- **发件人**: Stefan
- **类型**: bug / 内核崩溃
- **严重程度**: 🔴 严重
- **影响子系统**: arch/x86, mm, amdgpu
- **描述**: 在 4 级页表的 x86-64 机器上（AMD Ryzen AI Max+ 395 / Strix Halo），KASLR 可将 vmemmap 放置为结束地址超出 `vaddr_end` 1 GiB，导致 vmemmap 区域与 CPU_ENTRY_AREA_BASE（只读 IDT 别名）重叠。当 amdgpu (amdkfd) 设置 DEVICE_PRIVATE 内存时，`memmap_init_zone_device()` 向 `0xfffffe0000000000`（CPU 入口区基址）写入 struct page，触发页面错误。约 30 次启动中出现 1 次。根因：`kernel_randomize_memory()` 未将 PUD_SIZE 间隙从 `remain_entropy` 中扣除，且自 `7ffb7914` 和 `269031b1` 起 vmemmap 顶部实际被使用。
- **修复方案**: 提出了一行修复，在 `kernel_randomize_memory()` 中正确扣除区域间隙的熵。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/linux-mm/20261016150116.045010-1-stefan@maekto.com/)

### 5. KASAN slab-out-of-bounds：copy_folio_from_iter_atomic (ext4)

- **邮件列表**: linux-block
- **发件人**: Haocheng Yu
- **类型**: bug / 内存越界
- **严重程度**: 🔴 严重
- **影响子系统**: lib/iov_iter, fs/ext4
- **描述**: 在 Linux v7.3.0-rc5 上 fuzzing 时发现 `copy_folio_from_iter_atomic()` 中的 slab-out-of-bounds 读取。KASAN 报告在 `kmalloc-192` 分配的 144 字节之后 8 字节处读取了 8 字节。崩溃路径：`writev()` → `ext4_file_write_iter()` → `generic_perform_write()` → `copy_folio_from_iter_atomic()` → `iterate_iovec`。复现方式为挂载 ext4 镜像并通过精心构造的 iovec 数组执行写操作。
- **修复方案**: 问题仍在调查中，发帖者寻求社区帮助确认根因和修复方案。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/linux-block/20261018081203.2431035-1-l1za0.sec@gmail.com/)

### 6. CVE-2026-89603：seccomp 绕过（⚠️ 有争议）

- **邮件列表**: linux-kernel
- **发件人**: Kees Cook / Oleg Nesterov / Greg Kroah-Hartman
- **类型**: 安全漏洞（⚠️ 争议）
- **严重程度**: 🔴 严重（原始评级） / ⚠️ 争议
- **影响子系统**: kernel/entry, kernel/seccomp
- **描述**: 该 CVE 声称 ptrace + TSYNC 场景下存在竞态条件，可允许非特权进程执行被过滤的系统调用。但 Kees Cook 在邮件中**明确请求拒绝此 CVE**，指出：遗漏的检查仅影响线程的第一个过滤器，且 tracer 已完全控制 tracee，该窗口在没有 ptrace 的情况下同样存在（TSYNC 从不应用于已进入的系统调用）。修复补丁是一致性改进，但不构成实际安全漏洞。
- **修复方案**: 修复补丁作为一致性改进保留，但 CVE 本身被请求拒绝。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/all/20261015162119.1a1e0d-kees@kernel.org/)

### 7. syzbot 集中报告多个 use-after-free 和越界访问

- **邮件列表**: 多个列表
- **发件人**: syzbot
- **类型**: bug / 内存安全
- **严重程度**: 🔴 严重
- **影响子系统**: 多个子系统
- **描述**: 本周 syzbot 报告了 205 个问题，其中严重的包括：
  - `KASAN: slab-use-after-free Read in drm_sysfs_hotplug_event` (dri)
  - `KASAN: slab-out-of-bounds Read in pn533_recv_frame` (nfc)
  - `BUG: unable to handle kernel paging request in ndisc_alloc_skb` (net)
  - `kernel BUG in pskb_expand_head` (tipc)
  - `possible deadlock in lapb_connect_request` (x25)
  - `BUG: workqueue lockup in key_garbage_collector` (keyrings/lsm)
  - `general protection fault in smack_inode_getattr` (lsm)
  - `INFO: rcu detected stall in filemap_fault` (mm)
  - `KASAN: slab-use-after-free Write in config_item_put` (fs)
  - `INFO: task hung in pn533_usb_send_frame` (nfc)

---

## 🟠 高优先级问题

> 本周发现 10 个高优先级问题，涉及严重回归和功能不可用。

### 1. mt7925e 休眠恢复挂起（7.3 回归，修复仅排队至 7.4）

- **邮件列表**: regressions
- **发件人**: Илья Шидловский
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: net/wireless/mt7925
- **描述**: 提交 `c4edf9e3e0e5` ("wifi: mt76: mt7925: replace shared rx irq masks with per-chip definitions") 在 v7.3-rc1 合入，破坏了 MT7925 (PCIe) 的休眠恢复。镜像恢复后，芯片重置路径进入 MCU_WA NAPI（MT7925 上从未设置），导致机器硬挂（无控制台输出，只能断电）。修复 `c182bfe944eb` 已在 wireless-next 中，但仅排队至 7.4-rc1。v7.3 将带此回归发布。
- **修复方案**: 修复补丁 `c182bfe944eb` 已存在，发帖者请求通过无线树推送到 7.3 或至少回 port 到 7.3.y stable。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/regressions/20261012175612.1-1-shidlovskiyilya@gmail.com/)

### 2. DRM AMD 显示：DCN32 120Hz 单帧扫描损坏（7.3 回归）

- **邮件列表**: regressions
- **发件人**: Mikhail Gavrilov
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: drm/amd/display
- **描述**: 自提交 `a637a140ab86` 起，DCN32 在 120Hz 下出现单帧扫描损坏。问题与 SubVP（子视口）在高刷新率 FreeSync 显示器上的启用有关。`dcn32_allow_subvp_high_refresh_rate()` 通过读取 `stream->allow_freesync` 来禁用 SubVP，但该字段仅在首次 flip 后设置，因此恢复后的 modeset 验证时该字段未设置。S3 恢复后问题复发。
- **修复方案**: 发帖者提交了补丁修复 `allow_freesync` 字段的时序问题，确保 SubVP 在 rtcwake 后保持禁用。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/regressions/20261018221726.27259-1-mikhail.v.gavrilov@gmail.com/)

### 3. USB 音频：Behringer UMC204HD 在 7.2.10 中损坏

- **邮件列表**: regressions
- **发件人**: Will Bush
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: sound/usb-audio
- **描述**: 从 7.2.9 升级到 7.2.10 后，Behringer UMC204HD 音频设备停止工作。设备进入循环：不断枚举失败，`failed to read device descriptor: -110`。根因是提交 `b5c4823cf3e4` ("ALSA: usb-audio: Apply boot quirk for Behringer models generically") 将 boot quirk 过于宽泛地应用于所有 Behringer 设备（vendor ID 0x1397），而不仅限特定型号。
- **修复方案**: 将 quirk 条件从按 vendor ID 匹配改为按具体 USB ID 匹配。Takashi Iwai 已参与讨论。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/regressions/20261018113427.1-1-will@bush.id.au/)

### 4. mac80211_hwsim：CCMP 帧丢弃 + 挂起后信标停止

- **邮件列表**: regressions
- **发件人**: Loïc KASSEL
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: net/mac80211
- **描述**: 两个独立回归：(1) fast-xmit 路径中 CCMP 帧被丢弃，`ccmp_encrypt_skb` 触发 WARNING (net/mac80211/wpa.c:462)；(2) 主机挂起后信标停止，因为 `beacon_timer` 使用 `CLOCK_MONOTONIC` 但过期时间用 boottime 计算。
- **修复方案**: 需要分别修复 fast-xmit CCMP 加密路径和信标定时器的时钟类型。

### 5. SMB 客户端：SMB3 加密下 12 MiB 以上 EIO（6.12.y 回归）

- **邮件列表**: regressions
- **发件人**: Paul Aurich / Paulo Alcantara
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: fs/smb/client
- **描述**: 在 6.12.y 上，启用 SMB3 加密时超过 12 MiB 的写入返回 EIO。回归由批量 SRV_COPYCHUNK 条目优化引入。Greg KH 已介入讨论。
- **修复方案**: Paulo Alcantara 正在准备修复补丁。

### 6. xhci：设备从总线脱落且可能永久不可用

- **邮件列表**: regressions / linux-kernel
- **发件人**: Yiyu Zhou
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: usb/xhci
- **描述**: USB 设备从总线脱落，可能需要重启才能恢复。问题与端点重置后跳过 TD 回收有关。
- **修复方案**: v2 补丁 `xhci: give back all TDs when skipping endpoint reset after disconnect` 已提交，确保跳过端点重置时回收所有 TD。
- **补丁链接**: [lore.kernel.org](https://lore.kernel.org/lkml/20261013003711.1-1-yiyu@yiyuzhou.io/)

### 7. PHY: Qualcomm QMP-PCIe 链路从不重训

- **邮件列表**: regressions
- **发件人**: Michael Burch
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: phy/qcom/qmp-pcie
- **描述**: 自提交 "phy: qcom: qmp-pcie: Skip PHY reset if already up" 后，PCIe 链路从不重训，根端口上线时无端点设备。
- **修复方案**: 发帖者提交了 revert 补丁。

### 8. ASoC: cs35l56 DAI 未注册，无声卡（7.2.11 回归）

- **邮件列表**: regressions
- **发件人**: Connor Stevens
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: sound/soc/sdw_utils
- **描述**: 7.2.11 上 cs35l56-ot25 DAI 未注册，导致无声音设备。SoundWire utils 相关回归。

### 9. perf: 核心上下文重写破坏多路复用 cgroup 事件

- **邮件列表**: regressions
- **发件人**: Antoine HURET (Bull / Atos)
- **类型**: 回归问题
- **严重程度**: 🟠 高
- **影响子系统**: perf/core
- **描述**: perf 核心上下文处理重写破坏了多路复用 cgroup 事件的功能。影响生产环境中的性能监控。
- **修复方案**: 讨论进行中，需要修复核心上下文处理中的 cgroup 事件多路复用逻辑。

### 10. stress-ng.kvm 性能回归 28.2%

- **邮件列表**: linux-kernel
- **发件人**: kernel test robot (Intel)
- **类型**: 回归问题 / 性能
- **严重程度**: 🟠 高
- **影响子系统**: fs
- **描述**: 内核测试机器人报告提交 `7eb84d54fa` ("fs: don't let a migrating task hide its reference from do_umount()") 导致 `stress-ng.kvm.ops_per_sec` 性能下降 28.2%。测试环境：Intel Xeon 6740E (Sierra Forest) 192 线程，256G 内存。
- **修复方案**: 需要重新评估该提交对 KVM 工作负载的影响。

---

## 🟡 中等问题

> 本周发现大量中等优先级修复，以下列出重点关注项。

### 安全修复

1. **CVE-2026-89566 / CVE-2026-89567: jbd2 回 port** — Artem Dinaburg (Trail of Bits) — jbd2 相关 CVE 的 6.6.y 回 port 修复系列。

2. **CVE-2026-64017: 块设备安全修复回 port** — Artem Dinaburg — 6.6.y 回 port 修复系列。

3. **keys: Protect quota-owner reads in namespace filters** — Jarkko Sakkinen / Chengfeng Ye — 命名空间过滤器中保护配额所有者读取的修复（v4 版本）。

4. **keys/trusted_keys: PKWM wrapping key 系列** — Srish Srinivasan — 添加 PKWM wrapping key 选择和敏感缓冲区清理（v5，9 个补丁）。

### 网络修复

5. **ibmveth: 修复挂起、UAF 和 netpoll 竞态** — Mingming Cao — 9 个补丁的系列，修复 ibmveth 驱动中的多个严重问题（v5 版本）。

6. **net: team: stop reusing skb after queue override** — stable 回 port 失败的补丁，需在 5.10/5.15/6.1/6.6/6.12 多个稳定分支修复。

7. **wifi: ath9k: reject short WMI command responses** — 安全相关修复，防止短 WMI 命令响应。

8. **net: qrtr: ns: retry service announcements on -EAGAIN** — QRTR 命名空间服务公告重试修复，影响多平台。

### 内存管理

9. **NFSD: keep direct-mode I/O out of the page cache** — Chuck Lever / Mike Snitzer — v4 版本 10 补丁系列，重构 NFSD 直接模式 I/O 处理，涉及边界页面分割和追踪。

10. **ocfs2-dlm: hold reference across convert/grant AST enqueue** — Andrew Morton — 修复 ocfs2 DLM 中的引用计数问题（mm-nonmm-unstable 分支）。

### 文件系统

11. **Rust XArray for v7.4** — Daniel Gomez — Rust 绑定的 XArray 支持，通过 GIT PULL 请求合入 v7.4。

12. **hfs/hfsplus changes for v7.4-rc1** — HFS/HFS+ 文件系统更新。

13. **nilfs2 changes for v7.4-rc1** — nilfs2 文件系统更新。

### 架构相关

14. **arm64 updates for v7.4** — Catalin Marinas — ARM64 架构更新通过 GIT PULL 请求。

15. **s390 updates for v7.4 merge window** — s390 架构更新。

16. **KVM: x86: VMX+TDX / SVM / MMU / Misc changes for 7.4** — Sean Christopherson — 四个独立 GIT PULL 请求，涵盖 KVM x86 的 VMX+TDX、SVM、MMU 和杂项变更。

### 核心子系统

17. **Scheduler updates for v7.4** — Ingo Molnar — 调度器更新。

18. **Performance events updates for v7.4** — Ingo Molnar — perf 事件更新。

19. **locking updates for v7.4** — Ingo Molnar — 锁机制更新。

20. **RCU updates for v7.4** — Paul E. McKenney — RCU 更新，包括 CSD-lock 停顿恢复时间报告和两字节 cmpxchg 仿真。

21. **objtool updates for v7.4** — Ingo Molnar — objtool 更新。

22. **Rust for v7.4** — Miguel Ojeda — Rust 支持更新，包括 cpufreq、clk 等新绑定。

### 驱动更新

23. **cpufreq/arm updates for 7.4** — Viresh Kumar — CPU 频率驱动更新。

24. **thermal drivers for v7.4-rc1** — Daniel Lezcano — 温度传感器驱动更新。

25. **interconnect changes for 7.4** — Georgi Djakov — 互连框架更新。

26. **md-7.4** — Yu Kuai — MD/RAID 驱动更新。

27. **Driver core changes for 7.4-rc1** — Danilo Krummrich — 驱动核心框架更新。

28. **chrome-platform updates for v7.4** — Tzung-Bi Shih — Chrome 平台驱动更新。

29. **USB serial updates for 7.4-rc1** — USB 串口驱动更新。

30. **pwm: Changes for 7.4-rc1** — PWM 子系统更新。

---

## 🟢 常规更新

### 稳定版维护

- Greg Kroah-Hartman 持续维护 6.6/6.12/6.18/7.2 四个稳定分支，本周发布多个 RC
- Sasha Levin 持续自动稳定版回 port 工作
- 多个补丁在 5.10/5.15/6.1/6.6/6.12/6.18/7.2 分支上回 port 失败的报告，需要手动调整

### 合并窗口动态（v7.4）

v7.4 合并窗口已开启，本周收到的主要 GIT PULL 请求汇总：

| 子系统 | 维护者 | 说明 |
|--------|--------|------|
| Rust | Miguel Ojeda | Rust 语言支持更新 |
| Rust XArray | Daniel Gomez | XArray Rust 绑定 |
| Scheduler | Ingo Molnar | 调度器更新 |
| Performance events | Ingo Molnar | perf 事件更新 |
| locking | Ingo Molnar | 锁机制更新 |
| objtool | Ingo Molnar | objtool 更新 |
| RCU | Paul E. McKenney | RCU 更新 |
| KVM x86 (VMX+TDX) | Sean Christopherson | KVM VMX/TDX 变更 |
| KVM x86 (SVM) | Sean Christopherson | KVM SVM 变更 |
| KVM x86 (MMU) | Sean Christopherson | KVM MMU 变更 |
| KVM x86 (Misc) | Sean Christopherson | KVM 杂项变更 |
| KVM Selftests | Sean Christopherson | KVM 自测试 |
| arm64 | Catalin Marinas | ARM64 更新 |
| Driver core | Danilo Krummrich | 驱动核心更新 |
| Kbuild/Kconfig | Nathan Chancellor | 构建系统更新 |
| nolibc | Thomas Weißschuh | nolibc 更新 |
| smb client | Paulo Alcantara | SMB 客户端更新 |
| cpufreq/arm | Viresh Kumar | CPU 频率更新 |
| OPP | Viresh Kumar | OPP 框架更新 |
| thermal | Daniel Lezcano | 温度驱动更新 |
| interconnect | Georgi Djakov | 互连框架更新 |
| md | Yu Kuai | MD/RAID 更新 |
| chrome-platform | Tzung-Bi Shih | Chrome 平台更新 |
| hfs/hfsplus | — | HFS 文件系统更新 |
| nilfs2 | — | nilfs2 更新 |
| s390 | — | s390 架构更新 |
| configfs | — | configfs 更新 |
| USB serial | — | USB 串口更新 |
| pwm | — | PWM 更新 |

### v7.3 最终修复

本周同时进行了 v7.3 的最终修复推送：

| 子系统 | 维护者 | 说明 |
|--------|--------|------|
| Networking | Paolo Abeni | v7.3-rc8 网络修复 |
| i2c | Andi Shyti | v7.3-rc8 I2C 修复 |
| SCSI | James Bottomley | v7.3-rc7+ SCSI 修复 |
| SPI | Mark Brown | v7.3-rc7 SPI 修复 |
| regmap | Mark Brown | v7.3-rc7 regmap 修复 |
| sound | Takashi Iwai | v7.3-final 音频修复 |
| firewire | Takashi Sakamoto | v7.3-final firewire 修复 |
| DRM | Dave Airlie | v7.3 final DRM 修复 |
| Crypto | Eric Biggers | v7.3-final 加密修复 |
| KVM | Paolo Bonzini | v7.3-rc8/final KVM 修复 |
| printk | — | v7.3 printk 热修复 |
| tracing | — | v7.3 final tracing 修复 |
| probes | — | v7.3-rc7 probes 修复 |
| pin control | — | v7.3 pinctrl 修复 |
| gpio | — | v7.3 GPIO 修复 |
| power sequencing | Bartosz Golaszewski | v7.3 电源时序修复 |
| turbostat | Len Brown | v7.3-rc7 turbostat 修复 |

### 活跃贡献者 Top 10

| 排名 | 贡献者 | 邮件数 |
|------|--------|--------|
| 1 | Greg Kroah-Hartman | 4,131 |
| 2 | netdev-bot+sashiko | 742 |
| 3 | Sasha Levin | 437 |
| 4 | Konrad Dybcio | 339 |
| 5 | Yehyeong Lee | 279 |
| 6 | David Hildenbrand (Arm) | 271 |
| 7 | Lorenzo Stoakes (ARM) | 243 |
| 8 | Krzysztof Kozlowski | 214 |
| 9 | gregkh (bot) | 211 |
| 10 | Siddh Raman Pant | 204 |

---

## 📈 趋势分析

### 1. 安全问题趋势

本周安全问题集中爆发，14 个唯一 CVE 中：
- **AppArmor cred UAF** 是本周最严重的安全发现，由 Google 的 Jann Horn 报告并提供了可复现的 KASAN 测试用例
- **CVE-2026-89603 (seccomp)** 引发了关于 CVE 评估标准的讨论，Kees Cook 明确请求拒绝该 CVE，反映出 CVE 分配机制的潜在问题
- **SELinux avtab 损坏** 虽然由 fuzzing 发现，但可能指向更深层的安全策略加载问题
- 多个 CVE 集中在 6.6.y 回 port，表明 older stable 分支仍在持续接收安全修复

### 2. 回归问题趋势

v7.3 发布时仍有未解决的回归，Thorsten Leemhuis 的回归状态报告揭示了：
- **mt7925e 休眠恢复挂起** — 修复已存在但未能在 7.3 发布前合入，反映出版本发布与修复时机的协调问题
- **DCN32 显示损坏** — 影响 120Hz FreeSync 显示器，修复需要更深入理解 SubVP 与 FreeSync 的交互
- **xhci 设备脱落** 和 **vlan 网络中断** — 影响范围广，修复正在审查中
- USB 音频回归（Behringer UMC204HD）是典型的"过于宽泛的 quirk 匹配"问题

### 3. 内存安全问题

本周内存安全问题突出：
- blk-crypto 的 NULL deref + UAF 是典型的错误处理路径缺陷
- x86/kaslr vmemmap 重叠是 KASLR 与 zone device 内存管理的交互问题
- ext4 的 slab-out-of-bounds 可能涉及 iov_iter 与 folio API 的边界处理
- syzbot 报告的 205 个问题中，大量为 UAF 和越界访问

### 4. v7.4 合并窗口展望

v7.4 合并窗口开启，主要看点：
- **Rust 支持**持续扩展，新增 XArray 绑定、cpufreq 和 clk 框架绑定
- **KVM x86** 提交大量 TDX 和 SVM 变更，虚拟化安全特性持续增强
- **调度器**和**perf** 同步更新，可能涉及核心架构调整
- **RCU** 新增 CSD-lock 停顿恢复时间报告，改进可观测性
- **arm64** 和 **s390** 架构持续演进

### 5. 稳定版维护

6.6.160-rc1 以 1,674 个补丁创本周之最，反映出 6.6 LTS 分支的巨大维护工作量。6.12.114-rc1 也有 869 个补丁。多个补丁在旧分支（5.10/5.15/6.1）上回 port 失败，表明内核 API 演进对长期支持分支的挑战。

---

## 📋 本周 CVE 清单

| CVE 编号 | 描述 | 状态 |
|----------|------|------|
| CVE-2026-89603 | seccomp bypass after ptrace with TSYNC | ⚠️ 争议（Kees Cook 请求拒绝） |
| CVE-2026-64017 | 块设备安全修复 | 6.6.y 回 port 中 |
| CVE-2026-89566 | jbd2 相关 | 6.6.y 回 port 中 |
| CVE-2026-89567 | jbd2 相关 | 6.6.y 回 port 中 |
| CVE-2026-22996 | 安全修复 | 6.6.y 回 port 中 |
| CVE-2026-23035 | 安全修复 | 6.6.y 回 port 中 |
| CVE-2026-43317 | 安全修复 | 6.6.y 回 port 中 |
| CVE-2026-63974 | 安全修复 | 6.6.y 回 port 中 |
| CVE-2026-93275 | perf/x86/intel/pt | 修复中 |
| CVE-2026-98297 | 安全修复 | 6.6.y 回 port 中 |
| CVE-2026-89603 | seccomp bypass | 见上 |
| CVE-2024-40979 | 旧 CVE 回 port | 稳定分支 |
| CVE-2025-38060 | 旧 CVE 回 port | 稳定分支 |
| CVE-2025-38264 | 旧 CVE 回 port | 稳定分支 |
| CVE-2025-71272 | 旧 CVE 回 port | 稳定分支 |

---

*本报告由 AI 自动生成，基于 lore.kernel.org 公开邮件列表数据。如有遗漏或错误，请提交 Issue 反馈。*
