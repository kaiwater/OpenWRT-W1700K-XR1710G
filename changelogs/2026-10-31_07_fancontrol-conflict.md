# 2026-10-31 — 风扇控制包文件冲突

## 上游基点

- `22d02fe`（工作树 `4fbe066`）

## 第四次编译结果

Run `41199907668`（2026-10-31 12:10 → 14:51，约 2h41m）：

- 自动重试按预期执行
- `make package/install V=s` 抓到了真正的错误：

  ```text
  ERROR: luci-app-w1700k-fancontrol-26.304.43814~4fbe066:
    trying to overwrite etc/config/fan
    owned by luci-app-airoha-fancontrol-1.0.0-r1
  ERROR: ... etc/init.d/fan
  ERROR: ... usr/libexec/rpcd/luci.fan
  ERROR: ... www/luci-static/resources/view/fan/settings.js
  ERROR: ... www/luci-static/resources/view/fan/status.js
  ```

## 原因

设备 profile（`gemtek_xr1710g-ubi` 的 `DEVICE_PACKAGES`）自带 `luci-app-w1700k-fancontrol`；旧配置的软件包列表又选了 `luci-app-airoha-fancontrol`。两者安装同一组文件，不能共存。

之前把 `package/install` 判断为“偶发失败”是不准确的；实际是这两个包同时安装时的确定性冲突，只是前面几次 run 没有打印出细节。

## 改动

`configs/xr1710g-apps.seed`：

- 删除 `CONFIG_PACKAGE_luci-app-airoha-fancontrol=y`
- 删除 `CONFIG_PACKAGE_luci-i18n-airoha-fancontrol-zh-cn=y`

保留设备 profile 自带的 `luci-app-w1700k-fancontrol`。

本地 `make defconfig` 验证结果：

```text
CONFIG_PACKAGE_luci-app-w1700k-fancontrol=y
# CONFIG_PACKAGE_luci-app-airoha-fancontrol is not set
```

119 个 seed 符号全部保留。

## 回滚

把上面两行加回 seed（不建议，会再次冲突）。
