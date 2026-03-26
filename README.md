# 📡 RuView ESP32-S3 Setup Guide (Simple Version)

## 🎯 Goal

Set up an ESP32-S3 to:

- Capture WiFi CSI data  
- Send it via UDP  
- Process it with RuView  
- Visualize human presence + pose in real time  

---

# 🧰 Requirements

## Hardware
- ESP32-S3 dev board (with PSRAM recommended)
- USB cable
- WiFi network

## Software
- Python 3
- esptool
- Rust + Cargo
- Git

---

# 🚀 Step 1 — Install tools
```bash
pip3 install esptool
pip3 install esp-idf-nvs-partition-gen
```
---

# 🔌 Step 2 — Connect ESP32

Find port:
```bash
ls /dev/cu.*
```
Example:
/dev/cu.usbserial-110

---

# 🧹 Step 3 — Erase flash
```bash
esptool --chip esp32s3 --port /dev/cu.usbserial-110 erase-flash
```
---

# 📦 Step 4 — Flash RuView firmware

Files needed:
```bash
bootloader.bin
partition-table.bin
ota_data_initial.bin
esp32-csi-node.bin
```
Flash:
```bash
esptool --chip esp32s3 \
        --port /dev/cu.usbserial-110 \
        --baud 460800 write-flash \
        -z 0x0 bootloader.bin 0x8000 \
           partition-table.bin 0xe000 \
           ota_data_initial.bin 0x10000 \
           esp32-csi-node.bin
```

---

# ⚙️ Step 5 — Provision WiFi + Server
```bash
python3 provision.py  \
--port /dev/cu.usbserial-110   \
--ssid "YOUR_WIFI"  \
--password "YOUR_PASSWORD"   \
--target-ip YOUR_COMPUTER_IP
```
---

# 🔍 Step 6 — Verify ESP32
```bash
screen /dev/cu.usbserial-110 115200
```
Expected:
Connected to WiFi
CSI streaming active

---

# 🖥️ Step 7 — Run RuView server
```bash
cargo run --release \
          --bin sensing-server --   \
          --source esp32   \
          --http-port 3000   \
          --ws-port 3001   \
          --tick-ms 500
```
---

# 🌐 Step 8 — Open UI

http://localhost:3000/ui/index.html

---

# ✅ Expected Result

- Human silhouette appears
- Presence = PRESENT
- Persons detected
- Confidence ~60%+

---

# 🧠 Architecture

ESP32 → WiFi CSI → UDP → Rust Server → AI → Web UI

---

# 🚀 Next Steps

- Dockerize
- Kubernetes deploy
- Multi-node ESP setup
- Observability pipeline


---

# 🐳 Optional — Run with Docker

You can run the RuView server using Docker:

```bash
docker run -p 3000:3000 \
           -p 3001:3001 \
           -p 5005:5005/udp \
           -e CSI_SOURCE=esp32 \
           ruvnet/wifi-densepose:latest
```

Then open:

http://localhost:3000/ui/index.html
