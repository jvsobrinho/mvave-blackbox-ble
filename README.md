# M-Vave Blackbox - Reverse Engineering Project

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](./LICENSE-GPL)
[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC_BY--SA_4.0-lightgrey.svg)](./LICENSE-CC)

This repository documents the proprietary communication protocol of the **M-Vave Blackbox** multi-effects pedal. The goal is to provide the necessary technical foundation for creating third-party editors (Web/Desktop) and physical hardware controllers based on microcontrollers like the ESP32.

## Hardware Overview

* **Device Name:** `BlackBox_BLE`
* **Protocol:** Bluetooth Low Energy (BLE) / GATT
* **USB Interface:** Class-Compliant MIDI (SysEx Base-128 Bit-packing)

> **TECHNICAL NOTE (BLE SysEx Implementation Status):**
> Analysis of the official app logs shows it strictly relies on the custom GATT data tunnel (`ae40`) for binary communication. Although wired USB communication wraps the Base-128 protocol inside standard MIDI SysEx packets (as documented in [USB SysEx Protocol](docs/usb_sysex_protocol.md)), it **has not been tested** whether the pedal's Bluetooth stack accepts these same SysEx payloads over BLE MIDI when injected with proper BLE MIDI timestamps.

## The "Hidden" Protocol (Why this project exists)

While the official M-Vave manual documents standard MIDI commands (PC for presets, CC for basic toggles), the pedal's official app uses a completely different, **undocumented proprietary hexadecimal protocol**.

Why does this matter? Standard MIDI over BLE on this device is severely limited. It does not allow you to read the names of the IRs, extract full presets, or manipulate the signal chain routing. This reverse-engineering project bypasses the standard MIDI layer entirely, interacting directly with the proprietary `ae40` service to achieve 100% control over the DSP and Memory architecture.

## Service UUIDs and Handles

Bluetooth communication relies on the proprietary `ae40` service.

| Characteristic | UUID | Handle | Function |
| :--- | :--- | :--- | :--- |
| **TX (Write)** | `ae41` | `0x0062` | Command dispatch (RAM and Flash) |
| **RX (Notify)** | `ae42` | `0x0064` | State reception (State Dump) |

## Repository Contents

* `/docs`: Detailed technical specifications of the protocol, physical memory maps, and transport layers.
* `/tools`: Python reference scripts for preset reading and checksum validation.
* `/logs_reference`: Raw hexadecimal packet dumps captured via Wireshark.

## Acknowledgments for USB Protocol Analysis

While the base protocol was independently reverse-engineered via HCI snoop logs over BLE, the translation of this protocol to USB SysEx (Base-128 bit-packing) was confirmed and understood thanks to the prior investigative work and code from the open-source community:

* **Cube Baby Rust Parser** (GPL-3.0) by [pferreir](https://github.com/pferreir/cuvave-midi) - Provided the algorithmic structure of the Base-128 encode/decode process.
* **M-VAVE/CUVAVE Looper Pro protocol notes** (Gist) by [michaelforney](https://gist.github.com/michaelforney/d3a2790bb5f8cbcb1c931eabb50b5f20) - Validated the SysEx encapsulation payload structure.
* **M-Vave Chocolate SysEx dumps** by [cbix](https://github.com/cbix/mvave-chocolate-sysex) - Provided the empirical hex dump proof of the protocol.

---

## Disclaimer

**Note:** This project is an independent open-source initiative and is not endorsed by, affiliated with, or connected to M-Vave.

* **M-VAVE** and **CUVAVE** are registered trademarks of **Zhuhai Shengke Intelligent Technology Co., Ltd.**
* Any brand names, trademarks, or model names mentioned in this project are the property of their respective owners and are used solely for descriptive and educational purposes.

The use of direct flash memory write commands (Save) and the provided scripts is entirely at the user's own risk. The author is not responsible for any bricked devices, hardware damage, or data loss.

## Licensing

This is a mixed-license project to ensure both the software tools and the discovered knowledge remain open:

* **Source Code**: All scripts, tools, and code (e.g., `.py`, `.js`, `.html`) are licensed under the **GNU General Public License v3.0 (GPL-3.0)**. See [LICENSE-GPL](./LICENSE-GPL) for details.
* **Documentation & Research**: All technical specifications, protocol research, and memory maps (e.g., `.md` files in `/docs`) are licensed under the **Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**. See [LICENSE-CC](./LICENSE) for details.

By contributing to this project, you agree that your contributions will be licensed under these same terms.