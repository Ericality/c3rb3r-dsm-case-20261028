# IOC 清单

> 全部地址已打码，复制使用时请自行还原。

## 载荷投递

| 项 | 值 |
|---|---|
| 下载地址 | `hxxp://46[.]8[.]78[.]122/xdy5[.]6xc?x86_64` |
| 落地路径 | `/bin/l0ck.enc` → `/bin/libgld` |
| 文件属性 | `libgld`: mode 0755, uid 0 |
| atime | 2026-10-28 23:11:06 |
| dtime（自删除） | 2026-10-29 02:01:38 |

## 临时持久化文件

```
/etc/cron.d/synotcdu.9014 … 9018
/etc/cron.d/synotwky.9034 … 9038
/etc/cron.d/synotkeo.19947 … 19951
```

命名规则：`synot` + 3 字母 + PID 号。本案为 `cdu` / `wky` / `keo`；
另一受害者案例为 `gkk`。**后缀可能每次不同。**

## 运行特征

```
.L0CK3D          加密后缀
read-me3.txt     勒索信（2259 字节）
libgld           加密器进程名 / 二进制名
init-rsysd.lo    运行痕迹（0 字节，位于 /run/lock/）
0init-ld.lo      同上
```

## 加密参数

```
FOOTER_SIZE = 588
marker      本样本 75 30 b1 ec，另一案例 ce 35 a5 93（每会话可能不同）
mode        0x24 全文件 / 0x25 按百分比 / 0x26 仅前 1MiB
crypto_mode 0x20
正文算法    ChaCha8
主材料      48 字节 = IV(16) + AES-256 key(32)
```

## 检测建议

1. 监控 `/etc/cron.d/` 出现 `synot*` 前缀文件
2. 监控 `/bin` 下出现 `libgld` 或 `l0ck.enc`
3. 监控 `/run/lock/init-rsysd.lo`、`0init-ld.lo`
4. 监控 `user.xdg.origin.url` 指向 `xdy5.*` 的下载
5. 监控 588 字节固定增长的文件改名