# 2026-10-29 — 恢复 passwall feeds，导入 LuCI 应用包集合

## 上游基点

- 仓库：`pbodq2/OpenWRT-W1700K-XR1710G`（`kaiwater` fork 同步点）
- 分支：`ubi2`
- 基点：`22d02fe`

## 背景

旧 ImmortalWrt 构建的 `.config`（`~/Downloads/1710 (3).config`）里有一批常用的 LuCI 应用，但该配置是完整系统配置，包含 PON、内核模块、驱动、固件、board 等系统级选项，直接套到 OpenWrt 主线会与上游默认值冲突。

目标：只把 **LuCI 应用 + 显式运行时依赖** 带过来，系统级选项全部沿用上游默认。

## 改动

### 1. `feeds.conf.default`

在 OpenWrt 官方 feed 之外新增：

```text
src-git passwall_packages https://github.com/Openwrt-Passwall/openwrt-passwall-packages.git;main
src-git passwall_luci https://github.com/Openwrt-Passwall/openwrt-passwall.git;main
src-git app_airoha https://github.com/naoki66/luci-app-airoha.git
src-git lucky https://github.com/gdy666/luci-app-lucky.git;main
src-git rtp2httpd https://github.com/stackia/rtp2httpd.git;main
src-git imm_luci https://github.com/immortalwrt/luci.git
src-git imm_packages https://github.com/immortalwrt/packages.git
```

- `imm_luci` / `imm_packages` 只用于 openwrt/luci 中不存在的 App（argon、wechatpush、msd_lite、timewol、vlmcsd、arpbind、autoreboot、ddns-go、zerotier）。
- 官方 feed 排在前面，`scripts/feeds install -a` 时同名包会保留 OpenWrt 主线版本。
- 未加入 `pon_drivers` / `onu_config`。

### 2. `configs/xr1710g-apps.seed`

只包含：

- `CONFIG_TARGET_*`：`airoha/an7581` + `gemtek_xr1710g-ubi`
- `CONFIG_LUCI_LANG_zh_Hans=y`
- 36 个 `luci-app-*` / 主题 / 中文语言包
- 显式运行时依赖：`xray-core`、`sing-box`、`chinadns-ng`、`smartdns`、`lucky`、`msd_lite`、`rtp2httpd`、`ddns-go`、`zerotier` 等

### 3. `configs/README.md`

记录 feed 分工、构建步骤、排除项和已知限制。

## 验证

在本机执行：

```sh
./scripts/feeds update -a
./scripts/feeds install -a
cp configs/xr1710g-apps.seed .config
FORCE=1 gmake defconfig
```

结果：

- 120 个 seed 符号全部保留，无缺失；
- `luci-app-passwall`、`lucky`、`msd_lite`、`wechatpush`、`argon`、`zerotier`、`rtp2httpd`、`airoha-factory`、`netmode` 均成功选中；
- `xray-core`、`sing-box` 等依赖已解析。

本机为 macOS，`make defconfig` 需要绕过 OpenWrt 的宿主环境检查（`staging_dir/host/.prereq-build` + `FORCE=1`），仅影响本机验证，不属于仓库改动。

## 排除项

- PON：`pon_drivers`、`onu_config`、`kmod-airoha-xpon`、`airoha-pon*`、`kmod-sfp`、`luci-app-sfp-info`
- 内核/驱动/固件：全部 `kmod-*`、`airoha-*firmware`、`rtl826x-firmware`、`wireless-regdb`、`airoha-an7581-mt7996-board`
- 系统服务/基础包：`dnsmasq*`、`hostapd*`/`wpad*`、`ppp*`、`odhcp*`、`firewall4`、`procd*`、`base-files` 等
- ImmortalWrt 专有系统包：`default-settings*`、`autocore`、`cpufreq`、`shellsync`

## 已知限制

- `luci-theme-glass` 的 `Makefile` 在仓库根目录，`scripts/feeds` 不支持这种布局，未纳入；需要时手动放进 `package/`。
- `imm_luci` 的应用按 immortalwrt 的 `luci.mk` 编译，与 OpenWrt 主线 LuCI 版本不同，首次完整编译时需重点检查。
- `app_airoha` 里的 `luci-app-airoha-npu` 与仓库自带的同名包冲突，已删除 `package/feeds/app_airoha/luci-app-airoha-npu` 软链接，保留仓库自带版本。

## 回滚

```sh
git checkout 22d02fe -- feeds.conf.default
rm -rf configs/xr1710g-apps.seed configs/README.md
rm -rf .config feeds package/feeds tmp staging_dir
```
