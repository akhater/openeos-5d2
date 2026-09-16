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

## Next step (blocked on real data)

Need the actual advertised-operations evidence from a live 5D2 session:
the app's own diagnostics report (README: "Bounded, secret-redacted
capability evidence showing ... advertised commands") lists the raw
`0x----` operation codes the camera returned. Once we have that list, we
can tell definitively whether `0x9114/15/16/28/29` are present (parser
bug) or absent (need the same investigation for a different 5D2-specific
opcode set).
