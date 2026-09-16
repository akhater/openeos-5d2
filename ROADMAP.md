# Roadmap / known issues

Tracking issues found testing against a real Canon EOS 5D Mark II over
USB-OTG, so we don't lose track of them between sessions.

## Open

- **Live view zoom limited to 1x on the 5D2.** The camera's on-body
  Live View supports 5x/10x zoom, but the app's magnification control
  only ever shows 1x. Likely the 5D2's zoom property doesn't advertise
  5/10 as available values the same way newer bodies do (this would be
  correct capability-gating, not a bug) -- needs checking against the
  camera's actual advertised property values before doing anything.
  Lower priority than the item below.

- **Needs re-testing on real hardware**: the fix below removes the
  behavior that was firing repeated writes from scrolling, which was
  the leading theory for the live-view freeze / `PtpProtocolException:
  Android USB bulk write failed on endpoint 0x2 (result -1)` crash.
  Not yet confirmed the freeze is actually gone -- only that its known
  trigger (rapid writes from flicking) is removed. If it recurs even
  with deliberate single taps, the next suspect is the live-view
  polling loop overlapping with a slow property write on the shared
  `PtpSession.mutex` (see `startLiveViewLoopIfNeeded` in
  `android/app/src/main/java/dev/openeos/control/ui/CameraViewModel.kt`).

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
