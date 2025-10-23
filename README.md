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
    url: "http://<YOUR_PC_IP>:5000/power/shutdown"
    method: post
  restart_pc:
    url: "http://<YOUR_PC_IP>:5000/power/restart"
    method: post

# 3️⃣ Ping sensor for online status
binary_sensor:
  - platform: ping
    name: PC Online
    host: <YOUR_PC_IP>
    count: 2
    scan_interval: 10

📝 Replace
• 00:11:22:33:44:55 → your PC’s MAC address
• <YOUR_PC_IP> → your PC’s local IP address

Restart Home Assistant after saving.

---

## 🛠 Step 2 – Create REST listeners on your PC

The `rest_command` entries above expect a tiny web service running on your PC that
exposes `/power/shutdown` and `/power/restart` endpoints. The example below uses
Python and Flask, but any framework capable of accepting HTTP POST requests can
work.

1. **Install prerequisites** (Ubuntu example)

   ```bash
   sudo apt update
   sudo apt install python3 python3-venv
   ```

2. **Create a project folder and virtual environment**

   ```bash
   mkdir -p ~/pc-power-listener
   cd ~/pc-power-listener
   python3 -m venv .venv
   source .venv/bin/activate
   pip install flask gunicorn
   ```

3. **Create `app.py`**

   ```python
   from flask import Flask, jsonify
   import subprocess

   app = Flask(__name__)


   def run_command(command):
       subprocess.Popen(command, shell=False)


   @app.post("/power/shutdown")
   def shutdown_pc():
       run_command(["/usr/sbin/shutdown", "-h", "now"])
       return jsonify(status="shutting-down")


   @app.post("/power/restart")
   def restart_pc():
       run_command(["/usr/sbin/shutdown", "-r", "now"])
       return jsonify(status="restarting")
   ```

4. **Test the listener locally**

   With the virtual environment still active, start Flask in development mode:

   ```bash
   flask --app app run --host 0.0.0.0 --port 5000
   ```

   From another terminal (or your Home Assistant host), verify both endpoints
   respond with HTTP 200 before moving on:

   ```bash
   curl -X POST http://<YOUR_PC_IP>:5000/power/shutdown
   curl -X POST http://<YOUR_PC_IP>:5000/power/restart
   ```

   Press `Ctrl+C` to stop the development server.

5. **Create a persistent service**

   Running `flask run` manually is fine for testing, but a `systemd` service keeps
   the listener alive after reboots. Because the virtual environment already
   contains `gunicorn`, we can use it to serve the Flask app in production.

   Create `~/pc-power-listener/listener.service` with the following contents:

   ```ini
   [Unit]
   Description=PC power control REST listener
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=simple
   WorkingDirectory=/home/<YOUR_USER>/pc-power-listener
   Environment="PATH=/home/<YOUR_USER>/pc-power-listener/.venv/bin"
   ExecStart=/home/<YOUR_USER>/pc-power-listener/.venv/bin/gunicorn -b 0.0.0.0:5000 app:app
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```

   Copy it into place and enable the service:

   ```bash
   sudo cp listener.service /etc/systemd/system/pc-power-listener.service
   sudo systemctl daemon-reload
   sudo systemctl enable --now pc-power-listener.service
   ```

   Check the status with `systemctl status pc-power-listener.service`.

6. **Allow passwordless shutdown/restart (optional, recommended)**

   Add a sudoers rule so the account that runs Flask can execute the shutdown
   commands without a password:

   ```bash
   sudo visudo
   ```

   Then append a line similar to:

   ```
   yourusername ALL=(ALL) NOPASSWD: /usr/sbin/shutdown
   ```

   Log out and back in so the new sudo rule is applied.

Once the service is running, pressing the shutdown or restart buttons in Home
Assistant will trigger the corresponding endpoint and execute the command on your
PC.
