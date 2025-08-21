# joycon2py

A lightweight Python script that turns the Joy-Con 2's and the Pro Controller 2 into working PC Controllers.

---

## DISCLAIMER

THIS SPECIFIC PROJECT IS NO LONGER IN DEVELOPMENT BY ME. PLEASE CHECK OUT THE C++ VERSION:
https://github.com/TheFrano/joycon2cpp  
(however, you are free to use this version and make edits. just know this is more updated, has gyro and mouse, etc etc)



This project is **Windows-only**, primarily because `vgamepad` (used for virtual controller output) is exclusive to Windows.  
You're free to make your own macOS/Linux fork if you want.

Supported controllers:  
Joy-Con 2: Full support  
Pro Controller 2: Full support  
NSO GC Controller: Full support

If the program crashes, it means it couldn't connect to your joycon, often caused by:  
1. Disconnecting and connecting in a short amount of time. Fix - Wait a little before reconnecting them
2. Pairing through a button press - this never works, please use the Sync button as directed by the program

---

## Will there be GYRO or MOUSE support?

Their data WAS found, however vgamepad can't use the data. If you want this, check out the C++ version at the top.

##
---

## DEPENDENCIES

- Python (3.10+)
- [`bleak`](https://github.com/hbldh/bleak)  
  → `pip install bleak`  
- [`vgamepad`](https://github.com/yannbouteiller/vgamepad)  
  → `pip install vgamepad`  
  → Requires [ViGEmBus drivers](https://github.com/ViGEm/ViGEmBus/releases/latest) installed

---

## How do I use it?
- Download the whole repo (green button titled code, press download zip. you can delete readme.md if you want)
- Open the main.py script (dont open the other scripts those are modules)
- Pick your amount of players
- Pick everyone's controller
- If using a singular joycon you'll be asked if its Left or Right
- If using dual joycons itll ask you to pair one joycon then the other
- If using a pro controller/gc controller it just asks you to pair it
- When its all done, you'll have SDL controllers ready for every player to use.

- If the controller disconnects, the code will (eventually) notice the disconnection, and promptly wait for you to reconnect your controller. You do this by, again, holding down your sync button.

> 💡 Note: Bit layouts differ slightly between left and right Joy-Cons, so correct side pairing is important.
> 
---

# Joy-Con 2 BLE Notification Research

This document outlines some findings related to Joy-Con 2 BLE input behavior. If you're developing or reverse-engineering Joy-Con 2, Pro Controller 2, or other supported Nintendo controllers over BLE, this may be useful.

## ⚠️ Behavior Quirks

A notable quirk of these controllers is that if you attempt to connect or pair them repeatedly in a short time span, they may stop responding or fail to connect entirely for several minutes. This appears to be a controller-level cooldown behavior rather than an OS/BLE stack issue.

**If your controller stops connecting:**  
Wait a few minutes before trying again. It should recover on its own.

## 🔔 BLE Notification (with IMU enabled, Left Joy-Con)

Here’s an example notification received from a Joy-Con 2 via BLE, with the IMU command sent. (Pro Controller 2 and GC Controller notifications follow similar layouts but may shift certain fields.)

08670000000000e0ff0ffff77f23287a0000000000000000000000000000005f0e007907000000000001ce7b52010500beffb501ee0ffeff04000200000000


### Field Breakdown (based on known Joy-Con 2 layout)
huge thanks to [@german77](https://github.com/german77) for providing me with the notification layout below!!

| Offset | Size | Value              | Comment                      |
|--------|------|--------------------|------------------------------|
| `0x00` | 0x4  | Packet ID          | Sequence or timestamp        |
| `0x04` | 0x4  | Buttons            | Button state bitmap          |
| `0x08` | 0x3  | Left Stick         | 12-bit X/Y packed             |
| `0x0B` | 0x3  | Right Stick        | Likely garbage on Left Joy-Con |
| `0x0E` | 0x2  | Mouse X            | Unknown meaning               |
| `0x10` | 0x2  | Mouse Y            | Unknown meaning               |
| `0x12` | 0x2  | Mouse Unk          | Possibly extra motion data    |
| `0x14` | 0x2  | Mouse Distance     | Distance to IR/motion surface |
| `0x16` | 0x2  | Magnetometer X     |                              |
| `0x18` | 0x2  | Magnetometer Y     |                              |
| `0x1A` | 0x2  | Magnetometer Z     |                              |
| `0x1C` | 0x2  | Battery Voltage    | 1000 = 1V                     |
| `0x1E` | 0x2  | Battery Current    | 100 = 1mA                     |
| `0x20` | 0xE  | Reserved           | Undocumented region           |
| `0x2E` | 0x2  | Temperature        | `25°C + raw / 127`           |
| `0x30` | 0x2  | Accel X            | 4096 = 1G                     |
| `0x32` | 0x2  | Accel Y            |                              |
| `0x34` | 0x2  | Accel Z            |                              |
| `0x36` | 0x2  | Gyro X             | 48000 = 360°/s                |
| `0x38` | 0x2  | Gyro Y             |                              |
| `0x3A` | 0x2  | Gyro Z             |                              |
| `0x3C` | 0x1  | Analog Trigger L   |                              |
| `0x3D` | 0x1  | Analog Trigger R   |                              |

---

### 🧪 Field Example Breakdown

| Offset | Size | Field           | Raw Value     | Interpreted                  |
|--------|------|------------------|----------------|------------------------------|
| `0x00` | 4    | Packet ID        | `08 67 00 00`  | `0x00006708` → `26376`       |
| `0x04` | 4    | Buttons          | `00 00 00 00`  | No buttons pressed           |
| `0x08` | 3    | Left Stick       | `e0 ff 0f`     | X = `0x0FF0` = `4080`, Y = `0x0FE0` = `4064` |
| `0x0B` | 3    | Right Stick      | `ff f7 7f`     | Garbage on Left Joy-Con      |
| `0x2E` | 2    | Temperature      | `5f 0e`        | `0x0E5F` = `3679` → ~54°C     |
| `0x30` | 2    | Accel X          | `00 79`        | `0x7900` = `30976`           |
| `0x32` | 2    | Accel Y          | `07 00`        | `0x0007` = `7`               |
| `0x34` | 2    | Accel Z          | `00 00`        | `0`                          |
| `0x36` | 2    | Gyro X           | `01 ce`        | `0xCE01` = `52737`           |
| `0x38` | 2    | Gyro Y           | `7b 52`        | `0x527B` = `21115`           |
| `0x3A` | 2    | Gyro Z           | `01 05`        | `0x0501` = `1281`            |

---

### 📘 Notes

- Left Joy-Con **does not use Right Stick**, so data at `0x0B–0x0D` is typically junk.
- **Stick values** use 12-bit X/Y packed across 3 bytes:
  - X = upper 12 bits of first 1.5 bytes
  - Y = lower 12 bits of next 1.5 bytes
- **Accel/Gyro** fields are signed 16-bit:
  - Accelerometer: `4096 = 1G`
  - Gyroscope: `48000 = 360°/s`
- **Temperature**:  
  `25°C + (raw / 127)`  
  → `25 + (3679 / 127) ≈ 54°C`
- **Battery voltage**:  
  Reported as millivolts. `3000` = 3.0V. If `0x0000`, likely unavailable at that time.

---
