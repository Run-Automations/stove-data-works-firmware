# Run Automations — EPC Monitor Firmware

**Platform:** STM32F103C8T6 (BluePill) · **Programmer:** ST-Link V2 · **Flash tool:** STM32CubeProgrammer

---

## Table of Contents

1. [Hardware Overview](#1-hardware-overview)
2. [Prerequisites](#2-prerequisites)
3. [Before You Flash](#3-before-you-flash)
4. [Burning Firmware via ST-Link](#4-burning-firmware-via-st-link)
   - 4.1 [Wiring the ST-Link](#41-wiring-the-st-link)
   - 4.2 [Flashing with STM32CubeProgrammer](#42-flashing-with-stm32cubeprogrammer)
5. [First Boot Checklist](#5-first-boot-checklist)
6. [How the Device Works](#6-how-the-device-works)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Hardware Overview

| Component | Role |
|---|---|
| STM32F103C8T6 (BluePill) | Main microcontroller |
| SIM800L | GSM/GPRS modem — data upload and device identity |
| PZEM004T v3.0 | AC voltage, current and power measurement |
| GPS module | Location tagging |
| Relay | Remote switch control driven by the portal |

---

## 2. Prerequisites

- **Device registered on the Run Automations portal** — the IMEI of the SIM800L on the board must be registered on the portal before the board is powered on. See [§3 Before You Flash](#3-before-you-flash).
- **ST-Link V2** USB programmer (clone or genuine)
- **STM32CubeProgrammer** — [download from ST](https://www.st.com/en/development-tools/stm32cubeprog.html)
- ST-Link USB driver installed (Windows: run the driver installer bundled with STM32CubeProgrammer; Linux/macOS: libusb — usually automatic)
- Pre-built firmware binary: `epc_monitor.bin` (supplied by Run Automations)

---

## 3. Before You Flash

> ⚠️ **This step is mandatory. Do not skip it.**

Each board is bound to the Run Automations platform using the unique IMEI of its SIM800L module. The device cannot connect to the server or upload any data unless its IMEI has been registered on the portal first.

### Steps

1. **Find the IMEI.** The IMEI is a 15-digit number printed on a sticker on the SIM800L module. Note it down before assembling the enclosure.

2. **Log in to the Run Automations portal** and navigate to **Devices → Add Device**.

3. **Create a new device** and fill in:
   - **Device type:** EPC Monitor
   - **IMEI:** the 15-digit number from the SIM800L on this specific board
   - **Label / name:** a recognisable identifier for the deployment site

4. **Save the device** on the portal.

5. Only after saving should you proceed to flash and power on the board.

> **One board — one registration.** Every physical board has a unique SIM800L with a unique IMEI. Never register the same IMEI for more than one device entry.

---

## 4. Burning Firmware via ST-Link

### 4.1 Wiring the ST-Link

Connect the ST-Link V2 to the BluePill's **SWD header** (4-pin connector on the short end of the board):

```
ST-Link V2          BluePill SWD Header
─────────────────   ───────────────────
SWDIO  (pin 2)  →   SWDIO
SWDCLK (pin 6)  →   SWDCLK
GND    (pin 4)  →   GND
3.3V   (pin 8)  →   3V3   ← power the board from the programmer
```

> **Important:** Power the BluePill from the ST-Link's 3.3 V pin only — do **not** connect a separate 5 V USB supply at the same time.

Ensure the **BOOT0 jumper** is set to **0** (GND side). This is the default position for normal operation. Do not change it.

---

### 4.2 Flashing with STM32CubeProgrammer

1. Open **STM32CubeProgrammer**.
2. In the top-right connection panel set:
   - Interface: **ST-LINK**
   - Mode: **SWD**
   - Speed: **4000 kHz**
3. Click **Connect**. The device information panel should populate with the chip UID and flash size.
4. Go to the **Erasing & Programming** tab (rocket icon, left sidebar).
5. Configure the options:
   - **File path:** browse to `epc_monitor.bin`
   - **Start address:** `0x08000000`
   - ☑ **Verify programming**
   - ☑ **Run after programming**
6. Click **Start Programming**.

A successful flash looks like this in the log panel:

```
Memory Programming ...
File          : epc_monitor.bin
Size          : xx KB
Address       : 0x08000000

Programming...
████████████████████████ 100%

Verifying ...
████████████████████████ 100%

File download complete
Time elapsed: x.xx s
Start operation achieved successfully
```

The board resets automatically and begins running the firmware.

> **Connection fails?** Hold the **RESET** button on the BluePill, click **Connect** in CubeProgrammer, then release RESET. This forces a connect-under-reset.

---

## 5. First Boot Checklist

After flashing, connect a serial monitor at **115200 baud** to the debug port on the board and power it on. Within approximately 60 seconds you should see:

```
Run Automations — EPC Monitor
System ready.
```

If the device does not reach **System ready** within 2 minutes, refer to the [Troubleshooting](#7-troubleshooting) section.

Once the device is running, verify it appears as **online** in the Run Automations portal under the device entry you created in §3. The portal will begin receiving readings shortly after.

---

## 6. How the Device Works

The EPC Monitor is a self-contained field device. Once deployed and powered, it operates fully autonomously without any user interaction.

On boot the device connects to the mobile data network, authenticates with the Run Automations platform using its hardware identity, and begins monitoring the connected appliance. It periodically transmits readings — including power consumption, GPS location, and appliance status — to the platform. The relay output is controlled remotely from the portal in response to each transmission.

The device automatically detects the start and end of a cooking session based on power draw, and handles network interruptions and power cuts gracefully without losing session data.

All operational parameters are set at the factory in the firmware binary. No field configuration is required or possible beyond registering the device on the portal as described in §3.

---

## 7. Troubleshooting

**ST-Link not detected by CubeProgrammer**
- Reinstall ST-Link USB drivers from the driver folder inside the STM32CubeProgrammer installation directory.
- Try connect-under-reset: hold RESET on the BluePill, click Connect, then release RESET.
- ST-Link clones occasionally deliver low voltage — if the chip UID does not appear, power the BluePill from an external 3.3 V supply instead.

**"No target connected" during programming**
- Check that SWDIO and SWDCLK are not swapped.
- BOOT0 jumper must be at position **0** (GND side).
- On some BluePill clones the SWD interface is unresponsive if the board was previously running firmware. Hold RESET throughout the entire connect and program sequence.

**Device does not reach "System ready" after flashing**
- Confirm the SIM card is inserted and has an active data plan.
- Confirm the IMEI was registered on the portal before powering on (§3).
- Check that the SIM800L module is correctly seated and its power supply is stable.

**Device is online but shows no readings on the portal**
- Verify the GPS antenna has line of sight to the sky — indoor deployments may show zero coordinates until a fix is acquired.
- Confirm the PZEM sensor is connected to a live AC supply. The sensor requires AC on its measurement terminals to respond.

**Authentication failure shown in debug output**
- The IMEI on the SIM800L does not match any registered device on the portal. Re-check the IMEI and update or create the device entry in the portal.
- Confirm the SIM's APN setting matches the mobile operator in the deployment region. Contact Run Automations support if the APN needs to be updated.

**Relay not responding to portal commands**
- Confirm the relay wiring is intact and the coil supply voltage is correct.
- The relay follows commands from the portal only — it cannot be toggled locally.

---

*For support contact Run Automations. Do not attempt to modify or re-flash the firmware binary — doing so voids device support and will break platform authentication.*
