# Segment Control via the LAN API

This document describes how govee2mqtt implements per-segment color control
for devices such as the **H60B2** Tree Floor Lamp, using the Govee LAN API's
binary `ptReal` command.

## Background

Govee's documented Platform API exposes segment color capabilities for some
devices via the `devices.capabilities` endpoint. govee2mqtt uses this to create
per-segment Home Assistant light entities for any device the Platform API
describes as having `SegmentColorSetting` capability.

However, Platform API access requires a Govee API key and a live internet
connection. For LAN-API-capable devices, govee2mqtt also supports a
fully-local segment control path that works without any cloud dependency.
This was reverse-engineered and implemented in response to
[issue #493](https://github.com/wez/govee2mqtt/issues/493).

## Supported Devices

| SKU   | Name              | Segments | LAN Segment Control |
|-------|-------------------|----------|---------------------|
| H60B2 | Tree Floor Lamp   | 3        | Yes                 |

Other devices may be added to the quirks database as the binary packet format
is confirmed on additional hardware.

## How It Works

### Discovery

The H60B2 is registered in govee2mqtt's quirks database with a `segment_count`
of 3. When the device is discovered via the LAN API (no API key required),
govee2mqtt creates a separate Home Assistant MQTT light entity for each
segment (`Segment 001`, `Segment 002`, `Segment 003`).

### Command Path

Standard LAN API commands use a JSON payload (color, brightness, on/off).
Segment color is not available through that mechanism. Instead, govee2mqtt
uses the `ptReal` command, which accepts a raw binary payload encoded as a
base64 string. This is the same mechanism used by the Govee mobile app for
advanced lighting effects.

The binary packet format for segment color is:

```
Byte:   0     1     2     3     4  5  6  7–11      12      13    14–18     19
Data:  0x33  0x05  0x15  0x01   R  G  B  0×5   SEG_LO  SEG_HI   0×5   CHECKSUM
```

- **R, G, B** — color components, 0–255
- **SEG_LO / SEG_HI** — little-endian 16-bit bitmask identifying which
  segment(s) to address. Bit 0 = segment 0, bit 1 = segment 1, and so on.
- **CHECKSUM** — XOR of bytes 0–18, appended by `ble.rs::finish()`

The packet is padded to 19 bytes before the checksum is applied.

### Brightness

The Govee LAN API has no separate brightness command for individual segments.
Brightness is encoded directly in the RGB values: a red segment at 50%
brightness is sent as `(127, 0, 0)` rather than `(255, 0, 0)`.

Home Assistant sends brightness as a value from 0–100 (governed by the
`brightness_scale: 100` field in the MQTT discovery config). When a
combined color-and-brightness command arrives, the RGB values are scaled
by `brightness / 100.0` before transmission.

### State Reporting and the Brightness Limitation

The Govee LAN API's `devStatus` response returns a single color for the
whole device. When the device is operating in segment mode, this value is
`(0, 0, 0)` — it does not reflect any individual segment's color.

Because of this, govee2mqtt cannot query the device to determine what color
a segment is currently showing. Instead, it tracks the last color sent to
each segment within the current session (stored in `device.segment_colors`).

When Home Assistant sends a brightness-only command (no color included),
govee2mqtt scales the stored color by the new brightness level. If no color
has been sent to a segment in the current session, it falls back to white.

**Consequence:** if a segment's color is changed from outside govee2mqtt
(e.g. via the Govee mobile app or another controller), the stored color
will be stale. The next brightness adjustment from Home Assistant will scale
the old color rather than the current one. Setting the color from Home
Assistant once will resynchronise the stored state.

## Functional Features

- Per-segment RGB color control, fully local (no API key, no internet)
- Brightness control per segment (encoded as RGB scaling)
- Automatic Home Assistant MQTT discovery — segments appear as separate
  light entities without any manual configuration
- Works alongside the main device light entity

## Known Limitations

- **No per-segment state readback** — the LAN API does not report per-segment
  color in `devStatus`. govee2mqtt tracks the last-sent color in memory only.
- **Session state only** — the stored segment colors are lost on service
  restart. After a restart, set each segment's color from Home Assistant
  once before using the brightness slider.
- **Mobile app changes are not tracked** — colors set outside govee2mqtt
  (mobile app, other integrations) are not reflected in the stored state.
- **No segment on/off** — the LAN `ptReal` command controls color only.
  The power toggle shown in Home Assistant for segment entities has no effect.
  To darken a segment, set its brightness to 0 (sends black, `0x00 0x00 0x00`).
- **Mixed-color brightness** — the brightness slider adjusts a single segment
  at a time. There is no single command to change the brightness of all
  segments simultaneously while preserving their individual colors.

## Packet Format Sources and Accreditation

The `ptReal` binary packet format used here was reverse-engineered from:

- [govee2mqtt issue #105](https://github.com/wez/govee2mqtt/issues/105) —
  community research into the binary packet structure
- [govee-local-api](https://github.com/Bluetooth-Devices/govee-local-api) —
  the Python library whose packet construction was used as a reference
  implementation

The segment bitmask encoding (SEG_LO / SEG_HI as a little-endian u16) was
confirmed by cross-referencing both sources. The format has been tested on
the H60B2. Behavior on other multi-segment LAN API devices is unconfirmed.

## Adding Support for Other Devices

To add LAN segment control for another device, register it in
`src/service/quirks.rs` using the `.with_segments(N)` builder:

```rust
Quirk::lan_api_capable_light("HXXXX", ICON).with_segments(3),
```

Replace `HXXXX` with the device SKU and `3` with the actual segment count.
The binary packet format is expected to be the same across devices that
support `ptReal`, but this should be verified on hardware before shipping.
