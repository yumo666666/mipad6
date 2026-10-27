# Xiaomi Pad 6 Pro（liuqin）浏览器硬件视频解码任务书

## 为 qcom-iris（stateful V4L2 M2M）实现 VA-API 驱动，让 Firefox 用上硬解

| 项 | 内容 |
|---|---|
| 文档版本 | v1.0（2026-10-27） |
| 目标设备 | Xiaomi Pad 6 Pro 512GB，代号 **liuqin**，SoC **SM8475**（骁龙 8+ Gen 1），GPU Adreno 730 |
| 目标系统 | Ubuntu 26.04 LTS（resolute），内核 `6.17.0-rc1-gfe81794b5e1b`（自编，仓库 `github.com/yzddmr6/linux-sm8450-liuqin`，分支 `liuqin-6.17`） |
| 显示栈 | Wayland + GNOME 49/50，Mesa 26.0.3，freedreno（`msm_dri.so`），Vulkan ICD `freedreno_icd` |
| 目标 | 让浏览器（首要 Firefox，其次 Chromium/mpv 作为验证手段）通过 VA-API 使用 `/dev/video0`（`iris_driver`）硬解 H.264/HEVC/VP9 |
| 交付物 | 一个可安装的 VA-API 驱动 `iris_drv_video.so`（源码 + 构建脚本 + 测试脚本 + 部署说明） |
| 状态 | 可开工。第 2 章的"先例审计"是第一个决策门 |

---

## 0. 阅读约定：事实分级

本文所有结论都标了来源等级，**不要把【待验证】当成已知事实排期**。

| 标记 | 含义 | 处理方式 |
|---|---|---|
| 【实测】 | 2026-10-26/27 在设备上直接跑出来的结果 | 可直接作为设计输入 |
| 【文档】 | 来自内核/libva/FFmpeg/规范文档的确定行为，但未在本机逐条验证 | 开工时用一条命令复核 |
| 【待验证】 | 有依据的推断，或依赖本机内核/固件具体实现 | 必须写进 M0/M1 的验证清单，验证前不排后续工期 |
| 【检索】 | 只在网上检索到标题/简介，正文未读（本文档编写环境无法访问外网） | 开工第一步 clone/下载后审计 |

### 0.1 已确认的环境事实（【实测】）

**设备与内核**

- 设备：Xiaomi Pad 6 Pro 512GB（liuqin），SM8475，Adreno 730；8 核（1×X3 3.19 GHz + 3×A715 2.75 GHz + 4×A510 2.02 GHz），10.2 GiB 可用内存。
- 系统：Ubuntu 26.04 LTS，内核 `6.17.0-rc1-gfe81794b5e1b`。
- 桌面：Wayland 会话 + GNOME 49/50 级桌面；Mesa 26.0.3；gnome-shell 日志有 `Created gbm renderer for '/dev/dri/card0'`、`Obtained a high priority EGL context`。

**视频解码器**

- `/dev/video0`：driver name `iris_driver`，card type `Iris Decoder`，bus `platform:aa00000.video-codec`。
- 能力：`V4L2_CAP_VIDEO_M2M_MPLANE`（stateful memory-to-memory，多平面）。
- Device Caps 实测 `0x04204000` = `V4L2_CAP_VIDEO_M2M_MPLANE`(0x00004000) | `V4L2_CAP_EXT_PIX_FORMAT`(0x00200000) | `V4L2_CAP_STREAMING`(0x04000000)，**确认是 stateful M2M，不是 stateless Request API**。
- OUTPUT 格式：`H264`、`HEVC`、`VP90`，均带 `V4L2_FMT_FLAG_DYN_RESOLUTION`。
- CAPTURE 格式：**只枚举到 `NV12`**，没有 `NV12_10`/`P010`/`NV12_10BE_8L128` 等 10bit 格式。

**DRM / 图形**

- `/dev/dri/renderD128` 存在（`crw-rw---- root:render 226,128`），`/dev/dri/card0` 也在；当前用户 `yumo` 已在 `render` 组。
- Mesa 侧 `libEGL_mesa.so.0`、`libgbm.so.1`、`dri/msm_dri.so -> libdril_dri.so`（freedreno）齐备，dma-buf → EGL 导入的图形侧条件满足。

**VA / 用户态现状**

- `libva2 2.23.0-1ubuntu1` 已安装。
- **`vainfo` 命令行工具未安装**（仓库里有 `vainfo 2.22.0+ds1-2build1`，需 `apt install vainfo`）。此前记录"已安装"有误，以本条为准。
- `/usr/lib/aarch64-linux-gnu/dri/` 中现有 VA 驱动只有 Mesa 的 `d3d12_drv_video.so`、`nouveau_drv_video.so`、`r600_drv_video.so`、`radeonsi_drv_video.so`、`virtio_gpu_drv_video.so`——**没有任何 msm/iris/Qualcomm 的 VA 驱动**。因此无论 `LIBVA_DRIVER_NAME` 设成什么，当前都无可用驱动。
- Ubuntu 仓库**没有** `libva-v4l2-request` 包（`apt-cache search` 查不到）。
- GStreamer 1.28.2 有 `v4l2h264dec` / `v4l2h265dec` / `v4l2vp9dec`（Rank primary+1，标 Hardware，走 stateful M2M）——V4L2 这条路本身是通的。
- ffmpeg 8.0.1 有 `h264_v4l2m2m` / `hevc_v4l2m2m` 解码器，实测能解 720p H.264。
- mpv 0.41 的 `--hwdec=v4l2m2m` 不支持（mpv 只认 vaapi/vdpau/nvdec/vulkan/drm）。
- Firefox 149 是 snap 包（`firefox 149.0.2-1`）；浏览器内视频目前 100% 软解，实测 Firefox 主进程 + 内容进程合计占用 **140%~170% CPU**。
- 网络实测约 **283 kB/s**（2026-10-27 上午）。

### 0.2 关键参考实现（【检索】，正文待审计）

| 项目 | 定位 | 对本任务的意义 |
|---|---|---|
| `bootlin/libva-v4l2-request` | 把 **stateless** V4L2 Request API 解码器包装成 VA-API 驱动 | 代码组织、`vaExportSurfaceHandle`、surface/buffer 管理的参考；**解码路径不能复用** |
| `mxsrc/libva-v4l2` / `radxa-pkg/libva-v4l2` | GitHub 项目，简介明确写 "A VA-API driver backed by the Linux V4L2 **stateful** memory-to-memory decoder interface" | **最接近本任务**的现成骨架，可能已解决"VA 参数 → 完整码流"问题；M0 第一件事就是 clone 审计 |
| Debian ITP `#1152234: qcom-va-driver -- VA-API driver for Qualcomm Iris V4L2 devices`（2026-10） | 有人已向 Debian 提交"高通 Iris 的 VA-API 驱动"打包意向 | 若属实，可能直接复用/打包；M0 必须查清其源码地址与完成度 |
| Chromium `V4L2VideoDecodeAccelerator`（ChromeOS 路径） | stateful V4L2 的用户态封装 | 队列/格式协商/分辨率变化的参考，**不是 VA-API** |
| ffmpeg `libavcodec/v4l2_m2m*.c`、GStreamer `gstv4l2videodec` | stateful M2M 的成熟用法 | 队列、`SOURCE_CHANGE`、EOS 处理参考 |

> 说明：本文档编写时无法访问外网（web_fetch 被沙箱拒绝），上表两个项目的正文没有审计过，只有标题级证据。**不要把"它们已经能用"当作事实**，M0 的结论可能改写整个计划（见 §2.4 决策门）。

### 0.3 尚未验证的假设（开工前必须逐条打勾）

这张表是"假设台账"。任何一条被证伪，都可能改变设计或工期。

| # | 假设 | 验证方式 | 阶段 |
|---|---|---|---|
| A1 | iris 的 CAPTURE buffer 支持 `VIDIOC_EXPBUF` 导出 dma-buf | `tools/v4l2-probe` 调 EXPBUF，看返回值 | M0 |
| A2 | iris 在 CAPTURE 完成时调用 `v4l2_m2m_buf_copy_metadata()`，timestamp 会带回 | `grep` 内核源码 + probe 打印 timestamp | M0/M1 |
| A3 | iris 的 CAPTURE 队列接受 `V4L2_MEMORY_DMABUF` | probe 用 DMABUF 模式跑一遍 | M0 |
| A4 | ffmpeg 的 `start_frame(buffer,size)` 收到的是整个 packet（R2 直通的前提） | 读 ffmpeg 8.0.1 `h264dec.c` + 打日志 | M0 |
| A5 | Firefox 捆绑的 `libmozavcodec` 启用了 VA-API hwaccel | `about:support` / `MOZ_LOG` / `strings libmozavcodec.so` | M5 |
| A6 | Firefox 内容进程的 seccomp 不允许 `/dev/video0` | `strace` 看 `openat` 返回 EACCES | M5 |
| A7 | deb 版 Firefox 能读到 `LIBVA_DRIVER_NAME`/`LIBVA_DRIVERS_PATH` | 启动后 `cat /proc/<pid>/environ` | M5 |
| A8 | libva 在未设环境变量时用 DRM 驱动名（`msm`）回退 | `strace -e openat vainfo` | M1 |
| A9 | Mesa freedreno 能导入 linear NV12 dma-buf（`R8`+`GR88` 两层） | 最小 EGL 导入程序 + `eglinfo` | M4 |
| A10 | `gbm_bo_create(GBM_FORMAT_NV12)` 在 msm 上可用 | 10 行测试程序 | M3 |
| A11 | iris 支持 4K（3840×2160）解码 | `v4l2-ctl --set-fmt-video` + 4K 样片 | M3 |
| A12 | ffmpeg 在 `avcodec_flush_buffers()` 时销毁并重建 VA context（而不是保留） | 读 `vaapi_decode.c` + `LIBVA_IRIS_DEBUG` 日志 | M3 |
| A13 | Firefox 在 `RESOLUTION_NOT_SUPPORTED` 时会重建 decoder | 构造分辨率变化流 + `MOZ_LOG` | M6 |
| A14 | snap 版 Firefox 的 `/dev/video0` 被 AppArmor 拒绝、驱动 so 不可见 | `snap run --shell` + `snap connections` | M5 |
| A15 | `mxsrc/radxa libva-v4l2` 与 Debian `qcom-va-driver` 的真实能力 | clone/查 WNPP | M0 |

---

## 1. 目标与验收标准

### 1.1 一句话目标

在 Ubuntu 26.04/liuqin 上交付一个 VA-API 驱动，使**未经修改的 Firefox（或至少 Mozilla 官方 deb 版）**在播放 B 站 1080p H.264 时，解码工作由 `/dev/video0` 的 iris 硬件完成，CPU 占用从 140%~170% 降到 60% 以下，画面与软解一致。

### 1.2 分层验收（L0–L4）

每一层都是**可独立验收、可独立交付**的，前一层的通过是后一层开工的前提。

| 层 | 验收内容 | 判定命令/观察点 | 通过标准 |
|---|---|---|---|
| **L0 驱动可用** | VA 驱动能被 libva 加载，能力查询正确 | `vainfo --display drm --device /dev/dri/renderD128` | 退出码 0；输出含 `VAProfileH264High : VAEntrypointVLD`；`VA-API version` 与 `libva 2.23` 匹配；`Driver version` 显示本项目版本号 |
| **L1 能解码（有拷贝）** | 单文件 H.264 解码正确 | `ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 -i test_1080p_h264_high.mp4 -f null -` | `-v error` 无输出；`frame= N` 与 `ffprobe -count_frames` 一致；逐帧 PSNR 对软解 ≥ 50 dB（理想 inf） |
| **L2 零拷贝导出** | surface 能以 dma-buf 导出并被 EGL 采样 | `ffmpeg -hwaccel vaapi -hwaccel_output_format vaapi ... -f null -`；`mpv --hwdec=vaapi --vo=gpu-next test.mp4` | 不经过 `hwdownload` 也能出画面；`mpv` 的 `--msg-level=vo=v` 中无 `vaExportSurfaceHandle` 失败；播放 10 分钟无花屏/错序 |
| **L3 浏览器可用** | Firefox 走 VA-API 播放网页视频 | B 站 1080p（H.264）播放 10 分钟，`about:support` 的 Graphics/Media 段落显示 WebRender 与硬件解码正常（具体文案随版本）；`pidstat` 采样 | Firefox 全进程 CPU 合计 **≤ 60%**（基线 140%~170%）；`about:media`/`MOZ_LOG` 显示 `VA-API` 解码器；无 AV 不同步；拖动进度条 10 次不崩 |
| **L4 生产可用** | 多编码/分辨率/seek/长稳 | 第 9 章测试矩阵全绿 | 2 小时循环播放 RSS 增长 < 5%；4K30 H.264 可解；HEVC 1080p 可解；VP9 1080p 可解；10bit 明确返回不支持且不崩 |

### 1.3 量化指标

| 指标 | 基线（软解，【实测】） | L3 目标 | L4 目标 | 测量方法 |
|---|---|---|---|---|
| Firefox 总 CPU（1080p30 H.264，60 s 平均） | 140%~170% | ≤ 60% | ≤ 40% | `pidstat -h -p ALL 1 60` 过滤 firefox 进程，或 §9.5 的方法 |
| 单帧解码延迟（提交到 surface ready） | — | ≤ 30 ms（P95） | ≤ 20 ms | 驱动内打点 + `test/latency.c` |
| 首帧时间（打开到首帧上屏） | — | ≤ 500 ms | ≤ 300 ms | `mpv --msg-level=all=v` 时间戳 |
| 丢帧 | — | 0（1080p30） | 0（1080p60/4K30） | `mpv` 统计 / `about:media` |
| 内存增长（2 h） | — | < 10% | < 5% | `/proc/<pid>/smaps_rollup` 的 `Pss` |
| 整机功耗（1080p30） | 待测 | 比软解低 ≥ 15% | 低 ≥ 25% | `/sys/class/power_supply/battery/{current_now,voltage_now}` 乘积 |
| 解码正确性 | 基准 | PSNR ≥ 50 dB | PSNR = inf | §9.4 |

> 注意：CPU 目标不要拍脑袋定死。1080p30 H.264 在 A510 上软解约 1.5 核；硬解后 Firefox 仍需做颜色转换/合成（WebRender 用 GPU），**30%~50% 是合理区间**，"≤ 60%"是保守可交付线，"≤ 40%"是优化目标。

### 1.4 验收脚本（放 `test/accept.sh`）

```bash
#!/bin/sh
set -eu
export LIBVA_DRIVER_NAME=iris
export LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri

# L0
vainfo --display drm --device /dev/dri/renderD128 | tee /tmp/vainfo.log
grep -q 'VAProfileH264High *: *VAEntrypointVLD' /tmp/vainfo.log

# L1
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i "$1" -f null - 2>/tmp/ffmpeg.err
test ! -s /tmp/ffmpeg.err

# L1 正确性（逐帧 PSNR）
./test/psnr_vs_sw.sh "$1"

# L2
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi -i "$1" -f null -
```

### 1.5 非目标（明确不做，避免范围蔓延）

- **不做编码**（`VAEntrypointEncSlice`）——iris 是纯解码器，本项目只做 `VAEntrypointVLD`。
- **不做 10bit**（HEVC Main10 / VP9 Profile2 / H.264 High10）——capture 侧只有 NV12，见 §5.2。
- **不做 AV1/VP8/MPEG-2/VC-1**——`/dev/video0` 未枚举这些格式。
- **不做 DRM（Widevine/Widevine L3）受保护内容**——VA-API 的 protected session 路径不在范围内。
- **不改内核 iris 驱动**——除非 M1 证明内核有阻塞性 bug；改内核要单独评估。
- **不做 HDR/色彩管理**——8bit BT.709 之外的颜色空间先按"透传"处理。

---

## 2. 架构方案对比与选型

### 2.1 三个硬约束（决定了方案空间）

**约束 1：iris 是 stateful，不是 stateless。**

stateful 解码器的内核接口是"两条 vb2 队列 + 事件"：用户态把**完整码流**写进 OUTPUT 队列，内核（HFI 固件）自己解析 SPS/PPS/slice header，解出来的帧从 CAPTURE 队列按显示序返回。用户态不参与语法解析，也拿不到 per-slice 的控制接口。

stateless（Request API）则相反：用户态解析码流，把 SPS/PPS/解码参数/slice 参数作为 `V4L2_CID_STATELESS_*` control 绑到 `media_request` 上，内核只做"按参数驱动硬件"。

两者**不是同一个编程模型**，这是选型的第一决定因素。

**约束 2：VA-API 是"已解析参数"API，不提供原始码流。**

VA-API 的 H.264 解码接口给驱动的是：

- `VAPictureParameterBufferH264`（SPS/PPS 的解析结果：profile、分辨率、`log2_max_frame_num`、POC 类型、参考帧数、熵编码模式、QP、去块参数……）；
- `VASliceParameterBufferH264`（每个 slice 的解析结果）；
- `VAQMatrixBufferH264`（缩放矩阵）；
- `VASliceDataBufferType`（**只有 slice NAL 的字节**，不含 SPS/PPS，通常也不含起始码）。

也就是说：**VA-API 客户端（ffmpeg）不会把 SPS/PPS 的原始字节交给驱动**，它期望驱动自己用参数"喂硬件"。Intel 的 VA 驱动正是这么做的（把 SPS/PPS 写进硬件寄存器）。而 stateful V4L2 需要一份**语法完整的 Annex B 码流**。

→ 这是本项目最大的技术障碍：stateful 后端必须**从 VA 参数反向重建 SPS/PPS（HEVC 还要 VPS/RPS）**，或者让客户端把原始码流也送进来。详见 §2.6 与 §3.6。

**约束 3：Firefox 的要求比 `vainfo` 高得多。**

Firefox 不只是"能解码"，它要求：

1. `vaInitialize` 在 **render node**（`/dev/dri/renderD128`）上成功；
2. 有 `vaExportSurfaceHandle`，且能用 `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` 导出 **NV12 dma-buf**；
3. 导出的 dma-buf 能被 Mesa freedreno 的 `EGL_EXT_image_dma_buf_import` 导入；
4. 上述动作发生在 Firefox 的**内容进程/RDD 进程**里，受 seccomp/AppArmor 限制（见 §7.4）；
5. snap 版 Firefox 还多一层挂载命名空间隔离（见 §7.5）。

### 2.2 候选方案

| 方案 | 做法 | 复用 | 主要工作量 | 风险 | 结论 |
|---|---|---|---|---|---|
| **A. 新写 `libva-iris`** | 直接实现 libva 驱动 ABI（`va_backend.h`），后端是 V4L2 M2M | 参考 B 的导出/管理代码 | 驱动骨架 + 码流重建 + 队列状态机 | 码流重建正确性 | **推荐** |
| **B. fork `libva-v4l2-request` 加 stateful 后端** | 在 request 驱动里加一条 M2M 路径 | 约 30%（surface/buffer/export/config） | 等于 A，还多背一套 request 抽象 | 抽象不匹配，代码越改越拧 | 不推荐（见 §2.3） |
| **C. VA-API → GStreamer/ffmpeg 桥接** | VA 驱动内部起 `gst-launch`/ffmpeg 子进程，把帧传回 | 高 | 进程/线程通信、帧拷贝、同步 | 延迟、零拷贝、生命周期、Firefox 要求 | 不推荐（见 §2.5） |
| **D. 客户端直通补丁 + 驱动直通模式** | 给 ffmpeg 打补丁，把原始 packet 用私有方式送给驱动；驱动跳过码流重建 | 驱动侧省掉重建 | 打补丁 + 重编 ffmpeg（Firefox 要重编） | 与 Firefox 官方包不兼容 | **并行做**：作为调试通道与兜底（§2.6 R2） |
| **E. 等/用现成项目** | 用 `radxa-pkg/libva-v4l2` 或 Debian `qcom-va-driver` | 可能 100% | 适配 iris | 项目成熟度未知 | **M0 先审计**（§2.4） |

### 2.3 为什么不能直接复用 `libva-v4l2-request`

这一节是给"要不要 fork"的决策依据，逐条论证。

**(1) 内核接口不同，且本机根本没有 request API。**

`libva-v4l2-request` 的核心调用序列是：

```
open("/dev/media0") → MEDIA_IOC_REQUEST_ALLOC → 得到 request fd
对每个 slice：VIDIOC_S_EXT_CTRLS(V4L2_CID_STATELESS_H264_SPS / _PPS / _DECODE_PARAMS / _SLICE_PARAMS)
VIDIOC_QBUF(..., V4L2_BUF_FLAG_REQUEST_FD, request_fd)
MEDIA_REQUEST_IOC_QUEUE
poll(request_fd) / VIDIOC_DQBUF(CAPTURE)
```

iris 上：

- 没有 `/dev/media*` 节点（它走 `v4l2_m2m`，不是 media controller）；
- `MEDIA_IOC_REQUEST_ALLOC` 会返回 `ENOTTY`；
- `V4L2_CID_STATELESS_H264_*` 这组 control 在 stateful 驱动上不存在，`VIDIOC_S_CTRL` 会返回 `EINVAL`；
- `VIDIOC_QBUF` 带 `V4L2_BUF_FLAG_REQUEST_FD` 会返回 `EINVAL`。

结论：request 驱动的**提交路径 0% 可用**。

**(2) 数据模型相反：一个要"参数"，一个要"码流"。**

request 驱动拿到的是"SPS 结构体 + PPS 结构体 + slice 参数 + slice 数据"，正好是 VA-API 提供的东西，所以映射几乎 1:1。
stateful 驱动要的是"一份能自己解析的 Annex B/HEVC/VP9 码流"，VA-API 恰恰**不给**这个。方向相反，不是"改一改"能解决的。

**(3) 生命周期模型不同。**

request：一个 request 对应一帧，提交后异步完成，`poll(request_fd)` 拿结果；
M2M：OUTPUT/CAPTURE 两条队列长期存在，`QBUF`/`DQBUF` 与 `poll` 交错，还有 `SOURCE_CHANGE` 事件、`V4L2_DEC_CMD_STOP` 排空、`V4L2_BUF_FLAG_LAST` 收尾。状态机要重写。

**(4) 分辨率变化的责任方不同。**

stateless：客户端解析 SPS 后自己决定新格式，重建 CAPTURE 队列（客户端主导）。
stateful：内核发 `V4L2_EVENT_SOURCE_CHANGE`，用户态必须按固定序列处理（`G_FMT` → `S_FMT` → `STREAMOFF` → `REQBUFS(0)` → `REQBUFS` → `STREAMON`），而且 VA surface 是固定尺寸的，冲突要驱动自己兜（§6.5）。

**(5) 代码复用评估（诚实版）。**

| 模块 | 能否复用 | 说明 |
|---|---|---|
| `vaExportSurfaceHandle` / DRM_PRIME_2 导出 | **能**（可借鉴） | 结构体填充、fd dup、layer 布局是通用的 |
| surface/buffer 对象管理 | 部分 | request 驱动的 buffer 带 request 关联，要拆掉 |
| config/profile 表 | 能 | 但 profile 集合要按 iris 重写 |
| meson 构建、安装路径、`vainfo` 联调 | 能 | 直接抄 |
| **解码提交/完成路径** | **不能** | 全部重写 |
| 码流重建 | 不存在 | request 驱动不需要，本项目要新写 |

净复用约 30%，且是最不困难的那 30%；fork 还会带来"request 抽象残留"的理解成本。

**结论：不 fork。以 `libva-v4l2-request` 为代码风格参考，新写 `libva-iris`，内部按"V4L2 M2M 后端 + 码流重建器"分层。**

### 2.4 先例审计（M0 决策门）

开工前 2~3 天只做一件事：**确认有没有现成轮子**。

1. `git clone https://github.com/mxsrc/libva-v4l2`、`https://github.com/radxa-pkg/libva-v4l2`，读 `README`、`meson.build`、`src/`：
   - 它支持哪些 codec？H.264 的 SPS/PPS 是重建还是直通？
   - 它怎么处理 `V4L2_EVENT_SOURCE_CHANGE`、时间戳映射、`vaExportSurfaceHandle`？
   - 它有没有"客户端直通"模式？
2. 查 Debian WNPP `#1152234`（`qcom-va-driver`）：源码仓库、维护者、是否已进 experimental/mentors、支持哪些 SoC、是否已支持 iris 的 HFI 固件行为。
3. 在设备上 `apt-cache search va-driver | grep -i qcom`、`apt-cache search libva | grep -i v4l2` 复核仓库。

**决策规则：**

- 若 `radxa libva-v4l2` 已支持 H.264 且代码质量可接受 → 方案改为"**fork + 加 iris 适配**"，工期砍半（见 §10 的乐观列）。
- 若 Debian `qcom-va-driver` 已是可用驱动 → 直接打包/移植，本项目退化为"集成 + 验证 + 修 bug"。
- 若两者都不可用 → 按方案 A 执行，本文档其余部分即为实施方案。

### 2.5 为什么不做方案 C（VA → GStreamer/ffmpeg 桥接）

看起来"起个 `gst-launch` 或 ffmpeg 子进程解码，再把帧搬回来"最省事，但：

1. **零拷贝不可能**：帧必须跨进程/跨 API 边界搬一次（至少一次 memcpy，1080p30 NV12 约 93 MB/s，4K30 约 373 MB/s），CPU 省下来的又被拷贝吃回去。
2. **延迟与抖动**：VA-API 是同步/半同步调用模型，`vaSyncSurface` 期望毫秒级；桥接要多一次 IPC 往返 + 队列，seek 时清空管道很难做干净。
3. **Firefox 要的是"VA display 上的 surface 能导出 dma-buf"**：`vaExportSurfaceHandle` 必须返回一个**属于本进程 VA display** 的 dma-buf。桥接进程解出来的帧要么再导入回来（多一次 dma-buf 导入 + 同步），要么用共享内存（不再是 dma-buf，EGL 导入失败）。
4. **生命周期/错误传播**：子进程崩溃、卡死、版本不匹配，都会变成 Firefox 的"视频卡住"，很难定位。
5. **维护成本**：等于同时维护"VA 驱动外壳 + 一个播放器"，且两头升级都会破坏兼容。

结论：桥接只适合做**一次性验证工具**（例如用 GStreamer 验证 V4L2 通路），不能作为交付架构。

### 2.6 关键技术障碍与三种对策（本项目最需要拍板的地方）

**障碍：VA-API 不提供 SPS/PPS 原始字节，而 stateful V4L2 需要完整码流。**

三种对策：

| 对策 | 做法 | 优点 | 缺点 | 适用 |
|---|---|---|---|---|
| **R1 驱动内重建** | 用 `VAPictureParameterBufferH264` + `VAQMatrixBufferH264` 重建 SPS/PPS，slice NAL 原样透传（补起始码） | 对客户端零改动，Firefox/ffmpeg/GStreamer 通吃 | 工作量大；HEVC 的 RPS 可能信息不足；边界情况多 | **H.264 主路线**；VP9 可行；HEVC 待评估 |
| **R2 客户端直通** | 给 ffmpeg 打补丁，把 `start_frame` 拿到的整包（含 SPS/PPS）作为一个特殊 buffer 送给驱动；驱动直接转发 | 驱动侧极简，码流 100% 原样，HEVC/VP9 都能过 | 要重编 ffmpeg；Firefox 用的是自带 ffmpeg，得重编 Firefox；官方 snap/deb 不认 | **调试通道 + 兜底**（先做，能极大加速 bring-up） |
| **R3 混合** | H.264 用 R1；HEVC/VP9 先只支持 R2；或 HEVC 只支持"每个 IDR 前带 SPS/PPS 的流" | 平衡 | 体验不一致 | 时间紧时的现实选择 |

**建议：R1 与 R2 并行。** 第一周用 R2 打通"VA 驱动 → V4L2 → dma-buf → 出画面"的全链路（不碰码流重建），第二周起用 R1 替换，最终交付 R1；R2 保留为 `--enable-passthrough` 编译选项，用于 HEVC 紧急兜底和回归对比。

R2 的具体做法（给 ffmpeg 的最小补丁）：

```c
/* 伪代码（libavcodec/vaapi_h264.c: vaapi_h264_start_frame 开头）
 * buffer/size 是 h264dec 传进来的整个 packet（含 SPS/PPS/SEI/slices）。
 * 这里直接拿 VAAPIDecodeContext 里的 display/context 建一个带魔数的 buffer。 */
if (getenv("VAAPI_IRIS_PASSTHROUGH")) {
    uint8_t *raw = av_malloc(size + 8);
    memcpy(raw, "IRISRAW1", 8);
    memcpy(raw + 8, buffer, size);
    VABufferID id;
    vaCreateBuffer(ctx->hwctx->display, ctx->context, VASliceDataBufferType,
                   size + 8, 1, raw, &id);
    vaRenderPicture(ctx->hwctx->display, ctx->context, &id, 1);
    vaDestroyBuffer(ctx->hwctx->display, id);
    av_free(raw);
}
```

驱动侧：若 `vaRenderPicture` 收到的第一个 buffer 以魔数 `"IRISRAW1"` 开头（补丁里加 8 字节头），则把它当作**整个 AU 的完整码流**，直接 `QBUF`，忽略后续的 pic/slice 参数 buffer；否则走 R1 重建路径。这样同一个驱动二进制同时支持两种客户端，且不影响未打补丁的客户端。

> 注意：`buffer/size` 在 `start_frame` 里是否是整包，需要在 ffmpeg 8.0.1 源码里确认（`libavcodec/h264dec.c` 调用 `h->hwaccel->start_frame(avctx, ...)` 的实参）。若不是整包，退化为在 `decode_slice` 里累积并在 `end_frame` 时补发 SPS/PPS——但那样拿不到 SPS/PPS 字节，R2 就不成立。**这是 M0 必须验证的第一件事。**

### 2.7 推荐架构与模块划分

```
┌────────────────────────────────────────────────────────────┐
│ Firefox / ffmpeg / mpv / GStreamer（VA-API 客户端）          │
└───────────────────────────┬────────────────────────────────┘
                            │ vaInitialize / vaCreateConfig / ...
┌───────────────────────────▼────────────────────────────────┐
│ libva（2.23） 加载 iris_drv_video.so，转发到 VADriverVTable  │
└───────────────────────────┬────────────────────────────────┘
┌───────────────────────────▼────────────────────────────────┐
│ libva-iris（本项目）                                         │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │ VA 前端      │ │ 码流重建器    │ │ V4L2 M2M 后端         │ │
│  │ config/      │ │ h264_sps.c   │ │ open/S_FMT/REQBUFS    │ │
│  │ context/     │ │ h264_pps.c   │ │ QBUF/DQBUF/poll       │ │
│  │ surface/     │ │ h264_slice.c │ │ SOURCE_CHANGE/EOS     │ │
│  │ buffer       │ │ hevc_*/vp9_* │ │ 时间戳↔surface 映射    │ │
│  └─────────────┘ └──────────────┘ └──────────┬───────────┘ │
│  ┌──────────────────────────────────────────┐ │             │
│  │ export.c：vaExportSurfaceHandle / image.c │ │             │
│  └──────────────────────────────────────────┘ │             │
└───────────────────────────────────────────────┼─────────────┘
                                                │ ioctl
                                    ┌───────────▼───────────┐
                                    │ /dev/video0 iris_driver│
                                    │  (stateful M2M, NV12)  │
                                    └───────────────────────┘
```

模块与文件（建议布局，M1 就按这个建）：

```
libva-iris/
├── meson.build
├── meson_options.txt
├── README.md
├── src/
│   ├── driver.c        # VA_DRIVER_INIT_FUNC、VADriverVTable 装配、版本/厂商
│   ├── display.c       # drm fd、QUERYCAP、ENUM_FMT、能力缓存
│   ├── config.c        # profile/entrypoint/attribute 表
│   ├── context.c       # V4L2 打开、S_FMT、REQBUFS、STREAMON、SOURCE_CHANGE
│   ├── surface.c       # surface 池、dma-buf 绑定、状态
│   ├── buffer.c        # VABufferID 管理、map/unmap
│   ├── decode.c        # Begin/Render/End/Sync/QueryStatus
│   ├── export.c        # vaExportSurfaceHandle（DRM_PRIME_2）
│   ├── image.c         # vaDeriveImage/GetImage/PutImage/QueryImageFormats
│   ├── v4l2.c          # ioctl 封装、poll 循环、错误映射
│   ├── queue.c         # OUTPUT/CAPTURE 队列、timestamp↔surface 映射
│   ├── bitstream/
│   │   ├── bs_writer.c # RBSP 写位、EPB 插入、ue/se 编码
│   │   ├── h264_sps.c
│   │   ├── h264_pps.c
│   │   ├── h264_au.c   # AUD/SPS/PPS/slice 组装
│   │   ├── hevc_ps.c   # VPS/SPS/PPS（阶段 6）
│   │   └── vp9_hdr.c   # VP9 uncompressed header（阶段 6）
│   └── profile.c
├── test/
│   ├── va-dump.c       # 最小 VA 客户端：读 .h264/.ivf，解码，导出 NV12
│   ├── test_bs.c       # 码流重建器单测：合成 AU → ffmpeg 软解 → 比对
│   ├── psnr_vs_sw.sh
│   └── accept.sh
└── tools/
    └── v4l2-probe.c    # 直接跑 V4L2 的探针（不经过 VA），用于隔离问题
```

**分层原则：`bitstream/` 不碰任何 ioctl，`v4l2.c` 不碰任何 VA 结构体。** 这样码流重建器可以用纯用户态单测覆盖（§9.2），不用每次都上硬件。

---

## 3. VA-API → V4L2 具体映射

本章是"照着写"的部分。所有 V4L2 结构体以 `linux/videodev2.h` 为准，所有 VA 结构体以 `/usr/include/va/va*.h`（libva 2.23）为准。

### 3.1 总览表

| VA-API 入口（`VADriverVTable` 字段） | 驱动内部动作 | 对应 V4L2 ioctl / API | 备注 |
|---|---|---|---|
| `vaInitialize` → `VA_DRIVER_INIT_FUNC` | 打开/接管 drm fd、探测设备、装配 vtable | `VIDIOC_QUERYCAP`、`VIDIOC_ENUM_FMT`(OUTPUT/CAPTURE)、`VIDIOC_QUERYCTRL` | 探测结果缓存，后续查询不再进内核 |
| `vaTerminate` | 释放所有对象 | `close()` | 幂等 |
| `vaQueryConfigProfiles` | 返回 profile 数组 | 无（用 init 缓存） | 只返回 `VIDIOC_ENUM_FMT` 支持且 `TRY_FMT` 通过的 |
| `vaQueryConfigEntrypoints` | 返回 `{VAEntrypointVLD}` | 无 | 解码器只有 VLD |
| `vaGetConfigAttributes` | 返回 `VAConfigAttribRTFormat` 等 | 无 | 见 §3.2 |
| `vaCreateConfig` | 校验 profile/entrypoint/attrib，建 config 对象 | 无 | 不碰 V4L2，真正的设备在 context 打开 |
| `vaDestroyConfig` | 释放 | 无 | |
| `vaCreateContext` | **打开 `/dev/video0`，建立两条队列** | `open`、`VIDIOC_SUBSCRIBE_EVENT`、`S_FMT`×2、`G_CTRL`、`REQBUFS`×2、`QUERYBUF`、`mmap`、`EXPBUF`、`STREAMON`×2 | §3.3 有完整序列 |
| `vaDestroyContext` | 停流、释放 buffer、关 fd | `STREAMOFF`×2、`REQBUFS(0)`、`munmap`、`close` | |
| `vaCreateSurfaces` | 分配/导入 surface（拷贝模式：新分配 dma-buf；别名模式：取 CAPTURE 池） | 无（别名模式的池在 context 建好） | 见 §4.3 |
| `vaDestroySurfaces` | 归还池 | 无 | |
| `vaCreateBuffer` / `vaMapBuffer` / `vaUnmapBuffer` / `vaDestroyBuffer` | 用户态内存管理 | 无 | `vaMapBuffer` 返回 CPU 指针 |
| `vaBeginPicture` | 绑定 surface、清空 AU 缓冲、置 DECODING | 无 | |
| `vaRenderPicture` | 按 buffer type 分派；slice data 追加进 AU | 无 | §3.4 |
| `vaEndPicture` | 组装 AU → 取空闲 OUTPUT buffer → 拷贝 → 入队 | `VIDIOC_QBUF`(OUTPUT) | `bytesused`=AU 长度，`timestamp`=frame_seq |
| `vaSyncSurface` | 等到该 surface 对应的 CAPTURE buffer 出队 | `poll()`、`VIDIOC_DQBUF`(CAPTURE)、`VIDIOC_DQEVENT` | 见 §6 |
| `vaQuerySurfaceStatus` | 返回内部状态 | 无 | |
| `vaQuerySurfaceError` | 返回解码错误码 | 无 | |
| `vaExportSurfaceHandle` | `dup()` surface 的 dma-buf，填 `VADRMPRIMESurfaceDescriptor` | 无（fd 在 surface 创建时已有） | §4.4 |
| `vaDeriveImage` / `vaGetImage` / `vaPutImage` | 把 dma-buf 映射成 `VAImage` | `mmap`/`DMA_BUF_IOCTL_SYNC` | §3.5 |
| `vaQueryImageFormats` | 返回 NV12 | 无 | |
| `vaPutSurface` | 不实现 | 无 | 返回 `VA_STATUS_ERROR_UNIMPLEMENTED`（Firefox/mpv 不走它） |
| `vaQuerySurfaceAttributes` | 返回像素格式/尺寸范围 | 无 | ffmpeg 初始化时会查 |
| （无直接对应） | seek/flush | `STREAMOFF`×2 → `REQBUFS(0)` → 重新 `S_FMT`/`REQBUFS`/`STREAMON` | §6.6 |
| （无直接对应） | 排空/EOS | `VIDIOC_DECODER_CMD(V4L2_DEC_CMD_STOP)` → 等 `V4L2_BUF_FLAG_LAST` | §6.4 |

### 3.2 初始化与能力查询

**`vaInitialize`（驱动侧 `__vaDriverInit_1_0`）**

libva 会先打开 DRM 设备并把 fd 放进 `VADriverContext`，然后 `dlsym` 驱动 so 里的 `VA_DRIVER_INIT_FUNC`。驱动必须：

```c
VA_EXPORT VAStatus VA_DRIVER_INIT_FUNC(VADriverContextP ctx)
{
    struct iris_driver *drv = calloc(1, sizeof(*drv));
    ctx->pDriverData = drv;
    ctx->version     = VA_MAJOR_VERSION * 1000 + VA_MINOR_VERSION; /* 与 intel-media-driver 写法一致 */
    ctx->str_vendor  = "qcom-iris v4l2-m2m";
    ctx->max_profiles = 8;
    ctx->max_entrypoints = 2;
    ctx->max_attributes = 8;
    ctx->max_image_formats = 4;
    ctx->max_subpic_formats = 0;
    ctx->max_display_attributes = 0;
    ctx->vtable = &iris_vtable;

    drv->drm_fd = ctx->drm_fd;      /* 不要 close，libva 负责 */
    return iris_probe(drv);          /* QUERYCAP + ENUM_FMT，失败返回 VA_STATUS_ERROR_OPERATION_FAILED */
}
```

要点：

- 用 `nm -D iris_drv_video.so | grep vaDriverInit` 确认符号导出（`VA_EXPORT` = `__attribute__((visibility("default")))`）。
- `ctx->drm_fd` 是 libva 打开的 render node（`VA_DISPLAY_DRM`）。**不要重复 `open` 同一个节点**，也不要 `close` 它。
- 如果 `vaGetDisplay()` 走的是 Wayland/X11 display，`ctx->display_type` 会是 `VA_DISPLAY_WAYLAND`/`VA_DISPLAY_X11`；本项目只支持 `VA_DISPLAY_DRM`，其它类型在 `vaInitialize` 里返回 `VA_STATUS_ERROR_INVALID_DISPLAY`（Firefox/ffmpeg 都用 DRM，够用）。

**`iris_probe()` 具体做什么**

```c
struct v4l2_capability cap;
ioctl(fd, VIDIOC_QUERYCAP, &cap);
/* 必须满足 */
assert(cap.device_caps & V4L2_CAP_VIDEO_M2M_MPLANE);
assert(cap.device_caps & V4L2_CAP_STREAMING);
/* 记录 driver/card/bus_info 用于日志 */

/* OUTPUT：枚举压缩格式 */
for (i = 0; ; i++) {
    struct v4l2_fmtdesc d = { .index = i, .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE };
    if (ioctl(fd, VIDIOC_ENUM_FMT, &d) < 0) break;
    /* 期望看到 V4L2_PIX_FMT_H264 / V4L2_PIX_FMT_HEVC / V4L2_PIX_FMT_VP9 */
    /* d.flags & V4L2_FMT_FLAG_DYN_RESOLUTION 记下来，用于决定是否订阅 SOURCE_CHANGE */
}

/* CAPTURE：枚举原始格式 */
for (i = 0; ; i++) {
    struct v4l2_fmtdesc d = { .index = i, .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE };
    if (ioctl(fd, VIDIOC_ENUM_FMT, &d) < 0) break;
    /* 期望只有 V4L2_PIX_FMT_NV12；若将来出现 P010/NV12_10 再扩展 */
}

/* 每个候选 (codec, profile) 用 TRY_FMT 验证一次，避免"枚举支持但实际不能用" */
```

**为什么不每次查询都进内核**：`vaQueryConfigProfiles` 可能被客户端在启动时调用多次，而 `VIDIOC_ENUM_FMT`/`TRY_FMT` 在解码器已 STREAMON 时行为受限；缓存到 `drv->caps` 里最稳。

**`vaGetConfigAttributes` 返回值**

| attrib | 值 | 说明 |
|---|---|---|
| `VAConfigAttribRTFormat` | `VA_RT_FORMAT_YUV420` | 只支持 8bit 4:2:0；10bit 见 §5.2 |
| `VAConfigAttribDecSliceMode` | `VA_DEC_SLICE_MODE_NORMAL` | 不支持 `VA_DEC_SLICE_MODE_BASE`（那是 stateless 的 slice 输出模式） |
| `VAConfigAttribDecProcessing` | `VA_DEC_PROCESSING_NONE` | 不做逐帧后处理 |
| `VAConfigAttribMaxPictureWidth/Height` | 用 `VIDIOC_ENUM_FRAMESIZES` 或 `TRY_FMT` 探测（如 4096×2304） | 探测失败就给 1920×1088 保守值 |

> 注意：`VAConfigAttribDecSliceMode` 的语义是"驱动是否支持按 slice 提交"，不是"码流里有没有多 slice"。我们支持多 slice 码流，但提交单位始终是"一帧一个 AU"，所以填 `NORMAL`。

### 3.3 `vaCreateContext`：V4L2 管道的完整建立序列

这是整个驱动最关键的 100 行。顺序不能错：

```c
int fd = open("/dev/video0", O_RDWR | O_NONBLOCK | O_CLOEXEC);

/* 1. 订阅分辨率变化事件（必须在 STREAMON 之前） */
struct v4l2_event_subscription sub = { .type = V4L2_EVENT_SOURCE_CHANGE };
ioctl(fd, VIDIOC_SUBSCRIBE_EVENT, &sub);

/* 2. OUTPUT 格式：压缩码流 */
struct v4l2_format ofmt = { .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE };
ofmt.fmt.pix_mp.width       = ctx_width;     /* 来自 vaCreateContext 的 picture_width */
ofmt.fmt.pix_mp.height      = ctx_height;
ofmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_H264;  /* 按 profile 选 H264/HEVC/VP9 */
ofmt.fmt.pix_mp.field       = V4L2_FIELD_NONE;
ofmt.fmt.pix_mp.num_planes  = 1;
ofmt.fmt.pix_mp.plane_fmt[0].sizeimage = 4 * 1024 * 1024;  /* 单帧 AU 上限，见下 */
ioctl(fd, VIDIOC_S_FMT, &ofmt);

/* 3. CAPTURE 格式：NV12 */
struct v4l2_format cfmt = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE };
cfmt.fmt.pix_mp.width       = ctx_width;
cfmt.fmt.pix_mp.height      = ctx_height;
cfmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
cfmt.fmt.pix_mp.num_planes  = 1;              /* NV12 是单 plane 格式，Y/UV 在同一 buffer */
ioctl(fd, VIDIOC_S_FMT, &cfmt);
/* S_FMT 之后 cfmt 被驱动改写：以 cfmt.fmt.pix_mp.plane_fmt[0].bytesperline 为准 */

/* 4. 队列深度 */
struct v4l2_control c;
c.id = V4L2_CID_MIN_BUFFERS_FOR_CAPTURE;
ioctl(fd, VIDIOC_G_CTRL, &c);           /* c.value 通常是 4 左右 */
int cap_count = MAX(c.value + 4, surface_count_hint);
cap_count = MIN(cap_count, 32);         /* vb2 上限，具体以 REQBUFS 返回值为准 */

/* 5. 申请 buffer */
struct v4l2_requestbuffers req = {
    .count = 4, .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE, .memory = V4L2_MEMORY_MMAP };
ioctl(fd, VIDIOC_REQBUFS, &req);
/* 对每个 index：QUERYBUF → mmap → 记录 length/offset */

req.count = cap_count; req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
req.memory = V4L2_MEMORY_MMAP;          /* 或 V4L2_MEMORY_DMABUF，见 §4.1 */
ioctl(fd, VIDIOC_REQBUFS, &req);
/* 对每个 index：QUERYBUF → mmap → EXPBUF 拿 dma-buf fd */

/* 6. 开流 */
enum v4l2_buf_type t = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE;  ioctl(fd, VIDIOC_STREAMON, &t);
t = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;                    ioctl(fd, VIDIOC_STREAMON, &t);
```

几个容易踩的点：

- **`sizeimage` 必须够大**。一帧 1080p H.264 通常 < 1 MB，但 4K 高码率 + 多 slice 可能到几 MB。建议 4 MiB；若 `QBUF` 返回 `EINVAL`/`ENOSPC`，先怀疑这个。也可以按 `width*height*2` 动态算，下限 2 MiB。
- **CAPTURE 的 `bytesperline` 由驱动决定**，Qualcomm 可能按 128/256 对齐。**不要假设它等于 width**，导出 dma-buf 时必须用驱动返回的 `bytesperline` 和 `sizeimage`。
- **`num_planes`**：`V4L2_PIX_FMT_NV12` 在 MPLANE 队列上仍然是 1 个 plane（Y 和 UV 连续存放）。不要按 2 个 plane 去填 `plane_fmt[1]`。
- **延迟 STREAMON 也可以**：如果 surface 池大小要到第一次 `vaCreateSurfaces` 才知道，就把第 5、6 步推迟到第一次 `vaEndPicture`。代价是错误定位晚一点；建议 MVP 先按"`vaCreateContext` 时就固定池大小"，池大小取 `vaCreateContext` 的 hint 或 16，L2 再做动态扩容。
- **`VIDIOC_REQBUFS` 返回的 `count` 可能小于请求值**，以返回值为准；如果小于 `min_buffers`，直接 `VA_STATUS_ERROR_ALLOCATION_FAILED`。

**`vaCreateContext` 的 `picture_width/height` 是 coded size**。ffmpeg 8.0.x 传的是 `coded_width/coded_height`（1080p 通常是 1920×1088），不是显示尺寸 1920×1080。**不要拿显示尺寸去 `S_FMT`**，否则 CAPTURE 会被驱动改成 1088（或报错），而 surface 按 1080 分配就会越界。

### 3.4 解码路径：`vaBeginPicture` → `vaRenderPicture` → `vaEndPicture` → `vaSyncSurface`

**`vaBeginPicture(dpy, ctx, surface)`**

```c
surf = surface_lookup(surface_id);
if (surf->state != SURFACE_FREE) return VA_STATUS_ERROR_SURFACE_BUSY;
surf->state = SURFACE_DECODING;
ctx->cur_surface = surf;
ctx->au_len = 0;                 /* 清空 AU 累积缓冲 */
ctx->have_pic_param = false;
ctx->slice_count = 0;
```

**`vaRenderPicture(dpy, ctx, buffers, num)`**

按 `VABufferID` 取出类型，分派：

| `VABufferType` | 处理 |
|---|---|
| `VAPictureParameterBufferType` | 保存指针/拷贝到 `ctx->pic_param`（H.264 还要顺手取 SPS/PPS 字段） |
| `VASliceParameterBufferType` | 保存 slice 元数据（MVP 只用于校验，不参与组包） |
| `VASliceDataBufferType` | **追加到 AU 缓冲**（补起始码，见 §3.6） |
| `VAQMatrixBufferType` | 保存 scaling list（H.264 PPS 需要） |
| `VAProbabilityBufferType` / `VAIQMatrixBufferType` | 忽略（非 H.264） |
| 其它 | 返回 `VA_STATUS_ERROR_UNSUPPORTED_BUFFERTYPE` |

ffmpeg 的调用顺序是：picture param → qmatrix → slice param → slice data（可能多次），全部在同一个 `vaBeginPicture`/`vaEndPicture` 之间。**不要假设只有一个 slice data buffer**。

**`vaEndPicture(dpy, ctx)`**

```c
/* 1. 组包：AUD + SPS + PPS + slices（H.264），见 §3.6 */
au = build_access_unit(ctx, &au_len);
if (!au_len) { /* 空帧（如 EOS flush 的最后一次调用）*/ return VA_STATUS_SUCCESS; }

/* 2. 取一个空闲 OUTPUT buffer；没有就 poll 等 */
idx = output_get_free(ctx, /*block=*/true);

/* 3. 拷贝 + 入队 */
memcpy(ctx->out_va[idx], au, au_len);
struct v4l2_buffer b = {
    .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE, .memory = V4L2_MEMORY_MMAP,
    .index = idx, .bytesused = au_len, .field = V4L2_FIELD_NONE,
};
b.timestamp.tv_sec  = ctx->frame_seq / 1000000;
b.timestamp.tv_usec = ctx->frame_seq % 1000000;   /* 关键：frame_seq 作为关联键 */
ioctl(ctx->fd, VIDIOC_QBUF, &b);

/* 4. 记录 surface 与 frame_seq 的映射 */
queue_push_submitted(ctx, ctx->frame_seq, surf);
ctx->frame_seq++;
surf->state = SURFACE_DECODING;
```

`bytesused` 对 OUTPUT 队列是**有效码流字节数**，不是 buffer 大小。多平面时填 `plane_fmt[0].bytesused`（`v4l2_buffer.m.planes[0].bytesused`）。

**`vaSyncSurface(dpy, surface)`**

```c
surf = surface_lookup(surface_id);
while (!surf->decoded) {
    if (poll(ctx->fd, POLLIN, timeout_ms) <= 0) return VA_STATUS_ERROR_OPERATION_FAILED;
    drain_capture(ctx);          /* 内部循环 DQBUF，直到 EAGAIN */
    drain_events(ctx);           /* DQEVENT：SOURCE_CHANGE 在这里处理 */
}
return surf->error ? VA_STATUS_ERROR_DECODING_ERROR : VA_STATUS_SUCCESS;
```

`drain_capture()`：

```c
for (;;) {
    struct v4l2_buffer b = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
                             .memory = V4L2_MEMORY_MMAP };
    struct v4l2_plane planes[1] = {0};
    b.m.planes = planes; b.length = 1;
    if (ioctl(fd, VIDIOC_DQBUF, &b) < 0) {
        if (errno == EAGAIN) break;
        /* EIO 等：标记 pipeline 错误，唤醒所有等待者 */
    }
    surf = queue_pop_by_timestamp(ctx, ts_to_seq(b.timestamp));
    if (b.flags & V4L2_BUF_FLAG_ERROR) surf->error = true;
    surf->decoded = true;
    surf->state = SURFACE_READY;
    capture_return(ctx, b.index);      /* 若走 EXPBUF 别名，这里不能立即复用，见 §4.1 */
    if (b.flags & V4L2_BUF_FLAG_LAST) ctx->draining = false;
}
```

**`vaQuerySurfaceStatus`**：按内部状态返回：

| 内部状态 | `VASurfaceStatus` |
|---|---|
| 刚 `vaCreateSurfaces` | `VASurfaceReady` |
| `vaBeginPicture` 后、未 `vaSyncSurface` | `VASurfaceRendering`（或 `VASurfaceDecoding`，若头文件有） |
| `vaSyncSurface` 成功 | `VASurfaceReady` |
| 解码错误 | `VASurfaceSkipped`（并通过 `vaQuerySurfaceError` 返回 `VA_STATUS_ERROR_DECODING_ERROR`） |

> `VASurfaceStatus` 枚举在不同 libva 版本里有 `VASurfaceReady/VASurfaceRendering/VASurfaceDisplaying/VASurfaceSkipped` 四种；ffmpeg 只用它判断"是否可以取帧"，`Ready`/`Rendering` 的区分不严格，按上表映射即可。

### 3.5 导出与图像：`vaExportSurfaceHandle`、`vaDeriveImage`

**`vaExportSurfaceHandle`**（详见 §4.4）是 Firefox 路径的**必需项**。实现要点：

- 支持 `flags = VA_EXPORT_SURFACE_READ_ONLY | VA_EXPORT_SURFACE_SEPARATE_LAYERS`（ffmpeg 就是这么调的）；
- `desc.objects[0].fd` 必须是**新 dup 的 fd**（调用方负责 close）；
- NV12 用 **2 个 layer**：Y = `DRM_FORMAT_R8`，UV = `DRM_FORMAT_GR88`，同一个 object，不同 offset；
- `drm_format_modifier` 填 `DRM_FORMAT_MOD_LINEAR`（0），除非确认驱动输出的是 Qualcomm tiled 格式（见 §4.2）；
- `width/height` 填 surface 的 coded 尺寸（如 1920×1088），`pitch` 填 `bytesperline`。

**`vaDeriveImage`**：给 `ffmpeg -vf hwdownload` 用。

```c
VAImage img = {0};
img.image_id = ++ctx->image_seq;
img.format.fourcc = VA_FOURCC_NV12;
img.format.byte_order = VA_LSB_FIRST;
img.format.bits_per_pixel = 12;
img.width = surf->width; img.height = surf->height;
img.data_size = surf->size;
img.num_planes = 2;
img.pitches[0] = surf->stride;
img.pitches[1] = surf->stride;
img.offsets[0] = 0;
img.offsets[1] = surf->stride * surf->height;   /* 以驱动实际 UV 偏移为准 */
img.buf = create_internal_buffer(ctx, img.data_size);  /* vaMapBuffer 能映射到 dma-buf */
```

`vaMapBuffer(img.buf)` 返回 dma-buf 的 CPU 映射（`mmap` 一次，缓存），`vaUnmapBuffer` 只减引用。若 CPU 要读写，成对使用 `DMA_BUF_IOCTL_SYNC`（`DMA_BUF_SYNC_START/END`）避免非一致性架构上的缓存问题。

**`vaGetImage`/`vaPutImage`**：MVP 可以返回 `VA_STATUS_ERROR_UNIMPLEMENTED`；`ffmpeg` 的 hwdownload 优先走 `vaDeriveImage`，走不到才回退 `vaGetImage`。

**`vaPutSurface`**：不实现，返回 `VA_STATUS_ERROR_UNIMPLEMENTED`。Firefox 与 `mpv --hwdec=vaapi --vo=gpu-next` 都不调用它（它们走 export + EGL）。

### 3.6 码流组装（本项目的核心算法）

#### 3.6.1 H.264 的 AU 结构

VA-API 客户端每调用一次 `vaBeginPicture`/`vaEndPicture`，驱动就产出一个 **access unit（AU）**：

```
00 00 00 01 09 F0                  ← AUD，primary_pic_type=7（任意类型）
00 00 00 01 67 <SPS RBSP>          ← SPS（nal_ref_idc=3, nal_unit_type=7）
00 00 00 01 68 <PPS RBSP>          ← PPS（nal_ref_idc=3, nal_unit_type=8）
00 00 01 <slice NAL，原样>          ← 来自 VASliceDataBufferType
00 00 01 <slice NAL，原样>          ← 多 slice 依次追加
```

规则：

1. **一个 `vaEndPicture` = 一个 AU = 一次 `QBUF`**。多 slice 全部拼进同一个 buffer。
2. **slice NAL 原样透传**：`VASliceDataBufferType` 的内容是"NAL header + slice header + slice data"（不含起始码）。驱动只补 `00 00 01`。若客户端已经带了起始码（检测前 3/4 字节），不要重复添加。
3. **SPS/PPS 每个 AU 都重发**（最简单、最稳）。固件会忽略重复参数集；如果实测有性能问题，再优化成"参数变化或 IDR 前才发"。
4. **AUD 建议发**：帮固件的 AU 边界检测，成本 6 字节。若实测固件对 AUD 敏感（少数实现会），做成可配置。
5. **不要重写 slice header**：slice header 里的 `frame_num`、POC、参考列表、QP 等必须与原始码流一致，重写必错。驱动只做"补起始码"和"补参数集"。

#### 3.6.2 SPS 字段映射（H.264）

| SPS 语法元素 | 驱动取值来源 | 说明 |
|---|---|---|
| `profile_idc` | config 的 VAProfile（Baseline=66 / Main=77 / High=100） | VA 的 picture param 不直接给 profile_idc |
| `constraint_set0..5_flags` | 全 0 | 保守；Baseline 的 constraint_set0 不写也能解 |
| `level_idc` | 按 Annex A 计算最小 level（见下） | 写大了通常无害，写小了固件可能拒 |
| `seq_parameter_set_id` | 0 | 固定 |
| `chroma_format_idc` | `seq_fields.bits.chroma_format_idc`（仅 High 系写） | 8bit 4:2:0 = 1 |
| `bit_depth_luma_minus8` / `chroma` | 同名参数 | MVP 恒 0 |
| `qpprime_y_zero_transform_bypass_flag` | 0 | |
| `seq_scaling_matrix_present_flag` | 0 | 缩放矩阵放 PPS |
| `log2_max_frame_num_minus4` | `seq_fields.bits.log2_max_frame_num_minus4` | |
| `pic_order_cnt_type` | `seq_fields.bits.pic_order_cnt_type` | |
| `log2_max_pic_order_cnt_lsb_minus4` | 同名字段（type=0 时） | |
| `delta_pic_order_always_zero_flag` | 同名字段（type=1 时） | |
| `offset_for_non_ref_pic` / `offset_for_top_to_bottom_field` | 0 / 0 | type=1 且 VA 未提供时按 0 |
| `num_ref_frames_in_pic_order_cnt_cycle` | `num_ref_frames_in_pic_order_cnt_cycle` | 通常 0 |
| `max_num_ref_frames` | `num_ref_frames` | |
| `gaps_in_frame_num_value_allowed_flag` | `seq_fields.bits.gaps_in_frame_num_value_allowed_flag` | |
| `pic_width_in_mbs_minus1` | `picture_width_in_mbs_minus1` | |
| `pic_height_in_map_units_minus1` | 由 `picture_height_in_mbs_minus1` 与 `frame_mbs_only_flag` 推导 | |
| `frame_mbs_only_flag` | 恒 1 | **不支持场编码/PAFF/MBAFF**，遇到就返回 `VA_STATUS_ERROR_DECODING_ERROR` |
| `mb_adaptive_frame_field_flag` | 0 | |
| `direct_8x8_inference_flag` | 1 | |
| `frame_cropping_flag` | 0（见 §3.7） | |
| `vui_parameters_present_flag` | 0 | 无 VUI 不影响解码 |

**level_idc 的计算**（写死 51 也能跑，但按规范算更稳）：

```c
int pick_level(int w_mbs, int h_mbs, int max_refs) {
    int pic_size = w_mbs * h_mbs;                 /* PicSizeInMbs */
    int max_dpb  = pic_size * (max_refs + 1);     /* MaxDpbMbs 需求 */
    static const struct { int level, max_fs, max_dpb_mbs; } t[] = {
        {30, 1620, 8100}, {31, 3600, 18000}, {32, 5120, 20480},
        {40, 8192, 32768}, {41, 8192, 32768}, {42, 8704, 34816},
        {50, 22080, 110400}, {51, 36864, 184320}, {52, 36864, 184320},
    };
    for (i = 0; i < n; i++)
        if (pic_size <= t[i].max_fs && max_dpb <= t[i].max_dpb_mbs)
            return t[i].level;
    return 51;
}
```

#### 3.6.3 PPS 字段映射（H.264）

**PPS 的 `pic_parameter_set_id` 必须与 slice header 里的一致。** VA-API 不直接给这个值，但可以从 slice NAL 的头部解析出来：`first_mb_in_slice`（ue）、`slice_type`（ue）、`pic_parameter_set_id`（ue）——只需要三个 ue(v)，不依赖 PPS 内容。

| PPS 语法元素 | 来源 |
|---|---|
| `pic_parameter_set_id` | 从 slice NAL 解析（见上）；若同一 AU 内多个 slice 的 pps_id 不同 → 拒绝 |
| `seq_parameter_set_id` | 0 |
| `entropy_coding_mode_flag` | `pic_fields.bits.entropy_coding_mode_flag` |
| `bottom_field_pic_order_in_frame_present_flag` | `pic_fields.bits.pic_order_present_flag` |
| `num_slice_groups_minus1` | 0（不支持 FMO） |
| `num_ref_idx_l0_default_active_minus1` / `l1` | 同名字段 |
| `weighted_pred_flag` / `weighted_bipred_idc` | 同名字段 |
| `pic_init_qp_minus26` / `pic_init_qs_minus26` | 同名字段 |
| `chroma_qp_index_offset` | 同名字段 |
| `deblocking_filter_control_present_flag` | 1 |
| `constrained_intra_pred_flag` | 同名字段 |
| `redundant_pic_cnt_present_flag` | 0 |
| `transform_8x8_mode_flag` | `pic_fields.bits.transform_8x8_mode_flag` |
| `pic_scaling_matrix_present_flag` + 列表 | 由 `VAQMatrixBufferH264` 的 `scaling_lists_4x4[6][16]` / `scaling_lists_8x8[2][64]` 决定：全 16/64 就置 0，否则置 1 并写入 |
| `second_chroma_qp_index_offset` | 同名字段 |

#### 3.6.4 RBSP 写入与防竞争字节

```c
struct bs { uint8_t *buf; size_t cap, len; uint32_t bitpos; };

void bs_put_bits(struct bs *b, uint32_t val, int n);   /* 高位先写 */
void bs_put_ue(struct bs *b, uint32_t v);              /* Exp-Golomb */
void bs_put_se(struct bs *b, int32_t v);
void bs_rbsp_trailing(struct bs *b);                   /* 补 1 后对齐 0 */

/* 生成 NAL：先写 1 字节 NAL header，再写 RBSP 载荷，最后插 EPB */
size_t nal_wrap(uint8_t *out, int nal_ref_idc, int type,
                const uint8_t *rbsp, size_t rbsp_len)
{
    /* 00 00 00/01/02/03 之后插入 0x03 */
}
```

**单元测试方法（不需要硬件）**：把重建器产出的 SPS/PPS + 原始 slice NAL 拼成 `.h264` 文件，用 `ffmpeg -f h264 -i synth.h264 -f null -` 软解。若软解器接受且帧数与参考一致，说明语法正确。这个测试可以在 x86 PC 上跑，是码流重建器的主要开发手段（§9.2）。

#### 3.6.5 HEVC 与 VP9 的可行性

| Codec | VA 提供的参数 | 需要重建 | 结论 |
|---|---|---|---|
| H.264 | picture/slice/qmatrix，字段齐全 | SPS + PPS | **可行**（本项目主路线） |
| HEVC | `VAPictureParameterBufferHEVC` + `VASliceParameterBufferHEVC` | VPS + SPS（含 RPS）+ PPS | **高风险**：SPS 的 short-term RPS 若在 VA 参数里没有完整表达，slice header 的 `short_term_ref_pic_set_sps_flag=1` 就无法匹配。需核对 `va_dec_hevc.h`；核对不通过则 HEVC 只能走 R2 直通 |
| VP9 | `VADecPictureParameterBufferVP9`（uncompressed header 全字段） | VP9 uncompressed header | **可行**，但工作量大；压缩头/tile 数据在 slice data 里原样透传 |

**HEVC 的具体检查点**（M0 做）：在 `va_dec_hevc.h` 里找 `num_short_term_ref_pic_sets`、`st_rps`、`st_rps_bits` 之类的字段，并确认 `VASliceParameterBufferHEVC` 是否带 `slice_pic_parameter_set_id`。如果 RPS 信息不全，HEVC 阶段直接改用 R2。

### 3.7 coded size、裁剪与显示尺寸

H.264 的 coded 高度是 16 的倍数：1080p → 1088。VA surface 通常按 coded size 分配（ffmpeg 传 `coded_width/coded_height`），而显示尺寸是 1920×1080。三条路：

1. **SPS 写 `frame_cropping_flag=0`**（推荐 MVP）：固件输出 1088 行，surface 也是 1088 行；ffmpeg 的 `AVFrame` 通常带 `width=1920,height=1080`（显示尺寸），crop 由客户端处理。**必须实测确认 Firefox 的 `DMABufSurface` 用哪个高度建纹理**——若它用 descriptor 的 `height`（1088），底部会有 8 行绿边，需要在导出 descriptor 时填显示高度，或改用 cropping。
2. **从参数推导 cropping**：`frame_cropping_flag=1` + `frame_crop_bottom_offset=(coded_h - display_h)/2`。但 VA-API 的 H.264 picture param 是否提供显示高度需要核对；若没有，只能用 `vaCreateContext` 的 height（coded）与客户端后续行为推断，不可靠。
3. **在导出路径裁剪**：descriptor 的 `height` 填显示高度、`pitch` 不变。ffmpeg 的 `AVDRMFrameDescriptor` 会带上 `height`，Firefox 按它建纹理。这条路不依赖 SPS，最可控。

**建议：MVP 用方案 1 + 方案 3 兜底**，并在 §9.3 加一个"1080p 底部 8 行像素"的专项检查（用 `ffmpeg` 导出首帧 PNG，肉眼/像素比对）。

### 3.8 错误码映射

| 场景 | 返回 |
|---|---|
| V4L2 `open`/`QUERYCAP` 失败 | `VA_STATUS_ERROR_OPERATION_FAILED` |
| profile 不支持（如 `VAProfileHEVCMain10`） | `VA_STATUS_ERROR_UNSUPPORTED_PROFILE` |
| entrypoint 不是 VLD | `VA_STATUS_ERROR_UNSUPPORTED_ENTRYPOINT` |
| `rt_format` 不是 YUV420 | `VA_STATUS_ERROR_UNSUPPORTED_RT_FORMAT` |
| `vaCreateConfig` 参数非法 | `VA_STATUS_ERROR_INVALID_PARAMETER` |
| `REQBUFS`/`mmap` 失败 | `VA_STATUS_ERROR_ALLOCATION_FAILED` |
| `QBUF` 返回 `EINVAL` | `VA_STATUS_ERROR_INVALID_PARAMETER`（先查 `sizeimage`/`bytesused`） |
| `DQBUF` 带 `V4L2_BUF_FLAG_ERROR` | `VA_STATUS_ERROR_DECODING_ERROR` |
| `SOURCE_CHANGE` 后新分辨率 > surface 尺寸 | `VA_STATUS_ERROR_RESOLUTION_NOT_SUPPORTED` |
| `vaSyncSurface` 超时（默认 2 s） | `VA_STATUS_ERROR_OPERATION_FAILED`（同时把 pipeline 标为 error，后续调用快速失败） |
| 不支持的 slice data 分片（`slice_data_flag != ALL`） | `VA_STATUS_ERROR_FLAG_NOT_SUPPORTED` |
| 场编码 / FMO / 多 PPS 同一 AU | `VA_STATUS_ERROR_DECODING_ERROR` |

---

## 4. 缓冲区与零拷贝

### 4.1 两条队列的内存类型

| 队列 | 建议 memory | 数量 | 单 buffer 大小 | 理由 |
|---|---|---|---|---|
| OUTPUT（码流） | `V4L2_MEMORY_MMAP` | 4 | 2–4 MiB | 码流小、CPU 本来就要写，DMABUF 没有收益 |
| CAPTURE（帧）MVP 默认 | `V4L2_MEMORY_MMAP` + memcpy 到 surface 自有 dma-buf | 8 | `bytesperline*height*3/2` | 最稳：surface 与 V4L2 buffer 解耦，context 生命周期/seek 都不牵连（ADR-1，§6.1） |
| CAPTURE（帧）L2 优化 | `V4L2_MEMORY_MMAP` + `VIDIOC_EXPBUF` 别名 | 8–32 | 同上 | 零拷贝：V4L2 buffer 直接当 VA surface 导出；代价是 surface 数 = CAPTURE buffer 数（创建顺序有约束，见 §6.1） |
| CAPTURE（备选） | `V4L2_MEMORY_DMABUF` + gbm 分配的 NV12 bo | 同上 | 同上 | 若 `EXPBUF` 与拷贝都不合适时的第三条路 |

**为什么 MVP 不直接做零拷贝**：VA surface 是 display 级对象，`vaCreateSurfaces` 时还不知道 codec（codec 在 `vaCreateContext` 才确定），而 CAPTURE 队列的 `S_FMT`/`REQBUFS` 依赖 codec。顺序对不上，强行做会让 `vaCreateSurfaces` 和 `vaCreateContext` 互相依赖。拷贝模式用一次 memcpy 换掉这个耦合，1080p30 的代价约 93 MB/s，可接受（§6.1 ADR-1）。

**L2 为什么优先 `MMAP + EXPBUF` 而不是 gbm**：

- vb2 的 MMAP buffer 由内核分配，`VIDIOC_EXPBUF` 可以把它导出成 dma-buf（需要 memops 实现 `get_dmabuf`，iris 用 `vb2_dma_sg`，**待验证**）；
- 导出的 dma-buf 和 V4L2 正在写的内存是**同一块物理内存**，VA surface 直接别名它，`vaExportSurfaceHandle` 只是 `dup` fd，零拷贝；
- 不需要 gbm 参与，少一层分配器依赖。

**必须先验证的三件事**（M1 用 `tools/v4l2-probe.c` 做）：

```c
/* 1. EXPBUF 能不能用 */
struct v4l2_exportbuffer exp = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
                                 .index = 0, .plane = 0, .flags = O_RDWR };
if (ioctl(fd, VIDIOC_EXPBUF, &exp) < 0) perror("EXPBUF");   /* ENOTTY/EINVAL → 走备选 */

/* 2. 导出的 fd 能不能 mmap（能 mmap 才能 vaDeriveImage） */
void *p = mmap(NULL, len, PROT_READ, MAP_SHARED, exp.fd, 0);
if (p == MAP_FAILED) perror("mmap dmabuf");

/* 3. 导出 fd 的 size / modifier */
struct stat st; fstat(exp.fd, &st);
/* Qualcomm 若输出 tiled 格式，modifier 不会体现在 fd 上；需要查 iris 的 S_FMT 是否接受 linear 标志 */
```

如果 `EXPBUF` 不可用，退回"MMAP + memcpy"，并在 §9.6 记录拷贝开销；不要一开始就上 gbm，gbm 的 NV12 支持在 msm 上也要验证。

### 4.2 surface 的 dma-buf 从哪来

若走 `EXPBUF` 别名：surface 没有自己的 dma-buf，`surface->dmabuf_fd = exp.fd`，`surface->stride/offset/size` 全部来自 `VIDIOC_QUERYBUF` 的 `plane_fmt`。

若走拷贝路径：surface 在 `vaCreateSurfaces` 时分配一块 NV12 dma-buf。三种分配方式，按优先级：

```c
/* 方式 1：gbm（推荐，Mesa 栈里最自然） */
struct gbm_device *gbm = gbm_create_device(ctx->drm_fd);
struct gbm_bo *bo = gbm_bo_create(gbm, w, h, GBM_FORMAT_NV12,
                                  GBM_BO_USE_LINEAR | GBM_BO_USE_RENDERING);
int fd = gbm_bo_get_fd(bo);
uint32_t stride = gbm_bo_get_stride(bo);
uint64_t mod    = gbm_bo_get_modifier(bo);

/* 方式 2：DRM dumb buffer（linear 保证，但某些 GPU 不能采样） */
struct drm_mode_create_dumb c = { .width = w, .height = h * 3 / 2, .bpp = 8 };
ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &c);
ioctl(drm_fd, DRM_IOCTL_PRIME_HANDLE_TO_FD, &prime);

/* 方式 3：udmabuf（需要 /dev/udmabuf 与内核 CONFIG_UDMABUF） */
```

**关键决策：强制 LINEAR。** 原因：

- Firefox 的 EGL 导入路径对 modifier 的支持取决于 Mesa 版本；`DRM_FORMAT_MOD_LINEAR` 是最保守、最可移植的选择；
- Qualcomm 的 tiled NV12（`QCOM_COMPRESSED`/`UBWC`）需要 freedreno 与 iris 都协商一致，MVP 不碰；
- 拷贝路径下内存是我们自己分配的，天然 linear。

**对齐要求**（避免 EGL 导入失败）：

- `stride` 16 字节对齐（推荐 256，兼顾性能和兼容）；
- UV 平面偏移 `stride * height`，`height` 用 coded 高度（1088）以保证 4K/1080p 都够；
- 总大小 ≥ `stride * height * 3 / 2`，向上取整到 4096。

### 4.3 surface 对象与 `vaCreateSurfaces`

```c
struct iris_surface {
    VASurfaceID id;
    uint32_t width, height;        /* coded */
    uint32_t stride, y_offset, uv_offset, size;
    int dmabuf_fd;                 /* 拥有或别名 */
    uint64_t modifier;
    enum { SURFACE_FREE, SURFACE_DECODING, SURFACE_READY, SURFACE_ERROR } state;
    uint64_t frame_seq;            /* 提交时绑定 */
    bool exported;                 /* 已 dup 给客户端 */
};
```

`vaCreateSurfaces` 要处理两类请求：

1. **驱动分配**（Firefox/ffmpeg 解码的常态）：`attrib_list` 里通常只有 `VASurfaceAttribPixelFormat = VA_FOURCC_NV12`。别名模式下从 CAPTURE 池取一个空闲 buffer；MVP 的拷贝模式则按 §4.2 新分配一块 dma-buf，填 surface。
2. **外部内存导入**（`ffmpeg -hwaccel vaapi` 的某些路径、以及 `vaPutSurface` 类用法）：客户端给 `VASurfaceAttribMemoryType = VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` + `VASurfaceAttribExternalBuffers`。此时：

```c
VASurfaceAttribExternalBuffers *ext = attrib->value.pExtBuf;
/* ext->buffers[0] 是 dma-buf fd，ext->pitches[0]/offsets[0]/data_size 已给 */
/* 直接 mmap 或 dup，surface 不参与 CAPTURE 池 */
```

**MVP 可以只支持第 1 类**，第 2 类返回 `VA_STATUS_ERROR_UNSUPPORTED_MEMORY_TYPE`（先确认 Firefox 不走这条路；ffmpeg 的 VA-API 解码器默认走第 1 类）。

`vaQuerySurfaceAttributes` 返回：

```c
VASurfaceAttrib attrs[] = {
    { .type = VASurfaceAttribPixelFormat, .flags = VA_SURFACE_ATTRIB_GETTABLE,
      .value.type = VAGenericValueTypeInteger, .value.value.i = VA_FOURCC_NV12 },
    { .type = VASurfaceAttribMinWidth,  ... .i = 16 },
    { .type = VASurfaceAttribMaxWidth,  ... .i = 4096 },
    { .type = VASurfaceAttribMinHeight, ... .i = 16 },
    { .type = VASurfaceAttribMaxHeight, ... .i = 2304 },
    { .type = VASurfaceAttribMemoryType, .flags = VA_SURFACE_ATTRIB_GETTABLE,
      .value.value.i = VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2 },
};
```

### 4.4 `vaExportSurfaceHandle` 的实现

```c
VAStatus iris_ExportSurfaceHandle(VADriverContextP ctx, VASurfaceID sid,
                                  uint32_t mem_type, uint32_t flags, void *descriptor)
{
    if (mem_type != VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2)
        return VA_STATUS_ERROR_UNSUPPORTED_MEMORY_TYPE;
    if (flags & ~(VA_EXPORT_SURFACE_READ_ONLY | VA_EXPORT_SURFACE_SEPARATE_LAYERS))
        return VA_STATUS_ERROR_FLAG_NOT_SUPPORTED;

    struct iris_surface *s = surface_lookup(ctx, sid);
    if (!s) return VA_STATUS_ERROR_INVALID_SURFACE;
    iris_sync_surface(ctx, sid);          /* 幂等：确保已 DQBUF */

    VADRMPRIMESurfaceDescriptor *d = descriptor;
    memset(d, 0, sizeof(*d));
    d->fourcc = VA_FOURCC_NV12;
    d->width  = s->width;
    d->height = s->height;                /* 裁剪策略见 §3.7 */
    d->num_objects = 1;
    d->objects[0].fd = dup(s->dmabuf_fd);
    d->objects[0].size = s->size;
    d->objects[0].drm_format_modifier = s->modifier;

    d->num_layers = 2;
    d->layers[0].drm_format = DRM_FORMAT_R8;
    d->layers[0].num_planes = 1;
    d->layers[0].object_index[0] = 0;
    d->layers[0].offset[0] = s->y_offset;
    d->layers[0].pitch[0]  = s->stride;

    d->layers[1].drm_format = DRM_FORMAT_GR88;
    d->layers[1].num_planes = 1;
    d->layers[1].object_index[0] = 0;
    d->layers[1].offset[0] = s->uv_offset;
    d->layers[1].pitch[0]  = s->stride;

    return VA_STATUS_SUCCESS;
}
```

**注意**：

- `dup()` 出来的 fd 由调用方关闭；不要在导出后关闭 surface 自己的 fd。
- 若 surface 还没 ready，**在导出里做一次同步**（ffmpeg 有时不显式 `vaSyncSurface` 就 map），避免导出半成品。
- `VA_EXPORT_SURFACE_WRITE_ONLY` 不支持（解码输出只读）。
- 如果客户端要求 `COMPOSED_LAYERS`，可以返回 `VA_STATUS_ERROR_FLAG_NOT_SUPPORTED`；ffmpeg 默认用 `SEPARATE_LAYERS`。

### 4.5 Firefox/Wayland 侧怎么 import

链路：

```
ffmpeg hwcontext_vaapi.c: vaapi_map_to_drm_esh()
   └─ vaExportSurfaceHandle(VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2)
        └─ VADRMPRIMESurfaceDescriptor
             └─ AVDRMFrameDescriptor（ffmpeg 转换）
                  └─ Firefox DMABufSurface（CreateDMABufSurface / YUV 变体）
                       └─ EGL_EXT_image_dma_buf_import（每层一个 EGLImage）
                            └─ Mesa freedreno 采样 NV12 纹理
                                 └─ WebRender 合成到 wl_surface
```

要求：

1. **EGL 扩展**：`EGL_EXT_image_dma_buf_import`、`EGL_EXT_image_dma_buf_import_modifiers`、`EGL_KHR_image_base`。用 `eglinfo -B` 或 `eglQueryString(display, EGL_EXTENSIONS)` 确认（Mesa 26.0.3 的 freedreno 应该有）。
2. **格式**：`DRM_FORMAT_NV12`（Y=R8，UV=GR88）。如果 Mesa 对 `GR88` 的 NV12 导入有额外对齐要求，按 §4.2 的 256 字节 stride 对齐规避。
3. **modifier**：`DRM_FORMAT_MOD_LINEAR`。若用 Qualcomm tiled，需要 `EGL_EXT_image_dma_buf_import_modifiers` 且 freedreno 认识该 modifier；MVP 不做。
4. **同步**：V4L2 没有 dma-fence 导出，`DQBUF` 返回即表示硬件写完。Firefox 拿到的是"已经完成的帧"，不需要额外 fence；**不要在 DQBUF 之前导出**。
5. **Wayland**：解码帧是 WebRender 的纹理，不经过 `zwp_linux_dmabuf_v1` 送合成器；所以 `wl_drm`/`linux-dmabuf` 的兼容性问题不会卡住视频，只要 EGL 导入成功即可。

### 4.6 缓冲区生命周期规则（容易出 use-after-free 的地方）

| 事件 | 必须做什么 |
|---|---|
| `vaCreateSurfaces` | 从池取 buffer，标记 `IN_USE`；若走 EXPBUF 别名，**不能**把 V4L2 buffer 重新 `QBUF` 给下一次解码，直到 surface 被 `vaDestroySurfaces` |
| `vaEndPicture` | surface 绑定 `frame_seq`，buffer 仍属于 V4L2（已入队 CAPTURE） |
| `vaSyncSurface` | 出队后 surface 可导出；buffer 仍属于 surface |
| `vaDestroySurfaces` | 若 buffer 已出队且 surface 不再使用，重新 `QBUF` 回 CAPTURE 队列（别名模式）；若是自有 dma-buf，直接释放 |
| `vaDestroyContext` | 先 `STREAMOFF`（会丢弃在途 buffer），再 `REQBUFS(0)`，再 `munmap`/`close` |
| `vaTerminate` | 释放所有 config/context/surface/buffer，关闭 `/dev/video0` |

**别名模式的最大陷阱**：V4L2 CAPTURE 队列里的 buffer 数量 = surface 池大小。如果客户端创建的 surface 数 > 池大小，`vaCreateSurfaces` 会失败。缓解：

- 池大小 = `min(32, max(16, 客户端第一次 vaCreateSurfaces 的 num_surfaces + 4))`；
- 若后续 surface 数超出，做一次"停流 → `REQBUFS(0)` → 扩容 → 重建 → 开流"，并把已有 surface 重新绑定（复杂，L2 再做）；
- 或者退回拷贝模式（surface 自有 dma-buf，CAPTURE 只要 8 个 buffer），彻底解耦。

---

## 5. profile / entrypoint 矩阵

### 5.1 支持矩阵

| VA profile | 位深/色度 | V4L2 OUTPUT pixfmt | V4L2 CAPTURE pixfmt | 本项目 | 说明 |
|---|---|---|---|---|---|
| `VAProfileH264Baseline` | 8bit 4:2:0 | `V4L2_PIX_FMT_H264` | `V4L2_PIX_FMT_NV12` | ✅ L1 | MVP 首个目标（CAVLC、无 B 帧、无重排，最好调） |
| `VAProfileH264ConstrainedBaseline` | 8bit 4:2:0 | 同上 | 同上 | ✅ L1 | 很多 baseline 流实际标这个 |
| `VAProfileH264Main` | 8bit 4:2:0 | 同上 | 同上 | ✅ L2 | CABAC、B 帧 |
| `VAProfileH264High` | 8bit 4:2:0 | 同上 | 同上 | ✅ L2 | 8x8 transform、scaling list |
| `VAProfileH264High10` | 10bit 4:2:0 | （需 `V4L2_PIX_FMT_P010`） | 无 | ❌ | 返回 `VA_STATUS_ERROR_UNSUPPORTED_PROFILE` |
| `VAProfileH264MultiviewHigh` / `StereoHigh` | 8bit | 同上 | 同上 | ❌ | 返回不支持 |
| `VAProfileHEVCMain` | 8bit 4:2:0 | `V4L2_PIX_FMT_HEVC` | `NV12` | ⚠️ L4 | 依赖 VPS/SPS/RPS 重建，见 §3.6.5 |
| `VAProfileHEVCMain10` | 10bit 4:2:0 | （需 P010） | 无 | ❌ | capture 侧没有 10bit 格式 |
| `VAProfileVP9Profile0` | 8bit 4:2:0 | `V4L2_PIX_FMT_VP9` | `NV12` | ⚠️ L4 | 见 §3.6.5 |
| `VAProfileVP9Profile2` | 10bit 4:2:0 | （需 P010） | 无 | ❌ | |
| `VAProfileAV1Profile0/1` | — | — | — | ❌ | `/dev/video0` 未枚举 `AV1` |
| `VAProfileMPEG2*` / `VC1*` / `VP8` | — | — | — | ❌ | 未枚举 |

入口点只有 `VAEntrypointVLD`。`vaQueryConfigEntrypoints` 返回 1 项；对 `VAEntrypointEncSlice`/`VAEntrypointVideoProc` 返回 `VA_STATUS_ERROR_UNSUPPORTED_ENTRYPOINT`。

### 5.2 为什么 10bit 现在不能做

【实测】iris 的 CAPTURE 侧只枚举到 `NV12`。这意味：

- 内核/FW 没有把 10bit 输出格式暴露出来（可能是驱动未实现、也可能是该 SoC 的 HFI 固件能力未声明）；
- VA 侧若要支持 `VAProfileHEVCMain10`，必须能拿到 `P010`（或 `NV12_10`）的 dma-buf，并按 `DRM_FORMAT_P010` 导出；
- 在 CAPTURE 只有 NV12 的前提下，任何"10bit 软转 8bit"的做法都发生在内核/FW 之外，我们拿不到 10bit 数据，**无法支持**。

处理方式：

1. `vaQueryConfigProfiles` **不返回** 10bit profile；
2. `vaCreateConfig(VAProfileHEVCMain10)` 返回 `VA_STATUS_ERROR_UNSUPPORTED_PROFILE`；
3. 若客户端传 `rt_format = VA_RT_FORMAT_YUV420_10`，返回 `VA_STATUS_ERROR_UNSUPPORTED_RT_FORMAT`；
4. 在 README 与 `about:support` 排障说明里写清楚"10bit 走软解"，避免用户误判为 bug。

**后续验证点**：M1 时用 `v4l2-ctl -d /dev/video0 --list-formats` 再确认一次；若内核升级后出现 `P010`，按 §4.2 的 modifier/stride 规则扩展，工作量约 3~5 人天（VA 侧改动很小，主要是导出格式与 Firefox 的 10bit 纹理路径）。

### 5.3 `vaQueryConfigProfiles` 的实现

```c
static const struct {
    VAProfile va;
    uint32_t  pixfmt;
    const char *name;
} k_profiles[] = {
    { VAProfileH264Baseline,           V4L2_PIX_FMT_H264, "H.264 Baseline" },
    { VAProfileH264ConstrainedBaseline,V4L2_PIX_FMT_H264, "H.264 Constrained Baseline" },
    { VAProfileH264Main,               V4L2_PIX_FMT_H264, "H.264 Main" },
    { VAProfileH264High,               V4L2_PIX_FMT_H264, "H.264 High" },
    /* L4 打开：{ VAProfileHEVCMain, V4L2_PIX_FMT_HEVC, "HEVC Main" }, */
    /* L4 打开：{ VAProfileVP9Profile0, V4L2_PIX_FMT_VP9, "VP9 Profile0" }, */
};

VAStatus iris_QueryConfigProfiles(VADriverContextP ctx, VAProfile *profiles, int *num)
{
    struct iris_driver *drv = ctx->pDriverData;
    int n = 0;
    for (size_t i = 0; i < ARRAY_SIZE(k_profiles); i++) {
        if (!drv->has_pixfmt[k_profiles[i].pixfmt])   /* ENUM_FMT 缓存 */
            continue;
        if (!iris_try_fmt_ok(drv, k_profiles[i].pixfmt, 1920, 1088))
            continue;                                  /* TRY_FMT 复核 */
        profiles[n++] = k_profiles[i].va;
    }
    *num = n;
    return VA_STATUS_SUCCESS;
}
```

要点：

- 返回顺序不重要，但**不要返回重复项**（libva 会原样透传，客户端可能去重失败）。
- profile 列表要能被编译期开关裁剪（`-Dh264=disabled` 之类），方便定位"是驱动问题还是码流问题"。
- `iris_try_fmt_ok` 用 `VIDIOC_TRY_FMT`，不要用 `S_FMT`（会改设备状态）。

### 5.4 `vaCreateConfig` 的校验顺序

```c
VAStatus iris_CreateConfig(VADriverContextP ctx, VAProfile profile,
                           VAEntrypoint entrypoint, VAConfigAttrib *attribs, int n,
                           VAConfigID *config_id)
{
    if (entrypoint != VAEntrypointVLD) return VA_STATUS_ERROR_UNSUPPORTED_ENTRYPOINT;
    if (!profile_supported(profile))   return VA_STATUS_ERROR_UNSUPPORTED_PROFILE;

    uint32_t rt = VA_RT_FORMAT_YUV420;   /* 默认 */
    for (i = 0; i < n; i++) {
        if (attribs[i].type == VAConfigAttribRTFormat) rt = attribs[i].value;
    }
    if (rt != VA_RT_FORMAT_YUV420) return VA_STATUS_ERROR_UNSUPPORTED_RT_FORMAT;
    ...
}
```

### 5.5 V4L2 侧的 profile/level 信息（只读，不作为主依据）

stateful 解码器的 profile/level 由码流决定，内核通常暴露只读 control：

```sh
v4l2-ctl -d /dev/video0 -L | grep -A3 -i 'h264\|hevc\|vp9'
# 可能看到 V4L2_CID_MPEG_VIDEO_H264_PROFILE / _LEVEL，属性为 inactive/read-only
```

**不要依赖这些 control 做能力判断**：它们反映的是"当前码流"，不是"设备支持"。能力判断以 `ENUM_FMT` + `TRY_FMT` 为准。

---

## 6. 状态机与队列管理

### 6.1 对象与所有权（先定这个，后面才不返工）

```
iris_display（每个 VADisplay 一个）
 ├── drm_fd                （libva 给的，不关闭）
 ├── caps                  （QUERYCAP/ENUM_FMT/TRY_FMT 缓存）
 ├── v4l2 设备与队列        （/dev/video0 fd、OUTPUT/CAPTURE buffer、当前 S_FMT）
 ├── surface 池            （VASurfaceID → iris_surface，含 dma-buf）
 └── 全局锁

iris_config（每个 VAConfigID）
 └── profile / entrypoint / rt_format

iris_context（每个 VAContextID）
 ├── config 指针
 ├── picture_width/height  （coded size）
 ├── AU 累积缓冲、当前 surface
 ├── 已提交帧表            （frame_seq → surface，FIFO + 哈希）
 └── 解码子状态            （IDLE / DECODING / ERROR / DRAINING）
```

**为什么 surface 挂在 display 而不是 context**：VA-API 的 `vaCreateSurfaces` 只接受 `VADisplay`，不接受 `VAContextID`；而且 ffmpeg 在 `avcodec_flush_buffers()` 时很可能销毁并重建 VA context（需在 ffmpeg 8.0.1 的 `libavcodec/vaapi_decode.c` 里确认 `vaapi_decode_flush()`），但 hwframes/surfaces 会保留。如果 surface 绑定在 context 上，seek 一次就全废。

**为什么 V4L2 队列也挂在 display**：surface 是 display 级的，如果 CAPTURE buffer 由 context 持有，context 销毁时 buffer 被释放，surface 就悬空了。把队列放 display 级，context 只是"解码会话"，销毁时只做 `STREAMOFF` + `REQBUFS(0)`，surface 的内存仍在（拷贝模式下 surface 自有 dma-buf，天然安全）。

> **ADR-1（架构决策）：MVP 默认走"拷贝模式"。**
> 理由：拷贝模式下 surface 的内存与 V4L2 buffer 完全解耦，context 生命周期、surface 数量、seek/flush 都不会互相牵连；代价是每帧一次 NV12 memcpy（1080p30 ≈ 93 MB/s，实测内存带宽 > 20 GB/s，CPU 开销约 2%~5%）。
> 零拷贝（`EXPBUF` 别名）作为 L2 优化，前提是 M1 验证 `EXPBUF` 可用，且接受"surface 数 = CAPTURE buffer 数"的约束。

### 6.2 状态机

```
                    vaCreateContext
                          │
                          ▼
                    ┌───────────┐   S_FMT/REQBUFS/STREAMON 失败
                    │   INIT    │──────────────────────────► ERROR
                    └─────┬─────┘
                          │ 队列就绪
                          ▼
                    ┌───────────┐
        ┌──────────►│ STREAMING │◄──────────┐
        │           └─────┬─────┘           │ 重新 S_FMT/REQBUFS
        │                 │                 │
        │   SOURCE_CHANGE │                 │
        │                 ▼                 │
        │           ┌───────────┐           │
        └───────────│ RESOLUTION│───────────┘
                    │  _CHANGE  │  新尺寸 ≤ surface 尺寸
                    └─────┬─────┘
                          │ 新尺寸 > surface 尺寸
                          ▼
                    ┌───────────┐
                    │  ERROR    │（对客户端返回 RESOLUTION_NOT_SUPPORTED）
                    └───────────┘

   任意状态 ── vaDestroyContext ──► TERMINATED（STREAMOFF + REQBUFS(0)）
   STREAMING ── EOS/DEC_CMD_STOP ──► DRAINING（等 LAST）──► STREAMING/TERMINATED
```

### 6.3 提交流程与 surface 关联（本项目最容易错的地方）

VA-API 的模型是"surface 是解码目标"，客户端之后会对**同一个 surface** 调 `vaSyncSurface` / `vaExportSurfaceHandle`。而 V4L2 的 CAPTURE 队列是"按显示序出帧"的公共队列。两者之间必须建立映射。

**唯一可靠的映射键是 `v4l2_buffer.timestamp`。** 做法：

1. 每个 `vaEndPicture` 分配一个单调递增的 `frame_seq`（从 0 开始）；
2. `QBUF` 时把 `frame_seq` 写进 `b.timestamp`（`tv_sec = seq/1e6, tv_usec = seq%1e6`）；
3. 驱动若调用 `v4l2_m2m_buf_copy_metadata()`，CAPTURE buffer 出队时 `timestamp` 就是对应 OUTPUT buffer 的值；
4. `DQBUF` 后按 timestamp 反查 `frame_seq`，再查 surface，标记 ready。

**验证方法（M1 必做）**：

```sh
grep -rn "v4l2_m2m_buf_copy_metadata" /path/to/linux-sm8450-liuqin/drivers/media/platform/qcom/iris/
# 有调用 → 方案成立
# 没有   → 需要给内核打一行补丁（见下），或退回"串行化"模式（性能差，见本节后文）
```

**如果内核不复制 timestamp**，三个选择：

- **（推荐）给内核 iris 打一行补丁**：在 capture buffer 完成处调用 `v4l2_m2m_buf_copy_metadata(&dst->vb, &src->vb)`。这是 v4l2-mem2mem 的标准用法，其他 stateful 驱动（venus）都这么做，向上游提交也合理。
- **串行化模式**：一次只允许一个 AU 在途，`QBUF` 后立刻 `DRAIN`（`V4L2_DEC_CMD_STOP`）并等所有 CAPTURE 出队，再提交下一帧。这样输出顺序完全确定（第 N 次提交对应第 N 个出队帧），但**每帧一次 drain，性能会掉到个位数 fps**，只能用于功能验证。
- **放弃 B 帧**：只支持 baseline（无重排），此时提交序 = 显示序，FIFO 映射成立。MVP 的 baseline 测试可以这么干，但不能作为交付方案。

**为什么不能用 FIFO 直接映射**：H.264 High 普遍有 B 帧，V4L2 按显示序出帧，提交序是解码序，两者不同。第 1 个出队的帧可能是第 3 次提交的 surface。FIFO 映射会把画面张冠李戴（表现为"画面顺序错乱/闪回"），且难以定位。

**如果 timestamp 被内核覆盖**（打印出来全是单调时钟值，而不是 0,1,2…），还有两个备选键：

- `timecode`：`v4l2_m2m_buf_copy_metadata()` 也会复制 `timecode`，前提是 QBUF 时置 `V4L2_BUF_FLAG_TIMECODE` 并填 `struct v4l2_timecode`（帧号/秒号是 8bit 字段，容量有限，适合短测试不适合长片）；
- 内核补丁：`vb2_buffer` 没有通用的 64 位用户字段可借用，所以更现实的做法是确保 iris 调用 `v4l2_m2m_buf_copy_metadata()`，而不是另找关联键。

### 6.4 EOS / 排空（`V4L2_DEC_CMD_STOP`、`V4L2_BUF_FLAG_LAST`）

VA-API 没有 EOS 调用。客户端的"结束"表现为：

- 不再调用 `vaEndPicture`；
- seek 时调用 `avcodec_flush_buffers()` → 可能触发 `vaDestroyContext`；
- 关闭时 `vaDestroyContext`/`vaTerminate`。

因此驱动需要排空的场景主要是**内部测试**和**关闭时的干净收尾**：

```c
/* 排空：告诉解码器"输入到此为止" */
struct v4l2_decoder_cmd cmd = { .cmd = V4L2_DEC_CMD_STOP };
ioctl(fd, VIDIOC_DECODER_CMD, &cmd);

/* 然后一直 DQBUF CAPTURE，直到最后一个 buffer 带 V4L2_BUF_FLAG_LAST */
while (!last_seen) {
    poll(fd, POLLIN, 1000);
    dqbuf(...);
    if (b.flags & V4L2_BUF_FLAG_LAST) last_seen = true;
}
```

要点：

- `V4L2_DEC_CMD_STOP` 只对 stateful decoder 有效；如果返回 `EINVAL`，说明驱动不支持，退化为直接 `STREAMOFF`（丢弃剩余帧）。
- `V4L2_BUF_FLAG_LAST` 只会出现在 CAPTURE 队列的**最后一个** buffer 上；收到后不要继续 `DQBUF`。
- 排空后若要继续解码，用 `V4L2_DEC_CMD_START` 恢复（有些驱动要求重新 `STREAMON`，以实测为准）。
- **`STREAMOFF` 会丢弃所有在途帧**，所以"正常关闭"和"seek"都用它，但"想要最后几帧"必须先 STOP 排空。

### 6.5 `SOURCE_CHANGE`（分辨率动态变化）的完整处理序列

```c
void handle_source_change(struct iris_display *d)
{
    struct v4l2_event ev;
    ioctl(d->fd, VIDIOC_DQEVENT, &ev);
    if (ev.type != V4L2_EVENT_SOURCE_CHANGE) return;
    if (!(ev.u.src_change.changes & V4L2_EVENT_SRC_CH_RESOLUTION)) return;

    struct v4l2_format f = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE };
    ioctl(d->fd, VIDIOC_G_FMT, &f);
    uint32_t nw = f.fmt.pix_mp.width, nh = f.fmt.pix_mp.height;

    if (nw > d->max_surface_w || nh > d->max_surface_h) {
        fail_all_inflight(d, VA_STATUS_ERROR_RESOLUTION_NOT_SUPPORTED);
        d->state = IRIS_ERROR;
        return;
    }

    /* 1. 把 CAPTURE 队列里所有 buffer 收回（标记为跳过/错误） */
    drain_capture_all(d);

    /* 2. 停 CAPTURE 流 */
    enum v4l2_buf_type t = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    ioctl(d->fd, VIDIOC_STREAMOFF, &t);

    /* 3. 释放旧 buffer */
    struct v4l2_requestbuffers rb = { .count = 0, .type = t, .memory = V4L2_MEMORY_MMAP };
    ioctl(d->fd, VIDIOC_REQBUFS, &rb);

    /* 4. 设新格式 */
    f.fmt.pix_mp.width = nw; f.fmt.pix_mp.height = nh;
    f.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
    f.fmt.pix_mp.num_planes = 1;
    ioctl(d->fd, VIDIOC_S_FMT, &f);

    /* 5. 重新申请 + 映射/导出 + 开流 */
    rb.count = d->cap_count;
    ioctl(d->fd, VIDIOC_REQBUFS, &rb);
    setup_capture_buffers(d);
    ioctl(d->fd, VIDIOC_STREAMON, &t);

    d->cur_w = nw; d->cur_h = nh;
    d->state = IRIS_STREAMING;
}
```

细节与坑：

- **事件可能在第一帧之前就来**（用于告知初始分辨率）。此时 CAPTURE 队列可能还没 buffer，处理要幂等。
- **必须先 `G_FMT` 再 `S_FMT`**：`S_FMT` 会把请求值"就近取整"，直接 `S_FMT(1920x1080)` 可能被改成 1920×1088；`G_FMT` 拿到的才是驱动真正要输出的尺寸。
- **`STREAMOFF(CAPTURE)` 之后必须 `REQBUFS(0)`**：vb2 不允许在 buffer 未释放时改格式（会 `EBUSY`）。
- **OUTPUT 队列不要停**：已入队的码流还在解码，停了要重喂。若驱动要求两个队列都停（实测 `EBUSY`），只能整条管道 reset，并从下一个 IDR 重新开始（§6.6）。
- **VA surface 是固定尺寸的**：新分辨率 ≤ 旧 surface 时，后续帧仍解到旧尺寸 surface 里（多余区域不显示）；新分辨率更大时只能报错，让客户端重建解码器。Firefox 在 `RESOLUTION_NOT_SUPPORTED` 时会重建 decoder（需实测确认，见 §7.6）。
- **`V4L2_FMT_FLAG_DYN_RESOLUTION`** 是"这个格式可能动态变分辨率"的声明，不代表每帧都变。订阅事件即可，不要轮询。

### 6.6 flush / seek

VA-API 没有 flush 入口。实际触发路径：

| 客户端行为 | 驱动看到的 | 驱动要做 |
|---|---|---|
| `avcodec_flush_buffers()`（ffmpeg seek） | 通常是 `vaDestroyContext`（+ 之后重建） | `STREAMOFF` 两个队列 → `REQBUFS(0)` → 清空在途表 → 下次 `vaCreateContext` 重新 `S_FMT`/`REQBUFS`/`STREAMON` |
| `mpv` 拖动进度条 | 同上（mpv 内部 `avcodec_flush_buffers`） | 同上 |
| Firefox 切换 video src | 销毁 decoder → `vaDestroyContext` | 同上 |
| 解码出错后继续喂 | `vaBeginPicture` 时 `state == ERROR` | 若当前 AU 是 IDR，尝试 reset 后恢复；否则返回 `VA_STATUS_ERROR_DECODING_ERROR` |

实现建议：

- **不要关闭 `/dev/video0` fd**，reset 时复用；`open/close` 一次约几毫秒，seek 频繁时体感明显。
- reset 时把 `frame_seq` 继续递增（不要归零），避免旧 timestamp 与新帧碰撞。
- reset 后**强制要求下一个 AU 以 IDR 开始**：从 slice NAL 的 `nal_unit_type == 5` 判断（H.264）。若客户端给的是非 IDR（例如 seek 到了非关键帧），返回 `VA_STATUS_ERROR_DECODING_ERROR`，让客户端重新 seek——ffmpeg 默认 seek 到关键帧，正常不会发生。

### 6.7 多 slice、多帧与边界判定

| 情况 | 处理 |
|---|---|
| 一个 AU 多个 slice（常见） | 依次追加到同一个 AU buffer，`vaEndPicture` 时一次 `QBUF` |
| 一个 slice 被拆成多个 `VASliceDataBufferType`（`slice_data_flag` = BEGIN/MIDDLE/END） | 按 flag 拼接；MVP 只支持 `ALL`，其它返回 `VA_STATUS_ERROR_FLAG_NOT_SUPPORTED` |
| 同一个 `vaBeginPicture`/`vaEndPicture` 里出现两个 `first_mb_in_slice == 0` 的 slice | 视为两个 AU：遇到第二个时先把前一个 AU `QBUF` 出去（防御性处理，正常客户端不会这样） |
| `vaEndPicture` 时 AU 为空（EOS/flush 的空调用） | 直接返回 `VA_STATUS_SUCCESS`，不 `QBUF` |
| 一帧被拆成多次 `vaRenderPicture` | 正常，累积即可；不要假设 `vaRenderPicture` 调用次数 |

**如何判断 `first_mb_in_slice == 0`**：slice NAL 的第 1 字节是 NAL header，第 2 字节开始是 RBSP；`first_mb_in_slice` 是第一个 ue(v)。写一个 10 行的 `ue_read()` 就够。这个解析同时用于拿 `pic_parameter_set_id`（§3.6.3）。

### 6.8 错误恢复

```c
void iris_fail_pipeline(struct iris_display *d, VAStatus why)
{
    fail_all_inflight(d, why);       /* 所有未出队 surface 标记 error，唤醒等待者 */
    enum v4l2_buf_type t;
    t = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE;  ioctl(d->fd, VIDIOC_STREAMOFF, &t);
    t = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE; ioctl(d->fd, VIDIOC_STREAMOFF, &t);
    d->need_reset = true;            /* 下次 vaCreateContext/vaBeginPicture 时重建 */
    d->state = IRIS_ERROR;
}
```

触发条件：

- `DQBUF` 返回 `EIO`；
- CAPTURE buffer 带 `V4L2_BUF_FLAG_ERROR`；
- `poll` 超时（默认 2 s，可用 `LIBVA_IRIS_TIMEOUT_MS` 覆盖）；
- `QBUF` 连续返回 `EPIPE`/`ENOSPC`。

恢复策略：**重置整条管道**，并要求下一个 AU 是 IDR。不要试图"跳过坏帧继续"——stateful 解码器的参考帧状态已经脏了，继续解只会连续花屏。

### 6.9 线程与锁

- libva 客户端可能从多个线程调用（GStreamer 的 `vah264dec` 有独立解码线程；Firefox 每个 decoder 一个线程）。
- 用**一把 display 级互斥锁**保护 V4L2 fd 与队列状态，简单可靠；`vaRenderPicture` 只做 memcpy，持锁时间短；`vaEndPicture`/`vaSyncSurface` 可能阻塞，持锁等待会导致其它线程卡住——**阻塞等待要用条件变量释放锁**，或用"仅一个线程进解码路径"的约定（ffmpeg 天然单线程解码）。
- 不要在持锁时调用 `poll` 无限等待；用 100 ms 切片轮询，顺便检查 `destroying` 标志。

### 6.10 超时与看门狗

| 位置 | 默认超时 | 行为 |
|---|---|---|
| `vaEndPicture` 等空闲 OUTPUT buffer | 500 ms | 超时返回 `VA_STATUS_ERROR_OPERATION_FAILED` 并标记 pipeline error |
| `vaSyncSurface` 等 CAPTURE | 2000 ms | 同上 |
| `vaDestroyContext` 的 `STREAMOFF` | 不等待 | 直接停 |

超时时间用环境变量可调，方便在慢设备/调试时放宽：

```sh
LIBVA_IRIS_TIMEOUT_MS=5000 vainfo ...
```

---

## 7. Firefox 侧的具体要求

### 7.1 Firefox 的 VA-API 调用链

```
Firefox 内容进程 / RDD 进程
 └── FFmpegVideoDecoder（dom/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp）
      ├── dlopen("libmozavcodec.so")（捆绑的 ffmpeg，非系统 ffmpeg）
      ├── av_hwdevice_ctx_create(AV_HWDEVICE_TYPE_VAAPI)
      │     └── ffmpeg: libavutil/hwcontext_vaapi.c
      │           ├── open("/dev/dri/renderD128") → vaGetDisplayDRM()
      │           └── vaInitialize() → libva 加载 iris_drv_video.so
      ├── avcodec_send_packet / avcodec_receive_frame（AV_PIX_FMT_VAAPI）
      └── av_hwframe_map(frame, AV_HWFRAME_MAP_DIRECT)
            └── ffmpeg: vaapi_map_to_drm_esh()
                  └── vaExportSurfaceHandle(..., VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2, ...)
                        └── VADRMPRIMESurfaceDescriptor → AVDRMFrameDescriptor
                              └── Firefox DMABufSurface（gfx/2d/DMABufSurface.cpp）
                                    └── EGL_EXT_image_dma_buf_import → GL 纹理
                                          └── WebRender 合成
```

**关键结论：Firefox 用的是自己捆绑的 ffmpeg（`libmozavcodec.so`），不是系统 ffmpeg。** 这有两个后果：

1. 给系统 ffmpeg 打补丁（R2 直通）**不会影响 Firefox**；要让 Firefox 走直通，必须重编 Firefox 或替换其 `libmozavcodec.so`（后者 ABI 必须完全匹配，风险高）。
2. Firefox 的 ffmpeg 版本/编译选项决定它是否支持 VA-API。官方 Linux 构建是启用的（否则 VA-API 从来不可能工作），但**是否启用 `vaExportSurfaceHandle` 路径**要在 `about:support` 或 `MOZ_LOG` 里确认。

### 7.2 驱动必须满足的接口清单（Firefox 视角）

| 要求 | 不满足的后果 | 本项目对应 |
|---|---|---|
| `vaInitialize` 在 `/dev/dri/renderD128` 上成功 | Firefox 直接判定"无 VA-API"，全程软解 | §3.2 |
| `vaQueryConfigProfiles` 返回 `VAProfileH264High`（及 Baseline/Main） | ffmpeg 找不到可用 config，回退软解 | §5.3 |
| `vaCreateConfig`/`vaCreateContext` 成功 | 同上 | §3.2/§3.3 |
| `vaQuerySurfaceAttributes` 返回合理尺寸范围 | hwframe 池初始化失败 | §4.3 |
| `vaExportSurfaceHandle` + `DRM_PRIME_2` + NV12 两层 | `av_hwframe_map` 失败 → 回退软解或黑屏 | §4.4 |
| 导出的 fd 能被 `EGL_EXT_image_dma_buf_import` 导入 | 解码成功但无法上屏（黑屏/绿屏） | §4.5 |
| `vaSyncSurface` 不无限阻塞、错误能返回 | 视频卡死、标签页无响应 | §6.8/§6.10 |
| 线程安全（同一 display 多线程调用） | 随机崩溃 | §6.9 |

**不需要**的：`vaPutSurface`（Firefox 不用）、`vaCreateSubpicture`、`vaAcquireBufferHandle`（ffmpeg 的 `hwmap` 走 export 而非 acquire）。

**关于 `VADisplayAttribCopy`**：它属于 `vaPutSurface` 的显示属性（告诉驱动能否把 surface 拷贝到 drawable），Firefox 与 ffmpeg 的 VA-API 解码路径都不使用。我们的 `vaQueryDisplayAttributes` 返回 0 个属性、`vaSetDisplayAttributes` 返回 `VA_STATUS_SUCCESS`（或 `VA_STATUS_ERROR_UNIMPLEMENTED`）即可，**不需要实现 `VADisplayAttribCopy`**。真正必须的是 `vaExportSurfaceHandle`（§4.4）。

### 7.3 about:config 与 about:support

**about:config（按顺序试）**

| 首选项 | 值 | 作用 |
|---|---|---|
| `media.ffmpeg.vaapi.enabled` | `true` | VA-API 总开关（不同版本默认值不同，先确认） |
| `media.hardware-video-decoding.enabled` | `true` | 硬件解码总开关（默认通常为 true） |
| `media.hardware-video-decoding.force-enabled` | `true` | 绕过 blocklist（新驱动/未知 GPU 常被拦） |
| `gfx.webrender.all` | `true` | 确保 WebRender 开启（Wayland 默认开） |
| `gfx.webrender.software` | `false` | 软件 WebRender 下 DMABuf 路径不可用 |
| `widget.dmabuf.force-enabled` | `true` | 强制启用 DMABuf surface（部分环境默认关） |
| `security.sandbox.content.level` | `0` | **仅调试**：绕开内容进程 seccomp（见 §7.4），验证完必须改回 |
| `media.rdd-ffmpeg.enabled` | `false` | **仅调试**：让 ffmpeg 在内容进程跑，方便 strace；生产保持默认 |

> 上表里 `media.ffmpeg.vaapi.enabled` 的确切默认值随版本变化，用 about:config 搜索 "vaapi" 看当前值，不要假设。

**about:support 要看的三处**

1. **Graphics** → `Compositing` 必须是 `WebRender`（不是 `WebRender (Software)`）；
2. **Media** → `Hardware Video Decoding` 一栏应显示 VA-API 可用/已启用（不同版本文案不同）；
3. **Graphics** → `Failure Log` / `Decision Log`，若有 `VA-API` 相关失败会在这里。

**日志**

```sh
MOZ_LOG="PlatformDecoderModule:5,FFmpeg:5,DMABuf:5" \
MOZ_LOG_FILE=/tmp/firefox-media.log \
LIBVA_DRIVER_NAME=iris LIBVA_MESSAGING_LEVEL=3 \
firefox
```

期望在日志里看到：`VA-API`、`vaExportSurfaceHandle`、`DMABufSurface` 创建成功；若看到 `VA-API is not supported` 或 `Failed to create VA display`，回到 §7.2 逐项排查。

### 7.4 内容进程 sandbox：最容易被忽略的拦路虎

Firefox 的内容进程用 seccomp-bpf + broker 白名单限制文件访问。Linux 的策略在 `security/sandbox/linux/broker/SandboxBrokerPolicyFactory.cpp`，其中与图形相关的通常只有：

```cpp
AddPath("/dev/dri", ...);          /* 允许 VA-API / EGL */
AddPath("/dev/nvidia*", ...);
/* 没有 /dev/video* */
```

也就是说：**即使 VA 驱动加载成功，驱动里的 `open("/dev/video0")` 也会被拒绝（EACCES/EPERM）**。表现是 `vaCreateContext` 失败 → Firefox 回退软解，日志里只有一句含糊的初始化失败。

验证方法：

```sh
# 方法 1：直接看 strace
strace -f -e trace=openat -o /tmp/ff.strace firefox 2>/dev/null &
grep -n 'video0' /tmp/ff.strace        # 若有 EACCES，就是它

# 方法 2：临时关沙箱对比
MOZ_DISABLE_CONTENT_SANDBOX=1 firefox   # 或 security.sandbox.content.level=0
# 若关沙箱后 VA-API 生效 → 确诊
```

四条应对路线：

| 路线 | 做法 | 代价 | 适用 |
|---|---|---|---|
| **S1 关沙箱（仅测试）** | `security.sandbox.content.level=0` 或 `MOZ_DISABLE_CONTENT_SANDBOX=1` | 安全性下降，不能作为交付 | L3 验证阶段 |
| **S2 给 Firefox 打补丁** | 在 `SandboxBrokerPolicyFactory` 里给内容进程/RDD 进程加 `AddPath("/dev/video0", ...)` | 要重编 Firefox（或维护发行版补丁）；上游不一定接受 | 生产可用路线之一 |
| **S3 特权 helper** | 驱动不直接开 `/dev/video0`，而是连一个 systemd 服务（unix socket + `SCM_RIGHTS` 传 dma-buf fd），由服务做 V4L2 队列操作 | 架构复杂、多一次 IPC；但完全绕开沙箱，且可给 snap/Flatpak 用 | 长期/多浏览器方案 |
| **S4 换解码进程** | 让 ffmpeg 跑在 RDD 进程（`media.rdd-ffmpeg.enabled=true`），检查 RDD 的 sandbox 策略是否放行 `/dev/video*` | 取决于 RDD 策略；不一定可行 | 先验证，成本低 |

**建议顺序：S1 验证 → S4 试探 → S2 落地（deb 版 Firefox 自带补丁）→ S3 作为多浏览器/打包方案。**

### 7.5 snap 沙箱：为什么 snap 版基本走不通

【实测】Firefox 149 是 snap 包（`firefox 149.0.2-1`）。snap 版有三层额外障碍：

1. **挂载命名空间**：snap 的 rootfs 是 `core24` base + 内容 snap（`mesa-2404` 提供 `/usr/lib/aarch64-linux-gnu/dri`）。**宿主机 `/usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so` 在 snap 内不可见**，除非重建一个 content snap 提供它——而 Firefox 的 snap 没有声明对应的 plug，无法连接。
2. **AppArmor**：`snap.firefox.firefox` 的 profile 由接口生成。`opengl` 接口只放行 `/dev/dri/**`；`/dev/video*` 需要 `camera` 接口（且 snap 里 Firefox 是否 plug 了 camera 要 `snap connections firefox` 确认）。
3. **环境变量**：`snap run` 会清理部分环境变量，`LIBVA_DRIVER_NAME`/`LIBVA_DRIVERS_PATH` 能否传进去要实测；即使能传，路径也必须在 snap 命名空间内可见（`$SNAP_USER_COMMON` 可见，可以放驱动）。

可行的"折腾路线"（不推荐作为交付）：

```sh
# 1. 把驱动放进 snap 可见的用户目录
mkdir -p ~/snap/firefox/common/vaapi
cp /usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so ~/snap/firefox/common/vaapi/

# 2. 从终端带环境变量启动
LIBVA_DRIVER_NAME=iris \
LIBVA_DRIVERS_PATH=$HOME/snap/firefox/common/vaapi \
snap run firefox

# 3. 检查 /dev/video0 是否可见
snap run --shell firefox -c 'ls -l /dev/video0 /dev/dri/renderD128'
```

**推荐路线（与项目已有经验一致）**：安装 Mozilla 官方 APT 源的 **deb 版 Firefox**，移除 snap 版。理由：

- 宿主机文件系统直接可见，驱动 so 放标准路径即可；
- 没有 snap 的 AppArmor 层，只剩 Firefox 自己的 seccomp（§7.4，可用 S2 解决）；
- 项目 `gh/android-mainline-port/experience/ubuntu-userspace.md` 已经记录过"snap 应用 GPU 用户态来自 `mesa-2404`，宿主机自编 Mesa 进不了沙箱"的同类问题，以及"用 Mozilla deb 替换 snap"的做法。

**Flatpak 作为备选**：

```sh
flatpak install flathub org.mozilla.firefox
flatpak override --user --device=all \
  --filesystem=/usr/lib/aarch64-linux-gnu/dri:ro \
  --env=LIBVA_DRIVER_NAME=iris \
  --env=LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri \
  org.mozilla.firefox
```

`--device=all` 是为了放行 `/dev/video0`（只给 `--device=dri` 不够）；代价是沙箱更宽。

### 7.6 分辨率变化与 seek 在 Firefox 里的表现

- **seek**：Firefox 调 `avcodec_flush_buffers()` → ffmpeg 可能 `vaDestroyContext` → 我们的重置路径（§6.6）必须干净。若 seek 后花屏/黑屏，优先查"是否从 IDR 重新开始"。
- **分辨率变化**（B 站切清晰度、广告切换）：多数情况是**新建 decoder**（新 video element/新 hwframes），不是同一码流内变分辨率；但同一 `MediaSource` 内切换分辨率会触发 `SOURCE_CHANGE`。
  - 若新分辨率 ≤ 旧 surface：我们继续解，Firefox 按新尺寸显示，应正常；
  - 若新分辨率 > 旧 surface：`vaSyncSurface` 返回 `VA_STATUS_ERROR_RESOLUTION_NOT_SUPPORTED`，Firefox 应该重建 decoder（需实测；若不重建，表现为黑屏，需要在 Firefox 侧确认 fallback 行为）。
- **帧率变化**：V4L2 不关心帧率，VA-API 也不传递帧率；Firefox 用容器时间戳做 A/V 同步，不受影响。

### 7.7 最小化验证路径（不装 Firefox 也能验证浏览器侧）

在动 Firefox 之前，先用 `mpv` 验证"VA-API + EGL 导入"这条链路，因为它比 Firefox 好调试：

```sh
mpv --hwdec=vaapi --vo=gpu-next --msg-level=all=v video.mp4
# 关注：Using hardware decoding (vaapi)、VO: [gpu-next]、有无 vaExportSurfaceHandle 报错
```

`mpv` 通了，说明 §4.4/§4.5 没问题，剩下的是 Firefox 的 sandbox/config 问题。

---

## 8. 构建与部署

### 8.1 设备上的构建依赖（先解决"没有编译器和 vainfo"）

【实测】设备当前**没有** `gcc`/`meson`/`ninja`/`make`，`vainfo` 也没装。网络约 283 kB/s，装依赖要算时间。

```sh
sudo apt update
sudo apt install -y \
  build-essential meson ninja-build pkg-config git \
  libva-dev libdrm-dev libgbm-dev \
  vainfo v4l-utils \
  gstreamer1.0-tools gstreamer1.0-plugins-bad gstreamer1.0-libav \
  python3-pip
```

| 项 | 估算 | 说明 |
|---|---|---|
| 下载量 | 约 120–180 MB | `build-essential` + `libva-dev` + `libdrm-dev` + `vainfo` + `v4l-utils` 及其依赖 |
| 时间（283 kB/s） | 约 8–12 分钟 | 若走国内镜像可显著缩短；先 `apt update` 确认源速度 |
| 磁盘 | < 1 GB | 设备根分区余量 52 GB，充足 |

**镜像建议**：`/etc/apt/sources.list.d/` 里换成 `mirrors.ustc.edu.cn`/`mirrors.tuna.tsinghua.edu.cn` 的 Ubuntu 26.04 arm64 源（若当前源慢）。这是"一次投入、长期受益"的 5 分钟。

**验证工具链**：

```sh
gcc --version          # 期望 15/16.x
meson --version        # 期望 >= 1.4
ninja --version
pkg-config --modversion libva     # 期望 1.23.x（VA-API 版本，不是包版本 2.23）
vainfo --version
```

### 8.2 源码与 meson 构建

源码建议放 `~/src/libva-iris`（设备上），同时托管到项目 Git（`github.com/yzddmr6/...` 或内部仓库），**不要只放在设备上**。

`meson.build`（最小可用版本）：

```meson
project('libva-iris', 'c',
  version: '0.1.0',
  default_options: ['c_std=gnu11', 'warning_level=2', 'buildtype=debugoptimized'])

va_dep  = dependency('libva',  version: '>= 1.21')
drm_dep = dependency('libdrm')
gbm_dep = dependency('gbm', required: false)
thread_dep = dependency('threads')

src = files(
  'src/driver.c', 'src/display.c', 'src/config.c', 'src/context.c',
  'src/surface.c', 'src/buffer.c', 'src/decode.c', 'src/export.c',
  'src/image.c', 'src/v4l2.c', 'src/queue.c', 'src/profile.c',
  'src/bitstream/bs_writer.c', 'src/bitstream/h264_sps.c',
  'src/bitstream/h264_pps.c', 'src/bitstream/h264_au.c',
)

shared_library('iris_drv_video', src,
  name_prefix: '',
  dependencies: [va_dep, drm_dep, gbm_dep, thread_dep],
  include_directories: include_directories('src'),
  gnu_symbol_visibility: 'hidden',        # 只导出 VA_EXPORT 的符号
  install: true,
  install_dir: get_option('libdir') / 'dri',
)

# 可选：码流重建器单测（不依赖硬件）
if get_option('tests')
  test_exe = executable('test_bs', 'test/test_bs.c',
    files('src/bitstream/bs_writer.c', 'src/bitstream/h264_sps.c',
          'src/bitstream/h264_pps.c', 'src/bitstream/h264_au.c'),
    include_directories: include_directories('src'))
  test('bitstream', test_exe)
endif
```

构建与安装：

```sh
meson setup build --prefix=/usr --libdir=lib/aarch64-linux-gnu -Dtests=true
ninja -C build
sudo meson install -C build
# 安装结果：/usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so
```

**为什么 `--libdir=lib/aarch64-linux-gnu`**：Debian/Ubuntu 的 multiarch 路径；libva 编译时写死的驱动搜索路径就是 `/usr/lib/aarch64-linux-gnu/dri`。用默认 `lib` 会装到 `/usr/lib/dri`，libva 找不到。

**符号检查**：

```sh
nm -D /usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so | grep -i vadriverinit
# 期望：0000000000001234 T __vaDriverInit_1_0
```

### 8.3 安装后的加载配置

libva 查找驱动的顺序（`va/va.c` 的 `va_open_driver_*`）：

1. 环境变量 `LIBVA_DRIVER_NAME` 指定的名字 → `${LIBVA_DRIVERS_PATH:-/usr/lib/aarch64-linux-gnu/dri}/${name}_drv_video.so`；
2. 否则用 DRM 驱动名（`drmGetVersion(fd)->name`）——本机是 `msm` → 找 `msm_drv_video.so`；
3. 都没有 → `vaInitialize` 失败。

> 这是 libva 2.x 的自动探测逻辑。若你的 libva 版本行为不同，用 `strace -f -e trace=openat vainfo ... 2>&1 | grep dri` 看它实际找的是哪个文件名，再决定要不要建符号链接。

因此有**两种**部署方式：

**方式 A（显式，推荐开发期）**

```sh
export LIBVA_DRIVER_NAME=iris
export LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri
vainfo --display drm --device /dev/dri/renderD128
```

**方式 B（隐式，推荐交付期）**

```sh
# 让"不设环境变量"的应用也能用：利用驱动名回退
sudo ln -sf iris_drv_video.so /usr/lib/aarch64-linux-gnu/dri/msm_drv_video.so
```

这样 Firefox（从桌面图标启动、拿不到自定义环境变量时）也能通过 `msm` 回退加载到我们的驱动。**这是 snap/Flatpak 之外，让"零配置"生效的关键技巧。**

> 注意：`msm_drv_video.so` 这个名字不要和 Mesa 的 `msm_dri.so` 混淆，后者是 DRI/GL 驱动，`libva` 不会加载它。

**用户权限**：当前用户 `yumo` 已在 `render` 组（【实测】），能打开 `/dev/dri/renderD128`。`/dev/video0` 的权限如果是 `root:video 0660`，还需要 `video` 组（已在）。

### 8.4 验证命令（从底层到上层）

```sh
# 0. 设备与格式
v4l2-ctl -d /dev/video0 --all
v4l2-ctl -d /dev/video0 --list-formats-out
v4l2-ctl -d /dev/video0 --list-formats

# 1. VA 驱动加载
LIBVA_DRIVER_NAME=iris LIBVA_MESSAGING_LEVEL=3 \
  vainfo --display drm --device /dev/dri/renderD128

# 2. ffmpeg 硬解（不出图，只验证解码）
ffmpeg -v verbose -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i test_1080p_h264_high.mp4 -f null -

# 3. ffmpeg 导出 NV12 比对
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i test.mp4 -f rawvideo -pix_fmt nv12 /tmp/va.yuv
ffmpeg -v error -i test.mp4 -f rawvideo -pix_fmt nv12 /tmp/sw.yuv
cmp /tmp/va.yuv /tmp/sw.yuv && echo "BIT-EXACT"

# 4. 零拷贝路径（不下载到 CPU）
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi -i test.mp4 -f null -

# 5. mpv 出画面（验证 EGL 导入）
mpv --hwdec=vaapi --vo=gpu-next --msg-level=all=v test.mp4
```

### 8.5 交叉编译（可选，用于 CI/快速迭代）

设备 8 核编译一个 < 10k 行的小项目只需十几秒，**日常开发用原生编译最简单**。只有在"设备不在手边"或"CI 出包"时才需要交叉编译：

```sh
# 宿主机（x86_64 Ubuntu）
sudo apt install gcc-aarch64-linux-gnu pkg-config
# 需要一个 aarch64 sysroot（可从设备 rsync /usr/include + /usr/lib，或用 debootstrap）
cat > aarch64-cross.ini <<'EOF'
[binaries]
c = 'aarch64-linux-gnu-gcc'
ar = 'aarch64-linux-gnu-ar'
strip = 'aarch64-linux-gnu-strip'
pkg-config = 'pkg-config'
[host_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'aarch64'
endian = 'little'
[properties]
sys_root = '/path/to/aarch64-sysroot'
EOF
meson setup build-arm --cross-file aarch64-cross.ini --prefix=/usr --libdir=lib/aarch64-linux-gnu
```

坑：`pkg-config` 默认会找宿主机 x86_64 的 `libva.pc`，必须设置 `PKG_CONFIG_SYSROOT_DIR`/`PKG_CONFIG_LIBDIR`，否则链接到错误架构。**首次交叉编译至少预留半天。**

### 8.6 打包成 deb（交付形态）

最小 `debian/` 结构（用 `dpkg-buildpackage` 或 `debianize` 生成）：

```
debian/control:
  Package: libva-iris
  Architecture: arm64
  Depends: libva2 (>= 2.20), libdrm2, libgbm1
  Description: VA-API driver for Qualcomm Iris V4L2 stateful decoder
debian/install:
  usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so
  usr/lib/aarch64-linux-gnu/dri/msm_drv_video.so
```

或者先用 `checkinstall -D` 快速出包，验证安装/卸载流程，再补正式打包。

### 8.7 调试环境变量汇总

| 变量 | 作用 |
|---|---|
| `LIBVA_DRIVER_NAME=iris` | 指定驱动 |
| `LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri` | 指定搜索目录 |
| `LIBVA_MESSAGING_LEVEL=3` | libva 日志（0–3，3 最详细） |
| `LIBVA_TRACE=/tmp/va.trace` | libva 调用跟踪（需 libva 编译时启用 trace） |
| `LIBVA_IRIS_DEBUG=1` | 本项目自定义：打印每次 ioctl/组包大小 |
| `LIBVA_IRIS_TIMEOUT_MS=5000` | 本项目自定义：放宽同步超时 |
| `MOZ_LOG=PlatformDecoderModule:5,FFmpeg:5` | Firefox 媒体日志 |
| `MOZ_DISABLE_CONTENT_SANDBOX=1` | 临时关 Firefox 内容沙箱（仅测试） |

### 8.8 部署检查清单

- [ ] `iris_drv_video.so` 在 `/usr/lib/aarch64-linux-gnu/dri/`，权限 `0644`
- [ ] `ldd` 无 `not found`（`libva`、`libdrm`、`libgbm`、`libc`）
- [ ] `nm -D` 能看到 `__vaDriverInit_1_0`
- [ ] `vainfo` 能在 render node 上列出 H.264 profile
- [ ] 当前用户在 `render`、`video` 组
- [ ] 若交付"零配置"，`msm_drv_video.so` 符号链接存在
- [ ] `apt remove`/`meson uninstall` 能干净卸载，不留 `.so` 残骸

---

## 9. 测试计划

### 9.1 测试分层

| 层 | 对象 | 运行环境 | 频率 | 通过标准 |
|---|---|---|---|---|
| 单元 | 码流重建器（SPS/PPS/EPB/ue） | 任意 Linux（x86 也行） | 每次提交 | 合成码流能被 ffmpeg 软解，帧数/内容与参考一致 |
| 组件 | V4L2 管道（不含 VA） | 设备 | 每日 | `tools/v4l2-probe` 能解 60 帧 NV12，md5 稳定 |
| 集成 | VA 驱动 + ffmpeg | 设备 | 每日 | `ffmpeg -hwaccel vaapi` 通过，PSNR 达标 |
| 端到端 | Firefox + B 站/本地文件 | 设备 | 每阶段 | CPU 降幅达标、画面正确、可 seek |
| 长稳 | 2 小时循环 | 设备 | 每周/发版前 | RSS 增长 < 5%，无 kernel oops，无 fd 泄漏 |

### 9.2 码流重建器单测（不需要硬件，优先做）

思路：**用 ffmpeg 的软件 H.264 解码器当"语法裁判"**。

```
测试素材（x264 生成，已知 SPS/PPS）
   │
   ├─ 用 ffmpeg -bsf:v trace_headers 或 h264_metadata 提取原始 SPS/PPS/slice
   │
   ├─ 只把 slice NAL 喂给重建器（模拟 VA-API 客户端）
   │     └─ 重建器输出 SPS + PPS + slice 的完整 AU
   │
   ├─ 拼成 synth.h264
   │
   └─ ffmpeg -f h264 -i synth.h264 -f null -
         ├─ 退出码 0 且无 "error while decoding" → 语法通过
         └─ 解码帧的 md5 与原始文件软解一致 → 语义通过
```

实现技巧：

- `ffmpeg -i in.mp4 -c copy -bsf:v h264_mp4toannexb -f h264 out.h264` 得到 Annex B；
- 写一个 `test/split_au.c`：按 AUD/`first_mb_in_slice==0` 切 AU，把 SPS/PPS 丢掉，只留 slice；
- 单测里对每个 AU 调重建器，再拼回去；
- 用 `-f md5` 或逐帧 `framemd5` 比对：

```sh
ffmpeg -f h264 -i synth.h264 -f framemd5 - | tail -n +5 > synth.md5
ffmpeg -f h264 -i orig.h264  -f framemd5 - | tail -n +5 > orig.md5
diff synth.md5 orig.md5
```

这个测试能覆盖 80% 的码流重建 bug（字段写错、ue 编码错、EPB 漏插、slice 顺序错），而且秒级反馈。

### 9.3 测试素材矩阵

用 ffmpeg 现场生成（体积小、可控），再加 2~3 个真实素材。

```sh
# 通用片源：testsrc2 有运动、有细节，比 testsrc 更适合查花屏
GEN="-f lavfi -i testsrc2=size=%s:rate=30 -t 8 -pix_fmt yuv420p"

# H.264 Baseline（CAVLC、无 B 帧）
ffmpeg $GEN -c:v libx264 -profile:v baseline -level 3.1 -g 30 \
       -x264-params "cabac=0:bframes=0:ref=1:8x8dct=0" baseline_720p.mp4

# H.264 Main（CABAC、B 帧）
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 8 -pix_fmt yuv420p \
       -c:v libx264 -profile:v main -bf 2 -g 60 -b:v 8M main_1080p.mp4

# H.264 High（8x8、多参考、B 帧）
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 8 -pix_fmt yuv420p \
       -c:v libx264 -profile:v high -bf 3 -refs 4 -g 60 -b:v 10M high_1080p.mp4

# H.264 High 4K
ffmpeg -f lavfi -i testsrc2=size=3840x2160:rate=30 -t 5 -pix_fmt yuv420p \
       -c:v libx264 -profile:v high -level 5.1 -bf 2 -b:v 40M high_4k.mp4

# HEVC Main 8bit
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 8 -pix_fmt yuv420p \
       -c:v libx265 -x265-params "bframes=3:ref=4:keyint=60" -b:v 8M hevc_1080p.mp4

# VP9 Profile0
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 8 -pix_fmt yuv420p \
       -c:v libvpx-vp9 -b:v 6M -g 60 vp9_1080p.webm

# 10bit（预期：驱动返回不支持，不崩）
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 4 -pix_fmt yuv420p10le \
       -c:v libx265 -x265-params "bframes=3" -b:v 8M hevc_10bit.mp4
```

| 维度 | 取值 | 用途 |
|---|---|---|
| Codec | H.264 / HEVC / VP9 | 分阶段支持 |
| Profile | Baseline / Main / High / High10 / HEVC Main / Main10 / VP9 P0 / P2 | profile 矩阵（§5.1） |
| 分辨率 | 720p / 1080p / 4K | 尺寸上限、带宽 |
| 帧率 | 30 / 60 | 队列深度、丢帧 |
| 码率 | 低（2M）/ 高（40M） | `sizeimage` 是否够 |
| 结构 | 单 slice / 多 slice（`-x264-params slices=4`）/ B 帧 / 无 B 帧 | 组包与重排 |
| 特殊 | 分辨率变化（两段 TS 拼接）/ 10bit / 场编码（预期拒绝） | 异常路径 |

**真实素材**（比合成素材更能暴露问题）：B 站 1080p H.264（用 `yt-dlp` 抓公开视频，注意版权与站点条款）、手机拍摄的 H.264 视频、`ffmpeg` 转码的老电影（多 B 帧/多参考）。

### 9.4 正确性比对：md5 与 PSNR

**md5（最快，用于回归）**

```sh
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f rawvideo -pix_fmt nv12 - | md5sum
ffmpeg -v error -i in.mp4 -f rawvideo -pix_fmt nv12 - | md5sum
```

注意：md5 一致的前提是两条路径的 NV12 转换方式一致。VA-API 输出的是解码器原生 NV12，软解输出 `yuv420p` 再转 NV12 可能因色度插值/取整不同而不一致。**md5 不一致不等于驱动错**，用 PSNR 复核。

**PSNR（判定标准）**

```sh
# 分别导出
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f rawvideo -pix_fmt nv12 -s 1920x1080 va.yuv
ffmpeg -v error -i in.mp4 -f rawvideo -pix_fmt nv12 -s 1920x1080 sw.yuv

# 逐帧 PSNR
ffmpeg -f rawvideo -pix_fmt nv12 -s 1920x1080 -i va.yuv \
       -f rawvideo -pix_fmt nv12 -s 1920x1080 -i sw.yuv \
       -lavfi "[0:v][1:v]psnr=stats_file=/tmp/psnr.log" -f null -
```

| 结果 | 判定 |
|---|---|
| `psnr_avg: inf` | 完全一致（理想） |
| `psnr_avg > 45 dB` | 可接受（差异来自色度上采样/取整） |
| 40–45 dB | 可疑：检查是否有轻微错帧/色度错位 |
| < 40 dB 或个别帧突降 | 有 bug（参考帧管理/裁剪/字段错误） |

**首帧目检**：

```sh
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -frames:v 1 -vf "hwdownload,format=nv12" /tmp/first.png
```

重点看：底部 8 行（1088 裁剪问题）、色度是否错位、是否有绿色/紫色块。

### 9.5 CPU 与功耗测量

**CPU（60 秒平均，可复现）**

最省事的方式是用 `sysstat` 的 `pidstat`：

```sh
sudo apt install sysstat
pidstat -h -p ALL 1 60 > /tmp/pidstat.log
# 过滤 Firefox 相关进程，把 %CPU 求和后取平均
grep -E 'firefox|Web Content|RDD Process|Isolated Web Content' /tmp/pidstat.log \
  | awk '{s+=$8; n++} END {printf "avg sum CPU: %.1f%%\n", s/n}'
```

（`pidstat` 的列号随版本变化，先 `head -3` 确认 `%CPU` 在第几列。）

**对比方法**：

1. 同一视频、同一窗口大小、同一分辨率，播放 60 s；
2. 软解基线：`media.ffmpeg.vaapi.enabled=false`；
3. 硬解：`media.ffmpeg.vaapi.enabled=true`；
4. 各测 3 次取中位数；
5. 同时记录 `gnome-shell`/`Xwayland` 的 CPU，避免把合成开销算到解码头上。

**功耗（本机可行，因为能读到电池）**：

```sh
while true; do
  i=$(cat /sys/class/power_supply/battery/current_now)
  v=$(cat /sys/class/power_supply/battery/voltage_now)
  echo "$(date +%s) $((i*v/1000000000000))"   # 单位 W（current_now 可能为负）
  sleep 1
done
```

对比软解/硬解各 5 分钟的平均功率。注意充电状态下 `current_now` 可能为正（充入），要拔掉电源测。

### 9.6 内存与资源泄漏

设备上大概率没有 `valgrind`（即使有，ARM64 上跑 Firefox 也不现实）。替代方案：

| 层面 | 方法 | 判据 |
|---|---|---|
| 用户态堆 | `heaptrack`（`apt install heaptrack`）跑 `test/va-dump` 循环 1000 次 | 峰值不随循环次数线性增长 |
| 用户态错误 | ASan 构建：`meson setup build-asan -Db_sanitize=address,undefined`，跑单测与 `va-dump` | 无 leak/UAF 报告 |
| 进程 RSS | `grep Pss /proc/<pid>/smaps_rollup` 每 10 s 采样，2 小时 | 增长 < 5%，且趋于平稳 |
| fd 泄漏 | 统计 `/proc/<pid>/fd` 的条目数，每 10 s 采样 | 不持续增长（重点查 dma-buf fd） |
| 内核 dma-buf | `sudo cat /sys/kernel/debug/dma_buf/bufinfo`（若存在） | buffer 数量稳定，无持续增长 |
| 内核日志 | `dmesg` 输出里搜 `vb2`/`iris`/`dma_buf`/`iommu` 关键字 | 无 `WARNING`/`BUG`/`IOMMU fault` |
| 内核泄漏 | `/sys/kernel/debug/kmemleak`（若开启） | 无新增记录 |

**fd 泄漏是最常见的**：每次 `vaExportSurfaceHandle` 都 `dup` 一个 fd，Firefox 关闭 surface 时必须释放。建议在驱动里给 fd 加引用计数，并在 `vaDestroySurfaces` 时断言 `refcount == 1`。

### 9.7 回归测试清单（每次发版跑）

- [ ] `vainfo` 输出与上一版一致（profile 列表、driver version）
- [ ] 9.3 矩阵中所有"应支持"的素材：解码帧数正确、PSNR ≥ 45 dB
- [ ] 所有"应拒绝"的素材（10bit/AV1/场编码）：返回明确错误码，进程不崩
- [ ] seek 20 次（随机位置）：无花屏、无卡死、无 fd 增长
- [ ] 暂停/恢复 10 次：无花屏
- [ ] 窗口从 1080p 缩到 480p 再放大：无花屏（这是合成问题，但要确认不是解码问题）
- [ ] 两个标签页同时播 1080p：CPU 可接受，无互相卡死
- [ ] 挂起/唤醒一次（本机支持 `rtcwake`）：恢复后能继续播
- [ ] 2 小时循环播放：RSS/fd 稳定
- [ ] `dmesg` 无 oops/WARNING

### 9.8 性能基准（记录在 `BENCH.md`）

| 场景 | 指标 | 软解 | 硬解 | 提升 |
|---|---|---|---|---|
| 1080p30 H.264 High，mpv | CPU% | | | |
| 1080p30 H.264 High，Firefox | CPU% | 140–170% | | |
| 4K30 H.264 High，mpv | CPU% | | | |
| 1080p30，整机功耗 | W | | | |
| seek 到 60 s 的耗时 | ms | | | |
| 首帧时间 | ms | | | |

---

## 10. 里程碑与工作量估算

估算前提：1 名熟悉 Linux 图形/媒体栈的工程师，全职；设备在手；网络可用（283 kB/s）；不含"等上游修 bug"的时间。

| 阶段 | 交付内容 | 验收 | 人天（乐观/悲观） | 依赖 | 主要风险 |
|---|---|---|---|---|---|
| **M0 先例审计 + 事实核对** | 读完 `mxsrc/radxa libva-v4l2`、Debian `qcom-va-driver`；核对 ffmpeg `start_frame` 是否给整包；`grep` 内核 `v4l2_m2m_buf_copy_metadata`；`EXPBUF` 探针 | 决策：fork / 新写 / 直接用 | 2 / 4 | 网络 | 结论可能推翻计划 |
| **M1 驱动骨架** | `iris_drv_video.so` 能加载，`vainfo` 列出 H.264 Baseline/Main/High；`vaCreateConfig`/`vaCreateContext` 能开 `/dev/video0` 并 `S_FMT`/`REQBUFS`/`STREAMON` | L0 通过 | 3 / 5 | M0 | libva ABI 细节、iris `S_FMT` 限制 |
| **M2 单帧 H.264** | 码流重建器（SPS/PPS/AUD）+ 一次 `QBUF` + `DQBUF` + `vaDeriveImage` 导出 NV12；baseline 720p 首帧 PSNR > 45 dB | `ffmpeg -hwaccel vaapi -frames:v 1` 出 PNG，画面正确 | 5 / 9 | M1 | 重建器字段错、裁剪 |
| **M3 连续播放 + seek** | 多帧循环、B 帧重排（timestamp 映射）、`vaSyncSurface`、flush/seek、多 slice | 1080p High 全片 PSNR ≥ 45 dB；seek 20 次正常 | 4 / 8 | M2 | 时间戳不复制、seek 后非 IDR |
| **M4 零拷贝导出** | `vaExportSurfaceHandle` + dma-buf + EGL 导入（先用 `mpv --hwdec=vaapi` 验证） | L2 通过；1080p30 `mpv` 无丢帧 | 3 / 6 | M3 | `EXPBUF` 不支持、modifier/stride |
| **M5 Firefox 集成（调试态）** | deb 版 Firefox + `security.sandbox.content.level=0` + `media.ffmpeg.vaapi.enabled=true`，B 站 1080p 走硬解 | L3 通过（CPU ≤ 60%） | 3 / 6 | M4 | Firefox 版本行为、RDD 进程 |
| **M6 沙箱生产化 + 打包** | S2（补丁）或 S3（helper）落地；deb 包；`msm_drv_video.so` 兼容链接；文档 | 不关沙箱也能硬解；`apt install` 可用 | 4 / 10 | M5 | 需要重编 Firefox；上游接受度 |
| **M7 HEVC** | VPS/SPS/PPS 重建（或 R2 直通兜底） | HEVC 1080p PSNR ≥ 45 dB | 8 / 18 | M4 | RPS 信息不足，可能做不成 |
| **M8 VP9** | VP9 uncompressed header 重建 | VP9 1080p PSNR ≥ 45 dB | 5 / 9 | M4 | 字段多、易错 |
| **M9 稳定化/性能/文档** | 2 小时长稳、泄漏修复、CPU 调优、用户文档、`BENCH.md` | L4 通过 | 4 / 7 | M6 | 长尾 bug |

**汇总**：

| 范围 | 人天（乐观） | 人天（悲观） | 日历时间（1 人） |
|---|---|---|---|
| H.264 + Firefox（M0–M6） | 24 | 48 | 5–9 周 |
| 加 HEVC + VP9（M7–M8） | 37 | 75 | 8–15 周 |
| 加稳定化（M9） | 41 | 82 | 9–16 周 |

**关键路径**：M0 → M1 → M2 → M3 → M4 → M5 → M6。M7/M8 可以与 M5/M6 并行（不同的人）。

**如果只有 1 周**：只做 M0 + M1 + M2 的 baseline 部分，目标是"`ffmpeg -hwaccel vaapi` 解出第一帧 PNG"（见 §13）。

---

## 11. 风险清单

按"会不会让项目失败"排序。概率/影响用 高/中/低 表示。

| # | 风险 | 概率 | 影响 | 触发信号 | 缓解/兜底 |
|---|---|---|---|---|---|
| R1 | **VA-API 不提供 SPS/PPS，stateful 需要重建，字段不全导致解不出/花屏** | 高 | 高 | `ffmpeg` 报 "decode error"、PSNR 低、花屏 | 先做 §9.2 的软解单测；H.264 字段基本齐全；实在不行走 R2 直通（重编 ffmpeg/Firefox） |
| R2 | **HEVC 的 RPS 无法从 VA 参数重建** | 高 | 中 | HEVC 全黑/报错 | HEVC 阶段直接改用 R2；或在驱动里缓存"每个 IDR 的 SPS/PPS 原样"——但拿不到原样，只能 R2 |
| R3 | **Firefox 内容进程 seccomp 拒绝 `/dev/video0`** | 高 | 高 | `vaCreateContext` 失败、`strace` 看到 EACCES | §7.4 的 S1→S4→S2→S3；先关沙箱证明链路，再决定补丁/helper |
| R4 | **snap 版 Firefox 不可用**（驱动不可见 + AppArmor） | 高 | 中 | `snap run --shell` 里看不到 so/设备 | 换 Mozilla deb（项目已有先例）；snap 不作为交付目标 |
| R5 | **10bit 内容无法硬解**（capture 只有 NV12） | 确定 | 中 | HEVC Main10 回退软解 | 明确写进文档与 profile 列表；后续内核支持 P010 再扩展 |
| R6 | **`SOURCE_CHANGE` 与固定尺寸 VA surface 冲突** | 中 | 中 | 切清晰度黑屏/报错 | 新尺寸 ≤ 旧尺寸时继续；更大时返回 `RESOLUTION_NOT_SUPPORTED`，依赖客户端重建；B 站多数切换是新建 decoder |
| R7 | **iris 不复制 timestamp，surface↔帧无法关联** | 中 | 高 | 画面顺序错乱/闪回 | M1 `grep` 内核；没有就给内核打一行 `v4l2_m2m_buf_copy_metadata`；再不行串行化模式仅用于功能验证 |
| R8 | **`VIDIOC_EXPBUF` 不被 iris 支持** | 中 | 低 | `EXPBUF` 返回 ENOTTY/EINVAL | MVP 本来就用拷贝模式；零拷贝降级为 L2 可选优化 |
| R9 | **导出的 dma-buf 不能被 freedreno EGL 导入**（stride/modifier/格式） | 中 | 高 | 黑屏、EGL 错误 | 强制 LINEAR、256 字节对齐、`GR88` 两层；用 `eglinfo` + 最小 EGL 导入程序隔离 |
| R10 | **iris 驱动/FW 本身有 bug**（hang、花屏、4K 不支持） | 中 | 高 | `dmesg` 报错、`poll` 超时 | 用 `v4l2-ctl`/ffmpeg `v4l2m2m` 先证明内核路径；必要时给内核打补丁（单独评估） |
| R11 | **Firefox 即便一切正常也不走 VA-API**（blocklist/构建选项/版本行为） | 中 | 高 | `about:support` 显示未启用 | `media.hardware-video-decoding.force-enabled=true`；换 Firefox 版本；用 mpv/GStreamer 证明驱动没问题；最终兜底是"维护一个打补丁的 mpv/ffmpeg" |
| R12 | **性能不达标**（拷贝/同步开销） | 中 | 中 | CPU 只降到 90% | 上零拷贝；减少 `poll` 次数；批量 `DQBUF`；检查 WebRender 合成开销 |
| R13 | **多线程/生命周期 bug**（seek 崩溃、fd 泄漏） | 中 | 中 | 随机崩溃、fd 增长 | 单锁 + 严格所有权（§6.1）；ASan + fd 计数 |
| R14 | **libva ABI/版本差异**（2.23 vs 更新的 2.24） | 低 | 中 | 加载失败、vtable 偏移错 | 用 `VA_MAJOR_VERSION/VA_MINOR_VERSION` 条件编译；CI 里跑两个 libva 版本 |
| R15 | **上游不接受内核补丁，长期要带 patch** | 中 | 低 | review 被拒 | 补丁很小（一行 metadata 复制），先进项目内核仓库，再推上游 |
| R16 | **维护成本**（Firefox/内核/Mesa 升级破坏兼容） | 高 | 低 | 升级后回归失败 | 建立 §9.7 回归清单；CI 跑 `vainfo` + 固定素材；锁版本 |

**兜底方案（当 Firefox 路线彻底走不通时）**：

1. **Chromium**：`VaapiVideoDecoder` 同样需要 VA-API；如果我们的驱动在 `vainfo`/`mpv` 上没问题，Chromium 大概率能用（它对 sandbox 的处理不同，`/dev/video0` 可能仍需 `--no-sandbox` 或 policy）。
2. **mpv/ffmpeg 打补丁**：维护一个带 R2 直通的 ffmpeg + `mpv --hwdec=vaapi`，至少本地播放能用；这是"用户体验兜底"，不是浏览器方案。
3. **Waydroid/安卓 App**：安卓 B 站客户端走的是安卓的 Codec2，不经过 Linux VA-API，与本项目无关（但可作为"看视频"的替代路径）。

---

## 12. 附录

### 12.1 关键命令清单

**设备与格式**

```sh
v4l2-ctl -d /dev/video0 --all
v4l2-ctl -d /dev/video0 --info
v4l2-ctl -d /dev/video0 --list-formats-out
v4l2-ctl -d /dev/video0 --list-formats
v4l2-ctl -d /dev/video0 -L | grep -iE 'h264|hevc|vp9|min_buffers'
v4l2-ctl -d /dev/video0 --get-fmt-video-out
v4l2-ctl -d /dev/video0 --get-fmt-video
```

**VA**

```sh
# vainfo 需要先装
sudo apt install vainfo
LIBVA_DRIVER_NAME=iris LIBVA_MESSAGING_LEVEL=3 \
  vainfo --display drm --device /dev/dri/renderD128
LIBVA_DRIVER_NAME=iris vainfo --display drm --device /dev/dri/renderD128
```

**ffmpeg**

```sh
ffmpeg -hide_banner -hwaccels
ffmpeg -hide_banner -decoders | grep -E 'vaapi|v4l2m2m'

# 解码不到 CPU（最快验证）
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f null -

# 解码 + 下载 NV12（用于 PSNR/md5）
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f rawvideo -pix_fmt nv12 va.yuv

# 零拷贝路径
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi -i in.mp4 -f null -

# 首帧 PNG
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -frames:v 1 -vf "hwdownload,format=nv12" first.png
```

**GStreamer（验证 V4L2 通路，不经过 VA）**

```sh
gst-inspect-1.0 v4l2h264dec
gst-launch-1.0 filesrc location=in.mp4 ! qtdemux ! h264parse ! \
  v4l2h264dec ! video/x-raw,format=NV12 ! fakesink
GST_DEBUG=v4l2*:5,v4l2videodec:6 gst-launch-1.0 ... 2>&1 | head -100
```

**mpv**

```sh
mpv --hwdec=vaapi --vo=gpu-next --msg-level=all=v in.mp4
mpv --hwdec=vaapi --vd-lavc-software-fallback=no in.mp4   # 禁止回退，验证是否真硬解
```

**Firefox**

```sh
# 确认版本与包来源
snap list firefox            # 或 dpkg -l firefox
# 启动并抓媒体日志
MOZ_LOG="PlatformDecoderModule:5,FFmpeg:5,DMABuf:5" \
MOZ_LOG_FILE=/tmp/ffmedia.log \
LIBVA_DRIVER_NAME=iris \
firefox
# 抓 openat
strace -f -e trace=openat -o /tmp/ff.strace firefox
grep -n 'video0\|renderD128' /tmp/ff.strace
```

**内核与调试**

```sh
uname -a
dmesg | grep -iE 'iris|vb2|dma_buf|iommu'
ls -l /dev/dri /dev/video*
cat /sys/kernel/debug/dma_buf/bufinfo 2>/dev/null | head
grep -n 'v4l2_m2m_buf_copy_metadata' -r /path/to/linux/drivers/media/platform/qcom/iris/
```

### 12.2 本机实测输出摘录

> 以下为 2026-10-26/27 在设备上采集到的字段级结果（不是完整命令输出），用于回归对照。**改动环境后先复核这张表。**

| 查询 | 实测值 |
|---|---|
| `v4l2-ctl --info` driver | `iris_driver` |
| card | `Iris Decoder` |
| bus_info | `platform:aa00000.video-codec` |
| Device Caps | `0x04204000` = `V4L2_CAP_VIDEO_M2M_MPLANE`(0x4000) \| `V4L2_CAP_EXT_PIX_FORMAT`(0x200000) \| `V4L2_CAP_STREAMING`(0x4000000) |
| OUTPUT 格式 | `H264`、`HEVC`、`VP90`，均带 `V4L2_FMT_FLAG_DYN_RESOLUTION` |
| CAPTURE 格式 | `NV12`（无 10bit 格式） |
| `/dev/dri/renderD128` | `crw-rw---- root:render 226,128`，存在 |
| `/dev/dri/card0` | 存在 |
| VA 驱动目录 | `d3d12_drv_video.so`、`nouveau_drv_video.so`、`r600_drv_video.so`、`radeonsi_drv_video.so`、`virtio_gpu_drv_video.so`；**无 msm/iris** |
| `libva2` | `2.23.0-1ubuntu1` |
| `vainfo` | **未安装**（仓库有 `2.22.0+ds1-2build1`） |
| Mesa | `26.0.3`；`libEGL_mesa.so.0`、`libgbm.so.1`、`dri/msm_dri.so -> libdril_dri.so` |
| GStreamer | `1.28.2`；`v4l2h264dec`/`v4l2h265dec`/`v4l2vp9dec`（Rank primary+1, Hardware） |
| ffmpeg | `8.0.1`；`h264_v4l2m2m`/`hevc_v4l2m2m`；720p H.264 实测可解 |
| mpv | `0.41`；`--hwdec=v4l2m2m` 不支持 |
| Firefox | `149.0.2-1`（snap）；软解 1080p 时全进程 CPU 140%~170% |
| 网络 | 约 283 kB/s |

### 12.3 参考资料

**libva / VA-API**

- libva 源码与头文件：<https://github.com/intel/libva>
- libva API 文档：<https://intel.github.io/libva/>
- `va/va_backend.h`（驱动 ABI）：<https://github.com/intel/libva/blob/master/va/va_backend.h>
- `va/va_dec_h264.h` / `va_dec_hevc.h` / `va_dec_vp9.h`：同仓库 `va/` 目录
- `va/va_drmcommon.h`（`VADRMPRIMESurfaceDescriptor`）：同仓库

**stateful V4L2**

- 内核文档：Memory-to-Memory Stateful Video Decoder Interface：<https://docs.kernel.org/userspace-api/media/v4l/dev-decoder.html>
- 内核文档：Memory-to-Memory Devices：<https://docs.kernel.org/userspace-api/media/v4l/dev-mem2mem.html>
- 内核源码：`drivers/media/platform/qcom/iris/`（本机内核仓库同路径）
- 内核 `v4l2-mem2mem.c`（`v4l2_m2m_buf_copy_metadata`、`V4L2_BUF_FLAG_LAST`）
- `linux/videodev2.h` 的 `V4L2_DEC_CMD_STOP`、`V4L2_EVENT_SOURCE_CHANGE`、`V4L2_FMT_FLAG_DYN_RESOLUTION`

**参考驱动 / 先例**

- `bootlin/libva-v4l2-request`（stateless）：<https://github.com/bootlin/libva-v4l2-request>
- `mxsrc/libva-v4l2`（stateful M2M）：<https://github.com/mxsrc/libva-v4l2>
- `radxa-pkg/libva-v4l2`（stateful M2M 的 fork/打包）：<https://github.com/radxa-pkg/libva-v4l2>
- Debian WNPP `#1152234` `qcom-va-driver`：<https://bugs.debian.org/1152234>
- Chromium `V4L2VideoDecodeAccelerator`：<https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/v4l2/v4l2_video_decode_accelerator.cc>

**ffmpeg / GStreamer**

- `libavutil/hwcontext_vaapi.c`（`vaExportSurfaceHandle` 调用点）：<https://github.com/FFmpeg/FFmpeg/blob/master/libavutil/hwcontext_vaapi.c>
- `libavcodec/vaapi_decode.c` / `vaapi_h264.c`：同仓库
- `libavcodec/v4l2_m2m.c` / `v4l2_m2m_dec.c`：stateful M2M 用法参考
- GStreamer `gstv4l2videodec.c`、`gstv4l2bufferpool.c`：<https://gitlab.freedesktop.org/gstreamer/gstreamer>

**Firefox**

- `FFmpegVideoDecoder.cpp`：<https://searchfox.org/mozilla-central/source/dom/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp>
- `DMABufSurface.cpp`：<https://searchfox.org/mozilla-central/source/gfx/2d/DMABufSurface.cpp>
- 沙箱策略：`security/sandbox/linux/broker/SandboxBrokerPolicyFactory.cpp`
- 首选项：`modules/libpref/init/StaticPrefList.yaml`（搜 `media.ffmpeg.vaapi`）
- Firefox 源码里所有 VA-API 相关位置：<https://searchfox.org/mozilla-central/search?q=vaapi>

**图形**

- `EGL_EXT_image_dma_buf_import`：<https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import.txt>
- `EGL_EXT_image_dma_buf_import_modifiers`：同目录
- DRM format modifiers：<https://docs.kernel.org/gpu/drm-kms.html>
- Mesa freedreno：<https://docs.mesa3d.org/drivers/freedreno.html>
- libgbm：<https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gbm>

**编解码规范**

- ITU-T Rec. H.264（含 SPS/PPS/slice header 语法）：<https://www.itu.int/rec/T-REC-H.264>
- ITU-T Rec. H.265（HEVC）：<https://www.itu.int/rec/T-REC-H.265>
- VP9 bitstream spec：<https://www.webmproject.org/vp9/>

### 12.4 术语表

| 术语 | 含义 |
|---|---|
| **stateful decoder** | 内核驱动解析码流、管理参考帧，用户态只喂整帧码流；V4L2 的 M2M 模型 |
| **stateless decoder** | 用户态解析码流并通过 Request API 提交 per-slice 参数；V4L2 的 Request API 模型 |
| **M2M** | memory-to-memory，V4L2 的编解码框架（OUTPUT 队列进、CAPTURE 队列出） |
| **AU** | access unit，一个完整的编码图像（H.264 的一帧） |
| **RBSP / EPB** | 原始字节序列载荷 / 防竞争字节（`00 00 03`） |
| **SPS / PPS / VPS** | 序列参数集 / 图像参数集 / 视频参数集 |
| **VLD** | VA-API 的变长解码入口点（`VAEntrypointVLD`） |
| **DRM_PRIME_2** | VA surface 导出为 dma-buf 的现代接口（`VADRMPRIMESurfaceDescriptor`） |
| **DMABuf / dma-buf** | 跨设备共享缓冲区的内核框架 |
| **WebRender** | Firefox 的 GPU 合成器；DMABuf 纹理由它采样 |
| **RDD** | Firefox 的远程数据解码进程（可能承载 ffmpeg） |

---

## 13. 如果只做一件事：一周 MVP

**目标（可验收）**：7 天内，在设备上用 `ffmpeg -hwaccel vaapi` 解出一帧 **H.264 Baseline 720p** 的正确画面（PNG），且 `vainfo` 能列出 `VAProfileH264Baseline/Main/High : VAEntrypointVLD`。**不要求 Firefox、不要求 B 帧、不要求零拷贝。**

### 13.1 为什么选这个范围

- Baseline 没有 B 帧 → 不需要处理重排，surface↔帧的映射可以用 FIFO，绕开 timestamp 风险（R7）；
- Baseline 是 CAVLC、无 8x8 transform、无 scaling list → SPS/PPS 重建器可以砍掉一半字段；
- `ffmpeg -hwaccel vaapi` 是最容易控制的客户端，出错能看到明确日志；
- 一帧 PNG 就能证明"VA 驱动 → V4L2 → 硬件 → dma-buf → 导出"整条链路，后面只是加功能。

### 13.2 七天排期

| 天 | 任务 | 交付/验收 |
|---|---|---|
| **D1** | 装依赖（§8.1）；`v4l2-ctl` 复核设备；用 GStreamer/`ffmpeg -c:v h264_v4l2m2m` 证明内核通路；clone 并粗读 `mxsrc/radxa libva-v4l2`；写 `tools/v4l2-probe.c`：直接喂 Annex B 文件、MMAP、`DQBUF`、dump 首帧 NV12 | `v4l2-probe` 能 dump 出 720p NV12，转 PNG 画面正确；`vainfo` 可运行（未装驱动时预期报错） |
| **D2** | 驱动骨架：`VA_DRIVER_INIT_FUNC`、vtable、`vaQueryConfigProfiles`、`vaGetConfigAttributes`、`vaCreateConfig`、`vaDestroyConfig` | `vainfo` 退出码 0，输出含 H.264 三个 profile（此时 `vaCreateContext` 可返回 `UNIMPLEMENTED`） |
| **D3** | `vaCreateContext`（`open`/`SUBSCRIBE_EVENT`/`S_FMT`/`REQBUFS`/`STREAMON`）、`vaCreateSurfaces`（拷贝模式：gbm 或 `memfd` + `mmap`）、`vaDeriveImage`/`vaMapBuffer` | 空跑不崩；`vaCreateContext` 后 `v4l2-ctl` 能看到格式已设置 |
| **D4** | 码流重建器：`bs_writer`（位写入/ue/EPB）+ H.264 SPS/PPS（baseline 子集）+ AU 组装；用 §9.2 的软解单测验证 | `ffmpeg -f h264 -i synth.h264 -f null -` 通过；帧 md5 与原始一致 |
| **D5** | 打通 `vaBeginPicture`/`vaRenderPicture`/`vaEndPicture`/`vaSyncSurface`；`ffmpeg -frames:v 1` 出 PNG | **MVP 核心验收**：`/tmp/mvp.png` 画面正确 |
| **D6** | 连续 30 帧；PSNR 对比；修 bug（重点：`bytesused`、`sizeimage`、`stride`、裁剪） | 30 帧 PSNR ≥ 45 dB |
| **D7** | 收尾：`accept.sh`、README、记录阻塞点（timestamp/EXPBUF/sandbox 的验证结果）、下一阶段计划 | 可交接的仓库 + 一页结论 |

### 13.3 MVP 的验收命令

```sh
# 1. 驱动加载
LIBVA_DRIVER_NAME=iris vainfo --display drm --device /dev/dri/renderD128 \
  | grep -E 'VAProfileH264(Baseline|Main|High).*VAEntrypointVLD'

# 2. 出一帧
LIBVA_DRIVER_NAME=iris ffmpeg -v error \
  -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
  -i baseline_720p.mp4 -frames:v 1 \
  -vf "hwdownload,format=nv12" /tmp/mvp.png

# 3. 看画面
xdg-open /tmp/mvp.png
```

### 13.4 MVP 明确不做的事

- ❌ B 帧 / 重排 / timestamp 映射（留给 M3）
- ❌ seek / flush / 分辨率变化（留给 M3）
- ❌ 零拷贝 / `vaExportSurfaceHandle`（留给 M4）
- ❌ Firefox / sandbox / snap（留给 M5/M6）
- ❌ HEVC / VP9 / 10bit（留给 M7/M8）
- ❌ 多 slice（先假设单 slice；D6 若遇到多 slice 再补）

### 13.5 MVP 可能卡住的地方与应对

| 卡点 | 应对 |
|---|---|
| `vainfo` 加载失败（符号/路径/libva 版本） | 用 `LD_DEBUG=libs`、`nm -D`、`LIBVA_MESSAGING_LEVEL=3` 定位；先跑 `LIBVA_MESSAGING_LEVEL=3 vainfo --display drm --device /dev/dri/renderD128` 看详细 |
| `S_FMT` 被拒（`EINVAL`） | 先 `TRY_FMT` 看驱动接受的尺寸/格式；确认 `num_planes=1`、`sizeimage` 合理 |
| `QBUF` 后 `DQBUF` 一直 `EAGAIN` | 打开 `LIBVA_IRIS_DEBUG=1` 打印每次 ioctl；用 `v4l2-probe` 在同一码流上对比，确认是"驱动问题"还是"组包问题" |
| 组包后软解都不通过 | 回到 §9.2 单测，逐字段对照 `ffmpeg -bsf:v trace_headers` 的输出 |
| 出图但花屏/绿边 | 先查 `bytesperline`/`uv_offset`/裁剪（§3.7），再查色度平面顺序（NV12 是 UV 交错，不是 U 平面 + V 平面） |
| 时间不够 | 砍 D6 的连续帧，只交付"单帧 + 单测"，把问题记录清楚；**不要为了赶时间跳过 D4 的单测**，否则后面所有 bug 都会怀疑到重建器 |

### 13.6 一周之后

MVP 通过后，按 §10 的顺序推进：M3（连续/seek）→ M4（零拷贝）→ M5（Firefox 调试态）→ M6（沙箱/打包）。**每完成一个 M，跑一次 §9.7 的回归清单**，不要等到最后。

---

## 附：开工第一小时清单

1. `sudo apt install -y build-essential meson ninja-build pkg-config libva-dev libdrm-dev libgbm-dev vainfo v4l-utils`（约 8–12 分钟）
2. `v4l2-ctl -d /dev/video0 --all`，把输出贴进 `NOTES.md`，与 §12.2 对照
3. `git clone https://github.com/mxsrc/libva-v4l2 && cd libva-v4l2 && ls src/`，回答三个问题：支持 stateful 吗？H.264 的 SPS/PPS 怎么来？有 `vaExportSurfaceHandle` 吗？
4. `grep -rn "v4l2_m2m_buf_copy_metadata" drivers/media/platform/qcom/iris/`（在内核源码里）
5. 写 `tools/v4l2-probe.c` 的第一个版本：`open` → `S_FMT` → `REQBUFS` → `QBUF` 一个 `.h264` 文件 → `DQBUF` → 存 NV12
6. 把 `v4l2-probe` 的输出用 `ffmpeg -f rawvideo -pix_fmt nv12 -s 1280x720 -i out.nv12 out.png` 转成图，确认画面正确

做完这 6 步，你就有了"硬件通路是通的"这个最重要的确定性；剩下的都是软件工程。





