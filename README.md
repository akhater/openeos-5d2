# OpenEOS — 5D Mark II phone tether

Goal: control a Canon EOS 5D Mark II from an Android phone over a USB-OTG
cable — live view, shutter, exposure (ISO/Tv/Av/WB) — no Wi-Fi, no CCAPI
(the 5D2 predates both).

## Approach

This combines two upstream open-source projects, both Apache-2.0:

- **[open-eos-control](https://github.com/js051/open-eos-control)** —
  modern Android app (Gradle/Kotlin/Compose) with a working USB/PTP
  transport, live view UI, and connection flow. Its Canon property/command
  tables are pinned to the EOS R6 Mark III, which is why it currently
  rejects several 5D2 operations as "not advertised."
- **[remoteyourcam-usb](https://github.com/michaelzoech/remoteyourcam-usb)**
  — an older (2013) Android app built and used against 5D2-era Canon EOS
  bodies. Its `EosCamera`/`EosConstants`/`ptp/commands/eos` code is
  reference evidence for how this camera generation's Canon PTP vendor
  extension actually behaves.

Plan: keep open-eos-control's app shell and transport layer, and extend/
replace its Canon capability tables with 5D2-era values sourced from
remoteyourcam-usb, so the modern UI stops gating features the 5D2 actually
supports.

## Layout

- `vendor/open-eos-control` — unmodified upstream snapshot, for reference
  and diffing. Do not edit.
- `vendor/remoteyourcam-usb` — unmodified upstream snapshot, for reference
  and diffing. Do not edit.
- `android/` — our working app, seeded from `vendor/open-eos-control/android`
  and modified here.

## License

Apache License 2.0 (same as both upstream projects). See `LICENSE`.
Upstream copyright notices are preserved under `vendor/*/NOTICE.txt` and
`vendor/*/LICENSE.txt`.
