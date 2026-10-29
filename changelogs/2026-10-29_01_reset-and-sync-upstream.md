# 2026-10-29 — 放弃旧本地副本，重新浅克隆并同步上游

## 上游基点

- 仓库：`pbodq2/OpenWRT-W1700K-XR1710G`
- 分支：`ubi2`
- 同步到：`22d02fe`（wifi: mt76: npu: fix the rx descriptor ownership handshake）

## 背景

旧本地副本为 shallow clone，分叉点已不在上游当前历史中（上游改写过历史），本地仅有一个自定义提交：

```text
da6019e add passwall feeds (Openwrt-Passwall) for custom package support
作者：kaiwater <kaiwater@Gmail.com>
时间：2026-10-25 04:31:32 +0800
```

该提交只修改了 `feeds.conf.default`，新增两行：

```diff
+src-git passwall_packages https://github.com/Openwrt-Passwall/openwrt-passwall-packages.git;main
+src-git passwall_luci https://github.com/Openwrt-Passwall/openwrt-passwall.git;main
```

与此同时上游比旧分叉点领先约 306 个提交，且旧基点与上游已无共同祖先，无法直接 fast-forward。

## 操作

1. 旧目录改名为 `/Users/bob/OpenWRT-W1700K-XR1710G.bak`（暂留，确认后删除）。
2. 从 `kaiwater/OpenWRT-W1700K-XR1710G` 重新浅克隆 `ubi2` 分支：

   ```sh
   git clone --depth 1 --branch ubi2 https://github.com/kaiwater/OpenWRT-W1700K-XR1710G.git
   ```

3. 远程配置：

   ```text
   origin   https://github.com/kaiwater/OpenWRT-W1700K-XR1710G.git   （自己的 fork）
   upstream https://github.com/pbodq2/OpenWRT-W1700K-XR1710G.git      （上游）
   ```

4. `git reset --hard origin/ubi2` + `git clean -fdx`，丢弃全部本地改动；`ubi2` 跟踪 `origin/ubi2`。

## 结果

- 本地与自己的 fork、上游三者一致，HEAD 均为 `22d02fe`。
- 旧的自定义提交 `da6019e` 已丢弃（本文件保留了完整内容，需要时可重新应用）。
- 当前没有任何自定义改动。

## 待定

- 是否重新导入 passwall feeds（上面的两行）。
- 软件包建议列表 `~/Downloads/1710 (3).config` 尚未导入。计划只保留 `luci-app-*` 及其显式依赖，排除 PON、内核模块、驱动、固件、board 等系统级选项。

## 回滚

- 恢复旧副本：`mv /Users/bob/OpenWRT-W1700K-XR1710G.bak /Users/bob/OpenWRT-W1700K-XR1710G`
- 恢复 passwall feeds：把上面两行加回 `feeds.conf.default`。
