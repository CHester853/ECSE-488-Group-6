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