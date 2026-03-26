git clone --branch v0.5.0-esp32 https://github.com/ruvnet/RuView.git
cd RuView/firmware/esp32-csi-node
## Downoad binaries
```bash
curl -LOs https://github.com/ruvnet/RuView/releases/download/v0.5.0-esp32/bootloader.bin
curl -LOs https://github.com/ruvnet/RuView/releases/download/v0.5.0-esp32/esp32-csi-node.bin
curl -LOs https://github.com/ruvnet/RuView/releases/download/v0.5.0-esp32/ota_data_initial.bin
curl -LOs https://github.com/ruvnet/RuView/releases/download/v0.5.0-esp32/partition-table.bin
```
## Find MAC IP
```bash
export LAN_IP=$(ipconfig getifaddr en0)
```

## Find device
```bash
export DEVICE=/dev/cu.usbserial-11120
```

## Generate mesh key
```bash
export RUVIEW_MESH_KEY=$(openssl rand -hex 32)
```

```bash
export WIFI_SSID=
export WIFI_PASSWORD=
```

```bash
esptool --chip esp32s3 --port $DEVICE erase-flash
```
```bash
cat 
```
## Flash firmware
```bash
esptool --chip esp32s3 \
  --port $DEVICE \
  --baud 460800 write-flash \
  -z 0x0 bootloader.bin \
     0x8000 partition-table.bin \
     0xF000 ota_data_initial.bin \
     0x20000 esp32-csi-node.bin
```
## Provision Node 1
```bash
python provision.py \
  --port "$DEVICE" \
  --ssid "$WIFI_SSID" \
  --password "$WIFI_PASSWORD" \
  --target-ip "$LAN_IP" \
  --target-port 5005 \
  --node-id 1 \
  --tdm-slot 0 \
  --tdm-total 3 \
  --edge-tier 0
```

## Provision Node 2

```bash
python provision.py \
  --port "$DEVICE" \
  --ssid "$WIFI_SSID" \
  --password "$WIFI_PASSWORD" \
  --target-ip "$LAN_IP" \
  --target-port 5005 \
  --node-id 2 \
  --tdm-slot 1 \
  --tdm-total 3 \
  --edge-tier 0
```
## Provision Node 3

```bash
python provision.py \
  --port "$DEVICE" \
  --ssid "$WIFI_SSID" \
  --password "$WIFI_PASSWORD" \
  --target-ip "$LAN_IP" \
  --target-port 5005 \
  --node-id 3 \
  --tdm-slot 2 \
  --tdm-total 3 \
  --edge-tier 0
```

screen $DEVICE 115200 