# Changelog

All notable changes to the published configuration interface are documented here. This project uses
[Semantic Versioning](https://semver.org/) against the **public interface**: file paths, the hub
`id:`, substitution names/semantics (and which are required), and entity `id:`/`name:`. See
[CONTRIBUTING.md](CONTRIBUTING.md#8a-versioning-and-breaking-changes) for the rules.

- **MAJOR** — change a file path, hub `id`, substitution, or entity `id`/`name`.
- **MINOR** — additive: a new profile file, a new optional substitution *with a default*, a new entity.
- **PATCH** — decoder or bug fix with no interface change.

## [1.0.0] - 2026-05-30

Initial released interface. Restructured the monolithic per-controller YAMLs into composable
[ESPHome `packages`](https://esphome.io/components/packages/): a per-family **bus package** plus one
**equipment profile** per piece of gear, assembled by a thin **device config**.

### Added

- `hayward/aqualogic/`: `bus.yaml` (hub: UART 19200 8N2, sum16 CRC, gate `[0x01,0x01]`, wireless
  `command_format`, panel LEDs + display + diagnostics), `pump-vsp.yaml`, `heater.yaml`,
  `lights.yaml`, `aux-relay.yaml` (per-channel template), and `example-device.yaml`.
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

[1.0.0]: https://github.com/b3nj1/rs485_frame-examples/releases/tag/v1.0.0
