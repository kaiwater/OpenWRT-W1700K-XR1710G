# 2026-10-30 — 镜像已生成，仅 manifest 阶段失败

## 上游基点

- `22d02fe`（工作树 `32c741d`）

## 第二次编译结果

Run `40952689204`（2026-10-29 20:51 → 2026-10-30 00:14，约 3h23m）：

- `ccache` 命中率 **99.51%**（29008 / 29150），toolchain 和 C/C++ 包基本全命中
- 镜像实际已经生成成功：

  ```text
  bin/targets/airoha/an7581/openwrt-airoha-an7581-gemtek_xr1710g-ubi-squashfs-sysupgrade.itb
  bin/targets/airoha/an7581/openwrt-airoha-an7581-gemtek_xr1710g-ubi-initramfs-recovery.itb
  ```

- 失败点只有 `Image/Manifest`：

  ```text
  IPKG_INSTROOT=.../build_dir/.../root.orig-airoha ... apk --root .../root.orig-airoha list --manifest
  ERROR: Unable to open root: No such file or directory
  ```

## 原因

`Image/Manifest` 固定使用 `TARGET_DIR_ORIG`（`root.orig-<board>`）。这个目录是 per-device rootfs 流程里的“原始 rootfs”，而当前配置关闭了 `CONFIG_TARGET_PER_DEVICE_ROOTFS`，实际用于生成镜像的是 `TARGET_DIR`（`root-<board>`），`root.orig-<board>` 不存在。

## 改动

`include/image.mk`：

```make
MANIFEST_DIR := $(if $(TARGET_PER_DEVICE_ROOTFS),$(TARGET_DIR_ORIG),$(TARGET_DIR))

define Image/Manifest
	$(call apk,$(MANIFEST_DIR)) ...
```

- 关闭 per-device rootfs：用 `TARGET_DIR`，与实际镜像一致；
- 开启 per-device rootfs：仍用 `TARGET_DIR_ORIG`。

## 现状

- 失败 run 的 `xr1710g-firmware` artifact 已上传（160 MB），镜像本身可用；
- 本次修复后重新触发，预期只会在 manifest 处不再失败。

## 回滚

```sh
git checkout 22d02fe -- include/image.mk
```
