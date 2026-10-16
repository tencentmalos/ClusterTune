# Odin 3 minimum permission recovery

Validated on 2026-10-16 against upstream `AurelioB/ClusterTune` commit
`0ba154d` (version 1.3.0), on branch `codex/odin3-profile-recovery`.

The maintained branch is in [tencentmalos/ClusterTune](https://github.com/tencentmalos/ClusterTune/tree/codex/odin3-profile-recovery).
The original project remains [AurelioB/ClusterTune](https://github.com/AurelioB/ClusterTune).

## Observed failure

The installed 1.3.0 repeatedly failed profile applications with:

```
CPU minimum protection changed externally for policy0; release the profile before retrying
```

The root host still owned a same-boot minimum protection record for policy0 and
policy6, with original permissions 0664. Both actual minimum nodes were now
0644 and both CPU maximums had returned to stock (3532800 / 4320000 kHz).
The process was alive; this was not an application crash. The observations do
not identify the process that changed the permissions.

`HostMinimumProtection` rejected every later underclock while its stale ownership
record remained. Repeating Apply could not recover it.

## Change

On an explicit profile apply, an owned node whose permissions are unchanged
continues to be reused without rewriting its minimum. If permissions are now
writable, reacquire protection and reset the supported floor, journaling the
current permissions before any sysfs mutation. Stock/release restores this new
baseline instead of replaying obsolete permissions. If the apply fails,
reacquired nodes are rolled back like new acquisitions. A journal failure leaves
sysfs untouched and the operation retryable. A different read-only mode still
refuses takeover.

This repairs application after a reset; it does not introduce continuous polling
or identify the external writer.

## Validation

- Five regression cases added. The original implementation failed four of them;
  the read-only-conflict case already passed.
- All 27 minimum protection unit tests pass after the fix.
- Full unit run: 587 tests, 573 passed, 14 failed in existing shell-dependent
  tests on Windows (`HostLaunchScriptTest`, `HostProtocolTest`,
  `HostTelemetrySourceTest`, which require `sh` / `/bin/sh`).
- `lintDebug`: zero errors, 58 warnings. Debug and instrumentation APKs
  built successfully.
- Opt-in device regression `writableOemReset_reappliesWithoutStockRoundTrip`
  passed: three writable permission resets (0644, 0664, 0644), stock ceiling
  resets, successful reapplication without selecting Stock first, ten seconds of
  held state, and correct Stock permission release. Cleanup restored the initial
  ceilings and modes.
- Existing AVG profile was migrated and verified: policy0 2745600 kHz, policy6
  3072000 kHz, GPU 1100000000 Hz.
- Three real screen off/on cycles kept AVG CPU caps and protected permissions.
- A subsequent one-minute observation (six samples ten seconds apart) retained
  both AVG CPU caps and the 1100000000 Hz GPU cap in every sample.
- Boot restore was exercised with a targeted BOOT_COMPLETED broadcast, not a
  physical reboot. Long-duration gaming and real reboot behavior remain untested.

## Build and install the fix

Use the configured JDK 21 daemon toolchain and Android SDK 34. From this branch:

```powershell
.\gradlew.bat -I scripts/odin-fix.gradle testDebugUnitTest --tests '*HostMinimumProtectionTest'
.\gradlew.bat -I scripts/odin-fix.gradle lintDebug assembleDebug assembleDebugAndroidTest
```

On Linux/macOS use `./gradlew` instead of `.\gradlew.bat`. The full unit suite is
`testDebugUnitTest` without the `--tests` filter; some existing tests require a
Unix shell and cannot run as-is on Windows.

The checked-in init script only changes debug builds: the app ID becomes
`com.aure.clustertune.odinfix` and the version becomes `1.3.0-odinfix1`. A build
without this script retains upstream's package ID and cannot replace an official
APK signed with a different key. Debug signing uses the builder's configured key;
an APK built on another machine may also require uninstalling a previous local
fix build before installation, so export its profiles first.

Export any profiles you need from the existing app before switching. Select
**Stock** in it (to release its active tuning), then install the fix and disable
the old app, retaining its data. Replace `DEVICE` with the intended ADB serial:

```powershell
adb -s DEVICE install -r app/build/outputs/apk/debug/app-debug.apk
adb -s DEVICE shell am force-stop com.aure.clustertune
adb -s DEVICE shell pm disable-user --user 0 com.aure.clustertune
adb -s DEVICE shell am start -n com.aure.clustertune.odinfix/com.aure.clustertune.MainActivity
```

Import the exported profiles, or create **AVG** with these Odin 3 values:

| Domain | Maximum |
| --- | --- |
| CPU policy0 | 2745600 kHz (2.7456 GHz) |
| CPU policy6 | 3072000 kHz (3.072 GHz) |
| GPU | 1100000000 Hz (1.1 GHz) |

These values describe the tested device's existing custom profile, not a new
global default or a recommendation for other SoCs. Select AVG, enable
last-profile-on-boot, and choose AVG as the sleep profile if the same settings
should be restored across sleep/wake transitions. Enable Notifications and the
desired Accessibility/overlay features in the fix app, and replace the original
Quick Settings tile with the fix app's tile. Disable automatic update checks in
this local build: upstream releases retain their own signature/package identity.

The fix does not automatically migrate app data or change defaults for a fresh
installation. The following deployment record describes the explicitly configured
test device.

## Validated local deployment

The original release signature is unavailable. The locally signed build uses
`com.aure.clustertune.odinfix` and version `1.3.0-odinfix1`, via the excluded
`local-debug/isolated.gradle` init script, now published as
[`scripts/odin-fix.gradle`](../scripts/odin-fix.gradle). The original app is disabled, with its
data intact. Its APK and app data were additionally backed up under `local-debug/`.
Only the original ClusterTune accessibility/tile bindings were migrated; the
Odin Game Assistant accessibility service remains enabled.

The fix build keeps the existing AVG profile, enables last-profile-on-boot and
sleep/wake restoration using AVG, and disables automatic upstream update checks
for this locally signed package. The local setup fixture is not part of the
production APK or the source patch.

Local artifacts (excluded from Git; not required to build from this fork):

- `local-debug/ClusterTune-1.3.0-odinfix1.apk`
- `local-debug/original-1.3.0.apk`, `local-debug/clustertune-before.tar`
- `local-debug/before-errors.log`, `local-debug/baseline-tests.log`
- `local-debug/fixed-build.log`, `local-debug/apk-build.log`
- `local-debug/device-regression.log`, `local-debug/device-regression-report.txt`
- `local-debug/avg-setup.log`, `local-debug/wake-check.log`, `local-debug/avg-soak.log`

## Repeat the live regression

Install the isolated debug and androidTest APKs. Stop any other tuning owner and
select Stock first so minimums are writable. The opt-in test temporarily changes
CPU ceilings and permissions, then restores the initial ceilings and modes in its
cleanup path. It requires a working privileged host. Explicitly opt in:

```powershell
adb -s DEVICE install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk
adb -s DEVICE shell am instrument -w -r -e class com.aure.clustertune.root.host.MinimumProtectionInstrumentationTest#writableOemReset_reappliesWithoutStockRoundTrip -e minimumPermissionReset 1 com.aure.clustertune.odinfix.test/androidx.test.runner.AndroidJUnitRunner
adb -s DEVICE shell run-as com.aure.clustertune.odinfix cat files/minimum-reset-report.txt
adb -s DEVICE uninstall com.aure.clustertune.odinfix.test
```

Check the instrumentation result and report rather than the adb exit code alone;
without the opt-in argument the test is skipped. Reapply AVG after testing. Test
APKs, private app-data backups, raw device logs, signing material, and the local
setup fixture are not published with the source changes.

Building locally used the installed Microsoft JDK 21 and Android SDK 34, with
`gradle/gradle-daemon-jvm.properties` temporarily moved aside because downloading
the pinned JetBrains toolchain failed on this Windows host. That tracked file
was restored unchanged after building.

## Return to upstream

Select Stock in the fix build first, then:

```powershell
adb -s DEVICE shell am force-stop com.aure.clustertune.odinfix
adb -s DEVICE shell pm disable-user --user 0 com.aure.clustertune.odinfix
adb -s DEVICE shell pm enable --user 0 com.aure.clustertune
adb -s DEVICE shell am start -n com.aure.clustertune/.MainActivity
```

Re-enable the original app's accessibility service
and replace the fix tile with its original tile. Original profiles and settings
are still present. Do not enable both tuning apps simultaneously.
