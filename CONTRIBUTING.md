# Contributing configurations

This guide is for **contributors** who are reverse-engineering a controller and authoring reusable
config files for the library. If you just want to *use* an existing controller config, see the
[README](README.md) instead.

The library is built so that no two installations have to share a hand-edited file. A contributor
authors small, composable package files; an end user assembles a device config from them by listing
the equipment they have. This document explains how to author those files.

## 1. Scope and the single layering style

Standardize on ESPHome [`packages`](https://esphome.io/components/packages/). A device config pulls
reusable files in via `packages:` — remote git packages (pinned tag) by default, or local copies as
a fallback for users who must edit a decoder. **Do not mix `!include` / `packages` / `!extend`
styles across configs**: equipment profiles extend the shared hub with `!extend` (see below), and
that is the only place `!extend` appears.

**Exempt: diagnostic / bootstrap tools.** `generic/discovery.yaml`, `generic/sniffer.yaml`,
`generic/skeleton.yaml`, and the per-family `sniffer.yaml` are single-purpose tools for an *unknown*
bus — there are no equipment profiles to compose yet, so they stay monolithic and self-contained.

## 2. Terminology and repo layout

"Device" is reserved for the ESPHome node (matching the ESPHome / Home Assistant dashboard's own
usage). RS485 bus members are called **equipment**, never "devices." The phrase "device profile" is
not used.

| Term | What it is | Reusable? | Edited by | Example |
|---|---|---|---|---|
| **Device config** | The complete ESPHome config for one physical ESP board: `substitutions` + `wifi`/`api`/`ota`/secrets + a `packages:` list. The only file flashed. | No (per-installation) | End user | `hayward/aqualogic/example-device.yaml` |
| **Bus package** | The shared RS485 hub for a controller family: `uart`, `framing`, `crc`, `tx`, `command_format` + panel-wide entities. One per family. | Yes | Contributor | `hayward/aqualogic/bus.yaml` |
| **Equipment profile** | One piece of equipment, keyed on the `frame_type`s it owns. Contributes `on_frame` decoders + its own entities. | Yes | Contributor | `pump-vsp.yaml`, `swg.yaml` |

Layout: each controller family lives at `vendor/controller-family/` and holds `bus.yaml`, one file
per equipment profile, and `example-device.yaml` (one per operating mode where relevant, e.g. Jandy
passive vs. allbutton). Reusable skeletons live in `templates/`; captures in `captures/`.

**File naming:** lowercase, hyphenated, named for the equipment (`pump-vsp.yaml`, `aux-relay.yaml`,
`leds-display.yaml`). `bus.yaml` and `example-device.yaml` are fixed names.

## 3. The pluggable unit

An **equipment profile is the set of `frame_type`s it owns** — in either direction. A variable-speed
pump owns both the controller→pump command frame (Hayward `0x0c 0x01`) and the pump→bus report frame
(`0x00 0x0c`); both belong in `pump-vsp.yaml`. Keying on owned frames composes cleanly on a
panel-mediated bus (Hayward) and a poll-response bus (Jandy) alike.

A profile **may** contain:

- A partial hub entry that extends the shared hub by id (decoders only — see §4).
- Its own entities (`sensor`, `binary_sensor`, `text_sensor`, `button`, ...).

A profile **must not** contain `uart`, `framing`, `crc`, `command_format` (except where it converts a
passive base to active — see §6), or any `!secret`. Those belong to `bus.yaml` / the device config.

**The `bus.yaml` contract.** The bus package owns the bus-wide required hub fields (`uart`,
`framing`, `crc`, `tx`, optionally `command_format`) and panel-wide entities. It declares the hub
with a plain `id:` (e.g. `id: pool`); every profile and entity references that id.

## 4. Composition mechanics

ESPHome merges packages with `merge_config`: dicts merge key-by-key, and **lists are concatenated** —
there is *no* automatic "merge components by id." To add to an existing component, use ESPHome's
[`!extend`](https://esphome.io/components/packages/#extend):

```yaml
# bus.yaml — declares the hub
rs485_frame:
  - id: pool
    uart_id: pool_uart
    framing: { escape: { mode: escape_byte, byte: 0x00 } }
    crc: { type: sum16, tx_variant: header_inclusive }
    # ...
```

```yaml
# pump-vsp.yaml — adds to it
rs485_frame:
  - id: !extend pool          # <-- REQUIRED. A bare `id: pool` creates a SECOND hub and fails.
    on_frame:
      - frame_type: [0x0c, 0x01]
        then: [ lambda: !lambda "..." ]
```

`!extend pool` finds the hub declared in `bus.yaml` and merges the profile's dict into it; because
`on_frame` is a list, the profile's handlers are concatenated onto the hub's. Several profiles can
extend the same hub, each adding its own `on_frame` handlers and entities, and they all land in one
merged hub.

**Shared-frame decoding rule.** The hub fires **every** `on_frame` handler whose `frame_type` prefix
matches a received frame, not just the first. So one frame type can be decoded by several profiles:
the Hayward LED-mask frame `0x01 0x02` is read by `bus.yaml` (panel bits), `heater.yaml` (bit 0),
`lights.yaml` (bit 6), and each `aux-relay.yaml` channel — five independent handlers on the same
frame. **Register one handler per profile, guard only your own bytes/bits, and never assume sole
ownership of a frame type.**

## 5. The substitution contract

A device config must define these substitutions (the bus package references them):

- `name`, `friendly_name`, `board` — the ESPHome node identity.
- `tx_pin`, `rx_pin` — UART pins.
- `flow_control_pin` — DE/RE direction pin (omit only for a passive, never-transmitting base).

Per-vendor optional substitutions (e.g. a transmit-role command preamble, baud overrides) are
documented in that family's `bus.yaml` header. **Scalar substitutions** follow last-write-wins, so
defaults for purely scalar knobs (e.g. `temp_unit`) live in `bus.yaml` and are overridden in the
device config. **Exception: keep logically related knobs together.** All five `command_format` knobs
(`cmd_preamble`, `cmd_postamble`, `cmd_size`, `cmd_endian`, `cmd_repeat`) live in the device
config — none are defaulted in `bus.yaml` — so the transmit role is fully self-contained in one
place and contributors cannot accidentally split it.

**Two substitution gotchas, both verified with `esphome config`:**

1. **List-valued substitutions must be defined in exactly ONE place — the device config.** When the
   same list substitution is set in both a package file and the device config, ESPHome
   *concatenates* the two lists rather than overriding (`[0x00,0x83,0x01,0x00,0x83,0x01]`, not an
   override). A list-valued knob like a command preamble **must not be defaulted in a package file**;
   reference it whole-value (`preamble: ${cmd_preamble}`) and require the device config to supply it
   as an unquoted YAML list: `cmd_preamble: [0x00, 0x83, 0x01]`.
2. **`${var}` cannot appear inside a YAML flow sequence/mapping.** `preamble: [${cmd_preamble}]` and
   `esphome: { name: ${name} }` fail to parse (the `{` in `${...}` confuses the flow parser). Use
   block style (`esphome:` then `  name: ${name}`) or whole-value substitution (`preamble:
   ${cmd_preamble}` with `cmd_preamble` declared as a real YAML list).

**Secrets stay in the device config.** Remote git packages cannot contain `!secret`, so
`wifi`/`api`/`ota` and their secret lookups live in the device file.

## 6. Superset handling and role/mode selection

**AUX / VALVE channels — packages-as-template.** Rather than ship a bloated enabled-by-default
superset, `aux-relay.yaml` defines ONE channel parameterized by `vars`, and the device config lists
it once per channel:

```yaml
files:
  - hayward/aqualogic/bus.yaml
  - path: hayward/aqualogic/aux-relay.yaml
    vars: { n: 1, label: Blower, cmd: "0x00020000", bit: 7 }
  - path: hayward/aqualogic/aux-relay.yaml
    vars: { n: 2, label: UV,     cmd: "0x00040000", bit: 8 }
```

Each entry adds one button + one `running` binary_sensor + one `on_frame: [0x01,0x02]` handler
reading `bit ${bit}`. `n` makes the entity id unique (`aux_${n}_running`). For local-copy use the
`!include` form works the same: `!include { file: aux-relay.yaml, vars: { n: 1, ... } }`. A template
file may carry a `defaults:` block supplying `vars` not provided by the include.

The simpler **`!remove`-the-superset** alternative also works: ship an `aux-relays.yaml` with N
channels enabled and let the device config `!remove` the entity ids it lacks. The per-channel
template is preferred because the device config then names only what exists.

**Vendor role / mode selection.** Prefer a substitution over separate files when the difference is a
few bytes. Hayward's wireless/wired transmit role is the `${cmd_preamble}` substitution (a one-line
edit in the device config; verified values are listed in `hayward/aqualogic/bus.yaml`). When the
difference is structural (passive observer vs. active emulator), use a small profile that **extends**
the base: Jandy's `allbutton.yaml` flips `sniffer_only: false`, adds `command_format` + `tx`, and
extends the uart (`uart: [{ id: !extend pool_uart, flow_control_pin: ${flow_control_pin} }]`) to add
the direction pin the passive base omits.

## 6a. Keep the example menu complete (obligation)

`example-device.yaml` is the consumer's runnable, commented "menu" of every profile for a family.
**When you add a bus package or equipment profile, add its commented `packages:` line — a one-line
description + `Status: tested|UNTESTED` — to that family's `example-device.yaml`(s).** The bus line
stays uncommented (it is mandatory). This keeps the consumer-facing menu authoritative; the PR
checklist gates on it.

## 7. Templates and the standard metadata header

Two skeletons live in `templates/`: `templates/bus.yaml` (bus package) and `templates/profile.yaml`
(equipment profile). Copy the matching one when authoring a new file.

Every reusable file (`bus.yaml`, equipment profiles, and the `templates/*` skeletons) opens with the
**full** standard metadata header:

```yaml
# =============================================================================
# <Vendor> <Controller family> — <bus package | equipment profile: NAME>
# -----------------------------------------------------------------------------
# Status:      tested-on-hardware | UNTESTED-draft
# Tested on:   <controller model>, firmware <rev>; <ESP board>; ESPHome <x.y.z>;
#              rs485_frame <component version / git ref>
# Bus:         <baud> baud, <data><parity><stop>   (e.g. 19200 8N2)
# Owns frames: <frame_type list this file decodes/sends>  (bus package: gate + command frames)
# Offsets:     payload-relative — payload[0..N-1] = frame_type, data starts at payload[N]
# References:  <reverse-engineering source links>
#
# Contributors:
#   - <Name (@handle)> — <contribution>  <link>
# =============================================================================
```

`example-device.yaml` carries a **lighter** header — `Status`, `Tested on` (the exact system the
author verified), and `Contributors` — because frame ownership / offsets / references live in the
packages it pulls in.

## 8. Decoder rules and the offset convention

**Offset convention: payload-relative.** `payload[0]` is the first byte of the `frame_type` prefix.
For a 2-byte frame_type the first data byte is `payload[2]`. The `payload` vector the lambda receives
has already had the DLE+STX preamble stripped, escapes unwrapped, and CRC removed — but the
frame_type bytes are still at the start.

Community references often strip the frame_type before counting, so their "byte 0" is our
`payload[2]`. When porting offsets, add the frame_type length (usually 2) to translate.

| Source | Their "byte 0" of the frame | Equivalent in our `payload[]` |
|---|---|---|
| **`rs485_frame`** (this component) | first byte of the frame_type | `payload[0]` |
| [swilson/aqualogic](https://github.com/swilson/aqualogic) | first byte after the frame_type | `payload[2]` (2-byte frame_type) |
| [earlephilhower/aquaweb](https://github.com/earlephilhower/aquaweb) | mixed; see the Python source | translate per-field |
| Raw bus capture (dump_frames log) | `DLE` itself | DLE+STX stripped before `payload[0]` |

When you publish findings, state the convention explicitly ("offsets are payload-relative:
`payload[0..1]` = frame_type, data starts at `payload[2]`").

**`on_frame` lambda best practices** (from the hub docs):

- **No heap allocation in the lambda.** Decode into `static`/stack arrays; build one `std::string`
  only at the `publish_state` call site.
- **Always guard payload length** before indexing: `if (payload.size() < N) return;`.
- **Avoid `std::to_string`, `String::format`, and other allocating helpers.** Use `snprintf` into a
  stack buffer for formatted text.
- **Decoding multi-byte fields?** ESPHome's [`bytebuffer`](https://esphome.io/components/bytebuffer/)
  helper gives endian-aware, allocation-free accessors over a byte span.
- **Don't block.** A long loop or `delay()` stalls every other ESPHome component.

## 8a. Versioning and breaking changes

Pointing users at this repo makes the package files a **published interface**. The breaking surface
is: file **paths**, the hub **`id:`** (entities use `rs485_frame_id:`), **substitution
names/semantics** and which are required, and entity **`id:` / `name:`** (Home Assistant entity_ids
and users' automations derive from these).

**Guardrails:**

1. **Pinned tags only.** Every published example uses `ref: vX.Y.Z` (an immutable release tag), never
   a branch. A tag never moves, so `refresh:` cannot deliver a surprise; users opt in by bumping the
   one `ref:`.
2. **Single-ref mapping form.** One `remote_package` block (`url:` + `ref:` + `files:`) per device
   config, so the ref lives in one place and the commented menu + aux `vars` entries share it.
3. **Semver against the interface.** MAJOR = change a path / hub id / substitution / entity
   id-or-name. MINOR = additive (new profile file, new optional scalar substitution with a default,
   new entity). PATCH = decoder/bugfix. Additive-by-default: within a major line never hard-delete or
   rename a referenced file or entity; any new required substitution must ship a default if scalar,
   or be documented as required-in-device-config if list-valued (list defaults in packages
   concatenate rather than override — see §5).
4. **`CHANGELOG.md` + migration notes** for every breaking change.

## 9. `external_components` / staging ref

`rs485_frame` is not yet in an ESPHome release. The `external_components` block (pointing at the
staging branch) lives **once, in `bus.yaml`** — equipment profiles and device configs must not repeat
it (`external_components` is a list and would duplicate). Remove the block when `rs485_frame` ships in
an official ESPHome release.

## 10. UNTESTED policy

Status lives in the header's `Status` field and the README catalog badge — **not** in a directory
name. For untested or actively-transmitting configs (e.g. the Jandy drafts), keep the strong warning
banner at the top of each file and in the README catalog. Never silently promote a draft to "tested"
without a capture to back it.

## 11. Capture publishing

Publish a capture for any new controller. Start from [`captures/TEMPLATE.md`](captures/TEMPLATE.md)
and require: controller model, firmware revision(s), bus parameters, ESP board, ESPHome version, and
the `rs485_frame` component version; a `## Contributors` credit table; and a `Status` row. **Scrub
custom display text** (pool/spa names, schedule text) before sharing. The worked example is
[`captures/hayward-aqualogic-20260529.md`](captures/hayward-aqualogic-20260529.md).

## 12. PR checklist

- [ ] Standard metadata header present (full on packages/profiles/templates; lighter on
      `example-device.yaml`), and the `Status` field is accurate.
- [ ] Config validates: `esphome config` passes on a device config that includes the new file.
- [ ] Equipment profile extends the hub with `id: !extend <hub>` (not a bare `id:`), defines no
      `uart`/`framing`/`crc`/`secret`, and guards every `payload.size()`.
- [ ] New profile / bus added as a commented `packages:` line (description + Status) to the family's
      `example-device.yaml`(s); bus line kept uncommented.
- [ ] Capture attached for a new controller (`captures/`), with contributors credited.
- [ ] No duplicated `external_components` block (only `bus.yaml` has it).
- [ ] **Touches the public interface (paths / hub id / substitutions / entity id-or-name)? → MAJOR
      version bump + `CHANGELOG.md` entry + migration note.**

For how end users consume this library, see the [README](README.md).
