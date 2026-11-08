# PowerScribe One 3-Interface VEC Foot Pedal & RP2040 HID Remapper Architecture

Revert the previous custom descriptor changes and implement the exact 3-interface RP2040 USB composite architecture from the **PowerScribe One UIA & RP2040 HID Remapper Reference** document so Nuance PowerScribe One (`Nuance.PSOne.exe`) attaches to **Interface 0 (`MI_00`)** as a genuine **VEC Infinity USB Foot Pedal** (`VID=0x05F3`, `PID=0x00FF`) receiving pure 2-byte Report-ID-less packets, while **Interface 1 (`MI_01`)** provides full Keyboard & Mouse output and **Interface 2 (`MI_02`)** provides WebHID configuration.

## User Review & Critical Decisions

> [!IMPORTANT]
> Summary of decisions confirmed with your preferences and aligned with Section 4 & Section 5 of the attached reference document:

- **Confirmed Decision 1 — Global 3-Interface USB Composite Layout (`MI_00`, `MI_01`, `MI_02`)**:
  - **Interface 0 (`ITF_NUM_PEDAL = 0`, `EP 0x81`)**: Dedicated to the 29-byte, Report-ID-less `pedal_report_descriptor` (`Usage Page 0x0C, Usage 0x03`, 3 buttons + 13 bits constant padding = 2 bytes). PowerScribe One attaches to the first interface (`MI_00`) of `VID_05F3&PID_00FF` and receives pure 2-byte reports (`[01 00]`, `[02 00]`, `[04 00]`, `[00 00]`) with `report_id = 0` on the wire.
  - **Interface 1 (`ITF_NUM_HID = 1`, `EP 0x82`)**: Standard HID Remapper composite Keyboard, Mouse, and Consumer Control output.
  - **Interface 2 (`ITF_NUM_CONFIG = 2`, `EP 0x83`)**: HID Remapper Web Configurator & Monitor (`Usage Page 0xFF00`).
- **Confirmed Decision 2 — Internal `REPORT_ID_PEDAL (9)` Mapped to Button 17–19 (`0x00090011`–`0x00090013`)**:
  - Reverts the previous `0xFFF80000` / `secondary_descriptor` approach and instead embeds an internal `REPORT_ID_PEDAL (9)` collection inside `our_report_descriptor_kb_mouse` (and `our_report_descriptor_absolute`) mapped to **Button 17–Button 19 (`0x00090011`–`0x00090013`)** (`MAX_INPUT_REPORT_ID = 9`).
  - `parse_our_descriptor()` automatically exposes `0x00090011`–`0x00090013` to both the local Web Configurator and upstream `remapper.org/config` without colliding with standard mouse buttons (`Button 1`–`Button 8`).
  - `send_report()` intercepts `report_id == REPORT_ID_PEDAL (9)` and transmits the 2-byte payload out of **Interface 0 (`ITF_NUM_PEDAL = 0`)** with **`report_id = 0`** (omitting the Report ID byte on the wire), while routing all other reports to **Interface 1 (`ITF_NUM_HID = 1`)**.
- **Confirmed Decision 3 — Web Configurator Picker Buttons & CMake Submodule Fix**:
  - Exposes **Footpedal left (Button 17)** (`0x00090011`), **Footpedal middle (Button 18)** (`0x00090012`), and **Footpedal right (Button 19)** (`0x00090013`) directly in the Web Configurator target picker.
  - Preserves the CMake and GitHub Actions submodule fetch fix so RP2040 UF2 firmware artifacts compile cleanly on GitHub Actions.

---

## 1. Overview & Core Concept

- **What It Does**: Consolidates a VEC Infinity USB Foot Pedal (`VID 0x05F3`, `PID 0x00FF`) and a full HID Remapper (Keyboard/Mouse/Macros/Expressions) onto a single RP2040 microcontroller compatible with Nuance PowerScribe One (`Nuance.PSOne.exe`).
- **Target Audience / Persona**: Radiologists and transcription workflows using Nuance PowerScribe One who want any USB HID input device or GPIO switch intercepted by a single RP2040 to drive hardware-level VEC Foot Pedal controls alongside standard keyboard/mouse remapping and AutoHotkey v2 UIA scripts.
- **Key Value**: Avoids Windows `hid.dll` virtual child collection (`Col01`/`Col04`) and Report-ID prefix shifting by splitting the VEC Foot Pedal onto USB Interface 0 (`MI_00`, `EP 0x81`) at the `usbccgp.sys` bus level while keeping full compatibility with `remapper.org/config`.

---

## 2. User Experience & Visual Design

- **Key User Flows**:
  1. **Connect & Open Device**: Plug in the RP2040 running the updated firmware. Windows enumerates `VID_05F3&PID_00FF` with `MI_00` (VEC Foot Pedal), `MI_01` (Keyboard/Mouse), and `MI_02` (Web Configurator).
  2. **Map Inputs to Pedal Buttons 17–19**: In the Web Configurator (`Mappings` tab), map any input button, key, or GPIO pin to **Footpedal left (Button 17)** (`0x00090011`), **Footpedal middle (Button 18)** (`0x00090012`), or **Footpedal right (Button 19)** (`0x00090013`), or enter those codes as Custom Usages on `remapper.org/config`.
  3. **PowerScribe One Operation**: When triggered, Interface 0 (`MI_00`) emits raw 2-byte reports (`[01 00]`, `[02 00]`, `[04 00]`, `[00 00]`) with no Report ID byte; Windows `hid.dll` hands PowerScribe One the expected `[0x00, button_bitmask, 0x00]` buffer while Interface 1 (`MI_01`) handles all normal keyboard and mouse events.
- **Visual Identity & Theme**:
  - *Aesthetic Direction*: Clean, functional hardware configuration tool with direct target buttons for `Button 17`–`Button 19` and a built-in PowerScribe One UIA & RP2040 reference panel in the **Examples** tab.
  - *Typography & Hierarchy*: Crisp sans-serif UI controls paired with tabular monospace (`font-mono tabular-nums`) for usage hex codes (`0x00090011`–`0x00090013`), endpoint addresses (`0x81`–`0x83`), and AutoHotkey v2 UIA snippets.
- **Interactive Feedback & Motion**: One-click preset loading in the **Examples** tab and immediate visual confirmation when saving configuration to the device.

---

## 3. Key Product Decisions & Trade-Offs

- **Decision 1: Bus-Level Interface Split (`MI_00` Pedal, `MI_01` Keyboard/Mouse, `MI_02` Config)**
  - *Chosen Approach*: Place `pedal_report_descriptor` on `ITF_NUM_PEDAL = 0` (`EP 0x81`), `our_descriptor->descriptor` on `ITF_NUM_HID = 1` (`EP 0x82`), and `config_report_descriptor` on `ITF_NUM_CONFIG = 2` (`EP 0x83`).
  - *Why*: Windows `usbccgp.sys` splits separate USB interfaces before `hid.dll` parses them, allowing PowerScribe One to attach directly to `MI_00` and read unshifted 2-byte reports without Report ID prefixes.
- **Decision 2: Internal `REPORT_ID_PEDAL = 9` Mapped to `0x00090011`–`0x00090013` (Button 17–19)**
  - *Chosen Approach*: Include a `REPORT_ID_PEDAL (9)` collection inside `our_report_descriptor_kb_mouse` for `Button 17`–`Button 19` (3 bits data + 13 bits constant padding = 2 bytes), then strip the report ID in `send_report()` by sending `REPORT_ID_PEDAL` out of Interface 0 with `report_id = 0`.
  - *Why*: Allows `parse_our_descriptor()` to automatically register the 2-byte report buffer and advertise `0x00090011`–`0x00090013` via `GET_OUR_USAGES` without conflicting with Mouse `Button 1`–`Button 8` (`0x00090001`–`0x00090008`).

---

## 4. Technical Architecture & Data Strategy *(Technical Reference)*

- **Architecture & Component Diagram**:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                Intercepted Input Devices (USB Host / GPIO)                 │
└─────────────────────────────────────┬──────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                     HID Remapper Core Engine (remapper.cc)                 │
│  • Parses our_report_descriptor_kb_mouse including internal                │
│    REPORT_ID_PEDAL (9) for Button 17..19 (0x00090011..0x00090013)          │
│  • Allocates 2-byte buffer reports[9] (MAX_INPUT_REPORT_ID = 9)            │
└──────────────────┬──────────────────┴───────────────────┬──────────────────┘
                   │                                      │
   report_id == 9  │ (Intercepted in send_report)         │ report_id != 9
                   ▼                                      ▼
┌─────────────────────────────────────┐┌─────────────────────────────────────┐
│  Interface 0: ITF_NUM_PEDAL (MI_00) ││   Interface 1: ITF_NUM_HID (MI_01)  │
│  • Endpoint: 0x81                   ││   • Endpoint: 0x82                  │
│  • Descriptor: pedal_report_desc    ││   • Descriptor: our_descriptor      │
│    (29 bytes, NO Report ID)         ││     (Keyboard, Mouse, Consumer)     │
│  • Sent via:                        ││   • Sent via:                       │
│    tud_hid_n_report(0, 0, buf, 2)   ││     tud_hid_n_report(1, id, buf, n) │
└──────────────────┬──────────────────┘└──────────────────┬──────────────────┘
                   │                                      │
                   └──────────────────┬───────────────────┘
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│         Windows usbccgp.sys / hid.dll (VID: 0x05F3, PID: 0x00FF)           │
│  • MI_00 -> PowerScribe One reads [0x00, bitmask, 0x00] (L=01, M=02, R=04) │
│  • MI_01 -> Windows Keyboard & Mouse HID stack                             │
│  • MI_02 -> Endpoint 0x83 Web Configurator (Usage Page 0xFF00)             │
└────────────────────────────────────────────────────────────────────────────┘
```

- **Data Model & State**:
  - `REPORT_ID_PEDAL = 9` (`MAX_INPUT_REPORT_ID = 9` so `reports[9]`, `prev_reports[9]`, `report_masks_absolute[9]`, and `report_sizes[9]` are properly sized).
  - `pedal_report_descriptor`: 29-byte standalone descriptor (`0x05, 0x0C, 0x09, 0x03, 0xA1, 0x01, 0x05, 0x09, 0x19, 0x01, 0x29, 0x03, 0x15, 0x00, 0x25, 0x01, 0x95, 0x03, 0x75, 0x01, 0x81, 0x02, 0x95, 0x01, 0x75, 0x0D, 0x81, 0x01, 0xC0`).
  - `ITF_NUM_PEDAL = 0` (`EP 0x81`), `ITF_NUM_HID = 1` (`EP 0x82`), `ITF_NUM_CONFIG = 2` (`EP 0x83`).
- **Interactive Component & State Mapping**:
  - **Report Routing (`send_report` & `do_send_report`)**: When `report_id == REPORT_ID_PEDAL (9)`, checks `tud_hid_n_ready(0)` and calls `tud_hid_n_report(0, 0, report, 2)` to emit the unnumbered 2-byte pedal packet on Interface 0; for all other report IDs, checks `tud_hid_n_ready(1)` and calls `tud_hid_n_report(1, report_id, report, size)` on Interface 1.
  - **Web Configurator Routing (`ITF_NUM_CONFIG = 2`)**: Routes `handle_get_report1` / `handle_set_report1` and `send_monitor_report` to USB Interface 2 (`itf == 2`).
