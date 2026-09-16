# Roadmap / known issues

Tracking issues found testing against a real Canon EOS 5D Mark II over
USB-OTG, so we don't lose track of them between sessions.

## Open

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

- **Live view zoom limited to 1x on the 5D2.** The camera's on-body
  Live View supports 5x/10x zoom, but the app's magnification control
  only ever shows 1x. Likely the 5D2's zoom property doesn't advertise
  5/10 as available values the same way newer bodies do (this would be
  correct capability-gating, not a bug) -- needs checking against the
  camera's actual advertised property values before doing anything.
  Lowest priority of the open items.

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
