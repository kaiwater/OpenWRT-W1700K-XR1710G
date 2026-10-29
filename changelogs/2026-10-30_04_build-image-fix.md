# 2026-10-30 — 首次 Actions 编译失败与镜像阶段修复

## 上游基点

- `22d02fe`（工作树 `63da24e`）

## 首次编译结果

Run `40857043382`（2026-10-29 10:53 → 15:02，约 4h15m）：

- toolchain、kernel、全部包编译完成
- 在最后的 `target/linux install`（生成镜像）阶段失败
- 控制台只打印了：

  ```text
  ERROR: target/linux failed to build.
  ```

  具体错误被 OpenWrt 的 sub-make 吞掉，`build.log` 里没有细节。

## 本次改动

### 1. 去掉 `CONFIG_TARGET_PER_DEVICE_ROOTFS`

seed 中的这一项来自旧 ImmortalWrt 配置。OpenWrt 主线单设备构建不需要它，而且它会让 initramfs / FIT 镜像走 `gen-cpio.<ROOTFS_ID>` 和 `target-dir-<ROOTFS_ID>` 的 per-device 路径，是镜像阶段失败的重点嫌疑。

已从 `configs/xr1710g-apps.seed` 删除；`make defconfig` 确认保持 `# CONFIG_TARGET_PER_DEVICE_ROOTFS is not set`。

### 2. 失败诊断

workflow 增加：

- `make ... BUILD_LOG=1`：把每个包/镜像子步骤输出写到 `logs/`
- 失败时执行：

  ```sh
  make target/install V=s
  make -C target/linux install V=s
  ```

  输出到 `diagnostics.log`
- `if: always()` 上传 `build.log`、`diagnostics.log`、`logs.tar.gz` 和 `bin/targets/airoha/an7581/` 下的镜像

### 3. ccache 统计修正

`ccache -s` 之前统计的是 `~/.ccache`，所以显示 0.0 GB；实际构建使用的是 `$(TOPDIR)/.ccache`。已在 job env 中固定 `CCACHE_DIR=${{ github.workspace }}/.ccache`。

## 验证

重新触发 workflow。缓存已保存：

- `dl`：约 2.4 GB
- `.ccache`：约 293 MB

第二次编译应从 toolchain 阶段开始命中 ccache。

## 回滚

- 恢复 per-device rootfs：把 `CONFIG_TARGET_PER_DEVICE_ROOTFS=y` 加回 seed。
- 移除诊断：删除 workflow 中 `Diagnose image build` 步骤和 `BUILD_LOG=1`。
