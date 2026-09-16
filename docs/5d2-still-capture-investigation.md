# Investigation: "still capture is not advertised" on 5D Mark II

## Symptom
Connecting a 5D Mark II over USB, the app recognizes the camera but
reports still capture as not advertised.

## What the code actually requires

`android/app/src/main/java/dev/openeos/control/data/UsbPtpCameraBackend.kt`
gates `CameraFeature.STILL_CAPTURE` on:

```kotlin
if (info.supports(PtpOperationCode.INITIATE_CAPTURE) || supportsCanonRelease) {
    add(CameraFeature.STILL_CAPTURE)
}
```

`supportsCanonRelease` comes from
`android/app/src/main/java/dev/openeos/control/data/CanonEosPtp.kt`:

```kotlin
fun isCanonEos(info: PtpDeviceInfo): Boolean = info.vendorExtensionId == VENDOR_EXTENSION_ID // 0x0000000B

fun supportsRemotePreparation(info: PtpDeviceInfo): Boolean =
    isCanonEos(info) && remotePreparationOperations.all(info::supports) // 0x9114, 0x9115, 0x9116

fun supportsRemoteRelease(info: PtpDeviceInfo): Boolean =
    supportsRemotePreparation(info) &&
        info.supports(CanonEosOperationCode.REMOTE_RELEASE_ON) &&  // 0x9128
        info.supports(CanonEosOperationCode.REMOTE_RELEASE_OFF)    // 0x9129
```

So still capture needs: vendor extension ID `0x0B`, plus operations
`0x9114`, `0x9115`, `0x9116`, `0x9128`, `0x9129` all present in the
camera's `GetDeviceInfo` `OperationsSupported` list.

## Cross-check against remoteyourcam-usb (proven on 5D2-era bodies)

`vendor/remoteyourcam-usb/src/com/remoteyourcam/usb/ptp/PtpConstants.java`
defines the identical opcodes:

```java
public static final int EosSetPCConnectMode = 0x9114;
public static final int EosSetEventMode     = 0x9115;
public static final int EosEventCheck       = 0x9116;
```

and `EosRemoteReleaseOn` / `EosRemoteReleaseOff` at `0x9128` / `0x9129`
(same file). Critically, `PtpCamera.java` queues exactly one
`GetDeviceInfoCommand`, **before** `openSession()` runs
(`PtpCamera.java:124`), and `EosCamera.onOperationCodesReceived(...)`
reads the vendor capture/live-view/bulb operations straight out of that
same pre-session response. No second "re-fetch DeviceInfo after PC mode"
step exists anywhere in that codebase.

This means: on a real 5D2-class body, the very first `GetDeviceInfo`
response already advertises the full Canon remote-capture operation set,
with no extra handshake required — and that 2013 app relied on and shipped
against exactly that behavior.

## Conclusion

The 5D2 almost certainly *does* advertise `0x9114/0x9115/0x9116/0x9128/0x9129`.
The "not advertised" result is more likely one of:

1. The Android USB transport truncating or mis-parsing the
   `OperationsSupported` array in `GetDeviceInfo` for this camera
   (different array length/encoding than the R6 Mark III response this
   code was validated against).
2. `vendorExtensionId` being read from the wrong offset/width for this
   camera's `GetDeviceInfo` response shape, making `isCanonEos()` false
   even though the operations list is otherwise fine.
3. A genuinely different (older) `GetDeviceInfo` response shape for the
   5D2 that the parser doesn't handle.

## Resolved: confirmed root cause

Got a real diagnostic report from a 5D Mark II over USB (see
`docs/5d2-diagnostic-report-2026-09-16.txt`, 130 advertised operations,
count verified against `advertisedCommandCount=130`).

The camera advertises `0x9114`, `0x9115`, `0x9116`, `0x9128`, `0x9129`,
`0x9154`, `0x9160`, `0x9153`, `0x9110`, `0x9157` — the *entire* Canon EOS
vendor operation set assumed above. Manufacturer/model report as
`Canon Inc. Canon EOS 5D Mark II`.

But the report's `protocolVersions` line reads:

```
protocolVersions=PTP 1.00, vendor 0x00000006/200
```

`vendorExtensionId = 0x00000006` -- **not** `0x0000000B` (11), the value
`CanonEosPtp.isCanonEos()` hardcoded as the only accepted Canon vendor
extension ID. `0x00000006` is the PTP-registered Microsoft/MTP vendor
extension ID; the 5D2 apparently self-reports MTP compatibility mode in
this field even though every Canon vendor operation is present in the
same `OperationsSupported` list.

Since `isCanonEos()` gated `supportsRemotePreparation`,
`supportsRemoteRelease`, `supportsAutofocus`, `supportsLiveView`, and
every other Canon-specific capability check, this single hardcoded
comparison was the root cause of every "not advertised" rejection --
still capture, autofocus, live view, event polling, exposure control,
etc. all cascade from it.

This matches remoteyourcam-usb's own approach: it never reads the PTP
`vendorExtensionId` field at all. It decides Canon-vs-Nikon purely from
the **USB descriptor vendor ID** (`device.getVendorId() ==
PtpConstants.CanonVendorId`, in `PtpUsbService.java`), which is exactly
what open-eos-control's own `UsbPtpDiagnostics.kt` already does for the
pre-connection USB scan (`isCanon = vendorId == CANON_USB_VENDOR_ID`).
Only the later PTP-session-level `CanonEosPtp.isCanonEos()` check
reinvented this using the unreliable PTP-level field instead.

## Fix

`android/app/src/main/java/dev/openeos/control/data/CanonEosPtp.kt`:

```kotlin
fun isCanonEos(info: PtpDeviceInfo): Boolean =
    info.vendorExtensionId == VENDOR_EXTENSION_ID || info.manufacturer.contains("Canon", ignoreCase = true)
```

Keeps the original check (harmless for cameras that do report `0x0B`)
and adds the manufacturer-string fallback, which the 5D2's DeviceInfo
response already provides (`manufacturer = "Canon Inc."`). Still needs
a real-camera rebuild/retest to confirm `STILL_CAPTURE` and friends
move from `planned` to `supported`.
