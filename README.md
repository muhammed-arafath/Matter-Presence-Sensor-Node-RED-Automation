# 🏠 Matter Presence Sensor — Node-RED Automation

Real-time room presence detection and automatic light control using LeTianPai 24GHz mmWave radar sensors, python-matter-server, and Node-RED. No Home Assistant, no hub, no cloud, no subscription.

![License](https://img.shields.io/badge/license-MIT-blue)
![Docker](https://img.shields.io/badge/docker-required-blue)
![Matter](https://img.shields.io/badge/protocol-Matter-green)
![Node-RED](https://img.shields.io/badge/Node--RED-3.x-red)

---

## 📡 Architecture

```
LeTianPai Sensor (24GHz mmWave)
        ↓  Matter over WiFi
python-matter-server (Docker)
        ↓  WebSocket ws://127.0.0.1:5580/ws
Node-RED (Docker)
        ↓  MQTT
Light Relay (NSPanel / any MQTT switch)
```

- **No Home Assistant** required
- **No hub** required — sensor connects directly via WiFi Matter
- **No cloud** — everything runs locally on your server
- **Real-time** — sensor events arrive in under 1 second via `start_listening` + `attribute_updated`

---

## 🧰 Hardware

| Component | Model | Notes |
|---|---|---|
| Presence Sensor | LeTianPai Presence Sensor Box | 24GHz mmWave radar |
| Light Switch | NSPanel (or any MQTT relay) | Controls room lighting |
| Server | Any Linux PC | Runs Docker containers |

---

## 📦 Project Structure

```
matter-nodered-presence/
├── README.md
├── docker-compose.yaml          ← Docker services setup
├── flows/
│   ├── device1_flow.json        ← Node-RED flow for Device 1
│   ├── device2_flow.json        ← Node-RED flow for Device 2
│   └── letianpai_final.json     ← Single device flow (reference)
└── .gitignore
```

---

## ⚙️ Prerequisites

- Ubuntu Linux (or any Linux with Docker)
- Docker and Docker Compose
- Bluetooth adapter (for initial device commissioning only)
- LeTianPai app on phone (for one-time device reset only)
- MQTT broker (for light relay control)

---

## 🚀 Installation

### Step 1 — Install Docker

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

### Step 2 — Clone this repository

```bash
git clone https://github.com/YOUR_USERNAME/matter-nodered-presence.git
cd matter-nodered-presence
```

### Step 3 — Edit docker-compose.yaml

Replace `YOUR_USER` with your Linux username and `YOUR_TIMEZONE` with your timezone:

```bash
nano docker-compose.yaml
```

### Step 4 — Create data folders

```bash
mkdir -p matter-data
mkdir -p nodered-data
```

### Step 5 — Start services

```bash
docker compose up -d
docker ps
```

Both `matter-server` and `nodered` must show **Up**.

### Step 6 — Install wscat

```bash
sudo npm install -g wscat
```

---

## 📲 Device Commissioning (One Time Per Device)

### Connect to matter-server

```bash
wscat -c ws://127.0.0.1:5580/ws
```

### Set WiFi credentials (once only)

```json
{"message_id":"wifi","command":"set_wifi_credentials","args":{"ssid":"YOUR_WIFI","credentials":"YOUR_PASSWORD"}}
```

### Commission each device

**On phone (for each device):**
1. LeTianPai app → Device → Settings → **Remove Device** → Confirm
2. Blue LED starts flashing = pairing mode (3 minute window)
3. Check **11-digit code** on sticker on bottom of device

**In wscat immediately while LED is flashing:**

```json
{"message_id":"c1","command":"commission_with_code","args":{"code":"YOUR_11_DIGIT_CODE"}}
```

Success response:
```json
{"message_id":"c1","result":{"node_id":1,"available":true,...}}
```

> Note the `node_id` — Device 1 = 1, Device 2 = 2, etc.

**Phone is no longer needed after all devices are commissioned.**

### Verify all devices

```bash
docker logs matter-server | grep "Subscription succeeded"
```

---

## 🔧 Node-RED Setup

### Import flows

1. Open Node-RED: `http://YOUR_SERVER_IP:1880`
2. Menu ☰ → Import → paste content of `flows/device1_flow.json` → Import
3. Repeat for `device2_flow.json`
4. Click **Deploy**

### Configure MQTT broker

Double-click any relay node → pencil icon next to broker → update:
- **Broker**: your MQTT broker IP
- **Port**: 1883

### Configure relay topics

Update the topic in each MQTT out node to match your light switch:

| Device | Default Topic | Change To |
|---|---|---|
| Device 1 Relay 1 | `room1/relay1/set` | your actual topic |
| Device 1 Relay 2 | `room1/relay2/set` | your actual topic |
| Device 2 Relay 1 | `room2/relay1/set` | your actual topic |
| Device 2 Relay 2 | `room2/relay2/set` | your actual topic |

### Rename rooms

In each flow's `Parse Messages` function node, update the room names:

```javascript
// Device 1 flow — change "Room1" to your actual room name
const NODE_ID = 1;

// Device 2 flow — change "Room2" to your actual room name
const NODE_ID = 2;
```

---

## 📊 How Real-Time Works

The flow uses `start_listening` — required by python-matter-server before it pushes any events:

```
1. WebSocket connects → matter-server sends server_info
2. Parse Messages detects server_info → sends start_listening
3. matter-server responds with current snapshot of all sensor values
4. All badges populate immediately
5. matter-server starts pushing live attribute_updated events
6. Every sensor change → Node-RED updates in < 1 second
```

> Without `start_listening`, matter-server only responds to commands — no live events are pushed.

---

## 🔬 Sensor Data Reference

| Sensor | Cluster ID | Path | Raw → Value |
|---|---|---|---|
| Occupancy | 1030 | `1/1030/0` | `1` → Detected, `0` → Clear |
| Illuminance | 1024 | `2/1024/0` | `10^((v-1)/10000)` lux |
| Temperature | 1026 | `3/1026/0` | `value ÷ 100` = °C |
| Humidity | 1029 | `4/1029/0` | `value ÷ 100` = RH% |

### attribute_updated event format

```json
{
  "event": "attribute_updated",
  "data": [7, "1/1030/0", 1]
}
```

> `data` is an array: `[node_id, attribute_path, value]`

---

## 🔍 Useful Commands

```bash
# Check all commissioned devices
wscat -c ws://127.0.0.1:5580/ws
# then: {"message_id":"list","command":"get_nodes","args":{}}

# Watch live events
docker logs matter-server -f | grep "Node:"

# Check device is online
docker logs matter-server | grep "Subscription succeeded"

# Restart services
cd ~/docker/nodered
docker compose restart
```

---

## 🛠️ Troubleshooting

| Problem | Solution |
|---|---|
| `Parse Messages` stuck on "Connecting…" | Restart matter-server: `docker compose restart matter-server` |
| Badges show "Waiting…" after deploy | Check `Parse Messages` shows "Live" status |
| Commission failed — BLE timeout | Factory reset device (hold button 5-10s), retry within 3 min |
| `device available: false` | Power cycle the sensor — reconnects automatically |
| No live updates | `start_listening` must be sent — check WebSocket is connected |
| MQTT relay not firing | Verify broker IP and topic. Test: `mosquitto_sub -h YOUR_IP -t "#" -v` |

---

## 📝 Adding More Devices

1. Commission the new device (gets next node_id automatically)
2. Copy `device1_flow.json` → rename to `device3_flow.json`
3. Change all IDs in the file (search and replace `device1` → `device3`)
4. Change `NODE_ID = 1` to the new node_id
5. Import into Node-RED and deploy

---

## 🤝 Contributing

Pull requests welcome. For major changes, please open an issue first.

---

## 📄 License

MIT License — free to use, modify and distribute.

---

## 🙏 Credits

- [python-matter-server](https://github.com/home-assistant-libs/python-matter-server) — Matter controller
- [Node-RED](https://nodered.org) — Flow-based automation
- [LeTianPai](https://www.letianpai.com) — mmWave presence sensor
