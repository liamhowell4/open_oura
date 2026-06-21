# Oura Ring 5 Research Plan

## Questions

- What BLE services and characteristics does Ring 5 expose before pairing,
  after pairing, and while the official app is closed?
- Does Ring 5 still use the tag/length/payload framing described by ringverse
  for Ring 4?
- Are the known Ring 4 read-only commands accepted by Ring 5?
- What authentication or nonce flow gates user data access?

## Initial passive capture

1. Scan nearby BLE advertisements and identify the ring name/address.
2. Connect and enumerate services and characteristics.
3. Subscribe to notify characteristics without sending protocol commands.
4. Record handles, UUIDs, properties, MTU, and notification payloads.

## First Ring 5 observations

Captured on 2026-06-21 in Lisbon after disabling Bluetooth on the paired phone.
macOS CoreBluetooth does not expose the real BLE MAC address, so the addresses
below are macOS peripheral UUIDs rather than the ring MAC.

Known device details from Oura app:

- Model: Oura Ring 5
- Serial: `50380B2617647259`
- BLE MAC: `c9:bc:a2:5d:ac:56`

Advertisements observed:

- Ring:
  - macOS UUID: `F928A493-157D-B2B5-0D19-F43F8DB5680E`
  - Name: `Oura Ring 5`
  - RSSI: `-81`
  - Service UUID: `98ed0001-a541-11e4-b6a0-0002a5d5c51b`
  - Manufacturer data: `02b2:04706b01`
- Charging case:
  - macOS UUID: `724CE68A-F69F-B641-B08E-DD251A0EF3F9`
  - Name: `Oura Ring 5 Charging Case`
  - RSSI: `-84`
  - Service UUID: `8bc5888f-c577-4f5d-857f-377354093f13`
  - Manufacturer data: `02b2:04a00b00`

Direct service enumeration attempts against both macOS UUIDs timed out with
Bleak/CoreBluetooth. Next steps are to repeat while the ring is on the charger
and the phone Bluetooth remains off, then try a longer connection timeout.

## Low-risk active probes

These should only be attempted after service discovery confirms the likely
control characteristic.

- `0C00` - Get battery level.
- `0803000000` - Get firmware version.
- Product info reads with known Ring 4 request shapes.

Avoid reset, DFU, factory reset, flight mode, auth mutation, and user-info writes
until the Ring 5 protocol is better understood.

## Capture format

Each experiment should record:

- Date, timezone, OS, Bluetooth adapter, ring firmware if known.
- Pairing state and whether the official Oura app was running.
- Command hex, characteristic UUID/handle, response hex, and timing.
- Any visible side effect on the ring or the Oura app.
