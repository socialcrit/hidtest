# VEC USB Footpedal & Keyboard/Mouse Emulation on a Single HID Remapper

Enable a single HID Remapper device to intercept any USB or GPIO input device and present itself to the host computer as an authentic **VEC USB Footpedal** (`VID=0x05F3`, `PID=0x00FF`, Manufacturer `"VEC"`, Product `"VEC USB Footpedal"`) sending raw 2-byte unnumbered footpedal reports (`01 00` Left, `02 00` Middle, `04 00` Right) on a dedicated USB HID interface, while simultaneously retaining full Keyboard, Mouse, and Consumer Control capabilities on a separate USB HID interface.

## User Review & Critical Decisions

> [!IMPORTANT]
> Please review the following architectural decisions that eliminate the need for a second CircuitPython Pico over GPIO and allow a single HID Remapper to work natively with footpedal software and standard OS keyboard/mouse drivers:

- **Confirmed Decision 1 — Dedicated Unnumbered USB HID Interface for Footpedal Reports**: Rather than embedding the footpedal inside the Keyboard/Mouse report descriptor with `Report ID 4` (which prepends a `0x04` report ID byte and causes host footpedal software to reject the packet), the footpedal descriptor (`Usage Page 0x0C, Usage 0x03`, `report_id = 0`, 2-byte input reports) is exposed on its own independent USB HID interface while Keyboard/Mouse/Consumer reports (`Report IDs 1, 2, 3`) remain on a separate USB HID interface.
- **Confirmed Decision 2 — Exact VEC USB Identification & Primary Interface Ordering**: The emulated device sets `VID = 0x05F3`, `PID = 0x00FF`, USB Manufacturer string `"VEC"`, and USB Product string `"VEC USB Footpedal"`, and exposes the 35-byte VEC Footpedal report descriptor so host software sees the exact `01 00` (Left), `02 00` (Middle), and `04 00` (Right) 2-byte packets.
- **Confirmed Decision 3 — Single-Button (No-Rollover) Sanitization & Collision-Free Mapping**: Footpedal output reports are sanitized so only one pedal bit (`0x01`, `0x02`, or `0x04` in byte 0, with byte 1 always `0x00`) is transmitted at a time (matching the behavior of your working GPIO loop), and synthetic footpedal target usages are isolated from unmapped ADC passthrough so floating analog pins never hold pedal buttons down.
- **Confirmed Decision 4 — Web Configuration Tool Preset & Reference**: The Web Configuration Tool includes the VEC Footpedal device mode, target usage buttons (`Footpedal left`, `Footpedal middle`, `Footpedal right`), and a ready-to-load example preset demonstrating how to map any intercepted input to footpedal pedals alongside standard keyboard and mouse actions.

---

## 1. Overview & Core Concept

- **What It Does**: Allows any USB HID device (mouse, keyboard, gamepad, pedal, button box) or GPIO switch connected to a single HID Remapper to trigger VEC USB Footpedal pedals (`Left = 01 00`, `Middle = 02 00`, `Right = 04 00`) while continuing to pass through or remap standard keyboard keys, mouse movement/clicks, media keys, layers, and macros.
- **Target Audience / Persona**: Users who need transcription, medical, legal, or media-control software expecting a hardware VEC / Infinity IN-USB-2 Footpedal (`05F3:00FF`) to accept inputs from arbitrary ergonomic controllers, keyboards, mice, or custom switches without needing a second microcontroller bridged over GPIO.
- **Key Value**: Eliminates the two-Pico GPIO hardware bridge by fixing the underlying USB composite interface layout, endpoint readiness checks, USB string descriptors, and single-pedal packet formatting directly inside HID Remapper.

---

## 2. User Experience & Visual Design

- **Key User Flows**:
  1. **Select Emulated Device Type**: In the Web Configuration Tool under **Settings → Emulated device type**, select **Footpedal (VEC USB Footpedal / Infinity IN-USB-2 + Keyboard & Mouse)**.
  2. **Map Any Input to Footpedal or Keyboard/Mouse**: Under **Mappings**, choose any input from the intercepted device (or use unmapped passthrough for normal keyboard/mouse operation) and assign target outputs to **Footpedal left**, **Footpedal middle**, or **Footpedal right**, alongside any standard Keyboard or Mouse outputs.
  3. **Load Quick Preset / Reference**: Under **Examples**, load the **VEC USB Footpedal + Keyboard & Mouse** configuration preset or inspect the stored CircuitPython/HID descriptor reference.
  4. **Save to Device**: Click **Save to device**; HID Remapper re-enumerates with `VID=0x05F3`, `PID=0x00FF`, Manufacturer `"VEC"`, Product `"VEC USB Footpedal"`, and immediately sends `01 00`, `02 00`, `04 00` packets when pedals are triggered.
- **Visual Identity & Theme**:
  - *Aesthetic Direction*: Clean, utilitarian hardware-configuration interface consistent with the existing HID Remapper Web Configuration Tool.
  - *Color Palette & Mood*: High-contrast light surface (`#FFFFFF` / `#F8FAFC`), crisp structural borders (`#DEE2E6`), primary action blue (`#0D6EFD`), and semantic category highlights for mouse, keyboard, gamepad, and footpedal usages.
  - *Typography & Hierarchy*: Clear sans-serif form labels and navigation tabs paired with tabular monospace (`font-mono tabular-nums`) for hex usage codes (`0xfff80001`–`0xfff80003`), VID/PID values (`05F3:00FF`), and live packet inspection.
  - *Component Styling & Layout*: Tabbed configuration workspace (`Mappings`, `Settings`, `Macros`, `Expressions`, `Actions`, `Monitor`, `Examples`) with modal usage pickers grouped by functional category.
- **Interactive Feedback & Motion**: Immediate checkmark confirmation badge on **Save to device**, live input state rows in the **Monitor** tab, and one-click preset loading in **Examples**.

---

## 3. Key Product Decisions & Trade-Offs

- **Decision 1: Separate USB HID Interface (`report_id = 0`) vs. Composite Report ID (`Report ID 4`)**
  - *Chosen Approach*: Expose the VEC Footpedal report descriptor on its own dedicated USB HID interface with no Report ID (`report_id = 0`, 2-byte reports), and expose Keyboard/Mouse/Consumer on a separate USB HID interface (`Report IDs 1, 2, 3`).
  - *Why*: Host software written for the VEC USB Footpedal (`05F3:00FF`) reads raw 2-byte input reports (`01 00`, `02 00`, `04 00`, `00 00`) directly from the footpedal HID collection and fails when a Report ID byte (`0x04`) is prepended or when the collection is merged into a keyboard/mouse descriptor.
  - *Alternatives Considered*: Merging `REPORT_ID_FOOTPEDAL (4)` into `our_report_descriptor_kb_mouse` was rejected because the host OS prepends `0x04` to every packet (`04 01 00`), breaking footpedal recognition.
- **Decision 2: Strict Single-Pedal (No-Rollover) Packet Sanitization**
  - *Chosen Approach*: Add a `footpedal_sanitize_report` handler for the footpedal report (`report_id = 0`) that enforces mutually exclusive single-button output (`01 00`, `02 00`, `04 00`, or `00 00`) and guarantees the second padding byte is always `0x00`.
  - *Why*: Matches the exact behavior discovered in your CircuitPython GPIO test where sending one button at a time (`0100` for L, `0200` for M, `0400` for R, no rollover) ensures 100% recognition by strict host software.
  - *Alternatives Considered*: Allowing simultaneous multi-bit combinations (`03 00`, `07 00`) was passed over as the default because certain transcription decoders match exact 16-bit values (`0x0001`, `0x0002`, `0x0004`) rather[...].
- **Decision 3: Multi-Interface Endpoint Readiness & Unmapped Passthrough Isolation**
  - *Chosen Approach*: Check `tud_hid_n_ready(itf)` for the specific target USB interface (`0`, `1`, or `2`) before dequeuing and transmitting reports, and exclude synthetic footpedal usages (`0xFFF80001`–`0xFFF80003`) from automatic unmapped passthrough so floating ADC inputs (`0xFFF80000` page) or mouse buttons (`0x00090001`–`0x00090003`) never accidentally trigger or jam footpedal reports.
  - *Why*: Prevents dropped footpedal packets when the mouse/keyboard interface is actively sending reports, and ensures mouse clicks remain normal mouse clicks unless explicitly mapped to footpedal pedals (or vice versa).

---

## 4. Technical Architecture & Data Strategy *(Technical Reference)*

- **Architecture & Component Diagram**:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                   Intercepted Input Devices (USB Host / GPIO)              │
│        (Any Mouse, Keyboard, Gamepad, Footpedal, or GPIO Switches)         │
└─────────────────────────────────────┬──────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                        HID Remapper Core Engine                            │
│  ┌───────────────────────────────┐    ┌─────────────────────────────────┐  │
│  │   Mapping & Layer Processor   │    │  Unmapped Passthrough Filter    │  │
│  │ Maps any source usage to:     │    │ Passes unmapped Keyboard/Mouse  │  │
│  │ • Footpedal L (0xFFF80001)    │    │ inputs through while isolating  │  │
│  │ • Footpedal M (0xFFF80002)    │    │ synthetic Footpedal targets     │  │
│  │ • Footpedal R (0xFFF80003)    │    └─────────────────────────────────┘  │
│  │ • Standard KB / Mouse usages  │                                         │
│  └───────────────┬───────────────┘                                         │
│                  │                                                         │
│                  ▼                                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │              Outgoing Report Router & Sanitizer                      │  │
│  │  • Report ID 0 (Footpedal): 2-byte [01 00 | 02 00 | 04 00 | 00 00]   │  │
│  │    with no-rollover sanitization (single active pedal bit)           │  │
│  │  • Report IDs 1, 2, 3 (Mouse, Keyboard, Consumer): Standard reports  │  │
│  └───────────────┬──────────────────────────────────────┬───────────────┘  │
└──────────────────┼──────────────────────────────────────┼──────────────────┘
                   │                                      │
                   ▼                                      ▼
┌─────────────────────────────────────┐┌─────────────────────────────────────┐
│   USB HID Interface (Footpedal)     ││  USB HID Interface (KB & Mouse)     │
│ • Descriptor: 35-byte VEC Footpad   ││ • Descriptor: Standard KB + Mouse   │
│ • Usage Page: 0x0C, Usage: 0x03     ││ • Report IDs: 1 (Mouse), 2 (KB),    │
│ • Report ID: None (0), Length: 2B   ││   3 (Consumer)                      │
└──────────────────┬──────────────────┘└──────────────────┬──────────────────┘
                   └──────────────────┬───────────────────┘
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│            Host Computer USB Stack (VID: 0x05F3, PID: 0x00FF)              │
│         Manufacturer: "VEC"  ·  Product: "VEC USB Footpedal"               │
│     (+ WebHID Config & Monitor Interface on Usage Page 0xFF00 / 0x0020)    │
└────────────────────────────────────────────────────────────────────────────┘
```

- **Data Model & State**:
  - **USB Device Descriptor (`our_descriptors[6]`)**:
    - `vid`: `0x05F3`
    - `pid`: `0x00FF`
    - `manufacturer_name`: `"VEC"`
    - `product_name`: `"VEC USB Footpedal"`
    - `our_report_descriptor_footpedal`: Exact 35-byte descriptor (`0x05, 0x0C, 0x09, 0x03, 0xA1, 0x01, 0x05, 0x09, 0x19, 0x01, 0x29, 0x03, 0x15, 0x00, 0x25, 0x01, 0x95, 0x03, 0x75, 0x01, 0x81, 0x02, 0x95, 0x01, 0x75, 0x05, 0x81, 0x01, 0x95, 0x01, 0x75, 0x08, 0x81, 0x01, 0xC0`).
  - **Synthetic Target Usages**:
    - `0xFFF80001`: Footpedal Left (maps to `report_id = 0`, bit 0 → `01 00`)
    - `0xFFF80002`: Footpedal Middle (maps to `report_id = 0`, bit 1 → `02 00`)
    - `0xFFF80003`: Footpedal Right (maps to `report_id = 0`, bit 2 → `04 00`)
- **Interactive Component & State Mapping**:
  - **Endpoint-Aware Report Queue (`send_report` & `do_send_report`)**: Checks `tud_hid_n_ready(itf)` for the exact USB interface corresponding to the queued report (`report_id == 0` on the footpedal interface vs. `report_id > 0` on the keyboard/mouse interface, and `config_itf = 2` for monitor reports) before dequeuing from `outgoing_reports`, preventing packet loss when keyboard/mouse and footpedal reports are generated on the same 1ms frame.
  - **No-Rollover Sanitizer (`footpedal_sanitize_report`)**: When `report_id == 0`, ensures byte 1 is `0x00` and masks byte 0 to a single active bit (`val & -val`, so if multiple pedals are pressed simultaneously, only the highest-priority single pedal bit `0x01`, `0x02`, or `0x04` is emitted).
  - **Web Config Tool (`Examples` & `Mappings`)**: Adds a dedicated **VEC USB Footpedal + Keyboard & Mouse** example preset and CircuitPython/HID reference note so users can configure or test footpedal mappings in one click.
