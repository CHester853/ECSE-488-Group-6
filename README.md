# ECSE-488 — Group 6
### Raspberry Pi Surveillance System
**Jacob Hyman & Phurinat (Chester) Pinyomit**

---

## Hardware

| Component | Details |
|---|---|
| **Microcontroller** | Raspberry Pi 4B — Ubuntu 20.04 |
| **Camera** | Microsoft Kinect V1 |
| **Speaker** | USB Speaker |

Connect all components to the Raspberry Pi before running the system.

---

## Remote Access

It is recommended to use a **Linux/Ubuntu machine** to SSH into the Raspberry Pi:

```
ssh -X NAME@<IP Address>
```

Note: **MacOS is not recommended** — it does not natively support X11 forwarding, which means the GUI will not display correctly over SSH from a Mac. Use the Raspberry Pi directly with a connected monitor, or SSH from a Linux machine.

## Required Files

Before running, ensure all files are in place:
```
~/chester/code/python/
├── main_system.py
├── mode_1.py
├── mode_2.py
├── mode_3.py
├── mode_4.py
└── gui.py
```

Or you can found in here:
- [main_system.py](/Python/main_system.md)
- 