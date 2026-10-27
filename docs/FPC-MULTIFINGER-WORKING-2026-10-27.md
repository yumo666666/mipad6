# 指纹多指方案（已可用）：一个 OEM 库装多根手指

> 状态：**已实测可用**（2026-10-27 17:20）。左食指 + 右食指都能解锁。
> 局限：GNOME 里只显示一条、不能单独删除某一根。见文末"已知限制"。

---

## 1. 结论先说

| 项目 | 状态 |
|---|---|
| 能录第二根吗 | ✅ 能（TA 固件支持 **5 个槽位**） |
| 两根都能解锁吗 | ✅ 都能（实测通过） |
| GNOME 设置里显示几根 | ❌ 只有 1 条（`left-index-finger`） |
| 能单独删掉某一根吗 | ❌ 只能整库删除 |
| 最多几根 | 5 根（TA 固件的 `fpc_db_add_fingerprint` 上限） |

---

## 2. 为什么不能"每根手指一个文件"

这是整个问题的核心，走错一步就会得到"能录第二根、但只有第一根能解锁"的假成功。

`fprintd 1.94.5` 的 `device.c: verify_start()`：

```c
/* 当 gallery 里有多份模板，而设备不支持 IDENTIFY 时 */
只取 gallery[0] 去 VerifyStart
verify_cb 重试时也只重试同一份模板
```

而 `pam_fprintd` 默认只给 3 次机会。所以如果按"每根手指一个模板文件"来做：

- 录第二根 → 文件 `7` 被创建 ✅
- 解锁时 fprintd 只把**第一个**模板（`2`，左食指）交给驱动 ❌
- 结果：右食指按多少次都不匹配

**另一个被排除的怀疑**：Gatekeeper 凭据不是瓶颈。凭据只在 enroll 授权时使用（`authorize_oem.py --verify-enrol`），verify 路径（`oem_runtime --match-* <db>`）完全不碰凭据。

---

## 3. 采用的方案：一个库 + 改写手指号

### 3.1 原理

```
用户录右食指
  → libfprint TOD 驱动捕获 24 个样本
  → native_enrol.py：从 FP3 容器里抽出 raw OEM DB（fpc-oem-extract）
  → oem_runtime.py --enrol-existing <raw-db>   ← 在已有库上追加
  → TA 把新模板写进第 2 个槽位
  → 驱动用 fp_print_set_finger() 把返回的 print 手指号改回「库里原来那根」（2）
  → fprintd 认为还是同一个 print → 覆盖同一个模板文件
```

这样磁盘上**永远只有一份库**，fprintd 的 `gallery` 永远只有 1 项，绕开了上面那个 `gallery[0]` 的坑。

### 3.2 换掉的关键参数

| 原路径 | 新路径 | 说明 |
|---|---|---|
| `--bio-pipeline-single` | `--bio-pipeline-existing` | 追加而不是新建 |
| `--enrol-single` | `--enrol-existing` | 同上 |
| `--match-single` | `--match-existing` | 匹配库内任意模板；对单模板库与原路径等价 |

### 3.3 之前失败的原因（重要教训）

第一次尝试时 `status=1`，不是 TA 拒绝追加，而是**喂错了文件格式**：

- 传进去的是 **FP3 封装**的 `FpPrint` 文件（772,578 B）
- TA 只接受 **raw DB**（`fpi-data`，772,495 B）

复现证据（用未修改的官方二进制）：

```
TA 接受前 189 个分片
最后一个 1458 B 分片返回 0xe853009f
客户端: db_import_remaining_mismatch expected=0 actual=-397213537
→ 退出码 1 → journal 的 status=1
```

修复：新增 `fpc-oem-extract` 工具，从 FP3 容器里抽出 `fpi-data` 再传给 TA。

---

## 4. 文件与哈希（round2，2026-10-27 17:17 安装）

| 文件 | SHA256 |
|---|---|
| `/usr/local/lib/liuqin-fpc-oem/fpc-oem-extract` | `c74de4f0ea983f64…` |
| `/usr/local/lib/liuqin-fpc-oem/source/fpc-oem-extract.c` | `730d8c4aecdfe5bf…` |
| `/usr/local/lib/liuqin-fpc-oem/native_enrol.py` | `584a5c500215f328…` |
| `/usr/local/lib/liuqin-fpc-oem/tod/libfprint-tod-fpc1264-oem.so` | `e2195af1d70886c8…` |

驱动加载日志：

```
fpc1264_oem TOD driver build=20261027-round2 database=per-account extract=1
```

**原版备份（round2 之前）**：`/root/fpc-backup-2026-10-27/round2/`
- `.so` = `e128383a…`，provider = `d2a222d8…`
- 回滚脚本 `rollback-round2.sh` ⚠️ **会删掉新录的手指**

---

## 5. 成功证据

```
17:20:12  OEM enroll diagnostic: enrol_challenge_available=1
17:20:13  OEM enroll diagnostic: db_export=OK bytes=1161931 biometric_validation=unverified
17:20:13  OEM enroll database extended under stored finger 2 (requested finger 7)
17:20:31  OEM verify result=matched
17:20:46  OEM verify result=matched        ← 两根各匹配一次
```

模板文件：

```
772,578 B  →  1,162,014 B        （多了一根手指的模板）
```

---

## 6. 已知限制

1. **GNOME 只显示一条**：fprintd 只知道一个 print，界面固定显示最初那根（左食指）。两根都能用，但界面上看不出来。
2. **不能单独删除**：删指纹 = 删整个库（两根一起）。安全上更干净，但不方便。
3. **上限 5 根**：TA 固件 `fpc_db_add_fingerprint` 分配 16B 头 + 5×32B 槽，两处 `cmp ..., #5`，满槽返回 `0xe56004c4`。
4. **重复录同一根会累积浪费槽位**（可能写入重复模板）。
5. **`--bio-pipeline-existing` 的边界**：真手指验证只到"两根能解锁"，更多根没测过。

---

## 7. 两个坑（血泪）

### 7.1 模板哈希每次解锁都会变

**不要用模板哈希判断"是否被改动"。** verify 匹配成功时，OEM fprintd 补丁会调用
`file_storage_print_data_update()` 做自适应更新（原子 rename，同大小、不同字节）。
每次成功解锁都会写一次。

### 7.2 GNOME 设置页会独占指纹设备

打开"设置 → 用户 → 指纹登录"页面时，gnome-control-center 会长期 **Claim** 指纹设备。
副作用：

- `sudo` 的 `pam_fprintd` 拿不到设备 → `failed to claim device: Device was already claimed` → 静默回落到密码
- 锁屏/登录的指纹也可能抢不到

**结论**：用 sudo 指纹前，把设置窗口关掉。

---

## 8. 回滚

```sh
# 只回滚代码（保留当前库，包括后录的手指）
sh /root/fpc-backup-2026-10-27/round2/rollback-round2.sh

# 完整回滚（连模板一起恢复成单指时代的备份）
sh /root/fpc-backup-2026-10-27/rollback.sh full
```

⚠️ 两者都会让**后录的手指失效**；第二个还会把库恢复成备份时刻的内容。

---

## 9. 下一步（多库方案研究）

当前方案的代价是"界面上看不出来、不能单独删"。如果能做到**每根手指一个库文件**并让 fprintd 逐个尝试，体验会更好。

两条可能的路（见 `FPC-MULTIDB-RESEARCH-*.md`）：

1. **给驱动实现 `identify` vfunc**：fprintd 在 gallery>1 时会走 IDENTIFY 路径，由驱动负责"拿一次图像、比完所有库"；
2. **给 fprintd 打补丁**：项目的 fprintd 本来就是自己编译的（`source/fprintd-vendor.tar.gz`），改 `verify_start()` 让它遍历所有 print，而不是只取 `gallery[0]`。
