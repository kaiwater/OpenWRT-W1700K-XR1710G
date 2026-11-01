# 2026-11-01 — fanboy 上游 / pbodq2 mt76 分叉分析

## 结论

- **fanboy 的 OpenW1700k main** 用的是 `openwrt/mt76` **2026-10-28**（`0936617`），但只有 **2 个** 补丁，且都是 mt76x02/mt76x2u 的，和 MT7996/NPU 无关。
- **pbodq2** 并不是直接用 fanboy 的 OpenW1700k mt76，而是用另一个 fork：`OpenWRT-fanboy/mt76-firmware` **2026-10-08**（`639e128`）+ **9 个** NPU/RX/EHT 补丁。
- pbodq2 相对 fanboy OpenW1700k main 已经 **2053 commits ahead**；target 补丁数也不一样：
  - fanboy OpenW1700k：152
  - pbodq2：170
  - naoki66：186
- **naoki66 / lowaka** 用的是 `openwrt/mt76` **2026-10-08**（`467ae2e`）+ **5 个** NPU/RX 补丁，两者完全一致。

所以 pbodq2 的 mt76 是一个**深度定制的 fork**，不是 fanboy 上游的简单同步；用户说的“脱离上游很多”是成立的。

## 10-28 相比 10-08 多了什么

`openwrt/mt76` 从 `467ae2e`（10-08）到 `0936617`（10-28）共有 **83 个提交**，其中和当前 iPhone 问题高度相关的有：

- `wifi: mt76: honour the more-data flag when releasing buffered frames`
- `wifi: mt76: mt7996: take the link of an 802.3 frame from its station`
- `wifi: mt76: mt7996: keep scan and ROC links out of the MLD group`
- `wifi: mt76: mt7996: factor the vif link teardown out of remove_interface`
- `wifi: mt76: mt7996: clamp VHT/HE beamforming NSS`
- `wifi: mt76: mt7996: clamp EHT beamforming NSS`
- `wifi: mt76: mt7996: stop radios after add failure`
- `wifi: mt76: mt7996: only remove a BMC STA_REC that was added`
- 一大批 MCU/EEPROM/固件响应长度校验

这些正好覆盖：

- 省电/buffered frames 的 more-data 处理
- MLO/MLD 的 link 选择和 teardown
- beamforming NSS
- add failure 后停止 radio

也就是说 10-28 很可能比 10-08 更适合解决 iPhone 的“连一会儿后下行断流”。

## 当前构建

当前 run `41285585828` 用的是：

```text
openwrt/mt76 467ae2e (10-08) + naoki66/lowaka 的 5 个补丁
```

这是**已知在 lowaka 固件上 iPhone 正常**的组合，所以先让它构建/测试。

## 下一步候选

如果当前 10-08 + 5 补丁仍然有问题，下一版建议：

```text
openwrt/mt76 0936617 (10-28) + naoki66/lowaka 的 5 个 NPU/RX 补丁
```

需要先验证这 5 个补丁能否 apply 到 10-28（部分 NPU 代码可能已被 83 个上游提交改动）。如果不能：

1. 只保留能 apply 的补丁
2. 或使用 fanboy OpenW1700k 的 10-28 + target NPU 补丁
3. 或从 10-28 上游提交里 cherry-pick 关键 fix（more-data、MLD link、beamforming NSS）到当前 10-08 + NPU 组合

## 决策

- 继续使用 **pbodq2 作为 target/kernel 基线**
- mt76 先测试 naoki66/lowaka 的 10-08 + 5 补丁
- 如果不行，再切 10-28 + NPU 补丁，而不是继续在 pbodq2 的 fanboy fork 上打补丁
