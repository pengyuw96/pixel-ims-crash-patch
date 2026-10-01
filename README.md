# Pixel IMS Android 17 Crash Patch / Pixel IMS Android 17 闪退补丁

English | [中文](#中文)

Unofficial crash fix for **[Pixel IMS](https://github.com/kyujin-cho/pixel-volte-patch)** (`dev.bluehouse.enablevolte`) **v1.3.2** on **Android 17**.

Toggling VoLTE / IMS switches no longer crashes with `NoSuchMethodError` on `IActivityManager.stopDelegateShellPermissionIdentity`.

> **This is not an official release.** Binary is redistributed under the upstream **GPL-3.0** license, marked as modified. Download the APK only from [Releases](https://github.com/pengyuw96/pixel-ims-crash-patch/releases).

## Problem

On Android 17, Pixel IMS 1.3.2 crashes whenever any CarrierConfig / IMS toggle is flipped. The app's `BrokerInstrumentation` (started via Shizuku as shell) still calls the pre–Android 17 binder signature:

```text
IActivityManager.stopDelegateShellPermissionIdentity()   // no-arg — removed on Android 17
```

Android 17 changed hidden API signatures (verified against device `framework.jar` / `services.jar`):

| Method | ≤ Android 16 | Android 17 |
|--------|--------------|------------|
| `startDelegateShellPermissionIdentity` | `(int, Bundle)` | `(int, String[])` — `null` still means “delegate all shell permissions” |
| `stopDelegateShellPermissionIdentity` | `()` | `(int uid)` |

Stock 1.3.2 already uses the new `startDelegate…(I[Ljava/lang/String;)V` form. The **stop** calls still use the old no-arg form → `NoSuchMethodError` → process death.

## What this patch changes

In `BrokerInstrumentation.smali`, **four** `stopDelegateShellPermissionIdentity()V` call sites (success + exception paths in `applyConfig` / `clearConfig`) were updated to:

```smali
invoke-static {}, Landroid/system/Os;->getuid()I
move-result v0
invoke-interface {v1, v0}, Landroid/app/IActivityManager;->stopDelegateShellPermissionIdentity(I)V
```

No other logic, resources, or permissions were changed. Details: [`docs/TECHNICAL.md`](docs/TECHNICAL.md). Unified smali diff: [`patches/BrokerInstrumentation-stopDelegate-uid.patch`](patches/BrokerInstrumentation-stopDelegate-uid.patch).

## Tested device

| Item | Value |
|------|--------|
| Device | Google Pixel 10 Pro (`blazer`) |
| OS | Android 17 (SDK 37) |
| Upstream base | Pixel IMS **1.3.2** ([kyujin-cho/pixel-volte-patch](https://github.com/kyujin-cho/pixel-volte-patch/releases/tag/1.3.2)) |
| Date | 2026-10-01 (Asia/Shanghai) |

Verification: flip VoLTE → logcat shows `applyConfig` / `applyConfig done`, crash buffer empty, toggle state truly flips false↔true.

Other devices / builds are **unverified**.

## Install notes

1. Requires a working **Shizuku** session (shell).
2. Uninstall stock Pixel IMS first if signatures differ, then install the Release APK (debug-signed with a local Android Debug keystore — **not** Play Store signed).
3. Re-grant Shizuku authorization after reinstall (data is cleared).
4. Prefer the Release asset SHA-256 printed in the release notes.

### Known upstream issue (not fixed here)

Occasional `TransactionTooLargeException` when the app goes to background (~950 KB saved UI state vs Binder 1 MB limit) from `rememberSaveable` holding `SubscriptionInfo` + icon Bitmaps (`HomeActivity`). Present in stock 1.3.2 as well; needs a source-level fix.

## Credits / Attribution

- Upstream: **[kyujin-cho/pixel-volte-patch](https://github.com/kyujin-cho/pixel-volte-patch)** (Pixel IMS) — authors and contributors.
- Runtime dependency: [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku) (and any device-specific Shizuku fixes you use).
- Patch author: [pengyuw96](https://github.com/pengyuw96) (binary smali patch on top of v1.3.2).

## License

Same as upstream: **[GNU General Public License v3.0](LICENSE)**.

This repository distributes a **modified** form of Pixel IMS. Corresponding Source for the modification is the patch under `patches/` plus the documentation herein; the unmodified upstream source remains at [kyujin-cho/pixel-volte-patch](https://github.com/kyujin-cho/pixel-volte-patch) tag `1.3.2`.

---

## 中文

针对 **[Pixel IMS](https://github.com/kyujin-cho/pixel-volte-patch)**（包名 `dev.bluehouse.enablevolte`）**v1.3.2** 在 **Android 17** 上切换任意 IMS/VoLTE 开关即闪退的问题，提供非官方补丁 APK。

> **非官方发布。** 按上游 **GPL-3.0** 再分发修改版。请仅从 [Releases](https://github.com/pengyuw96/pixel-ims-crash-patch/releases) 下载 APK；本仓库 **git 树不含 APK**。

### 问题

Android 17 调整了隐藏 API `android.app.IActivityManager` 的方法签名。`BrokerInstrumentation`（经 Shizuku 以 shell 启动、用于写 CarrierConfig）仍调用旧的无参 `stopDelegateShellPermissionIdentity()`，触发 `NoSuchMethodError` 闪退。`startDelegate…` 在 1.3.2 中已是新签名，无需改动。

### 补丁做了什么

将 `applyConfig` / `clearConfig` 成功与异常路径上共 **4 处** `stopDelegateShellPermissionIdentity()V` 改为传入 `Os.getuid()` 的 `(I)V` 形式。未改动其他逻辑、资源或权限。详见 [`docs/TECHNICAL.md`](docs/TECHNICAL.md)。

### 测试设备

| 项目 | 值 |
|------|-----|
| 设备 | Pixel 10 Pro（`blazer`） |
| 系统 | Android 17（SDK 37） |
| 基础上游 | Pixel IMS **1.3.2** |
| 日期 | 2026-10-01（Asia/Shanghai） |

其他机型/版本未验证。安装前需可用的 Shizuku；因使用本地 debug 证书重签，需先卸载官方包并重新授权。

### 致谢与许可

- 上游：[kyujin-cho/pixel-volte-patch](https://github.com/kyujin-cho/pixel-volte-patch)
- 依赖：[RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)
- 许可：与上游相同的 [GPL-3.0](LICENSE)；本仓库提供修改说明与 smali 补丁作为对应源码的一部分。
