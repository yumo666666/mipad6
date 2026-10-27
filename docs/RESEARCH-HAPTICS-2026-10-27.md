# Xiaomi Pad 6 Pro（liuqin）振动马达 Linux 驱动可行性调研

- 日期：2026-10-27
- 设备：Xiaomi Pad 6 Pro（liuqin），SM8475，Ubuntu 26.04，内核 `6.17.0-rc1-gfe81794b5e1b`（`github.com/yzddmr6/linux-sm8450-liuqin` 分支 `liuqin-6.17`，commit `fe81794b5e1b`）
- 方法：本地内核/DT/固件静态分析 + 平板只读命令 + GitHub raw/API 源码核对
- 标注约定：**【已确认】**＝有直接证据（源码/DT/命令输出）；**【推测】**＝由证据推断，未物理验证

---

## 0. 摘要（TL;DR）

1. **DT 证据（已确认）**：零售固件 `dtbo` 里，liuqin 自己的 overlay 条目（entry 15，`xiaomi,miboard-id = 0x10`）把 PM8350B HV haptics 和 SWR haptics **两条通路都显式写成 `status = "disabled"`**；同一个 dtbo 里同世代其它 10 个小米机型全部是 `"ok"`。基座 DTB（`vendor_boot` 里 14 个 Qualcomm 参考板 DTB）里**根本没有 haptics 节点**。
2. **由此推断（置信度高）**：本机没有振动马达，所以「振动马达完全不可用」的准确表述是「**系统里没有执行器可驱动**」，而不是「内核缺一个驱动」。写驱动、改 DT、重编内核都无法让不存在的马达振动——只差最后一步物理确认。
3. 主线（torvalds/linux）**没有** PM8350B / hv-haptics / qcom,spmi-haptics 驱动；`drivers/input/misc/` 里 Qualcomm 只有老的 `pm8xxx-vibrator.c`（只支持 PM8058/PM8921/PM8916/PMI632，寄存器模型完全不同）。**【已确认】**
4. 厂商驱动是 `drivers/input/misc/qcom-hv-haptics.c`（**4846 行 / 123,771 字节**），platform driver + 父 SPMI regmap + input FF（`FF_CONSTANT`/`FF_GAIN`）+ sysfs class `qcom-haptics` + debugfs，另依赖下游专有的 `linux/qpnp/qpnp-pbs.h`。**【已确认】**
5. **建议：不做。** 除非先完成「拆机确认真有 LRA 马达」这一步。若真有马达（例如工程机/换过主板），最小驱动约 450–700 行、2–4 人日；完整移植厂商驱动 2–4 周。最大风险不是写不出来，而是**为一个不存在的器件白做**，其次是 hBoost/VMAX 配置错误损伤 PMIC。

---

## 1. 结论先行：这台机器到底有没有马达

### 1.1 证据链（从强到弱）

| # | 证据 | 结论 | 级别 |
|---|---|---|---|
| 1 | 零售 `dtbo.img`（44 个条目）中，**entry 15** = `model = "Liuqin..."`、`msm-id = <531 540 530>`、`xiaomi,miboard-id = <0x10>`；该条目的 `qcom,hv-haptics@f000` 为 **`status = "disabled"`**，`swr_haptics@f0170220` 也为 **`"disabled"`** | 零售 DT 不启用任何触觉通路 | 【已确认】 |
| 2 | 同一 dtbo 中，**11 个小米机型（Waipio/Cape/Ukee 世代，17 个条目）**里，**只有 liuqin 是 disabled**；Diting(12S)、Mayfly、Mondrian、Yudi、Marble、Thor、Cupid、Ingres、Loki、Unicorn 全部是 `status = "ok"` | 「禁用」是 liuqin 的刻意选择，不是模板默认 | 【已确认】 |
| 3 | `vendor_boot.img` 里的 **14 个基座 DTB**（Cape/Diwali/Ukee/Waipio 参考板）**一个 haptics 节点都没有**（`hap=0`） | 节点只可能来自 overlay，而 overlay 给的是 disabled | 【已确认】 |
| 4 | 厂商 `qcom/pm8350b.dtsi` 里 `pm8350b_haptics` 默认 `status = "disabled"`；liuqin 板级 DTS（`qcom/liuqin-sm8475.dtsi`，321 KB）**完全没有** `&pm8350b_haptics { status = "okay" }` 之类覆盖 | 板级从未启用 | 【已确认】 |
| 5 | 我们主线 DT 的 `liuqin-abl-overlay-sink` 节点内容 = dtbo entry 15（`vmax=1250 mV`、`lra-period=5882 us`、`brake-mode=1`、**`status="disabled"`** 完全一致） | 端口抓到的就是零售条目，值没抓错 | 【已确认】 |
| 6 | 厂商内核 `liuqin_GKI.config` 里 `CONFIG_INPUT_QCOM_HV_HAPTICS=m` 是开的，但 DT 禁用 → 驱动永远不 probe | 编译进去 ≠ 有硬件 | 【已确认】 |
| 7 | Xiaomi Pad 6 Pro 属 11 吋中端平板，官方规格页/评测均未提「线性马达」；站内问答有「小米平板 6 Pro 没有震动」条目（[ZOL 问答](https://page.sm.cn/blm/midpage-317/index?id=11_3bfea593c85bb83105f59b1f9e2d21eb&q=%e5%b0%8f%e7%b1%b3%e5%b9%b3%e6%9d%bf6%e6%8c%af%e5%8a%a8%e9%a9%ac%e8%be%be&title=%e5%b0%8f%e7%b1%b3%e5%b9%b3%e6%9d%bf6pro%e6%b2%a1%e6%9c%89%e9%9c%87%e5%8a%a8-ZOL%e9%97%ae%e7%ad%94)） | 与 DT 结论一致 | 【推测】 |

### 1.2 关键纠正

> 之前 `HARDWARE-2026-10-27.md` 写的是「只有 vendor DT 里有 `qcom,hv-haptics@f000`（在 `liuqin-abl-overlay-sink` 下），主线 DT 没实例化，内核也没有对应驱动；要支持得新写驱动 + 改 DT + 重编内核」。
>
> 这句话每一句都对，但**漏了最关键的 `status = "disabled"`**。节点存在只说明「PMIC 里这个 IP 存在」，不代表「板子上焊了马达」。零售 Android 也从来没有振过——不是 Linux 移植的问题。

### 1.3 唯一还缺的一步：物理确认

软件侧已经查到底了（DT 是硬件有无执行器的最终声明）。剩下只有两种低成本确认方式：

1. **拆后盖**：找 LRA 马达（圆形/方形金属块，两根细线接到主板）或马达 FPC 座。Pad 6 Pro 后盖为卡扣+胶，拆解视频可参考。
2. **对照拆解图/维修图**：确认该机型是否有振动器料号。

若拆开确实**没有**马达 → 本任务到此结案（见 §8 结论）。
若**有**马达（工程机、换主板、或 Xiaomi 隐藏了 SKU）→ 按 §6 方案 A 执行。

---

## 2. 主线有没有现成驱动（目标 1）

### 2.1 `drivers/input/misc/` 里的 Qualcomm 触觉驱动

用 GitHub contents API 拉取 `torvalds/linux@master` 的完整目录（98 个条目），其中与振动/触觉相关的只有：

| 文件 | 说明 | 能否用于 PM8350B |
|---|---|---|
| `pm8xxx-vibrator.c` | 老一代 PMIC vibrator，`ff-memless` 框架 | ❌ |
| `gpio-vibra.c` / `pwm-vibra.c` / `regulator-haptic.c` | 通用 GPIO/PWM/regulator 振动器 | ❌（无对应硬件） |
| `aw86927.c` / `cs40l50-vibra.c` / `da7280.c` / `drv260x.c` / `drv2665.c` / `drv2667.c` / `max77693-haptic.c` / `max8997_haptic.c` / `sc27xx-vibra.c` / `twl4030-vibra.c` / `twl6040-vibra.c` / `arizona-haptics.c` / `isa1200.c` | 第三方触觉芯片 | ❌（板上没有） |

`pm8xxx-vibrator.c` 的 compatible 只有 4 个：

```c
{ .compatible = "qcom,pm8058-vib", .data = &pm8058_regs },
{ .compatible = "qcom,pm8921-vib", .data = &pm8058_regs },
{ .compatible = "qcom,pm8916-vib", .data = &pm8916_regs },
{ .compatible = "qcom,pmi632-vib", .data = &pmi632_regs },
```

来源：<https://github.com/torvalds/linux/blob/master/drivers/input/misc/pm8xxx-vibrator.c>

它的模型是「PMIC 内的 LDO/DAC + 使能位」，而 PM8350B 是 **HAP HV（高压 H 桥 + 升压 + 波形引擎）**，寄存器完全不同，**不能复用**。

### 2.2 有没有 `pm8350b` / `hv-haptics` / `qcom,spmi-haptics` 支持

| 探测对象 | 结果 |
|---|---|
| `drivers/input/misc/pm8350b-haptics.c` | 404 【已确认】 |
| `drivers/input/misc/qcom-hv-haptics.c` | 404 【已确认】 |
| `drivers/input/misc/hv-haptics.c` | 404 【已确认】 |
| `drivers/input/misc/qcom-spmi-vibrator.c`、`spmi-haptics.c` | 404 【已确认】 |
| `drivers/mfd/pm8350b.c`、`qcom-pm8350b.c` | 404 【已确认】 |
| `Documentation/devicetree/bindings/input/qcom,spmi-haptics.yaml` | **200（存在）** |
| `Documentation/devicetree/bindings/input/qcom,pm8350b-haptics.yaml`、`qcom,hv-haptics.yaml` | 404 |

**关于 `qcom,spmi-haptics.yaml`**：这是 2026-08-07 合入的 **binding**（commit `660ee6792edf`，作者 Fenglin Wu，`Link: https://patch.msgid.link/20260806-qcom-spmi-haptics-v6-...`），针对 **PMIH0108** 里的 **HAP530_HV** 模块：

```yaml
compatible:
  items:
    - const: qcom,pmih0108-haptics
    - const: qcom,spmi-haptics
reg:
  items: [HAP_CFG base, HAP_PTN base]
```

但**驱动本体没有进 master**：在 master 全树搜索 `pmih0108-haptics` 只命中这个 yaml；`HAP_CFG` 字符串只出现在该 yaml 与两个网卡驱动里。也就是说 **截至 2026-10-27，主线只有文档、没有代码**（该 v6 系列以 patch 1 = binding 提交，驱动补丁未见于 master）。【已确认】

### 2.3 mainline 的 `pm8350b.dtsi` 现状

主线的 `arch/arm64/boot/dts/qcom/pm8350b.dtsi` 只有：

```dts
&spmi_bus {
	pm8350b: pmic@3 {
		compatible = "qcom,pm8350b", "qcom,spmi-pmic";
		reg = <0x3 SPMI_USID>;
		#address-cells = <1>;
		#size-cells = <0>;

		pm8350b_temp_alarm: temp-alarm@a00 { ... };
		pm8350b_gpios: gpio@8800 { ... };
	};
};
```

**没有** haptics 节点、**没有** `qcom,pbs@1900`、**没有** nvmem cell。主线对 `pm8350b` 也没有专属 MFD 驱动，靠 `qcom,spmi-pmic` 兜底绑定。

### 2.4 小结

| 问题 | 答案 |
|---|---|
| 主线上有现成的 PM8350B haptics 驱动吗？ | **没有** |
| 有可参考的上游同族驱动吗？ | 只有 **binding**（PMIH0108 HAP530_HV），驱动未合入 |
| `pm8xxx-vibrator` 能改了直接用吗？ | **不能**，寄存器模型不同 |
| 需要新写驱动吗？ | 是（如果有马达） |

---

## 3. 厂商驱动长什么样（目标 2）

### 3.1 位置与规模

| 项 | 值 |
|---|---|
| 路径 | `drivers/input/misc/qcom-hv-haptics.c` |
| 仓库 | `MiCode/Xiaomi_Kernel_OpenSource` 分支 `liuqin-t-oss`（ref `eb49e100`）|
| 同款镜像 | `LineageOS/android_kernel_xiaomi_sm8450` 分支 `lineage-24.0` 同路径 |
| 大小 | **123,771 字节 / 4846 行** |
| Kconfig | `CONFIG_INPUT_QCOM_HV_HAPTICS`（`liuqin_GKI.config` 里 `=m`）|
| Makefile | `obj-$(CONFIG_INPUT_QCOM_HV_HAPTICS) += qcom-hv-haptics.o` |

链接：
- <https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/liuqin-t-oss/drivers/input/misc/qcom-hv-haptics.c>
- <https://github.com/LineageOS/android_kernel_xiaomi_sm8450/blob/lineage-24.0/drivers/input/misc/qcom-hv-haptics.c>

### 3.2 框架与接口（重点）

```c
static const struct of_device_id haptics_match_table[] = {
	{ .compatible = "qcom,hv-haptics" },
	{ .compatible = "qcom,pm8350b-haptics" },
	{}
};
static struct platform_driver haptics_driver = {
	.driver = { .name = "qcom-hv-haptics", .of_match_table = haptics_match_table,
		    .pm = &haptics_pm_ops },
	.probe = haptics_probe, .remove = haptics_remove,
};
module_platform_driver(haptics_driver);
```

| 维度 | 实现 |
|---|---|
| 总线 | **platform driver**（子节点挂在 SPMI `pmic@3` 下，由 MFD populate 成 platform device） |
| 寄存器访问 | `chip->regmap = dev_get_regmap(chip->dev->parent, NULL)`，即**复用父 SPMI PMIC 的 regmap** |
| 用户态接口 | **input FF**：`FF_CONSTANT`、`FF_GAIN`；有 effect 子节点时再加 `FF_PERIODIC`/`FF_CUSTOM`/`FF_DAMPER` |
| 额外接口 | sysfs class **`qcom-haptics`**（`class_register`）、debugfs |
| 中断 | `fifo-empty`（threaded IRQ，`IRQF_ONESHOT`） |
| 电源 | `devm_regulator_get(dev, "qcom,hpwr")`（可选，缺失时为 dummy）；内部 hBoost 由寄存器控制 |
| 下游专有依赖 | `#include <linux/qpnp/qpnp-pbs.h>`（`qpnp_pbs_trigger_single_event`，由 `qcom,pbs-client` 触发，**可选**） |
| 其他 | nvmem（`hap_cfg_sdam` / `hap_cl_brake`，闭环刹车校准，**可选**）、SWR slave regulator 子节点（给 SoundWire 通路用，**可选**）、hrtimer 管理 hBoost 关断 |

### 3.3 关键函数

| 函数 | 作用 |
|---|---|
| `haptics_parse_dt()`（L3932） | 解析 `qcom,vmax-mv`、`qcom,lra-period-us`、`qcom,brake-*`、`qcom,drv-sig-shape`、effect 子节点 |
| `haptics_hw_init()`（L2678） | 存 CL brake 参数、读 5V clamp、设 VMAX、设驱动波形/刹车模式、RC_CLK 自动校准、设 T_LRA、预载 effect |
| `haptics_module_enable()`（L1876） | 写 `HAP_CFG_EN_CTL` BIT(7) |
| `haptics_set_vmax_mv()`（L1140） | `VMAX = mV / 50`（上限 11 V；若 clamp 在 5 V 则封顶 5 V） |
| `haptics_set_direct_play()`（L1216） | 写 `HAP_PTN_DIRECT_PLAY`（幅度 0–255） |
| `haptics_enable_play()`（L1453） | 清 fault → 开环驱动配置 → 写 `SPMI_PLAY`（`PLAY_EN` + pattern src）→ 保持/延时关 hBoost |
| `haptics_set_brake()`（L1500） | brake mode + 写 `HAP_PTN_BRAKE_AMP` 刹车波形 |
| `haptics_playback()`（L2515） | FF playback 回调；constant 走 DIRECT_PLAY，effect 走 PATTERN/FIFO |
| `haptics_set_gain()`（L2595） | FF gain，DIRECT_PLAY 模式下按增益缩放幅度 |
| `haptics_boost_vreg_enable()`（L1233） | 通过 **nvmem 写 SDAM**（`0x42` 参数 + `0xE5` 触发）让 PBS 开/关 hBoost |

### 3.4 复杂度评估

4846 行里真正跟「让马达转一下」有关的不到 400 行，其余是：

- effect 库与 `qcom,wf-*` 解析（PATTERN1/2、FIFO 波形）；
- FIFO 流式播放 + `fifo-empty` 中断补数据（长波形/自定义波形）；
- 闭环谐振跟踪（`rc_clk_cal`、`t_lra` 校准、auto-resonance 错误恢复）；
- hBoost 时序（20 ms 等待、延时关断定时器）；
- SWR（SoundWire）通路与 `hap-swr-slave-reg` regulator；
- debugfs、suspend/resume、calibration。

---

## 4. PM8350B HV haptics 的硬件接口（目标 3）

### 4.1 寄存器窗口与中断（来自厂商 DT + 驱动）

```dts
reg = <0xf000>, <0xf100>, <0xf200>;      /* HAP_CFG / HAP_PTN / HAP_BOOST */
interrupts = <0x3 0xf0 0x1 IRQ_TYPE_EDGE_RISING>;   /* SPMI usid 3, periph 0xf0, irq 1 */
interrupt-names = "fifo-empty";
```

驱动里对应 `cfg_addr_base / ptn_addr_base / hbst_addr_base` 三个基址，所有访问 = `base + offset`，经父 regmap（SPMI）读写。

### 4.2 关键寄存器（HAP_CFG @ 0xf000）

| offset | 名称 | 关键位/含义 |
|---|---|---|
| 0x01 | `REVISION2` | HAP_CFG v1=1 / v2=2 |
| 0x0C | `FAULT_STATUS` | 短路/过流等故障 |
| 0x10 | `INT_RT_STS` | BIT(1) `FIFO_EMPTY` |
| 0x46 | `EN_CTL` | **BIT(7) `HAPTICS_EN`** |
| 0x47 | `DRV_CTRL` | `PSTG_DLY[7:6]`、`DRV_SLEW_RATE[2:0]` |
| 0x48 | `VMAX` | **mV / 50**（步进 50 mV，上限 11000） |
| 0x49 | `DRV_WF_SEL` | 波形形状 `[1:0]`、数据格式 BIT(4) |
| 0x4A | `AUTO_SHUTDOWN_CFG` | 自动关断 |
| 0x4C | `SPMI_PLAY` | **BIT(7) `PLAY_EN`**、BIT(3) `BRAKE_EN`、`PAT_SRC[2:0]` |
| 0x50 | `BRAKE_MODE_CFG` | `mode[7:6]`、`sine_gain[3:2]`、`brake_wf[1:0]` |
| 0x5C/0x5D | `TLRA_OL_HIGH/LOW` | 开环周期，**步进 5 us**，上限 20475 us |
| 0x5E/0x5F | `RC_CLK_CAL_COUNT` | RC 时钟校准计数（谐振跟踪用） |
| 0x63 | `AUTORES_CFG` | **BIT(7) 自动谐振使能** |
| 0x66 | `FAULT_CLR` | BIT(2) SC、BIT(1) AUTO_RES_ERR、BIT(0) HPWR_RDY |
| 0x67 | `VMAX_HDRM` | headroom，步进 50 mV，上限 6350 |
| 0x68 | `VSET_CFG` | BIT(0) `FORCE_VREG_RDY`（非 hBoost 供电时强制） |
| 0x72 | `CAL_EN` | `CAL_RC_CLK[3:2]`：0=关、1=自动 |

### 4.3 HAP_PTN @ 0xf100 / HAP_BOOST @ 0xf200

| 窗口 | offset | 名称 | 说明 |
|---|---|---|---|
| PTN | 0x26 | `DIRECT_PLAY` | **幅度 0–255**，直接播放正弦 |
| PTN | 0x30–0x33 | `PTRN1/2_TLRA` | 每段 pattern 的周期 |
| PTN | 0x34 / 0x50 | `PTRN1/2_CFG` | pattern 配置/长度 |
| PTN | 0x70 | `BRAKE_AMP` | 刹车波形样本（最多 8 字节） |
| PTN | 0x20–0x2C | FIFO DIN / play rate / empty cfg | v1 与 v2 布局不同 |
| BOOST | 0x00/0x01 | `REVISION1/2` | v0p0=0 / v0p1=1 |
| BOOST | 0x41 | `HW_CTRL_FOLLOW` | 硬件跟随使能 |
| BOOST | 0x46 | `VREG_EN` | 升压使能位 |
| BOOST | 0xF1 / 0x70 | `CLAMP_5V` | v0p0 / v0p1 的 5 V 钳位 |

### 4.4 最小「响一下」序列【推测，按厂商代码归纳】

```
1. EN_CTL(0x46)      = 0x80                  ; 模块上电
2. FAULT_CLR(0x66)   = 0x07                  ; 清 SC / autores / hpwr 故障
3. VMAX(0x48)        = vmax_mv / 50          ; 例：1250 mV → 25
4. AUTORES_CFG(0x63) |= 0x80                 ; DIRECT_PLAY 必须开自动谐振
5. TLRA_OL(0x5C/0x5D)= lra_period_us / 5     ; 例：5882 us → 1176
6. BRAKE_MODE(0x50)  = mode<<6 | gain<<2 | wf
   PTN_BRAKE_AMP(0x70)= <pattern bytes>
7. PTN_DIRECT_PLAY(0x26) = amplitude(0..255)
8. SPMI_PLAY(0x4C)   = 0x80 | PAT_SRC(DIRECT_PLAY=0)
9. 到时/停止：SPMI_PLAY(0x4C) = 0，EN_CTL(0x46) = 0
```

注意 hBoost：`EN_CTL` 的写入会给 hBoost 投票，厂商在 toggle 前后各加 100 us 延时，并等待 hBoost ready 最多约 20 ms；`VMAX` 必须 ≤（hpwr 电压 − headroom），否则可能过压。

### 4.5 寄存器手册可得性

| 来源 | 可得性 |
|---|---|
| Qualcomm PM8350B datasheet / 寄存器手册 | ❌ 不公开（NDA 文档） |
| 下游驱动源码（GPLv2） | ✅ 完整，含全部寄存器定义与注释（见 §3） |
| 下游 binding 文档 `bindings/input/qcom-hv-haptics.txt` | ✅ 完整描述硬件语义（DIRECT_PLAY/FIFO/PATTERN/SWR、`qcom,wf-*` 全部属性） |
| 上游 `qcom,spmi-haptics.yaml` | ✅ 描述 HAP530_HV（PMIH0108）的同类行为，可作术语对照，**寄存器布局不同，不能照抄** |

结论：**没有手册，但下游驱动 + binding 足够逆向出一个最小驱动**；风险在于边界条件（过压/短路/时序）只能靠试。

---

## 5. DT 侧要写什么（目标 4）

### 5.1 零售 DT 的实际内容（liuqin，dtbo entry 15）

```dts
qcom,hv-haptics@f000 {
	compatible = "qcom,hv-haptics";
	reg = <0xf000>, <0xf100>, <0xf200>;
	interrupts = <0x3 0xf0 0x1 IRQ_TYPE_EDGE_RISING>;
	interrupt-names = "fifo-empty";
	qcom,vmax-mv = <1250>;
	qcom,brake-mode = <1>;            /* BRAKE_CLOSE_LOOP */
	qcom,brake-pattern = /bits/ 8 <0xff 0x3f 0x1f>;
	qcom,lra-period-us = <5882>;      /* ≈170 Hz */
	qcom,drv-sig-shape = <1>;         /* WF_SINE */
	qcom,brake-sig-shape = <1>;
	nvmem-cell-names = "hap_cl_brake";
	nvmem-cells = <&...>;
	nvmem-names = "hap_cfg_sdam";
	nvmem = <&...>;
	qcom,pbs-client = <&pm8350b_pbs2>;
	status = "disabled";              /* ← 关键 */

	effect_0..effect_5 { ... };       /* 6 个 effect 全是同一份模板波形 */
};
```

> 6 个 effect 的 `qcom,wf-pattern-data` 完全相同，说明这段是从公共模板拷来的，并非为 liuqin 调过——进一步支持「没有实际调校过的执行器」。

### 5.2 主线要不要改

| 项 | 结论 |
|---|---|
| 主线 `pm8350b.dtsi` 有节点吗 | 没有（只有 temp-alarm、gpio） |
| 主线需要加什么 | 在 `pm8350b` 下加一个子节点（`reg` 三段、`interrupts` 四段 SPMI 格式） |
| 需要改 MFD 驱动吗 | **不需要**。主线 `drivers/mfd/qcom-spmi-pmic.c` 会 `devm_regmap_init_spmi_ext()` 建 regmap（L259），并 `devm_of_platform_populate()`（L284）把子节点变成 platform device；`dev_get_regmap(parent)` 直接可用 |
| 需要改 SPMI 控制器吗 | 不需要（`CONFIG_SPMI=y`、`CONFIG_REGMAP_SPMI=y`，`/sys/bus/spmi/devices/0-03` 已绑定 `pmic-spmi`） |

### 5.3 DT 草案（**仅在确认有马达后**才加）

```dts
/* arch/arm64/boot/dts/qcom/sm8475-xiaomi-liuqin.dts（或 liuqin.dts） */

&pm8350b {
	hap: haptics@f000 {
		compatible = "qcom,pm8350b-haptics", "qcom,hv-haptics";
		reg = <0xf000>, <0xf100>, <0xf200>;
		interrupts = <0x3 0xf0 0x1 IRQ_TYPE_EDGE_RISING>;
		interrupt-names = "fifo-empty";

		qcom,vmax-mv = <1250>;
		qcom,lra-period-us = <5882>;
		qcom,brake-mode = <1>;			/* close loop */
		qcom,brake-pattern = /bits/ 8 <0xff 0x3f 0x1f>;
		qcom,drv-sig-shape = <1>;		/* sine */
		qcom,brake-sig-shape = <1>;

		status = "okay";
	};
};
```

说明：
- `&pm8350b` 是主线 `pm8350b.dtsi` 里的 label，`pmic@3` 已有 `#address-cells = <1>` / `#size-cells = <0>`，子节点写 `reg = <0xf000>` 即可；
- 三段 `reg` 分别对应 CFG / PTN / BOOST，需要驱动自己按顺序取；
- 上面用到的枚举来自厂商 `include/dt-bindings/input/qcom,hv-haptics.h`（`WF_SINE = 1`、`BRAKE_CLOSE_LOOP = 1`、`BRAKE_SINE_GAIN_X1 = 0`、`S_PERIOD_T_LRA = 0`），**主线没有这个头文件**，本地模块可直接用数值或自带一份；
- 上游 binding（`qcom,spmi-haptics.yaml`）是 **不同 PMIC**（PMIH0108），不能直接套；如果只做本地模块，**不写 YAML 也能跑**（`checkpatch`/`dtbs_check` 会警告）；
- **运行时 overlay 不可用**：`CONFIG_OF_OVERLAY=y` 但 `CONFIG_OF_CONFIGFS` 未开，`/sys/kernel/config/device-tree/overlays` 不存在 → DT 改动必须**重编 DTB 并重打包 boot**，不能热加载。

---

## 6. 如果真有马达：工作量、时间、风险（目标 6）

### 6.1 三条路线

| 路线 | 做法 | 代码量 | 时间（有硬件可测） | 评价 |
|---|---|---|---|---|
| **A. 最小自研驱动** | 只做 DIRECT_PLAY + `FF_CONSTANT` + `FF_GAIN` + 简单 brake；不做 effect/FIFO/SWR/PBS | **~450–700 行 C** + 20 行 DT | **2–4 人日**（盲写 1 周） | 推荐；够用（点击/长振） |
| **B. 移植厂商驱动** | 拿 `qcom-hv-haptics.c` 改 5.10→6.17：`qpnp-pbs`、nvmem、regulator、`input_ff` API、`class_create` 等 | 4846 行（改动 10–20%） | **2–4 周** | 只有需要波形库/低延迟 effect 才值得 |
| **C. 等上游** | 等 `qcom,spmi-haptics`（PMIH0108）驱动合入后适配 PM8350B | — | 不可控（驱动尚未进 master） | 不推荐 |

### 6.2 最小驱动骨架（示意，非完整代码）

```c
struct hv_hap {
	struct device *dev;
	struct regmap *map;          /* = dev_get_regmap(dev->parent, NULL) */
	struct input_dev *input;
	u32 vmax_mv, lra_us;
};

static int hv_hap_play(struct input_dev *idev, int effect_id, int val)
{
	struct hv_hap *h = input_get_drvdata(idev);
	if (!val)
		return regmap_write(h->map, 0xf04c, 0);      /* SPMI_PLAY = 0 */

	regmap_write(h->map, 0xf046, 0x80);               /* EN_CTL: enable */
	regmap_write(h->map, 0xf066, 0x07);               /* FAULT_CLR */
	regmap_write(h->map, 0xf048, h->vmax_mv / 50);    /* VMAX */
	regmap_update_bits(h->map, 0xf063, 0x80, 0x80);   /* AUTORES on */
	/* TLRA: 0xf05c[3:0] = msb, 0xf05d = lsb, 单位 5us */
	regmap_write(h->map, 0xf05d, (h->lra_us / 5) & 0xff);
	regmap_write(h->map, 0xf05c, (h->lra_us / 5) >> 8);
	regmap_write(h->map, 0xf026, 0xff);               /* DIRECT_PLAY 幅度 */
	return regmap_write(h->map, 0xf04c, 0x80);        /* PLAY_EN | DIRECT_PLAY */
}
```

（实际还要：brake 配置、hBoost 上电等待、`hrtimer` 自动关断、`FF_GAIN` 缩放、suspend/resume。）

### 6.3 风险清单

| 风险 | 等级 | 说明 | 缓解 |
|---|---|---|---|
| **硬件根本不存在** | 🔴 极高 | 本报告 §1 的主结论 | 先拆机确认，再动手 |
| 过压/过流损伤 PMIC 或 LRA | 🟠 中高 | `VMAX`、hBoost、clamp 配置错误；无手册 | 从零售 DT 的 1250 mV 起，逐级加；先读 `FAULT_STATUS` |
| 寄存器语义理解错 | 🟠 中 | 只能靠下游代码推断，v1/v2 布局不同 | 先读 `REVISION2` 判版本；对照 `pm8350b.dtsi` 的 vmax/lra |
| 内核 API 漂移（5.10→6.17） | 🟡 中 | `qpnp-pbs` 不存在；`input_ff`/`class_create` 签名变化 | 方案 A 绕开 PBS，只用 regmap+FF |
| 每次改 DT 都要重打包 boot | 🟡 中 | 无 configfs overlay | 用项目现有 `build-liuqin-kernel.py` + `repack_boot.py` 流程 |
| 内核升级后模块失效 | 🟡 中 | vermagic 必须精确匹配（`6.17.0-rc1-gfe81794b5e1b`），无 MODVERSIONS | 跟内核一起重编（项目已有 `build-liuqin-kernel-modules.sh`） |
| 桌面没有触觉反馈 | 🟢 低 | GNOME/libinput 不做 haptics；即使驱动成功，只有 `fftest`/`evtest` 能触发 | 用 `fftest` 验证；需要桌面反馈得自己写 daemon |

---

## 7. 有没有更简单的替代（目标 5）

| 方案 | 可行性 | 说明 |
|---|---|---|
| **USB / 蓝牙手柄震动** | ✅ 最实际 | Linux 原生 `hid-generic` + `FF_RUMBLE`，`fftest` 直接可用；平板当主机接手柄，游戏/模拟器有反馈 |
| 手机/手环联动 | ✅ | 通过 KDE Connect / 自建 MQTT 让手机振；不是平板自身 |
| 扬声器低频「体感」 | ❌ | 只能出声，无法产生有效振动；大音量低音可能轻微共振，不可控 |
| 触控板/键盘反馈 | ❌ | 该键盘保护套无马达 |
| 外接线性马达 + USB 驱动板 | 🟡 | 需要自制 USB HID 设备（如 RP2040 + DRV2605），可行但属于外设工程 |
| 硬件改装（焊 LRA 到 PM8350B HAP 输出） | 🟡 | 需要主板级返修 + 确认 HAP 输出是否引出；风险高，不推荐 |

**结论**：如果目标是「有振动反馈」，外接手柄是唯一低成本、当天可用的方案；如果目标是「让这台平板本身振」，在没有马达的前提下无解。

---

## 8. 最终结论

| 问题 | 回答 |
|---|---|
| **能不能做？** | 技术上能（有完整下游驱动可参考），但**本机大概率没有执行器**，做了也不会振 |
| **需要改内核吗？** | 需要：① 加 DT 节点（必须重编 DTB、重打包 boot，无法运行时 overlay）；② 加驱动（可作为外部模块 `insmod`，`CONFIG_MODULES=y`、`MODULE_SIG=n`、`MODVERSIONS=n`，只要 vermagic 匹配即可，无需整核重编） |
| **只加 DT + 编一个模块行不行？** | 行——前提是**马达真的存在**。`input_ff_create` 已内建（`CONFIG_INPUT_FF_MEMLESS=y`），父 SPMI regmap / 子设备 populate 都已就绪，模块可以直接 `dev_get_regmap(parent)` 拿寄存器 |
| **要多久？** | 有马达可测：最小驱动 2–4 人日；完整移植 2–4 周。没马达：0，且永远不会有结果 |
| **最大风险？** | ① 硬件不存在（白做，概率最高）；② 无手册导致过压/时序错误，损伤 PMIC 或马达；③ 即使成功，桌面端也没有触觉反馈生态，只有测试工具能触发 |
| **建议** | **先拆机确认马达是否存在**（5 分钟）。没有 → 结案，改用外接手柄；有 → 按 §6 方案 A 做最小驱动 |

---

## 9. 下一步可执行清单

### 第 0 步（必做，5 分钟）

- [ ] 拆后盖，确认是否有 LRA 马达（圆形/方形金属块 + 2 根线到主板）或马达 FPC 座
- [ ] 拍照记录马达型号/位置（若有）
- [ ] 若不便拆机：查该机型拆解视频/维修图，确认振动器料号

### 分支 A：确认没有马达（预期结果）

- [ ] 结案，不再投入驱动开发
- [ ] 更新 `HARDWARE-2026-10-27.md` 里「振动马达」那一行：补上 `status = "disabled"` 的证据
- [ ] 如需振动反馈：买一个支持 FF 的 USB/蓝牙手柄（`fftest` 验证），或做手机联动提醒

### 分支 B：确认有马达（异常机型/工程机）

1. [ ] 取参数：以 dtbo entry 15 的 `vmax=1250 mV`、`lra-period=5882 us`、`brake-mode=1` 为初值
2. [ ] 读 `REVISION2`（0xf001）确认 HAP_CFG v1/v2，读 `FAULT_STATUS`（0xf00c）确认无故障
3. [ ] 写最小驱动（§6.2），只做 DIRECT_PLAY + `FF_CONSTANT`/`FF_GAIN`
4. [ ] 加 DT 节点（§5.3，`status = "okay"`），重编 `sm8475-xiaomi-liuqin.dtb`
5. [ ] 重打包 boot + 安装模块（沿用 `tools/build-liuqin-kernel.py`、`tools/build-liuqin-kernel-modules.sh`、`_vm/repack_boot.py`）
6. [ ] 验证：`dmesg | grep -i haptic`、`/sys/class/input/*/capabilities/ff`、`fftest /dev/input/eventX`
7. [ ] 逐级调 `vmax`（1250 → 2000 → 2700 mV），每次读 `FAULT_STATUS`，确认不触发 SC/HPWR 故障
8. [ ] 若需要点击/长振以外的手感，再考虑 effect 子节点 / FIFO（方案 B）

### 第 2 步（无论哪个分支）

- [ ] 把本报告的结论同步到项目 issue/文档，避免后人重复调研
- [ ] 保留证据脚本：`<workdir>\_haptics\scan_dtbo2.py`、`scan_vendor_boot.py`（可重跑复核）

---

## 附录 A：证据与复核命令

### A.1 零售 dtbo 扫描（本地）

```powershell
python <workdir>\_haptics\scan_dtbo2.py     # 44 个条目，打印每个条目的 haptics status
python <workdir>\_haptics\dump_entry.py     # 打印 entry 15 / 17 的完整 haptics 属性
python <workdir>\_haptics\scan_vendor_boot.py  # 14 个基座 DTB，确认 hap=0
```

输出要点：

```
[15] xiaomi=16 msm=(531, 65536, 540, 65536, 530, 65536) model='Liuqin based on Qualcomm Tec' hap_nodes=9
       status 'disabled'   .../qcom,hv-haptics@f000
       status 'disabled'   .../swr_haptics@f0170220
[17] xiaomi=15 model='Marble...'  status 'ok'  .../qcom,hv-haptics@f000
```

### A.2 平板只读复核

```sh
# haptics 节点与状态（vendor overlay sink）
find /proc/device-tree/liuqin-abl-overlay-sink -maxdepth 3 -iname '*haptic*'
cat /proc/device-tree/liuqin-abl-overlay-sink/qcom,pm8350b@3/qcom,hv-haptics@f000/status

# 没有 FF 设备
for d in /sys/class/input/input*; do echo "$d $(cat $d/name) ff=$(cat $d/capabilities/ff)"; done
ls /sys/class/leds/

# 内核能力（已确认支持 FF / SPMI / 模块）
zcat /proc/config.gz | grep -E 'CONFIG_(INPUT_FF_MEMLESS|INPUT_UINPUT|SPMI|REGMAP_SPMI|MODULES)'
grep -m1 input_ff_create /proc/kallsyms
```

实测输出（2026-10-27）：

```
CONFIG_INPUT_FF_MEMLESS=y
CONFIG_INPUT_MISC=y
CONFIG_INPUT_UINPUT=y
CONFIG_MODULES=y
# CONFIG_MODULE_SIG is not set
# CONFIG_MODVERSIONS is not set
CONFIG_SPMI=y
CONFIG_REGMAP_SPMI=y
input_ff_create: ffffd448c024722c T input_ff_create
/sys/class/input/input0..9  ff=0   （全部无 FF 能力）
/sys/class/leds/ 只有 input2::capslock 等键盘 LED
```

### A.3 源码链接

| 内容 | URL |
|---|---|
| 主线 `drivers/input/misc/` 目录 | <https://github.com/torvalds/linux/tree/master/drivers/input/misc> |
| 主线 `pm8xxx-vibrator.c` | <https://github.com/torvalds/linux/blob/master/drivers/input/misc/pm8xxx-vibrator.c> |
| 主线 `qcom,spmi-haptics.yaml` | <https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/input/qcom,spmi-haptics.yaml> |
| 该 binding 的 commit | <https://github.com/torvalds/linux/commit/660ee6792edf> |
| 主线 `pm8350b.dtsi` | <https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/qcom/pm8350b.dtsi> |
| 主线 `qcom-spmi-pmic.c` | <https://github.com/torvalds/linux/blob/master/drivers/mfd/qcom-spmi-pmic.c> |
| 厂商 `qcom-hv-haptics.c` | <https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/liuqin-t-oss/drivers/input/misc/qcom-hv-haptics.c> |
| LineageOS 镜像 | <https://github.com/LineageOS/android_kernel_xiaomi_sm8450/blob/lineage-24.0/drivers/input/misc/qcom-hv-haptics.c> |
| 厂商 `pm8350b.dtsi` | <https://github.com/Xiaomi-SM8475/android_vendor_qcom_proprietary_devicetree/blob/liuqin-t-oss/qcom/pm8350b.dtsi> |
| 厂商 binding txt | <https://github.com/Xiaomi-SM8475/android_vendor_qcom_proprietary_devicetree/blob/liuqin-t-oss/bindings/input/qcom-hv-haptics.txt> |
| 项目内核 | <https://github.com/yzddmr6/linux-sm8450-liuqin>（分支 `liuqin-6.17`，commit `fe81794b5e1b`） |

### A.4 本地留档

| 文件 | 内容 |
|---|---|
| `<workdir>\_haptics\vendor-qcom-hv-haptics.c` | 厂商驱动全文（123,771 B） |
| `<workdir>\_haptics\binding.txt` | 厂商 binding 全文 |
| `<workdir>\_haptics\vendor-qcom-liuqin-sm8475.dtsi` | 厂商板级 DTS（321 KB，无 haptics 引用） |
| `<workdir>\_haptics\vendor-pm8350b.dtsi` | 厂商 PMIC dtsi（haptics 默认 disabled） |
| `<workdir>\_haptics\dtbo-scan.json` | 44 个 dtbo 条目的结构化扫描结果 |
| `<workdir>\_haptics\scan_dtbo2.py` / `dump_entry.py` / `scan_vendor_boot.py` | 复核脚本 |

---

## 附录 B：术语

| 术语 | 含义 |
|---|---|
| LRA | Linear Resonant Actuator，线性谐振马达 |
| ERM | Eccentric Rotating Mass，转子马达 |
| HV haptics / HAP | Qualcomm PMIC 内的高压触觉模块（H 桥 + 升压 + 波形引擎） |
| hBoost | HAP 模块内部的升压电路 |
| DIRECT_PLAY | 硬件按 `qcom,lra-period-us` 输出正弦，幅度由单字节控制 |
| PATTERN1/2 | 预置波形段序列，可低延迟播放 |
| FIFO | 8-bit 样本流式播放，靠 `fifo-empty` 中断补数据 |
| SWR | SoundWire，PM8350B 的另一条触觉通路（`qcom,pm8350b-swr-haptics`） |
| PBS | Programmable Boot Sequencer，只能通过 nvmem/SDAM 触发（下游专有） |
| SPMI USID | SPMI 从设备地址，本机 PM8350B 为 3 |
| FF | Linux input force-feedback 子系统（`/dev/input/eventX` + `fftest`） |
| dtbo | Device Tree Blob Overlay，ABL 按 msm-id/board-id/miboard-id 选择并应用 |
