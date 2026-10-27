# liuqin-mainline-drivers

Mainline Linux work for the **Xiaomi Pad 6 Pro** (`liuqin`, Qualcomm SM8475 / Snapdragon 8+ Gen 1),
running **Ubuntu 26.04 LTS from internal storage** (kernel `6.17.0-rc1-gfe81794b5e1b`, project
[yzddmr6/xiaomipad-6pro-mainline](https://github.com/yzddmr6/xiaomipad-6pro-mainline)).

This repository collects the device-specific findings, driver investigations and helper tools that
are not part of the upstream port: what works, what does not, and what would be needed to fix the
remaining gaps.

## Status at a glance

| Area | Status | Notes |
|---|---|---|
| Display / touch / pen / keyboard cover | working | DSI 1800x2880, `Xiaomi Touch`, `NVTCapacitivePen` |
| Wi-Fi / Bluetooth | working | ath11k + `hci0`; BT audio codecs (aptX, HFP) available |
| Camera (rear) | working, experimental | Samsung S5KJN1 via libcamera `simple` + CPU soft-ISP; no AF, no tuning |
| Camera (front) | **not supported** | Sony IMX596 — no mainline driver exists (see `docs/FRONTCAM-2026-10-27.md`) |
| Fingerprint | working | FPC1264; required a readiness-check fix (see `docs/DRIVERS-2026-10-27.md`) |
| Speakers | working | 4x CS35L41 (2 per side); per-driver frequency response measured |
| Microphone | working | WCD938x via AudioReach |
| Video decode (local) | working | `qcom-iris` stateful M2M: H.264 / HEVC / VP9 |
| Video decode in **browser** | **not working** | needs a VA-API driver for `iris`; none exists (see `docs/TASK-vaapi-iris-2026-10-27.md`) |
| Sensors / auto-rotate | **not working** | starting SLPI hard-resets this hardware revision (HW 16.9.0) |
| Suspend / resume | working | verified with `rtcwake -m mem` |
| Hardware VM (KVM) | unavailable | no `/dev/kvm`, no KVM in the kernel config |

## Documents

- `docs/HARDWARE-2026-10-27.md` — full hardware audit, what was fixed (user groups, console-setup, UCM), swap setup
- `docs/DRIVERS-2026-10-27.md` — driver status table and the fingerprint fix
- `docs/FRONTCAM-2026-10-27.md` — front-camera investigation: sensor identification (IMX596), DT nodes, missing driver
- `docs/TASK-vaapi-iris-2026-10-27.md` — task book: implementing a VA-API driver for the `iris` decoder so browsers can hardware-decode
- `docs/FEASIBILITY-UBUNTU-2026-10-27.md` — single-system feasibility, performance measurements, application memory budgets
- `docs/SOLVED-2026-10-27.md` — why the internal-storage boot failed and the vbmeta fix that solved it
- `docs/PROGRESS-2026-10-27.md` — chronological work log
- `docs/STATUS-2026-10-26-2110.md` — earlier status snapshot (installation, partition layout)

## Tools

- `tools/liuqin-speaker-swap` — swap the left/right channel assignment of the four CS35L41 amps
  (`liuqin-speaker-swap on|off|status`), for comparing landscape/portrait stereo imaging.

## Notes on the boot chain

The port boots from internal storage with `boot_a` holding a header-v2 boot image and
`vbmeta_a`/`vbmeta_b` flashed with `--disable-verity --disable-verification`. Verification has to be
disabled because this bootloader performs a sized read of the whole `boot_a` partition and fails
with `avb_slot_verify.c:210: ERROR: boot_a: Read fewer than requested bytes.` — see
`docs/SOLVED-2026-10-27.md`.
