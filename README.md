# RS485 Frame examples

Ready-to-flash ESPHome configurations for pool and spa controllers that use
DLE-framed RS485 buses. Each configuration uses the
[rs485_frame](https://github.com/esphome/esphome) ESPHome component.

## Status note

`rs485_frame` is not yet in the ESPHome repository. Each example includes an
`external_components` block that pulls it from the
[staging branch](https://github.com/b3nj1/esphome/tree/framed_rs485). Once
it is merged into an official ESPHome release, remove that block.

## Hardware

### Recommended: Waveshare ESP32-S3-RS485-CAN

The ~$20 USD direct / ~$25 on Amazon [Waveshare ESP32-S3-RS485-CAN
](https://www.waveshare.com/esp32-s3-rs485-can.htm) (This is a direct link for
your convenience; I am not an affiliate and receive no compensation for this
recommendation) is a near-turnkey option: it integrates an ESP32-S3 with a
built-in RS485 transceiver and can be powered directly from the Hayward
AquaLogic / ProLogic RS485 header, so no separate power supply or adapter is
needed.

**Hayward AquaLogic / ProLogic RS485 header wiring:**

| Controller header | Wire color | Board terminal |
|---|---|---|
| 10-12 VDC | RED | VCC |
| A+ | BLACK | A+ |
| B- | YELLOW | B- |
| GND | GREEN | GND |

When using this board, set the ESPHome board identifier to a valid ESP32-S3
board name from https://esphome.io/components/esp32 (for example
`esp32-s3-devkitc-1`) and adjust the UART and flow control pins:

```yaml
esp32:
  board: esp32-s3-devkitc-1   # see esphome.io/components/esp32 for the full list
  framework:
    type: esp-idf             # recommended for hardware RS485 half-duplex mode

uart:
  tx_pin: GPIO17
  rx_pin: GPIO18
  flow_control_pin: GPIO21    # built-in DE/RE pin; enables hardware half-duplex mode
```

### Recommended framework: ESP-IDF

All configs default to the ESP-IDF framework (`framework: {type: esp-idf}`).
Under ESP-IDF, setting `flow_control_pin` on the `uart:` component activates
the UART peripheral's hardware RS485 half-duplex mode: the DE/RE pin is driven
with shift-register-level timing by hardware. Under the Arduino framework the
`uart:` component does not drive `flow_control_pin` at all — Arduino users must
use an **auto-DE transceiver chip** (e.g. MAX13487, MAX22025) where DE follows
the TX line state automatically.

### Generic wiring

Any ESP32 variant with two UARTs works. You will also need a separate RS485 TTL
adapter. Half-duplex adapters with a DE/RE direction pin use `flow_control_pin`
on the `uart:` component; auto-direction or full-duplex adapters do not need it.
Most cheap adapters work fine at 3.3V logic.

Connect adapter TX/RX to the device UART pins shown in each config. If you
receive no data, swap `tx_pin` and `rx_pin` — some adapters label DI/RO from
the adapter's own perspective, which is the reverse of the device's.

For half-duplex RS485 adapters with a combined DE+RE pin:

```
GPIO21 --> DE/RE
GPIO17 --> DI  (TX)
GPIO18 --> RO  (RX)
```

## Writing your own decoders

All RX-side payload decoding is done through `on_frame:` handlers on the
`rs485_frame` hub. The hub diagnostic sensor and text_sensor platforms expose
counters and the most-recent frame type; user values land in `template` sensors
populated by your `on_frame:` lambdas. See the
[Lambda best practices](https://esphome.io/components/rs485_frame#writing-on-frame-lambdas-best-practices)
section of the hub docs for the pitfalls to avoid — most importantly:

- **No heap allocation in the lambda.** Decode into stack arrays; build one
  `std::string` only at the `publish_state` call site.
- **Always guard payload lengths.** `if (payload.size() < N) return;` before
  indexing.
- **Avoid `std::to_string`, `String::format`, and other allocating helpers.**
  Use `snprintf` into a stack buffer if you need formatted text.
- **Don't block.** A long loop or `delay()` inside the lambda stalls every
  other ESPHome component.
- **Decoding multi-byte fields?** ESPHome's
  [`bytebuffer`](https://esphome.io/components/bytebuffer/) helper offers endian-aware
  accessors over a byte span — cleaner and allocation-free compared to assembling integers
  from `payload[i]` shifts by hand.

These configurations follow those rules throughout.

## Offset convention: payload-relative

`rs485_frame` uses **payload-relative** byte offsets: `payload[0]` is the
first byte of the `frame_type` prefix. For the typical 2-byte frame_type, the
first data byte is at `payload[2]`. This matches what the lambda actually
receives at the C++ level — the `payload` vector has already had the DLE+STX
preamble stripped, escapes unwrapped, and CRC removed, but the frame_type
bytes are still at the start.

**Community references often use a different convention.** Some strip the
frame_type before counting, so what they call "byte 0" of the frame is our
`payload[2]`. When porting offsets from external research, add the
`frame_type` length (usually 2) to translate.

| Source | Their "byte 0" of the frame | Equivalent in our `payload[]` |
|---|---|---|
| **`rs485_frame`** (this component) | first byte of the frame_type | `payload[0]` |
| [swilson/aqualogic](https://github.com/swilson/aqualogic) | first byte **after** the frame_type | `payload[2]` (for 2-byte frame_type) |
| [earlephilhower/aquaweb](https://github.com/earlephilhower/aquaweb) | mixed; see the Python source for each frame | translate per-field |
| Raw bus capture (Wireshark, dump_frames log) | `DLE` itself | DLE+STX are stripped before `payload[0]` |

When publishing your own reverse-engineering findings, state the convention
explicitly ("offsets are payload-relative: `payload[0..1]` = frame_type, data
starts at `payload[2]`") — it costs one sentence and saves every reader an
off-by-two debugging session.

## Configurations

### hayward/

| File | Description |
|---|---|
| [aqualogic.yaml](hayward/aqualogic.yaml) | Full Hayward AquaLogic / ProLogic integration: buttons, LED state, temperatures, VSP pump |
| [sniffer.yaml](hayward/sniffer.yaml) | Passive sniffer for AquaLogic — log all frames and display text; useful for diagnosis or first-time setup |

Verified on Hayward AquaLogic and ProLogic (wireless remote emulation, 19200 8N2). Each config
declares the framing, CRC, gate frame, and `command_format` explicitly. See the comments at the
top of `hayward/aqualogic.yaml` for the wireless / wired-remote / wired-local transmit role
definitions.

### jandy-UNTESTED-DRAFT/

> **WARNING — UNTESTED:** These Jandy configurations are 100% untested and speculative. They
> have never been verified on physical hardware by anyone. The protocol bytes (notably the
> AllButton ACK third byte `0x80`), device addresses, CRC, and the ~60 ms poll-cycle ACK
> timing are community reverse-engineering guesses and may be wrong. The AllButton emulator
> actively transmits, so a wrong guess can disrupt a live controller. Use at your own risk,
> validate against your own captures, and please open an issue with results.

| File | Description |
|---|---|
| [aqualink-rs-passive.yaml](jandy-UNTESTED-DRAFT/aqualink-rs-passive.yaml) | Passive observer for Jandy AquaLink RS8/RS12/RS16: LED state, SWG, ePump, display |
| [aqualink-rs-allbutton.yaml](jandy-UNTESTED-DRAFT/aqualink-rs-allbutton.yaml) | Active AllButton emulator: all passive sensors plus button control |

### generic/

| File | Description |
|---|---|
| [discovery.yaml](generic/discovery.yaml) | Passive framing/CRC discovery for a completely unknown bus. Logs candidate `dle`/`stx`/`etx`, the escape scheme, and any matching CRC. Run this first when you don't know the framing bytes. |
| [sniffer.yaml](generic/sniffer.yaml) | Passive sniffer with `sniffer_stats:` once you know the framing. Logs a periodic per-frame-type table (cadence, unique payloads) to catalog frame types. |
| [skeleton.yaml](generic/skeleton.yaml) | Starting point for any DLE-framed RS485 device. Raw-form buttons (`frame_type:` + `payload:`) and the `rs485_frame.send_frame` action for computed payloads. Fork and replace the placeholder frame types with your device's. |

If you don't even know the framing bytes, start with `discovery.yaml`: it passively analyzes
raw traffic and suggests a `framing:` block (and CRC). Feed those values into `sniffer.yaml` to
catalog frame types, then into `skeleton.yaml` to write decoders.

A generic hub with no `command_format:` accepts any frame layout and rejects the `command:`
button shorthand (raw `frame_type:` + `payload:` is used instead). Write `on_frame:` decoders
and raw-form button entities for the commands you want to expose. If your device follows a
regular preamble + command bytes + postamble structure, add `command_format:` to the hub to use
the `command:` shorthand on buttons without writing raw payloads.

Jandy AquaLink RS uses 9600 baud 8N1 and a poll-response bus where the master probes each
device address in turn. The AllButton emulator responds to probes at address `0x08`
(configurable via `tx.gate.frame_type`) and sets `tx.idle_command` (typically `0x00`) so it
ACKs every probe — without that, the master marks the AllButton offline between user
commands. `tx.idle_command` requires a `command_format:` on the hub (config-validation time).

## Getting started

1. Copy the relevant YAML to your ESPHome configuration directory.
2. Fill in your board identifier in the `esp32:` block — see
   https://esphome.io/components/esp32 for the full list. Keep
   `framework: {type: esp-idf}` as-is; it is recommended.
3. Edit the `wifi` section (or add `!secret` entries to your `secrets.yaml`).
4. Adjust GPIO pins to match your wiring. If your RS485 adapter has a DE/RE
   direction pin, set `flow_control_pin` on the `uart:` component.
5. Review the commented-out sections — many sensors and buttons are optional.
6. Flash with the [ESPHome Builder App for Home Assistant](https://esphome.io/guides/getting_started_hassio/)
   or [ESPHome stand-alone](https://esphome.io/guides/getting_started_command_line/)

Start with the sniffer or passive config first to verify the bus is visible
before enabling command transmission.

## Sample output

Not sure what a good run looks like? See a real, annotated capture from a Hayward AquaLogic
controller, walked through from "unknown bus" to decoded frames — discovery + baud sweep, the
sniffer table on an idle bus and under live remote commands, and a version/ID frame cross-checked
against the controller's Diagnostic screen. Every CRC is verified by hand.

- [captures/hayward-aqualogic-20260529.md](captures/hayward-aqualogic-20260529.md)

It is also a good template for the format to use when you publish your own capture: include the
controller model, firmware revisions, bus parameters, and the component version, and scrub any
custom display text before sharing.

## Contributing

If you have a working configuration for a system not listed here, open a pull
request. Include the controller model, bus parameters (baud, parity, stop bits),
and whether it has been tested on physical hardware.

## Acknowledgements

The protocol decoders in these configurations build on community reverse
engineering of the Hayward and Jandy RS485 buses. Key references:

- **Hayward AquaLogic wireless remote** — frame layout and key encoding from
  [swilson/aqualogic](https://github.com/swilson/aqualogic).
- **Jandy AquaLink RS envelope format** (`DLE STX <data> <checksum> DLE ETX`) —
  [Jandy Pool Heater wiki](https://wiki.jmehan.com/display/KNOW/Jandy+Pool+Heater).
- **Jandy AllButton frame catalog, device addresses, and ACK byte sequence** —
  [earlephilhower/aquaweb](https://github.com/earlephilhower/aquaweb/blob/master/protocol.md).
