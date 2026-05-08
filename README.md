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

---
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
- [mode_1.py](/Python/mode_1.md)
- [mode_2.py](/Python/mode_2.md)
- [mode_3.py](/Python/mode_3.md)
- [mode_4.py](/Python/mode_4.md)
- [GUI.py](/Python/gui.md)

---
## Running the System

The system requires **4 terminals** opened on the Raspberry Pi (or via SSH).  
Launch them **in order**:
- Terminal 1: ROS Core
- Terminal 2: Kinect Camera Driver
- Terminal 3: Main Surveillance System
- Terminal 4: GUI

---
## GUI

Once the GUI is open, you can:
- 📷 **Switch between cameras** using the camera toggles
- 📋 **View the security event log** in real time
- 🖼️ **Browse captured images and video clips**
- 🔔 **Monitor the alarm status**
- 🔄 **Reset the system** using the Reset button
Note: If a camera crashes, toggle it **off then back on** in the GUI to restart it — no need to restart the whole system.

---
## System Modes

|Mode|Condition|Action|
|---|---|---|
|**Mode 1** — Idle|No person detected|Cycles between camera feeds|
|**Mode 2** — Far|Person detected > 4.0 m|Captures low-resolution images|
|**Mode 3** — Medium|Person detected 2.0 – 4.0 m|Captures high-res images, logs event|
|**Mode 4** — Close|Person detected < 2.0 m|Records video, triggers alarm|
