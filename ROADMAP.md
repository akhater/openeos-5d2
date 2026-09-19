# Roadmap / known issues

Tracking issues found testing against a real Canon EOS 5D Mark II over
USB-OTG, so we don't lose track of them between sessions.

## Open

- **Priority: expose the 5D Mark II's native Live View Exposure Simulation.**
  Canon documents that the 5D2 can switch Live View between a standard
  brightness display and Exposure Simulation, where the preview closely
  reflects the final exposure. This is preferable to artificially brightening
  or darkening frames in the app because it lets the camera account for ISO,
  shutter speed, aperture, exposure compensation, and its own processing.
  The USB path now recognizes the separate Canon `ExposureSimMode` property
  (`0xD1B7`) and exposes it as an On/Off control in the Live View settings
  sheet when the camera reports it. The implementation still needs validation
  on the real 5D2, including comparison with the JPEG returned by
  `GetViewFinderData`, because the app cannot synthesize the camera's exact
  processing or exposure-meter reading.
  The camera's on-body Exposure Simulation setting should also be compared
  with the JPEG returned by `GetViewFinderData` to confirm whether the USB
  preview already follows it. If the setting is not remotely writable, keep
  the fallback as a documented camera-side setting rather than inventing an
  inaccurate synthetic preview.

- **Priority: add composition overlays beyond aspect-ratio guides.** Add
  overlay-only guides for Rule of Thirds, Golden Ratio (initially a grid),
  and Diagonals. This should be a low-risk UI-only change: it uses the same
  Live View overlay layer as the existing frame guides and does not send any
  camera commands or affect the USB transaction path.

- **Add multi-shot capture for noise reduction and averaging.** Let the user
  request a bounded number of photos with the same settings, with a deliberate
  interval, progress, cancellation, and per-shot completion/media tracking.
  The existing still-capture path makes the capture sequence feasible; merging
  the resulting JPEG/RAW files into an averaged or median-denoised image is a
  separate follow-up and must account for alignment, storage, and RAW memory
  cost. This means repeated same-scene capture, not the camera's separate
  creative multiple-exposure mode.

- **Add exposure-bracketing workflows.** The Canon AEB property and value
  labels already exist in the USB model, and the 5D2 supports three successive
  bracketed shots across its documented range. What is missing is a safe app
  workflow to configure/verify AEB, trigger the sequence, wait for every image
  event, and present the resulting set. A software-controlled bracket remains
  a fallback for bodies that do not expose usable AEB, but it would require
  changing exposure between shots and therefore needs stricter serialization.

- **Add focus-stacking capture workflows.** The USB backend already exposes
  Canon manual focus drive with Near/Far steps 1-3, so an app-orchestrated
  stack is technically possible: capture, move focus, settle, capture, and
  repeat. The 5D2 does not provide the newer camera-native focus-bracketing
  contract, so this needs a user-defined start position, direction, step,
  count, and settle delay. Producing a finished stack in the app is a separate
  alignment/merge feature; the first version can safely deliver the source
  sequence.

- **`PtpProtocolException: Android USB bulk write failed on endpoint
  0x2 (result -1)` still happens with deliberate single taps, not just
  rapid flicks.** The tap-to-apply fix (below) removed the
  rapid-repeated-writes trigger, but retesting on the real 5D2 showed
  the crash is still reproducible. New, more specific data point: **the
  first setting change after connecting succeeds cleanly; it's the
  second one that triggers the failure.** That doesn't fit a simple
  "live view polling collides with a write" theory (which would predict
  *any* write can fail, not specifically the second one) -- something
  is stateful across writes. Checked and ruled out: every camera
  command (including live-view frame fetches) already funnels through
  one shared lock (`PtpSession.mutex` in
  `android/app/src/main/java/dev/openeos/control/data/PtpProtocol.kt`);
  there's no code path in `UsbPtpCameraBackend.kt` that bypasses it and
  calls the transport directly, so this isn't the app racing itself in
  an obvious way.

  Two remaining theories, not yet distinguished:
  1. Hardware/power: the 5D2 draws more current than the modern camera
     this app was built against, and sustained live-view streaming is
     real load. A marginal USB-OTG cable/adapter could produce a
     generic `-1` bulk-transfer failure under that load, especially on
     a second sustained write shortly after the first. Not a bug in
     this app at all if so.
  2. Something left over from the first write's completion (event
     draining, a stale property-change event, a timing assumption)
     puts the session into a state that only the *next* write trips
     over -- would explain "first works, second doesn't" better than
     the power theory does, but nothing concrete identified yet.

  Next diagnostic step: reproduce with live view **off** entirely
  (see the live-view-as-opt-in item below -- this needs that to test
  cleanly) and see if the first-works/second-fails pattern still holds
  with zero live-view traffic on the bus. That cleanly separates
  "live view involvement" from "just the Nth write in a session."

- **Live view auto-starts as soon as the camera connects over USB; it
  should be a separate, explicit opt-in action instead.** Requested
  directly, and it also makes debugging the above easier: with live
  view off by default, we can test "does writing settings alone (no
  live view at all) reproduce the crash" cleanly, instead of every
  test run having live view traffic as a confounding variable. Also a
  reasonable default on its own merits for an old, power-hungry camera
  on USB-OTG -- don't pull sustained live-view current unless the user
  actually asked for it.

- **Low priority: add 10x Live View zoom on the 5D2.** The current release
  supports 1x and 5x. The model supports 5x/10x on the camera, and the app
  already has an `X10` model value, but the USB capability list deliberately
  advertises only the values validated so far. Add 10x only after sending the
  Canon zoom command with value `10` has been validated on the real body and
  its readback/failure behavior is known.

## Fixed

- **Wheel-style settings dial (`ExposureDial`, ISO/Tv/Av) committed a
  real camera write on scroll-settle instead of requiring an explicit
  tap.** Flicking through values to browse them would commit whichever
  one the fling happened to land on, with no confirm step -- and rapid
  repeated flicks could fire several writes back-to-back onto the
  single shared USB connection. The 5D2 responds slowly enough
  (relative to the modern camera this app was built against) that a
  backlog of queued writes showed up as a multi-second live-view freeze,
  and in at least one observed case as a hard failure:
  `PtpProtocolException: Android USB bulk write failed on endpoint 0x2
  (result -1)`.

  Each dial item already had its own tap handler that applies
  deliberately and correctly; the only problem was a second effect in
  `android/app/src/main/java/dev/openeos/control/ui/CameraControlScreen.kt`
  (`LaunchedEffect(listState, values) { snapshotFlow { isScrollInProgress } ... }`)
  that *also* auto-applied whatever was centered every time scrolling
  stopped. Removed that effect entirely -- scrolling now only browses;
  tapping a value (centered or not) is what applies it. This was a
  UX fix requested directly ("should click on the one I want, not just
  where it falls"), and it also removes the only source of
  rapid-fire repeated writes we'd identified, which was the leading
  theory for the freeze/crash above.
