---

---
### Change fixed Wi-Fi
- Navigate to `sudo nano /etc/netplan/50-cloud-init.yaml`
- Change information to
```
network:
    version: 2
    renderer: NetworkManager
```
- Then reboot your system
	- `sudo reboot`

### IP Address
- Raspberry Pi #6 (Might change)
	- `172.20.18.178`
- Raspberry Pi #7 (Might change)
	- `172.20.54.44`

### Shared Workspace (Desktop Computer)
Password: 7TurtleBots!

