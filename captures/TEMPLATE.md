# <Controller> — <short capture title>

One-paragraph summary: which bus, and the path walked (discovery -> sniffer -> decoded frames).
Replace every <bracketed> field and the example rows below with your own data.

## Contributors

Credit everyone who captured, reverse-engineered, or verified this data. Add yourself when you
extend it. Link GitHub profiles where possible.

| Contributor | Contribution | Link |
|---|---|---|
| Jane Doe (@janedoe) | Capture, CRC verification, 0x0103 display decode | https://github.com/janedoe |

## Capture metadata

| | |
|---|---|
| Controller | <vendor / model> |
| Firmware revision(s) | <from the on-panel Diagnostic screen and/or the version frame> |
| Board | <ESP board + RS485 adapter> |
| ESPHome | <x.y.z> |
| rs485_frame | <component version or git ref> |
| Bus | <baud, data bits, parity, stop bits> |
| Date | <YYYY-MM-DD> |
| Status | <tested-on-hardware \| UNTESTED-draft> |

> **Scrub before publishing.** ASCII display frames can carry custom pool/spa names or schedule
> text you typed into the controller. Redact them. (Bus traffic never carries WiFi credentials.)

## Step 1: Discovery

<paste the generic/discovery.yaml log; note what each line teaches>

## Step 2: Sniffer — idle bus

<paste the sniffer_stats table; identify the keep-alive and the status/display cluster>

## Step 3: Sniffer — under commands

<paste; call out frame types that appear only under user input>

## Decoded frames

<per frame type: raw bytes, payload-relative offsets, and a hand-verified CRC>

## What to do next

<point at the device config built from this capture, e.g. hayward/aqualogic/example-device.yaml>

---

A complete, real worked example following this template:
[hayward-aqualogic-20260529.md](hayward-aqualogic-20260529.md).
