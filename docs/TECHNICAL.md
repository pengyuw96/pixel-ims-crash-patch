# Technical notes — how the crash was fixed

## Background

Pixel IMS uses a `BrokerInstrumentation` component started through Shizuku. That process obtains a binder to `android.app.IActivityManager`, calls `startDelegateShellPermissionIdentity` so it can act with shell permissions, writes CarrierConfig overrides, then calls `stopDelegateShellPermissionIdentity`.

On Android 17 (Pixel 10 Pro / `blazer`), the hidden AIDL signatures changed:

```text
// stop — before
void stopDelegateShellPermissionIdentity();

// stop — Android 17
void stopDelegateShellPermissionIdentity(int uid);
```

`startDelegateShellPermissionIdentity(int, String[])` was already used by upstream 1.3.2 (passing `null` for “all shell permissions”). Only **stop** call sites were stale.

## Symptom

Any UI toggle that triggers `applyConfig` / `clearConfig` → immediate crash:

```text
java.lang.NoSuchMethodError:
  No interface method stopDelegateShellPermissionIdentity()V
  in class Landroid/app/IActivityManager;
```

## Fix (smali)

File (apktool layout):

`smali_classes3/dev/bluehouse/enablevolte/BrokerInstrumentation.smali`

Replace each no-arg stop with:

```smali
invoke-static {}, Landroid/system/Os;->getuid()I
move-result <reg>
invoke-interface {<iam>, <reg>}, Landroid/app/IActivityManager;->stopDelegateShellPermissionIdentity(I)V
```

Four sites: success + catch paths for both `applyConfig` and `clearConfig`.

See [`../patches/BrokerInstrumentation-stopDelegate-uid.patch`](../patches/BrokerInstrumentation-stopDelegate-uid.patch).

## Build / packaging used for the Release APK

1. Start from official **1.3.2** APK (`dev.bluehouse.enablevolte`).
2. `apktool` decode → edit smali → rebuild.
3. `uber-apk-signer` (zipalign + resign with a local **Android Debug** keystore).

Therefore the Release APK is **not** Play Store–signed; uninstall the store build before installing.

## Validation checklist (device)

- Logcat contains `applyConfig` then `applyConfig done` without process death.
- Tombstone / dropbox crash buffer empty for the package.
- Toggle state persists / flips false ↔ true as expected.

## Out of scope

- `TransactionTooLargeException` from oversized `rememberSaveable` state (upstream issue).
- Full Gradle rebuild from Kotlin sources (would be preferred long-term; this release is a binary smali patch for immediate Android 17 usability).
