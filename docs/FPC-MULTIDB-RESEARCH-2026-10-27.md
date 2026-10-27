# 每根手指一个独立模板库文件（multi-DB）：设计 + 可行性报告

> 目标：把现在"一个 OEM 库装多根手指"改成"**每根手指一个模板文件**"，让 GNOME 能看到/单独删除每一根，同时所有手指都能解锁。
> 状态：**设计 + 可行性研究完成，未改动设备上任何已安装组件**（只读探测 + `/tmp` 内编译/实验）。
> 前置阅读：`FPC-MULTIFINGER-WORKING-2026-10-27.md`（当前"单库多指"已可用方案）。
> 标记约定：【已确认】= 有源码/命令输出证据；【推测】= 由证据推断但未实测；【待验证】= 需实验确认。

---

## 0. 结论先说（TL;DR）

| 方案 | 能否达成"每根手指一个文件 + 都能解锁" | 工作量 | 风险 | 建议 |
|---|---|---|---|---|
| **A. 驱动实现 `identify` vfunc** | 能，但只有在"一次采集能匹配多个库"时才划算 | 1.5–3 天（取决于客户端是否要改） | 中高：要重写驱动的匹配主流程；**会让每次录入多一次 identify 按压** | **暂不做**，先做 A0 验证实验 |
| **B. 给 fprintd 打补丁（遍历 print）** | **能** | 0.5–1 天（补丁已写好并通过应用/编译校验） | 中：改的是我们自己已经编译的 daemon，可回滚 | ✅ **推荐** |
| **C. 驱动内部多库、fprintd 只看到一个 print** | **不能**（UI/删除粒度由 fprintd 的 host storage 决定） | — | — | ❌ 不采用（理由见 §5） |

**推荐路线 = B + 配套改 `native_enrol.py`**（两处都必做，缺一不可）。

**关键约束（实测）**：多库方案下"一次 VerifyStart 里换库"必然要**换一次按压**（每个库一次 OEM 会话），单次按压前后开销约 **6–10 s**；而 `sudo` 走的是
`/etc/pam.d/common-auth:18 → pam_fprintd.so debug max-tries=2 timeout=15`，**15 s 只够 2 次按压**，所以落地时必须把 `timeout` 提到 30–40 s，否则"非第一根手指"的 sudo 会经常失败。

---

## 1. 根因：为什么现在只能"一个文件装多根"

### 1.1 fprintd 1.94.5 的确切行为【已确认】

源码：`source/fprintd-vendor.tar.gz` → `fprintd-v1.94.5/src/device.c`（本机解压到 `work/fprintd-src/`）。

`fprint_device_verify_start()`，`device.c:1648-1764`：

```c
1675:  if (finger == FP_FINGER_UNKNOWN)
1676:    {
1677:      gallery = load_user_prints (rdev, session->username);
...
1686:      else if (gallery->len == 1)          /* 只有 1 个模板：直接用它 */
1688:          FpPrint *fprint = g_ptr_array_index (gallery, 0);
1692:          finger = fp_print_get_finger (fprint);
1694:      else if (fp_device_has_feature (priv->dev, FP_DEVICE_FEATURE_IDENTIFY))
1698:          for (i = 0; i < gallery->len; i++)   /* 只有支持 IDENTIFY 才会全部下发 */
1702:              g_debug ("adding finger %s to the gallery", ...);
...
1708:  if (fp_device_has_feature (priv->dev, FP_DEVICE_FEATURE_IDENTIFY) &&
1709:      finger == FP_FINGER_UNKNOWN)
1711:      priv->current_action = ACTION_IDENTIFY;
1716:      fp_device_identify (priv->dev, gallery, ...);      /* IDENTIFY 路径 */
1720:  else
1722:      if (gallery)
1726:          print = g_object_ref (g_ptr_array_index (gallery, 0));   /* ← 只取 gallery[0] */
1727:          g_ptr_array_remove_index_fast (gallery, 0);
1729:          finger = fp_print_get_finger (print);
1751:      fp_device_verify (priv->dev, print, ...);          /* 只验这一个 */
```

`verify_cb()`，`device.c:1558-1601`，重试时也只重试同一个 print：

```c
1575:  if (error && error->domain == FP_DEVICE_RETRY)
1577:      fp_device_verify (priv->dev,
1578:                        priv->verify_data,        /* ← 同一个 print，永不前进 */
```

**结论**：`gallery->len > 1` 且设备没有 `FP_DEVICE_FEATURE_IDENTIFY` 时，`gallery[1..n]` 被**静默丢弃**，且重试也不前进。这与我们观察到的"录了第二根、只有第一根能解锁"完全一致。

### 1.2 我们现在的方案为什么没事【已确认】

`work/fpc1264-oem.c:170-180`（round2 驱动）：录入成功后用 `fp_print_set_finger()` 把返回 print 的手指号改回**库里原来那根**，于是 fprintd 的存储层覆盖同一个文件 → 磁盘上永远 1 个文件 → `gallery->len == 1` → 走 1686 分支，绕开上面的坑。代价就是 GNOME 只显示 1 条、不能单独删。

### 1.3 多库方案还缺的另外两块（容易被忽略）

1. **录入侧**：`native_enrol.py:160-170`【已确认】

```python
prints = FPRINT_ROOT / username / "fpc1264_oem" / "liuqin-fpc1264-oem"
# One OEM database per account carries every enrolled finger.
stored = [finger for finger in "123456789a" if os.path.lexists(prints / finger)]
if len(stored) > 1:
    print(f"native_enrol=REFUSED ambiguous_templates={len(stored)}", flush=True)
    return 2
existing = prints / stored[0] if stored else None
canonical_finger = int(stored[0], 16) if stored else 0
```

只要磁盘上出现第 2 个模板文件，**录入直接被拒绝**（`ambiguous_templates`）。所以多库方案必须同时改这里，否则"能录第二根"这一步就走不通。

2. **每根手指必须是独立 DB**：现在的 `--enrol-existing <base> <out>` 是把新手指**追加**进 base 库再整体导出（`fpc_build_info.c` 的 `--bio-pipeline-existing`），所以每个文件天然会包含所有已录手指。要多库就必须改成"**新手指 → 全新单模板 DB**"（即走原来的 `--enrol` 无 base 路径），或者按需只扩展"同一根手指"的文件。

---

## 2. 设备现状（实测事实，作为方案前提）

| 项目 | 值 | 证据 |
|---|---|---|
| fprintd | `1.94.5-4`（私有 daemon 在 `/usr/local/lib/liuqin-fpc-oem/fprintd`） | `dpkg -l`；`fprintd-fpc1264-oem.conf:14` |
| libfprint | `libfprint-2-2 1:1.95.1+tod1-0ubuntu1` | `dpkg -l`；`strings libfprint-2.so.2.0.0` → `Initializing FpContext (libfprint version 1.95.1+tod1)` |
| TOD ABI 库 | `libfprint-2-tod1 1:1.95.1+tod1-0ubuntu1`，`/usr/lib/aarch64-linux-gnu/libfprint-2-tod.so.1`（267,088 B，**独立文件，不是符号链接**） | `ls -l` / `dpkg -L libfprint-2-tod1` |
| 驱动 | `/usr/local/lib/liuqin-fpc-oem/tod/libfprint-tod-fpc1264-oem.so`（76,384 B，round2） | `ls -l` |
| 模板 | `/var/lib/fprint/<user>/fpc1264_oem/liuqin-fpc1264-oem/2`，**1,162,014 B（1 个文件、2 个模板）** | `ls -l` |
| 驱动编译方式 | `cc -shared -fPIC -O2 -std=c11 -D_GNU_SOURCE -Wall -Wextra -I$S/usr/include -I$S/usr/include/libfprint-2 -I$S/usr/include/libfprint-2/tod-1 … -o libfprint-tod-fpc1264-oem.so /usr/lib/aarch64-linux-gnu/libfprint-2-tod.so.1 …` | `_vm/fp2-01-build.sh:1187` |
| PAM（sudo/登录） | `auth sufficient pam_fprintd.so debug max-tries=2 timeout=15` | `/etc/pam.d/common-auth:18` |
| PAM（GDM/锁屏） | `auth required pam_fprintd.so`（默认 max-tries=3、无 timeout） | `/etc/pam.d/gdm-fingerprint:4` |

### 2.1 TOD 的 ABI 结构（这是方案 A 能否成立的关键）【已确认】

`libfprint-2.so.2.0.0` 自身**不导出任何 `fpi_*` 符号**（`nm -D … | grep -c " fpi_"` = `0`），内部 API 全部由 TOD shim 提供：

```
$ nm -D --defined-only /usr/lib/aarch64-linux-gnu/libfprint-2-tod.so.1 | grep -c " fpi_"
273
$ nm -D --defined-only …/libfprint-2-tod.so.1 | grep "identify"
0000000000012668 T fpi_device_get_identify_data@@LIBFPRINT_TOD_1.0.0
00000000000143b0 T fpi_device_identify_complete@@LIBFPRINT_TOD_1.0.0
0000000000013700 T fpi_device_identify_report@@LIBFPRINT_TOD_1.0.0
```

`libfprint-2.so.2.0.0` 里的加载器字符串：`../libfprint/tod/tod-shared-loader.c`、`FP_TOD_DRIVERS_DIR`、`FP_DRIVERS_ALLOWLIST`、`fpi_tod_shared_driver_get_type`、符号版本 `LIBFPRINT_TOD_1.0.0 / 1_1.92 / 1_1.94 / 1_1.95 / 1_1.95.1`。

即：**驱动模块对着冻结的 tod-1 头文件编译，由 shim 提供 `fpi_*`，真正的 libfprint 负责调度**。这就是我们能在设备上"只换一个 .so"的原因，也是方案 A 的技术基础。

---

## 3. 方案 A：给 TOD 驱动实现 `identify` vfunc

### 3.1 API 契约（头文件原文）

`/root/fpcbuild/sysroot/usr/include/libfprint-2/tod-1/fpi-device.h`（来自 `libfprint-2-tod-dev`，与已安装 shim 同版本）：

```c
/* fpi-device.h: "Class elements added after tod-v1" 之前的回调区 */
  void (*enroll)   (FpDevice *device);        /* :181 */
  void (*verify)   (FpDevice *device);        /* :182 前一行 */
  void (*identify) (FpDevice *device);        /* :182 */
  void (*capture)  (FpDevice *device);
  void (*list)     (FpDevice *device);
  void (*delete)   (FpDevice * device);
  void (*cancel)   (FpDevice *device);

  /* Class elements added after tod-v1 */
  FpDeviceFeature features;                   /* ← 可以直接写，也可以用 auto_init */
```

```c
void fpi_device_get_identify_data (FpDevice *device, GPtrArray **prints);   /* :279 */
void fpi_device_identify_complete (FpDevice *device, GError *error);        /* :322 */
void fpi_device_identify_report   (FpDevice *device, FpPrint *match,
                                   FpPrint *print, GError *error);          /* :347 */
```

`fp-device.h:67`：`FP_DEVICE_FEATURE_IDENTIFY = 1 << 1`。

**语义**（结合实测，见 §3.3）：`identify` = "对 gallery（调用方给的一组 `FpPrint`）做一次搜索"，完成时必须通过 `identify_report()` 报告"匹配到 gallery 里的哪一项"（或报告错误），然后 `identify_complete()`；`print` 参数只用于"模板存在设备里"的 MoC 场景，我们是 host storage，传 `NULL`。

### 3.2 fprintd 什么时候走 IDENTIFY？gallery>1 时是否一定走？

- **条件**（`device.c:1708-1709`）：`fp_device_has_feature (dev, FP_DEVICE_FEATURE_IDENTIFY) && finger == FP_FINGER_UNKNOWN`。
- `finger` 来自 `VerifyStart(finger_name)`：`"any"` → `FP_FINGER_UNKNOWN`（`device.c:565-579`）。`pam_fprintd` 固定发 `"any"`（`pam/pam_fprintd.c:496-505`）。
- 所以：**只要驱动声明了 IDENTIFY，且客户端发 `"any"`，fprintd 一定走 IDENTIFY**（`device.c:1711-1718`），gallery 就是全部 print。
- 反过来：**gallery==1 时不走 IDENTIFY**（走 1686 分支的 VERIFY）。所以方案 A 只在"多文件"时生效，正好是我们要的场景。
- 注意：如果客户端明确指定手指名（`VerifyStart("right-index-finger")`），则只加载那一个 print（`device.c:1733`），不会走 gallery/IDENTIFY。

### 3.3 驱动里 `identify` 到底收到什么？【已确认（实测）】

我在设备 `/tmp` 里做了一个**不碰硬件的** TOD 模块 + 宿主程序，用**已安装的 shim/libfprint** 实测（编译命令与真驱动完全同款）：

```
$ FP_TOD_DRIVERS_DIR=/tmp/mdb/tod FP_DRIVERS_ALLOWLIST=mdb_test_identify \
  FP_MDB_TEST_ENABLE=1 /tmp/mdb/harness
devices=1
dev[0] driver=mdb_test_identify features=0x6 identify=1 verify=1
** Message: MDB identify invoked gallery=2 mode=match-second fingers=2,7
RESULT identify_sync ok=1 err=none match_finger=7 newprint=(nil)
```

含义（全部【已确认】）：

1. **TOD 模块实现 `identify` 后，`fpi_device_class_auto_initialize_features()` 会自动置上 `FP_DEVICE_FEATURE_IDENTIFY`**（features=0x6 = IDENTIFY|VERIFY），不需要手写 `features`。
2. `fpi_device_get_identify_data()` 拿到的是**调用方给的 `GPtrArray<FpPrint*>`**（本例 2 项，手指号 2 和 7），不是图像。
3. `fpi_device_identify_report(device, gallery[1], NULL, NULL)` + `identify_complete(NULL)` → 调用方拿到 `match_finger=7`，`ok=1`。**一次 identify 会话只需一次"完成"。**
4. **"采集到了但没匹配"必须报错，不能什么都不报**：

```
** (process): libfprint-device-WARNING **: Driver reported successful identify complete
    but did not report the result earlier. Reporting error instead
RESULT identify_sync ok=0 err=An unspecified error occurred!
```

   正确写法是 `fpi_device_identify_report (device, NULL, NULL, error)`，其中
   - `FP_DEVICE_ERROR_DATA_NOT_FOUND` → fprintd 映射成**终结**状态 `verify-no-match`（`device.c:623-625`）；
   - `FP_DEVICE_RETRY_*` → `verify-retry-scan`，fprintd 会重新 `fp_device_identify()`（`device.c:1620-1628`），即**再给用户一次按压机会**。

### 3.4 致命约束：一次采集只能匹配**一个** DB【已确认】

`fpc_build_info.c` 的 `match_existing()`（`:1001-1064`）是唯一能"真匹配"的客户端入口，流程固定为：

```
BIO_INIT → init_hw_auth → SET_GID → load_database(一个文件) → SET_ACTIVE_GROUP
→ LOAD_INSTANCE_DATA → template_inventory → set_auth_challenge
→ wait_finger_down(IRQ) → CAPTURE_ENROL(一次采集) → identify(11/4, TA 内部对该 DB 匹配)
→ get_auth_result(HAT) → BIO_UPDATE_TEMPLATE → (changed 时) store_database 导出
```

也就是说：**"导入一个 DB"和"匹配"是同一个会话里的顺序动作，客户端没有"一次采集、多库匹配"的接口**。`identify` 回调要覆盖 N 个库，只有两条路：

| 实现方式 | 按压次数 | 说明 |
|---|---|---|
| **A1：驱动内串行 N 个会话** | N 次（与方案 B 相同） | 每次 `oem_runtime --match-existing <db>` 一个库，驱动自己挑匹配的那个 print 上报 |
| **A2：扩展客户端为 `--identify db1 db2 …`** | 1 次（理想） | 需要验证 TA 是否允许"先 capture、后多次 import+identify"（当前顺序相反：先 import 再 capture），见 §10 待验证 |

A1 的实测成本（下面 §3.5 有完整计时）：**每个库一次会话**，pre-press 就 3.7–5.3 s。

### 3.5 实测耗时（决定 UX 的关键数字）【已确认】

在设备上用**模板副本**（`/tmp/fp-re2/raw3.db`，772,495 B，单模板 raw DB，不碰 `/var/lib/fprint`）跑了一次 `--match-existing`，按行打时间戳，16 s 后 SIGTERM 取消：

| 阶段 | 时间 | 说明 |
|---|---|---|
| 进程启动 → `normal_world_power=held` | ~0 s | |
| listener + TA session + `hw_auth=OK` | 0.24 s | TA 会话建立很便宜 |
| **DB 导入**（`4/14` 起，`4/17` 分片 ×190，每片 4080 B） | **3.29 s**（17.3 ms/片，≈235 KB/s） | 主要成本 |
| `SET_ACTIVE_GROUP`/`LOAD_INSTANCE_DATA`/inventory | 0.13 s | `loaded_template_count=1` |
| **`READY finger_irq_armed=down press finger now`** | **3.70 s** | ← pre-press 总成本 |
| 之后等到 SIGTERM（取消） | 12.3 s | 取消路径干净：`oem_runtime_cleanup firmware_path_restored=1 sensor_power_off=1 listener_stopped=1` |

推算当前 1,162,014 B（双模板）库：导入 ≈ 4.95 s，**pre-press ≈ 5.3 s**；加按压、capture、identify、HAT、自适应更新/导出，**单次"一个库的验证"约 6–10 s**。

> 实验后设备状态复核：firmware path 为空、`spi1.0/power_state=off`、无残留 `qsee-supplicant`、fprintd `active`、模板文件大小/mtime 未变（1,162,014 B / 17:24）→ **没有副作用**。

### 3.6 副作用：声明 IDENTIFY 会改变**录入**流程【已确认】

`device.c:2239-2258`（`EnrollStart`）：

```c
2239:  if (fp_device_has_feature (priv->dev, FP_DEVICE_FEATURE_IDENTIFY))
2241:      g_autoptr(GPtrArray) all_prints = load_all_prints (rdev);
2250:      fp_device_identify (priv->dev, all_prints, …, enroll_identify_cb, rdev);
2259:  else
2264:      g_warning ("Device %s does not support duplicate identification …");
```

即：**一旦驱动声明 IDENTIFY，每次 `EnrollStart` 都会先跑一次 identify 做重复检测**。对我们是"每次录入多一次 OEM 会话 + 多一次按压"（而且 `enroll_identify_cb` 把匹配当成 `enroll-duplicate` 直接拒绝，见 `device.c:2077-2130`）。方案 B 没有这个副作用（fprintd 仍认为设备不支持 IDENTIFY）。

### 3.7 方案 A 结论

- 技术上**完全可行**（§3.3 已实测 tod-1 ABI 支持 identify，且 fprintd 会走 IDENTIFY 分支）。
- 但如果只做 A1（串行会话），**收益为 0**：按压次数、会话开销与方案 B 完全相同，却要在驱动里重写"gallery 遍历 + 每库暂存 + 取消 + 自适应更新归属 + 与 fprintd 的错误码约定"，还要额外承担 §3.6 的录入副作用。代码量约 200–250 行，风险高于 B 的 ~50 行。
- 只有 A2（一次采集多库）才有真正收益（**1 次按压解锁所有手指**），但它依赖 TA 语义未验证，且需要改 `fpc_build_info.c`（新增 `--identify` 模式）。

**因此：A 不作为首选；先做 §10 的 A0 实验，若 A0 成功再考虑把 A2 作为"体验升级"叠加在 B 之上。**

---

## 4. 方案 B：给 fprintd 打补丁（推荐）

fprintd 本来就是我们自己编译、私有安装的（`source/fprintd-vendor.tar.gz` + `fprintd-oem/prepare.py` + `fprintd-fpc1264-oem.conf:14`），改它不引入新的供应链。

### 4.1 改哪里（精确到函数）

| 位置 | 行（1.94.5 原始） | 改动 |
|---|---|---|
| `struct _FprintDevicePrivate` | `device.c:114` 之后 | 新增 `GPtrArray *verify_gallery`（本次 VerifyStart 还没试的 print，按手指顺序） |
| `fprint_device_finalize` 的泄漏断言 | `device.c:255-261` | 断言里加上 `verify_gallery` |
| `report_verify_status` 之后 | `device.c:1292` | 新增 `report_verify_next_print()`（发**非终结** `verify-retry-scan` + 再发一次 `VerifyFingerSelected`）与 `compare_print_finger()`（按手指号排序） |
| `stoppable_action_completed` | `device.c:1518` | 动作结束时清空队列 |
| `match_cb` | `device.c:1553-1555` | "没匹配 + 还有队列" → 不发终结状态，直接交回 `verify_cb` |
| `verify_cb` | `device.c:1574-1586` | 新增分支：取下一个 print，重新 `fp_device_verify()` |
| `fprint_device_verify_start` | `device.c:1726-1730` | 取 `gallery[0]` 后把**剩下的**排序放进队列 |

### 4.2 设计要点（为什么这么改）

1. **不消耗 PAM 的重试次数**：`pam_fprintd.c:277-281` 只在 `done == true` 时才把状态记成终结结果；非终结状态只走 `send_err_msg()`（`pam_fprintd.c:283-289`）提示"再放一次手指"。所以"换库"期间发 `verify-retry-scan`，PAM 的 `max_tries` **不会**被扣。
2. **不能被 `verify_status_reported` 吞掉最终匹配**：`report_verify_status()` 对**终结**状态会置 `session->verify_status_reported = TRUE`（`device.c:1278-1291`），之后任何终结状态都会被 "Verify status already reported. Ignoring …" 丢弃。因此**中间库失败时绝不能发 `verify-no-match`**——这正是补丁在 `match_cb` 里提前 `return` 的原因。
3. **最终一根手指保持原语义**：队列空时完全走原路径，`verify-no-match` 只在**所有** print 都试过之后才发出。
4. **取消/停止**：`VerifyStop` 只 `g_cancellable_cancel(priv->current_cancellable)`（`device.c:1532-1533`），驱动侧收到取消 → `verify_cb` 拿到 `G_IO_ERROR_CANCELLED` → 补丁的 `else if` 分支因 `!g_cancellable_is_cancelled()` 不成立而不触发，正常终结。`verify_has_completed()`（`device.c:1785-1790`）看 `verify_data` 非空，遍历期间始终非空，`VerifyStop` 的等待循环行为不变。
5. **Claim/Release 不动**：Claim 是 fprintd 层概念（`device.c:1094-1160`），遍历全程仍是一次 Claim；OEM runtime 的互斥由 `oem_runtime.py:108-114` 的 `operation.lock` 保证，因为我们的会话是串行的。
6. **D-Bus 语义**：`VerifyStart("any")` 仍然立刻返回；`VerifyFingerSelected` 允许多次发（`pam_fprintd.c:294-323` 只是提示语，且 `finger_str_to_msg()` 对 10 个合法手指名都有词条，见 `pam/fingerprint-strings.h:111-160`）；`VerifyStatus` 的 `done=false` 事件本来就是协议的一部分（驱动的 `verify-retry-scan` 一直这么用）。
7. **指定手指名不受影响**：`VerifyStart("right-index-finger")` 仍只加载该 print（`device.c:1733`），不走队列。
8. **顺序确定**：`file_storage_discover_prints()` 是 `g_dir_read_name()` + `g_slist_prepend()`（`file_storage.c:288-308`），gallery 顺序取决于目录项顺序、不可预期。补丁在入队前用 `g_ptr_array_sort (gallery, compare_print_finger)` 按手指号排序（写法与 fprintd 自己的 `garbage_collect_sort` 一致，`device.c:1874-1879`），因此按压顺序稳定：`left-thumb … right-little`；配合每次切换发 `VerifyFingerSelected`，用户始终知道该按哪根。

### 4.3 补丁 diff（完整，已通过应用 + 编译校验）

生成脚本：`work/fprintd-multidb.py`（同 `fprintd-oem/prepare.py` 的"锚点校验 + 生成 patch"风格，锚点缺失即报错退出）。
产物：`work/fprintd-multidb.patch`（143 行）。

```diff
--- a/src/device.c
+++ b/src/device.c
@@ -112,6 +112,9 @@
 
   /* Required to restart the operation on a retry failure. */
   FpPrint   *verify_data;
+  /* Prints still to try for the verify action in progress, in finger
+   * order.  Only used when the device cannot identify (see verify_start). */
+  GPtrArray *verify_gallery;
   GPtrArray *identify_data;
   int        enroll_data;
 
@@ -255,6 +258,7 @@
   if (priv->current_action != ACTION_NONE ||
       priv->_session ||
       priv->verify_data ||
+      priv->verify_gallery ||
       priv->identify_data ||
       priv->current_cancellable ||
       priv->current_cancel_invocation)
@@ -1289,6 +1293,39 @@
 
   if (done)
     session->verify_status_reported = TRUE;
+}
+
+/* Multiple stored templates without device-side identification: walk them one
+ * press at a time.  The current print was captured and did not match, but the
+ * client must not treat that as the final answer while another print is still
+ * queued, so report the non-terminal retry status (clients keep waiting and do
+ * not consume a retry of their own) and tell the user which finger is next. */
+static void
+report_verify_next_print (FprintDevice *rdev)
+{
+  FprintDevicePrivate *priv = fprint_device_get_instance_private (rdev);
+  FpPrint *next = g_ptr_array_index (priv->verify_gallery, 0);
+  FpFinger finger = fp_print_get_finger (next);
+
+  g_debug ("next stored print queued for this verification: %s",
+           fp_finger_to_name (finger));
+  g_signal_emit (rdev, signals[SIGNAL_VERIFY_STATUS], 0, "verify-retry-scan", FALSE);
+
+  if (FP_FINGER_IS_VALID (finger))
+    g_signal_emit (rdev, signals[SIGNAL_VERIFY_FINGER_SELECTED], 0,
+                   fp_finger_to_name (finger));
+}
+
+/* file_storage_discover_prints() walks the print directory in dirent order, so
+ * the gallery is not ordered by finger.  Sort it so the presses are requested
+ * in a stable, predictable order (left thumb 鈥?right little). */
+static gint
+compare_print_finger (gconstpointer a, gconstpointer b)
+{
+  FpPrint *pa = *(FpPrint * const *) a;
+  FpPrint *pb = *(FpPrint * const *) b;
+
+  return (gint) fp_print_get_finger (pa) - (gint) fp_print_get_finger (pb);
 }
 
 static void
@@ -1515,6 +1552,9 @@
       session->verify_status_reported = FALSE;
     }
 
+  /* The queued prints only belong to the verify action that built
+   * them; never let them survive into the next one. */
+  g_clear_pointer (&priv->verify_gallery, g_ptr_array_unref);
   g_clear_object (&priv->current_cancellable);
 }
 
@@ -1551,6 +1591,15 @@
 
   cancelled = g_cancellable_is_cancelled (priv->current_cancellable);
   matched = match != NULL && cancelled == FALSE;
+
+  if (!matched && !error && !cancelled && priv->verify_gallery &&
+      priv->verify_gallery->len > 0)
+    {
+      /* Not the final answer: the remaining prints are tried by
+       * verify_cb() with one more press each. */
+      report_verify_next_print (rdev);
+      return;
+    }
 
   report_verify_status (rdev, matched, error);
 }
@@ -1580,6 +1629,27 @@
                         match_cb, rdev, NULL,
                         (GAsyncReadyCallback) verify_cb,
                         rdev);
+    }
+  else if (!error && !match && priv->verify_gallery &&
+           priv->verify_gallery->len > 0 &&
+           !g_cancellable_is_cancelled (priv->current_cancellable))
+    {
+      /* The current print did not match.  No terminal status was sent
+       * for it (see match_cb), so continue with the next stored print
+       * while the client keeps waiting. */
+      FpPrint *next = g_object_ref (g_ptr_array_index (priv->verify_gallery, 0));
+
+      g_ptr_array_remove_index (priv->verify_gallery, 0);
+      g_clear_object (&priv->verify_data);
+      priv->verify_data = next;
+
+      fp_device_verify (priv->dev,
+                        priv->verify_data,
+                        priv->current_cancellable,
+                        match_cb, rdev, NULL,
+                        (GAsyncReadyCallback) verify_cb,
+                        rdev);
+      return;
     }
   else
     {
@@ -1721,12 +1791,26 @@
     {
       g_autoptr(FpPrint) print = NULL;
 
+      /* Only a completed action can have left prints queued. */
+      g_clear_pointer (&priv->verify_gallery, g_ptr_array_unref);
+
       if (gallery)
         {
-          /* TODO: Use g_ptr_array_remove_index_fast with GLib >= 2.58 */
           print = g_object_ref (g_ptr_array_index (gallery, 0));
-          g_ptr_array_remove_index_fast (gallery, 0);
+          g_ptr_array_remove_index (gallery, 0);
           finger = fp_print_get_finger (print);
+
+          /* The device cannot identify on its own and the user has
+           * more than one print stored: queue the rest for this very
+           * VerifyStart instead of silently verifying gallery[0]
+           * forever. */
+          if (gallery->len > 0)
+            {
+              g_ptr_array_sort (gallery, compare_print_finger);
+              priv->verify_gallery = g_ptr_array_ref (gallery);
+              g_debug ("%u further stored prints queued for verification",
+                       gallery->len);
+            }
         }
       else
         {
```

### 4.4 与现有 OEM 补丁的**组合顺序**（重要）【已确认】

`fprintd-oem/prepare.py` 的工作方式是：定位 `match_cb` 函数体，把其中**唯一一行**
`  report_verify_status (rdev, matched, error);` 替换成"自适应模板更新 + 上报"的一大段。因此：

> **必须先跑 `fprintd-multidb.py`，再跑 `fprintd-oem/prepare.py`。**

本补丁特意做到：新增代码全部插在那一行**之前**，且不引入同名行。已实测两者叠加成功（本机与设备 `/tmp` 各跑一遍）：

```
$ python3 fprintd-multidb.py <vendor> /tmp/mdb/fp-multi     → patched …/src/device.c
$ python3 prepare.py      /tmp/mdb/fp-multi /tmp/mdb/fp-oem → (无输出=成功，锚点全部命中)
$ wc -l generated-multidb.patch generated-oem.patch
  143 generated-multidb.patch
   63 generated-oem.patch
$ grep -n "priv->verify_data = next;" /tmp/mdb/fp-oem/src/device.c
1654:      priv->verify_data = next;
```

（`prepare.py` 的锚点缺失会直接 `SystemExit("Upstream patch anchor missing: …")`，所以"没有报错"本身就是锚点匹配的证据。）

### 4.5 重试次数 / `max-tries` / `timeout` 的真实交互（**这是本方案最大的落地风险**）

**代码层面**【已确认】：不会耗光次数（§4.2 第 1 条）。
**配置层面**【已确认 + 实测耗时】：会撞 `timeout`。

| 消费者 | 配置 | 一次 VerifyStart 的预算 | 多库（2 根手指）实际需要 |
|---|---|---|---|
| `sudo`（`common-auth:18`） | `max-tries=2 timeout=15` | **15 s** | 第一根失败 ≈6–8 s + 第二根 ≈6–10 s ⇒ **14–18 s**，经常超时 |
| GDM / 锁屏（`gdm-fingerprint:4`） | 默认 `max-tries=3`，无 timeout | 无硬限制 | 可以，但最坏 2–3 次按压 |

**结论**：落地时必须把 `common-auth` 那行改成例如
`auth sufficient pam_fprintd.so max-tries=2 timeout=40`（或 `max-tries=3 timeout=40`），
否则"用第二根手指 sudo"会经常失败——这是多库方案相对现状的**体验倒退点**，必须接受或规避。

规避选项（按推荐度）：
1. **调 timeout（最简单）**：40 s 足够 3 次按压。
2. **限制单次遍历数量**：例如一次 VerifyStart 最多试 3 个库，剩下的留给下一次 PAM try（改 `verify_start` 里的入队上限，一行）。
3. **记忆优先手指**：把上次成功的手指放到队首（进程内存即可），常态化情况下第一次按压就中。

### 4.6 构建 / 替换（**不要在本次研究里做**）

```sh
# 1) 取源码 + 打补丁（顺序不能反）
tar -xzf fprintd-vendor.tar.gz -C /tmp/build
python3 fprintd-multidb.py  /tmp/build/fprintd-v1.94.5 /tmp/build/multi      --patch-output /tmp/build/multi.patch
python3 fprintd-oem/prepare.py /tmp/build/multi      /tmp/build/multi-oem     --patch-output /tmp/build/oem.patch

# 2) 用项目既有脚本对着设备 ABI 构建（它已处理 sysroot / -l:libXXX.so.N 等细节）
tools/build-fprintd-oem.sh /tmp/build/multi-oem        # 产物：fprintd（静态链接 fprintd-private）

# 3) 私有安装（只替换 drop-in 指向的那个二进制，不动系统 /usr/libexec/fprintd）
install -o root -g root -m 0700 /tmp/build/multi-oem/fprintd \
        /usr/local/lib/liuqin-fpc-oem/fprintd.new
# 备份现网二进制后再原子替换：
#   cp -p /usr/local/lib/liuqin-fpc-oem/fprintd /root/fpc-backup-2026-10-27/binaries/fprintd.multidb-backup
#   mv /usr/local/lib/liuqin-fpc-oem/fprintd.new /usr/local/lib/liuqin-fpc-oem/fprintd
systemctl restart fprintd        # ⚠️ 仅在真正落地时执行
```

> 本次**没有**执行第 3 步（按任务要求不改设备）。构建依赖（glib/polkit/dbus/pam 的 `-dev`）此刻设备上并没有安装，需要先 `apt-get download` 一批 dev 包解到 sysroot（项目历史上就是这么做的，见 `_vm/fpb-09-devsysroot.sh`）。

### 4.7 本次做到的验证（以及没做到的）

| 验证项 | 结果 | 证据 |
|---|---|---|
| 补丁锚点在**真实 vendor 源码**上全部命中 | ✅ | 本机 + 设备 `/tmp/mdb` 各一次，`fprintd-multidb.py` 无报错 |
| 与既有 OEM 补丁可叠加（顺序正确） | ✅ | `prepare.py` 无报错，`generated-oem.patch` 63 行 |
| 补丁用到的 GLib/libfprint 惯用法在**设备真实头文件**下类型正确 | ✅ | `cc -fsyntax-only -Wall -Wextra -Werror=incompatible-pointer-types` + GLib 2.88 / libfprint 1.95.1 头文件，`idiom_rc=0`，无告警（含 `g_clear_pointer(g_ptr_array_unref)`、`g_ptr_array_remove_index`、`g_ptr_array_sort` 的比较器、`g_autoptr(FpPrint)`、`g_signal_emit` 参数个数） |
| **完整编译 `device.c` / 整个 daemon** | ❌ 未做 | 设备上没有 `gdbus-codegen`/`glib-mkenums`（`libglib2.0-dev` 系包在此发行版被拆成空壳）与 polkit/dbus dev 头，无法现场生成 `fprintd-dbus.h`；需要联网装 dev 包（约 10 分钟） |
| 端到端解锁 | ❌ 未做（需人按压 + 安装新 daemon） | 按任务要求不动设备 |

> 已修掉的一个真实编译坑：初版用 `g_set_object (&priv->verify_data, next)`，`FpPrint **` → `GObject **` 属 `-Wincompatible-pointer-types`，**GCC 14+ 默认当错误**；改为 `g_clear_object(); priv->verify_data = next;` 后通过（日志见上）。

### 4.8 方案 B 风险

| 风险 | 等级 | 说明/缓解 |
|---|---|---|
| 中间状态被客户端误判为终结 | 中 | 已按 `pam_fprintd.c:277-281` + `device.c:1274-1291` 两条约束设计；非终结状态不发 `verify-no-match` |
| `timeout=15` 导致 sudo 失败 | **高** | 必须调 PAM（§4.5） |
| 每次换库要重新训练用户"再按一次" | 中 | 已发 `VerifyFingerSelected` + `verify-retry-scan`，PAM 会提示具体手指 |
| 手指一直压在传感器上不抬起 | 中 | `wait_finger_down` 会先 `clear_pending_irq` 再等新的 down 沿（`fpc_build_info.c:703-713`），必须"抬起再按" |
| 自适应模板更新写错文件 | 低 | OEM hook 在 `match_cb` 里按 `match` 定位文件（`oem-update.inc:38-40`），每个 print 各自独立 |
| 上游 fprintd 升级 | 低 | 锚点校验会在升级时报错，而不是静默错改 |

---

## 5. 方案 C：驱动内部多库（评估 → 不采用）

### 5.1 它无法达成目标【已确认】

"每根手指一个文件"的**收益全部来自 fprintd 的 host storage**：

- `ListEnrolledFingers` 只列 host storage 的 print：`device.c:2318` `prints = load_user_prints (rdev, user);`，**:2321-2327** 逐个 `fp_print_get_finger()`。
- 删除也只删 host storage：`device.c:2437` `store.print_data_delete (priv->dev, finger, user)`；device storage 只是"顺带尝试"（`:2395-2433`，且仅当 `FP_DEVICE_FEATURE_STORAGE`）。
- `verify_start` 的 gallery 同样来自 `load_user_prints`（`:1677`）。

所以"驱动内部多库、fprintd 只看到一个 print"= GNOME 只显示 1 条、只能整库删——**和现状一模一样**，只是把复杂度搬进了驱动。C 不能替代 A/B。

### 5.2 "把模板做成 device-stored print"（`list`/`delete` vfunc）为什么也不行

即使实现 `list`/`delete`/`clear_storage` 让 fprintd 认为模板在设备里，UI 与删除仍走 host storage（上一节三处引用），并且会触发 `check_local_storage()` 的"设备/主机模板对账"逻辑（`device.c:1294-1360`）把不存在的 host 模板**删掉**——风险高、收益 0。**不采用。**

### 5.3 C 唯一有价值的变体

把 C 当作**方案 A2 的实现载体**：驱动在一次 `identify()` 里"采集一次 → 循环 import+identify 每个库 → 上报命中的 print"。这正是 §3.4 的 A2，属于 A 的增强，不是独立方案（且同样受 §3.6 录入副作用影响）。

---

## 6. 配套改造：`native_enrol.py`（多库方案的必做项）

现状（`native_enrol.py:160-170`，见 §1.3）在"多于 1 个模板文件"时直接拒绝。多库改造建议：

```python
# 目标形态（示意）：不再"一个账号一个库"，而是"一次录入一个独立库"
prints = FPRINT_ROOT / username / "fpc1264_oem" / "liuqin-fpc1264-oem"
requested = <本次录入的手指号，由驱动通过 argv 传入>      # 新增参数
stored = [f for f in "123456789a" if os.path.lexists(prints / f)]

# 方案①（最简单）：每次录入都新建单模板库，永不追加
existing = None
canonical_finger = 0

# 方案②（保留"同一根手指追加多个模板"）：只扩展同一根手指的文件
# existing = prints / requested if requested in stored else None
# canonical_finger = int(requested, 16) if existing else 0
```

配套改动：
- **驱动 verify 路径不用改**：round2 驱动已经调用 `--match-existing`（`work/fpc1264-oem.c:568`），而它接受任意模板数的库；本报告 §3.5 的实测里单模板 raw DB 也正常走完 `db_import=OK` → `loaded_template_count=1` → arm IRQ。所以"每根手指一个单模板库"对验证侧是**零改动**（`--match-single` 那条更严格的路径不要用）。
- 驱动 `work/fpc1264-oem.c:170-180` 里"把手指号改回原来那根"的 `fp_print_set_finger()` **可以删掉**（多库下返回什么手指就存什么文件）；`enrol_canonical_finger` 机制随之退役。
- `native_enrol.py` 需要知道"本次录入的手指号"，因此驱动 `enrol()` 的 `g_subprocess_launcher_spawn(...)` 参数要加一个 finger（`work/fpc1264-oem.c:352-353`）。
- 走 `--enrol`（无 base，全新库）路径时，`oem_runtime.py` 的 `--enrol DATABASE UID HANDLE DERIVED` 分支已存在（`oem_runtime.py:32`），不用改运行时。
- 【待验证】"在**已有凭据 + 已有另一个模板文件**的情况下，用全新库录入第二根手指"是否被 TA/授权流程接受。这是整个多库方案的第一道门槛（见 §10 的 B0）。

---

## 7. 工作量 / 风险对比 + 推荐

| | B（fprintd 补丁） | A1（驱动 identify，串行） | A2（驱动 identify + 客户端一次采集多库） |
|---|---|---|---|
| 代码量 | fprintd ~50 行（已写好）+ `native_enrol.py` ~20 行 | 驱动 ~200–250 行 + `native_enrol.py` ~20 行 | A1 + `fpc_build_info.c` ~80–120 行 + TA 语义验证 |
| 按压次数（2 根） | 最坏 2 次 | 最坏 2 次 | 1 次 |
| 单次按压开销 | 6–10 s | 6–10 s | ~6–10 s（含所有库，摊薄） |
| 录入副作用 | 无 | **每次录入多一次 identify 按压** | 同 A1 |
| 需要动 libfprint/TOD 头 | 否 | 否（tod-1 已支持） | 否 |
| 失败模式 | 状态机（可控） | 驱动动作状态机 + fprintd 契约（较难） | 还要赌 TA 行为 |
| 估计工时 | **0.5–1 天**（含构建验证） | 1.5–2 天 | +2–3 天，且可能推翻 |
| 风险 | 中 | 中高 | 高 |
| 结论 | ✅ **推荐** | 仅作备选 | 先做 A0 实验再定 |

**推荐：走 B。** 理由：
1. 目标（UI 可见 + 单独删除）**只由 host storage 决定**，而 B 正好只改这一层；
2. 代码量最小、可回滚（换回旧二进制即可）；
3. 不改变设备的 feature 集合，因此**不影响录入流程**（A 会）；
4. 只有在 A0 证明"一次采集可多库匹配"后，A2 才值得作为体验增强叠加在 B 之上（那时 B 的队列逻辑可以保留不用，两者不冲突：有 IDENTIFY 时 fprintd 根本不进 verify 路径）。

---

## 8. 落地步骤（建议顺序，每步都可独立验证/回滚）

**第 0 步：全量备份（必须）**
```sh
tar -C /var/lib/fprint -czf /root/fpc-backup-2026-10-27/fprint-var-multidb-$(date +%F).tar.gz .
cp -p /usr/local/lib/liuqin-fpc-oem/fprintd /root/fpc-backup-2026-10-27/binaries/fprintd.pre-multidb
cp -p /usr/local/lib/liuqin-fpc-oem/native_enrol.py /root/fpc-backup-2026-10-27/pre-install/native_enrol.py.pre-multidb
cp -p /usr/local/lib/liuqin-fpc-oem/tod/libfprint-tod-fpc1264-oem.so /root/fpc-backup-2026-10-27/pre-install/libfprint-tod-fpc1264-oem.so.pre-multidb
```

**第 1 步（先做，风险最低）：改 `native_enrol.py`，验证"第二根手指能录成独立文件"**
- 改哪个函数：`native_enrol.py:enrol()`，即 `:160-170` 的 `stored / ambiguous_templates / existing / canonical_finger` 段；同时给 `main()` 加 finger 参数、驱动 `enrol()` 传参。
- 验收（不需要新 fprintd）：录第二根 → `ls /var/lib/fprint/<user>/fpc1264_oem/liuqin-fpc1264-oem/` 应出现**两个**文件（如 `2` 与 `7`），各 772 KB 左右；`fpc-oem-extract` 抽出的 raw DB 用 `--template-inventory` 应显示 `loaded_template_count=1`。
- 此时现象：GNOME 显示 2 条、第二根**还不能解锁**（预期，正是 §1.1 的坑）。

**第 2 步：打 fprintd 补丁并私有安装**（§4.6），重启 fprintd。

**第 3 步：逐指验收**
```sh
python3 /usr/local/lib/liuqin-fpc-oem/deployment/verify_once.py <user> left-index-finger  # 期望 verify-match
python3 /usr/local/lib/liuqin-fpc-oem/deployment/verify_once.py <user> right-index-finger # 期望 verify-match
journalctl -u fprintd -f   # 观察 "next stored print queued for this verification: right-index-finger"
                           # 以及非终结 VerifyStatus=verify-retry-scan 后紧跟 verify-match
```
再用 `sudo -k; sudo true` 验证 PAM 路径（**先把 GNOME 设置页关掉**，见多指文档 §7.2 设备独占坑）。

**第 4 步：调 PAM 预算**（§4.5）：`/etc/pam.d/common-auth` 那行 `timeout=15` → `timeout=40`（`pam-auth-update` 管理的文件，改完注意后续升级会覆盖，建议同时写 `/usr/share/pam-configs/` 条目或做本地记录）。

**第 5 步（可选，体验增强）**：A0 实验（§10）成功后再做 A2。

---

## 9. 回滚方式

| 场景 | 操作 |
|---|---|
| 只回滚 fprintd（保留多文件模板） | `cp -p /root/fpc-backup-2026-10-27/binaries/fprintd.pre-multidb /usr/local/lib/liuqin-fpc-oem/fprintd && systemctl restart fprintd`（此时只有第一根手指能解锁，但仍能录/删） |
| 回滚 `native_enrol.py` | `cp -p /root/fpc-backup-2026-10-27/pre-install/native_enrol.py.pre-multidb /usr/local/lib/liuqin-fpc-oem/native_enrol.py` |
| 回滚模板到"单库多指"状态 | `rm -rf /var/lib/fprint/<user>/fpc1264_oem/liuqin-fpc1264-oem && tar -C /var/lib/fprint -xzf /root/fpc-backup-2026-10-27/fprint-var-multidb-<date>.tar.gz && systemctl restart fprintd`（⚠️ 会丢掉备份之后录的手指） |
| 整体回滚到 round2 之前 | `sh /root/fpc-backup-2026-10-27/round2/rollback-round2.sh`（会丢弃后录手指） |
| 驱动不用回滚 | 方案 B 完全不改驱动 `.so` |

---

## 10. 待验证清单（按优先级）

**B0（最高优先，决定多库方案是否成立）**：在已有凭据 + 已有另一个模板文件的前提下，用**全新库**录入第二根手指是否成功。
- 方法：改 `native_enrol.py` 为"永远 `existing=None`"→ GNOME 里录右食指 → 看 journal 是否 `native_enrol=OK`（无 `REFUSED`/`status=1`），并检查新文件 772 KB 左右。
- 证据点：`native_enrol.py:252-256` 的两条分支；`oem_runtime.py:32` 的 `--enrol` 参数形状。

**B1**：补丁版 daemon 的真实编译 + 端到端解锁（§4.7 未做项）。构建依赖准备约 10 分钟。

**A0（决定 A2 是否值得）**：TA 是否允许"一次采集、多库匹配"。
- 方法（**只读，不需要按压**）：给 `fpc_build_info.c` 加一个临时子命令 `--import-inventory DB1 DB2 …`：`BIO_INIT → SET_GID → load_database(DB1) → load_database(DB2) → template_inventory`，看 `loaded_template_count` 是 1 还是 2（append 语义 vs replace 语义）。
- 若 =2（可 append）：再验证"先 `CAPTURE_ENROL` 再多次 (import+identify)"是否被接受 → 这是 A2 的核心。
- 若 =1（replace）：A2 直接判死，方案 A 只能停留在 A1（无收益）。

**B2**：多库下 `BIO_UPDATE_TEMPLATE` 自适应更新是否只影响被匹配的那个库文件（预期是；`fpc_build_info.c:1037-1053` 只导出当前会话导入的那个 DB）。

**B3**：3 根以上手指时的按压次数/超时实测（`/etc/pam.d/gdm-fingerprint` 无 timeout，风险集中在 sudo）。

---

## 11. 证据索引（可复现）

设备侧只读探测（全部通过 `python _vm/fpb-run.py <script> <timeout>` 执行，脚本名见下）：

| 结论 | 命令/输出摘要 | 脚本 |
|---|---|---|
| 版本、TOD shim 是独立 .so、sysroot 有 tod-1 头 | `dpkg -l`；`ls -l libfprint-2*`；`ls /root/fpcbuild/sysroot/usr/include/libfprint-2/tod-1/` | `mdb-01-facts.sh` |
| shim 导出 273 个 `fpi_*`，含 identify 三件套，版本 `LIBFPRINT_TOD_1.0.0`；真 libfprint 导出 0 个 `fpi_*` | `nm -D --defined-only …` | `mdb-02-abi.sh` |
| `FpDeviceClass` 里 `identify` 位于 tod-v1 冻结区（`fpi-device.h:182`） | `awk '/^struct _FpDeviceClass/,/^};/'` | `mdb-06-hdrdump.sh` |
| **TOD 模块可实现 identify、feature 自动置位、gallery 可拿到、report 语义** | 自建模块 + harness：`features=0x6 identify=1` / `gallery=2 fingers=2,7` / `match_finger=7` / 不 report 时报 "did not report the result earlier" | `mdb-11-identify-test.sh` |
| PAM 实际预算（`max-tries=2 timeout=15` / gdm 无 timeout） | `grep -rn fprintd /etc/pam.d/`；`cat /etc/pam.d/gdm-fingerprint` | `mdb-13-pam.sh` |
| **单库验证 pre-press = 3.70 s；DB 导入 3.29 s / 772,495 B / 190 片** | 逐行时间戳 + `oem_runtime_cleanup … restored=1 power_off=1` | `mdb-18-timing3.sh` |
| 实验后设备状态无残留 | `fwpath` 空 / `power_state=off` / 无残留进程 / 模板 1,162,014 B 未变 | `mdb-17-state.sh` |
| 补丁在真源码上命中 + 与 OEM 补丁叠加 + 惯用法编译通过 | `python3 fprintd-multidb.py`、`prepare.py`、`cc -fsyntax-only … idiom_rc=0` | `mdb-20-push.sh` / `mdb-26-idiom3.sh` |

设备上留下的实验产物（**都在 `/tmp`，未安装任何东西**）：`/tmp/mdb/{test-tod.c,harness,idiom.c,fp,fp-multi,fp-oem,tod/libfprint-tod-mdb-test.so}`、`/tmp/mdb-debs/*.deb`、`/tmp/mdb-pol/`、`/tmp/mdb-glbin*/`。重启即消失。

本机产物：`work/fprintd-multidb.py`（补丁生成器）、`work/fprintd-multidb.patch`、`work/fprintd-oem-on-multidb.patch`、`work/multidb-test-oem/`（叠加后的完整 `device.c`）、`work/fprintd-src/fprintd-v1.94.5/`（上游源码）。

---

## 12. 一句话回答"如果做，第一步改哪个文件/哪个函数"

> **`/usr/local/lib/liuqin-fpc-oem/native_enrol.py` 的 `enrol()`（第 160–170 行那段 `stored / ambiguous_templates / existing / canonical_finger`）**——
> 先把"一个账号一个库"改成"一次录入一个独立单模板库"，这是所有多库方案的前置门槛，且**不需要**改 fprintd 就能单独验证（B0）。
> 通过之后，第二步改 `fprintd 1.94.5` 的 **`src/device.c: fprint_device_verify_start()` + `verify_cb()` + `match_cb()`**（补丁已备好：`work/fprintd-multidb.py`）。
