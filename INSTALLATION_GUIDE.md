# Smart Parking System — Installation Guide for a New PC

This document explains exactly how to install and run the entire Smart Parking System on a brand new Server PC and connect the Raspberry Pi 5 client to it.

---

## 🖥️ MACHINE 1: The Server PC (Windows Laptop/Desktop)

### Software to Install BEFORE Copying the Project

| # | Software | Download Link | Why It's Needed |
|---|----------|--------------|-----------------|
| 1 | **Docker Desktop** | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) | Runs the 3 blockchain containers (peer, orderer, cli) |
| 2 | **Node.js v18+** | [nodejs.org](https://nodejs.org/) (LTS version) | Runs the Express backend server |

> ⚠️ **Docker Desktop**: After installing, open it once, go to **Settings → General** and make sure **"Use the WSL 2 based engine"** is checked. Then restart Docker. Wait until the Docker icon in the system tray shows "Docker Desktop is running" before proceeding.

> ⚠️ **Node.js**: During installation, make sure you check the box that says **"Add to PATH"**. To verify, open a new terminal and type `node --version` — you should see `v18.x.x` or higher.

---

### What to Copy to the New PC

Copy the **entire `smart-parking` folder** to the new PC. You can **skip** these folders to save space (they'll be regenerated):

| Folder | Can Skip? | Why |
|--------|-----------|-----|
| `server-node/node_modules/` | ✅ Yes | Will be recreated by `npm install` |
| `venv/` (root level) | ✅ Yes | Old Python venv, not used on server |
| `server/venv/` | ✅ Yes | Old Python venv, not used on server |
| `fabric-network/crypto-config/` | ❌ **DO NOT skip** | Contains TLS certificates needed by the server |
| `fabric-network/channel-artifacts/` | ❌ **DO NOT skip** | Contains the genesis block |
| `bin/` | ❌ **DO NOT skip** | Contains `cryptogen.exe` and `configtxgen.exe` |

**Simplest approach:** Just copy the entire folder as-is. It works.

---

### Step-by-Step Setup on the New PC

#### Step 1: Install Node.js dependencies
```powershell
cd smart-parking\server-node
npm install
```
This reads `package.json` and installs these 4 libraries into `node_modules/`:
- `express` — web server framework
- `cors` — allows cross-origin requests from the Pi
- `@hyperledger/fabric-gateway` — connects to the blockchain
- `@grpc/grpc-js` — gRPC transport for blockchain communication

#### Step 2: Start the Blockchain Network
```powershell
cd smart-parking\fabric-network
.\network.ps1 -Action up
```
> If you get an "execution policy" error, run:
> `powershell -ExecutionPolicy Bypass -File .\network.ps1 up`

This does 3 things:
1. Generates crypto certificates (TLS keys, MSP identity)
2. Starts 3 Docker containers (`peer0`, `orderer`, `cli`)
3. Creates the blockchain channel `mychannel`

Wait ~30 seconds until you see `"Network up and channel created!"`

#### Step 3: Deploy the Smart Contract (first time only)
```powershell
cd smart-parking\fabric-network
powershell -ExecutionPolicy Bypass -File .\deployCC.ps1
```
This packages `chaincode/node/lib/parkingContract.js`, installs it on the peer, and commits it to the channel. Wait until you see `"Done!"`.

#### Step 4: Start the Server
```powershell
cd smart-parking\server-node
node index.js
```
Or just double-click `scripts\start_server.bat`.

You should see:
```
Connecting to Fabric Gateway...
Connected to Fabric Network successfully.
Starting Smart Parking API on 0.0.0.0:8000
```
The dashboard auto-opens at `http://localhost:8000/dashboard`.

#### Step 5: Find your Server IP (for the Pi)
```powershell
ipconfig
```
Look for **IPv4 Address** under your Wi-Fi adapter (e.g., `192.168.1.50`). The Pi will need this.

---

## 🍓 MACHINE 2: Raspberry Pi 5

### Software to Install on the Pi

The Pi should already have **Raspberry Pi OS** (64-bit recommended). Then:

```bash
# 1. Enable I2C (for NFC reader and Servo driver)
sudo raspi-config
# → Interface Options → I2C → Enable → Reboot

# 2. Install system-level packages
sudo apt update
sudo apt install -y python3 python3-venv python3-pip libopencv-dev
```

### What to Copy to the Pi

Only copy the `raspberry_pi/` folder and `scripts/start_raspberry_pi.sh` to the Pi:

```bash
# From your laptop, use SCP:
scp -r smart-parking/raspberry_pi/ pi@<PI_IP>:~/smart-parking/raspberry_pi/
scp smart-parking/scripts/start_raspberry_pi.sh pi@<PI_IP>:~/smart-parking/scripts/
```

Or copy the entire project folder via USB drive — the Pi will only use the `raspberry_pi/` directory.

### Step-by-Step Setup on the Pi

#### Step 1: Update the Server IP

Edit the startup script to point to the new server PC's IP:
```bash
nano ~/smart-parking/scripts/start_raspberry_pi.sh
```
Change line 30:
```bash
export SERVER_URL="http://<NEW_SERVER_IP>:8000"
```
For example: `http://192.168.1.50:8000`

#### Step 2: Run the startup script
```bash
cd ~/smart-parking/scripts
chmod +x start_raspberry_pi.sh
./start_raspberry_pi.sh
```

This script automatically:
1. Creates a Python virtual environment (`venv`) if one doesn't exist
2. Activates the venv
3. Installs all Python libraries from `requirements.txt`:

| Library | Purpose |
|---------|---------|
| `requests` | HTTP calls to the server API |
| `board` + `adafruit-blinka` | Raspberry Pi GPIO/I2C access |
| `adafruit-circuitpython-pn532` | PN532 NFC reader driver |
| `adafruit-circuitpython-pca9685` | PCA9685 servo motor driver |
| `opencv-python-headless` | Camera image capture |
| `easyocr` | License plate text recognition |
| `pyserial` | Serial port communication |

4. Starts `main.py` — the main parking loop

> ⚠️ **First-time install of `easyocr`** takes 5-10 minutes because it downloads ~200MB of OCR model files.

---

## 🔌 Network Requirement

Both the **Server PC** and the **Raspberry Pi** must be on the **same Wi-Fi network** (same LAN). The Pi sends HTTP requests to the server's IP address on port `8000`.

Quick test from the Pi to check connectivity:
```bash
curl http://<SERVER_IP>:8000/health
```
If you see `{"status":"online","app":"Smart Parking API","version":"1.0.0"}`, the connection works.

If it doesn't work, check:
1. **Windows Firewall** — allow inbound connections on port 8000
2. Both devices are on the **same Wi-Fi**
3. The server is actually running (`node index.js`)

---

## 📋 Quick Checklist Summary

| Machine | What to Install | What to Copy | What to Run |
|---------|----------------|-------------|-------------|
| **Server PC** | Docker Desktop, Node.js v18+ | Entire `smart-parking/` folder | `npm install` → `network.ps1 up` → `deployCC.ps1` → `node index.js` |
| **Raspberry Pi** | Python3, I2C enabled | `raspberry_pi/` + `scripts/` | Edit `SERVER_URL` → `./start_raspberry_pi.sh` |

---

## 🔄 Teardown & Reset

To completely destroy the blockchain network and wipe all ledger data (start fresh):
```powershell
cd fabric-network
.\network.ps1 -Action down
```

To bring it back up again, repeat Steps 2-4 from the Server Setup.
