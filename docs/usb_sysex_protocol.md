---
license: CC BY-SA 4.0
author: Joel Victor
---

# USB MIDI & SysEx Architecture (Base-128)

This document explains how the M-Vave Blackbox handles its proprietary communication over a wired USB connection. 

Unlike the Bluetooth GATT tunnel, which accepts the raw 8-bit packets documented in the [ble_protocol.md](ble_protocol.md) document, the USB connection exposes a Class-Compliant MIDI interface. This introduces a major technical barrier: the Universal MIDI standard.

## The MIDI 7-Bit Limitation

By international MIDI standards, any byte greater than `0x7F` (127 in decimal) is strictly reserved for **Status Bytes** (like Note On, Control Change, etc.). Data bytes must always be between `0x00` and `0x7F`.

The raw M-Vave configuration packets (e.g., `00 59...`) frequently contain bytes that exceed this limit (such as `0x90`, `0xFF`, `0xE1`). If you attempt to send these raw packets via standard USB MIDI, the operating system's MIDI driver or the browser's Web MIDI API will immediately block the transmission or throw a `TypeError` for violating the protocol.

To bypass this without triggering OS-level security sandboxing, M-Vave uses **System Exclusive (SysEx)** messages paired with a **Base-128 Bit-packing** algorithm.

For the Base-128 bit-packing transformation used in USB MIDI, the encoding function $E(B)$ performs the following bitwise shift:

$$E(B) = \sum_{i=0}^{7} b_i \cdot 2^i \pmod{2^7}$$

---

## The SysEx Wrapper

A SysEx message allows manufacturers to send custom data. It must always start with `F0` and end with `F7`, and every byte in between **must** be `0x7F` or lower.

To make the raw M-Vave packets fit into this strict rule, the pedal's firmware applies a continuous 8-to-7 bit packing (Base-128 encode) before transmission, and unpacks it (decode) upon reception.

**The Final USB Packet Structure:**

1. `0xF0` (SysEx Start Marker)
2. `[ ... Base-128 Encoded Payload ... ]` (Variable length, 7-bit safe)
3. `0xF7` (SysEx End Marker)

---

## Continuous Bit-Packing (8-to-7 bit Encoding)

The algorithm used by M-Vave is mathematically intense. Unlike simple "Nibble Expansion" (which splits one byte into two), M-Vave treats the entire raw payload as a single continuous river of bits.

The encoder strips the 8th bit of every byte, pushing the overflow into the next byte using Bitwise Shift operators (`<<` and `>>`). This compresses the data highly efficiently: a block of **7 raw bytes (8-bit)** is perfectly packed into **8 SysEx bytes (7-bit)**.

### Encoding Example (Toggle Block Command)

Let's take the raw 16-byte command to turn on an effect block (as documented in the Base Protocol):

* **Raw Array (8-bit):** `00 59 22 09 00 00 04 08 00 00 10 01 00 00 01 E1`

When passed through the Base-128 encoder, the bits are re-sliced into 7-bit chunks. The resulting array becomes slightly longer (19 bytes), but every byte is now safely under `0x7F`:

* **Encoded Array (7-bit safe):** `00 32 09 49 00 00 40 02 09 00 00 00 18 00 00 00 01 5E 01`

Finally, wrap it in SysEx markers to send it over the USB cable:

* **Final USB Message:** `F0 00 32 09 49 00 00 40 02 09 00 00 00 18 00 00 00 01 5E 01 F7`

---

## Development & Implementation Notes

If you are building a Web, Desktop, or Microcontroller client for this pedal via USB, your application architecture must follow this flow:

**Sending Commands (App to Pedal):**

1. Generate the standard `00 59...` raw payload (including the standard checksum calculation).
2. Pass the payload through a Base-128 `encode()` function.
3. Prepend `0xF0` and append `0xF7`.
4. Send via standard MIDI out.

**Receiving Data (Pedal to App):**

1. Listen for incoming SysEx messages starting with `0xF0`.
2. Strip the `0xF0` and `0xF7` markers.
3. Pass the payload through a Base-128 `decode()` function to restore the 8-bit bytes.
4. Process the resulting `00 59...` raw packet normally.

*(Note: The exact bitwise shift logic for the encode/decode functions can be found in the open-source community, specifically derived from the GPL-3.0 Cuvave MIDI parser written in Rust).*