# Xiaomi Pad 6 Pro锛坙iuqin锛夋祻瑙堝櫒纭欢瑙嗛瑙ｇ爜浠诲姟涔?
## 涓?qcom-iris锛坰tateful V4L2 M2M锛夊疄鐜?VA-API 椹卞姩锛岃 Firefox 鐢ㄤ笂纭В

| 椤?| 鍐呭 |
|---|---|
| 鏂囨。鐗堟湰 | v1.0锛?026-10-27锛?|
| 鐩爣璁惧 | Xiaomi Pad 6 Pro 512GB锛屼唬鍙?**liuqin**锛孲oC **SM8475**锛堥獊榫?8+ Gen 1锛夛紝GPU Adreno 730 |
| 鐩爣绯荤粺 | Ubuntu 26.04 LTS锛坮esolute锛夛紝鍐呮牳 `6.17.0-rc1-gfe81794b5e1b`锛堣嚜缂栵紝浠撳簱 `github.com/yzddmr6/linux-sm8450-liuqin`锛屽垎鏀?`liuqin-6.17`锛?|
| 鏄剧ず鏍?| Wayland + GNOME 49/50锛孧esa 26.0.3锛宖reedreno锛坄msm_dri.so`锛夛紝Vulkan ICD `freedreno_icd` |
| 鐩爣 | 璁╂祻瑙堝櫒锛堥瑕?Firefox锛屽叾娆?Chromium/mpv 浣滀负楠岃瘉鎵嬫锛夐€氳繃 VA-API 浣跨敤 `/dev/video0`锛坄iris_driver`锛夌‖瑙?H.264/HEVC/VP9 |
| 浜や粯鐗?| 涓€涓彲瀹夎鐨?VA-API 椹卞姩 `iris_drv_video.so`锛堟簮鐮?+ 鏋勫缓鑴氭湰 + 娴嬭瘯鑴氭湰 + 閮ㄧ讲璇存槑锛?|
| 鐘舵€?| 鍙紑宸ャ€傜 2 绔犵殑"鍏堜緥瀹¤"鏄涓€涓喅绛栭棬 |

---

## 0. 闃呰绾﹀畾锛氫簨瀹炲垎绾?
鏈枃鎵€鏈夌粨璁洪兘鏍囦簡鏉ユ簮绛夌骇锛?*涓嶈鎶娿€愬緟楠岃瘉銆戝綋鎴愬凡鐭ヤ簨瀹炴帓鏈?*銆?
| 鏍囪 | 鍚箟 | 澶勭悊鏂瑰紡 |
|---|---|---|
| 銆愬疄娴嬨€?| 2026-10-26/27 鍦ㄨ澶囦笂鐩存帴璺戝嚭鏉ョ殑缁撴灉 | 鍙洿鎺ヤ綔涓鸿璁¤緭鍏?|
| 銆愭枃妗ｃ€?| 鏉ヨ嚜鍐呮牳/libva/FFmpeg/瑙勮寖鏂囨。鐨勭‘瀹氳涓猴紝浣嗘湭鍦ㄦ湰鏈洪€愭潯楠岃瘉 | 寮€宸ユ椂鐢ㄤ竴鏉″懡浠ゅ鏍?|
| 銆愬緟楠岃瘉銆?| 鏈変緷鎹殑鎺ㄦ柇锛屾垨渚濊禆鏈満鍐呮牳/鍥轰欢鍏蜂綋瀹炵幇 | 蹇呴』鍐欒繘 M0/M1 鐨勯獙璇佹竻鍗曪紝楠岃瘉鍓嶄笉鎺掑悗缁伐鏈?|
| 銆愭绱€?| 鍙湪缃戜笂妫€绱㈠埌鏍囬/绠€浠嬶紝姝ｆ枃鏈锛堟湰鏂囨。缂栧啓鐜鏃犳硶璁块棶澶栫綉锛?| 寮€宸ョ涓€姝?clone/涓嬭浇鍚庡璁?|

### 0.1 宸茬‘璁ょ殑鐜浜嬪疄锛堛€愬疄娴嬨€戯級

**璁惧涓庡唴鏍?*

- 璁惧锛歑iaomi Pad 6 Pro 512GB锛坙iuqin锛夛紝SM8475锛孉dreno 730锛? 鏍革紙1脳X3 3.19 GHz + 3脳A715 2.75 GHz + 4脳A510 2.02 GHz锛夛紝10.2 GiB 鍙敤鍐呭瓨銆?- 绯荤粺锛歎buntu 26.04 LTS锛屽唴鏍?`6.17.0-rc1-gfe81794b5e1b`銆?- 妗岄潰锛歐ayland 浼氳瘽 + GNOME 49/50 绾ф闈紱Mesa 26.0.3锛沢nome-shell 鏃ュ織鏈?`Created gbm renderer for '/dev/dri/card0'`銆乣Obtained a high priority EGL context`銆?
**瑙嗛瑙ｇ爜鍣?*

- `/dev/video0`锛歞river name `iris_driver`锛宑ard type `Iris Decoder`锛宐us `platform:aa00000.video-codec`銆?- 鑳藉姏锛歚V4L2_CAP_VIDEO_M2M_MPLANE`锛坰tateful memory-to-memory锛屽骞抽潰锛夈€?- Device Caps 瀹炴祴 `0x04204000` = `V4L2_CAP_VIDEO_M2M_MPLANE`(0x00004000) | `V4L2_CAP_EXT_PIX_FORMAT`(0x00200000) | `V4L2_CAP_STREAMING`(0x04000000)锛?*纭鏄?stateful M2M锛屼笉鏄?stateless Request API**銆?- OUTPUT 鏍煎紡锛歚H264`銆乣HEVC`銆乣VP90`锛屽潎甯?`V4L2_FMT_FLAG_DYN_RESOLUTION`銆?- CAPTURE 鏍煎紡锛?*鍙灇涓惧埌 `NV12`**锛屾病鏈?`NV12_10`/`P010`/`NV12_10BE_8L128` 绛?10bit 鏍煎紡銆?
**DRM / 鍥惧舰**

- `/dev/dri/renderD128` 瀛樺湪锛坄crw-rw---- root:render 226,128`锛夛紝`/dev/dri/card0` 涔熷湪锛涘綋鍓嶇敤鎴?`yumo` 宸插湪 `render` 缁勩€?- Mesa 渚?`libEGL_mesa.so.0`銆乣libgbm.so.1`銆乣dri/msm_dri.so -> libdril_dri.so`锛坒reedreno锛夐綈澶囷紝dma-buf 鈫?EGL 瀵煎叆鐨勫浘褰晶鏉′欢婊¤冻銆?
**VA / 鐢ㄦ埛鎬佺幇鐘?*

- `libva2 2.23.0-1ubuntu1` 宸插畨瑁呫€?- **`vainfo` 鍛戒护琛屽伐鍏锋湭瀹夎**锛堜粨搴撻噷鏈?`vainfo 2.22.0+ds1-2build1`锛岄渶 `apt install vainfo`锛夈€傛鍓嶈褰?宸插畨瑁?鏈夎锛屼互鏈潯涓哄噯銆?- `/usr/lib/aarch64-linux-gnu/dri/` 涓幇鏈?VA 椹卞姩鍙湁 Mesa 鐨?`d3d12_drv_video.so`銆乣nouveau_drv_video.so`銆乣r600_drv_video.so`銆乣radeonsi_drv_video.so`銆乣virtio_gpu_drv_video.so`鈥斺€?*娌℃湁浠讳綍 msm/iris/Qualcomm 鐨?VA 椹卞姩**銆傚洜姝ゆ棤璁?`LIBVA_DRIVER_NAME` 璁炬垚浠€涔堬紝褰撳墠閮芥棤鍙敤椹卞姩銆?- Ubuntu 浠撳簱**娌℃湁** `libva-v4l2-request` 鍖咃紙`apt-cache search` 鏌ヤ笉鍒帮級銆?- GStreamer 1.28.2 鏈?`v4l2h264dec` / `v4l2h265dec` / `v4l2vp9dec`锛圧ank primary+1锛屾爣 Hardware锛岃蛋 stateful M2M锛夆€斺€擵4L2 杩欐潯璺湰韬槸閫氱殑銆?- ffmpeg 8.0.1 鏈?`h264_v4l2m2m` / `hevc_v4l2m2m` 瑙ｇ爜鍣紝瀹炴祴鑳借В 720p H.264銆?- mpv 0.41 鐨?`--hwdec=v4l2m2m` 涓嶆敮鎸侊紙mpv 鍙 vaapi/vdpau/nvdec/vulkan/drm锛夈€?- Firefox 149 鏄?snap 鍖咃紙`firefox 149.0.2-1`锛夛紱娴忚鍣ㄥ唴瑙嗛鐩墠 100% 杞В锛屽疄娴?Firefox 涓昏繘绋?+ 鍐呭杩涚▼鍚堣鍗犵敤 **140%~170% CPU**銆?- 缃戠粶瀹炴祴绾?**283 kB/s**锛?026-10-27 涓婂崍锛夈€?
### 0.2 鍏抽敭鍙傝€冨疄鐜帮紙銆愭绱€戯紝姝ｆ枃寰呭璁★級

| 椤圭洰 | 瀹氫綅 | 瀵规湰浠诲姟鐨勬剰涔?|
|---|---|---|
| `bootlin/libva-v4l2-request` | 鎶?**stateless** V4L2 Request API 瑙ｇ爜鍣ㄥ寘瑁呮垚 VA-API 椹卞姩 | 浠ｇ爜缁勭粐銆乣vaExportSurfaceHandle`銆乻urface/buffer 绠＄悊鐨勫弬鑰冿紱**瑙ｇ爜璺緞涓嶈兘澶嶇敤** |
| `mxsrc/libva-v4l2` / `radxa-pkg/libva-v4l2` | GitHub 椤圭洰锛岀畝浠嬫槑纭啓 "A VA-API driver backed by the Linux V4L2 **stateful** memory-to-memory decoder interface" | **鏈€鎺ヨ繎鏈换鍔?*鐨勭幇鎴愰鏋讹紝鍙兘宸茶В鍐?VA 鍙傛暟 鈫?瀹屾暣鐮佹祦"闂锛汳0 绗竴浠朵簨灏辨槸 clone 瀹¤ |
| Debian ITP `#1152234: qcom-va-driver -- VA-API driver for Qualcomm Iris V4L2 devices`锛?026-10锛?| 鏈変汉宸插悜 Debian 鎻愪氦"楂橀€?Iris 鐨?VA-API 椹卞姩"鎵撳寘鎰忓悜 | 鑻ュ睘瀹烇紝鍙兘鐩存帴澶嶇敤/鎵撳寘锛汳0 蹇呴』鏌ユ竻鍏舵簮鐮佸湴鍧€涓庡畬鎴愬害 |
| Chromium `V4L2VideoDecodeAccelerator`锛圕hromeOS 璺緞锛?| stateful V4L2 鐨勭敤鎴锋€佸皝瑁?| 闃熷垪/鏍煎紡鍗忓晢/鍒嗚鲸鐜囧彉鍖栫殑鍙傝€冿紝**涓嶆槸 VA-API** |
| ffmpeg `libavcodec/v4l2_m2m*.c`銆丟Streamer `gstv4l2videodec` | stateful M2M 鐨勬垚鐔熺敤娉?| 闃熷垪銆乣SOURCE_CHANGE`銆丒OS 澶勭悊鍙傝€?|

> 璇存槑锛氭湰鏂囨。缂栧啓鏃舵棤娉曡闂缃戯紙web_fetch 琚矙绠辨嫆缁濓級锛屼笂琛ㄤ袱涓」鐩殑姝ｆ枃娌℃湁瀹¤杩囷紝鍙湁鏍囬绾ц瘉鎹€?*涓嶈鎶?瀹冧滑宸茬粡鑳界敤"褰撲綔浜嬪疄**锛孧0 鐨勭粨璁哄彲鑳芥敼鍐欐暣涓鍒掞紙瑙?搂2.4 鍐崇瓥闂級銆?
### 0.3 灏氭湭楠岃瘉鐨勫亣璁撅紙寮€宸ュ墠蹇呴』閫愭潯鎵撳嬀锛?
杩欏紶琛ㄦ槸"鍋囪鍙拌处"銆備换浣曚竴鏉¤璇佷吉锛岄兘鍙兘鏀瑰彉璁捐鎴栧伐鏈熴€?
| # | 鍋囪 | 楠岃瘉鏂瑰紡 | 闃舵 |
|---|---|---|---|
| A1 | iris 鐨?CAPTURE buffer 鏀寔 `VIDIOC_EXPBUF` 瀵煎嚭 dma-buf | `tools/v4l2-probe` 璋?EXPBUF锛岀湅杩斿洖鍊?| M0 |
| A2 | iris 鍦?CAPTURE 瀹屾垚鏃惰皟鐢?`v4l2_m2m_buf_copy_metadata()`锛宼imestamp 浼氬甫鍥?| `grep` 鍐呮牳婧愮爜 + probe 鎵撳嵃 timestamp | M0/M1 |
| A3 | iris 鐨?CAPTURE 闃熷垪鎺ュ彈 `V4L2_MEMORY_DMABUF` | probe 鐢?DMABUF 妯″紡璺戜竴閬?| M0 |
| A4 | ffmpeg 鐨?`start_frame(buffer,size)` 鏀跺埌鐨勬槸鏁翠釜 packet锛圧2 鐩撮€氱殑鍓嶆彁锛?| 璇?ffmpeg 8.0.1 `h264dec.c` + 鎵撴棩蹇?| M0 |
| A5 | Firefox 鎹嗙粦鐨?`libmozavcodec` 鍚敤浜?VA-API hwaccel | `about:support` / `MOZ_LOG` / `strings libmozavcodec.so` | M5 |
| A6 | Firefox 鍐呭杩涚▼鐨?seccomp 涓嶅厑璁?`/dev/video0` | `strace` 鐪?`openat` 杩斿洖 EACCES | M5 |
| A7 | deb 鐗?Firefox 鑳借鍒?`LIBVA_DRIVER_NAME`/`LIBVA_DRIVERS_PATH` | 鍚姩鍚?`cat /proc/<pid>/environ` | M5 |
| A8 | libva 鍦ㄦ湭璁剧幆澧冨彉閲忔椂鐢?DRM 椹卞姩鍚嶏紙`msm`锛夊洖閫€ | `strace -e openat vainfo` | M1 |
| A9 | Mesa freedreno 鑳藉鍏?linear NV12 dma-buf锛坄R8`+`GR88` 涓ゅ眰锛?| 鏈€灏?EGL 瀵煎叆绋嬪簭 + `eglinfo` | M4 |
| A10 | `gbm_bo_create(GBM_FORMAT_NV12)` 鍦?msm 涓婂彲鐢?| 10 琛屾祴璇曠▼搴?| M3 |
| A11 | iris 鏀寔 4K锛?840脳2160锛夎В鐮?| `v4l2-ctl --set-fmt-video` + 4K 鏍风墖 | M3 |
| A12 | ffmpeg 鍦?`avcodec_flush_buffers()` 鏃堕攢姣佸苟閲嶅缓 VA context锛堣€屼笉鏄繚鐣欙級 | 璇?`vaapi_decode.c` + `LIBVA_IRIS_DEBUG` 鏃ュ織 | M3 |
| A13 | Firefox 鍦?`RESOLUTION_NOT_SUPPORTED` 鏃朵細閲嶅缓 decoder | 鏋勯€犲垎杈ㄧ巼鍙樺寲娴?+ `MOZ_LOG` | M6 |
| A14 | snap 鐗?Firefox 鐨?`/dev/video0` 琚?AppArmor 鎷掔粷銆侀┍鍔?so 涓嶅彲瑙?| `snap run --shell` + `snap connections` | M5 |
| A15 | `mxsrc/radxa libva-v4l2` 涓?Debian `qcom-va-driver` 鐨勭湡瀹炶兘鍔?| clone/鏌?WNPP | M0 |

---

## 1. 鐩爣涓庨獙鏀舵爣鍑?
### 1.1 涓€鍙ヨ瘽鐩爣

鍦?Ubuntu 26.04/liuqin 涓婁氦浠樹竴涓?VA-API 椹卞姩锛屼娇**鏈粡淇敼鐨?Firefox锛堟垨鑷冲皯 Mozilla 瀹樻柟 deb 鐗堬級**鍦ㄦ挱鏀?B 绔?1080p H.264 鏃讹紝瑙ｇ爜宸ヤ綔鐢?`/dev/video0` 鐨?iris 纭欢瀹屾垚锛孋PU 鍗犵敤浠?140%~170% 闄嶅埌 60% 浠ヤ笅锛岀敾闈笌杞В涓€鑷淬€?
### 1.2 鍒嗗眰楠屾敹锛圠0鈥揕4锛?
姣忎竴灞傞兘鏄?*鍙嫭绔嬮獙鏀躲€佸彲鐙珛浜や粯**鐨勶紝鍓嶄竴灞傜殑閫氳繃鏄悗涓€灞傚紑宸ョ殑鍓嶆彁銆?
| 灞?| 楠屾敹鍐呭 | 鍒ゅ畾鍛戒护/瑙傚療鐐?| 閫氳繃鏍囧噯 |
|---|---|---|---|
| **L0 椹卞姩鍙敤** | VA 椹卞姩鑳借 libva 鍔犺浇锛岃兘鍔涙煡璇㈡纭?| `vainfo --display drm --device /dev/dri/renderD128` | 閫€鍑虹爜 0锛涜緭鍑哄惈 `VAProfileH264High : VAEntrypointVLD`锛沗VA-API version` 涓?`libva 2.23` 鍖归厤锛沗Driver version` 鏄剧ず鏈」鐩増鏈彿 |
| **L1 鑳借В鐮侊紙鏈夋嫹璐濓級** | 鍗曟枃浠?H.264 瑙ｇ爜姝ｇ‘ | `ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 -i test_1080p_h264_high.mp4 -f null -` | `-v error` 鏃犺緭鍑猴紱`frame= N` 涓?`ffprobe -count_frames` 涓€鑷达紱閫愬抚 PSNR 瀵硅蒋瑙?鈮?50 dB锛堢悊鎯?inf锛?|
| **L2 闆舵嫹璐濆鍑?* | surface 鑳戒互 dma-buf 瀵煎嚭骞惰 EGL 閲囨牱 | `ffmpeg -hwaccel vaapi -hwaccel_output_format vaapi ... -f null -`锛沗mpv --hwdec=vaapi --vo=gpu-next test.mp4` | 涓嶇粡杩?`hwdownload` 涔熻兘鍑虹敾闈紱`mpv` 鐨?`--msg-level=vo=v` 涓棤 `vaExportSurfaceHandle` 澶辫触锛涙挱鏀?10 鍒嗛挓鏃犺姳灞?閿欏簭 |
| **L3 娴忚鍣ㄥ彲鐢?* | Firefox 璧?VA-API 鎾斁缃戦〉瑙嗛 | B 绔?1080p锛圚.264锛夋挱鏀?10 鍒嗛挓锛宍about:support` 鐨?Graphics/Media 娈佃惤鏄剧ず WebRender 涓庣‖浠惰В鐮佹甯革紙鍏蜂綋鏂囨闅忕増鏈級锛沗pidstat` 閲囨牱 | Firefox 鍏ㄨ繘绋?CPU 鍚堣 **鈮?60%**锛堝熀绾?140%~170%锛夛紱`about:media`/`MOZ_LOG` 鏄剧ず `VA-API` 瑙ｇ爜鍣紱鏃?AV 涓嶅悓姝ワ紱鎷栧姩杩涘害鏉?10 娆′笉宕?|
| **L4 鐢熶骇鍙敤** | 澶氱紪鐮?鍒嗚鲸鐜?seek/闀跨ǔ | 绗?9 绔犳祴璇曠煩闃靛叏缁?| 2 灏忔椂寰幆鎾斁 RSS 澧為暱 < 5%锛?K30 H.264 鍙В锛汬EVC 1080p 鍙В锛沄P9 1080p 鍙В锛?0bit 鏄庣‘杩斿洖涓嶆敮鎸佷笖涓嶅穿 |

### 1.3 閲忓寲鎸囨爣

| 鎸囨爣 | 鍩虹嚎锛堣蒋瑙ｏ紝銆愬疄娴嬨€戯級 | L3 鐩爣 | L4 鐩爣 | 娴嬮噺鏂规硶 |
|---|---|---|---|---|
| Firefox 鎬?CPU锛?080p30 H.264锛?0 s 骞冲潎锛?| 140%~170% | 鈮?60% | 鈮?40% | `pidstat -h -p ALL 1 60` 杩囨护 firefox 杩涚▼锛屾垨 搂9.5 鐨勬柟娉?|
| 鍗曞抚瑙ｇ爜寤惰繜锛堟彁浜ゅ埌 surface ready锛?| 鈥?| 鈮?30 ms锛圥95锛?| 鈮?20 ms | 椹卞姩鍐呮墦鐐?+ `test/latency.c` |
| 棣栧抚鏃堕棿锛堟墦寮€鍒伴甯т笂灞忥級 | 鈥?| 鈮?500 ms | 鈮?300 ms | `mpv --msg-level=all=v` 鏃堕棿鎴?|
| 涓㈠抚 | 鈥?| 0锛?080p30锛?| 0锛?080p60/4K30锛?| `mpv` 缁熻 / `about:media` |
| 鍐呭瓨澧為暱锛? h锛?| 鈥?| < 10% | < 5% | `/proc/<pid>/smaps_rollup` 鐨?`Pss` |
| 鏁存満鍔熻€楋紙1080p30锛?| 寰呮祴 | 姣旇蒋瑙ｄ綆 鈮?15% | 浣?鈮?25% | `/sys/class/power_supply/battery/{current_now,voltage_now}` 涔樼Н |
| 瑙ｇ爜姝ｇ‘鎬?| 鍩哄噯 | PSNR 鈮?50 dB | PSNR = inf | 搂9.4 |

> 娉ㄦ剰锛欳PU 鐩爣涓嶈鎷嶈剳琚嬪畾姝汇€?080p30 H.264 鍦?A510 涓婅蒋瑙ｇ害 1.5 鏍革紱纭В鍚?Firefox 浠嶉渶鍋氶鑹茶浆鎹?鍚堟垚锛圵ebRender 鐢?GPU锛夛紝**30%~50% 鏄悎鐞嗗尯闂?*锛?鈮?60%"鏄繚瀹堝彲浜や粯绾匡紝"鈮?40%"鏄紭鍖栫洰鏍囥€?
### 1.4 楠屾敹鑴氭湰锛堟斁 `test/accept.sh`锛?
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

# L1 姝ｇ‘鎬э紙閫愬抚 PSNR锛?./test/psnr_vs_sw.sh "$1"

# L2
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi -i "$1" -f null -
```

### 1.5 闈炵洰鏍囷紙鏄庣‘涓嶅仛锛岄伩鍏嶈寖鍥磋敁寤讹級

- **涓嶅仛缂栫爜**锛坄VAEntrypointEncSlice`锛夆€斺€攊ris 鏄函瑙ｇ爜鍣紝鏈」鐩彧鍋?`VAEntrypointVLD`銆?- **涓嶅仛 10bit**锛圚EVC Main10 / VP9 Profile2 / H.264 High10锛夆€斺€攃apture 渚у彧鏈?NV12锛岃 搂5.2銆?- **涓嶅仛 AV1/VP8/MPEG-2/VC-1**鈥斺€擿/dev/video0` 鏈灇涓捐繖浜涙牸寮忋€?- **涓嶅仛 DRM锛圵idevine/Widevine L3锛夊彈淇濇姢鍐呭**鈥斺€擵A-API 鐨?protected session 璺緞涓嶅湪鑼冨洿鍐呫€?- **涓嶆敼鍐呮牳 iris 椹卞姩**鈥斺€旈櫎闈?M1 璇佹槑鍐呮牳鏈夐樆濉炴€?bug锛涙敼鍐呮牳瑕佸崟鐙瘎浼般€?- **涓嶅仛 HDR/鑹插僵绠＄悊**鈥斺€?bit BT.709 涔嬪鐨勯鑹茬┖闂村厛鎸?閫忎紶"澶勭悊銆?
---

## 2. 鏋舵瀯鏂规瀵规瘮涓庨€夊瀷

### 2.1 涓変釜纭害鏉燂紙鍐冲畾浜嗘柟妗堢┖闂达級

**绾︽潫 1锛歩ris 鏄?stateful锛屼笉鏄?stateless銆?*

stateful 瑙ｇ爜鍣ㄧ殑鍐呮牳鎺ュ彛鏄?涓ゆ潯 vb2 闃熷垪 + 浜嬩欢"锛氱敤鎴锋€佹妸**瀹屾暣鐮佹祦**鍐欒繘 OUTPUT 闃熷垪锛屽唴鏍革紙HFI 鍥轰欢锛夎嚜宸辫В鏋?SPS/PPS/slice header锛岃В鍑烘潵鐨勫抚浠?CAPTURE 闃熷垪鎸夋樉绀哄簭杩斿洖銆傜敤鎴锋€佷笉鍙備笌璇硶瑙ｆ瀽锛屼篃鎷夸笉鍒?per-slice 鐨勬帶鍒舵帴鍙ｃ€?
stateless锛圧equest API锛夊垯鐩稿弽锛氱敤鎴锋€佽В鏋愮爜娴侊紝鎶?SPS/PPS/瑙ｇ爜鍙傛暟/slice 鍙傛暟浣滀负 `V4L2_CID_STATELESS_*` control 缁戝埌 `media_request` 涓婏紝鍐呮牳鍙仛"鎸夊弬鏁伴┍鍔ㄧ‖浠?銆?
涓よ€?*涓嶆槸鍚屼竴涓紪绋嬫ā鍨?*锛岃繖鏄€夊瀷鐨勭涓€鍐冲畾鍥犵礌銆?
**绾︽潫 2锛歏A-API 鏄?宸茶В鏋愬弬鏁?API锛屼笉鎻愪緵鍘熷鐮佹祦銆?*

VA-API 鐨?H.264 瑙ｇ爜鎺ュ彛缁欓┍鍔ㄧ殑鏄細

- `VAPictureParameterBufferH264`锛圫PS/PPS 鐨勮В鏋愮粨鏋滐細profile銆佸垎杈ㄧ巼銆乣log2_max_frame_num`銆丳OC 绫诲瀷銆佸弬鑰冨抚鏁般€佺喌缂栫爜妯″紡銆丵P銆佸幓鍧楀弬鏁扳€︹€︼級锛?- `VASliceParameterBufferH264`锛堟瘡涓?slice 鐨勮В鏋愮粨鏋滐級锛?- `VAQMatrixBufferH264`锛堢缉鏀剧煩闃碉級锛?- `VASliceDataBufferType`锛?*鍙湁 slice NAL 鐨勫瓧鑺?*锛屼笉鍚?SPS/PPS锛岄€氬父涔熶笉鍚捣濮嬬爜锛夈€?
涔熷氨鏄锛?*VA-API 瀹㈡埛绔紙ffmpeg锛変笉浼氭妸 SPS/PPS 鐨勫師濮嬪瓧鑺備氦缁欓┍鍔?*锛屽畠鏈熸湜椹卞姩鑷繁鐢ㄥ弬鏁?鍠傜‖浠?銆侷ntel 鐨?VA 椹卞姩姝ｆ槸杩欎箞鍋氱殑锛堟妸 SPS/PPS 鍐欒繘纭欢瀵勫瓨鍣級銆傝€?stateful V4L2 闇€瑕佷竴浠?*璇硶瀹屾暣鐨?Annex B 鐮佹祦**銆?
鈫?杩欐槸鏈」鐩渶澶х殑鎶€鏈殰纰嶏細stateful 鍚庣蹇呴』**浠?VA 鍙傛暟鍙嶅悜閲嶅缓 SPS/PPS锛圚EVC 杩樿 VPS/RPS锛?*锛屾垨鑰呰瀹㈡埛绔妸鍘熷鐮佹祦涔熼€佽繘鏉ャ€傝瑙?搂2.6 涓?搂3.6銆?
**绾︽潫 3锛欶irefox 鐨勮姹傛瘮 `vainfo` 楂樺緱澶氥€?*

Firefox 涓嶅彧鏄?鑳借В鐮?锛屽畠瑕佹眰锛?
1. `vaInitialize` 鍦?**render node**锛坄/dev/dri/renderD128`锛変笂鎴愬姛锛?2. 鏈?`vaExportSurfaceHandle`锛屼笖鑳界敤 `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` 瀵煎嚭 **NV12 dma-buf**锛?3. 瀵煎嚭鐨?dma-buf 鑳借 Mesa freedreno 鐨?`EGL_EXT_image_dma_buf_import` 瀵煎叆锛?4. 涓婅堪鍔ㄤ綔鍙戠敓鍦?Firefox 鐨?*鍐呭杩涚▼/RDD 杩涚▼**閲岋紝鍙?seccomp/AppArmor 闄愬埗锛堣 搂7.4锛夛紱
5. snap 鐗?Firefox 杩樺涓€灞傛寕杞藉懡鍚嶇┖闂撮殧绂伙紙瑙?搂7.5锛夈€?
### 2.2 鍊欓€夋柟妗?
| 鏂规 | 鍋氭硶 | 澶嶇敤 | 涓昏宸ヤ綔閲?| 椋庨櫓 | 缁撹 |
|---|---|---|---|---|---|
| **A. 鏂板啓 `libva-iris`** | 鐩存帴瀹炵幇 libva 椹卞姩 ABI锛坄va_backend.h`锛夛紝鍚庣鏄?V4L2 M2M | 鍙傝€?B 鐨勫鍑?绠＄悊浠ｇ爜 | 椹卞姩楠ㄦ灦 + 鐮佹祦閲嶅缓 + 闃熷垪鐘舵€佹満 | 鐮佹祦閲嶅缓姝ｇ‘鎬?| **鎺ㄨ崘** |
| **B. fork `libva-v4l2-request` 鍔?stateful 鍚庣** | 鍦?request 椹卞姩閲屽姞涓€鏉?M2M 璺緞 | 绾?30%锛坰urface/buffer/export/config锛?| 绛変簬 A锛岃繕澶氳儗涓€濂?request 鎶借薄 | 鎶借薄涓嶅尮閰嶏紝浠ｇ爜瓒婃敼瓒婃嫥 | 涓嶆帹鑽愶紙瑙?搂2.3锛?|
| **C. VA-API 鈫?GStreamer/ffmpeg 妗ユ帴** | VA 椹卞姩鍐呴儴璧?`gst-launch`/ffmpeg 瀛愯繘绋嬶紝鎶婂抚浼犲洖 | 楂?| 杩涚▼/绾跨▼閫氫俊銆佸抚鎷疯礉銆佸悓姝?| 寤惰繜銆侀浂鎷疯礉銆佺敓鍛藉懆鏈熴€丗irefox 瑕佹眰 | 涓嶆帹鑽愶紙瑙?搂2.5锛?|
| **D. 瀹㈡埛绔洿閫氳ˉ涓?+ 椹卞姩鐩撮€氭ā寮?* | 缁?ffmpeg 鎵撹ˉ涓侊紝鎶婂師濮?packet 鐢ㄧ鏈夋柟寮忛€佺粰椹卞姩锛涢┍鍔ㄨ烦杩囩爜娴侀噸寤?| 椹卞姩渚х渷鎺夐噸寤?| 鎵撹ˉ涓?+ 閲嶇紪 ffmpeg锛團irefox 瑕侀噸缂栵級 | 涓?Firefox 瀹樻柟鍖呬笉鍏煎 | **骞惰鍋?*锛氫綔涓鸿皟璇曢€氶亾涓庡厹搴曪紙搂2.6 R2锛?|
| **E. 绛?鐢ㄧ幇鎴愰」鐩?* | 鐢?`radxa-pkg/libva-v4l2` 鎴?Debian `qcom-va-driver` | 鍙兘 100% | 閫傞厤 iris | 椤圭洰鎴愮啛搴︽湭鐭?| **M0 鍏堝璁?*锛埪?.4锛?|

### 2.3 涓轰粈涔堜笉鑳界洿鎺ュ鐢?`libva-v4l2-request`

杩欎竴鑺傛槸缁?瑕佷笉瑕?fork"鐨勫喅绛栦緷鎹紝閫愭潯璁鸿瘉銆?
**(1) 鍐呮牳鎺ュ彛涓嶅悓锛屼笖鏈満鏍规湰娌℃湁 request API銆?*

`libva-v4l2-request` 鐨勬牳蹇冭皟鐢ㄥ簭鍒楁槸锛?
```
open("/dev/media0") 鈫?MEDIA_IOC_REQUEST_ALLOC 鈫?寰楀埌 request fd
瀵规瘡涓?slice锛歏IDIOC_S_EXT_CTRLS(V4L2_CID_STATELESS_H264_SPS / _PPS / _DECODE_PARAMS / _SLICE_PARAMS)
VIDIOC_QBUF(..., V4L2_BUF_FLAG_REQUEST_FD, request_fd)
MEDIA_REQUEST_IOC_QUEUE
poll(request_fd) / VIDIOC_DQBUF(CAPTURE)
```

iris 涓婏細

- 娌℃湁 `/dev/media*` 鑺傜偣锛堝畠璧?`v4l2_m2m`锛屼笉鏄?media controller锛夛紱
- `MEDIA_IOC_REQUEST_ALLOC` 浼氳繑鍥?`ENOTTY`锛?- `V4L2_CID_STATELESS_H264_*` 杩欑粍 control 鍦?stateful 椹卞姩涓婁笉瀛樺湪锛宍VIDIOC_S_CTRL` 浼氳繑鍥?`EINVAL`锛?- `VIDIOC_QBUF` 甯?`V4L2_BUF_FLAG_REQUEST_FD` 浼氳繑鍥?`EINVAL`銆?
缁撹锛歳equest 椹卞姩鐨?*鎻愪氦璺緞 0% 鍙敤**銆?
**(2) 鏁版嵁妯″瀷鐩稿弽锛氫竴涓"鍙傛暟"锛屼竴涓"鐮佹祦"銆?*

request 椹卞姩鎷垮埌鐨勬槸"SPS 缁撴瀯浣?+ PPS 缁撴瀯浣?+ slice 鍙傛暟 + slice 鏁版嵁"锛屾濂芥槸 VA-API 鎻愪緵鐨勪笢瑗匡紝鎵€浠ユ槧灏勫嚑涔?1:1銆?stateful 椹卞姩瑕佺殑鏄?涓€浠借兘鑷繁瑙ｆ瀽鐨?Annex B/HEVC/VP9 鐮佹祦"锛孷A-API 鎭版伆**涓嶇粰**杩欎釜銆傛柟鍚戠浉鍙嶏紝涓嶆槸"鏀逛竴鏀?鑳借В鍐崇殑銆?
**(3) 鐢熷懡鍛ㄦ湡妯″瀷涓嶅悓銆?*

request锛氫竴涓?request 瀵瑰簲涓€甯э紝鎻愪氦鍚庡紓姝ュ畬鎴愶紝`poll(request_fd)` 鎷跨粨鏋滐紱
M2M锛歄UTPUT/CAPTURE 涓ゆ潯闃熷垪闀挎湡瀛樺湪锛宍QBUF`/`DQBUF` 涓?`poll` 浜ら敊锛岃繕鏈?`SOURCE_CHANGE` 浜嬩欢銆乣V4L2_DEC_CMD_STOP` 鎺掔┖銆乣V4L2_BUF_FLAG_LAST` 鏀跺熬銆傜姸鎬佹満瑕侀噸鍐欍€?
**(4) 鍒嗚鲸鐜囧彉鍖栫殑璐ｄ换鏂逛笉鍚屻€?*

stateless锛氬鎴风瑙ｆ瀽 SPS 鍚庤嚜宸卞喅瀹氭柊鏍煎紡锛岄噸寤?CAPTURE 闃熷垪锛堝鎴风涓诲锛夈€?stateful锛氬唴鏍稿彂 `V4L2_EVENT_SOURCE_CHANGE`锛岀敤鎴锋€佸繀椤绘寜鍥哄畾搴忓垪澶勭悊锛坄G_FMT` 鈫?`S_FMT` 鈫?`STREAMOFF` 鈫?`REQBUFS(0)` 鈫?`REQBUFS` 鈫?`STREAMON`锛夛紝鑰屼笖 VA surface 鏄浐瀹氬昂瀵哥殑锛屽啿绐佽椹卞姩鑷繁鍏滐紙搂6.5锛夈€?
**(5) 浠ｇ爜澶嶇敤璇勪及锛堣瘹瀹炵増锛夈€?*

| 妯″潡 | 鑳藉惁澶嶇敤 | 璇存槑 |
|---|---|---|
| `vaExportSurfaceHandle` / DRM_PRIME_2 瀵煎嚭 | **鑳?*锛堝彲鍊熼壌锛?| 缁撴瀯浣撳～鍏呫€乫d dup銆乴ayer 甯冨眬鏄€氱敤鐨?|
| surface/buffer 瀵硅薄绠＄悊 | 閮ㄥ垎 | request 椹卞姩鐨?buffer 甯?request 鍏宠仈锛岃鎷嗘帀 |
| config/profile 琛?| 鑳?| 浣?profile 闆嗗悎瑕佹寜 iris 閲嶅啓 |
| meson 鏋勫缓銆佸畨瑁呰矾寰勩€乣vainfo` 鑱旇皟 | 鑳?| 鐩存帴鎶?|
| **瑙ｇ爜鎻愪氦/瀹屾垚璺緞** | **涓嶈兘** | 鍏ㄩ儴閲嶅啓 |
| 鐮佹祦閲嶅缓 | 涓嶅瓨鍦?| request 椹卞姩涓嶉渶瑕侊紝鏈」鐩鏂板啓 |

鍑€澶嶇敤绾?30%锛屼笖鏄渶涓嶅洶闅剧殑閭?30%锛沠ork 杩樹細甯︽潵"request 鎶借薄娈嬬暀"鐨勭悊瑙ｆ垚鏈€?
**缁撹锛氫笉 fork銆備互 `libva-v4l2-request` 涓轰唬鐮侀鏍煎弬鑰冿紝鏂板啓 `libva-iris`锛屽唴閮ㄦ寜"V4L2 M2M 鍚庣 + 鐮佹祦閲嶅缓鍣?鍒嗗眰銆?*

### 2.4 鍏堜緥瀹¤锛圡0 鍐崇瓥闂級

寮€宸ュ墠 2~3 澶╁彧鍋氫竴浠朵簨锛?*纭鏈夋病鏈夌幇鎴愯疆瀛?*銆?
1. `git clone https://github.com/mxsrc/libva-v4l2`銆乣https://github.com/radxa-pkg/libva-v4l2`锛岃 `README`銆乣meson.build`銆乣src/`锛?   - 瀹冩敮鎸佸摢浜?codec锛烪.264 鐨?SPS/PPS 鏄噸寤鸿繕鏄洿閫氾紵
   - 瀹冩€庝箞澶勭悊 `V4L2_EVENT_SOURCE_CHANGE`銆佹椂闂存埑鏄犲皠銆乣vaExportSurfaceHandle`锛?   - 瀹冩湁娌℃湁"瀹㈡埛绔洿閫?妯″紡锛?2. 鏌?Debian WNPP `#1152234`锛坄qcom-va-driver`锛夛細婧愮爜浠撳簱銆佺淮鎶よ€呫€佹槸鍚﹀凡杩?experimental/mentors銆佹敮鎸佸摢浜?SoC銆佹槸鍚﹀凡鏀寔 iris 鐨?HFI 鍥轰欢琛屼负銆?3. 鍦ㄨ澶囦笂 `apt-cache search va-driver | grep -i qcom`銆乣apt-cache search libva | grep -i v4l2` 澶嶆牳浠撳簱銆?
**鍐崇瓥瑙勫垯锛?*

- 鑻?`radxa libva-v4l2` 宸叉敮鎸?H.264 涓斾唬鐮佽川閲忓彲鎺ュ彈 鈫?鏂规鏀逛负"**fork + 鍔?iris 閫傞厤**"锛屽伐鏈熺爫鍗婏紙瑙?搂10 鐨勪箰瑙傚垪锛夈€?- 鑻?Debian `qcom-va-driver` 宸叉槸鍙敤椹卞姩 鈫?鐩存帴鎵撳寘/绉绘锛屾湰椤圭洰閫€鍖栦负"闆嗘垚 + 楠岃瘉 + 淇?bug"銆?- 鑻ヤ袱鑰呴兘涓嶅彲鐢?鈫?鎸夋柟妗?A 鎵ц锛屾湰鏂囨。鍏朵綑閮ㄥ垎鍗充负瀹炴柦鏂规銆?
### 2.5 涓轰粈涔堜笉鍋氭柟妗?C锛圴A 鈫?GStreamer/ffmpeg 妗ユ帴锛?
鐪嬭捣鏉?璧蜂釜 `gst-launch` 鎴?ffmpeg 瀛愯繘绋嬭В鐮侊紝鍐嶆妸甯ф惉鍥炴潵"鏈€鐪佷簨锛屼絾锛?
1. **闆舵嫹璐濅笉鍙兘**锛氬抚蹇呴』璺ㄨ繘绋?璺?API 杈圭晫鎼竴娆★紙鑷冲皯涓€娆?memcpy锛?080p30 NV12 绾?93 MB/s锛?K30 绾?373 MB/s锛夛紝CPU 鐪佷笅鏉ョ殑鍙堣鎷疯礉鍚冨洖鍘汇€?2. **寤惰繜涓庢姈鍔?*锛歏A-API 鏄悓姝?鍗婂悓姝ヨ皟鐢ㄦā鍨嬶紝`vaSyncSurface` 鏈熸湜姣绾э紱妗ユ帴瑕佸涓€娆?IPC 寰€杩?+ 闃熷垪锛宻eek 鏃舵竻绌虹閬撳緢闅惧仛骞插噣銆?3. **Firefox 瑕佺殑鏄?VA display 涓婄殑 surface 鑳藉鍑?dma-buf"**锛歚vaExportSurfaceHandle` 蹇呴』杩斿洖涓€涓?*灞炰簬鏈繘绋?VA display** 鐨?dma-buf銆傛ˉ鎺ヨ繘绋嬭В鍑烘潵鐨勫抚瑕佷箞鍐嶅鍏ュ洖鏉ワ紙澶氫竴娆?dma-buf 瀵煎叆 + 鍚屾锛夛紝瑕佷箞鐢ㄥ叡浜唴瀛橈紙涓嶅啀鏄?dma-buf锛孍GL 瀵煎叆澶辫触锛夈€?4. **鐢熷懡鍛ㄦ湡/閿欒浼犳挱**锛氬瓙杩涚▼宕╂簝銆佸崱姝汇€佺増鏈笉鍖归厤锛岄兘浼氬彉鎴?Firefox 鐨?瑙嗛鍗′綇"锛屽緢闅惧畾浣嶃€?5. **缁存姢鎴愭湰**锛氱瓑浜庡悓鏃剁淮鎶?VA 椹卞姩澶栧３ + 涓€涓挱鏀惧櫒"锛屼笖涓ゅご鍗囩骇閮戒細鐮村潖鍏煎銆?
缁撹锛氭ˉ鎺ュ彧閫傚悎鍋?*涓€娆℃€ч獙璇佸伐鍏?*锛堜緥濡傜敤 GStreamer 楠岃瘉 V4L2 閫氳矾锛夛紝涓嶈兘浣滀负浜や粯鏋舵瀯銆?
### 2.6 鍏抽敭鎶€鏈殰纰嶄笌涓夌瀵圭瓥锛堟湰椤圭洰鏈€闇€瑕佹媿鏉跨殑鍦版柟锛?
**闅滅锛歏A-API 涓嶆彁渚?SPS/PPS 鍘熷瀛楄妭锛岃€?stateful V4L2 闇€瑕佸畬鏁寸爜娴併€?*

涓夌瀵圭瓥锛?
| 瀵圭瓥 | 鍋氭硶 | 浼樼偣 | 缂虹偣 | 閫傜敤 |
|---|---|---|---|---|
| **R1 椹卞姩鍐呴噸寤?* | 鐢?`VAPictureParameterBufferH264` + `VAQMatrixBufferH264` 閲嶅缓 SPS/PPS锛宻lice NAL 鍘熸牱閫忎紶锛堣ˉ璧峰鐮侊級 | 瀵瑰鎴风闆舵敼鍔紝Firefox/ffmpeg/GStreamer 閫氬悆 | 宸ヤ綔閲忓ぇ锛汬EVC 鐨?RPS 鍙兘淇℃伅涓嶈冻锛涜竟鐣屾儏鍐靛 | **H.264 涓昏矾绾?*锛沄P9 鍙锛汬EVC 寰呰瘎浼?|
| **R2 瀹㈡埛绔洿閫?* | 缁?ffmpeg 鎵撹ˉ涓侊紝鎶?`start_frame` 鎷垮埌鐨勬暣鍖咃紙鍚?SPS/PPS锛変綔涓轰竴涓壒娈?buffer 閫佺粰椹卞姩锛涢┍鍔ㄧ洿鎺ヨ浆鍙?| 椹卞姩渚ф瀬绠€锛岀爜娴?100% 鍘熸牱锛孒EVC/VP9 閮借兘杩?| 瑕侀噸缂?ffmpeg锛汧irefox 鐢ㄧ殑鏄嚜甯?ffmpeg锛屽緱閲嶇紪 Firefox锛涘畼鏂?snap/deb 涓嶈 | **璋冭瘯閫氶亾 + 鍏滃簳**锛堝厛鍋氾紝鑳芥瀬澶у姞閫?bring-up锛?|
| **R3 娣峰悎** | H.264 鐢?R1锛汬EVC/VP9 鍏堝彧鏀寔 R2锛涙垨 HEVC 鍙敮鎸?姣忎釜 IDR 鍓嶅甫 SPS/PPS 鐨勬祦" | 骞宠　 | 浣撻獙涓嶄竴鑷?| 鏃堕棿绱ф椂鐨勭幇瀹為€夋嫨 |

**寤鸿锛歊1 涓?R2 骞惰銆?* 绗竴鍛ㄧ敤 R2 鎵撻€?VA 椹卞姩 鈫?V4L2 鈫?dma-buf 鈫?鍑虹敾闈?鐨勫叏閾捐矾锛堜笉纰扮爜娴侀噸寤猴級锛岀浜屽懆璧风敤 R1 鏇挎崲锛屾渶缁堜氦浠?R1锛汻2 淇濈暀涓?`--enable-passthrough` 缂栬瘧閫夐」锛岀敤浜?HEVC 绱ф€ュ厹搴曞拰鍥炲綊瀵规瘮銆?
R2 鐨勫叿浣撳仛娉曪紙缁?ffmpeg 鐨勬渶灏忚ˉ涓侊級锛?
```c
/* 浼唬鐮侊紙libavcodec/vaapi_h264.c: vaapi_h264_start_frame 寮€澶达級
 * buffer/size 鏄?h264dec 浼犺繘鏉ョ殑鏁翠釜 packet锛堝惈 SPS/PPS/SEI/slices锛夈€? * 杩欓噷鐩存帴鎷?VAAPIDecodeContext 閲岀殑 display/context 寤轰竴涓甫榄旀暟鐨?buffer銆?*/
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

椹卞姩渚э細鑻?`vaRenderPicture` 鏀跺埌鐨勭涓€涓?buffer 浠ラ瓟鏁?`"IRISRAW1"` 寮€澶达紙琛ヤ竵閲屽姞 8 瀛楄妭澶达級锛屽垯鎶婂畠褰撲綔**鏁翠釜 AU 鐨勫畬鏁寸爜娴?*锛岀洿鎺?`QBUF`锛屽拷鐣ュ悗缁殑 pic/slice 鍙傛暟 buffer锛涘惁鍒欒蛋 R1 閲嶅缓璺緞銆傝繖鏍峰悓涓€涓┍鍔ㄤ簩杩涘埗鍚屾椂鏀寔涓ょ瀹㈡埛绔紝涓斾笉褰卞搷鏈墦琛ヤ竵鐨勫鎴风銆?
> 娉ㄦ剰锛歚buffer/size` 鍦?`start_frame` 閲屾槸鍚︽槸鏁村寘锛岄渶瑕佸湪 ffmpeg 8.0.1 婧愮爜閲岀‘璁わ紙`libavcodec/h264dec.c` 璋冪敤 `h->hwaccel->start_frame(avctx, ...)` 鐨勫疄鍙傦級銆傝嫢涓嶆槸鏁村寘锛岄€€鍖栦负鍦?`decode_slice` 閲岀疮绉苟鍦?`end_frame` 鏃惰ˉ鍙?SPS/PPS鈥斺€斾絾閭ｆ牱鎷夸笉鍒?SPS/PPS 瀛楄妭锛孯2 灏变笉鎴愮珛銆?*杩欐槸 M0 蹇呴』楠岃瘉鐨勭涓€浠朵簨銆?*

### 2.7 鎺ㄨ崘鏋舵瀯涓庢ā鍧楀垝鍒?
```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?Firefox / ffmpeg / mpv / GStreamer锛圴A-API 瀹㈡埛绔級          鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?                            鈹?vaInitialize / vaCreateConfig / ...
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈻尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?libva锛?.23锛?鍔犺浇 iris_drv_video.so锛岃浆鍙戝埌 VADriverVTable  鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈻尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?libva-iris锛堟湰椤圭洰锛?                                        鈹?鈹? 鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?鈹? 鈹?VA 鍓嶇      鈹?鈹?鐮佹祦閲嶅缓鍣?   鈹?鈹?V4L2 M2M 鍚庣         鈹?鈹?鈹? 鈹?config/      鈹?鈹?h264_sps.c   鈹?鈹?open/S_FMT/REQBUFS    鈹?鈹?鈹? 鈹?context/     鈹?鈹?h264_pps.c   鈹?鈹?QBUF/DQBUF/poll       鈹?鈹?鈹? 鈹?surface/     鈹?鈹?h264_slice.c 鈹?鈹?SOURCE_CHANGE/EOS     鈹?鈹?鈹? 鈹?buffer       鈹?鈹?hevc_*/vp9_* 鈹?鈹?鏃堕棿鎴斥啍surface 鏄犲皠    鈹?鈹?鈹? 鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?鈹? 鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?            鈹?鈹? 鈹?export.c锛歷aExportSurfaceHandle / image.c 鈹?鈹?            鈹?鈹? 鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?            鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?                                                鈹?ioctl
                                    鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈻尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?                                    鈹?/dev/video0 iris_driver鈹?                                    鈹? (stateful M2M, NV12)  鈹?                                    鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

妯″潡涓庢枃浠讹紙寤鸿甯冨眬锛孧1 灏辨寜杩欎釜寤猴級锛?
```
libva-iris/
鈹溾攢鈹€ meson.build
鈹溾攢鈹€ meson_options.txt
鈹溾攢鈹€ README.md
鈹溾攢鈹€ src/
鈹?  鈹溾攢鈹€ driver.c        # VA_DRIVER_INIT_FUNC銆乂ADriverVTable 瑁呴厤銆佺増鏈?鍘傚晢
鈹?  鈹溾攢鈹€ display.c       # drm fd銆丵UERYCAP銆丒NUM_FMT銆佽兘鍔涚紦瀛?鈹?  鈹溾攢鈹€ config.c        # profile/entrypoint/attribute 琛?鈹?  鈹溾攢鈹€ context.c       # V4L2 鎵撳紑銆丼_FMT銆丷EQBUFS銆丼TREAMON銆丼OURCE_CHANGE
鈹?  鈹溾攢鈹€ surface.c       # surface 姹犮€乨ma-buf 缁戝畾銆佺姸鎬?鈹?  鈹溾攢鈹€ buffer.c        # VABufferID 绠＄悊銆乵ap/unmap
鈹?  鈹溾攢鈹€ decode.c        # Begin/Render/End/Sync/QueryStatus
鈹?  鈹溾攢鈹€ export.c        # vaExportSurfaceHandle锛圖RM_PRIME_2锛?鈹?  鈹溾攢鈹€ image.c         # vaDeriveImage/GetImage/PutImage/QueryImageFormats
鈹?  鈹溾攢鈹€ v4l2.c          # ioctl 灏佽銆乸oll 寰幆銆侀敊璇槧灏?鈹?  鈹溾攢鈹€ queue.c         # OUTPUT/CAPTURE 闃熷垪銆乼imestamp鈫攕urface 鏄犲皠
鈹?  鈹溾攢鈹€ bitstream/
鈹?  鈹?  鈹溾攢鈹€ bs_writer.c # RBSP 鍐欎綅銆丒PB 鎻掑叆銆乽e/se 缂栫爜
鈹?  鈹?  鈹溾攢鈹€ h264_sps.c
鈹?  鈹?  鈹溾攢鈹€ h264_pps.c
鈹?  鈹?  鈹溾攢鈹€ h264_au.c   # AUD/SPS/PPS/slice 缁勮
鈹?  鈹?  鈹溾攢鈹€ hevc_ps.c   # VPS/SPS/PPS锛堥樁娈?6锛?鈹?  鈹?  鈹斺攢鈹€ vp9_hdr.c   # VP9 uncompressed header锛堥樁娈?6锛?鈹?  鈹斺攢鈹€ profile.c
鈹溾攢鈹€ test/
鈹?  鈹溾攢鈹€ va-dump.c       # 鏈€灏?VA 瀹㈡埛绔細璇?.h264/.ivf锛岃В鐮侊紝瀵煎嚭 NV12
鈹?  鈹溾攢鈹€ test_bs.c       # 鐮佹祦閲嶅缓鍣ㄥ崟娴嬶細鍚堟垚 AU 鈫?ffmpeg 杞В 鈫?姣斿
鈹?  鈹溾攢鈹€ psnr_vs_sw.sh
鈹?  鈹斺攢鈹€ accept.sh
鈹斺攢鈹€ tools/
    鈹斺攢鈹€ v4l2-probe.c    # 鐩存帴璺?V4L2 鐨勬帰閽堬紙涓嶇粡杩?VA锛夛紝鐢ㄤ簬闅旂闂
```

**鍒嗗眰鍘熷垯锛歚bitstream/` 涓嶇浠讳綍 ioctl锛宍v4l2.c` 涓嶇浠讳綍 VA 缁撴瀯浣撱€?* 杩欐牱鐮佹祦閲嶅缓鍣ㄥ彲浠ョ敤绾敤鎴锋€佸崟娴嬭鐩栵紙搂9.2锛夛紝涓嶇敤姣忔閮戒笂纭欢銆?
---

## 3. VA-API 鈫?V4L2 鍏蜂綋鏄犲皠

鏈珷鏄?鐓х潃鍐?鐨勯儴鍒嗐€傛墍鏈?V4L2 缁撴瀯浣撲互 `linux/videodev2.h` 涓哄噯锛屾墍鏈?VA 缁撴瀯浣撲互 `/usr/include/va/va*.h`锛坙ibva 2.23锛変负鍑嗐€?
### 3.1 鎬昏琛?
| VA-API 鍏ュ彛锛坄VADriverVTable` 瀛楁锛?| 椹卞姩鍐呴儴鍔ㄤ綔 | 瀵瑰簲 V4L2 ioctl / API | 澶囨敞 |
|---|---|---|---|
| `vaInitialize` 鈫?`VA_DRIVER_INIT_FUNC` | 鎵撳紑/鎺ョ drm fd銆佹帰娴嬭澶囥€佽閰?vtable | `VIDIOC_QUERYCAP`銆乣VIDIOC_ENUM_FMT`(OUTPUT/CAPTURE)銆乣VIDIOC_QUERYCTRL` | 鎺㈡祴缁撴灉缂撳瓨锛屽悗缁煡璇笉鍐嶈繘鍐呮牳 |
| `vaTerminate` | 閲婃斁鎵€鏈夊璞?| `close()` | 骞傜瓑 |
| `vaQueryConfigProfiles` | 杩斿洖 profile 鏁扮粍 | 鏃狅紙鐢?init 缂撳瓨锛?| 鍙繑鍥?`VIDIOC_ENUM_FMT` 鏀寔涓?`TRY_FMT` 閫氳繃鐨?|
| `vaQueryConfigEntrypoints` | 杩斿洖 `{VAEntrypointVLD}` | 鏃?| 瑙ｇ爜鍣ㄥ彧鏈?VLD |
| `vaGetConfigAttributes` | 杩斿洖 `VAConfigAttribRTFormat` 绛?| 鏃?| 瑙?搂3.2 |
| `vaCreateConfig` | 鏍￠獙 profile/entrypoint/attrib锛屽缓 config 瀵硅薄 | 鏃?| 涓嶇 V4L2锛岀湡姝ｇ殑璁惧鍦?context 鎵撳紑 |
| `vaDestroyConfig` | 閲婃斁 | 鏃?| |
| `vaCreateContext` | **鎵撳紑 `/dev/video0`锛屽缓绔嬩袱鏉￠槦鍒?* | `open`銆乣VIDIOC_SUBSCRIBE_EVENT`銆乣S_FMT`脳2銆乣G_CTRL`銆乣REQBUFS`脳2銆乣QUERYBUF`銆乣mmap`銆乣EXPBUF`銆乣STREAMON`脳2 | 搂3.3 鏈夊畬鏁村簭鍒?|
| `vaDestroyContext` | 鍋滄祦銆侀噴鏀?buffer銆佸叧 fd | `STREAMOFF`脳2銆乣REQBUFS(0)`銆乣munmap`銆乣close` | |
| `vaCreateSurfaces` | 鍒嗛厤/瀵煎叆 surface锛堟嫹璐濇ā寮忥細鏂板垎閰?dma-buf锛涘埆鍚嶆ā寮忥細鍙?CAPTURE 姹狅級 | 鏃狅紙鍒悕妯″紡鐨勬睜鍦?context 寤哄ソ锛?| 瑙?搂4.3 |
| `vaDestroySurfaces` | 褰掕繕姹?| 鏃?| |
| `vaCreateBuffer` / `vaMapBuffer` / `vaUnmapBuffer` / `vaDestroyBuffer` | 鐢ㄦ埛鎬佸唴瀛樼鐞?| 鏃?| `vaMapBuffer` 杩斿洖 CPU 鎸囬拡 |
| `vaBeginPicture` | 缁戝畾 surface銆佹竻绌?AU 缂撳啿銆佺疆 DECODING | 鏃?| |
| `vaRenderPicture` | 鎸?buffer type 鍒嗘淳锛泂lice data 杩藉姞杩?AU | 鏃?| 搂3.4 |
| `vaEndPicture` | 缁勮 AU 鈫?鍙栫┖闂?OUTPUT buffer 鈫?鎷疯礉 鈫?鍏ラ槦 | `VIDIOC_QBUF`(OUTPUT) | `bytesused`=AU 闀垮害锛宍timestamp`=frame_seq |
| `vaSyncSurface` | 绛夊埌璇?surface 瀵瑰簲鐨?CAPTURE buffer 鍑洪槦 | `poll()`銆乣VIDIOC_DQBUF`(CAPTURE)銆乣VIDIOC_DQEVENT` | 瑙?搂6 |
| `vaQuerySurfaceStatus` | 杩斿洖鍐呴儴鐘舵€?| 鏃?| |
| `vaQuerySurfaceError` | 杩斿洖瑙ｇ爜閿欒鐮?| 鏃?| |
| `vaExportSurfaceHandle` | `dup()` surface 鐨?dma-buf锛屽～ `VADRMPRIMESurfaceDescriptor` | 鏃狅紙fd 鍦?surface 鍒涘缓鏃跺凡鏈夛級 | 搂4.4 |
| `vaDeriveImage` / `vaGetImage` / `vaPutImage` | 鎶?dma-buf 鏄犲皠鎴?`VAImage` | `mmap`/`DMA_BUF_IOCTL_SYNC` | 搂3.5 |
| `vaQueryImageFormats` | 杩斿洖 NV12 | 鏃?| |
| `vaPutSurface` | 涓嶅疄鐜?| 鏃?| 杩斿洖 `VA_STATUS_ERROR_UNIMPLEMENTED`锛團irefox/mpv 涓嶈蛋瀹冿級 |
| `vaQuerySurfaceAttributes` | 杩斿洖鍍忕礌鏍煎紡/灏哄鑼冨洿 | 鏃?| ffmpeg 鍒濆鍖栨椂浼氭煡 |
| 锛堟棤鐩存帴瀵瑰簲锛?| seek/flush | `STREAMOFF`脳2 鈫?`REQBUFS(0)` 鈫?閲嶆柊 `S_FMT`/`REQBUFS`/`STREAMON` | 搂6.6 |
| 锛堟棤鐩存帴瀵瑰簲锛?| 鎺掔┖/EOS | `VIDIOC_DECODER_CMD(V4L2_DEC_CMD_STOP)` 鈫?绛?`V4L2_BUF_FLAG_LAST` | 搂6.4 |

### 3.2 鍒濆鍖栦笌鑳藉姏鏌ヨ

**`vaInitialize`锛堥┍鍔ㄤ晶 `__vaDriverInit_1_0`锛?*

libva 浼氬厛鎵撳紑 DRM 璁惧骞舵妸 fd 鏀捐繘 `VADriverContext`锛岀劧鍚?`dlsym` 椹卞姩 so 閲岀殑 `VA_DRIVER_INIT_FUNC`銆傞┍鍔ㄥ繀椤伙細

```c
VA_EXPORT VAStatus VA_DRIVER_INIT_FUNC(VADriverContextP ctx)
{
    struct iris_driver *drv = calloc(1, sizeof(*drv));
    ctx->pDriverData = drv;
    ctx->version     = VA_MAJOR_VERSION * 1000 + VA_MINOR_VERSION; /* 涓?intel-media-driver 鍐欐硶涓€鑷?*/
    ctx->str_vendor  = "qcom-iris v4l2-m2m";
    ctx->max_profiles = 8;
    ctx->max_entrypoints = 2;
    ctx->max_attributes = 8;
    ctx->max_image_formats = 4;
    ctx->max_subpic_formats = 0;
    ctx->max_display_attributes = 0;
    ctx->vtable = &iris_vtable;

    drv->drm_fd = ctx->drm_fd;      /* 涓嶈 close锛宭ibva 璐熻矗 */
    return iris_probe(drv);          /* QUERYCAP + ENUM_FMT锛屽け璐ヨ繑鍥?VA_STATUS_ERROR_OPERATION_FAILED */
}
```

瑕佺偣锛?
- 鐢?`nm -D iris_drv_video.so | grep vaDriverInit` 纭绗﹀彿瀵煎嚭锛坄VA_EXPORT` = `__attribute__((visibility("default")))`锛夈€?- `ctx->drm_fd` 鏄?libva 鎵撳紑鐨?render node锛坄VA_DISPLAY_DRM`锛夈€?*涓嶈閲嶅 `open` 鍚屼竴涓妭鐐?*锛屼篃涓嶈 `close` 瀹冦€?- 濡傛灉 `vaGetDisplay()` 璧扮殑鏄?Wayland/X11 display锛宍ctx->display_type` 浼氭槸 `VA_DISPLAY_WAYLAND`/`VA_DISPLAY_X11`锛涙湰椤圭洰鍙敮鎸?`VA_DISPLAY_DRM`锛屽叾瀹冪被鍨嬪湪 `vaInitialize` 閲岃繑鍥?`VA_STATUS_ERROR_INVALID_DISPLAY`锛團irefox/ffmpeg 閮界敤 DRM锛屽鐢級銆?
**`iris_probe()` 鍏蜂綋鍋氫粈涔?*

```c
struct v4l2_capability cap;
ioctl(fd, VIDIOC_QUERYCAP, &cap);
/* 蹇呴』婊¤冻 */
assert(cap.device_caps & V4L2_CAP_VIDEO_M2M_MPLANE);
assert(cap.device_caps & V4L2_CAP_STREAMING);
/* 璁板綍 driver/card/bus_info 鐢ㄤ簬鏃ュ織 */

/* OUTPUT锛氭灇涓惧帇缂╂牸寮?*/
for (i = 0; ; i++) {
    struct v4l2_fmtdesc d = { .index = i, .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE };
    if (ioctl(fd, VIDIOC_ENUM_FMT, &d) < 0) break;
    /* 鏈熸湜鐪嬪埌 V4L2_PIX_FMT_H264 / V4L2_PIX_FMT_HEVC / V4L2_PIX_FMT_VP9 */
    /* d.flags & V4L2_FMT_FLAG_DYN_RESOLUTION 璁颁笅鏉ワ紝鐢ㄤ簬鍐冲畾鏄惁璁㈤槄 SOURCE_CHANGE */
}

/* CAPTURE锛氭灇涓惧師濮嬫牸寮?*/
for (i = 0; ; i++) {
    struct v4l2_fmtdesc d = { .index = i, .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE };
    if (ioctl(fd, VIDIOC_ENUM_FMT, &d) < 0) break;
    /* 鏈熸湜鍙湁 V4L2_PIX_FMT_NV12锛涜嫢灏嗘潵鍑虹幇 P010/NV12_10 鍐嶆墿灞?*/
}

/* 姣忎釜鍊欓€?(codec, profile) 鐢?TRY_FMT 楠岃瘉涓€娆★紝閬垮厤"鏋氫妇鏀寔浣嗗疄闄呬笉鑳界敤" */
```

**涓轰粈涔堜笉姣忔鏌ヨ閮借繘鍐呮牳**锛歚vaQueryConfigProfiles` 鍙兘琚鎴风鍦ㄥ惎鍔ㄦ椂璋冪敤澶氭锛岃€?`VIDIOC_ENUM_FMT`/`TRY_FMT` 鍦ㄨВ鐮佸櫒宸?STREAMON 鏃惰涓哄彈闄愶紱缂撳瓨鍒?`drv->caps` 閲屾渶绋炽€?
**`vaGetConfigAttributes` 杩斿洖鍊?*

| attrib | 鍊?| 璇存槑 |
|---|---|---|
| `VAConfigAttribRTFormat` | `VA_RT_FORMAT_YUV420` | 鍙敮鎸?8bit 4:2:0锛?0bit 瑙?搂5.2 |
| `VAConfigAttribDecSliceMode` | `VA_DEC_SLICE_MODE_NORMAL` | 涓嶆敮鎸?`VA_DEC_SLICE_MODE_BASE`锛堥偅鏄?stateless 鐨?slice 杈撳嚭妯″紡锛?|
| `VAConfigAttribDecProcessing` | `VA_DEC_PROCESSING_NONE` | 涓嶅仛閫愬抚鍚庡鐞?|
| `VAConfigAttribMaxPictureWidth/Height` | 鐢?`VIDIOC_ENUM_FRAMESIZES` 鎴?`TRY_FMT` 鎺㈡祴锛堝 4096脳2304锛?| 鎺㈡祴澶辫触灏辩粰 1920脳1088 淇濆畧鍊?|

> 娉ㄦ剰锛歚VAConfigAttribDecSliceMode` 鐨勮涔夋槸"椹卞姩鏄惁鏀寔鎸?slice 鎻愪氦"锛屼笉鏄?鐮佹祦閲屾湁娌℃湁澶?slice"銆傛垜浠敮鎸佸 slice 鐮佹祦锛屼絾鎻愪氦鍗曚綅濮嬬粓鏄?涓€甯т竴涓?AU"锛屾墍浠ュ～ `NORMAL`銆?
### 3.3 `vaCreateContext`锛歏4L2 绠￠亾鐨勫畬鏁村缓绔嬪簭鍒?
杩欐槸鏁翠釜椹卞姩鏈€鍏抽敭鐨?100 琛屻€傞『搴忎笉鑳介敊锛?
```c
int fd = open("/dev/video0", O_RDWR | O_NONBLOCK | O_CLOEXEC);

/* 1. 璁㈤槄鍒嗚鲸鐜囧彉鍖栦簨浠讹紙蹇呴』鍦?STREAMON 涔嬪墠锛?*/
struct v4l2_event_subscription sub = { .type = V4L2_EVENT_SOURCE_CHANGE };
ioctl(fd, VIDIOC_SUBSCRIBE_EVENT, &sub);

/* 2. OUTPUT 鏍煎紡锛氬帇缂╃爜娴?*/
struct v4l2_format ofmt = { .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE };
ofmt.fmt.pix_mp.width       = ctx_width;     /* 鏉ヨ嚜 vaCreateContext 鐨?picture_width */
ofmt.fmt.pix_mp.height      = ctx_height;
ofmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_H264;  /* 鎸?profile 閫?H264/HEVC/VP9 */
ofmt.fmt.pix_mp.field       = V4L2_FIELD_NONE;
ofmt.fmt.pix_mp.num_planes  = 1;
ofmt.fmt.pix_mp.plane_fmt[0].sizeimage = 4 * 1024 * 1024;  /* 鍗曞抚 AU 涓婇檺锛岃涓?*/
ioctl(fd, VIDIOC_S_FMT, &ofmt);

/* 3. CAPTURE 鏍煎紡锛歂V12 */
struct v4l2_format cfmt = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE };
cfmt.fmt.pix_mp.width       = ctx_width;
cfmt.fmt.pix_mp.height      = ctx_height;
cfmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
cfmt.fmt.pix_mp.num_planes  = 1;              /* NV12 鏄崟 plane 鏍煎紡锛孻/UV 鍦ㄥ悓涓€ buffer */
ioctl(fd, VIDIOC_S_FMT, &cfmt);
/* S_FMT 涔嬪悗 cfmt 琚┍鍔ㄦ敼鍐欙細浠?cfmt.fmt.pix_mp.plane_fmt[0].bytesperline 涓哄噯 */

/* 4. 闃熷垪娣卞害 */
struct v4l2_control c;
c.id = V4L2_CID_MIN_BUFFERS_FOR_CAPTURE;
ioctl(fd, VIDIOC_G_CTRL, &c);           /* c.value 閫氬父鏄?4 宸﹀彸 */
int cap_count = MAX(c.value + 4, surface_count_hint);
cap_count = MIN(cap_count, 32);         /* vb2 涓婇檺锛屽叿浣撲互 REQBUFS 杩斿洖鍊间负鍑?*/

/* 5. 鐢宠 buffer */
struct v4l2_requestbuffers req = {
    .count = 4, .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE, .memory = V4L2_MEMORY_MMAP };
ioctl(fd, VIDIOC_REQBUFS, &req);
/* 瀵规瘡涓?index锛歈UERYBUF 鈫?mmap 鈫?璁板綍 length/offset */

req.count = cap_count; req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
req.memory = V4L2_MEMORY_MMAP;          /* 鎴?V4L2_MEMORY_DMABUF锛岃 搂4.1 */
ioctl(fd, VIDIOC_REQBUFS, &req);
/* 瀵规瘡涓?index锛歈UERYBUF 鈫?mmap 鈫?EXPBUF 鎷?dma-buf fd */

/* 6. 寮€娴?*/
enum v4l2_buf_type t = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE;  ioctl(fd, VIDIOC_STREAMON, &t);
t = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;                    ioctl(fd, VIDIOC_STREAMON, &t);
```

鍑犱釜瀹规槗韪╃殑鐐癸細

- **`sizeimage` 蹇呴』澶熷ぇ**銆備竴甯?1080p H.264 閫氬父 < 1 MB锛屼絾 4K 楂樼爜鐜?+ 澶?slice 鍙兘鍒板嚑 MB銆傚缓璁?4 MiB锛涜嫢 `QBUF` 杩斿洖 `EINVAL`/`ENOSPC`锛屽厛鎬€鐤戣繖涓€備篃鍙互鎸?`width*height*2` 鍔ㄦ€佺畻锛屼笅闄?2 MiB銆?- **CAPTURE 鐨?`bytesperline` 鐢遍┍鍔ㄥ喅瀹?*锛孮ualcomm 鍙兘鎸?128/256 瀵归綈銆?*涓嶈鍋囪瀹冪瓑浜?width**锛屽鍑?dma-buf 鏃跺繀椤荤敤椹卞姩杩斿洖鐨?`bytesperline` 鍜?`sizeimage`銆?- **`num_planes`**锛歚V4L2_PIX_FMT_NV12` 鍦?MPLANE 闃熷垪涓婁粛鐒舵槸 1 涓?plane锛圷 鍜?UV 杩炵画瀛樻斁锛夈€備笉瑕佹寜 2 涓?plane 鍘诲～ `plane_fmt[1]`銆?- **寤惰繜 STREAMON 涔熷彲浠?*锛氬鏋?surface 姹犲ぇ灏忚鍒扮涓€娆?`vaCreateSurfaces` 鎵嶇煡閬擄紝灏辨妸绗?5銆? 姝ユ帹杩熷埌绗竴娆?`vaEndPicture`銆備唬浠锋槸閿欒瀹氫綅鏅氫竴鐐癸紱寤鸿 MVP 鍏堟寜"`vaCreateContext` 鏃跺氨鍥哄畾姹犲ぇ灏?锛屾睜澶у皬鍙?`vaCreateContext` 鐨?hint 鎴?16锛孡2 鍐嶅仛鍔ㄦ€佹墿瀹广€?- **`VIDIOC_REQBUFS` 杩斿洖鐨?`count` 鍙兘灏忎簬璇锋眰鍊?*锛屼互杩斿洖鍊间负鍑嗭紱濡傛灉灏忎簬 `min_buffers`锛岀洿鎺?`VA_STATUS_ERROR_ALLOCATION_FAILED`銆?
**`vaCreateContext` 鐨?`picture_width/height` 鏄?coded size**銆俧fmpeg 8.0.x 浼犵殑鏄?`coded_width/coded_height`锛?080p 閫氬父鏄?1920脳1088锛夛紝涓嶆槸鏄剧ず灏哄 1920脳1080銆?*涓嶈鎷挎樉绀哄昂瀵稿幓 `S_FMT`**锛屽惁鍒?CAPTURE 浼氳椹卞姩鏀规垚 1088锛堟垨鎶ラ敊锛夛紝鑰?surface 鎸?1080 鍒嗛厤灏变細瓒婄晫銆?
### 3.4 瑙ｇ爜璺緞锛歚vaBeginPicture` 鈫?`vaRenderPicture` 鈫?`vaEndPicture` 鈫?`vaSyncSurface`

**`vaBeginPicture(dpy, ctx, surface)`**

```c
surf = surface_lookup(surface_id);
if (surf->state != SURFACE_FREE) return VA_STATUS_ERROR_SURFACE_BUSY;
surf->state = SURFACE_DECODING;
ctx->cur_surface = surf;
ctx->au_len = 0;                 /* 娓呯┖ AU 绱Н缂撳啿 */
ctx->have_pic_param = false;
ctx->slice_count = 0;
```

**`vaRenderPicture(dpy, ctx, buffers, num)`**

鎸?`VABufferID` 鍙栧嚭绫诲瀷锛屽垎娲撅細

| `VABufferType` | 澶勭悊 |
|---|---|
| `VAPictureParameterBufferType` | 淇濆瓨鎸囬拡/鎷疯礉鍒?`ctx->pic_param`锛圚.264 杩樿椤烘墜鍙?SPS/PPS 瀛楁锛?|
| `VASliceParameterBufferType` | 淇濆瓨 slice 鍏冩暟鎹紙MVP 鍙敤浜庢牎楠岋紝涓嶅弬涓庣粍鍖咃級 |
| `VASliceDataBufferType` | **杩藉姞鍒?AU 缂撳啿**锛堣ˉ璧峰鐮侊紝瑙?搂3.6锛?|
| `VAQMatrixBufferType` | 淇濆瓨 scaling list锛圚.264 PPS 闇€瑕侊級 |
| `VAProbabilityBufferType` / `VAIQMatrixBufferType` | 蹇界暐锛堥潪 H.264锛?|
| 鍏跺畠 | 杩斿洖 `VA_STATUS_ERROR_UNSUPPORTED_BUFFERTYPE` |

ffmpeg 鐨勮皟鐢ㄩ『搴忔槸锛歱icture param 鈫?qmatrix 鈫?slice param 鈫?slice data锛堝彲鑳藉娆★級锛屽叏閮ㄥ湪鍚屼竴涓?`vaBeginPicture`/`vaEndPicture` 涔嬮棿銆?*涓嶈鍋囪鍙湁涓€涓?slice data buffer**銆?
**`vaEndPicture(dpy, ctx)`**

```c
/* 1. 缁勫寘锛欰UD + SPS + PPS + slices锛圚.264锛夛紝瑙?搂3.6 */
au = build_access_unit(ctx, &au_len);
if (!au_len) { /* 绌哄抚锛堝 EOS flush 鐨勬渶鍚庝竴娆¤皟鐢級*/ return VA_STATUS_SUCCESS; }

/* 2. 鍙栦竴涓┖闂?OUTPUT buffer锛涙病鏈夊氨 poll 绛?*/
idx = output_get_free(ctx, /*block=*/true);

/* 3. 鎷疯礉 + 鍏ラ槦 */
memcpy(ctx->out_va[idx], au, au_len);
struct v4l2_buffer b = {
    .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE, .memory = V4L2_MEMORY_MMAP,
    .index = idx, .bytesused = au_len, .field = V4L2_FIELD_NONE,
};
b.timestamp.tv_sec  = ctx->frame_seq / 1000000;
b.timestamp.tv_usec = ctx->frame_seq % 1000000;   /* 鍏抽敭锛歠rame_seq 浣滀负鍏宠仈閿?*/
ioctl(ctx->fd, VIDIOC_QBUF, &b);

/* 4. 璁板綍 surface 涓?frame_seq 鐨勬槧灏?*/
queue_push_submitted(ctx, ctx->frame_seq, surf);
ctx->frame_seq++;
surf->state = SURFACE_DECODING;
```

`bytesused` 瀵?OUTPUT 闃熷垪鏄?*鏈夋晥鐮佹祦瀛楄妭鏁?*锛屼笉鏄?buffer 澶у皬銆傚骞抽潰鏃跺～ `plane_fmt[0].bytesused`锛坄v4l2_buffer.m.planes[0].bytesused`锛夈€?
**`vaSyncSurface(dpy, surface)`**

```c
surf = surface_lookup(surface_id);
while (!surf->decoded) {
    if (poll(ctx->fd, POLLIN, timeout_ms) <= 0) return VA_STATUS_ERROR_OPERATION_FAILED;
    drain_capture(ctx);          /* 鍐呴儴寰幆 DQBUF锛岀洿鍒?EAGAIN */
    drain_events(ctx);           /* DQEVENT锛歋OURCE_CHANGE 鍦ㄨ繖閲屽鐞?*/
}
return surf->error ? VA_STATUS_ERROR_DECODING_ERROR : VA_STATUS_SUCCESS;
```

`drain_capture()`锛?
```c
for (;;) {
    struct v4l2_buffer b = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
                             .memory = V4L2_MEMORY_MMAP };
    struct v4l2_plane planes[1] = {0};
    b.m.planes = planes; b.length = 1;
    if (ioctl(fd, VIDIOC_DQBUF, &b) < 0) {
        if (errno == EAGAIN) break;
        /* EIO 绛夛細鏍囪 pipeline 閿欒锛屽敜閱掓墍鏈夌瓑寰呰€?*/
    }
    surf = queue_pop_by_timestamp(ctx, ts_to_seq(b.timestamp));
    if (b.flags & V4L2_BUF_FLAG_ERROR) surf->error = true;
    surf->decoded = true;
    surf->state = SURFACE_READY;
    capture_return(ctx, b.index);      /* 鑻ヨ蛋 EXPBUF 鍒悕锛岃繖閲屼笉鑳界珛鍗冲鐢紝瑙?搂4.1 */
    if (b.flags & V4L2_BUF_FLAG_LAST) ctx->draining = false;
}
```

**`vaQuerySurfaceStatus`**锛氭寜鍐呴儴鐘舵€佽繑鍥烇細

| 鍐呴儴鐘舵€?| `VASurfaceStatus` |
|---|---|
| 鍒?`vaCreateSurfaces` | `VASurfaceReady` |
| `vaBeginPicture` 鍚庛€佹湭 `vaSyncSurface` | `VASurfaceRendering`锛堟垨 `VASurfaceDecoding`锛岃嫢澶存枃浠舵湁锛?|
| `vaSyncSurface` 鎴愬姛 | `VASurfaceReady` |
| 瑙ｇ爜閿欒 | `VASurfaceSkipped`锛堝苟閫氳繃 `vaQuerySurfaceError` 杩斿洖 `VA_STATUS_ERROR_DECODING_ERROR`锛?|

> `VASurfaceStatus` 鏋氫妇鍦ㄤ笉鍚?libva 鐗堟湰閲屾湁 `VASurfaceReady/VASurfaceRendering/VASurfaceDisplaying/VASurfaceSkipped` 鍥涚锛沠fmpeg 鍙敤瀹冨垽鏂?鏄惁鍙互鍙栧抚"锛宍Ready`/`Rendering` 鐨勫尯鍒嗕笉涓ユ牸锛屾寜涓婅〃鏄犲皠鍗冲彲銆?
### 3.5 瀵煎嚭涓庡浘鍍忥細`vaExportSurfaceHandle`銆乣vaDeriveImage`

**`vaExportSurfaceHandle`**锛堣瑙?搂4.4锛夋槸 Firefox 璺緞鐨?*蹇呴渶椤?*銆傚疄鐜拌鐐癸細

- 鏀寔 `flags = VA_EXPORT_SURFACE_READ_ONLY | VA_EXPORT_SURFACE_SEPARATE_LAYERS`锛坒fmpeg 灏辨槸杩欎箞璋冪殑锛夛紱
- `desc.objects[0].fd` 蹇呴』鏄?*鏂?dup 鐨?fd**锛堣皟鐢ㄦ柟璐熻矗 close锛夛紱
- NV12 鐢?**2 涓?layer**锛歒 = `DRM_FORMAT_R8`锛孶V = `DRM_FORMAT_GR88`锛屽悓涓€涓?object锛屼笉鍚?offset锛?- `drm_format_modifier` 濉?`DRM_FORMAT_MOD_LINEAR`锛?锛夛紝闄ら潪纭椹卞姩杈撳嚭鐨勬槸 Qualcomm tiled 鏍煎紡锛堣 搂4.2锛夛紱
- `width/height` 濉?surface 鐨?coded 灏哄锛堝 1920脳1088锛夛紝`pitch` 濉?`bytesperline`銆?
**`vaDeriveImage`**锛氱粰 `ffmpeg -vf hwdownload` 鐢ㄣ€?
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
img.offsets[1] = surf->stride * surf->height;   /* 浠ラ┍鍔ㄥ疄闄?UV 鍋忕Щ涓哄噯 */
img.buf = create_internal_buffer(ctx, img.data_size);  /* vaMapBuffer 鑳芥槧灏勫埌 dma-buf */
```

`vaMapBuffer(img.buf)` 杩斿洖 dma-buf 鐨?CPU 鏄犲皠锛坄mmap` 涓€娆★紝缂撳瓨锛夛紝`vaUnmapBuffer` 鍙噺寮曠敤銆傝嫢 CPU 瑕佽鍐欙紝鎴愬浣跨敤 `DMA_BUF_IOCTL_SYNC`锛坄DMA_BUF_SYNC_START/END`锛夐伩鍏嶉潪涓€鑷存€ф灦鏋勪笂鐨勭紦瀛橀棶棰樸€?
**`vaGetImage`/`vaPutImage`**锛歁VP 鍙互杩斿洖 `VA_STATUS_ERROR_UNIMPLEMENTED`锛沗ffmpeg` 鐨?hwdownload 浼樺厛璧?`vaDeriveImage`锛岃蛋涓嶅埌鎵嶅洖閫€ `vaGetImage`銆?
**`vaPutSurface`**锛氫笉瀹炵幇锛岃繑鍥?`VA_STATUS_ERROR_UNIMPLEMENTED`銆侳irefox 涓?`mpv --hwdec=vaapi --vo=gpu-next` 閮戒笉璋冪敤瀹冿紙瀹冧滑璧?export + EGL锛夈€?
### 3.6 鐮佹祦缁勮锛堟湰椤圭洰鐨勬牳蹇冪畻娉曪級

#### 3.6.1 H.264 鐨?AU 缁撴瀯

VA-API 瀹㈡埛绔瘡璋冪敤涓€娆?`vaBeginPicture`/`vaEndPicture`锛岄┍鍔ㄥ氨浜у嚭涓€涓?**access unit锛圓U锛?*锛?
```
00 00 00 01 09 F0                  鈫?AUD锛宲rimary_pic_type=7锛堜换鎰忕被鍨嬶級
00 00 00 01 67 <SPS RBSP>          鈫?SPS锛坣al_ref_idc=3, nal_unit_type=7锛?00 00 00 01 68 <PPS RBSP>          鈫?PPS锛坣al_ref_idc=3, nal_unit_type=8锛?00 00 01 <slice NAL锛屽師鏍?          鈫?鏉ヨ嚜 VASliceDataBufferType
00 00 01 <slice NAL锛屽師鏍?          鈫?澶?slice 渚濇杩藉姞
```

瑙勫垯锛?
1. **涓€涓?`vaEndPicture` = 涓€涓?AU = 涓€娆?`QBUF`**銆傚 slice 鍏ㄩ儴鎷艰繘鍚屼竴涓?buffer銆?2. **slice NAL 鍘熸牱閫忎紶**锛歚VASliceDataBufferType` 鐨勫唴瀹规槸"NAL header + slice header + slice data"锛堜笉鍚捣濮嬬爜锛夈€傞┍鍔ㄥ彧琛?`00 00 01`銆傝嫢瀹㈡埛绔凡缁忓甫浜嗚捣濮嬬爜锛堟娴嬪墠 3/4 瀛楄妭锛夛紝涓嶈閲嶅娣诲姞銆?3. **SPS/PPS 姣忎釜 AU 閮介噸鍙?*锛堟渶绠€鍗曘€佹渶绋筹級銆傚浐浠朵細蹇界暐閲嶅鍙傛暟闆嗭紱濡傛灉瀹炴祴鏈夋€ц兘闂锛屽啀浼樺寲鎴?鍙傛暟鍙樺寲鎴?IDR 鍓嶆墠鍙?銆?4. **AUD 寤鸿鍙?*锛氬府鍥轰欢鐨?AU 杈圭晫妫€娴嬶紝鎴愭湰 6 瀛楄妭銆傝嫢瀹炴祴鍥轰欢瀵?AUD 鏁忔劅锛堝皯鏁板疄鐜颁細锛夛紝鍋氭垚鍙厤缃€?5. **涓嶈閲嶅啓 slice header**锛歴lice header 閲岀殑 `frame_num`銆丳OC銆佸弬鑰冨垪琛ㄣ€丵P 绛夊繀椤讳笌鍘熷鐮佹祦涓€鑷达紝閲嶅啓蹇呴敊銆傞┍鍔ㄥ彧鍋?琛ヨ捣濮嬬爜"鍜?琛ュ弬鏁伴泦"銆?
#### 3.6.2 SPS 瀛楁鏄犲皠锛圚.264锛?
| SPS 璇硶鍏冪礌 | 椹卞姩鍙栧€兼潵婧?| 璇存槑 |
|---|---|---|
| `profile_idc` | config 鐨?VAProfile锛圔aseline=66 / Main=77 / High=100锛?| VA 鐨?picture param 涓嶇洿鎺ョ粰 profile_idc |
| `constraint_set0..5_flags` | 鍏?0 | 淇濆畧锛汢aseline 鐨?constraint_set0 涓嶅啓涔熻兘瑙?|
| `level_idc` | 鎸?Annex A 璁＄畻鏈€灏?level锛堣涓嬶級 | 鍐欏ぇ浜嗛€氬父鏃犲锛屽啓灏忎簡鍥轰欢鍙兘鎷?|
| `seq_parameter_set_id` | 0 | 鍥哄畾 |
| `chroma_format_idc` | `seq_fields.bits.chroma_format_idc`锛堜粎 High 绯诲啓锛?| 8bit 4:2:0 = 1 |
| `bit_depth_luma_minus8` / `chroma` | 鍚屽悕鍙傛暟 | MVP 鎭?0 |
| `qpprime_y_zero_transform_bypass_flag` | 0 | |
| `seq_scaling_matrix_present_flag` | 0 | 缂╂斁鐭╅樀鏀?PPS |
| `log2_max_frame_num_minus4` | `seq_fields.bits.log2_max_frame_num_minus4` | |
| `pic_order_cnt_type` | `seq_fields.bits.pic_order_cnt_type` | |
| `log2_max_pic_order_cnt_lsb_minus4` | 鍚屽悕瀛楁锛坱ype=0 鏃讹級 | |
| `delta_pic_order_always_zero_flag` | 鍚屽悕瀛楁锛坱ype=1 鏃讹級 | |
| `offset_for_non_ref_pic` / `offset_for_top_to_bottom_field` | 0 / 0 | type=1 涓?VA 鏈彁渚涙椂鎸?0 |
| `num_ref_frames_in_pic_order_cnt_cycle` | `num_ref_frames_in_pic_order_cnt_cycle` | 閫氬父 0 |
| `max_num_ref_frames` | `num_ref_frames` | |
| `gaps_in_frame_num_value_allowed_flag` | `seq_fields.bits.gaps_in_frame_num_value_allowed_flag` | |
| `pic_width_in_mbs_minus1` | `picture_width_in_mbs_minus1` | |
| `pic_height_in_map_units_minus1` | 鐢?`picture_height_in_mbs_minus1` 涓?`frame_mbs_only_flag` 鎺ㄥ | |
| `frame_mbs_only_flag` | 鎭?1 | **涓嶆敮鎸佸満缂栫爜/PAFF/MBAFF**锛岄亣鍒板氨杩斿洖 `VA_STATUS_ERROR_DECODING_ERROR` |
| `mb_adaptive_frame_field_flag` | 0 | |
| `direct_8x8_inference_flag` | 1 | |
| `frame_cropping_flag` | 0锛堣 搂3.7锛?| |
| `vui_parameters_present_flag` | 0 | 鏃?VUI 涓嶅奖鍝嶈В鐮?|

**level_idc 鐨勮绠?*锛堝啓姝?51 涔熻兘璺戯紝浣嗘寜瑙勮寖绠楁洿绋筹級锛?
```c
int pick_level(int w_mbs, int h_mbs, int max_refs) {
    int pic_size = w_mbs * h_mbs;                 /* PicSizeInMbs */
    int max_dpb  = pic_size * (max_refs + 1);     /* MaxDpbMbs 闇€姹?*/
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

#### 3.6.3 PPS 瀛楁鏄犲皠锛圚.264锛?
**PPS 鐨?`pic_parameter_set_id` 蹇呴』涓?slice header 閲岀殑涓€鑷淬€?* VA-API 涓嶇洿鎺ョ粰杩欎釜鍊硷紝浣嗗彲浠ヤ粠 slice NAL 鐨勫ご閮ㄨВ鏋愬嚭鏉ワ細`first_mb_in_slice`锛坲e锛夈€乣slice_type`锛坲e锛夈€乣pic_parameter_set_id`锛坲e锛夆€斺€斿彧闇€瑕佷笁涓?ue(v)锛屼笉渚濊禆 PPS 鍐呭銆?
| PPS 璇硶鍏冪礌 | 鏉ユ簮 |
|---|---|
| `pic_parameter_set_id` | 浠?slice NAL 瑙ｆ瀽锛堣涓婏級锛涜嫢鍚屼竴 AU 鍐呭涓?slice 鐨?pps_id 涓嶅悓 鈫?鎷掔粷 |
| `seq_parameter_set_id` | 0 |
| `entropy_coding_mode_flag` | `pic_fields.bits.entropy_coding_mode_flag` |
| `bottom_field_pic_order_in_frame_present_flag` | `pic_fields.bits.pic_order_present_flag` |
| `num_slice_groups_minus1` | 0锛堜笉鏀寔 FMO锛?|
| `num_ref_idx_l0_default_active_minus1` / `l1` | 鍚屽悕瀛楁 |
| `weighted_pred_flag` / `weighted_bipred_idc` | 鍚屽悕瀛楁 |
| `pic_init_qp_minus26` / `pic_init_qs_minus26` | 鍚屽悕瀛楁 |
| `chroma_qp_index_offset` | 鍚屽悕瀛楁 |
| `deblocking_filter_control_present_flag` | 1 |
| `constrained_intra_pred_flag` | 鍚屽悕瀛楁 |
| `redundant_pic_cnt_present_flag` | 0 |
| `transform_8x8_mode_flag` | `pic_fields.bits.transform_8x8_mode_flag` |
| `pic_scaling_matrix_present_flag` + 鍒楄〃 | 鐢?`VAQMatrixBufferH264` 鐨?`scaling_lists_4x4[6][16]` / `scaling_lists_8x8[2][64]` 鍐冲畾锛氬叏 16/64 灏辩疆 0锛屽惁鍒欑疆 1 骞跺啓鍏?|
| `second_chroma_qp_index_offset` | 鍚屽悕瀛楁 |

#### 3.6.4 RBSP 鍐欏叆涓庨槻绔炰簤瀛楄妭

```c
struct bs { uint8_t *buf; size_t cap, len; uint32_t bitpos; };

void bs_put_bits(struct bs *b, uint32_t val, int n);   /* 楂樹綅鍏堝啓 */
void bs_put_ue(struct bs *b, uint32_t v);              /* Exp-Golomb */
void bs_put_se(struct bs *b, int32_t v);
void bs_rbsp_trailing(struct bs *b);                   /* 琛?1 鍚庡榻?0 */

/* 鐢熸垚 NAL锛氬厛鍐?1 瀛楄妭 NAL header锛屽啀鍐?RBSP 杞借嵎锛屾渶鍚庢彃 EPB */
size_t nal_wrap(uint8_t *out, int nal_ref_idc, int type,
                const uint8_t *rbsp, size_t rbsp_len)
{
    /* 00 00 00/01/02/03 涔嬪悗鎻掑叆 0x03 */
}
```

**鍗曞厓娴嬭瘯鏂规硶锛堜笉闇€瑕佺‖浠讹級**锛氭妸閲嶅缓鍣ㄤ骇鍑虹殑 SPS/PPS + 鍘熷 slice NAL 鎷兼垚 `.h264` 鏂囦欢锛岀敤 `ffmpeg -f h264 -i synth.h264 -f null -` 杞В銆傝嫢杞В鍣ㄦ帴鍙椾笖甯ф暟涓庡弬鑰冧竴鑷达紝璇存槑璇硶姝ｇ‘銆傝繖涓祴璇曞彲浠ュ湪 x86 PC 涓婅窇锛屾槸鐮佹祦閲嶅缓鍣ㄧ殑涓昏寮€鍙戞墜娈碉紙搂9.2锛夈€?
#### 3.6.5 HEVC 涓?VP9 鐨勫彲琛屾€?
| Codec | VA 鎻愪緵鐨勫弬鏁?| 闇€瑕侀噸寤?| 缁撹 |
|---|---|---|---|
| H.264 | picture/slice/qmatrix锛屽瓧娈甸綈鍏?| SPS + PPS | **鍙**锛堟湰椤圭洰涓昏矾绾匡級 |
| HEVC | `VAPictureParameterBufferHEVC` + `VASliceParameterBufferHEVC` | VPS + SPS锛堝惈 RPS锛? PPS | **楂橀闄?*锛歋PS 鐨?short-term RPS 鑻ュ湪 VA 鍙傛暟閲屾病鏈夊畬鏁磋〃杈撅紝slice header 鐨?`short_term_ref_pic_set_sps_flag=1` 灏辨棤娉曞尮閰嶃€傞渶鏍稿 `va_dec_hevc.h`锛涙牳瀵逛笉閫氳繃鍒?HEVC 鍙兘璧?R2 鐩撮€?|
| VP9 | `VADecPictureParameterBufferVP9`锛坲ncompressed header 鍏ㄥ瓧娈碉級 | VP9 uncompressed header | **鍙**锛屼絾宸ヤ綔閲忓ぇ锛涘帇缂╁ご/tile 鏁版嵁鍦?slice data 閲屽師鏍烽€忎紶 |

**HEVC 鐨勫叿浣撴鏌ョ偣**锛圡0 鍋氾級锛氬湪 `va_dec_hevc.h` 閲屾壘 `num_short_term_ref_pic_sets`銆乣st_rps`銆乣st_rps_bits` 涔嬬被鐨勫瓧娈碉紝骞剁‘璁?`VASliceParameterBufferHEVC` 鏄惁甯?`slice_pic_parameter_set_id`銆傚鏋?RPS 淇℃伅涓嶅叏锛孒EVC 闃舵鐩存帴鏀圭敤 R2銆?
### 3.7 coded size銆佽鍓笌鏄剧ず灏哄

H.264 鐨?coded 楂樺害鏄?16 鐨勫€嶆暟锛?080p 鈫?1088銆俈A surface 閫氬父鎸?coded size 鍒嗛厤锛坒fmpeg 浼?`coded_width/coded_height`锛夛紝鑰屾樉绀哄昂瀵告槸 1920脳1080銆備笁鏉¤矾锛?
1. **SPS 鍐?`frame_cropping_flag=0`**锛堟帹鑽?MVP锛夛細鍥轰欢杈撳嚭 1088 琛岋紝surface 涔熸槸 1088 琛岋紱ffmpeg 鐨?`AVFrame` 閫氬父甯?`width=1920,height=1080`锛堟樉绀哄昂瀵革級锛宑rop 鐢卞鎴风澶勭悊銆?*蹇呴』瀹炴祴纭 Firefox 鐨?`DMABufSurface` 鐢ㄥ摢涓珮搴﹀缓绾圭悊**鈥斺€旇嫢瀹冪敤 descriptor 鐨?`height`锛?088锛夛紝搴曢儴浼氭湁 8 琛岀豢杈癸紝闇€瑕佸湪瀵煎嚭 descriptor 鏃跺～鏄剧ず楂樺害锛屾垨鏀圭敤 cropping銆?2. **浠庡弬鏁版帹瀵?cropping**锛歚frame_cropping_flag=1` + `frame_crop_bottom_offset=(coded_h - display_h)/2`銆備絾 VA-API 鐨?H.264 picture param 鏄惁鎻愪緵鏄剧ず楂樺害闇€瑕佹牳瀵癸紱鑻ユ病鏈夛紝鍙兘鐢?`vaCreateContext` 鐨?height锛坈oded锛変笌瀹㈡埛绔悗缁涓烘帹鏂紝涓嶅彲闈犮€?3. **鍦ㄥ鍑鸿矾寰勮鍓?*锛歞escriptor 鐨?`height` 濉樉绀洪珮搴︺€乣pitch` 涓嶅彉銆俧fmpeg 鐨?`AVDRMFrameDescriptor` 浼氬甫涓?`height`锛孎irefox 鎸夊畠寤虹汗鐞嗐€傝繖鏉¤矾涓嶄緷璧?SPS锛屾渶鍙帶銆?
**寤鸿锛歁VP 鐢ㄦ柟妗?1 + 鏂规 3 鍏滃簳**锛屽苟鍦?搂9.3 鍔犱竴涓?1080p 搴曢儴 8 琛屽儚绱?鐨勪笓椤规鏌ワ紙鐢?`ffmpeg` 瀵煎嚭棣栧抚 PNG锛岃倝鐪?鍍忕礌姣斿锛夈€?
### 3.8 閿欒鐮佹槧灏?
| 鍦烘櫙 | 杩斿洖 |
|---|---|
| V4L2 `open`/`QUERYCAP` 澶辫触 | `VA_STATUS_ERROR_OPERATION_FAILED` |
| profile 涓嶆敮鎸侊紙濡?`VAProfileHEVCMain10`锛?| `VA_STATUS_ERROR_UNSUPPORTED_PROFILE` |
| entrypoint 涓嶆槸 VLD | `VA_STATUS_ERROR_UNSUPPORTED_ENTRYPOINT` |
| `rt_format` 涓嶆槸 YUV420 | `VA_STATUS_ERROR_UNSUPPORTED_RT_FORMAT` |
| `vaCreateConfig` 鍙傛暟闈炴硶 | `VA_STATUS_ERROR_INVALID_PARAMETER` |
| `REQBUFS`/`mmap` 澶辫触 | `VA_STATUS_ERROR_ALLOCATION_FAILED` |
| `QBUF` 杩斿洖 `EINVAL` | `VA_STATUS_ERROR_INVALID_PARAMETER`锛堝厛鏌?`sizeimage`/`bytesused`锛?|
| `DQBUF` 甯?`V4L2_BUF_FLAG_ERROR` | `VA_STATUS_ERROR_DECODING_ERROR` |
| `SOURCE_CHANGE` 鍚庢柊鍒嗚鲸鐜?> surface 灏哄 | `VA_STATUS_ERROR_RESOLUTION_NOT_SUPPORTED` |
| `vaSyncSurface` 瓒呮椂锛堥粯璁?2 s锛?| `VA_STATUS_ERROR_OPERATION_FAILED`锛堝悓鏃舵妸 pipeline 鏍囦负 error锛屽悗缁皟鐢ㄥ揩閫熷け璐ワ級 |
| 涓嶆敮鎸佺殑 slice data 鍒嗙墖锛坄slice_data_flag != ALL`锛?| `VA_STATUS_ERROR_FLAG_NOT_SUPPORTED` |
| 鍦虹紪鐮?/ FMO / 澶?PPS 鍚屼竴 AU | `VA_STATUS_ERROR_DECODING_ERROR` |

---

## 4. 缂撳啿鍖轰笌闆舵嫹璐?
### 4.1 涓ゆ潯闃熷垪鐨勫唴瀛樼被鍨?
| 闃熷垪 | 寤鸿 memory | 鏁伴噺 | 鍗?buffer 澶у皬 | 鐞嗙敱 |
|---|---|---|---|---|
| OUTPUT锛堢爜娴侊級 | `V4L2_MEMORY_MMAP` | 4 | 2鈥? MiB | 鐮佹祦灏忋€丆PU 鏈潵灏辫鍐欙紝DMABUF 娌℃湁鏀剁泭 |
| CAPTURE锛堝抚锛塎VP 榛樿 | `V4L2_MEMORY_MMAP` + memcpy 鍒?surface 鑷湁 dma-buf | 8 | `bytesperline*height*3/2` | 鏈€绋筹細surface 涓?V4L2 buffer 瑙ｈ€︼紝context 鐢熷懡鍛ㄦ湡/seek 閮戒笉鐗佃繛锛圓DR-1锛屄?.1锛?|
| CAPTURE锛堝抚锛塋2 浼樺寲 | `V4L2_MEMORY_MMAP` + `VIDIOC_EXPBUF` 鍒悕 | 8鈥?2 | 鍚屼笂 | 闆舵嫹璐濓細V4L2 buffer 鐩存帴褰?VA surface 瀵煎嚭锛涗唬浠锋槸 surface 鏁?= CAPTURE buffer 鏁帮紙鍒涘缓椤哄簭鏈夌害鏉燂紝瑙?搂6.1锛?|
| CAPTURE锛堝閫夛級 | `V4L2_MEMORY_DMABUF` + gbm 鍒嗛厤鐨?NV12 bo | 鍚屼笂 | 鍚屼笂 | 鑻?`EXPBUF` 涓庢嫹璐濋兘涓嶅悎閫傛椂鐨勭涓夋潯璺?|

**涓轰粈涔?MVP 涓嶇洿鎺ュ仛闆舵嫹璐?*锛歏A surface 鏄?display 绾у璞★紝`vaCreateSurfaces` 鏃惰繕涓嶇煡閬?codec锛坈odec 鍦?`vaCreateContext` 鎵嶇‘瀹氾級锛岃€?CAPTURE 闃熷垪鐨?`S_FMT`/`REQBUFS` 渚濊禆 codec銆傞『搴忓涓嶄笂锛屽己琛屽仛浼氳 `vaCreateSurfaces` 鍜?`vaCreateContext` 浜掔浉渚濊禆銆傛嫹璐濇ā寮忕敤涓€娆?memcpy 鎹㈡帀杩欎釜鑰﹀悎锛?080p30 鐨勪唬浠风害 93 MB/s锛屽彲鎺ュ彈锛埪?.1 ADR-1锛夈€?
**L2 涓轰粈涔堜紭鍏?`MMAP + EXPBUF` 鑰屼笉鏄?gbm**锛?
- vb2 鐨?MMAP buffer 鐢卞唴鏍稿垎閰嶏紝`VIDIOC_EXPBUF` 鍙互鎶婂畠瀵煎嚭鎴?dma-buf锛堥渶瑕?memops 瀹炵幇 `get_dmabuf`锛宨ris 鐢?`vb2_dma_sg`锛?*寰呴獙璇?*锛夛紱
- 瀵煎嚭鐨?dma-buf 鍜?V4L2 姝ｅ湪鍐欑殑鍐呭瓨鏄?*鍚屼竴鍧楃墿鐞嗗唴瀛?*锛孷A surface 鐩存帴鍒悕瀹冿紝`vaExportSurfaceHandle` 鍙槸 `dup` fd锛岄浂鎷疯礉锛?- 涓嶉渶瑕?gbm 鍙備笌锛屽皯涓€灞傚垎閰嶅櫒渚濊禆銆?
**蹇呴』鍏堥獙璇佺殑涓変欢浜?*锛圡1 鐢?`tools/v4l2-probe.c` 鍋氾級锛?
```c
/* 1. EXPBUF 鑳戒笉鑳界敤 */
struct v4l2_exportbuffer exp = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
                                 .index = 0, .plane = 0, .flags = O_RDWR };
if (ioctl(fd, VIDIOC_EXPBUF, &exp) < 0) perror("EXPBUF");   /* ENOTTY/EINVAL 鈫?璧板閫?*/

/* 2. 瀵煎嚭鐨?fd 鑳戒笉鑳?mmap锛堣兘 mmap 鎵嶈兘 vaDeriveImage锛?*/
void *p = mmap(NULL, len, PROT_READ, MAP_SHARED, exp.fd, 0);
if (p == MAP_FAILED) perror("mmap dmabuf");

/* 3. 瀵煎嚭 fd 鐨?size / modifier */
struct stat st; fstat(exp.fd, &st);
/* Qualcomm 鑻ヨ緭鍑?tiled 鏍煎紡锛宮odifier 涓嶄細浣撶幇鍦?fd 涓婏紱闇€瑕佹煡 iris 鐨?S_FMT 鏄惁鎺ュ彈 linear 鏍囧織 */
```

濡傛灉 `EXPBUF` 涓嶅彲鐢紝閫€鍥?MMAP + memcpy"锛屽苟鍦?搂9.6 璁板綍鎷疯礉寮€閿€锛涗笉瑕佷竴寮€濮嬪氨涓?gbm锛実bm 鐨?NV12 鏀寔鍦?msm 涓婁篃瑕侀獙璇併€?
### 4.2 surface 鐨?dma-buf 浠庡摢鏉?
鑻ヨ蛋 `EXPBUF` 鍒悕锛歴urface 娌℃湁鑷繁鐨?dma-buf锛宍surface->dmabuf_fd = exp.fd`锛宍surface->stride/offset/size` 鍏ㄩ儴鏉ヨ嚜 `VIDIOC_QUERYBUF` 鐨?`plane_fmt`銆?
鑻ヨ蛋鎷疯礉璺緞锛歴urface 鍦?`vaCreateSurfaces` 鏃跺垎閰嶄竴鍧?NV12 dma-buf銆備笁绉嶅垎閰嶆柟寮忥紝鎸変紭鍏堢骇锛?
```c
/* 鏂瑰紡 1锛歡bm锛堟帹鑽愶紝Mesa 鏍堥噷鏈€鑷劧锛?*/
struct gbm_device *gbm = gbm_create_device(ctx->drm_fd);
struct gbm_bo *bo = gbm_bo_create(gbm, w, h, GBM_FORMAT_NV12,
                                  GBM_BO_USE_LINEAR | GBM_BO_USE_RENDERING);
int fd = gbm_bo_get_fd(bo);
uint32_t stride = gbm_bo_get_stride(bo);
uint64_t mod    = gbm_bo_get_modifier(bo);

/* 鏂瑰紡 2锛欴RM dumb buffer锛坙inear 淇濊瘉锛屼絾鏌愪簺 GPU 涓嶈兘閲囨牱锛?*/
struct drm_mode_create_dumb c = { .width = w, .height = h * 3 / 2, .bpp = 8 };
ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &c);
ioctl(drm_fd, DRM_IOCTL_PRIME_HANDLE_TO_FD, &prime);

/* 鏂瑰紡 3锛歶dmabuf锛堥渶瑕?/dev/udmabuf 涓庡唴鏍?CONFIG_UDMABUF锛?*/
```

**鍏抽敭鍐崇瓥锛氬己鍒?LINEAR銆?* 鍘熷洜锛?
- Firefox 鐨?EGL 瀵煎叆璺緞瀵?modifier 鐨勬敮鎸佸彇鍐充簬 Mesa 鐗堟湰锛沗DRM_FORMAT_MOD_LINEAR` 鏄渶淇濆畧銆佹渶鍙Щ妞嶇殑閫夋嫨锛?- Qualcomm 鐨?tiled NV12锛坄QCOM_COMPRESSED`/`UBWC`锛夐渶瑕?freedreno 涓?iris 閮藉崗鍟嗕竴鑷达紝MVP 涓嶇锛?- 鎷疯礉璺緞涓嬪唴瀛樻槸鎴戜滑鑷繁鍒嗛厤鐨勶紝澶╃劧 linear銆?
**瀵归綈瑕佹眰**锛堥伩鍏?EGL 瀵煎叆澶辫触锛夛細

- `stride` 16 瀛楄妭瀵归綈锛堟帹鑽?256锛屽吋椤炬€ц兘鍜屽吋瀹癸級锛?- UV 骞抽潰鍋忕Щ `stride * height`锛宍height` 鐢?coded 楂樺害锛?088锛変互淇濊瘉 4K/1080p 閮藉锛?- 鎬诲ぇ灏?鈮?`stride * height * 3 / 2`锛屽悜涓婂彇鏁村埌 4096銆?
### 4.3 surface 瀵硅薄涓?`vaCreateSurfaces`

```c
struct iris_surface {
    VASurfaceID id;
    uint32_t width, height;        /* coded */
    uint32_t stride, y_offset, uv_offset, size;
    int dmabuf_fd;                 /* 鎷ユ湁鎴栧埆鍚?*/
    uint64_t modifier;
    enum { SURFACE_FREE, SURFACE_DECODING, SURFACE_READY, SURFACE_ERROR } state;
    uint64_t frame_seq;            /* 鎻愪氦鏃剁粦瀹?*/
    bool exported;                 /* 宸?dup 缁欏鎴风 */
};
```

`vaCreateSurfaces` 瑕佸鐞嗕袱绫昏姹傦細

1. **椹卞姩鍒嗛厤**锛團irefox/ffmpeg 瑙ｇ爜鐨勫父鎬侊級锛歚attrib_list` 閲岄€氬父鍙湁 `VASurfaceAttribPixelFormat = VA_FOURCC_NV12`銆傚埆鍚嶆ā寮忎笅浠?CAPTURE 姹犲彇涓€涓┖闂?buffer锛汳VP 鐨勬嫹璐濇ā寮忓垯鎸?搂4.2 鏂板垎閰嶄竴鍧?dma-buf锛屽～ surface銆?2. **澶栭儴鍐呭瓨瀵煎叆**锛坄ffmpeg -hwaccel vaapi` 鐨勬煇浜涜矾寰勩€佷互鍙?`vaPutSurface` 绫荤敤娉曪級锛氬鎴风缁?`VASurfaceAttribMemoryType = VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` + `VASurfaceAttribExternalBuffers`銆傛鏃讹細

```c
VASurfaceAttribExternalBuffers *ext = attrib->value.pExtBuf;
/* ext->buffers[0] 鏄?dma-buf fd锛宔xt->pitches[0]/offsets[0]/data_size 宸茬粰 */
/* 鐩存帴 mmap 鎴?dup锛宻urface 涓嶅弬涓?CAPTURE 姹?*/
```

**MVP 鍙互鍙敮鎸佺 1 绫?*锛岀 2 绫昏繑鍥?`VA_STATUS_ERROR_UNSUPPORTED_MEMORY_TYPE`锛堝厛纭 Firefox 涓嶈蛋杩欐潯璺紱ffmpeg 鐨?VA-API 瑙ｇ爜鍣ㄩ粯璁よ蛋绗?1 绫伙級銆?
`vaQuerySurfaceAttributes` 杩斿洖锛?
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

### 4.4 `vaExportSurfaceHandle` 鐨勫疄鐜?
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
    iris_sync_surface(ctx, sid);          /* 骞傜瓑锛氱‘淇濆凡 DQBUF */

    VADRMPRIMESurfaceDescriptor *d = descriptor;
    memset(d, 0, sizeof(*d));
    d->fourcc = VA_FOURCC_NV12;
    d->width  = s->width;
    d->height = s->height;                /* 瑁佸壀绛栫暐瑙?搂3.7 */
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

**娉ㄦ剰**锛?
- `dup()` 鍑烘潵鐨?fd 鐢辫皟鐢ㄦ柟鍏抽棴锛涗笉瑕佸湪瀵煎嚭鍚庡叧闂?surface 鑷繁鐨?fd銆?- 鑻?surface 杩樻病 ready锛?*鍦ㄥ鍑洪噷鍋氫竴娆″悓姝?*锛坒fmpeg 鏈夋椂涓嶆樉寮?`vaSyncSurface` 灏?map锛夛紝閬垮厤瀵煎嚭鍗婃垚鍝併€?- `VA_EXPORT_SURFACE_WRITE_ONLY` 涓嶆敮鎸侊紙瑙ｇ爜杈撳嚭鍙锛夈€?- 濡傛灉瀹㈡埛绔姹?`COMPOSED_LAYERS`锛屽彲浠ヨ繑鍥?`VA_STATUS_ERROR_FLAG_NOT_SUPPORTED`锛沠fmpeg 榛樿鐢?`SEPARATE_LAYERS`銆?
### 4.5 Firefox/Wayland 渚ф€庝箞 import

閾捐矾锛?
```
ffmpeg hwcontext_vaapi.c: vaapi_map_to_drm_esh()
   鈹斺攢 vaExportSurfaceHandle(VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2)
        鈹斺攢 VADRMPRIMESurfaceDescriptor
             鈹斺攢 AVDRMFrameDescriptor锛坒fmpeg 杞崲锛?                  鈹斺攢 Firefox DMABufSurface锛圕reateDMABufSurface / YUV 鍙樹綋锛?                       鈹斺攢 EGL_EXT_image_dma_buf_import锛堟瘡灞備竴涓?EGLImage锛?                            鈹斺攢 Mesa freedreno 閲囨牱 NV12 绾圭悊
                                 鈹斺攢 WebRender 鍚堟垚鍒?wl_surface
```

瑕佹眰锛?
1. **EGL 鎵╁睍**锛歚EGL_EXT_image_dma_buf_import`銆乣EGL_EXT_image_dma_buf_import_modifiers`銆乣EGL_KHR_image_base`銆傜敤 `eglinfo -B` 鎴?`eglQueryString(display, EGL_EXTENSIONS)` 纭锛圡esa 26.0.3 鐨?freedreno 搴旇鏈夛級銆?2. **鏍煎紡**锛歚DRM_FORMAT_NV12`锛圷=R8锛孶V=GR88锛夈€傚鏋?Mesa 瀵?`GR88` 鐨?NV12 瀵煎叆鏈夐澶栧榻愯姹傦紝鎸?搂4.2 鐨?256 瀛楄妭 stride 瀵归綈瑙勯伩銆?3. **modifier**锛歚DRM_FORMAT_MOD_LINEAR`銆傝嫢鐢?Qualcomm tiled锛岄渶瑕?`EGL_EXT_image_dma_buf_import_modifiers` 涓?freedreno 璁よ瘑璇?modifier锛汳VP 涓嶅仛銆?4. **鍚屾**锛歏4L2 娌℃湁 dma-fence 瀵煎嚭锛宍DQBUF` 杩斿洖鍗宠〃绀虹‖浠跺啓瀹屻€侳irefox 鎷垮埌鐨勬槸"宸茬粡瀹屾垚鐨勫抚"锛屼笉闇€瑕侀澶?fence锛?*涓嶈鍦?DQBUF 涔嬪墠瀵煎嚭**銆?5. **Wayland**锛氳В鐮佸抚鏄?WebRender 鐨勭汗鐞嗭紝涓嶇粡杩?`zwp_linux_dmabuf_v1` 閫佸悎鎴愬櫒锛涙墍浠?`wl_drm`/`linux-dmabuf` 鐨勫吋瀹规€ч棶棰樹笉浼氬崱浣忚棰戯紝鍙 EGL 瀵煎叆鎴愬姛鍗冲彲銆?
### 4.6 缂撳啿鍖虹敓鍛藉懆鏈熻鍒欙紙瀹规槗鍑?use-after-free 鐨勫湴鏂癸級

| 浜嬩欢 | 蹇呴』鍋氫粈涔?|
|---|---|
| `vaCreateSurfaces` | 浠庢睜鍙?buffer锛屾爣璁?`IN_USE`锛涜嫢璧?EXPBUF 鍒悕锛?*涓嶈兘**鎶?V4L2 buffer 閲嶆柊 `QBUF` 缁欎笅涓€娆¤В鐮侊紝鐩村埌 surface 琚?`vaDestroySurfaces` |
| `vaEndPicture` | surface 缁戝畾 `frame_seq`锛宐uffer 浠嶅睘浜?V4L2锛堝凡鍏ラ槦 CAPTURE锛?|
| `vaSyncSurface` | 鍑洪槦鍚?surface 鍙鍑猴紱buffer 浠嶅睘浜?surface |
| `vaDestroySurfaces` | 鑻?buffer 宸插嚭闃熶笖 surface 涓嶅啀浣跨敤锛岄噸鏂?`QBUF` 鍥?CAPTURE 闃熷垪锛堝埆鍚嶆ā寮忥級锛涜嫢鏄嚜鏈?dma-buf锛岀洿鎺ラ噴鏀?|
| `vaDestroyContext` | 鍏?`STREAMOFF`锛堜細涓㈠純鍦ㄩ€?buffer锛夛紝鍐?`REQBUFS(0)`锛屽啀 `munmap`/`close` |
| `vaTerminate` | 閲婃斁鎵€鏈?config/context/surface/buffer锛屽叧闂?`/dev/video0` |

**鍒悕妯″紡鐨勬渶澶ч櫡闃?*锛歏4L2 CAPTURE 闃熷垪閲岀殑 buffer 鏁伴噺 = surface 姹犲ぇ灏忋€傚鏋滃鎴风鍒涘缓鐨?surface 鏁?> 姹犲ぇ灏忥紝`vaCreateSurfaces` 浼氬け璐ャ€傜紦瑙ｏ細

- 姹犲ぇ灏?= `min(32, max(16, 瀹㈡埛绔涓€娆?vaCreateSurfaces 鐨?num_surfaces + 4))`锛?- 鑻ュ悗缁?surface 鏁拌秴鍑猴紝鍋氫竴娆?鍋滄祦 鈫?`REQBUFS(0)` 鈫?鎵╁ 鈫?閲嶅缓 鈫?寮€娴?锛屽苟鎶婂凡鏈?surface 閲嶆柊缁戝畾锛堝鏉傦紝L2 鍐嶅仛锛夛紱
- 鎴栬€呴€€鍥炴嫹璐濇ā寮忥紙surface 鑷湁 dma-buf锛孋APTURE 鍙 8 涓?buffer锛夛紝褰诲簳瑙ｈ€︺€?
---

## 5. profile / entrypoint 鐭╅樀

### 5.1 鏀寔鐭╅樀

| VA profile | 浣嶆繁/鑹插害 | V4L2 OUTPUT pixfmt | V4L2 CAPTURE pixfmt | 鏈」鐩?| 璇存槑 |
|---|---|---|---|---|---|
| `VAProfileH264Baseline` | 8bit 4:2:0 | `V4L2_PIX_FMT_H264` | `V4L2_PIX_FMT_NV12` | 鉁?L1 | MVP 棣栦釜鐩爣锛圕AVLC銆佹棤 B 甯с€佹棤閲嶆帓锛屾渶濂借皟锛?|
| `VAProfileH264ConstrainedBaseline` | 8bit 4:2:0 | 鍚屼笂 | 鍚屼笂 | 鉁?L1 | 寰堝 baseline 娴佸疄闄呮爣杩欎釜 |
| `VAProfileH264Main` | 8bit 4:2:0 | 鍚屼笂 | 鍚屼笂 | 鉁?L2 | CABAC銆丅 甯?|
| `VAProfileH264High` | 8bit 4:2:0 | 鍚屼笂 | 鍚屼笂 | 鉁?L2 | 8x8 transform銆乻caling list |
| `VAProfileH264High10` | 10bit 4:2:0 | 锛堥渶 `V4L2_PIX_FMT_P010`锛?| 鏃?| 鉂?| 杩斿洖 `VA_STATUS_ERROR_UNSUPPORTED_PROFILE` |
| `VAProfileH264MultiviewHigh` / `StereoHigh` | 8bit | 鍚屼笂 | 鍚屼笂 | 鉂?| 杩斿洖涓嶆敮鎸?|
| `VAProfileHEVCMain` | 8bit 4:2:0 | `V4L2_PIX_FMT_HEVC` | `NV12` | 鈿狅笍 L4 | 渚濊禆 VPS/SPS/RPS 閲嶅缓锛岃 搂3.6.5 |
| `VAProfileHEVCMain10` | 10bit 4:2:0 | 锛堥渶 P010锛?| 鏃?| 鉂?| capture 渚ф病鏈?10bit 鏍煎紡 |
| `VAProfileVP9Profile0` | 8bit 4:2:0 | `V4L2_PIX_FMT_VP9` | `NV12` | 鈿狅笍 L4 | 瑙?搂3.6.5 |
| `VAProfileVP9Profile2` | 10bit 4:2:0 | 锛堥渶 P010锛?| 鏃?| 鉂?| |
| `VAProfileAV1Profile0/1` | 鈥?| 鈥?| 鈥?| 鉂?| `/dev/video0` 鏈灇涓?`AV1` |
| `VAProfileMPEG2*` / `VC1*` / `VP8` | 鈥?| 鈥?| 鈥?| 鉂?| 鏈灇涓?|

鍏ュ彛鐐瑰彧鏈?`VAEntrypointVLD`銆俙vaQueryConfigEntrypoints` 杩斿洖 1 椤癸紱瀵?`VAEntrypointEncSlice`/`VAEntrypointVideoProc` 杩斿洖 `VA_STATUS_ERROR_UNSUPPORTED_ENTRYPOINT`銆?
### 5.2 涓轰粈涔?10bit 鐜板湪涓嶈兘鍋?
銆愬疄娴嬨€慽ris 鐨?CAPTURE 渚у彧鏋氫妇鍒?`NV12`銆傝繖鎰忓懗锛?
- 鍐呮牳/FW 娌℃湁鎶?10bit 杈撳嚭鏍煎紡鏆撮湶鍑烘潵锛堝彲鑳芥槸椹卞姩鏈疄鐜般€佷篃鍙兘鏄 SoC 鐨?HFI 鍥轰欢鑳藉姏鏈０鏄庯級锛?- VA 渚ц嫢瑕佹敮鎸?`VAProfileHEVCMain10`锛屽繀椤昏兘鎷垮埌 `P010`锛堟垨 `NV12_10`锛夌殑 dma-buf锛屽苟鎸?`DRM_FORMAT_P010` 瀵煎嚭锛?- 鍦?CAPTURE 鍙湁 NV12 鐨勫墠鎻愪笅锛屼换浣?10bit 杞浆 8bit"鐨勫仛娉曢兘鍙戠敓鍦ㄥ唴鏍?FW 涔嬪锛屾垜浠嬁涓嶅埌 10bit 鏁版嵁锛?*鏃犳硶鏀寔**銆?
澶勭悊鏂瑰紡锛?
1. `vaQueryConfigProfiles` **涓嶈繑鍥?* 10bit profile锛?2. `vaCreateConfig(VAProfileHEVCMain10)` 杩斿洖 `VA_STATUS_ERROR_UNSUPPORTED_PROFILE`锛?3. 鑻ュ鎴风浼?`rt_format = VA_RT_FORMAT_YUV420_10`锛岃繑鍥?`VA_STATUS_ERROR_UNSUPPORTED_RT_FORMAT`锛?4. 鍦?README 涓?`about:support` 鎺掗殰璇存槑閲屽啓娓呮"10bit 璧拌蒋瑙?锛岄伩鍏嶇敤鎴疯鍒や负 bug銆?
**鍚庣画楠岃瘉鐐?*锛歁1 鏃剁敤 `v4l2-ctl -d /dev/video0 --list-formats` 鍐嶇‘璁や竴娆★紱鑻ュ唴鏍稿崌绾у悗鍑虹幇 `P010`锛屾寜 搂4.2 鐨?modifier/stride 瑙勫垯鎵╁睍锛屽伐浣滈噺绾?3~5 浜哄ぉ锛圴A 渚ф敼鍔ㄥ緢灏忥紝涓昏鏄鍑烘牸寮忎笌 Firefox 鐨?10bit 绾圭悊璺緞锛夈€?
### 5.3 `vaQueryConfigProfiles` 鐨勫疄鐜?
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
    /* L4 鎵撳紑锛歿 VAProfileHEVCMain, V4L2_PIX_FMT_HEVC, "HEVC Main" }, */
    /* L4 鎵撳紑锛歿 VAProfileVP9Profile0, V4L2_PIX_FMT_VP9, "VP9 Profile0" }, */
};

VAStatus iris_QueryConfigProfiles(VADriverContextP ctx, VAProfile *profiles, int *num)
{
    struct iris_driver *drv = ctx->pDriverData;
    int n = 0;
    for (size_t i = 0; i < ARRAY_SIZE(k_profiles); i++) {
        if (!drv->has_pixfmt[k_profiles[i].pixfmt])   /* ENUM_FMT 缂撳瓨 */
            continue;
        if (!iris_try_fmt_ok(drv, k_profiles[i].pixfmt, 1920, 1088))
            continue;                                  /* TRY_FMT 澶嶆牳 */
        profiles[n++] = k_profiles[i].va;
    }
    *num = n;
    return VA_STATUS_SUCCESS;
}
```

瑕佺偣锛?
- 杩斿洖椤哄簭涓嶉噸瑕侊紝浣?*涓嶈杩斿洖閲嶅椤?*锛坙ibva 浼氬師鏍烽€忎紶锛屽鎴风鍙兘鍘婚噸澶辫触锛夈€?- profile 鍒楄〃瑕佽兘琚紪璇戞湡寮€鍏宠鍓紙`-Dh264=disabled` 涔嬬被锛夛紝鏂逛究瀹氫綅"鏄┍鍔ㄩ棶棰樿繕鏄爜娴侀棶棰?銆?- `iris_try_fmt_ok` 鐢?`VIDIOC_TRY_FMT`锛屼笉瑕佺敤 `S_FMT`锛堜細鏀硅澶囩姸鎬侊級銆?
### 5.4 `vaCreateConfig` 鐨勬牎楠岄『搴?
```c
VAStatus iris_CreateConfig(VADriverContextP ctx, VAProfile profile,
                           VAEntrypoint entrypoint, VAConfigAttrib *attribs, int n,
                           VAConfigID *config_id)
{
    if (entrypoint != VAEntrypointVLD) return VA_STATUS_ERROR_UNSUPPORTED_ENTRYPOINT;
    if (!profile_supported(profile))   return VA_STATUS_ERROR_UNSUPPORTED_PROFILE;

    uint32_t rt = VA_RT_FORMAT_YUV420;   /* 榛樿 */
    for (i = 0; i < n; i++) {
        if (attribs[i].type == VAConfigAttribRTFormat) rt = attribs[i].value;
    }
    if (rt != VA_RT_FORMAT_YUV420) return VA_STATUS_ERROR_UNSUPPORTED_RT_FORMAT;
    ...
}
```

### 5.5 V4L2 渚х殑 profile/level 淇℃伅锛堝彧璇伙紝涓嶄綔涓轰富渚濇嵁锛?
stateful 瑙ｇ爜鍣ㄧ殑 profile/level 鐢辩爜娴佸喅瀹氾紝鍐呮牳閫氬父鏆撮湶鍙 control锛?
```sh
v4l2-ctl -d /dev/video0 -L | grep -A3 -i 'h264\|hevc\|vp9'
# 鍙兘鐪嬪埌 V4L2_CID_MPEG_VIDEO_H264_PROFILE / _LEVEL锛屽睘鎬т负 inactive/read-only
```

**涓嶈渚濊禆杩欎簺 control 鍋氳兘鍔涘垽鏂?*锛氬畠浠弽鏄犵殑鏄?褰撳墠鐮佹祦"锛屼笉鏄?璁惧鏀寔"銆傝兘鍔涘垽鏂互 `ENUM_FMT` + `TRY_FMT` 涓哄噯銆?
---

## 6. 鐘舵€佹満涓庨槦鍒楃鐞?
### 6.1 瀵硅薄涓庢墍鏈夋潈锛堝厛瀹氳繖涓紝鍚庨潰鎵嶄笉杩斿伐锛?
```
iris_display锛堟瘡涓?VADisplay 涓€涓級
 鈹溾攢鈹€ drm_fd                锛坙ibva 缁欑殑锛屼笉鍏抽棴锛? 鈹溾攢鈹€ caps                  锛圦UERYCAP/ENUM_FMT/TRY_FMT 缂撳瓨锛? 鈹溾攢鈹€ v4l2 璁惧涓庨槦鍒?       锛?dev/video0 fd銆丱UTPUT/CAPTURE buffer銆佸綋鍓?S_FMT锛? 鈹溾攢鈹€ surface 姹?           锛圴ASurfaceID 鈫?iris_surface锛屽惈 dma-buf锛? 鈹斺攢鈹€ 鍏ㄥ眬閿?
iris_config锛堟瘡涓?VAConfigID锛? 鈹斺攢鈹€ profile / entrypoint / rt_format

iris_context锛堟瘡涓?VAContextID锛? 鈹溾攢鈹€ config 鎸囬拡
 鈹溾攢鈹€ picture_width/height  锛坈oded size锛? 鈹溾攢鈹€ AU 绱Н缂撳啿銆佸綋鍓?surface
 鈹溾攢鈹€ 宸叉彁浜ゅ抚琛?           锛坒rame_seq 鈫?surface锛孎IFO + 鍝堝笇锛? 鈹斺攢鈹€ 瑙ｇ爜瀛愮姸鎬?           锛圛DLE / DECODING / ERROR / DRAINING锛?```

**涓轰粈涔?surface 鎸傚湪 display 鑰屼笉鏄?context**锛歏A-API 鐨?`vaCreateSurfaces` 鍙帴鍙?`VADisplay`锛屼笉鎺ュ彈 `VAContextID`锛涜€屼笖 ffmpeg 鍦?`avcodec_flush_buffers()` 鏃跺緢鍙兘閿€姣佸苟閲嶅缓 VA context锛堥渶鍦?ffmpeg 8.0.1 鐨?`libavcodec/vaapi_decode.c` 閲岀‘璁?`vaapi_decode_flush()`锛夛紝浣?hwframes/surfaces 浼氫繚鐣欍€傚鏋?surface 缁戝畾鍦?context 涓婏紝seek 涓€娆″氨鍏ㄥ簾銆?
**涓轰粈涔?V4L2 闃熷垪涔熸寕鍦?display**锛歴urface 鏄?display 绾х殑锛屽鏋?CAPTURE buffer 鐢?context 鎸佹湁锛宑ontext 閿€姣佹椂 buffer 琚噴鏀撅紝surface 灏辨偓绌轰簡銆傛妸闃熷垪鏀?display 绾э紝context 鍙槸"瑙ｇ爜浼氳瘽"锛岄攢姣佹椂鍙仛 `STREAMOFF` + `REQBUFS(0)`锛宻urface 鐨勫唴瀛樹粛鍦紙鎷疯礉妯″紡涓?surface 鑷湁 dma-buf锛屽ぉ鐒跺畨鍏級銆?
> **ADR-1锛堟灦鏋勫喅绛栵級锛歁VP 榛樿璧?鎷疯礉妯″紡"銆?*
> 鐞嗙敱锛氭嫹璐濇ā寮忎笅 surface 鐨勫唴瀛樹笌 V4L2 buffer 瀹屽叏瑙ｈ€︼紝context 鐢熷懡鍛ㄦ湡銆乻urface 鏁伴噺銆乻eek/flush 閮戒笉浼氫簰鐩哥壍杩烇紱浠ｄ环鏄瘡甯т竴娆?NV12 memcpy锛?080p30 鈮?93 MB/s锛屽疄娴嬪唴瀛樺甫瀹?> 20 GB/s锛孋PU 寮€閿€绾?2%~5%锛夈€?> 闆舵嫹璐濓紙`EXPBUF` 鍒悕锛変綔涓?L2 浼樺寲锛屽墠鎻愭槸 M1 楠岃瘉 `EXPBUF` 鍙敤锛屼笖鎺ュ彈"surface 鏁?= CAPTURE buffer 鏁?鐨勭害鏉熴€?
### 6.2 鐘舵€佹満

```
                    vaCreateContext
                          鈹?                          鈻?                    鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  S_FMT/REQBUFS/STREAMON 澶辫触
                    鈹?  INIT    鈹傗攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈻?ERROR
                    鈹斺攢鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹?                          鈹?闃熷垪灏辩华
                          鈻?                    鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?        鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈻衡攤 STREAMING 鈹傗梽鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?        鈹?          鈹斺攢鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹?          鈹?閲嶆柊 S_FMT/REQBUFS
        鈹?                鈹?                鈹?        鈹?  SOURCE_CHANGE 鈹?                鈹?        鈹?                鈻?                鈹?        鈹?          鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?          鈹?        鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?RESOLUTION鈹傗攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?                    鈹? _CHANGE  鈹? 鏂板昂瀵?鈮?surface 灏哄
                    鈹斺攢鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹?                          鈹?鏂板昂瀵?> surface 灏哄
                          鈻?                    鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?                    鈹? ERROR    鈹傦紙瀵瑰鎴风杩斿洖 RESOLUTION_NOT_SUPPORTED锛?                    鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?
   浠绘剰鐘舵€?鈹€鈹€ vaDestroyContext 鈹€鈹€鈻?TERMINATED锛圫TREAMOFF + REQBUFS(0)锛?   STREAMING 鈹€鈹€ EOS/DEC_CMD_STOP 鈹€鈹€鈻?DRAINING锛堢瓑 LAST锛夆攢鈹€鈻?STREAMING/TERMINATED
```

### 6.3 鎻愪氦娴佺▼涓?surface 鍏宠仈锛堟湰椤圭洰鏈€瀹规槗閿欑殑鍦版柟锛?
VA-API 鐨勬ā鍨嬫槸"surface 鏄В鐮佺洰鏍?锛屽鎴风涔嬪悗浼氬**鍚屼竴涓?surface** 璋?`vaSyncSurface` / `vaExportSurfaceHandle`銆傝€?V4L2 鐨?CAPTURE 闃熷垪鏄?鎸夋樉绀哄簭鍑哄抚"鐨勫叕鍏遍槦鍒椼€備袱鑰呬箣闂村繀椤诲缓绔嬫槧灏勩€?
**鍞竴鍙潬鐨勬槧灏勯敭鏄?`v4l2_buffer.timestamp`銆?* 鍋氭硶锛?
1. 姣忎釜 `vaEndPicture` 鍒嗛厤涓€涓崟璋冮€掑鐨?`frame_seq`锛堜粠 0 寮€濮嬶級锛?2. `QBUF` 鏃舵妸 `frame_seq` 鍐欒繘 `b.timestamp`锛坄tv_sec = seq/1e6, tv_usec = seq%1e6`锛夛紱
3. 椹卞姩鑻ヨ皟鐢?`v4l2_m2m_buf_copy_metadata()`锛孋APTURE buffer 鍑洪槦鏃?`timestamp` 灏辨槸瀵瑰簲 OUTPUT buffer 鐨勫€硷紱
4. `DQBUF` 鍚庢寜 timestamp 鍙嶆煡 `frame_seq`锛屽啀鏌?surface锛屾爣璁?ready銆?
**楠岃瘉鏂规硶锛圡1 蹇呭仛锛?*锛?
```sh
grep -rn "v4l2_m2m_buf_copy_metadata" /path/to/linux-sm8450-liuqin/drivers/media/platform/qcom/iris/
# 鏈夎皟鐢?鈫?鏂规鎴愮珛
# 娌℃湁   鈫?闇€瑕佺粰鍐呮牳鎵撲竴琛岃ˉ涓侊紙瑙佷笅锛夛紝鎴栭€€鍥?涓茶鍖?妯″紡锛堟€ц兘宸紝瑙佹湰鑺傚悗鏂囷級
```

**濡傛灉鍐呮牳涓嶅鍒?timestamp**锛屼笁涓€夋嫨锛?
- **锛堟帹鑽愶級缁欏唴鏍?iris 鎵撲竴琛岃ˉ涓?*锛氬湪 capture buffer 瀹屾垚澶勮皟鐢?`v4l2_m2m_buf_copy_metadata(&dst->vb, &src->vb)`銆傝繖鏄?v4l2-mem2mem 鐨勬爣鍑嗙敤娉曪紝鍏朵粬 stateful 椹卞姩锛坴enus锛夐兘杩欎箞鍋氾紝鍚戜笂娓告彁浜や篃鍚堢悊銆?- **涓茶鍖栨ā寮?*锛氫竴娆″彧鍏佽涓€涓?AU 鍦ㄩ€旓紝`QBUF` 鍚庣珛鍒?`DRAIN`锛坄V4L2_DEC_CMD_STOP`锛夊苟绛夋墍鏈?CAPTURE 鍑洪槦锛屽啀鎻愪氦涓嬩竴甯с€傝繖鏍疯緭鍑洪『搴忓畬鍏ㄧ‘瀹氾紙绗?N 娆℃彁浜ゅ搴旂 N 涓嚭闃熷抚锛夛紝浣?*姣忓抚涓€娆?drain锛屾€ц兘浼氭帀鍒颁釜浣嶆暟 fps**锛屽彧鑳界敤浜庡姛鑳介獙璇併€?- **鏀惧純 B 甯?*锛氬彧鏀寔 baseline锛堟棤閲嶆帓锛夛紝姝ゆ椂鎻愪氦搴?= 鏄剧ず搴忥紝FIFO 鏄犲皠鎴愮珛銆侻VP 鐨?baseline 娴嬭瘯鍙互杩欎箞骞诧紝浣嗕笉鑳戒綔涓轰氦浠樻柟妗堛€?
**涓轰粈涔堜笉鑳界敤 FIFO 鐩存帴鏄犲皠**锛欻.264 High 鏅亶鏈?B 甯э紝V4L2 鎸夋樉绀哄簭鍑哄抚锛屾彁浜ゅ簭鏄В鐮佸簭锛屼袱鑰呬笉鍚屻€傜 1 涓嚭闃熺殑甯у彲鑳芥槸绗?3 娆℃彁浜ょ殑 surface銆侳IFO 鏄犲皠浼氭妸鐢婚潰寮犲啝鏉庢埓锛堣〃鐜颁负"鐢婚潰椤哄簭閿欎贡/闂洖"锛夛紝涓旈毦浠ュ畾浣嶃€?
**濡傛灉 timestamp 琚唴鏍歌鐩?*锛堟墦鍗板嚭鏉ュ叏鏄崟璋冩椂閽熷€硷紝鑰屼笉鏄?0,1,2鈥︼級锛岃繕鏈変袱涓閫夐敭锛?
- `timecode`锛歚v4l2_m2m_buf_copy_metadata()` 涔熶細澶嶅埗 `timecode`锛屽墠鎻愭槸 QBUF 鏃剁疆 `V4L2_BUF_FLAG_TIMECODE` 骞跺～ `struct v4l2_timecode`锛堝抚鍙?绉掑彿鏄?8bit 瀛楁锛屽閲忔湁闄愶紝閫傚悎鐭祴璇曚笉閫傚悎闀跨墖锛夛紱
- 鍐呮牳琛ヤ竵锛歚vb2_buffer` 娌℃湁閫氱敤鐨?64 浣嶇敤鎴峰瓧娈靛彲鍊熺敤锛屾墍浠ユ洿鐜板疄鐨勫仛娉曟槸纭繚 iris 璋冪敤 `v4l2_m2m_buf_copy_metadata()`锛岃€屼笉鏄彟鎵惧叧鑱旈敭銆?
### 6.4 EOS / 鎺掔┖锛坄V4L2_DEC_CMD_STOP`銆乣V4L2_BUF_FLAG_LAST`锛?
VA-API 娌℃湁 EOS 璋冪敤銆傚鎴风鐨?缁撴潫"琛ㄧ幇涓猴細

- 涓嶅啀璋冪敤 `vaEndPicture`锛?- seek 鏃惰皟鐢?`avcodec_flush_buffers()` 鈫?鍙兘瑙﹀彂 `vaDestroyContext`锛?- 鍏抽棴鏃?`vaDestroyContext`/`vaTerminate`銆?
鍥犳椹卞姩闇€瑕佹帓绌虹殑鍦烘櫙涓昏鏄?*鍐呴儴娴嬭瘯**鍜?*鍏抽棴鏃剁殑骞插噣鏀跺熬**锛?
```c
/* 鎺掔┖锛氬憡璇夎В鐮佸櫒"杈撳叆鍒版涓烘" */
struct v4l2_decoder_cmd cmd = { .cmd = V4L2_DEC_CMD_STOP };
ioctl(fd, VIDIOC_DECODER_CMD, &cmd);

/* 鐒跺悗涓€鐩?DQBUF CAPTURE锛岀洿鍒版渶鍚庝竴涓?buffer 甯?V4L2_BUF_FLAG_LAST */
while (!last_seen) {
    poll(fd, POLLIN, 1000);
    dqbuf(...);
    if (b.flags & V4L2_BUF_FLAG_LAST) last_seen = true;
}
```

瑕佺偣锛?
- `V4L2_DEC_CMD_STOP` 鍙 stateful decoder 鏈夋晥锛涘鏋滆繑鍥?`EINVAL`锛岃鏄庨┍鍔ㄤ笉鏀寔锛岄€€鍖栦负鐩存帴 `STREAMOFF`锛堜涪寮冨墿浣欏抚锛夈€?- `V4L2_BUF_FLAG_LAST` 鍙細鍑虹幇鍦?CAPTURE 闃熷垪鐨?*鏈€鍚庝竴涓?* buffer 涓婏紱鏀跺埌鍚庝笉瑕佺户缁?`DQBUF`銆?- 鎺掔┖鍚庤嫢瑕佺户缁В鐮侊紝鐢?`V4L2_DEC_CMD_START` 鎭㈠锛堟湁浜涢┍鍔ㄨ姹傞噸鏂?`STREAMON`锛屼互瀹炴祴涓哄噯锛夈€?- **`STREAMOFF` 浼氫涪寮冩墍鏈夊湪閫斿抚**锛屾墍浠?姝ｅ父鍏抽棴"鍜?seek"閮界敤瀹冿紝浣?鎯宠鏈€鍚庡嚑甯?蹇呴』鍏?STOP 鎺掔┖銆?
### 6.5 `SOURCE_CHANGE`锛堝垎杈ㄧ巼鍔ㄦ€佸彉鍖栵級鐨勫畬鏁村鐞嗗簭鍒?
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

    /* 1. 鎶?CAPTURE 闃熷垪閲屾墍鏈?buffer 鏀跺洖锛堟爣璁颁负璺宠繃/閿欒锛?*/
    drain_capture_all(d);

    /* 2. 鍋?CAPTURE 娴?*/
    enum v4l2_buf_type t = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    ioctl(d->fd, VIDIOC_STREAMOFF, &t);

    /* 3. 閲婃斁鏃?buffer */
    struct v4l2_requestbuffers rb = { .count = 0, .type = t, .memory = V4L2_MEMORY_MMAP };
    ioctl(d->fd, VIDIOC_REQBUFS, &rb);

    /* 4. 璁炬柊鏍煎紡 */
    f.fmt.pix_mp.width = nw; f.fmt.pix_mp.height = nh;
    f.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
    f.fmt.pix_mp.num_planes = 1;
    ioctl(d->fd, VIDIOC_S_FMT, &f);

    /* 5. 閲嶆柊鐢宠 + 鏄犲皠/瀵煎嚭 + 寮€娴?*/
    rb.count = d->cap_count;
    ioctl(d->fd, VIDIOC_REQBUFS, &rb);
    setup_capture_buffers(d);
    ioctl(d->fd, VIDIOC_STREAMON, &t);

    d->cur_w = nw; d->cur_h = nh;
    d->state = IRIS_STREAMING;
}
```

缁嗚妭涓庡潙锛?
- **浜嬩欢鍙兘鍦ㄧ涓€甯т箣鍓嶅氨鏉?*锛堢敤浜庡憡鐭ュ垵濮嬪垎杈ㄧ巼锛夈€傛鏃?CAPTURE 闃熷垪鍙兘杩樻病 buffer锛屽鐞嗚骞傜瓑銆?- **蹇呴』鍏?`G_FMT` 鍐?`S_FMT`**锛歚S_FMT` 浼氭妸璇锋眰鍊?灏辫繎鍙栨暣"锛岀洿鎺?`S_FMT(1920x1080)` 鍙兘琚敼鎴?1920脳1088锛沗G_FMT` 鎷垮埌鐨勬墠鏄┍鍔ㄧ湡姝ｈ杈撳嚭鐨勫昂瀵搞€?- **`STREAMOFF(CAPTURE)` 涔嬪悗蹇呴』 `REQBUFS(0)`**锛歷b2 涓嶅厑璁稿湪 buffer 鏈噴鏀炬椂鏀规牸寮忥紙浼?`EBUSY`锛夈€?- **OUTPUT 闃熷垪涓嶈鍋?*锛氬凡鍏ラ槦鐨勭爜娴佽繕鍦ㄨВ鐮侊紝鍋滀簡瑕侀噸鍠傘€傝嫢椹卞姩瑕佹眰涓や釜闃熷垪閮藉仠锛堝疄娴?`EBUSY`锛夛紝鍙兘鏁存潯绠￠亾 reset锛屽苟浠庝笅涓€涓?IDR 閲嶆柊寮€濮嬶紙搂6.6锛夈€?- **VA surface 鏄浐瀹氬昂瀵哥殑**锛氭柊鍒嗚鲸鐜?鈮?鏃?surface 鏃讹紝鍚庣画甯т粛瑙ｅ埌鏃у昂瀵?surface 閲岋紙澶氫綑鍖哄煙涓嶆樉绀猴級锛涙柊鍒嗚鲸鐜囨洿澶ф椂鍙兘鎶ラ敊锛岃瀹㈡埛绔噸寤鸿В鐮佸櫒銆侳irefox 鍦?`RESOLUTION_NOT_SUPPORTED` 鏃朵細閲嶅缓 decoder锛堥渶瀹炴祴纭锛岃 搂7.6锛夈€?- **`V4L2_FMT_FLAG_DYN_RESOLUTION`** 鏄?杩欎釜鏍煎紡鍙兘鍔ㄦ€佸彉鍒嗚鲸鐜?鐨勫０鏄庯紝涓嶄唬琛ㄦ瘡甯ч兘鍙樸€傝闃呬簨浠跺嵆鍙紝涓嶈杞銆?
### 6.6 flush / seek

VA-API 娌℃湁 flush 鍏ュ彛銆傚疄闄呰Е鍙戣矾寰勶細

| 瀹㈡埛绔涓?| 椹卞姩鐪嬪埌鐨?| 椹卞姩瑕佸仛 |
|---|---|---|
| `avcodec_flush_buffers()`锛坒fmpeg seek锛?| 閫氬父鏄?`vaDestroyContext`锛? 涔嬪悗閲嶅缓锛?| `STREAMOFF` 涓や釜闃熷垪 鈫?`REQBUFS(0)` 鈫?娓呯┖鍦ㄩ€旇〃 鈫?涓嬫 `vaCreateContext` 閲嶆柊 `S_FMT`/`REQBUFS`/`STREAMON` |
| `mpv` 鎷栧姩杩涘害鏉?| 鍚屼笂锛坢pv 鍐呴儴 `avcodec_flush_buffers`锛?| 鍚屼笂 |
| Firefox 鍒囨崲 video src | 閿€姣?decoder 鈫?`vaDestroyContext` | 鍚屼笂 |
| 瑙ｇ爜鍑洪敊鍚庣户缁杺 | `vaBeginPicture` 鏃?`state == ERROR` | 鑻ュ綋鍓?AU 鏄?IDR锛屽皾璇?reset 鍚庢仮澶嶏紱鍚﹀垯杩斿洖 `VA_STATUS_ERROR_DECODING_ERROR` |

瀹炵幇寤鸿锛?
- **涓嶈鍏抽棴 `/dev/video0` fd**锛宺eset 鏃跺鐢紱`open/close` 涓€娆＄害鍑犳绉掞紝seek 棰戠箒鏃朵綋鎰熸槑鏄俱€?- reset 鏃舵妸 `frame_seq` 缁х画閫掑锛堜笉瑕佸綊闆讹級锛岄伩鍏嶆棫 timestamp 涓庢柊甯х鎾炪€?- reset 鍚?*寮哄埗瑕佹眰涓嬩竴涓?AU 浠?IDR 寮€濮?*锛氫粠 slice NAL 鐨?`nal_unit_type == 5` 鍒ゆ柇锛圚.264锛夈€傝嫢瀹㈡埛绔粰鐨勬槸闈?IDR锛堜緥濡?seek 鍒颁簡闈炲叧閿抚锛夛紝杩斿洖 `VA_STATUS_ERROR_DECODING_ERROR`锛岃瀹㈡埛绔噸鏂?seek鈥斺€攆fmpeg 榛樿 seek 鍒板叧閿抚锛屾甯镐笉浼氬彂鐢熴€?
### 6.7 澶?slice銆佸甯т笌杈圭晫鍒ゅ畾

| 鎯呭喌 | 澶勭悊 |
|---|---|
| 涓€涓?AU 澶氫釜 slice锛堝父瑙侊級 | 渚濇杩藉姞鍒板悓涓€涓?AU buffer锛宍vaEndPicture` 鏃朵竴娆?`QBUF` |
| 涓€涓?slice 琚媶鎴愬涓?`VASliceDataBufferType`锛坄slice_data_flag` = BEGIN/MIDDLE/END锛?| 鎸?flag 鎷兼帴锛汳VP 鍙敮鎸?`ALL`锛屽叾瀹冭繑鍥?`VA_STATUS_ERROR_FLAG_NOT_SUPPORTED` |
| 鍚屼竴涓?`vaBeginPicture`/`vaEndPicture` 閲屽嚭鐜颁袱涓?`first_mb_in_slice == 0` 鐨?slice | 瑙嗕负涓や釜 AU锛氶亣鍒扮浜屼釜鏃跺厛鎶婂墠涓€涓?AU `QBUF` 鍑哄幓锛堥槻寰℃€у鐞嗭紝姝ｅ父瀹㈡埛绔笉浼氳繖鏍凤級 |
| `vaEndPicture` 鏃?AU 涓虹┖锛圗OS/flush 鐨勭┖璋冪敤锛?| 鐩存帴杩斿洖 `VA_STATUS_SUCCESS`锛屼笉 `QBUF` |
| 涓€甯ц鎷嗘垚澶氭 `vaRenderPicture` | 姝ｅ父锛岀疮绉嵆鍙紱涓嶈鍋囪 `vaRenderPicture` 璋冪敤娆℃暟 |

**濡備綍鍒ゆ柇 `first_mb_in_slice == 0`**锛歴lice NAL 鐨勭 1 瀛楄妭鏄?NAL header锛岀 2 瀛楄妭寮€濮嬫槸 RBSP锛沗first_mb_in_slice` 鏄涓€涓?ue(v)銆傚啓涓€涓?10 琛岀殑 `ue_read()` 灏卞銆傝繖涓В鏋愬悓鏃剁敤浜庢嬁 `pic_parameter_set_id`锛埪?.6.3锛夈€?
### 6.8 閿欒鎭㈠

```c
void iris_fail_pipeline(struct iris_display *d, VAStatus why)
{
    fail_all_inflight(d, why);       /* 鎵€鏈夋湭鍑洪槦 surface 鏍囪 error锛屽敜閱掔瓑寰呰€?*/
    enum v4l2_buf_type t;
    t = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE;  ioctl(d->fd, VIDIOC_STREAMOFF, &t);
    t = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE; ioctl(d->fd, VIDIOC_STREAMOFF, &t);
    d->need_reset = true;            /* 涓嬫 vaCreateContext/vaBeginPicture 鏃堕噸寤?*/
    d->state = IRIS_ERROR;
}
```

瑙﹀彂鏉′欢锛?
- `DQBUF` 杩斿洖 `EIO`锛?- CAPTURE buffer 甯?`V4L2_BUF_FLAG_ERROR`锛?- `poll` 瓒呮椂锛堥粯璁?2 s锛屽彲鐢?`LIBVA_IRIS_TIMEOUT_MS` 瑕嗙洊锛夛紱
- `QBUF` 杩炵画杩斿洖 `EPIPE`/`ENOSPC`銆?
鎭㈠绛栫暐锛?*閲嶇疆鏁存潯绠￠亾**锛屽苟瑕佹眰涓嬩竴涓?AU 鏄?IDR銆備笉瑕佽瘯鍥?璺宠繃鍧忓抚缁х画"鈥斺€攕tateful 瑙ｇ爜鍣ㄧ殑鍙傝€冨抚鐘舵€佸凡缁忚剰浜嗭紝缁х画瑙ｅ彧浼氳繛缁姳灞忋€?
### 6.9 绾跨▼涓庨攣

- libva 瀹㈡埛绔彲鑳戒粠澶氫釜绾跨▼璋冪敤锛圙Streamer 鐨?`vah264dec` 鏈夌嫭绔嬭В鐮佺嚎绋嬶紱Firefox 姣忎釜 decoder 涓€涓嚎绋嬶級銆?- 鐢?*涓€鎶?display 绾т簰鏂ラ攣**淇濇姢 V4L2 fd 涓庨槦鍒楃姸鎬侊紝绠€鍗曞彲闈狅紱`vaRenderPicture` 鍙仛 memcpy锛屾寔閿佹椂闂寸煭锛沗vaEndPicture`/`vaSyncSurface` 鍙兘闃诲锛屾寔閿佺瓑寰呬細瀵艰嚧鍏跺畠绾跨▼鍗′綇鈥斺€?*闃诲绛夊緟瑕佺敤鏉′欢鍙橀噺閲婃斁閿?*锛屾垨鐢?浠呬竴涓嚎绋嬭繘瑙ｇ爜璺緞"鐨勭害瀹氾紙ffmpeg 澶╃劧鍗曠嚎绋嬭В鐮侊級銆?- 涓嶈鍦ㄦ寔閿佹椂璋冪敤 `poll` 鏃犻檺绛夊緟锛涚敤 100 ms 鍒囩墖杞锛岄『渚挎鏌?`destroying` 鏍囧織銆?
### 6.10 瓒呮椂涓庣湅闂ㄧ嫍

| 浣嶇疆 | 榛樿瓒呮椂 | 琛屼负 |
|---|---|---|
| `vaEndPicture` 绛夌┖闂?OUTPUT buffer | 500 ms | 瓒呮椂杩斿洖 `VA_STATUS_ERROR_OPERATION_FAILED` 骞舵爣璁?pipeline error |
| `vaSyncSurface` 绛?CAPTURE | 2000 ms | 鍚屼笂 |
| `vaDestroyContext` 鐨?`STREAMOFF` | 涓嶇瓑寰?| 鐩存帴鍋?|

瓒呮椂鏃堕棿鐢ㄧ幆澧冨彉閲忓彲璋冿紝鏂逛究鍦ㄦ參璁惧/璋冭瘯鏃舵斁瀹斤細

```sh
LIBVA_IRIS_TIMEOUT_MS=5000 vainfo ...
```

---

## 7. Firefox 渚х殑鍏蜂綋瑕佹眰

### 7.1 Firefox 鐨?VA-API 璋冪敤閾?
```
Firefox 鍐呭杩涚▼ / RDD 杩涚▼
 鈹斺攢鈹€ FFmpegVideoDecoder锛坉om/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp锛?      鈹溾攢鈹€ dlopen("libmozavcodec.so")锛堟崋缁戠殑 ffmpeg锛岄潪绯荤粺 ffmpeg锛?      鈹溾攢鈹€ av_hwdevice_ctx_create(AV_HWDEVICE_TYPE_VAAPI)
      鈹?    鈹斺攢鈹€ ffmpeg: libavutil/hwcontext_vaapi.c
      鈹?          鈹溾攢鈹€ open("/dev/dri/renderD128") 鈫?vaGetDisplayDRM()
      鈹?          鈹斺攢鈹€ vaInitialize() 鈫?libva 鍔犺浇 iris_drv_video.so
      鈹溾攢鈹€ avcodec_send_packet / avcodec_receive_frame锛圓V_PIX_FMT_VAAPI锛?      鈹斺攢鈹€ av_hwframe_map(frame, AV_HWFRAME_MAP_DIRECT)
            鈹斺攢鈹€ ffmpeg: vaapi_map_to_drm_esh()
                  鈹斺攢鈹€ vaExportSurfaceHandle(..., VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2, ...)
                        鈹斺攢鈹€ VADRMPRIMESurfaceDescriptor 鈫?AVDRMFrameDescriptor
                              鈹斺攢鈹€ Firefox DMABufSurface锛坓fx/2d/DMABufSurface.cpp锛?                                    鈹斺攢鈹€ EGL_EXT_image_dma_buf_import 鈫?GL 绾圭悊
                                          鈹斺攢鈹€ WebRender 鍚堟垚
```

**鍏抽敭缁撹锛欶irefox 鐢ㄧ殑鏄嚜宸辨崋缁戠殑 ffmpeg锛坄libmozavcodec.so`锛夛紝涓嶆槸绯荤粺 ffmpeg銆?* 杩欐湁涓や釜鍚庢灉锛?
1. 缁欑郴缁?ffmpeg 鎵撹ˉ涓侊紙R2 鐩撮€氾級**涓嶄細褰卞搷 Firefox**锛涜璁?Firefox 璧扮洿閫氾紝蹇呴』閲嶇紪 Firefox 鎴栨浛鎹㈠叾 `libmozavcodec.so`锛堝悗鑰?ABI 蹇呴』瀹屽叏鍖归厤锛岄闄╅珮锛夈€?2. Firefox 鐨?ffmpeg 鐗堟湰/缂栬瘧閫夐」鍐冲畾瀹冩槸鍚︽敮鎸?VA-API銆傚畼鏂?Linux 鏋勫缓鏄惎鐢ㄧ殑锛堝惁鍒?VA-API 浠庢潵涓嶅彲鑳藉伐浣滐級锛屼絾**鏄惁鍚敤 `vaExportSurfaceHandle` 璺緞**瑕佸湪 `about:support` 鎴?`MOZ_LOG` 閲岀‘璁ゃ€?
### 7.2 椹卞姩蹇呴』婊¤冻鐨勬帴鍙ｆ竻鍗曪紙Firefox 瑙嗚锛?
| 瑕佹眰 | 涓嶆弧瓒崇殑鍚庢灉 | 鏈」鐩搴?|
|---|---|---|
| `vaInitialize` 鍦?`/dev/dri/renderD128` 涓婃垚鍔?| Firefox 鐩存帴鍒ゅ畾"鏃?VA-API"锛屽叏绋嬭蒋瑙?| 搂3.2 |
| `vaQueryConfigProfiles` 杩斿洖 `VAProfileH264High`锛堝強 Baseline/Main锛?| ffmpeg 鎵句笉鍒板彲鐢?config锛屽洖閫€杞В | 搂5.3 |
| `vaCreateConfig`/`vaCreateContext` 鎴愬姛 | 鍚屼笂 | 搂3.2/搂3.3 |
| `vaQuerySurfaceAttributes` 杩斿洖鍚堢悊灏哄鑼冨洿 | hwframe 姹犲垵濮嬪寲澶辫触 | 搂4.3 |
| `vaExportSurfaceHandle` + `DRM_PRIME_2` + NV12 涓ゅ眰 | `av_hwframe_map` 澶辫触 鈫?鍥為€€杞В鎴栭粦灞?| 搂4.4 |
| 瀵煎嚭鐨?fd 鑳借 `EGL_EXT_image_dma_buf_import` 瀵煎叆 | 瑙ｇ爜鎴愬姛浣嗘棤娉曚笂灞忥紙榛戝睆/缁垮睆锛?| 搂4.5 |
| `vaSyncSurface` 涓嶆棤闄愰樆濉炪€侀敊璇兘杩斿洖 | 瑙嗛鍗℃銆佹爣绛鹃〉鏃犲搷搴?| 搂6.8/搂6.10 |
| 绾跨▼瀹夊叏锛堝悓涓€ display 澶氱嚎绋嬭皟鐢級 | 闅忔満宕╂簝 | 搂6.9 |

**涓嶉渶瑕?*鐨勶細`vaPutSurface`锛團irefox 涓嶇敤锛夈€乣vaCreateSubpicture`銆乣vaAcquireBufferHandle`锛坒fmpeg 鐨?`hwmap` 璧?export 鑰岄潪 acquire锛夈€?
**鍏充簬 `VADisplayAttribCopy`**锛氬畠灞炰簬 `vaPutSurface` 鐨勬樉绀哄睘鎬э紙鍛婅瘔椹卞姩鑳藉惁鎶?surface 鎷疯礉鍒?drawable锛夛紝Firefox 涓?ffmpeg 鐨?VA-API 瑙ｇ爜璺緞閮戒笉浣跨敤銆傛垜浠殑 `vaQueryDisplayAttributes` 杩斿洖 0 涓睘鎬с€乣vaSetDisplayAttributes` 杩斿洖 `VA_STATUS_SUCCESS`锛堟垨 `VA_STATUS_ERROR_UNIMPLEMENTED`锛夊嵆鍙紝**涓嶉渶瑕佸疄鐜?`VADisplayAttribCopy`**銆傜湡姝ｅ繀椤荤殑鏄?`vaExportSurfaceHandle`锛埪?.4锛夈€?
### 7.3 about:config 涓?about:support

**about:config锛堟寜椤哄簭璇曪級**

| 棣栭€夐」 | 鍊?| 浣滅敤 |
|---|---|---|
| `media.ffmpeg.vaapi.enabled` | `true` | VA-API 鎬诲紑鍏筹紙涓嶅悓鐗堟湰榛樿鍊间笉鍚岋紝鍏堢‘璁わ級 |
| `media.hardware-video-decoding.enabled` | `true` | 纭欢瑙ｇ爜鎬诲紑鍏筹紙榛樿閫氬父涓?true锛?|
| `media.hardware-video-decoding.force-enabled` | `true` | 缁曡繃 blocklist锛堟柊椹卞姩/鏈煡 GPU 甯歌鎷︼級 |
| `gfx.webrender.all` | `true` | 纭繚 WebRender 寮€鍚紙Wayland 榛樿寮€锛?|
| `gfx.webrender.software` | `false` | 杞欢 WebRender 涓?DMABuf 璺緞涓嶅彲鐢?|
| `widget.dmabuf.force-enabled` | `true` | 寮哄埗鍚敤 DMABuf surface锛堥儴鍒嗙幆澧冮粯璁ゅ叧锛?|
| `security.sandbox.content.level` | `0` | **浠呰皟璇?*锛氱粫寮€鍐呭杩涚▼ seccomp锛堣 搂7.4锛夛紝楠岃瘉瀹屽繀椤绘敼鍥?|
| `media.rdd-ffmpeg.enabled` | `false` | **浠呰皟璇?*锛氳 ffmpeg 鍦ㄥ唴瀹硅繘绋嬭窇锛屾柟渚?strace锛涚敓浜т繚鎸侀粯璁?|

> 涓婅〃閲?`media.ffmpeg.vaapi.enabled` 鐨勭‘鍒囬粯璁ゅ€奸殢鐗堟湰鍙樺寲锛岀敤 about:config 鎼滅储 "vaapi" 鐪嬪綋鍓嶅€硷紝涓嶈鍋囪銆?
**about:support 瑕佺湅鐨勪笁澶?*

1. **Graphics** 鈫?`Compositing` 蹇呴』鏄?`WebRender`锛堜笉鏄?`WebRender (Software)`锛夛紱
2. **Media** 鈫?`Hardware Video Decoding` 涓€鏍忓簲鏄剧ず VA-API 鍙敤/宸插惎鐢紙涓嶅悓鐗堟湰鏂囨涓嶅悓锛夛紱
3. **Graphics** 鈫?`Failure Log` / `Decision Log`锛岃嫢鏈?`VA-API` 鐩稿叧澶辫触浼氬湪杩欓噷銆?
**鏃ュ織**

```sh
MOZ_LOG="PlatformDecoderModule:5,FFmpeg:5,DMABuf:5" \
MOZ_LOG_FILE=/tmp/firefox-media.log \
LIBVA_DRIVER_NAME=iris LIBVA_MESSAGING_LEVEL=3 \
firefox
```

鏈熸湜鍦ㄦ棩蹇楅噷鐪嬪埌锛歚VA-API`銆乣vaExportSurfaceHandle`銆乣DMABufSurface` 鍒涘缓鎴愬姛锛涜嫢鐪嬪埌 `VA-API is not supported` 鎴?`Failed to create VA display`锛屽洖鍒?搂7.2 閫愰」鎺掓煡銆?
### 7.4 鍐呭杩涚▼ sandbox锛氭渶瀹规槗琚拷鐣ョ殑鎷﹁矾铏?
Firefox 鐨勫唴瀹硅繘绋嬬敤 seccomp-bpf + broker 鐧藉悕鍗曢檺鍒舵枃浠惰闂€侺inux 鐨勭瓥鐣ュ湪 `security/sandbox/linux/broker/SandboxBrokerPolicyFactory.cpp`锛屽叾涓笌鍥惧舰鐩稿叧鐨勯€氬父鍙湁锛?
```cpp
AddPath("/dev/dri", ...);          /* 鍏佽 VA-API / EGL */
AddPath("/dev/nvidia*", ...);
/* 娌℃湁 /dev/video* */
```

涔熷氨鏄锛?*鍗充娇 VA 椹卞姩鍔犺浇鎴愬姛锛岄┍鍔ㄩ噷鐨?`open("/dev/video0")` 涔熶細琚嫆缁濓紙EACCES/EPERM锛?*銆傝〃鐜版槸 `vaCreateContext` 澶辫触 鈫?Firefox 鍥為€€杞В锛屾棩蹇楅噷鍙湁涓€鍙ュ惈绯婄殑鍒濆鍖栧け璐ャ€?
楠岃瘉鏂规硶锛?
```sh
# 鏂规硶 1锛氱洿鎺ョ湅 strace
strace -f -e trace=openat -o /tmp/ff.strace firefox 2>/dev/null &
grep -n 'video0' /tmp/ff.strace        # 鑻ユ湁 EACCES锛屽氨鏄畠

# 鏂规硶 2锛氫复鏃跺叧娌欑瀵规瘮
MOZ_DISABLE_CONTENT_SANDBOX=1 firefox   # 鎴?security.sandbox.content.level=0
# 鑻ュ叧娌欑鍚?VA-API 鐢熸晥 鈫?纭瘖
```

鍥涙潯搴斿璺嚎锛?
| 璺嚎 | 鍋氭硶 | 浠ｄ环 | 閫傜敤 |
|---|---|---|---|
| **S1 鍏虫矙绠憋紙浠呮祴璇曪級** | `security.sandbox.content.level=0` 鎴?`MOZ_DISABLE_CONTENT_SANDBOX=1` | 瀹夊叏鎬т笅闄嶏紝涓嶈兘浣滀负浜や粯 | L3 楠岃瘉闃舵 |
| **S2 缁?Firefox 鎵撹ˉ涓?* | 鍦?`SandboxBrokerPolicyFactory` 閲岀粰鍐呭杩涚▼/RDD 杩涚▼鍔?`AddPath("/dev/video0", ...)` | 瑕侀噸缂?Firefox锛堟垨缁存姢鍙戣鐗堣ˉ涓侊級锛涗笂娓镐笉涓€瀹氭帴鍙?| 鐢熶骇鍙敤璺嚎涔嬩竴 |
| **S3 鐗规潈 helper** | 椹卞姩涓嶇洿鎺ュ紑 `/dev/video0`锛岃€屾槸杩炰竴涓?systemd 鏈嶅姟锛坲nix socket + `SCM_RIGHTS` 浼?dma-buf fd锛夛紝鐢辨湇鍔″仛 V4L2 闃熷垪鎿嶄綔 | 鏋舵瀯澶嶆潅銆佸涓€娆?IPC锛涗絾瀹屽叏缁曞紑娌欑锛屼笖鍙粰 snap/Flatpak 鐢?| 闀挎湡/澶氭祻瑙堝櫒鏂规 |
| **S4 鎹㈣В鐮佽繘绋?* | 璁?ffmpeg 璺戝湪 RDD 杩涚▼锛坄media.rdd-ffmpeg.enabled=true`锛夛紝妫€鏌?RDD 鐨?sandbox 绛栫暐鏄惁鏀捐 `/dev/video*` | 鍙栧喅浜?RDD 绛栫暐锛涗笉涓€瀹氬彲琛?| 鍏堥獙璇侊紝鎴愭湰浣?|

**寤鸿椤哄簭锛歋1 楠岃瘉 鈫?S4 璇曟帰 鈫?S2 钀藉湴锛坉eb 鐗?Firefox 鑷甫琛ヤ竵锛夆啋 S3 浣滀负澶氭祻瑙堝櫒/鎵撳寘鏂规銆?*

### 7.5 snap 娌欑锛氫负浠€涔?snap 鐗堝熀鏈蛋涓嶉€?
銆愬疄娴嬨€慒irefox 149 鏄?snap 鍖咃紙`firefox 149.0.2-1`锛夈€俿nap 鐗堟湁涓夊眰棰濆闅滅锛?
1. **鎸傝浇鍛藉悕绌洪棿**锛歴nap 鐨?rootfs 鏄?`core24` base + 鍐呭 snap锛坄mesa-2404` 鎻愪緵 `/usr/lib/aarch64-linux-gnu/dri`锛夈€?*瀹夸富鏈?`/usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so` 鍦?snap 鍐呬笉鍙**锛岄櫎闈為噸寤轰竴涓?content snap 鎻愪緵瀹冣€斺€旇€?Firefox 鐨?snap 娌℃湁澹版槑瀵瑰簲鐨?plug锛屾棤娉曡繛鎺ャ€?2. **AppArmor**锛歚snap.firefox.firefox` 鐨?profile 鐢辨帴鍙ｇ敓鎴愩€俙opengl` 鎺ュ彛鍙斁琛?`/dev/dri/**`锛沗/dev/video*` 闇€瑕?`camera` 鎺ュ彛锛堜笖 snap 閲?Firefox 鏄惁 plug 浜?camera 瑕?`snap connections firefox` 纭锛夈€?3. **鐜鍙橀噺**锛歚snap run` 浼氭竻鐞嗛儴鍒嗙幆澧冨彉閲忥紝`LIBVA_DRIVER_NAME`/`LIBVA_DRIVERS_PATH` 鑳藉惁浼犺繘鍘昏瀹炴祴锛涘嵆浣胯兘浼狅紝璺緞涔熷繀椤诲湪 snap 鍛藉悕绌洪棿鍐呭彲瑙侊紙`$SNAP_USER_COMMON` 鍙锛屽彲浠ユ斁椹卞姩锛夈€?
鍙鐨?鎶樿吘璺嚎"锛堜笉鎺ㄨ崘浣滀负浜や粯锛夛細

```sh
# 1. 鎶婇┍鍔ㄦ斁杩?snap 鍙鐨勭敤鎴风洰褰?mkdir -p ~/snap/firefox/common/vaapi
cp /usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so ~/snap/firefox/common/vaapi/

# 2. 浠庣粓绔甫鐜鍙橀噺鍚姩
LIBVA_DRIVER_NAME=iris \
LIBVA_DRIVERS_PATH=$HOME/snap/firefox/common/vaapi \
snap run firefox

# 3. 妫€鏌?/dev/video0 鏄惁鍙
snap run --shell firefox -c 'ls -l /dev/video0 /dev/dri/renderD128'
```

**鎺ㄨ崘璺嚎锛堜笌椤圭洰宸叉湁缁忛獙涓€鑷达級**锛氬畨瑁?Mozilla 瀹樻柟 APT 婧愮殑 **deb 鐗?Firefox**锛岀Щ闄?snap 鐗堛€傜悊鐢憋細

- 瀹夸富鏈烘枃浠剁郴缁熺洿鎺ュ彲瑙侊紝椹卞姩 so 鏀炬爣鍑嗚矾寰勫嵆鍙紱
- 娌℃湁 snap 鐨?AppArmor 灞傦紝鍙墿 Firefox 鑷繁鐨?seccomp锛埪?.4锛屽彲鐢?S2 瑙ｅ喅锛夛紱
- 椤圭洰 `gh/android-mainline-port/experience/ubuntu-userspace.md` 宸茬粡璁板綍杩?snap 搴旂敤 GPU 鐢ㄦ埛鎬佹潵鑷?`mesa-2404`锛屽涓绘満鑷紪 Mesa 杩涗笉浜嗘矙绠?鐨勫悓绫婚棶棰橈紝浠ュ強"鐢?Mozilla deb 鏇挎崲 snap"鐨勫仛娉曘€?
**Flatpak 浣滀负澶囬€?*锛?
```sh
flatpak install flathub org.mozilla.firefox
flatpak override --user --device=all \
  --filesystem=/usr/lib/aarch64-linux-gnu/dri:ro \
  --env=LIBVA_DRIVER_NAME=iris \
  --env=LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri \
  org.mozilla.firefox
```

`--device=all` 鏄负浜嗘斁琛?`/dev/video0`锛堝彧缁?`--device=dri` 涓嶅锛夛紱浠ｄ环鏄矙绠辨洿瀹姐€?
### 7.6 鍒嗚鲸鐜囧彉鍖栦笌 seek 鍦?Firefox 閲岀殑琛ㄧ幇

- **seek**锛欶irefox 璋?`avcodec_flush_buffers()` 鈫?ffmpeg 鍙兘 `vaDestroyContext` 鈫?鎴戜滑鐨勯噸缃矾寰勶紙搂6.6锛夊繀椤诲共鍑€銆傝嫢 seek 鍚庤姳灞?榛戝睆锛屼紭鍏堟煡"鏄惁浠?IDR 閲嶆柊寮€濮?銆?- **鍒嗚鲸鐜囧彉鍖?*锛圔 绔欏垏娓呮櫚搴︺€佸箍鍛婂垏鎹級锛氬鏁版儏鍐垫槸**鏂板缓 decoder**锛堟柊 video element/鏂?hwframes锛夛紝涓嶆槸鍚屼竴鐮佹祦鍐呭彉鍒嗚鲸鐜囷紱浣嗗悓涓€ `MediaSource` 鍐呭垏鎹㈠垎杈ㄧ巼浼氳Е鍙?`SOURCE_CHANGE`銆?  - 鑻ユ柊鍒嗚鲸鐜?鈮?鏃?surface锛氭垜浠户缁В锛孎irefox 鎸夋柊灏哄鏄剧ず锛屽簲姝ｅ父锛?  - 鑻ユ柊鍒嗚鲸鐜?> 鏃?surface锛歚vaSyncSurface` 杩斿洖 `VA_STATUS_ERROR_RESOLUTION_NOT_SUPPORTED`锛孎irefox 搴旇閲嶅缓 decoder锛堥渶瀹炴祴锛涜嫢涓嶉噸寤猴紝琛ㄧ幇涓洪粦灞忥紝闇€瑕佸湪 Firefox 渚х‘璁?fallback 琛屼负锛夈€?- **甯х巼鍙樺寲**锛歏4L2 涓嶅叧蹇冨抚鐜囷紝VA-API 涔熶笉浼犻€掑抚鐜囷紱Firefox 鐢ㄥ鍣ㄦ椂闂存埑鍋?A/V 鍚屾锛屼笉鍙楀奖鍝嶃€?
### 7.7 鏈€灏忓寲楠岃瘉璺緞锛堜笉瑁?Firefox 涔熻兘楠岃瘉娴忚鍣ㄤ晶锛?
鍦ㄥ姩 Firefox 涔嬪墠锛屽厛鐢?`mpv` 楠岃瘉"VA-API + EGL 瀵煎叆"杩欐潯閾捐矾锛屽洜涓哄畠姣?Firefox 濂借皟璇曪細

```sh
mpv --hwdec=vaapi --vo=gpu-next --msg-level=all=v video.mp4
# 鍏虫敞锛歎sing hardware decoding (vaapi)銆乂O: [gpu-next]銆佹湁鏃?vaExportSurfaceHandle 鎶ラ敊
```

`mpv` 閫氫簡锛岃鏄?搂4.4/搂4.5 娌￠棶棰橈紝鍓╀笅鐨勬槸 Firefox 鐨?sandbox/config 闂銆?
---

## 8. 鏋勫缓涓庨儴缃?
### 8.1 璁惧涓婄殑鏋勫缓渚濊禆锛堝厛瑙ｅ喅"娌℃湁缂栬瘧鍣ㄥ拰 vainfo"锛?
銆愬疄娴嬨€戣澶囧綋鍓?*娌℃湁** `gcc`/`meson`/`ninja`/`make`锛宍vainfo` 涔熸病瑁呫€傜綉缁滅害 283 kB/s锛岃渚濊禆瑕佺畻鏃堕棿銆?
```sh
sudo apt update
sudo apt install -y \
  build-essential meson ninja-build pkg-config git \
  libva-dev libdrm-dev libgbm-dev \
  vainfo v4l-utils \
  gstreamer1.0-tools gstreamer1.0-plugins-bad gstreamer1.0-libav \
  python3-pip
```

| 椤?| 浼扮畻 | 璇存槑 |
|---|---|---|
| 涓嬭浇閲?| 绾?120鈥?80 MB | `build-essential` + `libva-dev` + `libdrm-dev` + `vainfo` + `v4l-utils` 鍙婂叾渚濊禆 |
| 鏃堕棿锛?83 kB/s锛?| 绾?8鈥?2 鍒嗛挓 | 鑻ヨ蛋鍥藉唴闀滃儚鍙樉钁楃缉鐭紱鍏?`apt update` 纭婧愰€熷害 |
| 纾佺洏 | < 1 GB | 璁惧鏍瑰垎鍖轰綑閲?52 GB锛屽厖瓒?|

**闀滃儚寤鸿**锛歚/etc/apt/sources.list.d/` 閲屾崲鎴?`mirrors.ustc.edu.cn`/`mirrors.tuna.tsinghua.edu.cn` 鐨?Ubuntu 26.04 arm64 婧愶紙鑻ュ綋鍓嶆簮鎱級銆傝繖鏄?涓€娆℃姇鍏ャ€侀暱鏈熷彈鐩?鐨?5 鍒嗛挓銆?
**楠岃瘉宸ュ叿閾?*锛?
```sh
gcc --version          # 鏈熸湜 15/16.x
meson --version        # 鏈熸湜 >= 1.4
ninja --version
pkg-config --modversion libva     # 鏈熸湜 1.23.x锛圴A-API 鐗堟湰锛屼笉鏄寘鐗堟湰 2.23锛?vainfo --version
```

### 8.2 婧愮爜涓?meson 鏋勫缓

婧愮爜寤鸿鏀?`~/src/libva-iris`锛堣澶囦笂锛夛紝鍚屾椂鎵樼鍒伴」鐩?Git锛坄github.com/yzddmr6/...` 鎴栧唴閮ㄤ粨搴擄級锛?*涓嶈鍙斁鍦ㄨ澶囦笂**銆?
`meson.build`锛堟渶灏忓彲鐢ㄧ増鏈級锛?
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
  gnu_symbol_visibility: 'hidden',        # 鍙鍑?VA_EXPORT 鐨勭鍙?  install: true,
  install_dir: get_option('libdir') / 'dri',
)

# 鍙€夛細鐮佹祦閲嶅缓鍣ㄥ崟娴嬶紙涓嶄緷璧栫‖浠讹級
if get_option('tests')
  test_exe = executable('test_bs', 'test/test_bs.c',
    files('src/bitstream/bs_writer.c', 'src/bitstream/h264_sps.c',
          'src/bitstream/h264_pps.c', 'src/bitstream/h264_au.c'),
    include_directories: include_directories('src'))
  test('bitstream', test_exe)
endif
```

鏋勫缓涓庡畨瑁咃細

```sh
meson setup build --prefix=/usr --libdir=lib/aarch64-linux-gnu -Dtests=true
ninja -C build
sudo meson install -C build
# 瀹夎缁撴灉锛?usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so
```

**涓轰粈涔?`--libdir=lib/aarch64-linux-gnu`**锛欴ebian/Ubuntu 鐨?multiarch 璺緞锛沴ibva 缂栬瘧鏃跺啓姝荤殑椹卞姩鎼滅储璺緞灏辨槸 `/usr/lib/aarch64-linux-gnu/dri`銆傜敤榛樿 `lib` 浼氳鍒?`/usr/lib/dri`锛宭ibva 鎵句笉鍒般€?
**绗﹀彿妫€鏌?*锛?
```sh
nm -D /usr/lib/aarch64-linux-gnu/dri/iris_drv_video.so | grep -i vadriverinit
# 鏈熸湜锛?000000000001234 T __vaDriverInit_1_0
```

### 8.3 瀹夎鍚庣殑鍔犺浇閰嶇疆

libva 鏌ユ壘椹卞姩鐨勯『搴忥紙`va/va.c` 鐨?`va_open_driver_*`锛夛細

1. 鐜鍙橀噺 `LIBVA_DRIVER_NAME` 鎸囧畾鐨勫悕瀛?鈫?`${LIBVA_DRIVERS_PATH:-/usr/lib/aarch64-linux-gnu/dri}/${name}_drv_video.so`锛?2. 鍚﹀垯鐢?DRM 椹卞姩鍚嶏紙`drmGetVersion(fd)->name`锛夆€斺€旀湰鏈烘槸 `msm` 鈫?鎵?`msm_drv_video.so`锛?3. 閮芥病鏈?鈫?`vaInitialize` 澶辫触銆?
> 杩欐槸 libva 2.x 鐨勮嚜鍔ㄦ帰娴嬮€昏緫銆傝嫢浣犵殑 libva 鐗堟湰琛屼负涓嶅悓锛岀敤 `strace -f -e trace=openat vainfo ... 2>&1 | grep dri` 鐪嬪畠瀹為檯鎵剧殑鏄摢涓枃浠跺悕锛屽啀鍐冲畾瑕佷笉瑕佸缓绗﹀彿閾炬帴銆?
鍥犳鏈?*涓ょ**閮ㄧ讲鏂瑰紡锛?
**鏂瑰紡 A锛堟樉寮忥紝鎺ㄨ崘寮€鍙戞湡锛?*

```sh
export LIBVA_DRIVER_NAME=iris
export LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri
vainfo --display drm --device /dev/dri/renderD128
```

**鏂瑰紡 B锛堥殣寮忥紝鎺ㄨ崘浜や粯鏈燂級**

```sh
# 璁?涓嶈鐜鍙橀噺"鐨勫簲鐢ㄤ篃鑳界敤锛氬埄鐢ㄩ┍鍔ㄥ悕鍥為€€
sudo ln -sf iris_drv_video.so /usr/lib/aarch64-linux-gnu/dri/msm_drv_video.so
```

杩欐牱 Firefox锛堜粠妗岄潰鍥炬爣鍚姩銆佹嬁涓嶅埌鑷畾涔夌幆澧冨彉閲忔椂锛変篃鑳介€氳繃 `msm` 鍥為€€鍔犺浇鍒版垜浠殑椹卞姩銆?*杩欐槸 snap/Flatpak 涔嬪锛岃"闆堕厤缃?鐢熸晥鐨勫叧閿妧宸с€?*

> 娉ㄦ剰锛歚msm_drv_video.so` 杩欎釜鍚嶅瓧涓嶈鍜?Mesa 鐨?`msm_dri.so` 娣锋穯锛屽悗鑰呮槸 DRI/GL 椹卞姩锛宍libva` 涓嶄細鍔犺浇瀹冦€?
**鐢ㄦ埛鏉冮檺**锛氬綋鍓嶇敤鎴?`yumo` 宸插湪 `render` 缁勶紙銆愬疄娴嬨€戯級锛岃兘鎵撳紑 `/dev/dri/renderD128`銆俙/dev/video0` 鐨勬潈闄愬鏋滄槸 `root:video 0660`锛岃繕闇€瑕?`video` 缁勶紙宸插湪锛夈€?
### 8.4 楠岃瘉鍛戒护锛堜粠搴曞眰鍒颁笂灞傦級

```sh
# 0. 璁惧涓庢牸寮?v4l2-ctl -d /dev/video0 --all
v4l2-ctl -d /dev/video0 --list-formats-out
v4l2-ctl -d /dev/video0 --list-formats

# 1. VA 椹卞姩鍔犺浇
LIBVA_DRIVER_NAME=iris LIBVA_MESSAGING_LEVEL=3 \
  vainfo --display drm --device /dev/dri/renderD128

# 2. ffmpeg 纭В锛堜笉鍑哄浘锛屽彧楠岃瘉瑙ｇ爜锛?ffmpeg -v verbose -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i test_1080p_h264_high.mp4 -f null -

# 3. ffmpeg 瀵煎嚭 NV12 姣斿
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i test.mp4 -f rawvideo -pix_fmt nv12 /tmp/va.yuv
ffmpeg -v error -i test.mp4 -f rawvideo -pix_fmt nv12 /tmp/sw.yuv
cmp /tmp/va.yuv /tmp/sw.yuv && echo "BIT-EXACT"

# 4. 闆舵嫹璐濊矾寰勶紙涓嶄笅杞藉埌 CPU锛?ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi -i test.mp4 -f null -

# 5. mpv 鍑虹敾闈紙楠岃瘉 EGL 瀵煎叆锛?mpv --hwdec=vaapi --vo=gpu-next --msg-level=all=v test.mp4
```

### 8.5 浜ゅ弶缂栬瘧锛堝彲閫夛紝鐢ㄤ簬 CI/蹇€熻凯浠ｏ級

璁惧 8 鏍哥紪璇戜竴涓?< 10k 琛岀殑灏忛」鐩彧闇€鍗佸嚑绉掞紝**鏃ュ父寮€鍙戠敤鍘熺敓缂栬瘧鏈€绠€鍗?*銆傚彧鏈夊湪"璁惧涓嶅湪鎵嬭竟"鎴?CI 鍑哄寘"鏃舵墠闇€瑕佷氦鍙夌紪璇戯細

```sh
# 瀹夸富鏈猴紙x86_64 Ubuntu锛?sudo apt install gcc-aarch64-linux-gnu pkg-config
# 闇€瑕佷竴涓?aarch64 sysroot锛堝彲浠庤澶?rsync /usr/include + /usr/lib锛屾垨鐢?debootstrap锛?cat > aarch64-cross.ini <<'EOF'
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

鍧戯細`pkg-config` 榛樿浼氭壘瀹夸富鏈?x86_64 鐨?`libva.pc`锛屽繀椤昏缃?`PKG_CONFIG_SYSROOT_DIR`/`PKG_CONFIG_LIBDIR`锛屽惁鍒欓摼鎺ュ埌閿欒鏋舵瀯銆?*棣栨浜ゅ弶缂栬瘧鑷冲皯棰勭暀鍗婂ぉ銆?*

### 8.6 鎵撳寘鎴?deb锛堜氦浠樺舰鎬侊級

鏈€灏?`debian/` 缁撴瀯锛堢敤 `dpkg-buildpackage` 鎴?`debianize` 鐢熸垚锛夛細

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

鎴栬€呭厛鐢?`checkinstall -D` 蹇€熷嚭鍖咃紝楠岃瘉瀹夎/鍗歌浇娴佺▼锛屽啀琛ユ寮忔墦鍖呫€?
### 8.7 璋冭瘯鐜鍙橀噺姹囨€?
| 鍙橀噺 | 浣滅敤 |
|---|---|
| `LIBVA_DRIVER_NAME=iris` | 鎸囧畾椹卞姩 |
| `LIBVA_DRIVERS_PATH=/usr/lib/aarch64-linux-gnu/dri` | 鎸囧畾鎼滅储鐩綍 |
| `LIBVA_MESSAGING_LEVEL=3` | libva 鏃ュ織锛?鈥?锛? 鏈€璇︾粏锛?|
| `LIBVA_TRACE=/tmp/va.trace` | libva 璋冪敤璺熻釜锛堥渶 libva 缂栬瘧鏃跺惎鐢?trace锛?|
| `LIBVA_IRIS_DEBUG=1` | 鏈」鐩嚜瀹氫箟锛氭墦鍗版瘡娆?ioctl/缁勫寘澶у皬 |
| `LIBVA_IRIS_TIMEOUT_MS=5000` | 鏈」鐩嚜瀹氫箟锛氭斁瀹藉悓姝ヨ秴鏃?|
| `MOZ_LOG=PlatformDecoderModule:5,FFmpeg:5` | Firefox 濯掍綋鏃ュ織 |
| `MOZ_DISABLE_CONTENT_SANDBOX=1` | 涓存椂鍏?Firefox 鍐呭娌欑锛堜粎娴嬭瘯锛?|

### 8.8 閮ㄧ讲妫€鏌ユ竻鍗?
- [ ] `iris_drv_video.so` 鍦?`/usr/lib/aarch64-linux-gnu/dri/`锛屾潈闄?`0644`
- [ ] `ldd` 鏃?`not found`锛坄libva`銆乣libdrm`銆乣libgbm`銆乣libc`锛?- [ ] `nm -D` 鑳界湅鍒?`__vaDriverInit_1_0`
- [ ] `vainfo` 鑳藉湪 render node 涓婂垪鍑?H.264 profile
- [ ] 褰撳墠鐢ㄦ埛鍦?`render`銆乣video` 缁?- [ ] 鑻ヤ氦浠?闆堕厤缃?锛宍msm_drv_video.so` 绗﹀彿閾炬帴瀛樺湪
- [ ] `apt remove`/`meson uninstall` 鑳藉共鍑€鍗歌浇锛屼笉鐣?`.so` 娈嬮

---

## 9. 娴嬭瘯璁″垝

### 9.1 娴嬭瘯鍒嗗眰

| 灞?| 瀵硅薄 | 杩愯鐜 | 棰戠巼 | 閫氳繃鏍囧噯 |
|---|---|---|---|---|
| 鍗曞厓 | 鐮佹祦閲嶅缓鍣紙SPS/PPS/EPB/ue锛?| 浠绘剰 Linux锛坸86 涔熻锛?| 姣忔鎻愪氦 | 鍚堟垚鐮佹祦鑳借 ffmpeg 杞В锛屽抚鏁?鍐呭涓庡弬鑰冧竴鑷?|
| 缁勪欢 | V4L2 绠￠亾锛堜笉鍚?VA锛?| 璁惧 | 姣忔棩 | `tools/v4l2-probe` 鑳借В 60 甯?NV12锛宮d5 绋冲畾 |
| 闆嗘垚 | VA 椹卞姩 + ffmpeg | 璁惧 | 姣忔棩 | `ffmpeg -hwaccel vaapi` 閫氳繃锛孭SNR 杈炬爣 |
| 绔埌绔?| Firefox + B 绔?鏈湴鏂囦欢 | 璁惧 | 姣忛樁娈?| CPU 闄嶅箙杈炬爣銆佺敾闈㈡纭€佸彲 seek |
| 闀跨ǔ | 2 灏忔椂寰幆 | 璁惧 | 姣忓懆/鍙戠増鍓?| RSS 澧為暱 < 5%锛屾棤 kernel oops锛屾棤 fd 娉勬紡 |

### 9.2 鐮佹祦閲嶅缓鍣ㄥ崟娴嬶紙涓嶉渶瑕佺‖浠讹紝浼樺厛鍋氾級

鎬濊矾锛?*鐢?ffmpeg 鐨勮蒋浠?H.264 瑙ｇ爜鍣ㄥ綋"璇硶瑁佸垽"**銆?
```
娴嬭瘯绱犳潗锛坸264 鐢熸垚锛屽凡鐭?SPS/PPS锛?   鈹?   鈹溾攢 鐢?ffmpeg -bsf:v trace_headers 鎴?h264_metadata 鎻愬彇鍘熷 SPS/PPS/slice
   鈹?   鈹溾攢 鍙妸 slice NAL 鍠傜粰閲嶅缓鍣紙妯℃嫙 VA-API 瀹㈡埛绔級
   鈹?    鈹斺攢 閲嶅缓鍣ㄨ緭鍑?SPS + PPS + slice 鐨勫畬鏁?AU
   鈹?   鈹溾攢 鎷兼垚 synth.h264
   鈹?   鈹斺攢 ffmpeg -f h264 -i synth.h264 -f null -
         鈹溾攢 閫€鍑虹爜 0 涓旀棤 "error while decoding" 鈫?璇硶閫氳繃
         鈹斺攢 瑙ｇ爜甯х殑 md5 涓庡師濮嬫枃浠惰蒋瑙ｄ竴鑷?鈫?璇箟閫氳繃
```

瀹炵幇鎶€宸э細

- `ffmpeg -i in.mp4 -c copy -bsf:v h264_mp4toannexb -f h264 out.h264` 寰楀埌 Annex B锛?- 鍐欎竴涓?`test/split_au.c`锛氭寜 AUD/`first_mb_in_slice==0` 鍒?AU锛屾妸 SPS/PPS 涓㈡帀锛屽彧鐣?slice锛?- 鍗曟祴閲屽姣忎釜 AU 璋冮噸寤哄櫒锛屽啀鎷煎洖鍘伙紱
- 鐢?`-f md5` 鎴栭€愬抚 `framemd5` 姣斿锛?
```sh
ffmpeg -f h264 -i synth.h264 -f framemd5 - | tail -n +5 > synth.md5
ffmpeg -f h264 -i orig.h264  -f framemd5 - | tail -n +5 > orig.md5
diff synth.md5 orig.md5
```

杩欎釜娴嬭瘯鑳借鐩?80% 鐨勭爜娴侀噸寤?bug锛堝瓧娈靛啓閿欍€乽e 缂栫爜閿欍€丒PB 婕忔彃銆乻lice 椤哄簭閿欙級锛岃€屼笖绉掔骇鍙嶉銆?
### 9.3 娴嬭瘯绱犳潗鐭╅樀

鐢?ffmpeg 鐜板満鐢熸垚锛堜綋绉皬銆佸彲鎺э級锛屽啀鍔?2~3 涓湡瀹炵礌鏉愩€?
```sh
# 閫氱敤鐗囨簮锛歵estsrc2 鏈夎繍鍔ㄣ€佹湁缁嗚妭锛屾瘮 testsrc 鏇撮€傚悎鏌ヨ姳灞?GEN="-f lavfi -i testsrc2=size=%s:rate=30 -t 8 -pix_fmt yuv420p"

# H.264 Baseline锛圕AVLC銆佹棤 B 甯э級
ffmpeg $GEN -c:v libx264 -profile:v baseline -level 3.1 -g 30 \
       -x264-params "cabac=0:bframes=0:ref=1:8x8dct=0" baseline_720p.mp4

# H.264 Main锛圕ABAC銆丅 甯э級
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 8 -pix_fmt yuv420p \
       -c:v libx264 -profile:v main -bf 2 -g 60 -b:v 8M main_1080p.mp4

# H.264 High锛?x8銆佸鍙傝€冦€丅 甯э級
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

# 10bit锛堥鏈燂細椹卞姩杩斿洖涓嶆敮鎸侊紝涓嶅穿锛?ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -t 4 -pix_fmt yuv420p10le \
       -c:v libx265 -x265-params "bframes=3" -b:v 8M hevc_10bit.mp4
```

| 缁村害 | 鍙栧€?| 鐢ㄩ€?|
|---|---|---|
| Codec | H.264 / HEVC / VP9 | 鍒嗛樁娈垫敮鎸?|
| Profile | Baseline / Main / High / High10 / HEVC Main / Main10 / VP9 P0 / P2 | profile 鐭╅樀锛埪?.1锛?|
| 鍒嗚鲸鐜?| 720p / 1080p / 4K | 灏哄涓婇檺銆佸甫瀹?|
| 甯х巼 | 30 / 60 | 闃熷垪娣卞害銆佷涪甯?|
| 鐮佺巼 | 浣庯紙2M锛? 楂橈紙40M锛?| `sizeimage` 鏄惁澶?|
| 缁撴瀯 | 鍗?slice / 澶?slice锛坄-x264-params slices=4`锛? B 甯?/ 鏃?B 甯?| 缁勫寘涓庨噸鎺?|
| 鐗规畩 | 鍒嗚鲸鐜囧彉鍖栵紙涓ゆ TS 鎷兼帴锛? 10bit / 鍦虹紪鐮侊紙棰勬湡鎷掔粷锛?| 寮傚父璺緞 |

**鐪熷疄绱犳潗**锛堟瘮鍚堟垚绱犳潗鏇磋兘鏆撮湶闂锛夛細B 绔?1080p H.264锛堢敤 `yt-dlp` 鎶撳叕寮€瑙嗛锛屾敞鎰忕増鏉冧笌绔欑偣鏉℃锛夈€佹墜鏈烘媿鎽勭殑 H.264 瑙嗛銆乣ffmpeg` 杞爜鐨勮€佺數褰憋紙澶?B 甯?澶氬弬鑰冿級銆?
### 9.4 姝ｇ‘鎬ф瘮瀵癸細md5 涓?PSNR

**md5锛堟渶蹇紝鐢ㄤ簬鍥炲綊锛?*

```sh
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f rawvideo -pix_fmt nv12 - | md5sum
ffmpeg -v error -i in.mp4 -f rawvideo -pix_fmt nv12 - | md5sum
```

娉ㄦ剰锛歮d5 涓€鑷寸殑鍓嶆彁鏄袱鏉¤矾寰勭殑 NV12 杞崲鏂瑰紡涓€鑷淬€俈A-API 杈撳嚭鐨勬槸瑙ｇ爜鍣ㄥ師鐢?NV12锛岃蒋瑙ｈ緭鍑?`yuv420p` 鍐嶈浆 NV12 鍙兘鍥犺壊搴︽彃鍊?鍙栨暣涓嶅悓鑰屼笉涓€鑷淬€?*md5 涓嶄竴鑷翠笉绛変簬椹卞姩閿?*锛岀敤 PSNR 澶嶆牳銆?
**PSNR锛堝垽瀹氭爣鍑嗭級**

```sh
# 鍒嗗埆瀵煎嚭
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f rawvideo -pix_fmt nv12 -s 1920x1080 va.yuv
ffmpeg -v error -i in.mp4 -f rawvideo -pix_fmt nv12 -s 1920x1080 sw.yuv

# 閫愬抚 PSNR
ffmpeg -f rawvideo -pix_fmt nv12 -s 1920x1080 -i va.yuv \
       -f rawvideo -pix_fmt nv12 -s 1920x1080 -i sw.yuv \
       -lavfi "[0:v][1:v]psnr=stats_file=/tmp/psnr.log" -f null -
```

| 缁撴灉 | 鍒ゅ畾 |
|---|---|
| `psnr_avg: inf` | 瀹屽叏涓€鑷达紙鐞嗘兂锛?|
| `psnr_avg > 45 dB` | 鍙帴鍙楋紙宸紓鏉ヨ嚜鑹插害涓婇噰鏍?鍙栨暣锛?|
| 40鈥?5 dB | 鍙枒锛氭鏌ユ槸鍚︽湁杞诲井閿欏抚/鑹插害閿欎綅 |
| < 40 dB 鎴栦釜鍒抚绐侀檷 | 鏈?bug锛堝弬鑰冨抚绠＄悊/瑁佸壀/瀛楁閿欒锛?|

**棣栧抚鐩**锛?
```sh
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -frames:v 1 -vf "hwdownload,format=nv12" /tmp/first.png
```

閲嶇偣鐪嬶細搴曢儴 8 琛岋紙1088 瑁佸壀闂锛夈€佽壊搴︽槸鍚﹂敊浣嶃€佹槸鍚︽湁缁胯壊/绱壊鍧椼€?
### 9.5 CPU 涓庡姛鑰楁祴閲?
**CPU锛?0 绉掑钩鍧囷紝鍙鐜帮級**

鏈€鐪佷簨鐨勬柟寮忔槸鐢?`sysstat` 鐨?`pidstat`锛?
```sh
sudo apt install sysstat
pidstat -h -p ALL 1 60 > /tmp/pidstat.log
# 杩囨护 Firefox 鐩稿叧杩涚▼锛屾妸 %CPU 姹傚拰鍚庡彇骞冲潎
grep -E 'firefox|Web Content|RDD Process|Isolated Web Content' /tmp/pidstat.log \
  | awk '{s+=$8; n++} END {printf "avg sum CPU: %.1f%%\n", s/n}'
```

锛坄pidstat` 鐨勫垪鍙烽殢鐗堟湰鍙樺寲锛屽厛 `head -3` 纭 `%CPU` 鍦ㄧ鍑犲垪銆傦級

**瀵规瘮鏂规硶**锛?
1. 鍚屼竴瑙嗛銆佸悓涓€绐楀彛澶у皬銆佸悓涓€鍒嗚鲸鐜囷紝鎾斁 60 s锛?2. 杞В鍩虹嚎锛歚media.ffmpeg.vaapi.enabled=false`锛?3. 纭В锛歚media.ffmpeg.vaapi.enabled=true`锛?4. 鍚勬祴 3 娆″彇涓綅鏁帮紱
5. 鍚屾椂璁板綍 `gnome-shell`/`Xwayland` 鐨?CPU锛岄伩鍏嶆妸鍚堟垚寮€閿€绠楀埌瑙ｇ爜澶翠笂銆?
**鍔熻€楋紙鏈満鍙锛屽洜涓鸿兘璇诲埌鐢垫睜锛?*锛?
```sh
while true; do
  i=$(cat /sys/class/power_supply/battery/current_now)
  v=$(cat /sys/class/power_supply/battery/voltage_now)
  echo "$(date +%s) $((i*v/1000000000000))"   # 鍗曚綅 W锛坈urrent_now 鍙兘涓鸿礋锛?  sleep 1
done
```

瀵规瘮杞В/纭В鍚?5 鍒嗛挓鐨勫钩鍧囧姛鐜囥€傛敞鎰忓厖鐢电姸鎬佷笅 `current_now` 鍙兘涓烘锛堝厖鍏ワ級锛岃鎷旀帀鐢垫簮娴嬨€?
### 9.6 鍐呭瓨涓庤祫婧愭硠婕?
璁惧涓婂ぇ姒傜巼娌℃湁 `valgrind`锛堝嵆浣挎湁锛孉RM64 涓婅窇 Firefox 涔熶笉鐜板疄锛夈€傛浛浠ｆ柟妗堬細

| 灞傞潰 | 鏂规硶 | 鍒ゆ嵁 |
|---|---|---|
| 鐢ㄦ埛鎬佸爢 | `heaptrack`锛坄apt install heaptrack`锛夎窇 `test/va-dump` 寰幆 1000 娆?| 宄板€间笉闅忓惊鐜鏁扮嚎鎬у闀?|
| 鐢ㄦ埛鎬侀敊璇?| ASan 鏋勫缓锛歚meson setup build-asan -Db_sanitize=address,undefined`锛岃窇鍗曟祴涓?`va-dump` | 鏃?leak/UAF 鎶ュ憡 |
| 杩涚▼ RSS | `grep Pss /proc/<pid>/smaps_rollup` 姣?10 s 閲囨牱锛? 灏忔椂 | 澧為暱 < 5%锛屼笖瓒嬩簬骞崇ǔ |
| fd 娉勬紡 | 缁熻 `/proc/<pid>/fd` 鐨勬潯鐩暟锛屾瘡 10 s 閲囨牱 | 涓嶆寔缁闀匡紙閲嶇偣鏌?dma-buf fd锛?|
| 鍐呮牳 dma-buf | `sudo cat /sys/kernel/debug/dma_buf/bufinfo`锛堣嫢瀛樺湪锛?| buffer 鏁伴噺绋冲畾锛屾棤鎸佺画澧為暱 |
| 鍐呮牳鏃ュ織 | `dmesg` 杈撳嚭閲屾悳 `vb2`/`iris`/`dma_buf`/`iommu` 鍏抽敭瀛?| 鏃?`WARNING`/`BUG`/`IOMMU fault` |
| 鍐呮牳娉勬紡 | `/sys/kernel/debug/kmemleak`锛堣嫢寮€鍚級 | 鏃犳柊澧炶褰?|

**fd 娉勬紡鏄渶甯歌鐨?*锛氭瘡娆?`vaExportSurfaceHandle` 閮?`dup` 涓€涓?fd锛孎irefox 鍏抽棴 surface 鏃跺繀椤婚噴鏀俱€傚缓璁湪椹卞姩閲岀粰 fd 鍔犲紩鐢ㄨ鏁帮紝骞跺湪 `vaDestroySurfaces` 鏃舵柇瑷€ `refcount == 1`銆?
### 9.7 鍥炲綊娴嬭瘯娓呭崟锛堟瘡娆″彂鐗堣窇锛?
- [ ] `vainfo` 杈撳嚭涓庝笂涓€鐗堜竴鑷达紙profile 鍒楄〃銆乨river version锛?- [ ] 9.3 鐭╅樀涓墍鏈?搴旀敮鎸?鐨勭礌鏉愶細瑙ｇ爜甯ф暟姝ｇ‘銆丳SNR 鈮?45 dB
- [ ] 鎵€鏈?搴旀嫆缁?鐨勭礌鏉愶紙10bit/AV1/鍦虹紪鐮侊級锛氳繑鍥炴槑纭敊璇爜锛岃繘绋嬩笉宕?- [ ] seek 20 娆★紙闅忔満浣嶇疆锛夛細鏃犺姳灞忋€佹棤鍗℃銆佹棤 fd 澧為暱
- [ ] 鏆傚仠/鎭㈠ 10 娆★細鏃犺姳灞?- [ ] 绐楀彛浠?1080p 缂╁埌 480p 鍐嶆斁澶э細鏃犺姳灞忥紙杩欐槸鍚堟垚闂锛屼絾瑕佺‘璁や笉鏄В鐮侀棶棰橈級
- [ ] 涓や釜鏍囩椤靛悓鏃舵挱 1080p锛欳PU 鍙帴鍙楋紝鏃犱簰鐩稿崱姝?- [ ] 鎸傝捣/鍞ら啋涓€娆★紙鏈満鏀寔 `rtcwake`锛夛細鎭㈠鍚庤兘缁х画鎾?- [ ] 2 灏忔椂寰幆鎾斁锛歊SS/fd 绋冲畾
- [ ] `dmesg` 鏃?oops/WARNING

### 9.8 鎬ц兘鍩哄噯锛堣褰曞湪 `BENCH.md`锛?
| 鍦烘櫙 | 鎸囨爣 | 杞В | 纭В | 鎻愬崌 |
|---|---|---|---|---|
| 1080p30 H.264 High锛宮pv | CPU% | | | |
| 1080p30 H.264 High锛孎irefox | CPU% | 140鈥?70% | | |
| 4K30 H.264 High锛宮pv | CPU% | | | |
| 1080p30锛屾暣鏈哄姛鑰?| W | | | |
| seek 鍒?60 s 鐨勮€楁椂 | ms | | | |
| 棣栧抚鏃堕棿 | ms | | | |

---

## 10. 閲岀▼纰戜笌宸ヤ綔閲忎及绠?
浼扮畻鍓嶆彁锛? 鍚嶇啛鎮?Linux 鍥惧舰/濯掍綋鏍堢殑宸ョ▼甯堬紝鍏ㄨ亴锛涜澶囧湪鎵嬶紱缃戠粶鍙敤锛?83 kB/s锛夛紱涓嶅惈"绛変笂娓镐慨 bug"鐨勬椂闂淬€?
| 闃舵 | 浜や粯鍐呭 | 楠屾敹 | 浜哄ぉ锛堜箰瑙?鎮茶锛?| 渚濊禆 | 涓昏椋庨櫓 |
|---|---|---|---|---|---|
| **M0 鍏堜緥瀹¤ + 浜嬪疄鏍稿** | 璇诲畬 `mxsrc/radxa libva-v4l2`銆丏ebian `qcom-va-driver`锛涙牳瀵?ffmpeg `start_frame` 鏄惁缁欐暣鍖咃紱`grep` 鍐呮牳 `v4l2_m2m_buf_copy_metadata`锛沗EXPBUF` 鎺㈤拡 | 鍐崇瓥锛歠ork / 鏂板啓 / 鐩存帴鐢?| 2 / 4 | 缃戠粶 | 缁撹鍙兘鎺ㄧ炕璁″垝 |
| **M1 椹卞姩楠ㄦ灦** | `iris_drv_video.so` 鑳藉姞杞斤紝`vainfo` 鍒楀嚭 H.264 Baseline/Main/High锛沗vaCreateConfig`/`vaCreateContext` 鑳藉紑 `/dev/video0` 骞?`S_FMT`/`REQBUFS`/`STREAMON` | L0 閫氳繃 | 3 / 5 | M0 | libva ABI 缁嗚妭銆乮ris `S_FMT` 闄愬埗 |
| **M2 鍗曞抚 H.264** | 鐮佹祦閲嶅缓鍣紙SPS/PPS/AUD锛? 涓€娆?`QBUF` + `DQBUF` + `vaDeriveImage` 瀵煎嚭 NV12锛沚aseline 720p 棣栧抚 PSNR > 45 dB | `ffmpeg -hwaccel vaapi -frames:v 1` 鍑?PNG锛岀敾闈㈡纭?| 5 / 9 | M1 | 閲嶅缓鍣ㄥ瓧娈甸敊銆佽鍓?|
| **M3 杩炵画鎾斁 + seek** | 澶氬抚寰幆銆丅 甯ч噸鎺掞紙timestamp 鏄犲皠锛夈€乣vaSyncSurface`銆乫lush/seek銆佸 slice | 1080p High 鍏ㄧ墖 PSNR 鈮?45 dB锛泂eek 20 娆℃甯?| 4 / 8 | M2 | 鏃堕棿鎴充笉澶嶅埗銆乻eek 鍚庨潪 IDR |
| **M4 闆舵嫹璐濆鍑?* | `vaExportSurfaceHandle` + dma-buf + EGL 瀵煎叆锛堝厛鐢?`mpv --hwdec=vaapi` 楠岃瘉锛?| L2 閫氳繃锛?080p30 `mpv` 鏃犱涪甯?| 3 / 6 | M3 | `EXPBUF` 涓嶆敮鎸併€乵odifier/stride |
| **M5 Firefox 闆嗘垚锛堣皟璇曟€侊級** | deb 鐗?Firefox + `security.sandbox.content.level=0` + `media.ffmpeg.vaapi.enabled=true`锛孊 绔?1080p 璧扮‖瑙?| L3 閫氳繃锛圕PU 鈮?60%锛?| 3 / 6 | M4 | Firefox 鐗堟湰琛屼负銆丷DD 杩涚▼ |
| **M6 娌欑鐢熶骇鍖?+ 鎵撳寘** | S2锛堣ˉ涓侊級鎴?S3锛坔elper锛夎惤鍦帮紱deb 鍖咃紱`msm_drv_video.so` 鍏煎閾炬帴锛涙枃妗?| 涓嶅叧娌欑涔熻兘纭В锛沗apt install` 鍙敤 | 4 / 10 | M5 | 闇€瑕侀噸缂?Firefox锛涗笂娓告帴鍙楀害 |
| **M7 HEVC** | VPS/SPS/PPS 閲嶅缓锛堟垨 R2 鐩撮€氬厹搴曪級 | HEVC 1080p PSNR 鈮?45 dB | 8 / 18 | M4 | RPS 淇℃伅涓嶈冻锛屽彲鑳藉仛涓嶆垚 |
| **M8 VP9** | VP9 uncompressed header 閲嶅缓 | VP9 1080p PSNR 鈮?45 dB | 5 / 9 | M4 | 瀛楁澶氥€佹槗閿?|
| **M9 绋冲畾鍖?鎬ц兘/鏂囨。** | 2 灏忔椂闀跨ǔ銆佹硠婕忎慨澶嶃€丆PU 璋冧紭銆佺敤鎴锋枃妗ｃ€乣BENCH.md` | L4 閫氳繃 | 4 / 7 | M6 | 闀垮熬 bug |

**姹囨€?*锛?
| 鑼冨洿 | 浜哄ぉ锛堜箰瑙傦級 | 浜哄ぉ锛堟偛瑙傦級 | 鏃ュ巻鏃堕棿锛? 浜猴級 |
|---|---|---|---|
| H.264 + Firefox锛圡0鈥揗6锛?| 24 | 48 | 5鈥? 鍛?|
| 鍔?HEVC + VP9锛圡7鈥揗8锛?| 37 | 75 | 8鈥?5 鍛?|
| 鍔犵ǔ瀹氬寲锛圡9锛?| 41 | 82 | 9鈥?6 鍛?|

**鍏抽敭璺緞**锛歁0 鈫?M1 鈫?M2 鈫?M3 鈫?M4 鈫?M5 鈫?M6銆侻7/M8 鍙互涓?M5/M6 骞惰锛堜笉鍚岀殑浜猴級銆?
**濡傛灉鍙湁 1 鍛?*锛氬彧鍋?M0 + M1 + M2 鐨?baseline 閮ㄥ垎锛岀洰鏍囨槸"`ffmpeg -hwaccel vaapi` 瑙ｅ嚭绗竴甯?PNG"锛堣 搂13锛夈€?
---

## 11. 椋庨櫓娓呭崟

鎸?浼氫笉浼氳椤圭洰澶辫触"鎺掑簭銆傛鐜?褰卞搷鐢?楂?涓?浣?琛ㄧず銆?
| # | 椋庨櫓 | 姒傜巼 | 褰卞搷 | 瑙﹀彂淇″彿 | 缂撹В/鍏滃簳 |
|---|---|---|---|---|---|
| R1 | **VA-API 涓嶆彁渚?SPS/PPS锛宻tateful 闇€瑕侀噸寤猴紝瀛楁涓嶅叏瀵艰嚧瑙ｄ笉鍑?鑺卞睆** | 楂?| 楂?| `ffmpeg` 鎶?"decode error"銆丳SNR 浣庛€佽姳灞?| 鍏堝仛 搂9.2 鐨勮蒋瑙ｅ崟娴嬶紱H.264 瀛楁鍩烘湰榻愬叏锛涘疄鍦ㄤ笉琛岃蛋 R2 鐩撮€氾紙閲嶇紪 ffmpeg/Firefox锛?|
| R2 | **HEVC 鐨?RPS 鏃犳硶浠?VA 鍙傛暟閲嶅缓** | 楂?| 涓?| HEVC 鍏ㄩ粦/鎶ラ敊 | HEVC 闃舵鐩存帴鏀圭敤 R2锛涙垨鍦ㄩ┍鍔ㄩ噷缂撳瓨"姣忎釜 IDR 鐨?SPS/PPS 鍘熸牱"鈥斺€斾絾鎷夸笉鍒板師鏍凤紝鍙兘 R2 |
| R3 | **Firefox 鍐呭杩涚▼ seccomp 鎷掔粷 `/dev/video0`** | 楂?| 楂?| `vaCreateContext` 澶辫触銆乣strace` 鐪嬪埌 EACCES | 搂7.4 鐨?S1鈫扴4鈫扴2鈫扴3锛涘厛鍏虫矙绠辫瘉鏄庨摼璺紝鍐嶅喅瀹氳ˉ涓?helper |
| R4 | **snap 鐗?Firefox 涓嶅彲鐢?*锛堥┍鍔ㄤ笉鍙 + AppArmor锛?| 楂?| 涓?| `snap run --shell` 閲岀湅涓嶅埌 so/璁惧 | 鎹?Mozilla deb锛堥」鐩凡鏈夊厛渚嬶級锛泂nap 涓嶄綔涓轰氦浠樼洰鏍?|
| R5 | **10bit 鍐呭鏃犳硶纭В**锛坈apture 鍙湁 NV12锛?| 纭畾 | 涓?| HEVC Main10 鍥為€€杞В | 鏄庣‘鍐欒繘鏂囨。涓?profile 鍒楄〃锛涘悗缁唴鏍告敮鎸?P010 鍐嶆墿灞?|
| R6 | **`SOURCE_CHANGE` 涓庡浐瀹氬昂瀵?VA surface 鍐茬獊** | 涓?| 涓?| 鍒囨竻鏅板害榛戝睆/鎶ラ敊 | 鏂板昂瀵?鈮?鏃у昂瀵告椂缁х画锛涙洿澶ф椂杩斿洖 `RESOLUTION_NOT_SUPPORTED`锛屼緷璧栧鎴风閲嶅缓锛汢 绔欏鏁板垏鎹㈡槸鏂板缓 decoder |
| R7 | **iris 涓嶅鍒?timestamp锛宻urface鈫斿抚鏃犳硶鍏宠仈** | 涓?| 楂?| 鐢婚潰椤哄簭閿欎贡/闂洖 | M1 `grep` 鍐呮牳锛涙病鏈夊氨缁欏唴鏍告墦涓€琛?`v4l2_m2m_buf_copy_metadata`锛涘啀涓嶈涓茶鍖栨ā寮忎粎鐢ㄤ簬鍔熻兘楠岃瘉 |
| R8 | **`VIDIOC_EXPBUF` 涓嶈 iris 鏀寔** | 涓?| 浣?| `EXPBUF` 杩斿洖 ENOTTY/EINVAL | MVP 鏈潵灏辩敤鎷疯礉妯″紡锛涢浂鎷疯礉闄嶇骇涓?L2 鍙€変紭鍖?|
| R9 | **瀵煎嚭鐨?dma-buf 涓嶈兘琚?freedreno EGL 瀵煎叆**锛坰tride/modifier/鏍煎紡锛?| 涓?| 楂?| 榛戝睆銆丒GL 閿欒 | 寮哄埗 LINEAR銆?56 瀛楄妭瀵归綈銆乣GR88` 涓ゅ眰锛涚敤 `eglinfo` + 鏈€灏?EGL 瀵煎叆绋嬪簭闅旂 |
| R10 | **iris 椹卞姩/FW 鏈韩鏈?bug**锛坔ang銆佽姳灞忋€?K 涓嶆敮鎸侊級 | 涓?| 楂?| `dmesg` 鎶ラ敊銆乣poll` 瓒呮椂 | 鐢?`v4l2-ctl`/ffmpeg `v4l2m2m` 鍏堣瘉鏄庡唴鏍歌矾寰勶紱蹇呰鏃剁粰鍐呮牳鎵撹ˉ涓侊紙鍗曠嫭璇勪及锛?|
| R11 | **Firefox 鍗充究涓€鍒囨甯镐篃涓嶈蛋 VA-API**锛坆locklist/鏋勫缓閫夐」/鐗堟湰琛屼负锛?| 涓?| 楂?| `about:support` 鏄剧ず鏈惎鐢?| `media.hardware-video-decoding.force-enabled=true`锛涙崲 Firefox 鐗堟湰锛涚敤 mpv/GStreamer 璇佹槑椹卞姩娌￠棶棰橈紱鏈€缁堝厹搴曟槸"缁存姢涓€涓墦琛ヤ竵鐨?mpv/ffmpeg" |
| R12 | **鎬ц兘涓嶈揪鏍?*锛堟嫹璐?鍚屾寮€閿€锛?| 涓?| 涓?| CPU 鍙檷鍒?90% | 涓婇浂鎷疯礉锛涘噺灏?`poll` 娆℃暟锛涙壒閲?`DQBUF`锛涙鏌?WebRender 鍚堟垚寮€閿€ |
| R13 | **澶氱嚎绋?鐢熷懡鍛ㄦ湡 bug**锛坰eek 宕╂簝銆乫d 娉勬紡锛?| 涓?| 涓?| 闅忔満宕╂簝銆乫d 澧為暱 | 鍗曢攣 + 涓ユ牸鎵€鏈夋潈锛埪?.1锛夛紱ASan + fd 璁℃暟 |
| R14 | **libva ABI/鐗堟湰宸紓**锛?.23 vs 鏇存柊鐨?2.24锛?| 浣?| 涓?| 鍔犺浇澶辫触銆乿table 鍋忕Щ閿?| 鐢?`VA_MAJOR_VERSION/VA_MINOR_VERSION` 鏉′欢缂栬瘧锛汣I 閲岃窇涓や釜 libva 鐗堟湰 |
| R15 | **涓婃父涓嶆帴鍙楀唴鏍歌ˉ涓侊紝闀挎湡瑕佸甫 patch** | 涓?| 浣?| review 琚嫆 | 琛ヤ竵寰堝皬锛堜竴琛?metadata 澶嶅埗锛夛紝鍏堣繘椤圭洰鍐呮牳浠撳簱锛屽啀鎺ㄤ笂娓?|
| R16 | **缁存姢鎴愭湰**锛團irefox/鍐呮牳/Mesa 鍗囩骇鐮村潖鍏煎锛?| 楂?| 浣?| 鍗囩骇鍚庡洖褰掑け璐?| 寤虹珛 搂9.7 鍥炲綊娓呭崟锛汣I 璺?`vainfo` + 鍥哄畾绱犳潗锛涢攣鐗堟湰 |

**鍏滃簳鏂规锛堝綋 Firefox 璺嚎褰诲簳璧颁笉閫氭椂锛?*锛?
1. **Chromium**锛歚VaapiVideoDecoder` 鍚屾牱闇€瑕?VA-API锛涘鏋滄垜浠殑椹卞姩鍦?`vainfo`/`mpv` 涓婃病闂锛孋hromium 澶ф鐜囪兘鐢紙瀹冨 sandbox 鐨勫鐞嗕笉鍚岋紝`/dev/video0` 鍙兘浠嶉渶 `--no-sandbox` 鎴?policy锛夈€?2. **mpv/ffmpeg 鎵撹ˉ涓?*锛氱淮鎶や竴涓甫 R2 鐩撮€氱殑 ffmpeg + `mpv --hwdec=vaapi`锛岃嚦灏戞湰鍦版挱鏀捐兘鐢紱杩欐槸"鐢ㄦ埛浣撻獙鍏滃簳"锛屼笉鏄祻瑙堝櫒鏂规銆?3. **Waydroid/瀹夊崜 App**锛氬畨鍗?B 绔欏鎴风璧扮殑鏄畨鍗撶殑 Codec2锛屼笉缁忚繃 Linux VA-API锛屼笌鏈」鐩棤鍏筹紙浣嗗彲浣滀负"鐪嬭棰?鐨勬浛浠ｈ矾寰勶級銆?
---

## 12. 闄勫綍

### 12.1 鍏抽敭鍛戒护娓呭崟

**璁惧涓庢牸寮?*

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
# vainfo 闇€瑕佸厛瑁?sudo apt install vainfo
LIBVA_DRIVER_NAME=iris LIBVA_MESSAGING_LEVEL=3 \
  vainfo --display drm --device /dev/dri/renderD128
LIBVA_DRIVER_NAME=iris vainfo --display drm --device /dev/dri/renderD128
```

**ffmpeg**

```sh
ffmpeg -hide_banner -hwaccels
ffmpeg -hide_banner -decoders | grep -E 'vaapi|v4l2m2m'

# 瑙ｇ爜涓嶅埌 CPU锛堟渶蹇獙璇侊級
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f null -

# 瑙ｇ爜 + 涓嬭浇 NV12锛堢敤浜?PSNR/md5锛?ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -f rawvideo -pix_fmt nv12 va.yuv

# 闆舵嫹璐濊矾寰?ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi -i in.mp4 -f null -

# 棣栧抚 PNG
ffmpeg -v error -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
       -i in.mp4 -frames:v 1 -vf "hwdownload,format=nv12" first.png
```

**GStreamer锛堥獙璇?V4L2 閫氳矾锛屼笉缁忚繃 VA锛?*

```sh
gst-inspect-1.0 v4l2h264dec
gst-launch-1.0 filesrc location=in.mp4 ! qtdemux ! h264parse ! \
  v4l2h264dec ! video/x-raw,format=NV12 ! fakesink
GST_DEBUG=v4l2*:5,v4l2videodec:6 gst-launch-1.0 ... 2>&1 | head -100
```

**mpv**

```sh
mpv --hwdec=vaapi --vo=gpu-next --msg-level=all=v in.mp4
mpv --hwdec=vaapi --vd-lavc-software-fallback=no in.mp4   # 绂佹鍥為€€锛岄獙璇佹槸鍚︾湡纭В
```

**Firefox**

```sh
# 纭鐗堟湰涓庡寘鏉ユ簮
snap list firefox            # 鎴?dpkg -l firefox
# 鍚姩骞舵姄濯掍綋鏃ュ織
MOZ_LOG="PlatformDecoderModule:5,FFmpeg:5,DMABuf:5" \
MOZ_LOG_FILE=/tmp/ffmedia.log \
LIBVA_DRIVER_NAME=iris \
firefox
# 鎶?openat
strace -f -e trace=openat -o /tmp/ff.strace firefox
grep -n 'video0\|renderD128' /tmp/ff.strace
```

**鍐呮牳涓庤皟璇?*

```sh
uname -a
dmesg | grep -iE 'iris|vb2|dma_buf|iommu'
ls -l /dev/dri /dev/video*
cat /sys/kernel/debug/dma_buf/bufinfo 2>/dev/null | head
grep -n 'v4l2_m2m_buf_copy_metadata' -r /path/to/linux/drivers/media/platform/qcom/iris/
```

### 12.2 鏈満瀹炴祴杈撳嚭鎽樺綍

> 浠ヤ笅涓?2026-10-26/27 鍦ㄨ澶囦笂閲囬泦鍒扮殑瀛楁绾х粨鏋滐紙涓嶆槸瀹屾暣鍛戒护杈撳嚭锛夛紝鐢ㄤ簬鍥炲綊瀵圭収銆?*鏀瑰姩鐜鍚庡厛澶嶆牳杩欏紶琛ㄣ€?*

| 鏌ヨ | 瀹炴祴鍊?|
|---|---|
| `v4l2-ctl --info` driver | `iris_driver` |
| card | `Iris Decoder` |
| bus_info | `platform:aa00000.video-codec` |
| Device Caps | `0x04204000` = `V4L2_CAP_VIDEO_M2M_MPLANE`(0x4000) \| `V4L2_CAP_EXT_PIX_FORMAT`(0x200000) \| `V4L2_CAP_STREAMING`(0x4000000) |
| OUTPUT 鏍煎紡 | `H264`銆乣HEVC`銆乣VP90`锛屽潎甯?`V4L2_FMT_FLAG_DYN_RESOLUTION` |
| CAPTURE 鏍煎紡 | `NV12`锛堟棤 10bit 鏍煎紡锛?|
| `/dev/dri/renderD128` | `crw-rw---- root:render 226,128`锛屽瓨鍦?|
| `/dev/dri/card0` | 瀛樺湪 |
| VA 椹卞姩鐩綍 | `d3d12_drv_video.so`銆乣nouveau_drv_video.so`銆乣r600_drv_video.so`銆乣radeonsi_drv_video.so`銆乣virtio_gpu_drv_video.so`锛?*鏃?msm/iris** |
| `libva2` | `2.23.0-1ubuntu1` |
| `vainfo` | **鏈畨瑁?*锛堜粨搴撴湁 `2.22.0+ds1-2build1`锛?|
| Mesa | `26.0.3`锛沗libEGL_mesa.so.0`銆乣libgbm.so.1`銆乣dri/msm_dri.so -> libdril_dri.so` |
| GStreamer | `1.28.2`锛沗v4l2h264dec`/`v4l2h265dec`/`v4l2vp9dec`锛圧ank primary+1, Hardware锛?|
| ffmpeg | `8.0.1`锛沗h264_v4l2m2m`/`hevc_v4l2m2m`锛?20p H.264 瀹炴祴鍙В |
| mpv | `0.41`锛沗--hwdec=v4l2m2m` 涓嶆敮鎸?|
| Firefox | `149.0.2-1`锛坰nap锛夛紱杞В 1080p 鏃跺叏杩涚▼ CPU 140%~170% |
| 缃戠粶 | 绾?283 kB/s |

### 12.3 鍙傝€冭祫鏂?
**libva / VA-API**

- libva 婧愮爜涓庡ご鏂囦欢锛?https://github.com/intel/libva>
- libva API 鏂囨。锛?https://intel.github.io/libva/>
- `va/va_backend.h`锛堥┍鍔?ABI锛夛細<https://github.com/intel/libva/blob/master/va/va_backend.h>
- `va/va_dec_h264.h` / `va_dec_hevc.h` / `va_dec_vp9.h`锛氬悓浠撳簱 `va/` 鐩綍
- `va/va_drmcommon.h`锛坄VADRMPRIMESurfaceDescriptor`锛夛細鍚屼粨搴?
**stateful V4L2**

- 鍐呮牳鏂囨。锛歁emory-to-Memory Stateful Video Decoder Interface锛?https://docs.kernel.org/userspace-api/media/v4l/dev-decoder.html>
- 鍐呮牳鏂囨。锛歁emory-to-Memory Devices锛?https://docs.kernel.org/userspace-api/media/v4l/dev-mem2mem.html>
- 鍐呮牳婧愮爜锛歚drivers/media/platform/qcom/iris/`锛堟湰鏈哄唴鏍镐粨搴撳悓璺緞锛?- 鍐呮牳 `v4l2-mem2mem.c`锛坄v4l2_m2m_buf_copy_metadata`銆乣V4L2_BUF_FLAG_LAST`锛?- `linux/videodev2.h` 鐨?`V4L2_DEC_CMD_STOP`銆乣V4L2_EVENT_SOURCE_CHANGE`銆乣V4L2_FMT_FLAG_DYN_RESOLUTION`

**鍙傝€冮┍鍔?/ 鍏堜緥**

- `bootlin/libva-v4l2-request`锛坰tateless锛夛細<https://github.com/bootlin/libva-v4l2-request>
- `mxsrc/libva-v4l2`锛坰tateful M2M锛夛細<https://github.com/mxsrc/libva-v4l2>
- `radxa-pkg/libva-v4l2`锛坰tateful M2M 鐨?fork/鎵撳寘锛夛細<https://github.com/radxa-pkg/libva-v4l2>
- Debian WNPP `#1152234` `qcom-va-driver`锛?https://bugs.debian.org/1152234>
- Chromium `V4L2VideoDecodeAccelerator`锛?https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/v4l2/v4l2_video_decode_accelerator.cc>

**ffmpeg / GStreamer**

- `libavutil/hwcontext_vaapi.c`锛坄vaExportSurfaceHandle` 璋冪敤鐐癸級锛?https://github.com/FFmpeg/FFmpeg/blob/master/libavutil/hwcontext_vaapi.c>
- `libavcodec/vaapi_decode.c` / `vaapi_h264.c`锛氬悓浠撳簱
- `libavcodec/v4l2_m2m.c` / `v4l2_m2m_dec.c`锛歴tateful M2M 鐢ㄦ硶鍙傝€?- GStreamer `gstv4l2videodec.c`銆乣gstv4l2bufferpool.c`锛?https://gitlab.freedesktop.org/gstreamer/gstreamer>

**Firefox**

- `FFmpegVideoDecoder.cpp`锛?https://searchfox.org/mozilla-central/source/dom/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp>
- `DMABufSurface.cpp`锛?https://searchfox.org/mozilla-central/source/gfx/2d/DMABufSurface.cpp>
- 娌欑绛栫暐锛歚security/sandbox/linux/broker/SandboxBrokerPolicyFactory.cpp`
- 棣栭€夐」锛歚modules/libpref/init/StaticPrefList.yaml`锛堟悳 `media.ffmpeg.vaapi`锛?- Firefox 婧愮爜閲屾墍鏈?VA-API 鐩稿叧浣嶇疆锛?https://searchfox.org/mozilla-central/search?q=vaapi>

**鍥惧舰**

- `EGL_EXT_image_dma_buf_import`锛?https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import.txt>
- `EGL_EXT_image_dma_buf_import_modifiers`锛氬悓鐩綍
- DRM format modifiers锛?https://docs.kernel.org/gpu/drm-kms.html>
- Mesa freedreno锛?https://docs.mesa3d.org/drivers/freedreno.html>
- libgbm锛?https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gbm>

**缂栬В鐮佽鑼?*

- ITU-T Rec. H.264锛堝惈 SPS/PPS/slice header 璇硶锛夛細<https://www.itu.int/rec/T-REC-H.264>
- ITU-T Rec. H.265锛圚EVC锛夛細<https://www.itu.int/rec/T-REC-H.265>
- VP9 bitstream spec锛?https://www.webmproject.org/vp9/>

### 12.4 鏈琛?
| 鏈 | 鍚箟 |
|---|---|
| **stateful decoder** | 鍐呮牳椹卞姩瑙ｆ瀽鐮佹祦銆佺鐞嗗弬鑰冨抚锛岀敤鎴锋€佸彧鍠傛暣甯х爜娴侊紱V4L2 鐨?M2M 妯″瀷 |
| **stateless decoder** | 鐢ㄦ埛鎬佽В鏋愮爜娴佸苟閫氳繃 Request API 鎻愪氦 per-slice 鍙傛暟锛沄4L2 鐨?Request API 妯″瀷 |
| **M2M** | memory-to-memory锛孷4L2 鐨勭紪瑙ｇ爜妗嗘灦锛圤UTPUT 闃熷垪杩涖€丆APTURE 闃熷垪鍑猴級 |
| **AU** | access unit锛屼竴涓畬鏁寸殑缂栫爜鍥惧儚锛圚.264 鐨勪竴甯э級 |
| **RBSP / EPB** | 鍘熷瀛楄妭搴忓垪杞借嵎 / 闃茬珵浜夊瓧鑺傦紙`00 00 03`锛?|
| **SPS / PPS / VPS** | 搴忓垪鍙傛暟闆?/ 鍥惧儚鍙傛暟闆?/ 瑙嗛鍙傛暟闆?|
| **VLD** | VA-API 鐨勫彉闀胯В鐮佸叆鍙ｇ偣锛坄VAEntrypointVLD`锛?|
| **DRM_PRIME_2** | VA surface 瀵煎嚭涓?dma-buf 鐨勭幇浠ｆ帴鍙ｏ紙`VADRMPRIMESurfaceDescriptor`锛?|
| **DMABuf / dma-buf** | 璺ㄨ澶囧叡浜紦鍐插尯鐨勫唴鏍告鏋?|
| **WebRender** | Firefox 鐨?GPU 鍚堟垚鍣紱DMABuf 绾圭悊鐢卞畠閲囨牱 |
| **RDD** | Firefox 鐨勮繙绋嬫暟鎹В鐮佽繘绋嬶紙鍙兘鎵胯浇 ffmpeg锛?|

---

## 13. 濡傛灉鍙仛涓€浠朵簨锛氫竴鍛?MVP

**鐩爣锛堝彲楠屾敹锛?*锛? 澶╁唴锛屽湪璁惧涓婄敤 `ffmpeg -hwaccel vaapi` 瑙ｅ嚭涓€甯?**H.264 Baseline 720p** 鐨勬纭敾闈紙PNG锛夛紝涓?`vainfo` 鑳藉垪鍑?`VAProfileH264Baseline/Main/High : VAEntrypointVLD`銆?*涓嶈姹?Firefox銆佷笉瑕佹眰 B 甯с€佷笉瑕佹眰闆舵嫹璐濄€?*

### 13.1 涓轰粈涔堥€夎繖涓寖鍥?
- Baseline 娌℃湁 B 甯?鈫?涓嶉渶瑕佸鐞嗛噸鎺掞紝surface鈫斿抚鐨勬槧灏勫彲浠ョ敤 FIFO锛岀粫寮€ timestamp 椋庨櫓锛圧7锛夛紱
- Baseline 鏄?CAVLC銆佹棤 8x8 transform銆佹棤 scaling list 鈫?SPS/PPS 閲嶅缓鍣ㄥ彲浠ョ爫鎺変竴鍗婂瓧娈碉紱
- `ffmpeg -hwaccel vaapi` 鏄渶瀹规槗鎺у埗鐨勫鎴风锛屽嚭閿欒兘鐪嬪埌鏄庣‘鏃ュ織锛?- 涓€甯?PNG 灏辫兘璇佹槑"VA 椹卞姩 鈫?V4L2 鈫?纭欢 鈫?dma-buf 鈫?瀵煎嚭"鏁存潯閾捐矾锛屽悗闈㈠彧鏄姞鍔熻兘銆?
### 13.2 涓冨ぉ鎺掓湡

| 澶?| 浠诲姟 | 浜や粯/楠屾敹 |
|---|---|---|
| **D1** | 瑁呬緷璧栵紙搂8.1锛夛紱`v4l2-ctl` 澶嶆牳璁惧锛涚敤 GStreamer/`ffmpeg -c:v h264_v4l2m2m` 璇佹槑鍐呮牳閫氳矾锛沜lone 骞剁矖璇?`mxsrc/radxa libva-v4l2`锛涘啓 `tools/v4l2-probe.c`锛氱洿鎺ュ杺 Annex B 鏂囦欢銆丮MAP銆乣DQBUF`銆乨ump 棣栧抚 NV12 | `v4l2-probe` 鑳?dump 鍑?720p NV12锛岃浆 PNG 鐢婚潰姝ｇ‘锛沗vainfo` 鍙繍琛岋紙鏈椹卞姩鏃堕鏈熸姤閿欙級 |
| **D2** | 椹卞姩楠ㄦ灦锛歚VA_DRIVER_INIT_FUNC`銆乿table銆乣vaQueryConfigProfiles`銆乣vaGetConfigAttributes`銆乣vaCreateConfig`銆乣vaDestroyConfig` | `vainfo` 閫€鍑虹爜 0锛岃緭鍑哄惈 H.264 涓変釜 profile锛堟鏃?`vaCreateContext` 鍙繑鍥?`UNIMPLEMENTED`锛?|
| **D3** | `vaCreateContext`锛坄open`/`SUBSCRIBE_EVENT`/`S_FMT`/`REQBUFS`/`STREAMON`锛夈€乣vaCreateSurfaces`锛堟嫹璐濇ā寮忥細gbm 鎴?`memfd` + `mmap`锛夈€乣vaDeriveImage`/`vaMapBuffer` | 绌鸿窇涓嶅穿锛沗vaCreateContext` 鍚?`v4l2-ctl` 鑳界湅鍒版牸寮忓凡璁剧疆 |
| **D4** | 鐮佹祦閲嶅缓鍣細`bs_writer`锛堜綅鍐欏叆/ue/EPB锛? H.264 SPS/PPS锛坆aseline 瀛愰泦锛? AU 缁勮锛涚敤 搂9.2 鐨勮蒋瑙ｅ崟娴嬮獙璇?| `ffmpeg -f h264 -i synth.h264 -f null -` 閫氳繃锛涘抚 md5 涓庡師濮嬩竴鑷?|
| **D5** | 鎵撻€?`vaBeginPicture`/`vaRenderPicture`/`vaEndPicture`/`vaSyncSurface`锛沗ffmpeg -frames:v 1` 鍑?PNG | **MVP 鏍稿績楠屾敹**锛歚/tmp/mvp.png` 鐢婚潰姝ｇ‘ |
| **D6** | 杩炵画 30 甯э紱PSNR 瀵规瘮锛涗慨 bug锛堥噸鐐癸細`bytesused`銆乣sizeimage`銆乣stride`銆佽鍓級 | 30 甯?PSNR 鈮?45 dB |
| **D7** | 鏀跺熬锛歚accept.sh`銆丷EADME銆佽褰曢樆濉炵偣锛坱imestamp/EXPBUF/sandbox 鐨勯獙璇佺粨鏋滐級銆佷笅涓€闃舵璁″垝 | 鍙氦鎺ョ殑浠撳簱 + 涓€椤电粨璁?|

### 13.3 MVP 鐨勯獙鏀跺懡浠?
```sh
# 1. 椹卞姩鍔犺浇
LIBVA_DRIVER_NAME=iris vainfo --display drm --device /dev/dri/renderD128 \
  | grep -E 'VAProfileH264(Baseline|Main|High).*VAEntrypointVLD'

# 2. 鍑轰竴甯?LIBVA_DRIVER_NAME=iris ffmpeg -v error \
  -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
  -i baseline_720p.mp4 -frames:v 1 \
  -vf "hwdownload,format=nv12" /tmp/mvp.png

# 3. 鐪嬬敾闈?xdg-open /tmp/mvp.png
```

### 13.4 MVP 鏄庣‘涓嶅仛鐨勪簨

- 鉂?B 甯?/ 閲嶆帓 / timestamp 鏄犲皠锛堢暀缁?M3锛?- 鉂?seek / flush / 鍒嗚鲸鐜囧彉鍖栵紙鐣欑粰 M3锛?- 鉂?闆舵嫹璐?/ `vaExportSurfaceHandle`锛堢暀缁?M4锛?- 鉂?Firefox / sandbox / snap锛堢暀缁?M5/M6锛?- 鉂?HEVC / VP9 / 10bit锛堢暀缁?M7/M8锛?- 鉂?澶?slice锛堝厛鍋囪鍗?slice锛汥6 鑻ラ亣鍒板 slice 鍐嶈ˉ锛?
### 13.5 MVP 鍙兘鍗′綇鐨勫湴鏂逛笌搴斿

| 鍗＄偣 | 搴斿 |
|---|---|
| `vainfo` 鍔犺浇澶辫触锛堢鍙?璺緞/libva 鐗堟湰锛?| 鐢?`LD_DEBUG=libs`銆乣nm -D`銆乣LIBVA_MESSAGING_LEVEL=3` 瀹氫綅锛涘厛璺?`LIBVA_MESSAGING_LEVEL=3 vainfo --display drm --device /dev/dri/renderD128` 鐪嬭缁?|
| `S_FMT` 琚嫆锛坄EINVAL`锛?| 鍏?`TRY_FMT` 鐪嬮┍鍔ㄦ帴鍙楃殑灏哄/鏍煎紡锛涚‘璁?`num_planes=1`銆乣sizeimage` 鍚堢悊 |
| `QBUF` 鍚?`DQBUF` 涓€鐩?`EAGAIN` | 鎵撳紑 `LIBVA_IRIS_DEBUG=1` 鎵撳嵃姣忔 ioctl锛涚敤 `v4l2-probe` 鍦ㄥ悓涓€鐮佹祦涓婂姣旓紝纭鏄?椹卞姩闂"杩樻槸"缁勫寘闂" |
| 缁勫寘鍚庤蒋瑙ｉ兘涓嶉€氳繃 | 鍥炲埌 搂9.2 鍗曟祴锛岄€愬瓧娈靛鐓?`ffmpeg -bsf:v trace_headers` 鐨勮緭鍑?|
| 鍑哄浘浣嗚姳灞?缁胯竟 | 鍏堟煡 `bytesperline`/`uv_offset`/瑁佸壀锛埪?.7锛夛紝鍐嶆煡鑹插害骞抽潰椤哄簭锛圢V12 鏄?UV 浜ら敊锛屼笉鏄?U 骞抽潰 + V 骞抽潰锛?|
| 鏃堕棿涓嶅 | 鐮?D6 鐨勮繛缁抚锛屽彧浜や粯"鍗曞抚 + 鍗曟祴"锛屾妸闂璁板綍娓呮锛?*涓嶈涓轰簡璧舵椂闂磋烦杩?D4 鐨勫崟娴?*锛屽惁鍒欏悗闈㈡墍鏈?bug 閮戒細鎬€鐤戝埌閲嶅缓鍣?|

### 13.6 涓€鍛ㄤ箣鍚?
MVP 閫氳繃鍚庯紝鎸?搂10 鐨勯『搴忔帹杩涳細M3锛堣繛缁?seek锛夆啋 M4锛堥浂鎷疯礉锛夆啋 M5锛團irefox 璋冭瘯鎬侊級鈫?M6锛堟矙绠?鎵撳寘锛夈€?*姣忓畬鎴愪竴涓?M锛岃窇涓€娆?搂9.7 鐨勫洖褰掓竻鍗?*锛屼笉瑕佺瓑鍒版渶鍚庛€?
---

## 闄勶細寮€宸ョ涓€灏忔椂娓呭崟

1. `sudo apt install -y build-essential meson ninja-build pkg-config libva-dev libdrm-dev libgbm-dev vainfo v4l-utils`锛堢害 8鈥?2 鍒嗛挓锛?2. `v4l2-ctl -d /dev/video0 --all`锛屾妸杈撳嚭璐磋繘 `NOTES.md`锛屼笌 搂12.2 瀵圭収
3. `git clone https://github.com/mxsrc/libva-v4l2 && cd libva-v4l2 && ls src/`锛屽洖绛斾笁涓棶棰橈細鏀寔 stateful 鍚楋紵H.264 鐨?SPS/PPS 鎬庝箞鏉ワ紵鏈?`vaExportSurfaceHandle` 鍚楋紵
4. `grep -rn "v4l2_m2m_buf_copy_metadata" drivers/media/platform/qcom/iris/`锛堝湪鍐呮牳婧愮爜閲岋級
5. 鍐?`tools/v4l2-probe.c` 鐨勭涓€涓増鏈細`open` 鈫?`S_FMT` 鈫?`REQBUFS` 鈫?`QBUF` 涓€涓?`.h264` 鏂囦欢 鈫?`DQBUF` 鈫?瀛?NV12
6. 鎶?`v4l2-probe` 鐨勮緭鍑虹敤 `ffmpeg -f rawvideo -pix_fmt nv12 -s 1280x720 -i out.nv12 out.png` 杞垚鍥撅紝纭鐢婚潰姝ｇ‘

鍋氬畬杩?6 姝ワ紝浣犲氨鏈変簡"纭欢閫氳矾鏄€氱殑"杩欎釜鏈€閲嶈鐨勭‘瀹氭€э紱鍓╀笅鐨勯兘鏄蒋浠跺伐绋嬨€?




