# VEC USB Footpedal (Infinity IN-USB-2) Emulation Reference

## Stored CircuitPython Reference (`boot.py` / `code.py`)

```python
FOOTPAD_REPORT_DESCRIPTOR = bytes((  
    0x05, 0x0C,  #   Usage Page (Consumer)    
    0x09, 0x03,  #   Usage (Programmable Buttons)
    0xA1, 0x01,  #   Collection (Application)    
    0x05, 0x09,  #   Usage Page (Button)
    0x19, 0x01,  #   Usage Minimum (0x01)
    0x29, 0x03,  #   Usage Maximum (0x03)
    0x15, 0x00,  #   Logical Minimum (0)
    0x25, 0x01,  #   Logical Maximum (1)
    0x95, 0x03,  #   Report Count (3)  Button? (1 bit per each of 3 buttons)
    0x75, 0x01,  #   Report Size (1)
    0x81, 0x02,  #   Input
    0x95, 0x01,  #   Report Count (1)
    0x75, 0x05,  #   Report Size (5)  5 bits of padding?
    0x81, 0x01,  #   Input
    0x95, 0x01,  #   Report Count (1)
    0x75, 0x08,  #   Report Size (8)  8 bits of padding?
    0x81, 0x01,  #   Input
    0xC0,        #   End Collection
)) # 35 bytes

footpad = usb_hid.Device(
    report_descriptor=FOOTPAD_REPORT_DESCRIPTOR,
    usage_page=0x0C,           #
    usage=0x03,                #
    report_ids=(0,),           #
    in_report_lengths=(2,),    # This footpad sends 3 bits of data + 5 bits of padding + 8 bits of padding in its report.
    out_report_lengths=(0,),   # It does not receive any reports.
)

supervisor.set_usb_identification(
    manufacturer='VEC',
    product='VEC USB Footpedal',
    vid=0x05F3, 
    pid=0x00FF 
)
```

## Single-Pico Composite Architecture in HID Remapper (`our_descriptors[6]`)

### Why Embedding `REPORT_ID_FOOTPEDAL (4)` in the Keyboard/Mouse Descriptor Failed
When the footpedal collection was merged into the primary Keyboard/Mouse report descriptor using `Report ID 4`, the USB HID stack prepended `0x04` to every footpedal packet (`04 01 00` instead of `01 00`). Host software expecting a VEC USB Footpedal (`VID=0x05F3`, `PID=0x00FF`) opens the `Usage Page 0x0C / Usage 0x03` collection expecting an **unnumbered** 2-byte report (`report_id = 0`, length `2`: `01 00` for Left, `02 00` for Middle, `04 00` for Right, `00 00` for released).

### How HID Remapper Emulates Both on a Single Pico
1. **Separate USB HID Interfaces**:
   - **Interface 0 (`0x81`)**: Standard Keyboard, Mouse, and Consumer Control (`Report IDs 1, 2, 3`).
   - **Interface 1 (`0x82`)**: Dedicated VEC USB Footpedal (`our_report_descriptor_footpedal`, `Usage Page 0x0C, Usage 0x03`, `report_id = 0`, 2-byte input reports).
   - **Interface 2 (`0x83`)**: WebHID Configuration & Monitor interface (`Usage Page 0xFF00, Usage 0x0020`).
2. **Exact USB Device Identification**:
   - `idVendor = 0x05F3`, `idProduct = 0x00FF`
   - `iManufacturer = "VEC"`, `iProduct = "VEC USB Footpedal"`
3. **Single-Button (No-Rollover) Packet Sanitization (`footpedal_sanitize_report`)**:
   - Ensures each 2-byte footpedal report on `report_id = 0` emits at most one active button bit at a time (`01 00` Left, `02 00` Middle, `04 00` Right, or `00 00` released), matching the single-button GPIO bridge behavior while keeping full Keyboard & Mouse passthrough active.
4. **Target Usages in Web Config Tool**:
   - `0xfff80001` — **Footpedal left** (`01 00`)
   - `0xfff80002` — **Footpedal middle** (`02 00`)
   - `0xfff80003` — **Footpedal right** (`04 00`)
