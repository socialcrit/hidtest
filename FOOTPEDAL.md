# VEC USB Footpedal (`05F3:00FF`) & PowerScribe One RP2040 Consolidation Reference

## 1. USB Identification & 3-Interface Composite Architecture

To satisfy applications such as Nuance PowerScribe One (which query `VID=0x05F3`, `PID=0x00FF` and require a standalone Report-ID-less 2-byte HID interface on Interface 0) while retaining full HID Remapper Keyboard, Mouse, and WebHID Configurator capabilities on a single RP2040, the firmware exposes a 3-interface USB composite layout:

| Interface | Constant | Endpoint | Purpose |
| :--- | :--- | :--- | :--- |
| **Interface 0** | `ITF_NUM_PEDAL = 0` | `EPNUM_PEDAL = 0x81` | Standalone 29-byte Report-ID-less VEC Footpedal (`Usage Page 0x0C`, `Usage 0x03`, 2-byte unnumbered reports: `01 00` Left, `02 00` Middle, `04 00` Right) |
| **Interface 1** | `ITF_NUM_HID = 1` | `EPNUM_HID = 0x82` | Standard HID Remapper composite Keyboard, Mouse, Media, and Gamepad output (`Report IDs 1–5`) |
| **Interface 2** | `ITF_NUM_CONFIG = 2` | `EPNUM_CONFIG = 0x83` | HID Remapper WebHID Configurator & Input Monitor (`Usage Page 0xFF00`) |

## 2. Internal Mapping via `REPORT_ID_PEDAL (9)` (`0x00090011`–`0x00090013`)

Inside `our_report_descriptor_kb_mouse` and `our_report_descriptor_absolute`, an internal `REPORT_ID_PEDAL (9)` collection maps `Button 17`–`Button 19` (`0x00090011`–`0x00090013`):

- **`0x00090011`**: Footpedal left (`Button 17`) → Bit 0 (`01 00`)
- **`0x00090012`**: Footpedal middle (`Button 18`) → Bit 1 (`02 00`)
- **`0x00090013`**: Footpedal right (`Button 19`) → Bit 2 (`04 00`)

When `send_report()` in `firmware/src/remapper.cc` processes a report with `report_id == REPORT_ID_PEDAL (9)`, it strips the internal Report ID (`9` → `0`) and transmits the 2-byte payload directly out of **Interface 0 (`ITF_NUM_PEDAL = 0`)**. Standard Keyboard and Mouse reports are transmitted out of **Interface 1 (`ITF_NUM_HID = 1`)**, and WebHID monitor reports are transmitted out of **Interface 2 (`ITF_NUM_CONFIG = 2`)**.

## 3. Original CircuitPython Reference

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
)) # 67 bytes

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
