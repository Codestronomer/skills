---
name: ble-protocol-re-hci-snoop
description: "Use when reverse engineering a BLE device protocol, capturing HCI traffic from an Android phone, or parsing btsnoop logs — extract btsnoop via adb bugreport and parse ATT/GATT packets"
metadata:
  origin: auto-extracted
---

# BLE Protocol RE: HCI Snoop Capture + ATT Parsing

**Extracted:** 2026-11-21
**Context:** Reverse engineering proprietary BLE device protocols when the official app is available on Android but the protocol is undocumented

## Problem
BLE device protocols are invisible without either (a) a hardware sniffer or (b) capturing
the HCI traffic from the phone running the official app. Android stores HCI snoop logs in
root-only directories on most devices (Samsung: `/data/log/bt/btsnoop_hci.log`).

## Solution

### 1. Enable HCI snoop logging (no root needed)
```bash
adb shell settings put secure bluetooth_hci_log_enabled 1
# Toggle Bluetooth off/on to activate
```

### 2. Capture: use the official app, then toggle Bluetooth off to flush

### 3. Extract the log via adb bugreport (Samsung stores it in root-only /data/log/bt/)
```bash
adb bugreport bugreport.zip
unzip bugreport.zip FS/data/log/bt/btsnoop_hci.log
```

Wireless debugging for when USB is inconvenient:
```bash
# On Android 11+, discover via mDNS:
dns-sd -B _adb-tls-connect._tcp local.
dns-sd -L adb-<serial> _adb-tls-connect._tcp local.  # resolve port
adb connect IP:PORT
```

### 4. Parse btsnoop format
```
Header: "btsnoop\0" + version(4B BE) + datalink(4B BE, 1002=HCI UART)
Records: orig_len(4B) + incl_len(4B) + flags(4B, bit0: 0=sent,1=recv) + drops(4B) + packet
H4 packets: byte 0 = type (0x02=ACL, 0x04=event)
ACL: handle_flags(2B LE) + dlen(2B LE) + payload
L2CAP: length(2B LE) + cid(2B LE) — ATT cid = 0x0004
ATT: opcode(1B) + params
  0x52 = Write Command: handle(2B LE) + value
  0x1B = Handle Value Notification: handle(2B LE) + value
```

### 5. Reassemble ACL fragments
Accumulate payload fragments by connection handle; parse L2CAP header when the
accumulated length meets or exceeds the declared L2CAP length.

### 6. GATT characteristic discovery from the capture
The ATT discovery phase (READ_BY_TYPE_REQ/RSP, FIND_INFO_REQ/RSP) reveals all
characteristic handles and UUIDs. The app's actual write/notify handles may differ
from what you'd expect from the service layout — always use the handles observed
in the capture.

### 7. Identify the protocol framing
Look for:
- Magic header bytes (e.g. `0x5A5A`, `0xAA`) in the first payload bytes
- Length fields (usually at fixed offset after magic)
- Frame structure: magic + flags/version + length + payload + checksum
- The checksum algorithm (common: CRC16-ARC reflected 0xA001 init 0)

## Key tips
- Samsung build-tools `dexdump` is useful for quick class listing, but `jadx` is better for full decompilation
- Reassemble ACL fragments: accumulate by handle, parse L2CAP when complete
- Decompile the Android app with `jadx` to understand the protocol from the software side simultaneously
- JNI native functions in the app may handle crypto — pull the `.so` from the phone's split APK via `adb shell unzip -l /data/app/.../split_config.arm64_v8a.apk`

## When to Use
- Reverse engineering any BLE device protocol
- Need to see what the official app sends to the device
- No hardware BLE sniffer available
- Device uses custom GATT services with proprietary framing
- Need to understand the pairing/session-establishment handshake
