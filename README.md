# 第二个独立受害样本：C3RB3R / libgld 在 DSM 上被内核 OOM 杀死

> 脱敏取证记录。不含密钥、内存、样本、业务数据、可识别受害者的路径或网络标识。

## 摘要

2026-10-28 夜间，一台 Synology DS1821+（DSM 7.2.0-64570-1）被 C3RB3R 勒索软件加密。
证据显示与 [soimy/c3rb3r-rescue](https://github.com/soimy/c3rb3r-rescue) 记录的是同一活动。

与常见叙述不同的是：**这次加密没有跑完。加密器因自身内存膨胀被内核 OOM killer 杀死，此后再无新增加密。**

## 环境

- 物理机（非虚拟化），32 GiB RAM
- 4 台 KVM 客户机常驻，`mem-lock=on`，约 10~11 GiB 不可回收
- 数据卷为 btrfs，其中一卷已 100% 占满
- DSM 7.2.0-64570-1（低于 CVE-2024-10441 修复基线 7.2-64570-4）

## 时间线

| 相对时刻 | 事件 |
|---|---|
| T+0 | `SYNO.API.Auth_7_login` PAM 认证失败 ×3 |
| T+1m | 计划任务目录 mtime 变更 |
| T+7m | 首个加密文件落地 |
| T+7m → T+2h50m | 持续加密（先小文件，后 GB 级大文件） |
| T+2h32m | btrfs 写回开始停滞 |
| T+2h34m | 全系统 `blocked for more than 120 seconds` |
| T+2h50m | 最后一个文件加密完成 |
| T+2h51m | 内核 OOM 连续杀死两个 `libgld` 进程 |
| 之后 | 全盘零新增 |

## 为什么它死了

OOM 时刻的内存账本（内核日志原文）：

```text
Normal zone  present 30,133,504 kB   free 62,900 kB
  mlocked        10,596,772 kB   ← 约 10.1 GiB，VM 客户机内存，不可回收
  shmem           9,901,736 kB   ← 约 9.4 GiB
  active_file        29,968 kB   ← 文件缓存只剩 29 MB
  inactive_file      31,912 kB

libgld:5595  total_vm 5,209,026 pages (≈20 GiB)
             rss 1,616,611 (≈6.2 GiB)
             swapents 3,444,696 (≈13.1 GiB)
```

| 项目 | 占用 |
|---|---|
| 4 台 KVM 客户机内存（`oom_score_adj=-850`，受保护） | ≈ 10~11 GiB |
| `libgld` 匿名内存（6.2 GiB 常驻 + 13.1 GiB 换出） | **≈ 19 GiB** |
| 内核 slab / 页表 / 栈 | ≈ 1 GiB |
| 合计 | ≈ 31 GiB / 32 GiB |

**一个进程的胃口，正好等于虚拟机锁走之后剩下的全部内存。**

它并非流式加密——匿名内存随单个文件大小线性增长，最后处理的是 5~7 GB 的视频。

### btrfs 的角色：放大器，不是元凶

卡死时的内核栈：

```text
btrfs_create
  ← btrfs_end_transaction
  ← btrfs_run_delayed_refs
  ← __btrfs_free_extent
  ← add_to_free_space_tree
  ← mutex_lock          （阻塞 120 秒以上）
```

`space_cache=v2` 的空闲空间树更新需要按 block group 加锁。海量元数据变更
（每个目录"加密 + 改名 + 投毒"三个操作）+ 接近满盘，导致延迟引用堆积、事务提交变慢、
脏页写不回去；而此时文件缓存已被榨到只剩 60 MB，内核无可回收，只剩下匿名内存——于是 OOM。

这不是 btrfs 的缺陷，也不是 DSM 的缺陷，而是**病态负载下的性能崩塌**放大了加密器自身的设计问题。

## 文件格式观察

与 c3rb3r-rescue 记录一致，本样本逐字节验证通过：

```text
FOOTER_SIZE = 588
footer[568:572] marker      本样本为 75 30 b1 ec（对方样本为 ce 35 a5 93）
footer[572]     mode        0x24 全文件 / 0x25 按百分比 / 0x26 仅前 1MiB
footer[573]     percent
footer[574]     crypto_mode 恒为 0x20
footer[580:588] 原始大小（u64 LE）
footer[0:48]   AES-CBC(48 字节主材料) 包裹的 [key32][nonce8][nonce8]
```

- 主材料 = 48 字节 = IV(16) + AES 密钥(32)
- 正文算法 ChaCha8（8 轮）

**marker 在两起事件中不同，提示它可能是每会话随机值而非样本固定特征。**
这一点对任何把 marker 写死的工具都有直接影响。

## 全量盘点（4,184 个 `.L0CK3D`，剔除快照副本后 2,392 个真实文件）

| mode | pct | 含义 | 文件数 | 字节 | 实际加密比例 |
|---|---|---|---|---|---|
| `0x25` | 10 | 分段部分加密 | 1,162 | 4,168.30 GB | **12%** |
| `0x24` | 0 | 全文件加密 | 1,017 | 0.27 GB | 100% |
| `0x26` | 0 | 仅前 1 MiB | 203 | 0.49 GB | ≈0 |
| `0x25` | 20 | 分段部分加密 | 8 | 14.16 GB | 21% |
| `0x6d` | — | 改名未加密 | 2 | 0.03 GB | 0% |

**99.6% 的加密字节只被加密了 12%；真正全文件加密的只有 0.27 GB，且全是小文件。**

## 对其他受害者的意义

1. 看到海量 `read-me3.txt` 不等于数据全毁。先看加密器**停在哪里**。
2. 统计 `.L0CK3D` 的 footer，能立刻知道每个文件实际被加密的比例。
3. 若机器上有多台虚拟机锁定内存、且某个 btrfs 卷接近满，这套工具**很可能自我终止**。
4. 但不要因此拖延：它也可能在别的配置下跑完。

## 不发布的内容

密钥材料、RAM 内容、样本二进制、业务数据、任何可定位受害者的路径或网络标识。

## 相关

- 上游取证项目：https://github.com/soimy/c3rb3r-rescue
- 本仓库对应 Issue：[在该项目中](https://github.com/soimy/c3rb3r-rescue/issues)