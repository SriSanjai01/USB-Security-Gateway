# 🛡️ USB Security Gateway

**ESP32-P4NRW32-Based Hardware USB Threat Detection & Access-Control Gateway**

USB Security Gateway is a standalone hardware firewall that sits between an untrusted USB device and a host PC. It identifies a device from its descriptors, applies a whitelist and class policy, monitors its power draw, logs every event offline, and can cut the device off in hardware (power and data) even if the firmware hangs. It is built around the ESP32-P4NRW32 and was developed for the PCB Hackathon 2026.

---

## 📌 Project Overview

USB is the most common way for untrusted hardware to touch a computer. Operating systems trust a device the moment it enumerates, so a keyboard-emulating flash drive or a device with abnormal power draw can compromise a host within seconds. Labs, kiosks, industrial control PCs and shared workstations need a small unit that decides what a device is allowed to be before the host ever sees it.

This project provides that unit by combining:

- Dual-USB proxy architecture (High-Speed host side, Full-Speed device side)
- Descriptor-based whitelisting (VID / PID / class / serial)
- Hardware power and data isolation
- Live current monitoring of the untrusted port
- Secure element for signed whitelist updates
- Local display, LEDs, buzzer and offline microSD logging

## 🎯 Applications

- Labs and shared workstations
- Public kiosks
- Industrial control PCs
- Secure / air-gapped environments
- Factory automation HMIs
- USB security research and education

## 🚀 Features

- 🔌 **Proxy firewall:** the P4 is the USB host for the unknown device and the USB device for the PC, so the PC only sees a sanitized, re-created device
- 📋 **Descriptor whitelisting:** decisions are ALLOW, BLOCK, MONITOR or QUARANTINE
- ⚡ **Hardware isolation:** TPS2553 VBUS load switch + TS3USB221A High-Speed data switch
- 🔒 **Fail-safe defaults:** port unpowered and data open after reset until firmware enables it
- 🖐 **Hardware override buttons:** S2 cuts VBUS and S3 opens the data lines while held, independent of firmware
- 📈 **Current monitor:** INA219 flags abnormal power draw on the untrusted port
- 🔐 **Security:** ESP32-P4 secure boot v2, flash encryption, HMAC, ECDSA, TRNG and an ATECC608B secure element
- 🖥 **Local feedback:** 2.0" ST7789 TFT, four status LEDs, buzzer, approve/block buttons
- 💾 **Offline logging:** microSD card on SDMMC slot 1
- 🛠 **Debug friendly:** USB-Serial-JTAG, UART0 / I2C / UART1 / SPI headers and test points
- 🧱 **4-layer PCB** with a solid ground plane

## 🏗 System Architecture
<img width="1456" height="836" alt="image" src="https://github.com/user-attachments/assets/0da7312a-475b-4e46-929b-f31750c5bc43" />

## Power and Functional Architecture
<img width="1276" height="724" alt="image" src="https://github.com/user-attachments/assets/cab58559-61e8-405f-ba90-9738a4a73fef" />

Untrusted USB device -> J1 USB-A -> U8 data switch -> ESP32-P4 (USB host, HS)
                                                          |
                                                    whitelist + policy
                                                          |
Host PC <- J2 USB-C <- ESP32-P4 (USB device, FS) <- sanitized device
```

## 🧩 Hardware Components

| Component | Purpose |
|---|---|
| ESP32-P4NRW32 (U3) | Main controller, HS USB host + FS USB device, 32 MB in-package PSRAM |
| W25Q128JVSIQ (U4) | 16 MB QSPI flash (firmware, OTA, whitelist) |
| ISL80102IRAJZ (U1) | 3.3 V linear regulator |
| TPS7A1111PDRVT (U2) | 1.1 V linear regulator for the P4 core |
| TPS2553QDBVRQ1 (U6) | VBUS load switch with current limit and fault flag |
| INA219AIDR (U7) | Current monitor on the untrusted port (I2C) |
| TS3USB221ARSER (U8) | High-Speed USB D+/D- data switch |
| ATECC608B-MAHCZ-T (U14) | Secure element for signed whitelist updates |
| TPD4E05U06 (U5) | Low-capacitance (≤1 pF) ESD array for the HS pair |
| USBLC6-2SC6 (U9, U10) | ESD protection for the PC port and debug port |
| TPD2S017 (U11–U13) | ESD protection for UART / I2C header lines |
| ST7789 2.0" TFT (J8) | Status display |
| microSD socket (J9) | Offline event log |
| MLT-8530 buzzer (B1) | Audible alert |

## ⚙ Working Principle

1. The 5 V USB input powers the board; linear regulators make 3.3 V and 1.1 V.
2. On reset, U6 (VBUS) is off and U8 (data) is open, so the untrusted port is dead.
3. Firmware boots, checks its configuration and whitelist, and waits for a device on J1.
4. When a device appears, firmware enables U6 (limited power) and then U8 (data path).
5. The P4 acts as USB host and reads VID, PID, class, serial and other descriptors.
6. The INA219 reports current; abnormal draw or unexpected behaviour raises an alert.
7. If allowed, firmware presents a sanitized USB device to the PC through J2. If not, it disables U8 and U6 and logs the event.
8. The decision is shown on the TFT and LEDs, the buzzer sounds for a block, and the event is written to the microSD log.
9. At any time, holding S2 forces power off and holding S3 forces the data lines open.

> This is switching isolation, not galvanic isolation: grounds are shared.

## 🔧 Protection Features

- Schottky diode on the 5 V input (reverse-current protection)
- TVS diode and 100 µF bulk capacitor on VBUS_A
- Current-limited, fault-reporting VBUS switch
- ≤1 pF ESD array on the High-Speed pair
- ESD arrays on the PC port, debug port, headers and tamper input
- Fail-safe defaults (pull-down on U6 EN, pull-up on U8 /OE)
- PC VBUS is only sensed through a divider, never joined to the board supply (no back-feed)
- Decoupling at every supply pin and a solid ground plane
- Series resistors on the debug USB lines, microSD lines and display clock

## 🧱 2-Layer → 4-Layer: What I Learned

This is my first 4-layer board. Planned stack-up:

| Layer | Use |
|---|---|
| L1 | Signal and components |
| L2 | Solid ground plane |
| L3 | Power and signal |
| L4 | Signal |

## ⚠ Known Limitations

- Detection is descriptor-level; it does not scan file contents or detect malware.
- The PC-side link is Full-Speed (12 Mbps); the untrusted side can be High-Speed.
- Transparent High-Speed pass-through, packet capture and SuperSpeed need extra hardware.
- The ESP-IDF host stack has no hub support, so one device is attached directly (no hub).
- The S2/S3 override is momentary (only while held).
- The board uses linear regulators, so heat grows with load and needs a thermal check.
- Wi-Fi and Bluetooth are not on the P4 (an ESP32-C6 companion chip would be needed).

## 📂 Repository Structure

```
USB-Security-Gateway/
│
├── Hardware/
│   ├── Schematic/
│   ├── PCB/
│   └── Gerber/
│
│
├── Images/
│   ├── PCB/
│   ├── 3D/
│   └── BlockDiagram/
│
├── Documentation/
│   └── Project_Report.docx
│
└── README.md
```

## 🛠 Software & Tools

- Altium Designer
- GitHub

## 📈 Future Improvements

- Wi-Fi / BLE alerts through an ESP32-C6 companion
- Optional Ethernet alerts / syslog
- Latching hardware lock switch
- Input fuse and surge protection
- Signed whitelist update tool for PC
- Switching regulator for better efficiency
- Transparent High-Speed pass-through with a USB hub / bridge IC
- Enclosure with tamper switch

## 👨‍💻 Team

**Circuit Architects**

- Sri Sanjai A – Electronics and Communication Engineering
- Vinith M K – Electrical and Electronics Engineering

**Institution:** Bannari Amman Institute of Technology

## 📜 License

This project was developed as part of the PCB Hackathon 2026 and to actually know how to route 4 layer PCB Design
