---
license: CC BY-SA 4.0
author: Joel Victor
---

# Blackbox Memory Map & State Dump

The memory architecture of the M-Vave Blackbox is perfectly sequential and aligned. The "State Dump" read from the Flash memory (or returned via RAM Polling) is a continuous array of **92 bytes** (`0x5C`).

## Reading Rules and Endianness (Decoding)

* **Offset Rule:** The Write Address (Hex ID) of a parameter (Write Command) is calculated using the formula: `Write_Address = Dump_Index + 2`.
* **Resolution Endianness:** Dynamic parameters (*Knobs*) have a 16-bit resolution. They are transmitted and received using the **Little Endian** format (Least Significant Byte first).
* **Normalized Scale:** Regardless of the physical screen displaying various ranges (-100 to +100, 10Hz to 18kHz, etc.), the pedal translates everything over Bluetooth into a strict decimal scale from **0 to 100** (Hex `00 00` to `64 00`). The display logic is handled by the client (App).

---

## The "6-Slot Rule" (Dynamic Architecture)

To ensure perfect 92-byte alignment per preset, the architecture permanently reserves **12 bytes (6 16-bit parameters)** for each effect block, regardless of whether the loaded pedal model uses all the knobs.
If a pedal only uses 3 knobs (like Cabinets), the addresses for knobs 4, 5, and 6 will still exist in memory as *Ghost Slots* (Reserved), filled with `00 00` and not affecting the DSP signal.

---

## Memory Mapping Table

| Write Addr (Hex) | Dump Index | Size | Block | Parameter | Mapped Values / Notes |
| :--------------: | :--------: | :--: | :---- | :-------- | :-------------------- |
| **`00`** | `00` | 2b | **GLOBAL** | Patch Volume | Master Level (`00 00` to `64 00`). |
| **`02`** | `02` | 6b | **ROUTING** | Signal Chain | `00`=FX, `01`=AMP, `02`=MOD, `03`=DLY, `04`=REV, `05`=CAB |
| **`08`** | `08` | 1b | **STATES** | FX On/Off | `00` (Off) / `01` (On) |
| **`09`** | `09` | 1b | **STATES** | AMP On/Off | `00` (Off) / `01` (On) |
| **`0A`** | `0A` | 1b | **STATES** | MOD On/Off | `00` (Off) / `01` (On) |
| **`0B`** | `0B` | 1b | **STATES** | DLY On/Off | `00` (Off) / `01` (On) |
| **`0C`** | `0C` | 1b | **STATES** | REV On/Off | `00` (Off) / `01` (On) |
| **`0D`** | `0D` | 1b | **STATES** | CAB On/Off | `00` (Off) / `01` (On) |
| **`0E`** | `14` | 1b | **MODELS** | FX Type | `00`=NoiseGate++, `01`=Boost, `02`=Compress, `03`=AIGate MS |
| **`0F`** | `15` | 1b | **MODELS** | AMP Type | `00` to `13` (Total: 20 Amps) |
| **`10`** | `16` | 1b | **MODELS** | MOD Type | `00`=Chorus, `01`=Phaser, `02`=Tremolo, `03`=Flanger, `04`=Vibrato, `05`=Univibe, `06`=Autofilter |
| **`11`** | `17` | 1b | **MODELS** | DLY Type | `00`=Analog, `01`=Duck, `02`=Dtape, `03`=Dual, `04`=Lofi |
| **`12`** | `18` | 1b | **MODELS** | REV Type | `00`=Room, `01`=Hall, `02`=Swell, `03`=Spring, `04`=Shimmer, `05`=Cloud |
| **`13`** | `19` | 1b | **MODELS** | CAB Type | `00` to `13` (Total: 20 Cabs/IRs) |

---

## Signal Chain (Routing)

The order of the pedals in the audio chain is defined by a continuous block of 6 bytes stored starting at address `02`. Each byte represents the physical position of an effect block in the chain, from left to right.

* **Block Dictionary:** `00`=FX, `01`=AMP, `02`=MOD, `03`=DLY, `04`=REV, `05`=CAB.
* **Default Example:** `00 01 02 03 04 05` (Order: FX -> AMP -> MOD -> DLY -> REV -> CAB)
* **Altered Example:** `04 00 01 02 03 05` (Order: REV -> FX -> AMP -> MOD -> DLY -> CAB)

---

### Dynamic Knob Parameter Dictionary

| Write Addr | Slot | AMP (Fixed) | CAB (Fixed) |
| :--------: | :--: | :---------- | :---------- |
| `20` / `50` | Knob 1 | Gain | Level |
| `22` / `52` | Knob 2 | Level | Low Cut (L-CUT) |
| `24` / `54` | Knob 3 | Bass | High Cut (H-CUT) *(Inverted)* |
| `26` / `56` | Knob 4 | Mid | *[Ghost]* |
| `28` / `58` | Knob 5 | Treble | *[Ghost]* |
| `2A` / `5A` | Knob 6 | *[Ghost]* | *[Ghost]* |

| Write Addr | Slot | FX 0: Noise Gate++ | FX 1: Boost | FX 2: Compress | FX 3: AI Gate MS |
| :--------: | :--: | :----------------- | :---------- | :------------- | :--------------- |
| `14` | Knob 1 | Gate | Gate | Gate | Gate |
| `16` | Knob 2 | *[Ghost]* | Gain | Sustain | Bias |
| `18` | Knob 3 | *[Ghost]* | *[Ghost]* | Attack | *[Ghost]* |
| `1A` | Knob 4 | *[Ghost]* | *[Ghost]* | Level | *[Ghost]* |
| `1C` | Knob 5 | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* |
| `1E` | Knob 6 | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* |

| Write Addr | Slot | MOD 0: Chorus | MOD 1: Phaser | MOD 2: Tremolo | MOD 3: Flanger | MOD 4: Vibrato | MOD 5: Univibe | MOD 6: Autofilter |
| :--------: | :--: | :------------ | :------------ | :------------- | :------------- | :------------- | :------------- | :---------------- |
| `2C` | Knob 1 | Speed | Speed | Speed | Speed | Speed | Speed | Speed |
| `2E` | Knob 2 | Depth | Midcut | Depth | Depth | Depth | Depth | Min |
| `30` | Knob 3 | Mix | Reso | Level | Feedback (FB) | *[Ghost]* | Mix | Max |
| `32` | Knob 4 | *[Ghost]* | Feedback (FB) | *[Ghost]* | Mix | *[Ghost]* | *[Ghost]* | Mix |
| `34` | Knob 5 | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | Feedback (FB) |
| `36` | Knob 6 | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* |

| Write Addr | Slot | DLY 0: Analog | DLY 1: Duck | DLY 2: Dtape | DLY 3: Dual | DLY 4: Lofi |
| :--------: | :--: | :------------ | :---------- | :----------- | :---------- | :---------- |
| `38` | Knob 1 | Time | Time | Time | Time | Time |
| `3A` | Knob 2 | Feedback (FB) | Feedback (FB) | Feedback (FB) | Feedback (FB) | Feedback (FB) |
| `3C` | Knob 3 | Mix | Mix | Mix | Mix | Mix |
| `3E` | Knob 4 | Phaser | Unpack | Grit | T-Mode | Grit |
| `40` | Knob 5 | Pitch | Speed | Speed | Speed | Speed |
| `42` | Knob 6 | *[Ghost]* | Depth | Depth | Depth | Depth |

| Write Addr | Slot | REV 0: Room | REV 1: Hall | REV 2: Swell | REV 3: Spring | REV 4: Shimmer | REV 5: Cloud |
| :--------: | :--: | :---------- | :---------- | :----------- | :------------ | :------------- | :----------- |
| `44` | Knob 1 | Decay | Decay | Decay | Decay | Decay | Decay |
| `46` | Knob 2 | Mix | Mix | Mix | Mix | Mix | Mix |
| `48` | Knob 3 | High Pass | High Pass | High Pass | High Pass | Tone | High Pass |
| `4A` | Knob 4 | Low Pass | Low Pass | Low Pass | Low Pass | Pitch | Low Pass |
| `4C` | Knob 5 | Depth | Depth | Rise-T | Combs | Amount (Amt) | Diffusion (Diff) |
| `4E` | Knob 6 | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* | *[Ghost]* |

---

## Customization and AMP / IR (CAB) Names

The pedal allows the user to load custom files to replace the factory models: Amplifier Captures (`.am4`) and Impulse Responses for Cabinets (`.wav`).

Although the BLE commands for changing the AMP (Addr `0F`) and CAB (Addr `13`) models only use the **numeric IDs (00 to 13 Hex)** to change the sound, the pedal stores the textual names of these files in hidden blocks in its Flash memory.

When the official application connects to the pedal (the phase where it shows messages like *"reading amps..."*), it issues Flash Read commands (`00 59 23`) to these hidden addresses to collect the actual names and synchronize the User Interface (UI).

* **AMP Names Base Address:** Flash `80 18 01 00` (Little Endian)
* **CAB (IR) Names Base Address:** Flash `90 18 01 00` (Little Endian)

### Name Translation (ASCII Decoding)

Reading these blocks returns long hexadecimal strings that are directly converted into **ASCII** characters. The names are delimited by null bytes (`00`).

Captured mapping examples from Flash to UI:

* **Hex:** `41 43 2d 53 65 56 69 6e` -> **Translation:** `AC-SeVin`
* **Hex:** `4a 56 4d 5f 31 39 36 30 5f 35 37` -> **Translation:** `JVM_1960_57`
* **Hex:** `56 4f 58 5f 41 43 33 30` -> **Translation:** `VOX_AC30`

*(Development Note: In a simple MIDI controller or Web Editor project, reading these blocks can be ignored, operating only with the Numeric ID matrix of `0 to 19`).*
