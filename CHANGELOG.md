# Changelog

All notable changes to the published configuration interface are documented here. This project uses
[Semantic Versioning](https://semver.org/) against the **public interface**: file paths, the hub
`id:`, substitution names/semantics (and which are required), and entity `id:`/`name:`. See
[CONTRIBUTING.md](CONTRIBUTING.md#8a-versioning-and-breaking-changes) for the rules.

- **MAJOR** — change a file path, hub `id`, substitution, or entity `id`/`name`.
- **MINOR** — additive: a new profile file, a new optional substitution *with a default*, a new entity.
- **PATCH** — decoder or bug fix with no interface change.

## [1.0.1] - 2026-06-05

### Changed

- **uart block moved to device config** (breaking for users upgrading from 1.0.0). The `uart:`
  block is no longer defined inside `hayward/aqualogic/bus.yaml` or `jandy/aqualink-rs/bus.yaml`.
  It must be declared directly in the device config with `id: pool_uart` (Hayward) or
  `- id: pool_uart` (Jandy). The family-specific constants (baud rate, data bits, parity,
  stop bits, rx_buffer_size) belong there too. The `jandy/aqualink-rs/allbutton.yaml`
  `uart: !extend` for `flow_control_pin` is likewise removed.

- **`substitutions:` now contains all user-facing setup values**: `name`, `friendly_name`,
  `board`, `tx_pin`, `rx_pin`, and `flow_control_pin` are all declared there as `BOARDXX` /
  `GPIOXX` placeholders. The `esp32:` and `uart:` blocks reference them via `${...}` and no
  longer need editing. To omit `flow_control_pin` (auto-direction adapters), delete the
  substitution and the `flow_control_pin: ${flow_control_pin}` line from the `uart:` block.

- **Placeholder values** changed from literal GPIO numbers and a specific board identifier to
  `GPIOXX` and `BOARDXX` throughout all config examples, matching the ESPHome documentation
  convention. ESPHome validation rejects these strings, ensuring users replace them before flash.

### Migration from 1.0.0

1. Add a `uart:` block directly to your device config **before** the `packages:` block, using
   `id: pool_uart` (Hayward) or `- id: pool_uart` (Jandy). Copy the family-specific baud/parity
   settings from the updated `example-device.yaml` (or Jandy equivalent).
2. Move `tx_pin`, `rx_pin`, and `flow_control_pin` into your `substitutions:` block with their
   real pin values. The `uart:` block references them via `${tx_pin}` etc.
3. Add `board` to `substitutions:` and change `esp32: board:` to `${board}`.
4. Bump your package ref to `ref: v1.0.1`.

---

## [1.0.0] - 2026-05-30

Initial released interface. Restructured the monolithic per-controller YAMLs into composable
[ESPHome `packages`](https://esphome.io/components/packages/): a per-family **bus package** plus one
**equipment profile** per piece of gear, assembled by a thin **device config**.

### Added

- `hayward/aqualogic/`: `bus.yaml` (hub: UART 19200 8N2, sum16_big_endian CRC, gate `[0x01,0x01]`,
  wireless `command_format`, panel LEDs + display + diagnostics + service button), `pump-vsp.yaml`,
  `heater.yaml`, `button.yaml` (parameterized nav/AUX buttons), `led.yaml` (parameterized LED
  binary sensor), and `example-device.yaml`.
- `jandy/aqualink-rs/` (UNTESTED drafts): `bus.yaml` (passive base), `leds-display.yaml`, `swg.yaml`,
  `epump.yaml`, `heater.yaml`, `allbutton.yaml` (active emulator), `example-passive.yaml`,
  `example-allbutton.yaml`.
- `templates/bus.yaml` and `templates/profile.yaml` skeletons with the standard metadata header.
- `CONTRIBUTING.md` (authoring guide), `captures/TEMPLATE.md` (capture publishing template), and a
  `## Contributors` section on the existing Hayward capture.

### Changed

- `hayward/aqualogic.yaml` → split into the `hayward/aqualogic/` package set.
- `hayward/sniffer.yaml` → `hayward/aqualogic/sniffer.yaml` (stays a monolithic diagnostic tool).
- `jandy-UNTESTED-DRAFT/` → `jandy/aqualink-rs/`. UNTESTED status now lives in each file's `Status`
  header and the README catalog, not the directory name.
- `README.md` rewritten for consumers (assemble-a-device-config flow + family catalog); decoder rules
  and the offset convention moved to `CONTRIBUTING.md`.
- `generic/` discovery / sniffer / skeleton remain monolithic bootstrap tools (exempt from the
  package structure) with header notes pointing at it for real integrations.

[1.0.1]: https://github.com/b3nj1/rs485_frame-examples/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/b3nj1/rs485_frame-examples/releases/tag/v1.0.0
