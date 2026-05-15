# Overmesh-RC — Remote Collector Firmware

A fork of [MeshCore](https://github.com/meshcore-dev/MeshCore) RPTR firmware that turns a remote repeater node into a passive RF intelligence collector for [OverMesh](https://github.com/Slofi/overmesh).

---

## What it does

A deployed Overmesh-RC node sits at a remote vantage point (hilltop, rooftop, solar-powered) and passively logs every MC mesh node it hears. When triggered by OverMesh over the mesh, it delivers the collected observations back as encrypted DMs — which OM stores, maps, and visualizes automatically.

**Two data streams:**

| Type | Source | Data |
|------|--------|------|
| `OBS\|ADV` | Every received advertisement | Full sender pubkey + RSSI + SNR + timestamp |
| `OBS\|RX` | All other received packets | 4-byte packet hash + RSSI + SNR + timestamp |

ADV observations carry full node identity — useful for passive intel (who is on the mesh, how strong). RX observations capture anonymous activity — useful for coverage mapping (how far the mesh reaches from this vantage point).

---

## Required hardware

### RPTR board (runs this firmware)
- **Heltec T114 v1 or v2** — nRF52840 + HT-RA62 LoRa module
- Any other nRF52840 + HT-RA62 MeshCore-compatible board (e.g. Faketec)
- Build target: `Heltec_t114_without_display_repeater`
- The onboard display is not used and can be physically removed (ZIF connector)

### Collector board (buffers and delivers observations)
- **Waveshare RP2040-PiZero**
- Connected to the RPTR via hardware UART (see wiring below)
- Powered from the RPTR 3.3V rail — no separate power supply needed
- Runs MicroPython with the collector script (`main.py`)

### Power (off-grid deployment)
- LiPo battery → T114 VBAT connector
- Small solar panel → T114 Solar connector (dedicated input, onboard charging)
- T114 3.3V rail → RP2040-PiZero pin 1 (3V3)

---

## Wiring (T114 to RP2040-PiZero)

| T114 pad | Direction | RP2040-PiZero |
|----------|-----------|----------------|
| GPIO 10 (TX) | → | GP1 (UART0 RX) |
| GPIO 9 (RX) | ← | GP0 (UART0 TX) |
| GND | — | GND |
| 3.3V pad | → | Pin 1 (3V3) |

Both boards are 3.3V logic — direct connection, no level shifter needed.

---

## What this firmware adds (patches over stock MeshCore RPTR)

- **`onAdvertRecv`** — outputs `OBS|ADV|<pubkey>|<rssi>|<snr>|<ts>` to Serial and Serial1
- **`logRx`** — outputs `OBS|RX|<hash4>|<rssi>|<snr>|<ts>` to Serial and Serial1
- **`onPeerDataRecv`** — intercepts `OMCOLLECT` DM, stores requester identity/path, signals RP2040 via serial
- **`handleCommand`** — handles `RELAY|<line>` from RP2040: re-encrypts each as a TXT_MSG DM back to the OM requester
- **`main.cpp` setup** — initializes Serial1 hardware UART on GPIO 9 (RX) / GPIO 10 (TX) at 115200 baud

---

## Build

PlatformIO (VS Code extension or CLI):

```bash
pio run -e Heltec_t114_without_display_repeater --target upload --upload-port /dev/ttyACM0
```

---

## Configuration

Configure the node via OverMesh → Settings → MeshCore → Remote Manage:
- Set name, position (lat/lon), advert intervals, admin password
- Radio parameters must match your mesh (SLO mesh: 869.618 MHz, BW 62.5 kHz, SF8, CR8)

---

## OverMesh integration

Add the node in OverMesh → Settings → MeshCore → Remote Collectors. Press **Collect** to trigger a dump. OM sends `OMCOLLECT` to the RPTR over the mesh → RPTR signals RP2040 → RP2040 sends buffered observations as `RELAY|` lines → RPTR re-encrypts each as a DM back to OM → OM stores and visualizes automatically.

Enable the **Signal Heatmap** in Map → Overlays → Data layers for a weather-radar style RSSI coverage overlay built from collected observations.

---

## Upstream

This firmware is a patch on top of [meshcore-dev/MeshCore](https://github.com/meshcore-dev/MeshCore). All original MeshCore code and licenses apply.
