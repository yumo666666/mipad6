# FPC1264（Xiaomi Pad 6 Pro / liuqin）TEE 录入协议逆向报告

- 日期：2026-10-27
- 设备：Xiaomi Pad 6 Pro（liuqin），Ubuntu 26.04，内核 `6.17.0-rc1-gfe81794b5e1b`
- 对象：`/usr/local/lib/liuqin-fpc-oem/` 这套 OEM 指纹栈（TOD 驱动 + native 提供者 + 私有 fprintd + FPC TA `fpcliu`）
- 目标：查清"在已有数据库上追加手指"为什么 `status=1`，逆向 TEE 录入协议，判断多指可行性
- 证据分级：**【已确认】**＝有源码/反汇编/实测输出直接支撑；**【推测】**＝由多条间接证据推出；**【待验证】**＝需要真手指或额外实验

---

## 0. 三分钟结论

1. **`status=1` 的根因是"喂错了文件格式"，不是 TA 拒绝追加。**【已确认】
   改版 `native_enrol.py` 把 `/var/lib/fprint/.../2`（**FP3 封装的 FpPrint 序列化文件**，772,578 B）
   原样当作"OEM 数据库"传给 `--bio-pipeline-existing`。TA 的 `load_database` 只接受
   **raw OEM DB**（772,495 B，从 `fpi-data` 里取出来的那一段）。TA 在导入的最后一个分片做校验，
   返回 `0xe853009f`，客户端打印 `db_import_remaining_mismatch expected=0 actual=-397213537` 后退出 1。
   整条链上这个具体错误被**两层白名单**吞掉，journal 里只剩 `status=1`。

2. **"失败过程改写了模板"是误判。**【已确认】
   写 `/var/lib/fprint/.../2` 的只有一处：**验证成功后**的 adaptive template update
   （OEM fprintd 补丁 `match_cb` → `file_storage_print_data_update`，原子 rename）。
   失败的 enroll 在采集前就退出了，不产生 print，fprintd 的 `print_data_save` 根本不会执行。
   时间线也吻合：16:54:36 `OEM verify result=matched` 之后哈希变了；失败 enroll 在 16:54:55；
   16:57:47 用户指纹解锁成功又写了一次。**当前模板已经不是 `2a22b83c…`，而是 `b63bb413…`**
   （mtime 16:57:47），`2a22b83c…` 只存在于备份里。

3. **多指：能，上限 5 根，但还差"最后一公里"的真手指验证。**【已确认 + 待验证】
   - TA 反汇编里指纹集合就是 **16 字节头 + 5 × 32 字节槽**（`fpc_db_add_fingerprint` 分配 `0xB0`，
     两处 `cmp x8,#5`），满 5 根返回 `0xe56004c4`。
   - 用**正确的 raw DB** 跑 `--bio-pipeline-existing`，已实测走到
     `BIO_LOAD_INSTANCE_DATA → 3/2 挑战 → Gatekeeper 签名 → 3/3 → BIO_BEGIN_ENROL → READY 等按压`，
     全程 TA 状态 0。**采集之后**的 `BIO_ENROL / BIO_END_ENROL / DB 导出`是唯一没验证的环节。
   - 最小修复：让 provider 先把 FP3 容器里的 `fpi-data` 抽出来（约 60 行 C 辅助程序，复用
     `fpc-oem-print.c` 的私有 FD 模式），再调 `--enrol-existing`。

4. **验证路径不受影响是设计使然。**【已确认】改版驱动用 `--match-existing`（要求 `count>0`）；
   TA 的 `IDENTIFY(11/4)` 是在**已加载的整个 DB** 上做 1:N，所以一个 DB 里 N 根手指都能匹配；
   fprintd 侧始终只看到**一个 print、一个文件**，不触发"多 print 只验第一个"的坑。

---

## 1. 现场基线

### 1.1 实验开始/结束时的系统状态（均未改动）

| 项目 | 值 | 说明 |
|---|---|---|
| 驱动 `.so` | `e128383ae639890f…`（原版） | `strings` 仍含 `Only one enrolled finger per user is supported` / `--match-single` |
| `native_enrol.py` | `d2a222d8038de0ab…`（原版） | 见 §1.2 |
| `fprintd` | 1.94.5-4，bundle 内 `463c8b79…`，**带 OEM 补丁** | `strings` 命中 `OEM verify storage_error phase=adaptive-template-update` |
| `libfprint` | `1:1.95.1+tod1-0ubuntu1`（dpkg 原包） | 未替换 |
| 模板 | `b63bb413574de32d…`，772,578 B，mtime `16:57:47` | **不是备份里的 `2a22b83c…`** |
| 凭据 | `credential.json` / `gatekeeper.handle` mtime `16:10:02` | 实验全程只读 |
| 传感器 | `power_state=off`，`firmware_class/path` 为空 | 每轮实验后都确认 |
| 服务 | `fprintd` active | — |

实验前后模板哈希、驱动/提供者哈希、凭据 mtime **全部一致**（见 §9 的收尾校验）。

### 1.2 改版 `native_enrol.py` 的错误假设

`<workdir>\work\native_enrol.py`（= 设备上 `post-install/native_enrol.py.multifinger`，
sha256 `6fa3876a8e927397…`, 15,845 B）在追加分支里：

```python
# work/native_enrol.py:238-254
with tempfile.TemporaryDirectory(prefix=".native-enrol-", dir=output.parent) as temporary:
    database = Path(temporary) / "database"
    base = None
    if existing is not None:
        base = Path(temporary) / "base-database"
        shutil.copyfile(existing, base)          # ← 这里复制的是 FP3 文件，不是 raw DB
        os.chmod(base, 0o600)
        ...
    if base is not None:
        status = runtime.run([*command, "--enrol-existing", str(base), str(database),
                              str(uid), str(secret_fd)], secret_fd)
```

`existing` 是 `FPRINT_ROOT/<user>/fpc1264_oem/liuqin-fpc1264-oem/<finger>`，即 fprintd 的
**存储文件**。这个文件是 `fp_print_serialize()` 的产物（magic `FP3`），`fpi-data` 才是 raw DB。
两者差 83 字节包装，但对 TA 来说是**完全不同的字节流**。

同一个仓库里其实已经写清楚了：
- `fpc1264-oem.c:478-537` 的 `verify()` 从 print 的 `fpi-data` 里取数据库、写成临时文件再交给 runtime；
- `fpc-oem-print.c` 是把 raw DB **封装成** FP3 的单向工具，没有反向工具。

改版 provider 把"存储文件"当成"数据库"，是把两层搞混了。

### 1.3 时间线（journal，`journalctl -b`）

| 时间 | 事件 | 含义 |
|---|---|---|
| 16:54:34 | `fpc1264_oem TOD driver build=20261027-multifinger` | 当时是改版驱动 |
| 16:54:36 | `OEM verify result=matched` | **adaptive update 写模板**（推测为 `ec9bfa2b…`） |
| 16:54:54 | `does not support duplicate identification…` | 开始录入 |
| 16:54:55 | `native_enrol_runtime=FAILED operation=enrol status=1 cleanup_complete=1` | 失败（采集前） |
| 16:55:40-41 | 同上，第二次失败 | 可重复 |
| 16:56:23-41 | 多次 stop/start fprintd | 回滚（`rollback.sh code` + restore） |
| 16:57:43-46 | 屏幕 resume；`gdm-fingerprint`；`qcom_qseecom_tee: loaded 'fpcliu'` | 真实解锁 |
| 16:57:47 | `OEM verify result=matched` | **又一次 adaptive update → 当前 `b63bb413…`** |
| 17:00-17:08 | 本报告的全部实验 | 模板 mtime 仍是 16:57:47，未被触碰 |

---

## 2. `status=1` 到底从哪来

### 2.1 退出码链：`status` 是进程退出码，不是 TA 状态

```
fpc1264-oem.c:143   status = g_subprocess_get_exit_status()        # native_enrol.py 的退出码
  native_enrol.py:229/231  status = runtime.run(...)               # oem_runtime.py 的退出码
    oem_runtime.py:162       result = client_process.wait()        # authorize_oem.py 的退出码
      authorize_oem.py:92-94 client_status = client.wait()         # fpc_build_info 的退出码
        fpc_build_info.c:1347  result = store_database(...) ? 1 : 0
```

所以 journal 里的 `status=1` = **`fpc_build_info` 退出 1**，与 FPC 状态码 `1` 无关
（`fpc_build_info.c:149-150` 的 `1` 是 `FPC_STATUS_WAIT_TIME`，别混）。

### 2.2 复现实验 A：把 FP3 文件当数据库喂进去（与改版 provider 完全同构）

在 device 上（细节见 §8）：

```
python3 native_credentials.py --state-dir <state> --enrol-existing \
        /tmp/fp-re/tpl-fpchk.bin  /tmp/fp-re/out-fp3.db  <native_uid>  <secret_fd>
```

`tpl-fpchk.bin` 就是备份里的原始模板（`2a22b83c…`, 772,578 B）的拷贝。输出（节选，**真实**）：

```
command=4 subcommand=14 db_word3=1 db_word4=772578        ← 把 772578 字节整个当 DB 导入
tee_ret=0 origin=0 response_status=0 status_hex=00000000
...（190 个分片：前 189 片各 4080 字节，最后 1458 字节；response_status 依次递减，全部接受）...
command=4 subcommand=17 db_word3=1458 db_word4=0
tee_ret=0 origin=0 response_status=-397213537 status_hex=e853009f   ← 最后一片校验失败
db_import_remaining_mismatch expected=0 actual=-397213537
command=4 subcommand=15 db_word3=0 db_word4=0             ← DB_CLOSE
command=10 subcommand=5                                    ← SENSOR_SLEEP
gatekeeper_input_incomplete=1
oem_runtime_cleanup firmware_path_restored=1 sensor_power_off=1 listener_stopped=1
```

- `native_credentials` 返回 1 → `oem_runtime` 返回 1 → provider 打印
  `native_enrol_runtime=FAILED operation=enrol status=1 cleanup_complete=1`。
- 与 journal 的 3 行**逐字对应**。
- `gatekeeper_input_incomplete=1` 是下游副作用：Gatekeeper 提供者先启动，等客户端写挑战，
  客户端在导入阶段就退出，管道 EOF，它才打印这句——**不是**根因。

### 2.3 为什么 journal 里看不到 `db_import_remaining_mismatch`

两层白名单都在丢信息：

1. `native_enrol.py:27-30` 的 `FORWARD` 只转发
   `READY / enrol_progress_status / capture_rejected / oem_runtime_cleanup /
   native_credential_provider=FAILED|BUSY / pipeline_incomplete / end_enrol_not_submitted /
   enrol_authorization / oem_runtime_timed_out|cancelled`；
2. `fpc1264-oem.c:188-192` 的 `failures[]` 只转
   `native_enrol_runtime=FAILED / native_enrol=FAILED / pipeline_incomplete= /
   end_enrol_not_submitted= / enrol_authorization=TA_rejected`。

`db_import_*`、`tee_ret=`、`response_status=`、`gatekeeper_*` 全被丢。于是 journal 里只剩
提供者的摘要行。**这是本次排障成本高的直接原因**，建议后续把
`db_import_` / `db_export_` / `enrol_challenge_` / `hat_` 加进两层白名单（只放状态行，不放模板字节）。

### 2.4 复现实验 B：换成正确的 raw DB，同一调用链直接走到"等按压"

`raw-fpchk.db` = 从 `tpl-fpchk.bin` 的 `fpi-data` 里抽出的 772,495 字节（sha256 `2adafd8740…`）。
同一条 `--enrol-existing` 链路输出（节选，**真实**）：

```
command=4 subcommand=14 db_word3=1 db_word4=772495
...（分片全部接受）...
command=4 subcommand=15 db_word3=0 db_word4=0
db_import=OK path=/tmp/fp-re/raw-fpchk.db biometric_validation=unverified
command=11 subcommand=10                      ← BIO_SET_ACTIVE_GROUP      status 0
command=11 subcommand=19                      ← BIO_LOAD_INSTANCE_DATA    status 0
keymaster_api=4.1 ta_version=4.1012
keymaster_command=0x207 app_status=0
keymaster_command=0x21002 app_status=0
gatekeeper_verified_hat_forwarded=1 round=1 rounds=2 bytes=69
command=3 subcommand=3  → 0                   ← AUTHORIZE_ENROL(HAT)
enrol_authorization=OK
command=11 subcommand=1 → 0                   ← BIO_BEGIN_ENROL
READY enrol attempt=1: press same finger and hold
stale_irq_drained=before_down
command=10 subcommand=3 → 0
READY finger_irq_armed=down press finger now
```

**结论【已确认】：换正确的字节流后，"加载已有库 + 设置 GID/组 + 加载实例数据 + 挑战签名 +
BEGIN_ENROL"全部成功，失败点确实只在"输入文件不是 raw DB"。**

### 2.5 同一份 raw DB 的模板计数

单独跑 `--match-existing`（不按手指，10 s 后主动终止）：

```
db_import=OK path=/tmp/fp-re/raw-fpchk.db
command=11 subcommand=10 → 0
command=11 subcommand=19 → 0
template_enumerate_app_status=0x00000000
loaded_template_count=1
authenticator_id_app_status=0x00000000
authenticator_id_nonzero=1
template_inventory_scope=current_loaded_TA_DB persistent_storage_not_searched=1
READY finger_irq_armed=down press finger now
```

`loaded_template_count=1` 是 TA 自己数的（`11/8`），不是从文件里猜的。

---

## 3. OEM 客户端 ↔ TA 协议

### 3.1 TEE 传输层（`fpc_build_info.c`）

| 环节 | 实现 | 证据 |
|---|---|---|
| 打开设备 | 遍历 `/dev/teepriv%d`（特权）与 `/dev/tee%d`，用 `TEE_IOC_VERSION` 检查 `impl_id == 5`（QSEE） | `fpc_build_info.c:44-63` |
| 打开会话 | **按名字**打开 `"fpcliu"`（不是 UUID）：`TEE_IOC_OPEN_SESSION`，1 个 memref input 传 app name | `:85-117`（`APP_NAME` 在 `:22`） |
| 共享内存 | `TEE_IOC_SHM_ALLOC` + `mmap`，`request=64 B`、`response=64 B`、`payload=4096 B` | `:65-83`, `:24-27` |
| 调用 | `TEE_IOC_INVOKE`，固定 **4 个 param**：`[0]` input 64B request、`[1]` output 64B response、`[2]` value input `a=4,b=4`、`[3]` inout payload | `:177-195` |
| 请求头 | `request[0] = payload.size`（u32，LE） | `:180` |
| 响应 | `response[0]` = TA 的 int32 应用状态；`tee_ret/ret_origin` 是 TEE 传输层结果 | `:195-205` |
| payload 布局 | `words[0]=模块/命令类`，`words[1]=子命令`，`words[2..3]=参数`，数据区在 **+16** | `:471-565` |

实测形态：`tee_ret=0 origin=0` 表示 TEE 传输 OK；真正的业务结果看 `response_status`。
**注意**：`4/16`、`4/17` 这两个分片命令把"剩余字节数"放在应用状态里返回（正数），
所以看到 `response_status=768498` 不是错误。

### 3.2 两个会话与 TA 加载

`main()` 先 `open_qsee_device(1)`（`/dev/teepriv0`）打开 `fpcliu`，再 `open_qsee_device(0)`
（`/dev/tee0`）打开同一 TA（`:1182-1194`）。特权会话负责触发 QSEE 从
`/sys/module/firmware_class/parameters/path` 指向的目录加载 TA 镜像；
`oem_runtime.py:70-75,137` 就是干这个的（校验 `firmware/SHA256.json`、临时改 path、结束时还原）。

**TA 镜像就在磁盘上**：`/usr/local/lib/liuqin-fpc-oem/firmware/fpcliu.{mdt,b00..b08}`（约 2.1 MB），
QSEE 分段格式，可静态分析（见 §4.3）。

### 3.3 命令表（从 `fpc_build_info.c` 的 `query_output()` 反推，均已实测命中）

| 模块 | 子命令 | 语义 | 备注 |
|---|---|---|---|
| 12 | 1 | BUILD_INFO | 返回 JSON 长度/内容 |
| 10 | 0 | SENSOR_INIT | |
| 10 | 1 | CHECK_FINGER_LOST | |
| 10 | 2 | WAIT_FINGER_LOST | |
| 10 | 3 | WAIT_FINGER_DOWN | 配合 `/dev/fpc1020` 的 IRQ |
| 10 | 4 | CAPTURE | `words[2]`/`words[5]` 区分 HAL 的两种 capture 模式 |
| 10 | 5 | SENSOR_SLEEP | 收尾必发 |
| 10 | 6 | SENSOR_OTP | |
| 10 | 11 | CHECK_FINGER_STATE | |
| 11 | 0 | BIO_INIT | |
| 11 | 1 | BIO_BEGIN_ENROL | |
| 11 | 2 | BIO_ENROL | 返回 remaining（`words[2]`） |
| 11 | 3 | BIO_END_ENROL | 返回 template id（`words[2]`） |
| 11 | 4 | BIO_IDENTIFY | 返回候选 id（`words[2]`） |
| 11 | 5 | BIO_UPDATE_TEMPLATE | 返回 changed（`words[2]`） |
| 11 | 7 | BIO_LOAD_EMPTY_DB | |
| 11 | 8 | BIO_ENUMERATE_TEMPLATES | `words[2]=请求上限 5`，返回实际 count |
| 11 | 10 | BIO_SET_ACTIVE_GROUP | |
| 11 | 11 | AUTHENTICATOR_ID | 8 字节，在 +16 |
| 11 | 19 | BIO_LOAD_INSTANCE_DATA | **追加路径的关键一步** |
| 11 | 20 | BIO_SET_GID | `words[2]=0` |
| 3 | 1 | SET_AUTH_CHALLENGE | 64-bit，+8 |
| 3 | 2 | GET_ENROL_CHALLENGE | 64-bit，+8 |
| 3 | 3 | AUTHORIZE_ENROL | 69 字节 HAT，+16 |
| 3 | 4 | GET_AUTH_RESULT | 69 字节 HAT，+16 |
| 4 | 6 | DB_SIZE | 返回 total（`words[3]`） |
| 4 | 14 | DB_OPEN | `word3=1` 导入 / `0` 导出；`word4=size` |
| 4 | 15 | DB_CLOSE | |
| 4 | 16 | DB_READ | 导出分片，状态=remaining |
| 4 | 17 | DB_WRITE | 导入分片，状态=remaining |
| 5 | 0 / 6 | SELF_TEST / CAPTURE_UNCALIBRATED | 诊断用 |

### 3.4 数据库导入/导出（`load_database` / `store_database`）

导入（`fpc_build_info.c:855-905`）：
```
4/14 word3=1 word4=<file_size>     # "我要导入这么多字节"
4/17 word3=<len> data@+16          # 重复，直到 remaining==0；每次返回剩余
4/15                                # close → TA 在此做最终校验
```
导出（`:787-850`）：
```
4/6   → total
4/14 word3=0 word4=total
4/16 word3=<len> data@+16          # 重复，状态=remaining
4/15
```

实测：**导入的每个分片都不做内容校验，校验发生在最后一个 `4/17` 分片**。
所以 FP3 文件前 771 KB 都被"正常接受"，最后一小片才返回 `0xe853009f`（`4/15` 随后正常关闭）。

### 3.5 授权（HAT / Gatekeeper）

录入需要**两次**签名挑战（`fpc_build_info.c:1297-1300` 与 `:1328-1333`）：
```
3/2 GET_ENROL_CHALLENGE → challenge(64bit)
   → 写 challenge 到 pipe → qsee-gatekeeper --verify-enrol → 读 69B HAT
3/3 AUTHORIZE_ENROL(HAT) → 0
```
`init_hw_auth()`（`:280-328`）先向 Keymaster 应用 `"keymaster64"` 发 `0x205/2` 拿封装数据，
再用 `3/5` 注入 FPC 会话。实测 Gatekeeper 侧日志：`keymaster_api=4.1`、
`keymaster_command=0x207`、`0x21002`、`gatekeeper_verified_hat_forwarded=1 ... rounds=2 bytes=69`。

`verify` 路径用 `3/1 SET_AUTH_CHALLENGE` + `3/4 GET_AUTH_RESULT`，与录入的 3/2+3/3 是两套。

### 3.6 录入状态机：fresh vs existing

`--bio-pipeline`（新建，`:1291-1292`）：
```
BIO_INIT(11/0) → 3/5 hw_auth → SET_GID(11/20) → LOAD_EMPTY_DB(11/7)
→ SET_ACTIVE_GROUP(11/10) → 3/2+3/3 → BEGIN_ENROL(11/1)
→ [READY/采集/BIO_ENROL] × N → 3/2+3/3 → END_ENROL(11/3)
→ 校验 count==1 → 导出
```
`--bio-pipeline-existing`（追加，`:1291-1296`）：
```
BIO_INIT → 3/5 → SET_GID → load_database(4/14+4/17+4/15)
→ SET_ACTIVE_GROUP → BIO_LOAD_INSTANCE_DATA(11/19)     ← 多出来的一步
→ 3/2+3/3 → BEGIN_ENROL → …（同上，但跳过 count==1 校验）
```
`11/19` 在 verify 路径里也有（`match_existing():1009-1013`），所以它能跑通并不意外。

### 3.7 传感器 IRQ

`/dev/fpc1020` 以 `O_RDWR` 打开，`poll()` 等 IRQ；`READY finger_irq_armed=down press finger now`
之后才允许按压（`fpc_build_info.c:659-713`）。这也是为什么"没有 READY 行"能直接判定
**失败发生在采集之前**。

---

## 4. 数据库格式

### 4.1 FP3 容器（fprintd 存储文件，772,578 B）

本机（用户名 `yumo`，4 字符）实测布局【已确认】：

| 偏移 | 长度 | 内容 |
|---|---|---|
| `0x00` | 3 | `"FP3"` |
| `0x03` | 4 | u32 LE = `1`（版本） |
| `0x07` | 12 | driver 字符串 `"fpc1264_oem\0"` |
| `0x13` | 19 | device 字符串 `"liuqin-fpc1264-oem\0"` |
| `0x26` | 1 | `0x00` |
| `0x27`(39) | 1 | **finger**（本机 `2`；改成 7 时只有这个字节变） |
| `0x28` | 5 | username `"yumo\0"` |
| `0x2d` | 10 | `00 00 4c 4a 0b 00 01 00 00 00`（日期/标志，未完全解析） |
| `0x37`(55) | 772,495 | **raw OEM DB**（就是 `fpi-data`） |
| `0xBC9C6` | 28 | `00 28 75 61 79 29 00 76 30 00 00 00 2b…10 00 00 00` = `\0(uay)\0v0\0` + 4×u32，GVariant 风格类型串/偏移表 |

**偏移不是常数**：把 username 从 4 字符改成 6 字符，raw 起点从 55 移到 63（整文件 +8 B）。
所以任何"硬编码 offset 55"的 Python 切片都是错的；必须解析容器或用 libfprint 反序列化。

> 校验：本地按 `d[55:55+772495]` 取出的字节，sha256 与设备上 `fpi-dump` 抽出的
> `2adafd8740…` **完全一致**，证明上面这张表是对的。

### 4.2 raw OEM DB（772,495 B）

| 偏移 | 长度 | 内容 | 证据 |
|---|---|---|---|
| 0 | 4 | u32 LE = `772411`，恰好 `772495 - 84` | 两次导出相同 |
| 4 | 32 | `caced32189c1ef8d…`（32 字节，已截断显示） | 两次导出**完全相同** |
| 36 | 48 | 头部其余部分 | 两次导出不同 |
| 84 | 772,411 | 不透明主体，熵 ≈ 6.5 bit/B | 两次导出 **99.6% 字节不同** |

- 32 字节字段不是内容哈希：对 `A[36:]`、`A[84:]`、`A[36:36+772411]` 做 sha256 都不匹配；
  **推测**是 DB/密钥标识（TA 有 `fpc_db_get_database_id` 符号）。
- 主体 99.6% 不同说明每次导出都整体重加密/重排；**没有明文 magic/version/count**。
  想看模板数量只能用 TA 的 `11/8`（本机返回 1）。
- 本次对比的是 **`2a22b83c…`（备份）** 与 **`b63bb413…`（当前）** 两次导出；
  失败当时的 `ec9bfa2b…` 已被 16:57:47 的解锁覆盖，**没有留下可对比的字节**。
- 【待验证】"同一逻辑内容重复导出是否逐字节相同"——需要一次 `--db-import-export`（本报告
  未做，因为它不在 `oem_runtime.py` 的命令表里，需要另写编排）。

### 4.3 TA 内部的指纹集合：**硬上限 5**

TA 镜像是 QSEE 分段 ELF，可以按 program header 重建（`b01`→vaddr 0，`b02`→0xea000，
`b03`→0xeb000，`b04`→0xed000，`b05`/`b06`→0x1ef000，`b08`→0x213000），重建后有
`.dynsym`（2206 个符号）。从符号表拿到：

| 符号 | 地址 | 大小 |
|---|---|---|
| `fpc_check_enrollment_allowance` | `0x4d9c` | `0x70` |
| `fpc_db_add_fingerprint` | `0x2c8c` | `0x170` |
| `fpc_db_get_fingerprint_count` | `0x2ef8` | `0x9c` |

`fpc_db_add_fingerprint` 反汇编（关键片段）：

```asm
2d38: mov  w0, #0xb0            ; malloc 176 字节 = 0x10 头 + 5*0x20
2d54: bl   0x6d78               ; memset
2d5c: mov  w9, #0x10
2d68: str  wzr, [x23, x9]       ; 初始化 5 个槽
2d6c: add  x8, x8, #1
2d74: cmp  x8, #0x5
2d78: b.ne 0x2d68
...
2d80: mov  w0, #0x4c4
2d8c: movk w0, #0xe560, lsl #16 ; 满槽错误 0xe56004c4
2d90: ldur w10, [x9, #-8]       ; 槽状态
2d98: add  x8, x8, #1
2da0: cmp  x8, #0x5
2da4: b.ne 0x2d90
2da8: b    0x2ddc               ; 5 个都非空 → 返回错误
2dac: ...                       ; 找到空槽：写入模板指针/长度/状态=1
```

`fpc_db_get_fingerprint_count` 同样是 `cmp x9, #0x5` 遍历 5 个槽、统计 `state==1`。

**结论【已确认】：一个 OEM DB 最多 5 根手指**（16 字节集合头 + 5×32 字节槽；
与 FPC Android HAL 每用户 5 指上限一致，后者为推测）。
`11/8` 请求里 `words[2]=5` 正是这个上限。

### 4.4 模板计数怎么查（不按压）

- 走 TA：`--match-existing <raw.db>` 会先 `db_import → 11/10 → 11/19 → 11/8`，打印
  `loaded_template_count=N`，然后才 `READY` 等按压；在 READY 前/后杀掉即可，**不写任何文件**。
- 不要用"文件哈希"判断模板有没有变——见 §5.4。

---

## 5. 验证路径为什么不受影响

### 5.1 `--match-single` vs `--match-existing`

| | 原版驱动 | 改版驱动 |
|---|---|---|
| 客户端参数 | `--match-single` | `--match-existing` |
| 前置条件 | `count == 1`（`fpc_build_info.c:1019-1021`） | `count > 0`（`:1015-1018`） |
| 1 模板库行为 | 完全一致 | 行为等价 |

所以"只改录入、不改验证"在 1 模板库上**行为等价**；一旦库里有 2 个模板，
原版驱动会打印 `match_unavailable=single_template_database_required touch_not_collected=1`
并且**不采集**——这就是"多指库 + 回滚旧驱动 = 解锁失效"的原因（旧报告 §7 的警告是对的）。

### 5.2 TA 端是 1:N

`match_existing()` 在 `11/19` 之后调 `11/4 IDENTIFY`（`fpc_build_info.c:1031`），
TA 在**当前加载的整个 DB** 内找候选并返回候选 template id（`matched = words[2] != 0`）。
DB 本身就是 5 槽集合（§4.3），所以 identify 天然是 1:N（**推测**，由 5 槽结构 + 返回候选 id 推出）。
`get_auth_result(3/4)` 只校验 HAT 的版本和 challenge（`:967-993`），
`signature_validation=not_performed`，桌面认证开关在驱动层是关的（`desktop_authentication=disabled`）。
=> 一个 DB 里 N 根手指，任意一根按下去都能匹配，**一次按压**即可。

### 5.3 fprintd 侧只看到一个 print

驱动把 print 的手指号改回主手指（`work/fpc1264-oem.c:173-180`），fprintd 的
`print_data_save()` 按 `fp_print_get_finger()` 决定文件名（`fprintd/device.c:2045-2051`），
于是永远只覆盖同一个文件 `/var/lib/fprint/<user>/fpc1264_oem/liuqin-fpc1264-oem/2`。
不存在"多 print、gallery 只取第一个"的问题：`fprintd/device.c:1686-1693` 在
`gallery->len == 1` 时直接验那一个 print；`1727` 的"多 print + 无 IDENTIFY 只取
`gallery[0]`"是旧分析的坑，对"一库多指"方案不适用。

### 5.4 模板哈希漂移的真相：adaptive template update

OEM fprintd 带补丁（`fprintd-oem/fprintd-oem-update.patch:41-60` + `oem-update.inc`）：

```
verify 匹配成功
  → 驱动 accept_updated_database()（fpc1264-oem.c:336-382）把 TA 导出的新 DB 设回 print，
    并把旧 print 挂到 qdata "liuqin-fpc-oem-original"
  → 补丁在 match_cb 里调用 file_storage_print_data_update(original, match, …)
  → 临时文件 + fsync + hardlink 备份 + renameat 原子替换目标文件
  → 成功后才 report_verify_status(matched)
```

补丁命中证据（设备上）：

```
$ strings /usr/local/lib/liuqin-fpc-oem/fprintd | grep 'OEM verify storage_error'
OEM verify storage_error phase=adaptive-template-update domain=%s code=%d
```

**所以每次成功解锁都会重写模板文件（同大小、不同字节）——这是设计行为，不是损坏。**
把模板 sha256 当"是否被改动"的判据是无效的；失败 enroll 的那次哈希变化是**之前那次
16:54:36 的成功 verify** 造成的，不是失败 enroll 造成的（失败 enroll 在采集前退出，
`print == NULL`，`fprintd/device.c:2045` 的 `if (print)` 根本不成立）。

---

## 6. 多指可行性判断

### 6.1 判断

**能实现，上限 5 根；当前只差一个输入格式修复 + 一次真手指的端到端验证。**
没有发现 TA 侧的"单模板硬限制"；相反，TA 的指纹集合就是 5 槽数组。

### 6.2 支持证据（已确认的部分）

1. TA 有 `fpc_db_add_fingerprint` / `fpc_db_get_fingerprint_count` / 5 槽数组（§4.3）。
2. `--bio-pipeline-existing` 在真实 1 模板库上跑通了
   `load → GID → LOAD_INSTANCE_DATA → 两次 HAT → BEGIN_ENROL`（§2.4）。
3. 客户端源码本身为"N 模板库"设计：existing 分支**跳过** `count==1` 检查
   （`fpc_build_info.c:1340-1346`），注释明确说"不要把 1:N gallery 变成单指桌面凭据"。
4. `--match-existing` 在 N 模板库上是 TA 端 1:N（§5.2），验证侧不需要改。
5. 同一份 DB 用 TA 自带的 `11/8` 数出来是 1（§2.5），说明计数链路可用。

### 6.3 风险与限制

| 风险 | 说明 | 处置 |
|---|---|---|
| **5 根上限** | TA 硬编码 5 槽，满槽错误 `0xe56004c4` | 记录，不试图突破 |
| **同名手指会先被删** | `fprintd/device.c:2204-2220`：`EnrollStart` 若发现**请求的手指名**已有模板，会先 `delete_enrolled_fingers()`，把整个库删掉 | 追加时**必须请求一个不同的 finger**（例如已存 `left-index`，就请求 `right-index`） |
| **采集后环节未验证** | `BIO_ENROL / BIO_END_ENROL / 导出` 需要真手指 | 见 §7.2 |
| **回滚旧驱动会坏** | 库里 >1 模板时，原版 `--match-single` 拒绝加载 → 解锁失效 | 回滚必须连模板一起从备份恢复（`rollback.sh full`） |
| **删除是全有全无** | 没有"删单根"接口；`fprintd-delete` 删整个库 | 文档化 |
| **列表只显示主手指** | 库里 N 根，fprintd 只知道 1 个 print | 文档化 |
| **每次解锁重写模板** | adaptive update（§5.4），属正常 | 不要用哈希做完整性判断 |

### 6.4 需要满足的条件

1. **修 provider**：从 FP3 容器抽 `fpi-data` 再传给 `--enrol-existing`。
2. **用改版驱动**：`--match-existing` + `canonical_finger`（`work/fpc1264-oem.c:173-180,568`）。
3. **追加时用不同的 finger 名**（物理手指可以是同一根，见 §7.2 的技巧）。
4. **真手指完成一次采集**，确认 `BIO_ENROL/END_ENROL/导出` 成功、模板计数变 2、两根都能匹配。

---

## 7. 最小下一步

### 7.1 代码改动（最小、可回滚）

**方案 A（推荐）：加一个 `fpc-oem-extract` 小工具，复用 `fpc-oem-print.c` 的私有 FD 约束。**

- 位置：`/usr/local/lib/liuqin-fpc-oem/source/fpc-oem-extract.c` + 构建脚本
  （与 `build-tools/build-userspace.sh` 里 `fpc-oem-print` 同款）。
- 接口：`fpc-oem-extract --extract LINUX_USERNAME FINGER_NAME PRINT_FD OUTPUT_FD`
  - `fp_print_deserialize()` → 校验 `driver=="fpc1264_oem"`、`device_id`、`username`、`finger`；
  - 取 `fpi-data` 的 `(uay)`，校验 `version==1`、`0<len<=16 MiB`；
  - 输出 `O_EXCL|0600`，`fsync`；失败 `ftruncate(0)`。
- 然后改 `work/native_enrol.py` 的追加分支（当前 238-256 行）：

```python
base = Path(temporary) / "base-database"
print_fd = os.open(existing, os.O_RDONLY | os.O_CLOEXEC | os.O_NOFOLLOW)
raw_fd = os.open(base, os.O_WRONLY | os.O_CREAT | os.O_EXCL | os.O_CLOEXEC | os.O_NOFOLLOW, 0o600)
try:
    status = runtime.run([os.path.join(here, "fpc-oem-extract"), "--extract",
                          username, str(canonical_finger), str(print_fd), str(raw_fd)],
                         print_fd)   # 需要让 runtime.run 支持多个 pass_fds
finally:
    os.close(print_fd); os.close(raw_fd)
status = runtime.run([... "--enrol-existing", str(base), str(database), str(uid), str(secret_fd)], secret_fd)
```

> `runtime.run()` 现在只支持一个 `pass_fds`（`work/native_enrol.py:91-102`），需要扩成 tuple；
> 或者让 helper 用路径参数并自己按 `fpc-oem-print` 的规则校验（0600/root/regular，非 symlink）。

**方案 B（不推荐）：在 Python 里解析 FP3。** 需要处理变长字符串、对齐、GVariant 尾表；
一旦 libfprint 升级格式就坏。只适合临时验证。

> 打包注意：新工具要加进 `SHA256.json`/`SHA256SUMS` 与 `package_fingerprint.py` /
> `build-liuqin-fingerprint-deb.py` 的文件清单，否则安装/校验会漏。

**如果不想加二进制**：也可以把"抽 raw DB"放进驱动 `enrol()`——驱动已经链接 libfprint，
`fp_print_deserialize()` + 写临时文件是现成能力；但要把 fprintd 的存储路径规则复制进驱动，
比方案 A 侵入性大。

### 7.2 需要用户按手指的测试清单（我不执行）

**第一步（不需要第二根手指，验证协议）**：
1. 装好修复后的 provider + 改版驱动，重启 `fprintd`。
2. 让用户用**已录的那根手指**（left-index）按，但**请求的手指名必须不同**：
   ```sh
   fprintd-enroll -f right-index-finger yumo      # 需要 polkit 授权
   ```
   原因：如果 `-f left-index-finger`，fprintd 会在开始前删掉整个库（`device.c:2213-2220`）。
3. 期望：journal 出现驱动自己的
   `OEM enroll database extended under stored finger 2 (requested finger 7)`；
   provider 的 `native_enrol=OK database=extended canonical_finger=2` **不会**出现在 journal
   （驱动只把它当控制行消费，不转发）；文件仍是 `.../2`，大小可变。
4. 不需要按压也能查数量：用 §4.4 的 `--match-existing <raw>`，应看到 `loaded_template_count=2`。
5. 再用同一根手指 `fprintd-verify yumo`（不带 `-f`），期望 `verify-match`。

**第二步（真正的第二根手指）**：
6. `fprintd-enroll -f right-middle-finger yumo`，用户换一根手指按完流程。
7. 两根手指各 `fprintd-verify yumo` 一次，期望都 match；锁屏/PAM 各试一次。

**任一步失败**：`sh /root/fpc-backup-2026-10-27/rollback.sh full`，然后确认
`fprintd-list` 只有 `#0: left-index-finger`、`fprintd-verify` 仍可 match，把 journal 发回。

### 7.3 验收/回滚要点

- 装之前备份 `native_enrol.py`、`.so`、模板（模板已有 `fprint-var.tar.gz`）。
- 只重启 `fprintd`，不重启设备。
- 实验期间不要跑 `fprintd-delete`/`fprintd-enroll -f <主手指>`。
- 回滚必须用 `rollback.sh full`（含模板），否则多模板库 + 旧驱动 = 解锁坏。

---

## 8. 复现步骤（全部在 `/tmp/fp-re/`，不碰真实数据）

以下命令都在平板上以 root 执行。`<uid>` 用
`UID=$((0x60000000 + $(id -u yumo)))` 计算；`<state>` 是
`/var/lib/liuqin-fingerprint/native-enrollment`。

### 8.1 准备（只读拷贝）

```sh
mkdir -p /tmp/fp-re && cd /tmp/fp-re
cp /var/lib/fprint/yumo/fpc1264_oem/liuqin-fpc1264-oem/2 tpl-now.bin
mkdir -p /tmp/fpchk && tar -xzf /root/fpc-backup-2026-10-27/fprint-var.tar.gz -C /tmp/fpchk
cp /tmp/fpchk/fprint/yumo/fpc1264_oem/liuqin-fpc1264-oem/2 tpl-backup.bin   # 2a22b83c…
```

### 8.2 抽 raw DB（libfprint 反序列化，避免猜偏移）

编译 `fpi-dump.c`（源码见本报告 §10 的脚本副本 `fp-re-02-extract.sh`），
`S=/root/fpcbuild/sysroot`，链接系统 `libfprint-2.so.2.0.0`：

```sh
gcc -O2 -o fpi-dump fpi-dump.c -I$S/usr/include -I$S/usr/include/libfprint-2 \
  -I$S/usr/include/glib-2.0 -I$S/usr/lib/aarch64-linux-gnu/glib-2.0/include \
  -I$S/usr/include/gusb-1 -I$S/usr/include/json-glib-1.0 \
  -L/usr/lib/aarch64-linux-gnu -l:libfprint-2.so.2.0.0 -l:libglib-2.0.so.0 \
  -l:libgobject-2.0.so.0 -l:libgio-2.0.so.0 -l:libgusb.so.2 -l:libjson-glib-1.0.so.0
./fpi-dump tpl-backup.bin raw-backup.db      # raw_len=772495
```

### 8.3 复现失败（把 FP3 当 DB）

写一个 wrapper，按 `native_enrol.py` 的方式把 32 字节机器凭据做成 FD、exec 真实
`native_credentials.py`（脚本见 §10 `fp-re-run-enrol.py`）：

```sh
timeout -s TERM -k 8 25 python3 run-enrol.py /tmp/fp-re/tpl-backup.bin /tmp/fp-re/out-fp3.db
# 期望：db_import_remaining_mismatch expected=0 actual=-397213537
#       oem_runtime_cleanup firmware_path_restored=1 sensor_power_off=1 listener_stopped=1
#       echo $? = 1
```

### 8.4 复现成功到"等按压"

```sh
timeout -s TERM -k 8 14 python3 run-enrol.py /tmp/fp-re/raw-backup.db /tmp/fp-re/out-raw.db
# 期望：db_import=OK … READY enrol attempt=1 … READY finger_irq_armed=down press finger now
#       timeout 会在 14 s 后 SIGTERM；客户端会发 SENSOR_SLEEP 并清理
```

> **不要**在 READY 之后按手指（会真的开始录入）；到时间自动终止即可。

### 8.5 查模板数量

```sh
timeout -s TERM -k 6 10 python3 /usr/local/lib/liuqin-fpc-oem/oem_runtime.py \
    --ufs-rpmb-authenticated --match-existing /tmp/fp-re/raw-backup.db
# 期望：loaded_template_count=1，然后 READY …（被 timeout 杀掉）
```

### 8.6 关键校验（每轮都做）

```sh
cat /sys/module/firmware_class/parameters/path   # 应为空
cat /sys/bus/spi/devices/spi1.0/power_state      # off
systemctl is-active fprintd                      # active
sha256sum /var/lib/fprint/yumo/fpc1264_oem/liuqin-fpc1264-oem/2
pgrep -af 'oem_runtime|fpc_build_info|qsee-supplicant|authorize_oem'  # 应无
```

---

## 9. 清理步骤

```sh
# 1) 确认没有后台残留
pgrep -af 'oem_runtime|fpc_build_info|qsee-supplicant|authorize_oem|native_credentials' || echo clean
cat /sys/module/firmware_class/parameters/path   # 空
cat /sys/bus/spi/devices/spi1.0/power_state      # off

# 2) 删除所有生物特征拷贝（重要；本报告实验结束时已执行）
rm -f /tmp/fp-re/tpl-*.bin /tmp/fp-re/raw-*.db /tmp/fp-re/ta-image.bin /tmp/fp-re/out-*.db
# 需要彻底清空时：
#   rm -rf /tmp/fp-re
# 注意：/tmp/fpchk 是父任务回滚脚本 fp-restore.sh 的还原源（备份模板拷贝），
#       不属于本次实验，是否删除由父任务决定；不需要还原时建议一并删除。

# 3) 确认真实数据未被改动
sha256sum /var/lib/fprint/yumo/fpc1264_oem/liuqin-fpc1264-oem/2
# 实验前=实验后=b63bb413574de32d12be330562618c984e0a5dbc0fff20bb9b74fc736d95cf0e
stat -c '%y' /var/lib/fprint/yumo/fpc1264_oem/liuqin-fpc1264-oem/2   # 16:57:47（不是实验时间）
systemctl is-active fprintd
```

本报告的所有实验**没有**修改 `/var/lib/fprint/`、没有改 `/etc/pam.d/*`、没有动
`tod/*.so` / `native_enrol.py`；凭据目录 mtime 仍是 16:10:02。

---

## 10. 证据索引

| 内容 | 位置 |
|---|---|
| 改版 provider（错误来源） | `<workdir>\work\native_enrol.py:238-256`（尤其 243 行 `shutil.copyfile`） |
| 失败摘要行 | 同文件 `257-260` |
| 原版 provider（单指拒绝） | `<workdir>\backup\extract\source\native_enrol.py:157-160` |
| runtime 参数转发 | `oem_runtime.py:28-33, 89-94, 138, 141-142, 166-185` |
| `--enrol-existing` → `--bio-pipeline-existing` | `authorize_oem.py:75-77` |
| TEE 传输层 | `fpc_build_info.c:44-63, 85-117, 164-215` |
| 命令表 | `fpc_build_info.c:472-565` |
| DB 导入/导出 | `fpc_build_info.c:756-785, 787-850, 855-905` |
| 录入状态机 | `fpc_build_info.c:1284-1365` |
| match / single 检查 | `fpc_build_info.c:1001-1064`（1019-1021） |
| 模板枚举 | `fpc_build_info.c:613-657` |
| 驱动 enroll 失败分支 | `fpc1264-oem.c:134-170`（155-159 错误串） |
| 驱动日志白名单 | `fpc1264-oem.c:188-192` |
| provider 转发白名单 | `native_enrol.py:27-30` |
| 改版驱动 canonical finger | `work/fpc1264-oem.c:41, 173-180, 227-234, 568, 626` |
| fprintd 同名手指删除 | `fprintd/device.c:2204-2220` |
| fprintd enroll 保存点 | `fprintd/device.c:2045-2051` |
| adaptive update 补丁 | `fprintd-oem/fprintd-oem-update.patch:41-60`、`fprintd-oem/oem-update.inc:17-141` |
| TA 镜像 | `/usr/local/lib/liuqin-fpc-oem/firmware/fpcliu.{mdt,b00..b08}` |
| TA 5 槽证据 | `fpc_db_add_fingerprint` @ `0x2c8c`（`0xB0` 分配、两处 `cmp …, #5`）；`fpc_db_get_fingerprint_count` @ `0x2ef8` |
| 实验脚本（Windows 侧） | `<workdir>\_vm\fp-re-*.sh`、`<workdir>\_vm\fp-re-run-enrol.py` |
| 失败/成功原始日志 | 平板 `/tmp/fp-re/{fp3.log,raw.log,match.log,ta_dis.log}`（只含状态行，不含模板字节；仍在设备上） |

### 附：本报告用到的三次关键实测输出

**A. FP3 当 DB（失败）**

```
command=4 subcommand=14 db_word3=1 db_word4=772578
... 190 个分片全部接受 ...
command=4 subcommand=17 db_word3=1458 db_word4=0
tee_ret=0 origin=0 response_status=-397213537 status_hex=e853009f
db_import_remaining_mismatch expected=0 actual=-397213537
command=4 subcommand=15 db_word3=0 db_word4=0
oem_runtime_cleanup firmware_path_restored=1 sensor_power_off=1 listener_stopped=1
```

**B. raw DB（成功到等按压）**

```
command=4 subcommand=14 db_word3=1 db_word4=772495
db_import=OK path=/tmp/fp-re/raw-fpchk.db
command=11 subcommand=10 → 0
command=11 subcommand=19 → 0
gatekeeper_verified_hat_forwarded=1 round=1 rounds=2 bytes=69
command=3 subcommand=3 → 0
enrol_authorization=OK
command=11 subcommand=1 → 0
READY enrol attempt=1: press same finger and hold
READY finger_irq_armed=down press finger now
```

**C. 模板计数**

```
loaded_template_count=1
authenticator_id_nonzero=1
```

---

## 11. 术语与状态码速查

| 名称 | 值 | 含义 |
|---|---|---|
| `FPC_STATUS_OK` | 0 | 成功 |
| `FPC_STATUS_WAIT_TIME` | 1 | 采集等待/重试（**不是** journal 里的 `status=1`） |
| `FPC_STATUS_FINGER_LOST` | 3 | 手指离开 |
| `FPC_STATUS_BAD_QUALITY` | 4 | 质量差 |
| `FPC_STATUS_ENROLL_LOW_COVERAGE` | 7 | 覆盖不足 |
| DB 导入满槽错误 | `0xe56004c4` | 5 个指纹槽已满 |
| FP3 导入末片错误 | `0xe853009f` | DB 内容校验失败（本次的根因） |

---

## 附录 A：`run-enrol.py`（把真实机器凭据做成 FD 的 wrapper，复现用）

```python
#!/usr/bin/python3
import json, os, pwd, sys
STATE = "/var/lib/liuqin-fingerprint/native-enrollment"
USER = "yumo"                                     # 目标 Linux 用户
UID = str(0x60000000 + pwd.getpwnam(USER).pw_uid)  # native UID
if len(sys.argv) != 3:
    sys.exit("usage: run-enrol.py BASE OUTPUT")
base, out = sys.argv[1], sys.argv[2]
with open(os.path.join(STATE, UID, "credential.json"), encoding="ascii") as f:
    secret = bytes.fromhex(json.load(f)["secret_hex"])
fd = os.memfd_create("fp-re-secret", 0)
os.write(fd, secret); os.lseek(fd, 0, os.SEEK_SET); os.set_inheritable(fd, True)
secret = b"\0" * 32
os.execv("/usr/bin/python3", ["/usr/bin/python3",
    "/usr/local/lib/liuqin-fpc-oem/native_credentials.py", "--state-dir", STATE,
    "--enrol-existing", base, out, UID, str(fd)])
```

> `fpi-dump.c` 与 `fp3-meta.c` 的完整源码在 `<workdir>\_vm\fp-re-02-extract.sh` /
> `fp-re-13-fp3meta.sh` 里（heredoc 内嵌），可直接复用。

（报告完）
