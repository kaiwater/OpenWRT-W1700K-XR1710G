# 2026-11-01 — mt76 切到 naoki66/lowaka 版本（排查 iPhone 下行问题）

## 决策

继续以 **pbodq2 为基线**，只把 `package/kernel/mt76` 同步到 **naoki66 / lowaka** 使用的版本。

已验证两者完全一致：

- `PKG_SOURCE_URL=https://github.com/openwrt/mt76`
- `PKG_SOURCE_VERSION=467ae2e3f190baf737e15e318906da0dc7d0ab91`
- 同一组 5 个补丁

## 背景

iPhone（`7e:d0:07:d6:0d:b5` / `76:09:72:e5:ce:8c`）在 pbodq2 构建上表现为：

- 关联正常，上行 1200-2400 Mbps
- AP 下行 `tx bitrate 24 Mbps`、`last ack signal 0`
- DHCP/ARP 失败，ping 100% 丢包
- 换 WPA2、HE80、关 MLO、关 U-APSD、关 PPE 硬件加速都无效
- 重连后短暂恢复，过一段时间又坏

lowaka 的 ImmortalWrt 构建没有这个问题，因此对比了两边的 mt76。

## 差异

| | lowaka | pbodq2（原） |
| --- | --- | --- |
| mt76 源 | `openwrt/mt76` @ `467ae2e` | `OpenWRT-fanboy/mt76-firmware` @ `639e128` |
| NPU 补丁 | `notify NPU on station removal to flush sta`、`set skb device for NPU rx`、`pass operating mode to rate control` | RX descriptor ownership、keep NPU RRO data on original RX queue、token/NPU counters、WTBL station stats、txfree guard |
| mt76-core 依赖 | 无 | `+kmod-airoha-eth` |
| kernel | 6.18 | 6.18.55 |
| hostapd | `831364bf` | `831364bf`（相同） |

iPhone 的症状（连一会儿后下行死掉、重连暂时好）很像 **NPU station 表/RX 路径残留**，lowaka 的 `notify NPU on station removal to flush sta` 正好针对这个问题。

## 改动

`package/kernel/mt76`：

- `Makefile` 改为 lowaka 版本：
  - `PKG_SOURCE_URL=https://github.com/openwrt/mt76`
  - `PKG_SOURCE_VERSION=467ae2e3f190baf737e15e318906da0dc7d0ab91`
  - `PKG_MIRROR_HASH=2456aeba...`
  - 去掉 `+TARGET_airoha_an7581:kmod-airoha-eth`
- 删除原 fanboy/NPU 补丁：
  - `0001`、`0003`、`0005`、`0010`、`0011`
  - `NoTengoBattery-...rx-descriptor...`
  - `hurryman-9990/9991/9992`
- 使用 lowaka 的 5 个补丁：
  - `0003-wifi-mt76-notify-NPU-on-station-removal-to-flush-sta.patch`
  - `100-wifi-mt76-mt7996-Use-tx_power-from-default-fw-if-EEP.patch`
  - `110-mt7915-add-mt7916-cal-free-merge.patch`
  - `9993-wifi-mt76-mt7996-pass-operating-mode-to-rate-control.patch`
  - `9994-wifi-mt76-mt7996-set-skb-device-for-npu-rx.patch`

kernel、hostapd、设备树、feeds、seed 均未改。

## 缓存

- `dl` 缓存：2455 MB
- 最新 ccache：308 MB（key `ccache-7327...-41260491406`）
- seed/feeds 未变，restore-keys 可命中；但 build_dir 不缓存，Go 包和打包仍要重跑，预计仍需要约 2-3 小时，不是完全免构建

## 构建

- commit：`620e865 fix(mt76): switch to lowaka openwrt/mt76 NPU patches`
- run：`41285585828`
- 结果待定；如果 mt76 补丁应用失败或 NPU API 不兼容，会在 Build 步骤报错

## 设备侧临时诊断配置

为了隔离 iPhone 问题，设备上临时改过：

- 2.4G `EHT20`
- 5G `HE80`
- `mlo0` 禁用
- 新增 `test-iphone`（WPA2、U-APSD off、可见）
- `radio1.he_twt_responder=0`
- firewall `flow_offloading_hw=0`

这些**不在固件里**，只在设备配置中。备份：

- `/root/wireless.bak-20261101-1455`
- `/root/wireless.bak-mlo-20261101-1500`
- `/root/wireless.bak-testssid-*`
- `/root/wireless.bak-he80-*`
- `/root/wireless.bak-iphone-*`

如果 mt76 修复生效，下一步恢复为 EHT20 + EHT160 + MLO，删除 `test-iphone`，恢复 firewall HW offload。

## 验证

1. 触发 GitHub Actions 构建
2. 刷机后让 iPhone 连 `3000M8_`（或先 `test-iphone`）
3. 检查 `tx bitrate` 是否不再卡 24 Mbps、DHCP 是否正常、ARP 是否 REACHABLE
4. 持续使用一段时间，确认不再出现“连一会儿后断流”

## 回滚

```sh
git checkout 9f45f0a -- package/kernel/mt76
```

如果仅换 mt76 不够，下一步考虑把 lowaka 的 airoha target/NPU 补丁也一起对比移植。
