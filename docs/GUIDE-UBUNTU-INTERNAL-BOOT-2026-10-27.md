# 小米平板 6 Pro 512GB：Ubuntu 单系统内部启动稳定指南

> 面向：要给 **Xiaomi Pad 6 Pro（代号 liuqin，SM8475，512GB）** 装 Ubuntu 单系统、并且希望它
> **每次重启都能稳定从内部存储启动**的人（或 agent）。
> 本文所有关键结论都在一台实机（512GB，HW 16.9.0）上验证过；凭经验推测的地方会明确标注。

---

## 0. 先看结论：最容易踩的坑

装完 Ubuntu 后"第一次能用、一重启就回 Fastboot"，**几乎 100% 是同一个原因**：

> bootloader 在解锁状态下会整分区读取 `boot_a` 做 AVB 校验，而它在这个型号上会
> **读不满请求的字节数**，于是把槽位标成 `unbootable`，两个槽都不可启动 → 进 Fastboot。

**修复只有一条命令**（详细原理见第 6 节）：

```sh
fastboot --disable-verity --disable-verification flash vbmeta_a rom/images/vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_b rom/images/vbmeta.img
```

改的只是 vbmeta 里的 4 字节 flags，**不动 boot 镜像本身**，可完全回滚。

---

## 1. 前提条件

| 项目 | 要求 |
|---|---|
| 机型 | Xiaomi Pad 6 Pro（liuqin）。**512GB / 256GB 都行**，但 512GB 的分区尺寸需要按第 3 节核对 |
| 解锁 | bootloader 已解锁（`fastboot flashing unlock`，且 `critical unlock`） |
| Android 版本 | 与项目 pin 的 ROM 一致（本项目用 `OS2.0.6.0.VMYCNXM`）；不一致会导致 firmware 哈希校验失败 |
| PC | Linux 或 Windows + fastboot；能连 USB 3.x |
| 时间 | 首次安装约 1–2 小时（主要花在刷 super 镜像和下载） |

---

## 2. 准备的东西

1. **项目安装包**：`yzddmr6/xiaomipad-6pro-mainline` 的 release（v0.6.0 或更新），里面有
   `installer.img`、`rootfs.tar.gz`、`boot.img`、`rom/images/*`。
2. **原厂 ROM**（与 pin 版本一致）里的：`boot.img`、`vendor_boot.img`、`dtbo.img`、`vbmeta.img`、
   `super.img`、`NON-HLOS.bin`、`BTFM.bin`、`dspso.bin`。
3. **fastboot**（platform-tools）。Windows 上建议用官方 platform-tools，别用第三方精简包。

---

## 3. 512GB 的分区布局（实测）

`/dev/sda`（UFS，473.8 GiB 可用）关键分区：

| 分区 | 大小 | 用途 |
|---|---|---|
| `super` (sda28) | 8.5 GiB | Android 动态分区容器 |
| `userdata` (sda35) | 128 GiB | Android 用户数据 |
| `linux_root` (sda36) | 64 GiB | **Ubuntu 根分区**（EXT4，LABEL=`LIUQIN_ROOT`） |
| `linux_home` (sda37) | 271.3 GiB | **Ubuntu home**（LABEL=`LIUQIN_HOME`） |

`/dev/sde` 上是启动相关分区，**不要动**：

| 分区 | 大小 | 说明 |
|---|---|---|
| `boot_a` (sde14) | 192 MiB | Ubuntu 启动镜像（header v2，DTB 内嵌） |
| `vbmeta_a` (sde17) | 8 KiB | AVB 元数据（要打 disable 标志） |
| `dtbo_a` (sde18) | 24 MiB | 设备树叠加层 |
| `vendor_boot_a` (sde25) | 96 MiB | Android 用；Ubuntu 启动时也会被 bootloader 加载 |

> **单系统可以把 Android 的 `super` + `userdata` 删掉腾出 136 GiB**（见第 9 节），但
> **删了就不能再跑 Android**，要恢复得重刷整包。

---

## 4. 安装流程（官方 installer）

1. `fastboot boot installer.img`（RAM 启动安装器）；
2. 通过安装器的 USB 网络（`<usbnet-ip>`）或 telnet 进去，按 `INSTALL-TESTING.md` 的说明选
   **dual** 或 **linux-only** 布局；
   - **单系统**：选 linux-only（不保留 Android 分区）；
   - **双系统**：选 dual（保留 Android，Ubuntu 用 64G + 271G）；
3. 安装器会：分区 → 刷 `super`/`vendor_boot`/`dtbo`/`vbmeta`/firmware → 解 rootfs → 装 boot 镜像；
4. 完成后 `fastboot reboot`。

**512GB 专属注意**：安装器按 256GB 的分区尺寸表工作，512GB 上 `linux_home` 会更大、
`userdata` 是 128 GiB。装完请核对：

```sh
fastboot getvar current-slot          # 应为 a
lsblk -o NAME,SIZE,LABEL | grep -E 'linux_root|linux_home'
```

---

## 5. 装完第一次启动：先别高兴太早

第一次 `fastboot reboot` 大概率会：

- 屏幕亮 → 小米 logo → **黑屏 → 又回 logo**，或者直接进 Fastboot。

先别急着重装。按第 6 节处理，**90% 是这个原因**。

---

## 6. 核心问题：AVB 整分区读取导致槽位被判死

### 现象

- 重启后进 Fastboot，`fastboot getvar current-slot` 可能是 `a`，但 `fastboot getvar slot-unbootable:a` = yes；
- 平板 `logfs` 分区（`/dev/sda13`，FAT）里的 `UefiLog0..4.txt` 有原始记录：

```
Active Slot _a is bootable, retry count 6
Booting from slot (_a)
Load Image boot_a total time: 102 ms
avb_slot_verify.c:210: ERROR: boot_a: Read fewer than requested bytes.
No bootable slots found enter fastboot mode
VB2: boot state: red(3)
```

对比能启动的原厂镜像：

```
Load Image boot_a total time: 26 ms
VB2: Authenticate complete! boot state is: orange
```

### 原因

设备是解锁态（`verifiedbootstate=orange`），libavb 走 `allow_verification_error` 路径时
用 `get_size_of_partition()` **整分区读取**再算哈希；这台机器的 ABL 在这条路径上返回的字节数
不足 → 校验失败 → 槽位 `unbootable`。原厂镜像带 AVB footer、走另一条读取路径，所以没事。

### 修复

```sh
# 只改 vbmeta 的 flags：HASHTREE_DISABLED | VERIFICATION_DISABLED = 0x3
fastboot --disable-verity --disable-verification flash vbmeta_a rom/images/vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_b rom/images/vbmeta.img
fastboot --set-active=a
fastboot reboot
```

**不需要**改 DTB、不需要给 boot 镜像加 AVB footer、不需要换 kernel —— 这些都是走过的弯路。

### 验证

```sh
# 板上执行
python3 - <<'PY'
import struct
d = open('/dev/sde17','rb').read(4096)
print(hex(struct.unpack_from('>I', d, 120)[0]))   # 期望 0x3
PY
```

刷完第一次启动后，initramfs 会把槽位标成 `successful=1 unbootable=0`，此后**每次重启都能直接进 Ubuntu**。

---

## 7. initramfs 根契约：这 41 个文件不能动

Ubuntu 的 initramfs 内置了一份**哈希契约**（在 initramfs 镜像里，不在磁盘上），启动时会校验
41 个文件；任何一个不匹配就**拒绝持久根**，落到应急 shell：

```
liuqin-initramfs: native root rejected: not a regular file: /etc/systemd/system/liuqin-hexagonrpcd-sdsp.service
liuqin-initramfs: persistent root marker or root contract is invalid
liuqin-initramfs: starting local emergency shell
```

契约包含（部分）：

- `/etc/systemd/system/liuqin-*.service`、`.target`（slpi、hexagonrpcd、ssc-sample-gate、sensor-stack…）
- `/usr/libexec/iio-sensor-proxy`、`/usr/libexec/liuqin-ssc-sample-gate`
- `/usr/local/sbin/liuqin-slpi`、`/usr/local/bin/liuqin-shell` 等
- 触控固件 `novatek_nt36532_m81_fw_{csot,tm}.bin`、音频 topology、`hexagonrpcd` 等

**规则**：不要移动、不要改这些文件。要禁用某个服务，用 **systemd drop-in**
（`/etc/systemd/system/<unit>.d/*.conf`）或加 `ConditionPathExists`，不要动原文件。

### 违反了怎么救

initramfs 的应急 shell 会开在 **USB 网卡 `<usbnet-ip>:2323`**（telnet/nc）。从里面：

```sh
blockdev --setrw /dev/sda
mount /dev/sda36 /mnt/r
# 根文件系统在 /mnt/r/native-root
mv /mnt/r/native-root/root/你的备份/* /mnt/r/native-root/etc/systemd/system/
sync
reboot -f
```

---

## 8. 装完必须做的收尾（否则体验很糟）

### 8.1 用户组（否则没有声音、没有摄像头）

```sh
usermod -aG audio,video,render,input <你的用户名>
```

不做的后果：`/dev/snd/*` 打不开 → PipeWire 只有 "Dummy Output"；`/dev/video*` 没权限。

### 8.2 指纹（FPC1264）

项目 v0.6.0 的指纹模块会**卡在依赖上**（它要求 `iio-sensor-proxy` active，而后者依赖 SLPI）。
本机的修法（不改任何契约文件）：

- 给 `liuqin-ssc-sample-gate.service` 加 drop-in：
  `ConditionPathExists=!/etc/liuqin-slpi-disabled`
- 给 `/usr/local/lib/liuqin-fpc-oem/stable_module.py` 加判断：SLPI 标记存在时不要求 `iio-sensor-proxy`

然后 `systemctl restart liuqin-fpc-stable-module`，`/dev/fpc1020` 出现，GDM 里就能录指纹。

### 8.3 console-setup 失败

`/etc/default/keyboard` 里 `XKBLAYOUT="default,us"` 在现代 xkb-data 上无效，改成 `us`。

### 8.4 swap

内核没有 zram 模块，只能 swapfile：

```sh
fallocate -l 16G /home/swapfile && chmod 600 /home/swapfile
mkswap /home/swapfile && swapon /home/swapfile
echo '/home/swapfile none swap sw,nofail,pri=10 0 0' >> /etc/fstab
```

### 8.5 关闭自动更新对启动的干扰

`unattended-upgrades` 会在关机时持"block inhibitor"，导致 `systemctl reboot` 被拒：

```sh
systemctl mask apt-daily.timer apt-daily-upgrade.timer
```

---

## 9. 单系统：把 Android 的空间收回来（可选）

确认不再需要 Android 后：

```sh
sgdisk --backup=/root/gpt-backup.bin /dev/sda        # 先备份 GPT
sgdisk --delete=28 --delete=29 --delete=30 --delete=31 \
       --delete=32 --delete=33 --delete=34 --delete=35 /dev/sda
sgdisk --new=28:480768:36313087 --typecode=28:8300 --change-name=28:linux_data /dev/sda
partprobe /dev/sda
mkfs.ext4 -L LIUQIN_DATA /dev/sda28
mkdir -p /data && mount LABEL=LIUQIN_DATA /data
echo 'LABEL=LIUQIN_DATA /data ext4 defaults,nofail,x-systemd.device-timeout=30s 0 2' >> /etc/fstab
```

结果：`/` 63G、`/home` 266G、**`/data` 134G**。

> **警告**：删掉 `super` 和 `userdata` 后 Android 无法启动，恢复需要重刷整包 ROM。

---

## 10. 备份与回滚（强烈建议装完立刻做）

| 要备份的东西 | 命令 / 位置 |
|---|---|
| boot 镜像（Ubuntu 官方） | `dd if=/dev/sde14 of=/root/boot-ubuntu.img bs=1M`（192MB） |
| vbmeta 原厂 | `dd if=/dev/sde17 of=/root/vbmeta-stock.img`（8KB） |
| GPT | `sgdisk --backup=/root/gpt.bin /dev/sda`、`/dev/sde` |
| `misc` 前 4K | `dd if=/dev/sda11 of=/root/misc-first4k.bin bs=4k count=1` |

回滚到"能启动"的最小操作：

```sh
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta-stock.img
fastboot flash boot_a boot-ubuntu.img
fastboot --set-active=a
fastboot reboot
```

---

## 11. 故障排查表

| 症状 | 原因 | 处理 |
|---|---|---|
| 重启回 Fastboot，`avb_slot_verify.c:210` | 第 6 节的 AVB 短读 | flash vbmeta（disable 标志） |
| 两个槽都 `unbootable`，连 `fastboot boot` 都报 `Load Error` | 槽被标死 | `fastboot --set-active=a` 会重置该槽状态 |
| 卡在小米 logo 循环、USB 不出现 | 刷了不兼容的镜像（如 Android 的 header v4 镜像） | 长按电源 12 秒 → 电源+音量减进 Fastboot → 刷回 Ubuntu 镜像 |
| 落到 initramfs 应急 shell，`native root rejected` | 动了根契约文件 | 第 7 节从应急 shell 恢复 |
| Ubuntu 起来但没有声音 | 用户不在 audio 组 | 第 8.1 节 |
| 指纹模块 failed | 依赖链被 SLPI 拖死 | 第 8.2 节 |
| `systemctl reboot` 被拒（block inhibitor） | unattended-upgrades | 第 8.5 节 |
| `/` 或 `/home` 挂不上、fsck 失败 | `e2fsck` 修（`/home` 曾出现目录 inode 损坏） | `e2fsck -y /dev/sda37` |

---

## 12. 双系统 vs 单系统（本机的真实教训）

- **双系统的硬约束**：Ubuntu 需要 **disable 版 vbmeta**（第 6 节），而 Android 能否在 disable 版下启动
  **未验证**（本机三次尝试都失败：原厂 boot、KernelSU boot、干净 userdata 均在 USB 初始化前挂死）。
  如果 Android 需要原厂 vbmeta，那两者冲突，切换就必须**同时换 vbmeta**，而 vbmeta 只能经 fastboot 写。
- **本机最终选择**：Ubuntu 单系统 + `/data` 回收 Android 空间。
- 如果一定要双系统：先用 `fastboot boot` 验证 Android 镜像能起（注意 header v4 镜像 RAM 启动
  需要 `vendor_boot`，多数情况下**不可行**），再决定。

---

## 13. 一页速查（TL;DR）

```sh
# 1) 装（官方 installer，RAM 启动）
fastboot boot installer.img      # 按 INSTALL-TESTING 操作

# 2) 关键修复：关 AVB 校验（否则重启回 fastboot）
fastboot --disable-verity --disable-verification flash vbmeta_a rom/images/vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_b rom/images/vbmeta.img
fastboot --set-active=a && fastboot reboot

# 3) 起来后收尾
usermod -aG audio,video,render,input $USER
# 指纹/console/swap 见第 8 节

# 4) 立刻备份
dd if=/dev/sde14 of=/root/boot-ubuntu.img bs=1M
dd if=/dev/sde17 of=/root/vbmeta-stock.img
sgdisk --backup=/root/gpt.bin /dev/sda
```

**记住三条**：

1. **vbmeta 必须 disable 校验**，否则重启必回 fastboot；
2. **别动 initramfs 根契约里的 41 个文件**，要禁用服务用 drop-in；
3. **装完立刻备份 boot / vbmeta / GPT**，回滚只需要这三样。
