---
license: CC BY-SA 4.0
author: Joel Victor
---

# M-Vave Blackbox BLE Protocol

Reverse engineering of the Bluetooth Low Energy (BLE) communication format for the M-Vave Blackbox multi-effects pedal.

## Summary

BLE communication with the M-Vave Blackbox does not use the Universal MIDI standard. Instead, the pedal uses proprietary Hexadecimal packets transmitted via GATT (Generic Attribute Profile) using the *Write Without Response* method.

> **TECHNICAL NOTE (Deviation from MIDI Standard):**
> This protocol does not use the standard Apple/MMA MIDI over BLE service (Service UUID: `03B80E5A-EDE8-4B33-A751-6CE34EC4C700`). Instead, it communicates through a generic data tunnel (GATT) using the base UUID `0000ae40...`. **Do not attempt to use native WebMIDI or CoreMIDI libraries to send these commands**, as the packets must be injected as `Uint8Array` directly into the TX/RX characteristics below.

* **Device Name:** `BlackBox_BLE`
* **Service UUID:** `0000ae40-0000-1000-8000-00805f9b34fb`
* **TX / Write (Command Dispatch):** `ae41` (Handle: `0x0062`)
* **RX / Notify (State Reception):** `ae42` (Handle: `0x0064`)

All write commands follow a structured base format. `N` represents the total number of bytes in the message (which can be 15, 16, 17, or 107 bytes).

|Offset|Bytes|Function|Description|
|------:|-----:|:-------|:----------|
|`0`|3|**Header**|Start marker. `00 59 22` (RAM) or `00 59 23` (Flash / Polling).|
|`3`|1|**Command Type**|Defines the operation: `08`, `09`, `0A`, `64`, etc.|
|`4`|3|**Routing**|Default padding: `00 00 04`.|
|`7`|1|**Target ID**|Target address. In RAM parameters, it indicates the Block ID. In DMA, it indicates the Flash Memory Address.|
|`8`|2|**Padding**|Usually `00 00`.|
|`10`|2|**Sub-address**|Identifies the action: `10 01` (Toggle), `10 02` (Knob), `E0 01` (System). In DMA commands, it indicates the Payload Size.|
|`12`|*varies*|**Payload / Value**|The command value (1, 2, or 92 bytes).|
|`N-1`|1|**Checksum**|Verification sum. The pedal silently ignores invalid packets.|

---

## The Universal Checksum Formula (Encoding Example)

M-Vave requires the last byte to ensure a strict sum based on the useful bytes of the packet (from Offset 4 to the penultimate byte).

**Formula:** `Checksum = 0xFF - Sum(Bytes from offset 4 to N-2)`

**Encoding Example:** Turn on the FX block (Size = 16 bytes)
Base command without the checksum: `00 59 22 09 00 00 04 08 00 00 10 01 00 00 01`

1. Extract the bytes to sum (Offset 4 to 14): `00 00 04 08 00 00 10 01 00 00 01`
2. Sum the values (Hex): `00 + 00 + 04 + 08 + 00 + 00 + 10 + 01 + 00 + 00 + 01` = `0x1E`
3. Subtract from 0xFF: `0xFF - 0x1E` = **`0xE1`**
4. Final packet: `00 59 22 09 00 00 04 08 00 00 10 01 00 00 01 E1`

*(Implementation Note: In typed languages, apply a `& 0xFF` mask to the sum before subtracting to avoid overflow errors).*

---

## System Commands

Commands for global state changes and UI management. Offset 10 uses the sub-address identifier `E0 01`.

| Action | Base Hexadecimal Payload |
| :----- | :----------------------- |
| **Load Preset** | `00 59 22 09 00 00 04 [PRESET_ID] 00 00 E0 01 00 00 01 [CS]` Note: `[PRESET_ID]` is 0-indexed (P01 = `00`, P80 = `4F`). |
| **Commit / UI Update** | `00 59 22 08 00 00 04 00 00 00 F0 00 00 00 0B` Note: Forces the physical OLED screen update and RAM reload. |

---

## Live Memory Write (RAM / Live Parameters)

Commands sent in real-time to alter the currently processed sound.

| Action | Base Hexadecimal Payload |
| :----- | :----------------------- |
| **Toggle Block** | `00 59 22 09 00 00 04 [BLOCK_ID] 00 00 10 01 00 00 [STATE] [CS]` `[STATE]` = `00` (Off) or `01` (On). |
| **Change Model** | `00 59 22 09 00 00 04 [MODEL_ADDR] 00 00 10 01 00 00 [MODEL_ID] [CS]` Changes the active effect type in the block. |
| **Change Knob** | `00 59 22 0A 00 00 04 [KNOB_ADDR] 00 00 10 02 00 00 [VAL_LSB] [VAL_MSB] [CS]` 17 bytes. Value formatted in Little Endian (0 to 100 decimal). |
| **Change Effect Chain (Routing)** | `00 59 22 0E 00 00 04 02 00 00 10 06 00 00 [B1] [B2] [B3] [B4] [B5] [B6] D4` 21 bytes. Alters the block order (Values: `00` to `05`). CS is fixed at `D4`. |

---

## Direct Flash Memory Access (DMA)

The pedal allows direct Read/Write access to the Flash memory for Preset manipulation (Backup & Restore). The size of a preset is strictly **92 bytes** (`0x5C`).

**Address Calculation:** `Address = Preset_Index * 92`
*(The address must be injected into the commands in a 4-byte Little Endian format).*

| Action | Base Hexadecimal Payload |
| :----- | :----------------------- |
| **Read Flash (Request)** | `00 59 23 08 00 00 04 [ADDR_4B] 5C 00 00 [CS]` Extracts a preset silently. Returns via Notify (`0x0064`). |
| **Write Flash (Save)** | `00 59 22 64 00 00 04 [ADDR_4B] 5C 00 00 55 [92_BYTES_PAYLOAD] [CS]` Permanently saves the matrix to the slot. The 92-byte Payload must always be preceded by the `55` marker. |

---

## State Reading (Polling / RAM Dump)

Command used to request the current state of all active parameters in RAM from the pedal (what the user is hearing and seeing on the screen right now).

| Action | Base Hexadecimal Payload |
| :----- | :----------------------- |
| **Request RAM Dump** | `00 59 23 08 00 00 04 00 00 00 10 5C 00 00 8F` The return is sent via Notify (`0x0064`) containing a continuous 92-byte dump, aligned with the Memory Map. |
