# Changelog

本目录记录本仓库相对上游 [`pbodq2/OpenWRT-W1700K-XR1710G`](https://github.com/pbodq2/OpenWRT-W1700K-XR1710G) 的所有自定义改动、决策和踩坑记录，便于上游更新时判断冲突、迁移或回滚。

## 约定

- 文件名：`YYYY-MM-DD_NN_slug.md`，同一天多批改动用 `_01`、`_02` 递增。
- 每批改动一个文件，不追加到旧文件。
- 每个文件尽量包含：
  - 上游基点
  - 背景 / 原因
  - 改动内容（文件、关键 diff）
  - 验证方式 / 结果
  - 回滚方式
- 纯上游同步也记一条。

## 索引

| 日期 | 文件 | 内容 |
| --- | --- | --- |
| 2026-10-29 | [`2026-10-29_01_reset-and-sync-upstream.md`](2026-10-29_01_reset-and-sync-upstream.md) | 放弃旧浅克隆，重新同步上游，建立 changelog |
| 2026-10-29 | [`2026-10-29_02_feeds-and-apps.md`](2026-10-29_02_feeds-and-apps.md) | 恢复 passwall feeds，导入 LuCI 应用包集合 |
| 2026-10-29 | [`2026-10-29_03_ci-build.md`](2026-10-29_03_ci-build.md) | GitHub Actions 编译，前置 dl/ccache 缓存 |
| 2026-10-30 | [`2026-10-30_04_build-image-fix.md`](2026-10-30_04_build-image-fix.md) | 首次镜像阶段失败，去掉 per-device rootfs，补诊断 |
| 2026-10-30 | [`2026-10-30_05_manifest-fix.md`](2026-10-30_05_manifest-fix.md) | manifest 使用 `root.orig` 导致失败，改为按 rootfs 类型选择 |
| 2026-10-31 | [`2026-10-31_06_build-retry.md`](2026-10-31_06_build-retry.md) | 构建失败自动重试，补 `package/install` 诊断 |
| 2026-10-31 | [`2026-10-31_07_fancontrol-conflict.md`](2026-10-31_07_fancontrol-conflict.md) | `w1700k-fancontrol` 与 `airoha-fancontrol` 文件冲突 |
| 2026-10-31 | [`2026-10-31_08_xr1710g-fancontrol.md`](2026-10-31_08_xr1710g-fancontrol.md) | XR1710G 改用 `airoha-fancontrol`，构建成功 |
| 2026-11-01 | [`2026-11-01_09_xr1710g-layout.md`](2026-11-01_09_xr1710g-layout.md) | 确认 XR1710G 存在 0x700000 / 0x20000 两套布局 |
| 2026-11-01 | [`2026-11-01_10_fw_env-xr1710g.md`](2026-11-01_10_fw_env-xr1710g.md) | XR1710G `fw_env.config` 必须用 `0x4000` |
| 2026-11-01 | [`2026-11-01_11_xr1710g-compat-name.md`](2026-11-01_11_xr1710g-compat-name.md) | 兼容 `gemtek,xr1710g`，镜像名加入 `_0x700000` |
| 2026-11-01 | [`2026-11-01_12_flash-xr1710g.md`](2026-11-01_12_flash-xr1710g.md) | 实际刷机成功记录与 `cd /tmp` 经验 |
| 2026-11-01 | [`2026-11-01_13_release-xr1710g.md`](2026-11-01_13_release-xr1710g.md) | 发布 GitHub Release `26.11.01_xr1710g` |
| 2026-11-01 | [`2026-11-01_14_mlo-fix.md`](2026-11-01_14_mlo-fix.md) | 2.4G EHT20 修复 MLO；客户端互操作问题记录 |

## 当前状态

- 上游基点：`pbodq2/OpenWRT-W1700K-XR1710G` `ubi2` @ `22d02fe`
- 分支：`ubi2`
- Feeds：OpenWrt 官方 `packages/luci/routing/telephony` + passwall + `app_airoha` + lucky + rtp2httpd + 辅助 `imm_luci/imm_packages`
- 应用配置：`configs/xr1710g-apps.seed`
- CI：`.github/workflows/build-firmware.yml`
- XR1710G 实际布局：UBI `0x700000`，`fit` 卷，reserved BMT，ubi2.0 chainloader
- 风扇：XR1710G 用 `luci-app-airoha-fancontrol`；W1700K profile 保持 `luci-app-w1700k-fancontrol`
- U-Boot env：XR1710G 使用 `envsize=0x4000`、`secsize=0x1f000`
- 设备状态：已刷入 OpenWrt SNAPSHOT `r0-9f45f0a`，board `gemtek,xr1710g-ubi`，compat 2.0，`fit` 卷 84.0MiB，配置保留
- 无线：2.4G `EHT20` + 5G `EHT160`，`ap-mld0` 两个 link 正常；客户端 `4E:5D:25:EE:71:9E` 仍有 MLO 下行互操作问题，建议用 5G 普通 SSID 验证
- Release：`26.11.01_xr1710g`，https://github.com/kaiwater/OpenWRT-W1700K-XR1710G/releases/tag/26.11.01_xr1710g

## 关键决策 / 思路

1. **只带 App，不带系统级改动**
   旧 ImmortalWrt 的完整 `.config` 只取 `luci-app-*` 和显式运行时依赖；PON、kmod、驱动、固件、board、hostapd/wpad、dnsmasq 等全部交给上游默认。

2. **官方 feed 优先，imm 只做补充**
   `packages/luci/routing/telephony` 保持 OpenWrt 官方版本；`imm_luci/imm_packages` 只用于官方 feed 没有的 App（argon、wechatpush、msd_lite 等）。同名包由官方 feed 先安装。

3. **CI 缓存尽量前置**
   checkout 后立刻恢复 `dl/` 和 `.ccache/`，再装依赖、跑 feeds；seed 里启用 `CONFIG_DEVEL` + `CONFIG_CCACHE`，让 toolchain 第一次编译就命中 ccache。`CCACHE_COMPILERCHECK=content` 避免 toolchain 重建导致缓存失效。

4. **失败也要留证据**
   `BUILD_LOG=1`、失败时 `make package/install V=s` / `target/install V=s`，并上传 `build.log`、`diagnostics.log`、`logs.tar.gz`、镜像。失败自动重试一次。

5. **不要把失败 run 的镜像当成品**
   `package/install` 失败后，诊断步骤里的 `target/install` 仍可能生成 `.itb`，但 rootfs 可能不完整。只有 Build 步骤成功的 run 的 artifact 才能刷。

6. **XR1710G 只有 0x700000 布局可用**
   设备是 stock 签名 U-Boot + 0x600000 处的 ubi2.0 chainloader；0x20000 是 bl2/fip 新启动链，签名锁下不可用。W1700K 同理。

7. **XR1710G 专用差异**
   - 风扇用 `airoha-fancontrol`，不是 `w1700k-fancontrol`
   - `fw_env.config` 用 `0x4000`，不是通用 `0x1f000`
   - `SUPPORTED_DEVICES` 同时接受 `gemtek,xr1710g` 和 `gemtek,xr1710g-ubi`
   - 镜像名带 `_0x700000`，避免和 naoki66 的 0x20000 布局混淆

## 构建状态

- 最近成功：run `41260491406`（`9f45f0a`，2026-11-01 01:17 → 04:22，约 3h05m）
  - artifact：`xr1710g-firmware`（160 MB）、`build-diagnostics`（1 MB）
  - 已包含：`fw_env` 0x4000、`gemtek,xr1710g` 兼容、`_0x700000` 文件名
  - 镜像：
    - `openwrt-airoha-an7581-gemtek_xr1710g-ubi_0x700000-squashfs-sysupgrade.itb`（88,101,727 bytes / 84.0 MiB）
    - `openwrt-airoha-an7581-gemtek_xr1710g-ubi_0x700000-initramfs-recovery.itb`（80,609,280 bytes / 76.9 MiB）
- 上一个成功 run `41213689379`（`d874329`）已被本次取代；它没有 fw_env 和文件名修正。
- 中间 run `41254498291`（只含 fw_env 修正）构建失败且没有诊断 artifact，已由 `41260491406` 取代。
- 刷机状态：已用该镜像升级成功（见 `2026-11-01_12_flash-xr1710g.md`）。设备当前 OpenWrt SNAPSHOT `r0-9f45f0a`，board `gemtek,xr1710g-ubi`，compat 2.0，`fit` 卷 84.0MiB。
- Release：`26.11.01_xr1710g` 已发布（公开下载），assets 含 sysupgrade/recovery/manifest/sha256sums/buildinfo。

## 重要提醒

- **刷机只使用 Build 成功的 run 的 artifact**；之前 160MB 的失败 run 镜像不要用。
- 0x700000 镜像只能配 ubi2.0 chainloader；0x20000 镜像（naoki66 `-ubi`）不能混刷。
- 从当前 `gemtek,xr1710g` 系统升级时，设备名已兼容；compat 2.0 仍可能提示，确认 chainloader/布局后用 `-F`。
- SSH 升级必须 `cd /tmp` 后再执行 `sysupgrade`；从 `/root` 启动会因 overlay busy 导致卷重建失败（第一次尝试就是这样，`fit` 卷仍停在 79.3MiB）。
- MLO：2.4G 必须是 `EHT20`，否则 hostapd 报 `mld_ap` 错误、2.4G link 不启动。当前两个 link 正常；特定客户端 `4E:5D:25:EE:71:9E` 仍有下行互操作问题，先用普通 5G SSID `3000M8_` 验证。
- 如果不想用 `-F`，可以在旧系统执行 `uci set system.@system[0].compat_version='2.0'` 后重试 LuCI。
- 保留 `/etc/config/ubootenv` 时 `fw_env.config` 可沿用 0x4000；干净安装依赖新镜像里的修正。
