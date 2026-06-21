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
