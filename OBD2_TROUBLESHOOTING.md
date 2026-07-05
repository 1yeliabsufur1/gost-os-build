# OBD2 Troubleshooting Guide for GOST OS

This guide helps you fix OBD2 connectivity issues between your vehicle and the GOST OS backend.

---

## Quick Status Check

The **frontend shows OBD status in 3 places**:

1. **Status Bar (top)**: Shows `● LIVE · OBDLINK` (green, real truck) or `◌ SHOWCASE (simulated)`
2. **DRIVE Tab**: Banner shows actual OBD status + port info
3. **Settings Tab**: `AUTO` (tries real first) vs `SHOWCASE` (always simulated)

If you see `*** NOT CONNECTED ***`, the backend telemetry has no data. Check the backend logs.

---

## Level 1: Check Hardware & Drivers

### 1.1 OBD2 Adapter Not Recognized

**Symptoms:**
- No `/dev/ttyUSB*` or `/dev/ttyACM*` device
- USB adapter plugged in but not showing in `ls /dev/tty*`

**Fix:**

```bash
# Check if adapter appears
lsusb
# Look for "FTDI", "CH340", or "Prolific" devices

# Install drivers (Linux)
sudo apt-get install -y linux-headers-$(uname -r) build-essential
sudo apt-get install -y ftdi-eeprom libftdi1 libftdi-dev  # for FTDI chips
sudo apt-get install -y ch341-dkms                         # for CH340 chips

# Reboot for driver to load
sudo reboot

# Check again
ls -la /dev/ttyUSB* /dev/ttyACM*
```

**Expected output:**
```
crw-rw---- 1 root dialout 188, 0 Jul  5 10:23 /dev/ttyUSB0
```

If still not appearing:
- Try a **different USB cable** (some only charge, not data)
- Try a **different USB port**
- Check if adapter is powered (usually has an LED)

---

### 1.2 Add User to dialout Group

**Symptom:** Permission denied when accessing `/dev/ttyUSB0`

**Fix:**
```bash
# Check if your user is in the dialout group
id $USER

# Add to dialout group
sudo usermod -a -G dialout $USER

# Apply (no reboot needed, but logout/login required)
newgrp dialout

# Verify
ls -la /dev/ttyUSB0
# Should show `rw` for group
```

---

## Level 2: Test OBD Adapter Connection

### 2.1 Manual Adapter Test

```bash
python3 << 'EOF'
import glob
import time

print("=== OBD2 Adapter Scan ===")
ports = sorted(glob.glob('/dev/ttyUSB*') + glob.glob('/dev/ttyACM*'))
print(f"Available ports: {ports}\n")

if not ports:
    print("ERROR: No serial ports found!")
    exit(1)

try:
    import obd
except ImportError:
    print("ERROR: obd library not installed")
    print("Install with: pip3 install obd-python")
    exit(1)

print(f"Testing {len(ports)} port(s)...\n")

for port in ports:
    for baud in [115200, 9600, 230400, 38400]:
        try:
            print(f"  {port} @ {baud}...", end=" ", flush=True)
            conn = obd.OBD(portstr=port, baudrate=baud, fast=False, timeout=2)
            
            if conn.is_connected():
                print("✓ CONNECTED!")
                print(f"    Protocol: {conn.protocol_name()}")
                print(f"    Supported PIDs: {len(conn.supported_commands)}")
                
                # Try a test query
                try:
                    r = conn.query(obd.commands.RPM)
                    print(f"    RPM test: {r}")
                except:
                    print(f"    RPM test: (query error)")
                
                conn.close()
                exit(0)
            else:
                print("✗ not connected")
                conn.close()
        except Exception as e:
            print(f"✗ error: {str(e)[:30]}")

print("\n❌ NO ADAPTER FOUND on any port/baud!")
EOF
```

**Expected success output:**
```
  /dev/ttyUSB0 @ 115200...✓ CONNECTED!
    Protocol: ISO 15765-4 (CAN)
    Supported PIDs: 42
    RPM test: 650.0 revolutions_per_minute
```

---

### 2.2 Wrong Baud Rate?

If the test above finds a connection but with the wrong baud, edit `hud_server.py`:

**Line ~782 in hud_server.py:**
```python
# Current (tries these in order):
OBD_BAUDS = [115200, 230400, 38400, 9600]

# If your adapter uses 9600, put it first:
OBD_BAUDS = [9600, 115200, 230400, 38400]

# Or if you know it's exactly one, replace with:
OBD_BAUDS = [115200]  # e.g. OBDLink EX
```

Then restart hud_server.

---

## Level 3: Start Backend with Verbose Logging

### 3.1 Run in Foreground (NOT as systemd service)

```bash
cd /path/to/gost-os-build

# Kill any running instance
killall python3 hud_server.py 2>/dev/null

# Run with full logging
python3 hud_server.py 2>&1 | tee hud_debug.log

# Watch output for:
# ✓ "OBD: CONNECTED -- LIVE /dev/ttyUSB0 @ 115200 [ISO_15765_4, 42 PIDs]"
# ✗ "OBD: no adapter found"
# ✗ "OBD: no ECU link after 3 tries"
```

---

### 3.2 Add Extra Debug Output

**Edit hud_server.py around line 787:**

```python
async def connect_obd(self):
    """Try hard to link a real OBDLink/ELM327 adapter..."""
    try:
        import obd
    except ImportError:
        self.obd_status = "python-obd not installed"
        log("DEBUG: OBD import failed")
        return False
    
    import glob
    ports = sorted(glob.glob("/dev/ttyUSB*") + glob.glob("/dev/ttyACM*"))
    log(f"DEBUG: Detected ports: {ports}")  # ADD THIS LINE
    
    if not ports:
        self.obd_linked = False
        self.obd_status = "no adapter found (no /dev/ttyUSB* or /dev/ttyACM*)"
        log("OBD:", self.obd_status)
        log("DEBUG: Port scan failed")  # ADD THIS LINE
        return False
    
    # ... rest continues ...
```

Save and restart. Now you'll see exact port detection issues.

---

## Level 4: Check Frontend Connection

Once backend is running, test the WebSocket:

```bash
python3 << 'EOF'
import websockets
import json
import asyncio

async def test():
    print("Connecting to ws://127.0.0.1:8765 ...")
    try:
        async with websockets.connect("ws://127.0.0.1:8765", timeout=5) as ws:
            print("✓ WebSocket connected\n")
            
            # Get 5 messages
            for i in range(5):
                msg = json.loads(await ws.recv())
                print(f"Message {i}: {msg['type']}")
                
                if msg['type'] == 'telemetry':
                    print(f"  Connected: {msg['connected']}")
                    print(f"  Live OBD: {msg['live']}")
                    print(f"  OBD Status: {msg['obd_status']}")
                    print(f"  Vehicle Type: {msg['vtype']}")
                    if msg['connected'] and msg['values']:
                        print(f"  Speed: {msg['values'].get('SPEED', 'N/A')}")
                        print(f"  RPM: {msg['values'].get('RPM', 'N/A')}")
                    break
    except Exception as e:
        print(f"✗ Connection failed: {e}")
        print("Make sure hud_server.py is running!")

asyncio.run(test())
EOF
```

**Expected success:**
```
Connecting to ws://127.0.0.1:8765 ...
✓ WebSocket connected

Message 0: hello
Message 1: library
Message 2: config
Message 3: telemetry
  Connected: True
  Live OBD: True
  OBD Status: LIVE /dev/ttyUSB0 @ 115200 [ISO_15765_4, 42 PIDs]
  Vehicle Type: hybrid
  Speed: 55.5
  RPM: 1250.0
```

---

## Level 5: Vehicle Link Issues

### 5.1 "CONNECTED but not LIVE" (Adapter found but no car data)

**Symptoms:**
- Backend says `LIVE /dev/ttyUSB0 @ 115200 [ISO_15765_4, 42 PIDs]`
- But frontend shows no speed/RPM/fuel data

**Causes:**
1. OBD2 connector not fully plugged into vehicle
2. Vehicle is off (won't respond)
3. ECU protocol mismatch (adapter supports ISO 15765-4 but your car uses J1939)
4. After market security modules blocking OBD

**Fixes:**
```bash
# 1. Check physical connection
# - OBD2 female connector under steering wheel (varies by vehicle)
# - Should click/lock when fully seated
# - Wiggle test: shake connector gently; should be solid

# 2. Turn vehicle ON (key position 2, or engine running)
# - OBD2 requires vehicle power
# - Don't need engine running for most PIDs

# 3. Verify ECU responsiveness
python3 << 'EOF'
import obd
import time

conn = obd.OBD(portstr="/dev/ttyUSB0", baudrate=115200, timeout=2)
print(f"Connected: {conn.is_connected()}")
print(f"Protocol: {conn.protocol_name()}")

# Try multiple PIDs
for pid_name in ["SPEED", "RPM", "FUEL_LEVEL"]:
    try:
        cmd = getattr(obd.commands, pid_name, None)
        if cmd:
            r = conn.query(cmd)
            print(f"{pid_name}: {r}")
        else:
            print(f"{pid_name}: NOT SUPPORTED")
    except Exception as e:
        print(f"{pid_name}: ERROR - {e}")
        
conn.close()
EOF
```

If all PIDs return errors → vehicle protocol not matching adapter.

---

### 5.2 Hybrid Vehicle Special Case

**F-150 PowerBoost / other hybrids:**

The frontend has special detection logic (line ~917 in index.html):

```javascript
// Runtime upgrade to hybrid: if we EVER see the truck moving with the
// engine off, it's a hybrid/EV regardless of which PIDs answered.
```

**Issue:** Ford often doesn't expose the hybrid battery SoC PID that the obd-python library expects.

**Solution:** The backend **auto-detects** hybrids by checking:
1. If RPM < 250 AND speed > 1 mph → it's a hybrid/EV
2. Falls back to `HYBRID_BATTERY_REMAINING` if available
3. Falls back to `FUEL_LEVEL` if not hybrid

This is automatic. Just make sure the adapter can query `SPEED` and `RPM`.

---

## Level 6: Source Mode Configuration

### 6.1 Set AUTO (Real OBD or Fallback)

**File:** `/path/to/state/config.json`

```json
{
  "setup_done": true,
  "source_mode": "AUTO",
  "theme": "gost"
}
```

Behavior:
- **Real OBD adapter connected & working** → LIVE data, green badge
- **No adapter or fails to link** → SHOWCASE demo (simulated), grey badge, keeps retrying every 6s
- **Operator disables in Settings** → switches to SHOWCASE on demand

### 6.2 Force DEMO/SHOWCASE Mode (for testing UI)

```json
{
  "source_mode": "DEMO"
}
```

- Always simulated data
- Never tries real OBD adapter
- Good for debugging frontend without hardware

---

## Full Diagnostic Command

Run this **once the backend is running**:

```bash
#!/bin/bash
set -e

echo "=== GOST OBD2 FULL DIAGNOSTIC ==="
echo

# 1. Hardware
echo "1. USB Adapter Detected:"
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null | head -1 || echo "   NONE FOUND"
echo

# 2. Python OBD
echo "2. Python OBD Library:"
python3 -c "import obd; print('   ✓ Installed')" 2>/dev/null || echo "   ✗ NOT INSTALLED"
echo

# 3. Backend WS
echo "3. Backend WebSocket (ws://127.0.0.1:8765):"
python3 << 'EOF'
import websockets
import json
import asyncio

async def test():
    try:
        async with websockets.connect("ws://127.0.0.1:8765", timeout=2) as ws:
            msg = json.loads(await ws.recv())  # hello
            msg = json.loads(await ws.recv())  # library
            msg = json.loads(await ws.recv())  # config
            msg = json.loads(await ws.recv())  # telemetry
            print(f"   ✓ Connected")
            print(f"   OBD Status: {msg.get('obd_status', 'N/A')}")
            print(f"   Live: {msg.get('live', False)}")
            print(f"   Connected: {msg.get('connected', False)}")
    except Exception as e:
        print(f"   ✗ Failed: {e}")

asyncio.run(test())
EOF
echo
echo "=== END DIAGNOSTIC ==="
```

---

## Common Issues & Quick Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No adapter found (no /dev/ttyUSB*)` | USB driver missing or device not connected | Install driver, check USB cable |
| `CONNECTED but data is NULL` | ECU protocol mismatch or car off | Turn on vehicle, check adapter compatibility |
| `Connection times out after 3 tries` | Wrong baud rate OR adapter hanging | Edit `OBD_BAUDS`, try different USB port |
| `Frontend shows SHOWCASE/DEMO` | Backend can't connect or is failing | Check backend logs with `hud_debug.log` |
| `RPM/Speed work but Fuel/Battery N/A` | Vehicle doesn't expose that PID | Normal — fallback to available data |
| `Adapter disappeared after reboot` | Kernel module didn't load | `sudo modprobe ch341` or reboot |

---

## Reset to Defaults

If you've made edits and want to start fresh:

```bash
# 1. Reset config
rm /path/to/state/config.json
# (backend will recreate with defaults)

# 2. Restore hud_server.py defaults
cd /path/to/gost-os-build
git checkout hud_server.py  # if in git

# 3. Reinstall obd-python
pip3 install --upgrade --force-reinstall obd-python

# 4. Reboot
sudo reboot
```

---

## Next Steps

If still stuck:

1. **Run the full diagnostic** (command above)
2. **Capture backend logs** with timestamps:
   ```bash
   python3 hud_server.py > /tmp/gost-debug.log 2>&1 &
   # Let it run for 30s
   cat /tmp/gost-debug.log
   ```
3. **Check vehicle compatibility** at [python-obd supported vehicles](https://github.com/brendanwhitfield/python-obd/wiki/Supported-OBD-Commands)

Good luck! 🚗📡
