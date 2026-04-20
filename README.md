# Moonraker on Creality Nebula Pad (Ender 3 V3 SE)

This guide explains how to install [Moonraker](https://github.com/Arksine/moonraker) on the Creality Nebula Pad so you can connect [OrcaSlicer](https://github.com/SoftFever/OrcaSlicer) (or any other Moonraker-compatible slicer/frontend) directly to your printer over Wi-Fi.

> [!CAUTION]
> 3D FDM printers involve high temperatures, molten plastic, and electricity. Modifications may break your
> printer or cause damage. Everything here is done at your own risk.

> [!NOTE]
> This guide does **not** modify any printer parameters (speeds, temperatures, offsets, etc.).
> Only the minimum required changes are made to `printer.cfg` to enable Moonraker connectivity.

---

## What You Get

- Send G-code files directly from OrcaSlicer to your printer over Wi-Fi
- Start prints remotely from OrcaSlicer
- Moonraker API available at `http://<nebula-pad-ip>:7125`
- No Fluidd, no Mainsail — just the API (you can add those later if you want)

---

## Prerequisites

**You need a rooted Nebula Pad with SSH enabled.**

Use the firmware patching tool in this repository to create a root-enabled firmware image:

```bash
uv run main.py
```

This creates a patched `.img` file in the `firmware/` directory. Flash it to your Nebula Pad via the Creality firmware update process (copy to USB, update from the printer menu). After flashing:

- SSH user: `root`
- SSH password: `creality` (or whatever you set with `--root-password`)

Find your Nebula Pad's IP address in your router's DHCP table or in the printer's Wi-Fi settings.

---

## System Information

The Nebula Pad runs:

| Item | Value |
|------|-------|
| OS | Buildroot 2020.02.1 |
| Architecture | MIPS |
| Python | 3.8.2 |
| Init system | BusyBox init (no systemd) |
| Filesystem | OverlayFS — read-only rootfs + 290MB writable overlay |
| Data partition | `/usr/data` — 5.9GB ext4, **this is where everything persistent goes** |

> [!IMPORTANT]
> Always install Python packages to `/usr/data/packages` using `pip install --target /usr/data/packages`.
> The overlay filesystem unreliably persists compiled packages (`.so` files). The `/usr/data` partition
> is a real ext4 volume and survives reboots without issues.

---

## Installation

All commands below are run from **your computer**, not on the printer. Replace `192.168.x.x` with your Nebula Pad's IP address.

### 1. Connect via SSH

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x
```

Or run commands directly:

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "<command>"
```

### 2. Download Moonraker

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "
  cd /usr/data && \
  wget -O moonraker.zip https://github.com/Arksine/moonraker/archive/refs/heads/master.zip && \
  unzip moonraker.zip && \
  mv moonraker-master moonraker && \
  rm moonraker.zip
"
```

### 3. Create Required Directories

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "
  mkdir -p /usr/data/packages \
            /usr/data/tmp \
            /usr/data/pip-cache \
            /usr/data/printer_data/config \
            /usr/data/printer_data/logs \
            /usr/data/printer_data/gcodes \
            /usr/data/printer_data/database
"
```

### 4. Update pip

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "
  pip install --upgrade pip 2>&1
"
```

### 5. Install Dependencies

The Nebula Pad's MIPS architecture cannot compile C extensions from source (no cross-compiler available on the device). The packages below are all that can be installed — two packages from Moonraker's full requirements (`pillow` and `streaming-form-data`) cannot be compiled on this device, but Moonraker works fine without them for basic OrcaSlicer connectivity.

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "
  TMPDIR=/usr/data/tmp pip install \
    --target /usr/data/packages \
    --cache-dir /usr/data/pip-cache \
    'tornado>=6.2.0,<=6.5.5' \
    'pyserial==3.4' \
    'distro==1.9.0' \
    'inotify-simple==2.0.1' \
    'libnacl==2.1.0' \
    'paho-mqtt==2.1.0' \
    'preprocess-cancellation==0.2.1' \
    'jinja2==3.1.6' \
    'apprise>=1.9.3,<=1.9.8' \
    'ldap3==2.9.1' \
    'python-periphery==2.4.1' \
    'importlib_metadata>=6.7.0,<=8.7.1' \
    'dbus-fast>=2.21.3,<=3.1.2' \
    'zeroconf>=0.131.0,<=0.148.0' \
    2>&1
"
```

> [!NOTE]
> `TMPDIR=/usr/data/tmp` is required. The default `/tmp` is a 98MB tmpfs and will fill up when
> downloading packages like `pillow` (46MB), causing the install to fail.

### 6. Create Moonraker Configuration

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "cat > /usr/data/printer_data/config/moonraker.conf << 'EOF'
[server]
host: 0.0.0.0
port: 7125
klippy_uds_address: /tmp/klippy_uds

[authorization]
trusted_clients:
    10.0.0.0/8
    127.0.0.0/8
    169.254.0.0/16
    172.16.0.0/12
    192.168.0.0/16
    FE80::/10
    ::1/128
cors_domains:
    *.lan
    *.local
    *://localhost
    *://localhost:*
    *://my.mainsail.xyz
    *://app.fluidd.xyz

[file_manager]
enable_object_processing: False

[octoprint_compat]

[history]

[update_manager]
channel: dev
refresh_interval: 168

[machine]
provider: none
EOF"
```

> [!NOTE]
> `[machine] provider: none` disables the systemd integration. The Nebula Pad uses BusyBox init,
> not systemd, so without this setting you'll see a harmless but noisy warning in the logs.

### 7. Add Required Sections to printer.cfg

Moonraker requires three sections in `printer.cfg` to function. **These sections only add connectivity features — they do not change any printer parameters.**

First, make a backup:

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x \
  "cp /usr/data/printer_data/config/printer.cfg /usr/data/printer_data/config/printer.cfg.backup"
```

Then add the sections. The safest way is to add them before the `#*# <---------------------- SAVE_CONFIG ---------------------->` block at the end of the file:

```ini
[virtual_sdcard]
path: /usr/data/printer_data/gcodes

[pause_resume]

[display_status]
```

You can do this with a single command:

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "
  sed -i '/^#\*# <-/i [virtual_sdcard]\npath: /usr/data/printer_data/gcodes\n\n[pause_resume]\n\n[display_status]\n' \
    /usr/data/printer_data/config/printer.cfg
"
```

Or connect via SSH and edit the file manually with `vi`.

### 8. Create the Startup Script

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "cat > /etc/init.d/S58moonraker << 'EOF'
#!/bin/sh

MOONRAKER_DIR=/usr/data/moonraker
DATA_PATH=/usr/data/printer_data
LOG_FILE=\${DATA_PATH}/logs/moonraker.log
PYTHON=/usr/bin/python3
PACKAGES=/usr/data/packages

start() {
    echo \"Starting Moonraker...\"
    mkdir -p \${DATA_PATH}/logs \${DATA_PATH}/gcodes \${DATA_PATH}/database
    PYTHONPATH=\${PACKAGES} \${PYTHON} \${MOONRAKER_DIR}/moonraker/moonraker.py \\
        -d \${DATA_PATH} \\
        >> \${LOG_FILE} 2>&1 &
    echo \$! > /tmp/moonraker.pid
    echo \"Moonraker started (PID \$(cat /tmp/moonraker.pid))\"
}

stop() {
    echo \"Stopping Moonraker...\"
    if [ -f /tmp/moonraker.pid ]; then
        kill \$(cat /tmp/moonraker.pid) 2>/dev/null
        rm /tmp/moonraker.pid
    fi
}

restart() {
    stop
    sleep 2
    start
}

case \"\$1\" in
    start) start ;;
    stop)  stop  ;;
    restart) restart ;;
    *) echo \"Usage: \$0 {start|stop|restart}\"; exit 1 ;;
esac
EOF
chmod +x /etc/init.d/S58moonraker"
```

### 9. Start Moonraker

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "/etc/init.d/S58moonraker start"
```

Wait a few seconds, then verify it's running:

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "ps aux | grep -v grep | grep moonraker"
```

And test the API from your computer:

```bash
curl http://192.168.x.x:7125/printer/info
```

You should get a JSON response like:

```json
{"result":{"state":"ready","state_message":"Printer is ready","hostname":"Nebula",...}}
```

---

## Connecting OrcaSlicer

1. Open OrcaSlicer
2. Go to **Device** → click the **"+"** button
3. Select **Klipper** as the printer type
4. Enter:
   - **Host**: your Nebula Pad's IP address (e.g. `192.168.1.100`)
   - **Port**: `7125`
5. Click **Connect**

You can now send G-code files directly from OrcaSlicer and start prints remotely.

---

## Filesystem Layout

| Path | Purpose |
|------|---------|
| `/usr/data/moonraker/` | Moonraker source code |
| `/usr/data/packages/` | Python dependencies (`pip --target`) |
| `/usr/data/printer_data/config/moonraker.conf` | Moonraker configuration |
| `/usr/data/printer_data/config/printer.cfg` | Klipper configuration |
| `/usr/data/printer_data/logs/moonraker.log` | Moonraker log |
| `/usr/data/printer_data/logs/klippy.log` | Klipper log |
| `/usr/data/printer_data/gcodes/` | Uploaded G-code files |
| `/etc/init.d/S58moonraker` | Auto-start script |

---

## Managing Moonraker

```bash
# Start
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "/etc/init.d/S58moonraker start"

# Stop
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "/etc/init.d/S58moonraker stop"

# Restart
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "/etc/init.d/S58moonraker restart"

# View live logs
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "tail -f /usr/data/printer_data/logs/moonraker.log"
```

---

## Known Limitations

| Limitation | Reason |
|-----------|--------|
| No thumbnail preview in Mainsail/Fluidd | `pillow` cannot be compiled on MIPS — no C compiler available on the device |
| File uploads may lack multipart streaming | `streaming-form-data` cannot be compiled on MIPS — same reason. OrcaSlicer uploads work fine. |
| No systemd integration | Nebula Pad uses BusyBox init, not systemd |
| No automatic updates via Moonraker update manager | Would require `git` and internet access from the device |

---

## Troubleshooting

**Moonraker starts but immediately exits:**

Check the log:
```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "tail -50 /usr/data/printer_data/logs/moonraker.log"
```

Common causes:
- `ModuleNotFoundError` — a dependency is missing, re-run step 5
- `klippy_uds_address` error — check that `/tmp/klippy_uds` exists (Klipper must be running)

**Port 7125 not reachable from the network:**

```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "netstat -tln | grep 7125"
```

If not listed, Moonraker isn't running. If listed but not reachable, check for a firewall:
```bash
sshpass -p creality ssh -o StrictHostKeyChecking=no root@192.168.x.x "nft list ruleset 2>/dev/null || iptables -L 2>/dev/null"
```

**`/tmp` fills up during pip install:**

Always use `TMPDIR=/usr/data/tmp` when running pip. The `/tmp` filesystem is only 98MB (tmpfs).

**Warning: "org.freedesktop.systemd1 was not provided by any .service files":**

This is harmless. It means Moonraker's `machine` component tried to connect to systemd via D-Bus and failed. Add `[machine]\nprovider: none` to `moonraker.conf` to silence it (see step 6).

---

## Acknowledgements

- [Moonraker](https://github.com/Arksine/moonraker) by Arksine
- Firmware rooting tool based on work by [Koen Erens](https://github.com/koen01/nebula_firmware) and [Jason Pell](https://github.com/pellcorp/creality)
