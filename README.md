# 🖥️ Home Assistant PC Power Control Dashboard  
### (Wake-on-LAN + REST API Shutdown + Online Status Indicator)

This setup lets you **wake**, **shut down**, **restart**, and **monitor** your computer (like an Ubuntu PC) directly from **Home Assistant**.  
It uses a **Wake-on-LAN shell command**, **REST endpoints** for power control, and a **ping sensor** that changes the button color when your PC is online.  

---

## 🧠 Features

✅ Wake your PC using `wakeonlan`  
✅ Shut down or restart via REST command  
✅ Ping-based online detection  
✅ Color-changing buttons (green = online / red = offline)  
✅ 100% local — works even on mesh networks like TP-Link Deco  

---

## ⚙️ Requirements

1. **Enable Wake-on-LAN** on your PC  
   - BIOS → enable **Resume by PCI-E Device / Wake on LAN**  
   - Disable **ErP Ready** (keeps NIC powered while off)
   - In Ubuntu:
     ```bash
     sudo apt install ethtool
     sudo ethtool -s <your_nic> wol g
     ```

2. **Install the `wakeonlan` tool** on your Home Assistant host  
   - **Home Assistant OS / SSH & Web Terminal add-on:**
     ```bash
     apk add wakeonlan
     ```
   - **Home Assistant Core / Container (Ubuntu):**
     ```bash
     sudo apt install wakeonlan
     ```

3. Both Home Assistant and your PC must be on the **same LAN segment**.

---

## 🧩 Step 1 – Add to `configuration.yaml`

```yaml
# 1️⃣ Wake-on-LAN shell command
shell_command:
  wake_pc: "wakeonlan 00:11:22:33:44:55"

# 2️⃣ REST commands (optional)
rest_command:
  shutdown_pc:
    url: "http://192.168.68.109:8125"
  restart_pc:
    url: "http://192.168.68.109:8126"

# 3️⃣ Ping sensor for online status
binary_sensor:
  - platform: ping
    name: PC Online
    host: 192.168.68.109
    count: 2
    scan_interval: 10
