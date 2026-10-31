# Config seed — Gemtek XR1710G (W1700K)

`xr1710g-apps.seed` 只包含：

- 目标设备选择（`airoha/an7581`，`gemtek_xr1710g` + `gemtek_xr1710g-ubi`）
- LuCI 应用（`luci-app-*`、主题、中文语言包）
- 这些应用的显式运行时依赖（xray-core、sing-box、chinadns-ng、smartdns 等）

系统级选项（内核模块、驱动、固件、board、hostapd/wpad、dnsmasq、PON 等）不写入 seed，交给上游默认配置。

## 用法

```sh
./scripts/feeds update -a
./scripts/feeds install -a
cp configs/xr1710g-apps.seed .config
make defconfig
```

`make defconfig` 会补齐依赖并丢弃当前源码树里不存在的符号。

seed 里已启用 `CONFIG_DEVEL=y` + `CONFIG_CCACHE=y`，ccache 目录默认为源码树下的 `.ccache`。

## GitHub Actions

`.github/workflows/build-firmware.yml` 会在 push 到 `ubi2`（`configs/**`、`feeds.conf.default` 变化）或手动触发时编译固件：

```sh
gh workflow run build-firmware.yml --ref ubi2
```

缓存策略：

- `dl/` 与 `.ccache/` 在 checkout 后立即恢复，早于依赖安装和编译；
- ccache 从 toolchain 阶段开始生效（`CCACHE_COMPILERCHECK=content`，toolchain 重建不会失效）；
- 编译失败也会保存缓存（`if: always()`），方便连续试错。

## Feeds 说明

| Feed | 用途 |
| --- | --- |
| `packages` / `luci` / `routing` / `telephony` | OpenWrt 官方 feed（与上游主线一致，勿替换成 immortalwrt 版本） |
| `passwall_packages` / `passwall_luci` | passwall 及其代理内核 |
| `app_airoha` | `luci-app-airoha-factory` / `fancontrol` / `recovery` / `mesh-conf` / `netmode` |
| `lucky` | `lucky` + `luci-app-lucky` |
| `rtp2httpd` | `rtp2httpd` + `luci-app-rtp2httpd`（`openwrt-support/` 子目录） |
| `imm_luci` / `imm_packages` | 辅助 feed，只用于 openwrt/luci 中没有的 App：argon 主题、wechatpush、msd_lite、timewol、vlmcsd、arpbind、autoreboot、ddns-go、zerotier |

`imm_luci` / `imm_packages` 与官方 feed 有大量同名包。`scripts/feeds` 先处理官方 feed，已安装的同名包会被跳过，因此 `install -a` 不会覆盖 OpenWrt 主线版本。仍建议在升级上游后检查一次 `git status` 和 `make defconfig` 输出。

## 未纳入 / 已知限制

- **PON 相关**：`pon_drivers`、`onu_config` 及 `kmod-airoha-xpon`、`airoha-pon*`、`kmod-sfp` 等全部排除。
- **ImmortalWrt 系统级包**：`default-settings`、`default-settings-chn`、`autocore`、`cpufreq`、`shellsync` 不在 feed 中，且属于系统层，未纳入。
- **luci-theme-glass**：仓库把 `Makefile` 放在根目录，`scripts/feeds` 的扫描深度不会索引根目录下的包，无法直接作为 feed。需要的话只能手动放进 `package/`。
- **风扇控制二选一**：`luci-app-airoha-fancontrol` 与 `luci-app-w1700k-fancontrol` 会安装同一组文件（`etc/config/fan`、`etc/init.d/fan`、`luci.fan`、`view/fan/*`），不能同时选中。参考 naoki66 的 XR1710G 配置，本仓库把 XR1710G profile 改为使用 `luci-app-airoha-fancontrol`；W1700K profile 仍保留 `luci-app-w1700k-fancontrol`。用这个 seed 编 W1700K 时会冲突，需要改 seed。
- `simple-obfs-client`、`smartdns-ui` 是旧包名，当前 feed 中可能已不存在；`make defconfig` 会自动丢弃，并用 `luci-app-smartdns` 替代。
