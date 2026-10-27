# Xiaomi Pad 6 Pro（liuqin）启动 SLPI 触发整机硬复位 —— 可行解法调研

- 调研日期：2026-10-27
- 设备：Xiaomi Pad 6 Pro（liuqin，SM8475），Ubuntu 26.04，内核 `6.17.0-rc1-gfe81794b5e1b`
  （项目仓库 `yzddmr6/linux-sm8450-liuqin`，分支 `liuqin-6.17` / 发布分支 `release/v0.6.0-dp-fingerprint`，commit `fe81794b5e1b`）
- 结论口径：**「已确认」= 本次调研有一手命令输出/源码/固件字符串直接支撑；「推断」= 由多条间接证据推出，尚无决定性证据；「未确认」= 只能提出假设与验证方法。**
- 本文不包含 token、设备序列号、MAC、UUID、校准数值。

---

## 0. 结论摘要（先看这里）

| 问题 | 判断 | 依据强度 |
|---|---|---|
| 复位是什么性质？ | **SoC 侧硬复位**，不是 PMIC 自身故障、不是 Linux panic、不是 AP 侧看门狗；内核完全来不及记录 | 已确认（`pdreason`=PS_HOLD；无任何内核消息；pstore 空；`panic=0`） |
| 是不是「SLPI 固件 assert → SCM/PIL 致命错误 → PMIC 复位」？ | **其中「PMIC 复位」一半不成立**（PMIC 没记录自身故障）；**「固件致命错误 → 整机复位」大概率成立**：SLPI 固件里有一条**故意触发看门狗咬（DOG bite）**的错误处理路径 | 推断（强）：固件字符串 + 时序 + AP 无感知 |
| 是不是 secure world 认证失败？ | **不是**：PAS 认证与固件加载都成功（`remote processor slpi is now up`），fastrpc 全部 probe 成功 | 已确认 |
| 是不是电源/时钟没配（LDO/时钟）？ | **不是**：同 DT 的 ADSP/CDSP 正常；SLPI 也能起来并保持数秒；DT 里 `lcx/lmx` 电源域、`xo` 时钟都在 | 已确认 |
| 有没有人已经修好？ | **没有公开的修法**。项目维护者在自己 16.9.0 机器上 SLPI 连跑 9 天不复位；另一台同 HW 用户也正常；issue #10 已关闭（不可复现），#18 是第二台复现机 | 已确认（issue 文本） |
| 能修吗？ | **「有条件可修」但概率不高**：可修的只有「让 SSC 启动期不触发致命错误」这一条路（喂给它完全匹配的 registry/config/file-service）。若致命原因是硬件差异/安全世界策略，则远端不可修 | 推断 |
| 能绕开 SLPI 拿传感器吗？ | **不能**。加速度计/陀螺仪/光感全部挂在 SSC（SLPI）上；ADSP 上没有 sns 服务；AP 侧 I2C/SPI 没有任何 sensor 节点 | 已确认（DT/固件/仓库） |
| 有没有非 SLPI 的替代输入？ | 只有 **hall（磁吸键盘/保护盖）**：`SW_LID` + `SW_TABLET_MODE`，走 TLMM GPIO，可在无 SLPI 时工作；但它给不了屏幕旋转 | 已确认（`liuqin.dts`） |

**一句话**：这不是「内核驱动写错了」，而是「SLPI 里的 SSC 固件在启动阶段走到致命错误，然后按设计咬整机看门狗」。在本机能做的只有两件事：**(a) 用完全匹配的 registry/config/file-service 让它不报致命错**；**(b) 接受没有传感器**。远端无法调试固件内部，也没有第二条传感器通路。

---

## 1. 本次新取得的一手证据

> 采集方式：只读 shell（`_vm/tabrun.py` 走 VM 的 nc 桥），未启动 SLPI、未改任何文件。
> 原始输出落在 `<workdir>\_vm\last-tab-out.txt`（每次运行覆盖，本文引用的是 2026-10-27 16:0x–16:3x 的几次采集）。

### 1.1 复位的「签名」：PMIC 记录的是 PS_HOLD，而不是 PMIC 自身故障（已确认）

设备启动参数（`/proc/cmdline`，过滤后）：

```text
hung_task_panic=1
softlockup_panic=1
panic=0
qcom_q6v5_pas.slpi_auto_boot=0
androidboot.verifiedbootstate=orange
bootinfo.pureason=0x80001
bootinfo.pdreason=0x2
androidboot.hwversion=16.9.0
androidboot.slot_suffix=_a
```

逐 boot 从持久化 journal 的 `Kernel command line` 里取回的历史值：

| boot | pureason | pdreason | 该 boot 的 journal 结束方式 |
|---|---|---|---|
| -4 | 0x80001 | 0x2 | 有序（`systemd-reboot.service` → `Journal stopped`，11:27:59） |
| -3 | 0x800001 | 0x2 | **abrupt**：最后一行 11:41:40，无 shutdown/reboot 记录（`shutdown/reboot` 标记计数 = 0） |
| -2 | 0x80001 | 0x2 | **abrupt**：最后一行 11:45:54（`journald: Received client request to sync journal (with offlining)`），计数 = 0 |
| -1 | 0x80001 | 0x2 | 有序（11:50:58 `Journal stopped`） |
| 0（当前） | 0x80001 | 0x2 | 运行中（uptime ≈ 16 000 s） |

解码依据：小米 downstream `techpack/bootinfo/bootinfo.c` + `bootinfo.h`（例：
`MiYume0721/android_kernel_xiaomi_sm8350_miyume` 的 `techpack/bootinfo/`）。

- `powerup_reason_t`：bit0=`HWRST`、bit1=`SMPL`、bit2=`RTC`、bit3=`DC_CHG`、bit4=`USB_CHG`、
  bit5=`PON1`、bit6=`CABLE`、bit7=`KPD`、bit8=`WARMRST`、bit9=`LPK`；
  高 16 位是 restart event：`0x10000`=WDOG、`0x20000`=KPANIC、`0x40000`=NORMAL、`0x80000`=OTHER。
- `powerdown_reason_t`：bit0=`SOFT`、**bit1=`PS_HOLD`**、bit2=`PMIC_WD`、bit3=`GP1`、bit4=`GP2`、
  bit5=`KPD_AND_RESIN`、bit6=`RESIN`、bit7=`KPDPWR`。

**能得出的结论**（保守版）：

1. `pdreason = 0x2 = PS_HOLD`：PMIC 认为「SoC 松开了 PS_HOLD」，即**复位源在 SoC 一侧**；
   PMIC 没有记录自身看门狗（会是 `PMIC_WD`）或外部触发（`GP1/GP2`/`RESIN`）。→ **排除 PMIC 自身故障复位**。
2. `pureason` 的 bit0（HWRST）在任何一次重启（含 `systemctl reboot`）后都会置位，高位的
   `0x80000`/`0x800000` 在**正常重启**后也分别出现过（boot -3 与 boot 0 就是反例），
   **所以 pureason 不能用来区分「SLPI 复位」与「正常重启」**。这一点必须说清楚，避免过度解读。
3. 结合「无 panic（`panic=0`）」「pstore 空」「journal 突然截断」「内核一行相关日志都没有」，
   可以确认：**复位发生在 Linux 无法感知的层级（TZ / AOP / SSC 子系统硬件）**，而不是内核崩溃后重启。

### 1.2 复位时序：有没有用户态，死亡时间完全不同（已确认）

从持久化 journal 按时间窗取回（`journalctl --since/--until`，跨 boot）：

实验 A（全栈：`liuqin-slpi start` + hexagonrpcd 正常拉起），2026-10-27 11:18：

```text
11:18:38.876  kernel: remoteproc remoteproc0: powering up slpi
11:18:38.879  kernel: remoteproc remoteproc0: Booting fw image qcom/sm8475/liuqin/slpi.mbn, size 5244
11:18:39.073  kernel: remoteproc remoteproc0: remote processor slpi is now up
11:18:39.088  kernel: probe of ...fastrpcglink-apps-dsp... returned 0
11:18:39.093  systemd: Started liuqin-hexagonrpcd-sdsp.service
11:18:39.1xx  hexagonrpcd: Could not open /sys/devices/soc0/hw_platform: No such file or directory
              （platform_subtype / platform_subtype_id / platform_version / soc_id / revision 同样反复失败）
... 约 40–60 秒后整机复位（11:20:06 已经在下一个 boot 里跑 fwupd）
```

实验 B（隔离：sensor 相关 unit 全部移走，只 `liuqin-slpi start`），2026-10-27 11:41：

```text
11:41:37.387  kernel: remoteproc remoteproc0: powering up slpi
11:41:37.390  kernel: Booting fw image qcom/sm8475/liuqin/slpi.mbn, size 5244
11:41:37.587  kernel: remote processor slpi is now up
11:41:37.6xx  fastrpc compute-cb@1..3 probe 成功
<最后一条 journal：11:41:40.4 —— 之后直接复位，约 3 秒>
```

**结论**：SLPI 能起来、能活；**没有 hexagonrpcd 时 3 秒就死，有 hexagonrpcd 时能活约 1 分钟**。
说明 SSC 固件在启动期**确实依赖宿主提供的文件服务**，而且两种情况下最终都走到了致命路径。

### 1.3 复位前内核没有任何消息（已确认）

- 11:41 实验里，`remoteproc0: remote processor slpi is now up` 之后**没有任何** `fatal error received` /
  `watchdog received` / `crash detected in slpi` / `q6v5` 相关打印；
- `/sys/fs/pstore/` 为空；
- `remoteproc0` 的 `recovery` 当前是 `enabled`（不是 disabled），也就是说内核本来准备走「崩溃→恢复」流程，
  但根本没有机会执行（驱动连 fatal 中断都没收到，或者收到前整机已经没了）。

对照：同项目 Pad 8 Pro（piano）的 ADSP 传感器看门狗案例里，AP 侧是能看到
`qcom_q6v5_pas ... fatal error received: ... sns_watchdog.c:84 ...` 和
`remoteproc remoteproc0: crash detected in adsp: type fatal error` 的
（<https://github.com/yzddmr6/xiaomipad-8pro-mainline/issues/3>）。
本机**一条都没有**，说明本机不是「AP 看到 fatal 后处理失败」，而是**AP 压根没被通知**。

### 1.4 SLPI 固件里有一条「故意咬看门狗」的错误路径（已确认字符串存在）

对仓库里 24 个 `slpi.*` 固件做字符串提取（`device/firmware/usr/lib/firmware/qcom/sm8475/liuqin/slpi.b*`）：

```text
slpi.b10  err_inject_crash_root.c:SLPI:Entering STM and injecting DOG Bite.
slpi.b10  err_inject_crash_root.c:SLPI:Injecting DOG multithreaded Bite
slpi.b10  err_post_stm_force_wdog_bite.c:SLPI:Registration of err_post_stm_force_wdog_bite_cb failed, rv = %d
slpi.b08  err_wdog_fbite
slpi.b08  %s:%d:Entering STM and injecting DOG Bite.
slpi.b08  %s:%d:Injecting DOG multithreaded Bite
slpi.b08  SFR Init: wdog or kernel error suspected.
slpi.b08  %s:%d:Could not copy fatal err data to coredump in user pd
slpi.b08  %s:%d:Could not copy err_fatal_params from user pd
slpi.b08  SLPI:SSCTL SUBSYS_EVENT: %s
```

含义（推断，但非常直接）：

- SSC 固件在检测到 **fatal error** 时，会**主动往 STM 注入一次「DOG bite」（看门狗咬）**，
  即它自己把系统看门狗喂爆，让硬件复位整颗 SoC——这正是「无任何 AP 日志、pstore 空、PS_HOLD 掉电」的形状。
- 这套机制在 Android 上由 `subsystem_restart` + `SSCTL/SSR` 承接（downstream `msm_subsystem_restart.c` 里
  `restart_level` 可以是 `SYSTEM`/`RELATED`/`INDEPENDENT`）；**mainline 没有这套东西**，
  所以一旦固件选择 bite，AP 侧没有任何补救机会。

### 1.5 SSC 固件在启动期要读哪些宿主文件（已确认，且与 1.2 的报错对得上）

设备上 ROM 原始的 `sns_reg.conf`（由 `sns_reg_config` 安装而来）：

```text
version=12
file=hw_platform=/sys/devices/soc0/hw_platform
file=platform_subtype=/sys/devices/soc0/platform_subtype
file=platform_subtype_id=/sys/devices/soc0/platform_subtype_id
file=platform_version=/sys/devices/soc0/platform_version
file=soc_id=/sys/devices/soc0/soc_id
file=revision=/sys/devices/soc0/revision
file=output=/mnt/vendor/persist/sensors/registry/registry
property=persist.vendor.sensors.enable.property=/mnt/vendor/persist/sensors/registry/file1
property=persist.vendor.sensors.enable.property1=/mnt/vendor/persist/sensors/registry/file2
```

- 这 6 个 `file=` 就是 1.2 里 hexagonrpcd 反复打不开的那 6 个路径；
- 本机 `/var/lib/liuqin-sensors/registry/` 有 **207 个文件**（`find ... -type f | wc -l`），
  但**没有任何文件引用 `persist.vendor.sensors.*` 之外的属性**（`grep -rl property` 空），
  所以 project 的 `sns_registry` 窄垫片（只放行 `persist.vendor.sensors.enable.property[1]`）**理论上够用**；
- 实验 A 里 journal 出现过 `sns_registry rejected a non-whitelisted property`（项目记录
  `repo/docs/PROGRESS-2026-10-27.md` 11:18–11:25 段），说明 **SSC 实际请求了垫片白名单之外的属性名**。
  这是个可以直接验证/修复的点（见 §5 E4）。

补充：`sns_reg.conf` 的 output 路径是 Android 的 `/mnt/vendor/persist/sensors/registry/registry`，
而本项目把 registry 放在 `/var/lib/liuqin-sensors/registry`，靠 hexagonrpcd 的 hexagonfs 映射过去。
**这条映射是否 100% 等价（尤其写路径 `temp.json`/`fstempfile`/`registry`）必须验证**——Pad 8 Pro 的案例
（`persist` 挂载点不对 → DSP 看门狗每 40 秒咬一次）就是同类问题
（<https://github.com/yzddmr6/xiaomipad-8pro-mainline/issues/3>、<https://github.com/FisiFla/nabu-cachyos/issues/35>）。

### 1.6 设备当前状态（只读采集，已确认）

```text
/sys/class/remoteproc/remoteproc0 name=slpi fw=qcom/sm8475/liuqin/slpi.mbn state=offline recovery=enabled
/sys/class/remoteproc/remoteproc1 name=adsp fw=qcom/sm8475/liuqin/adsp.mbn state=running  recovery=enabled
/sys/class/remoteproc/remoteproc2 name=cdsp fw=qcom/sm8475/liuqin/cdsp.mbn state=running  recovery=enabled
/etc/liuqin-slpi-disabled 存在（0 字节，2026-10-26 17:40）
```

- ADSP/CDSP 用**同一个** `qcom_q6v5_pas` 驱动、同一份 DT 描述方式，都能稳定 running；
  → 可以**排除**「q6v5_pas 驱动整体不工作」「SM8475 的 PAS/SCM 通路坏了」这类解释。
- `soc0` 只有 mainline 的属性：`family / machine / power / revision / serial_number`，
  没有 downstream 的 `hw_platform / platform_subtype* / platform_version / soc_id`。

---

## 2. 机制判定

### 2.1 排除项（已确认）

| 假设 | 结论 | 证据 |
|---|---|---|
| PAS/SCM 认证失败 | 排除 | `Booting fw image ... slpi.mbn, size 5244` 后 `remote processor slpi is now up`；若认证失败会打印 `failed to authenticate` 并 abort |
| 固件缺失/损坏 | 排除 | 项目记录：24 个 `slpi.*` 与仓库 sha256 全同；`firmware.manifest` 里有全部哈希；`dmesg` 能加载 5244 字节的 `slpi.mbn` 并成功 bring-up |
| 内核 panic | 排除 | cmdline `panic=0`；bootinfo 无 KPANIC 位；pstore 空；journal 无 oops/panic |
| AP 看门狗 | 排除（基本） | 无 `watchdog received`、无 `crash detected`；bootinfo 无 WDOG 位（高位不可靠，但结合前两条足够） |
| PMIC 自身故障/PMIC 看门狗 | 排除 | `pdreason=0x2`（PS_HOLD），不是 `PMIC_WD`/`GP1`/`GP2`/`RESIN` |
| 电源域/时钟没开 | 排除 | DT 里 SLPI 有 `power-domains = <&rpmhpd LCX>, <&rpmhpd LMX>`、`clocks = <&rpmhcc RPMH_CXO_CLK>`；`clk_ignore_unused`/`pd_ignore_unused` 在 cmdline；SLPI 能 up 并活 3–60 s；ADSP/CDSP 正常 |
| 用户态 registry 链路 | 排除为「唯一原因」 | 实验 B 把 hexagonrpcd/ssccli/sample-gate/proxy 全部移走，仍然 3 秒复位 |

### 2.2 最可能机制（推断，证据强）：SSC 启动期 fatal → 固件主动 DOG bite → SoC 硬复位

证据链：

1. SLPI 固件里存在**显式**的「注入 DOG bite」错误路径（§1.4）；
2. 复位是 SoC 侧硬复位、AP 完全无感知（§1.1、§1.3）；
3. 死亡时间与「宿主文件服务是否存在」强相关（3 s vs ~60 s，§1.2），说明死因在 SSC 的**启动初始化**里；
4. 复位前没有任何 `q6v5` fatal 中断记录——如果只是「SLPI 崩了但没咬狗」，AP 必然会看到
   `fatal error received`（piano/nabu 的案例都是这样），本机没有，说明固件根本没给 AP 机会。

**这意味着**：想「修」，本质上是要**让 SSC 的启动初始化不产生 fatal error**，而不是修内核。
一旦 fatal 发生，复位是固件设计行为，Linux 侧无法拦截（除非能改固件或关掉 SSC 的看门狗，两者都不现实）。

### 2.3 不能排除的次可能机制（推断，证据中/弱）

| 机制 | 支持 | 反对/未知 | 如何区分 |
|---|---|---|---|
| TZ/XPU 违规触发安全世界复位 | SLPI 固件字符串里有 `AOPSS_MPU_XPU4`、`AOSS_PERIPH_MPU_XPU4`、`PMIC_ARB_XPU4`；TZ 对 XPU 违规的默认动作就是整机复位；AP 同样无日志 | 同一 DT/固件在别的机器上不复位，除非那些机器的 XBL/TZ 版本不同 | 比对故障机与正常机的 `androidboot.bootloader`/XBL 版本；或读 TZ 的 reset reason（需要 TZ 调试口，基本不可行） |
| AOP 判定 SSC 无响应后复位整机 | AOP 管 SLPI 的低功耗岛进出；固件里有 `/pmic/client/sensor_dsp_proxy`、`/vcs/vdd_ssc_int` 等 AOP/PMIC 代理 | 无直接证据 | 同上；或对比 `slpi_auto_boot=1`（更早启动）是否改变死亡时间 |
| SSC 访问了未授权/未映射的保留内存 | `slpi_mem` 是 `no-map` 25 MiB（0x88000000+0x1900000），与 upstream 一致 | 若映射错，加载阶段就会失败 | 对比 `/sys/firmware/devicetree/base/reserved-memory/memory@88000000/reg` 与 upstream/vendor 值 |

### 2.4 为什么同 HW 16.9.0 的其它机器不复位（未确认，三种解释）

维护者在 issue #10 明确说：**他自己的 16.9.0 机器上 SLPI 已连续运行 9 天**
（`liuqin-slpi.service` + `slpi_auto_boot=0`）；另一位同 HW 用户 `yuzelingsha` 也说 SLPI/SSC 正常。
所以「HW 16.9.0」不是充分条件（维护者结论一致）。

可能性排序：

1. **per-unit 的 SSC 数据不匹配**（最可操作）：`persist` 里的 sensor registry / `sns_reg_config` /
   hexagonfs 映射，只要和这台机器上跑的 SLPI 固件版本对不上，SSC 就会在启动期报错。
   本机 registry 有 207 个文件、含 `akm_fusion*` 等磁力计融合项；不同批次/不同 ROM 的 `persist`
   内容不同 → 解释「同 HW 不同机」。
2. **bootloader/TZ 版本不同**（最不可控）：`androidboot.hwversion` 相同不代表 XBL/TZ 相同；
   不同 OTA 版本可能带不同的 XPU/PIL 策略。
3. **传感器实际populated 不同**（不可控）：SSC 启动时会探测它自己的 I2C/SPI 上的 sensor；
   少一颗/换型号都可能让它报 fatal。厂商 DT（`_haptics/vendor-qcom-liuqin-sm8475.dtsi`）里
   `board-sensor = "VIRTUAL-SENSOR"`，AP 侧 DTB 里没有任何 sensor 节点，说明 sensor 完全由 SSC 侧描述，
   AP 无法确认实际型号。

> 结论：**不能再用「HW 16.9.0 批次问题」当结论**；必须按「per-unit 数据/固件/硬件差异」去查。

---

## 3. 有没有人已经修好

### 3.1 上游 mainline 的 SM8450 SLPI 状态（已确认：支持是「有」的）

- `arch/arm64/boot/dts/qcom/sm8450.dtsi` 里有完整 SLPI 节点：
  `compatible = "qcom,sm8450-slpi-pas"`、`memory-region = <&slpi_mem>`（0x88000000 + 0x1900000）、
  `power-domains = <&rpmhpd LCX>, <&rpmhpd LMX>`、`qcom,smem-states = <&smp2p_slpi_out 0>`（"stop"）、
  glink-edge（`qcom,remote-pid = <3>`）+ `fastrpc { label = "sdsp"; qcom,non-secure-domain; }`
  （<https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/qcom/sm8450.dtsi>）。
- 上游**明确把 SLPI 打开**的 SM8450 机器：`sm8450-hdk.dts`、`sm8450-qrd.dts`、
  `sm8450-sony-xperia-nagara.dtsi`（Xperia 1 IV / 5 IV），都是
  `&remoteproc_slpi { status = "okay"; firmware-name = "qcom/sm8450/.../slpi.mbn"; }`。
  → 说明「SM8450 + mainline + SLPI」本身是被验证过的组合，不是无人区。
- 相关上游演进（说明这块一直在被修，但都不是「整机复位」）：
  - `91d70eb708` arm64: dts: qcom: sm8450: add fastrpc nodes（2022）
  - `5dd227ccfb` arm64: dts: qcom: sdm845: Use the Low Power Island CX/MX for SLPI（2023-12）
  - `2cb0c97ce4` dt-bindings: remoteproc: qcom,adsp: Allow cx-supply on qcom,sdm845-slpi-pas（2025-12）
  - `ba3125b470` remoteproc: qcom: pas: add SDM845 SLPI compatible；`637e98bfe2` refactor SLPI remoteproc init
- 结论：**上游没有任何「SLPI 启动导致整机复位」的已知 bug/fix**；相反，SLPI 在 sdm845/sc8280xp/sm8250
  上被大量设备启用（传感器在那些平台上可用）。

### 3.2 项目 issue 状态（已确认）

| issue | 内容 | 状态 |
|---|---|---|
| [#10](https://github.com/yzddmr6/xiaomipad-6pro-mainline/issues/10) | 本现象 + CS35L41 `-13` | **closed**。维护者：自己 16.9.0 机器 SLPI 跑 9 天不复现；给了取证清单（bootinfo、串流 kmsg、`sha256sum slpi*`、reserved-memory）；音频问题已修（commit 92f1110），SLPI 复位「not reproducible here」 |
| [#18](https://github.com/yzddmr6/xiaomipad-6pro-mainline/issues/18) | 512GB 机器同样「starting SLPI resets the device」，只能改 rootfs 加 marker | open。**第二台复现机**，说明不是 256GB 独有 |
| 内核仓库 PR/issue | 只有 camera/touch/DP/fingerprint，**没有 SLPI 相关 PR** | 无 |

项目自己的规避就是 `/etc/liuqin-slpi-disabled` + unit 的
`ConditionPathExists=!/etc/liuqin-slpi-disabled`
（`device/sensors-overlay/etc/systemd/system/liuqin-slpi.service`、
`docs/INSTALL-TESTING.md` 第 405–410 行）。

### 3.3 同类案例与它们的修法（可借鉴的「已修好」样本）

| 案例 | 现象 | 修法 | 对本机的意义 |
|---|---|---|---|
| [nabu #35](https://github.com/FisiFla/nabu-cachyos/issues/35)（SM8150） | SLPI `sensor_process` PD 每 40 秒 crash（`USER-PD DOG detects stalled initialization`），加速度计不出现 | `persist` 挂到 `/mnt/vendor/persist`（原来是 `/persist`）；`-s` 而不是 `-S`；udev 加 `ssc-accel` tag | **证明「宿主文件路径不对」足以让 SSC 看门狗咬**，且可修 |
| [piano #3](https://github.com/yzddmr6/xiaomipad-8pro-mainline/issues/3)（SM8750） | ADSP 上的 `qsh_process`/`sns_watchdog` 每 8 次/分 crash，声卡起不来 | 用本机 `persist` 重新 stage `registry-private`，清掉 wedged 的 `/var/lib/.../registry` | **证明 registry 状态不匹配会打死 DSP**；且「换成本机自己的 registry」是有效修法 |
| [tb-j606l #18](https://github.com/haltia-de/device-lenovo-tb-j606l/issues/18)（SM6115） | 同时早启 hexagonrpcd 的 adsp-rootpd + sensorspd → **SoC 直接重启**（进 `05c6:900e` crash dump，pstore 空） | 不早启 RootPD（放弃部分功能） | **证明 FastRPC PD 早启可以把整机搞复位**，与本机形状最接近 |
| [sm-x800 #46](https://github.com/aaronsb/sm-x800-linux/issues/46)（X1E） | SLPI 每个采样周期翻转 SMP2P ready/handover 位；stock 还有 `sleepstate`/`sleepstate_see` 两个 mainline 没描述的 entry | ratelimit/dbg；未修根因 | 提醒：mainline 对 SLPI 的 SMP2P 建模**不完整**（缺 sleepstate），但那是功耗语义，不是复位 |
| [pmaports #3007](https://gitlab.com/postmarketOS/pmaports/-/issues/3007) | OP6 上 `iio-sensor-proxy` 起不来 | 服务/依赖问题 | 用户态链路的常见问题，与复位无关 |

> **没有任何一条公开记录是「SM8450 上 SLPI 一启动整机就复位、并且被修好」的。**

### 3.4 检索方法与限制（要写清楚，避免「查不到 = 不存在」）

- **GitHub**：`api.github.com` 可直连（Windows `curl.exe` + token）。检索了
  `slpi reset` / `slpi sm8450` / `fastrpc-sdsp` / `hexagonrpcd` / `libssc` / `remoteproc reset qualcomm`
  以及 `torvalds/linux` 的 commit/code 搜索（71 条含 `slpi` 的上游 commit，已逐条看标题）。
- **GitLab**：`gitlab.com/api/v4` 可用；查了 `postmarketOS/pmaports`（`slpi` / `ssc sensor` /
  `remoteproc reset` / `hexagonrpcd`）。
- **lore.kernel.org / patchwork.kernel.org：被 Anubis 反爬挡住**（返回
  `Making sure you're not a bot!`，HTTP 200 但内容是 JS challenge），`marc.info` 不可达，
  `spinics.net` 只有列表首页。**因此本报告没有直接引用 lore 线程**；linux-arm-msm 的结论是通过
  上游 commit 历史 + GitHub/GitLab issue + 固件字符串交叉得到的。
- GitHub 上搜到的 `aaronsb/sm-x800-linux`（X1E）、`FisiFla/nabu-cachyos`（SM8150）、
  `haltia-de/device-lenovo-tb-j606l`（SM6115）是最接近的三组一手记录，已在 §3.3 引用。

---

## 4. 绕开 SLPI 拿传感器：可行性

### 4.1 ADSP 路径（不存在，已确认）

- Android/Qualcomm 的传感器框架（SSC / `sns_*`）跑在 **SLPI（sensor low power island）** 上，
  不是 ADSP。证据：
  - SLPI 固件里有 `/dev/qup_ssc`、`/dev/qup_ssc/11`、`/dev/qup_ssc/15`、`SSC_PRAM`、`SSC_SDC`、
    `sns_*`、`/vendor/etc/sensors/sns_reg_config`、`sns_registry` 等整包传感器栈字符串；
  - 原厂 `dspso.bin`（ADSP 固件容器）里只有音频编解码（LC3/aptX 等）字符串，**没有** `sns_`/`ssc` 传感器栈。
- mainline 的 fastrpc 实例是 `sdsp`（SLPI）上的 `label = "sdsp"`；ADSP 的 fastrpc 是给音频
  （`q6apm`/`AudioReach`）用的，不提供 `sns_*` 服务。
- 所以「把 SSC 挂到 ADSP 上」在固件层面就不存在。

### 4.2 libssc / hexagonrpcd 是否支持 ADSP 上的 SSC（不支持）

- `hexagonrpcd` 的上游就是 **Linux MSM 的 FastRPC 反向隧道**（<https://github.com/linux-msm/hexagonrpc>），
  它按 `-f /dev/fastrpc-<dsp>` 打开一个 DSP；SSC 客户端（`libssc`，Codeberg
  `DylanVanAssche/libssc`，GitHub 有镜像 `GUF296/libssc`）只面向 **sdsp**（`/dev/fastrpc-sdsp`）。
- 本项目的 unit 就是 `hexagonrpcd -f /dev/fastrpc-sdsp -d sdsp -s -R /usr/share/qcom/sm8450/Xiaomi/liuqin`
  （`device/sensors-overlay/etc/systemd/system/liuqin-hexagonrpcd-sdsp.service`），
  `iio-sensor-proxy` 也是按 `ssc-accel`/`ssc-light` 的 udev tag 找 `/dev/fastrpc-sdsp*`。
- 结论：**换 DSP 不能绕开 SLPI**；SSC 只存在于 SLPI。

### 4.3 I2C/SPI 直驱（不可行，已确认到「DT 里没有」这一层）

- mainline `liuqin.dts` 里 AP 侧 I2C 只有：`i2c1`（触摸/键盘相关）、`i2c5`、`i2c13`、
  `i2c15`（TL/TR CS35L41）、`i2c17`（BL/BR CS35L41），**没有任何 accelerometer/gyro/ALS 节点**
  （`grep -i 'lsm6\|icm\|bmi\|akm\|ak09\|mmc56'` 在 mainline DT、vendor DT、`images/vendor_dtbs/*.dtb`
  全部 0 命中）。
- vendor DT 的 `board-sensor = "VIRTUAL-SENSOR"`（`_haptics/vendor-qcom-liuqin-sm8475.dtsi` 第 38 行）
  进一步说明：sensor 不是 AP 侧设备，而是 SSC 的「虚拟」传感器。
- 物理上 sensor 挂在 **SSC 自己的 I2C/SPI（QUP）** 上，这些 QUP 由 TZ 分配给 SLPI（`/dev/qup_ssc/*`），
  AP 无法直接访问；要让 AP 拿到，需要改 TZ 的 QUP ownership 配置（不可行）。
- 结论：**没有「LSM6DSO 挂在 AP I2C 上」这种捷径**；即使硬件是 LSM6DSO，AP 也看不到那条总线。

### 4.4 光感（同样在 SSC，不可行）

ALS 也是 SSC 的 sensor 之一（registry 里有 `akm_fusion`、`ambient_light` 等类型；
`iio-sensor-proxy` 的 `ssc-light` 走 `/dev/fastrpc-sdsp`）。没有独立的 AP 侧 ALS。

### 4.5 唯一的非 SLPI 位置类输入：hall（已确认存在，但不是旋转）

`liuqin.dts` 里有 `gpio-keys`：

```text
event-hall-lid          label = "Hall Lid";          → SW_LID
event-hall-tablet-mode  label = "Hall Tablet Mode";  → SW_TABLET_MODE
```

两个 hall 开关是普通 TLMM GPIO（`hall_default` pinctrl），**不经过 SLPI**。
所以即使 SLPI 永久禁用，也能拿到「保护盖/键盘是否合上」「平板模式/笔记本模式」两个布尔量；
但它给不了加速度/陀螺仪/光感，也**不能替代自动旋转**。

### 4.6 结论表

| 目标 | 是否可行 | 原因 |
|---|---|---|
| 走 ADSP 拿 SSC | ❌ | SSC 只在 SLPI 固件里 |
| hexagonrpcd/libssc 换 DSP | ❌ | 只实现 sdsp |
| AP I2C/SPI 直驱 accelerometer | ❌ | 总线归 SLPI（TZ 分配），AP DT 里没有 sensor 节点 |
| 只拿光感 | ❌ | ALS 也在 SSC |
| 拿 hall（盖/键盘） | ✅ | GPIO `gpio-keys`，与 SLPI 无关 |
| 拿自动旋转 | ❌（除非 SLPI 能起） | 需要 accelerometer |

---

## 5. 安全实验方案（**只写方案，不要执行**）

### 5.0 通用安全框架（每组实验都必须遵守）

1. **保命三件套**：`/etc/liuqin-slpi-disabled` 必须在；`boot_a` 必须是 Ubuntu；
   VM 上的 fastboot 自动恢复脚本（`_vm\auto-recover.sh` / `recover-ubuntu.sh`）必须在线。
2. **每次实验前**：`journalctl --sync`（把日志刷盘），并确认 `sync` 完成；
   否则复位后 journal 会丢最后几秒（这正是历史上「没有日志」的原因之一）。
3. **取证**：从 VM 用 `ssh` 串流内核日志（维护者在 issue #10 里的方法）：
   ```bash
   # VM 侧
   ssh <user>@<tablet> 'journalctl -kf' | tee ~/slpi-kmsg-$(date +%H%M).log
   ```
4. **每次复位后**第一件事（只读）：
   ```sh
   tr ' ' '\n' < /proc/cmdline | grep -E 'bootinfo\.(pureason|pdreason)'
   journalctl -b -1 -n 30 --no-pager -o short-precise
   ls -la /sys/fs/pstore/
   ```
5. **禁止**：改 `liuqin-slpi.service` / `/usr/local/sbin/liuqin-slpi`（root contract 会拒绝启动）；
   要手动控制就**直接调脚本** `/usr/local/sbin/liuqin-slpi start|stop`。
6. **每组只改一个变量**，并且记录「实验前 uptime / 实验后 uptime」以确认是复位而不是重启。

### 5.1 实验总表

| # | 目的 | 改动 | 风险 | 预期 | 判定标准 |
|---|---|---|---|---|---|
| E1 | 确认「有宿主文件服务时死得更晚」 | 全栈正常启动（**不动 marker**，直接 `sudo /usr/local/sbin/liuqin-slpi start`） | 中（会复位 1 次） | ~40–60 s 复位 | 复位前 journal 里有 hexagonrpcd 的 registry 报错 → 支持「registry 不匹配」 |
| E2 | 把 registry 换成**本机自己的 persist** 重新 stage | 用 `liuqin-sensor-registry-import` / `stage-sensor-registry.py` 重建 `/var/lib/liuqin-sensors/registry` | 中 | 若 SSC 不再 fatal → 直接修好 | `dmesg` 里 `crash detected in slpi` 计数为 0，且 `/sys/bus/iio/devices/` 出现设备 |
| E3 | 验证 hexagonfs 的 persist 映射 | 只读对比 hexagonrpcd 实际打开/写入的路径 vs `sns_reg.conf` 的 `/mnt/vendor/persist/...` | 低 | 发现路径不等价 | `strace -f -e trace=openat hexagonrpcd ...` 看到 ENOENT/EACCES |
| E4 | 抓「non-whitelisted property」的真实属性名 | 重编 hexagonrpcd，在 `sns_registry.c` 拒绝分支打印属性名（十六进制） | 低（只改用户态） | 看到实际请求的属性名 | 名字在 `sns_reg.conf` 之外 → 扩展白名单；在里面 → 说明是 ABI/长度问题 |
| E5 | 判断 fatal 是否与时间相关 | `liuqin-slpi start` 后 **0.5–1 s 内** `liuqin-slpi stop` | 中 | 若不复位 → fatal 需要时间累积（可能可规避）；若仍复位 → fatal 在更早阶段 | 连续 3 次「start→1s→stop」都不复位 = 强信号 |
| E6 | 用 kernel 的 `recovery=disabled` 排除「恢复流程导致复位」 | `echo disabled > /sys/class/remoteproc/remoteproc0/recovery` 后再 start | 中 | 预期仍复位（因为没 fatal 中断） | 若仍复位 → 恢复流程无关，坐实「固件自己咬狗」 |
| E7 | 换 SLPI 固件版本（不同 ROM 的 `dspso.bin` 提取） | 替换 `/usr/lib/firmware/qcom/sm8475/liuqin/slpi.*` | 高（可能变砖式 bootloop） | 若换版本能活 → 固件与数据的匹配问题 | 需在有按键在场时做；失败要能从 fastboot 恢复 |
| E8 | `slpi_auto_boot=1`（内核早期启动） | 改 cmdline/DT，**必须去掉 marker** | 高（bootloop，无法进系统） | 只在能物理按键恢复时做 | 观察死亡时间是否变化；**不建议远程做** |
| E9 | 关掉 SSC 看门狗（devmem 写 AOSS/SSC WDT 寄存器） | 需要先逆向寄存器地址 | 极高 | 可能直接挂死/无输出 | **不建议**；除非有 JTAG/串口 |

### 5.2 每组的关键细节

**E1（基线复核，低风险但会复位）**

```sh
# 平板（root）：marker 保持原样（否则复位后下次开机会自动再启 SLPI，可能连环复位）
journalctl --sync
sudo /usr/local/sbin/liuqin-slpi start      # 绕过 unit 的 Condition，等价于 unit 的 ExecStart
```

> 注意：`systemctl start liuqin-slpi` 会被 unit 的 `ConditionPathExists=!/etc/liuqin-slpi-disabled`
> 拦住，所以必须直接调脚本（这也正是维护者在 #10 里建议的做法）。

**E2（最值得做：registry 重建）**

```sh
# 步骤 1（VM / 构建机）：从本机 persist 镜像里提取它自己的 registry 目录
debugfs -R "rdump /sensors/registry /tmp/persist-reg" persist.img
# 步骤 2（VM）：把提取出的 registry 目录送到平板（路径按现场调整）
scp -r /tmp/persist-reg/registry tyh@<tablet>:/tmp/registry-own
# 步骤 3（平板）：用官方导入脚本替换（它会自动备份成 registry.backup.<UTC>）
sudo /usr/local/sbin/liuqin-sensor-registry-import /tmp/registry-own
```

判定：`dmesg | grep -ci 'crash detected in slpi'` 为 0，且
`ls /sys/bus/iio/devices/` 出现 `iio:deviceN`（加速度计），`monitor-sensor` 有输出。

**E4（抓属性名，最便宜的「可能直接修好」）**

```sh
# 构建机（VM）：
#   1) 在 hexagonrpcd/sns_registry.c 的拒绝分支加：
#      fprintf(stderr, "reject prop: %.*s\n", (int)prop_name_len, (char *)inbufs[1].p);
#   2) 重编、替换 /usr/bin/hexagonrpcd（先备份）
# 平板：
sudo systemctl restart liuqin-hexagonrpcd-sdsp 2>/dev/null || true
journalctl -u liuqin-hexagonrpcd-sdsp -f
```

判定：如果打印出的属性名不在 `sns_reg.conf` 的两个 `property=` 里，就把它加进白名单
（只返回空串，不暴露通用 Android property 桥），再重复 E1。

---

## 6. 结论：能修 / 不能修 / 需要什么条件

### 6.1 判断

- **不能通过「打个内核补丁」修好。** 复位发生在 Linux 之下的层级，且 mainline 的 SLPI 支持
  （DT、PAS、fastrpc）在本机已经全部正常工作。
- **不是「固件版本不对/固件缺失」这种一眼可修的问题。** 24 个 `slpi.*` 与仓库逐字节一致，
  PAS 认证通过，SLPI 能 up。
- **唯一现实的修法是「让 SSC 启动期不产生 fatal error」**，也就是让它拿到**完全匹配的
  registry / config / file-service**。这条路上有三个具体可测点：
  1. `persist` registry 的内容与本机 SLPI 固件是否匹配（E2）；
  2. hexagonfs 对 `/mnt/vendor/persist/sensors/registry/*` 与 `/sys/devices/soc0/*` 的映射是否完整（E3）；
  3. `sns_registry` 窄垫片是否拒绝了 SSC 真正需要的属性（E4）。
- **如果三点都排除仍复位**：说明 fatal 来自硬件差异（sensor 实际型号/存在性）或安全世界策略
  （XBL/TZ 版本），**远端不可修**；此时保留 `/etc/liuqin-slpi-disabled` 是唯一正确选择。

### 6.2 需要的条件（如果还想继续）

| 条件 | 为什么 | 现状 |
|---|---|---|
| 能物理接触设备（按键进 fastboot） | 任何一次 SLPI 实验都会复位；E7/E8 可能 bootloop | 需用户在场 |
| VM 上的 fastboot 自动恢复在线 | 复位后自动回 Ubuntu | 已有（`_vm\auto-recover.sh`、`recover-ubuntu.sh`） |
| 能重编/替换 hexagonrpcd | E4 需要打印被拒属性名 | 需构建环境（项目有 `tools/build-liuqin-sensors-stack.sh`） |
| 能提取另一版本 ROM 的 `slpi.*` | E7 需要对照固件 | 需 `dspso.bin` 解析（项目已有 ROM 提取脚本） |
| 串口/网络 kmsg 串流 | 抓复位前最后一行 | 维护者在 #10 给了方法；WiFi 可用 |

### 6.3 如果修不了：能拿到什么

- **一个传感器都拿不到**（加速度/陀螺/光感全在 SSC）。
- 唯一可用的替代输入是 **hall**（`SW_LID`/`SW_TABLET_MODE`），可用于「合盖熄屏/键盘模式」，
  但**不能自动旋转**。
- 因此「没有自动旋转」在可预见的将来是这台机器的**固件级限制**，不是配置问题。

---

## 7. 下一步可执行清单（按风险从低到高）

> 全部命令都假定：marker `/etc/liuqin-slpi-disabled` 在、设备能从 fastboot 恢复。
> 只读项（0–2）可以现在就做；3 以后需要用户在场。

**0. 只读基线（零风险，可随时做）**

```sh
# 平板
tr ' ' '\n' < /proc/cmdline | grep -E 'bootinfo|slpi|panic=|hwversion'
for r in /sys/class/remoteproc/remoteproc*; do
  echo "$r $(cat $r/name) $(cat $r/state) recovery=$(cat $r/recovery)"
done
ls -la /sys/fs/pstore/
journalctl --list-boots --no-pager | tail -6
sha256sum /usr/lib/firmware/qcom/sm8475/liuqin/slpi.* | head
```

**1. 对齐「复位前最后一行」（低风险，只需一次实验）**

```sh
# 在 VM（与平板同网）上，ssh 到平板的 sshd（平板需已联网；没有 sshd 就只能靠串口）
ssh <tablet-user>@<tablet-ip> 'journalctl -kf' | tee ~/slpi-kmsg.log
# 平板（另一个会话）
journalctl --sync
sudo /usr/local/sbin/liuqin-slpi start
```

**2. 检查 hexagonfs 映射（低风险，只读 + 用户态 strace）**

```sh
# 平板
sudo strace -f -e trace=openat,newfstatat -o /tmp/hrpc.strace \
  /usr/bin/hexagonrpcd -f /dev/fastrpc-sdsp -d sdsp -s -R /usr/share/qcom/sm8450/Xiaomi/liuqin &
PID=$!; sleep 5; sudo kill "$PID"
grep -E 'persist|soc0' /tmp/hrpc.strace | head -40
```

**3. 抓被拒属性名（低风险，改用户态）**

见 §5.2 E4。若发现新属性名 → 扩展白名单 → 重跑第 1 步。

**4. registry 重建（中风险，会复位一次）**

见 §5.2 E2。

**5. 「start 后 1 秒 stop」时序实验（中风险）**

```sh
sudo /usr/local/sbin/liuqin-slpi start
sleep 1
sudo /usr/local/sbin/liuqin-slpi stop
cat /sys/class/remoteproc/remoteproc0/state
```

判定：连续 3 次都不复位 → fatal 是「时间累积型」，可继续找阈值；
仍复位 → fatal 在 1 秒内，基本无解。

**6. 换固件版本（高风险，需按键在场）**

只在 1–5 全部失败、且用户明确同意时做；先备份现有 24 个文件，
从另一版 ROM 的 `dspso.bin` 提取 `slpi.*`，逐个替换并 `sha256sum` 校验，
每次只换一个分区段（b00–b23），失败立刻回滚。

**7. 不建议做的**

- 改 `liuqin-slpi.service` / `liuqin-slpi` 脚本（root contract 会拒绝启动，直接掉到应急 shell）；
- `slpi_auto_boot=1` 远程实验（可能 bootloop，且无法进系统修）；
- 用 `devmem` 写 AOSS/SSC 看门狗寄存器（几乎必然挂死）。

---

## 8. 证据索引

### 8.1 一手命令输出（本机，只读）

| 内容 | 来源 |
|---|---|
| `pureason=0x80001 / pdreason=0x2`（当前 boot） | `_vm/last-tab-out.txt`（tab-read.sh 运行结果） |
| 历史 boot 的 `pureason/pdreason` 与 journal 结束方式 | `_vm/last-tab-out.txt`（tab-read2.sh） |
| 11:18 全栈实验日志（SLPI up + hexagonrpcd soc0 报错） | `_vm/last-tab-out.txt`（tab-read6.sh） |
| 11:41 隔离实验日志（3 秒复位） | `_vm/last-tab-out.txt`（tab-read3.sh，正文见 §1.2） |
| `remoteproc0..2` 状态、pstore 空、`liuqin-slpi-disabled` 存在 | `_vm/last-tab-out.txt`（tab-read.sh） |
| `sns_reg.conf` 六条 `file=`、两条 `property=`、registry 207 个文件 | `_vm/last-tab-out.txt`（tab-read9.sh） |
| `/sys/devices/soc0` 只有 5 个 mainline 属性 | `_vm/last-tab-out.txt`（tab-read8.sh） |

### 8.2 代码/固件

| 内容 | 路径 |
|---|---|
| `slpi_auto_boot` 门控（`sysmon_name == "slpi"`） | `_vm/qcom_q6v5_pas.c:721-722`；项目 commit `569f79be4`、`a54c2b550` |
| `liuqin-slpi` 启停脚本 | `gh/xiaomipad-6pro-mainline/device/sensors-overlay/usr/local/sbin/liuqin-slpi` |
| unit 的 marker 条件 | `.../etc/systemd/system/liuqin-slpi.service` |
| hexagonrpcd 启动参数 | `.../etc/systemd/system/liuqin-hexagonrpcd-sdsp.service` |
| `sns_registry` 窄垫片（白名单/拒绝分支） | `device/sensors/patches/0002-hexagonrpcd-add-a-narrow-sns_registry-property-shim.patch` |
| 固件字符串（DOG bite / sns / xpu） | `device/firmware/usr/lib/firmware/qcom/sm8475/liuqin/slpi.b08`、`slpi.b10`（`strings` 结果见 §1.4） |
| 固件哈希清单 | `device/firmware/firmware.manifest`（24 个 `slpi.*`） |
| mainline liuqin DT（hall、i2c、SLPI 节点） | `_vm/liuqin.dts`（1997 行；hall 在 181–215，i2c15/17 在 861/912） |
| vendor DT 片段（`board-sensor = "VIRTUAL-SENSOR"`） | `_haptics/vendor-qcom-liuqin-sm8475.dtsi:38` |

### 8.3 上游/社区 URL

- 项目仓库：<https://github.com/yzddmr6/xiaomipad-6pro-mainline>
- 内核仓库：<https://github.com/yzddmr6/linux-sm8450-liuqin>（`liuqin-6.17` / `release/v0.6.0-dp-fingerprint`）
- issue #10：<https://github.com/yzddmr6/xiaomipad-6pro-mainline/issues/10>
- issue #18：<https://github.com/yzddmr6/xiaomipad-6pro-mainline/issues/18>
- 上游 sm8450.dtsi（SLPI 节点）：<https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/qcom/sm8450.dtsi>
- 上游 sm8450-hdk.dts（`&remoteproc_slpi { status = "okay"; }`）：<https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/qcom/sm8450-hdk.dts>
- 上游 sm8450-sony-xperia-nagara.dtsi：<https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/qcom/sm8450-sony-xperia-nagara.dtsi>
- `91d70eb708` sm8450 fastrpc nodes：<https://github.com/torvalds/linux/commit/91d70eb708>
- `5dd227ccfb` sdm845 LPI CX/MX for SLPI：<https://github.com/torvalds/linux/commit/5dd227ccfb>
- `2cb0c97ce4` cx-supply on sdm845-slpi-pas：<https://github.com/torvalds/linux/commit/2cb0c97ce4>
- hexagonrpc：<https://github.com/linux-msm/hexagonrpc>
- libssc（Codeberg）：<https://codeberg.org/DylanVanAssche/libssc>；GitHub 镜像：<https://github.com/GUF296/libssc>
- iio-sensor-proxy：<https://gitlab.freedesktop.org/iio-sensor-proxy/iio-sensor-proxy>
- nabu #35（persist 路径 → PD 看门狗）：<https://github.com/FisiFla/nabu-cachyos/issues/35>
- piano #3（registry → ADSP 看门狗）：<https://github.com/yzddmr6/xiaomipad-8pro-mainline/issues/3>
- tb-j606l #18（早启 PD → SoC 复位）：<https://github.com/haltia-de/device-lenovo-tb-j606l/issues/18>
- sm-x800 #46（SLPI SMP2P sleepstate 缺失）：<https://github.com/aaronsb/sm-x800-linux/issues/46>
- pmaports #3007 / #2455：<https://gitlab.com/postmarketOS/pmaports/-/issues/3007>、<https://gitlab.com/postmarketOS/pmaports/-/issues/2455>

---

## 9. 方法与局限

**方法**

1. 先用只读 shell 把「复位签名」拿到（cmdline bootinfo、per-boot journal、pstore、remoteproc 状态）；
2. 用项目仓库 + 上游 `torvalds/linux` 的 DT/驱动做静态对照，确认「内核侧该做的都做了」；
3. 对 24 个 SLPI 固件做字符串挖掘，找错误处理路径（这是唯一能「看到固件内部」的途径）；
4. GitHub API（issue/commit/code 搜索）+ GitLab API（postmarketOS）做社区检索；
5. 所有结论标注「已确认 / 推断 / 未确认」。

**局限（必须诚实说明）**

- **无法看到 SSC 的 fatal 原因**：没有 SSC UART/QDSS，复位又发生在 AP 之外；pstore 空，
  `q6v5` 连 fatal 中断都没收到。E5（start→1 s stop）是唯一可能「抢在 bite 之前」的办法。
- **lore.kernel.org / patchwork 被反爬挡住**，所以「linux-arm-msm 上没有相关讨论」这句话
  **不能**由本报告下结论；只能说「在可检索的 GitHub/GitLab/上游 commit 里没有」。
- **per-unit 差异无法从本机取证**：`androidboot.hwversion` 相同不代表 XBL/TZ/persist 相同；
  需要故障机与正常机各一份 `sha256sum slpi.*`、`/proc/cmdline`、`persist` 的 sensor 目录清单
  （不含内容）才能进一步定位。
- 本报告没有执行任何会启动 SLPI 的命令；§5 的所有实验都只是方案。
