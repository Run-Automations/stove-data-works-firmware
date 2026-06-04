# Run Automations — EPC Monitor Firmware - Namoos

**Platform:** STM32F103C8T6 (BluePill) · **Programmer:** ST-Link V2 · **Flash tool:** STM32CubeProgrammer

---

## Table of Contents

1. [Hardware Overview](#1-hardware-overview)
2. [Prerequisites](#2-prerequisites)
3. [Before You Flash](#before-you-flash)
4. [Burning Firmware via ST-Link](#3-burning-firmware-via-st-link)
   - 4.1 [Wiring the ST-Link](#31-wiring-the-st-link)
   - 4.2 [Flashing with STM32CubeProgrammer](#32-flashing-with-stm32cubeprogrammer)
5. [First Boot Checklist](#4-first-boot-checklist)
6. [Firmware Process Flow](#5-firmware-process-flow)
   - 6.1 [Boot & Initialisation](#51-boot--initialisation)
   - 6.2 [Device Authentication](#52-device-authentication)
   - 6.3 [Main Loop](#53-main-loop)
   - 6.4 [Cooking Session State Machine](#54-cooking-session-state-machine)
   - 6.5 [Data Upload](#55-data-upload)
   - 6.6 [Connectivity Recovery](#56-connectivity-recovery)
   - 6.7 [Hourly Watchdog](#57-hourly-watchdog)
7. [EEPROM Layout](#6-eeprom-layout)
8. [Configuration Reference](#7-configuration-reference)
9. [Troubleshooting](#8-troubleshooting)

---

## 1. Hardware Overview

| Component | Role |
|---|---|
| STM32F103C8T6 (BluePill) | Main microcontroller |
| SIM800L | GSM/GPRS modem — data upload + IMEI source |
| PZEM004T v3.0 | AC voltage, current and power measurement |
| GPS module (NMEA, 9600 baud) | Location tagging |
| Relay (active HIGH) | Remote switch control via server response |

### Pin Map

| Pin | Function |
|---|---|
| PA2 | SIM800L TX → Serial RX |
| PA3 | SIM800L RX ← Serial TX |
| PA8 | SIM800L PWRKEY |
| PA2 / PA3 | PZEM UART (Serial2) |
| PB10 | Debug monitor TX (Serial3) |
| PB11 | Debug monitor RX (Serial3) |
| PB8 | GPS RX (SoftwareSerial) |
| PB9 | GPS TX (SoftwareSerial) |
| PA4 | Relay control (HIGH = ON) |

---

## 2. Prerequisites

- **Device registered on the Run Automations portal** — the IMEI of the SIM800L module on the board must be registered as a device on the portal **before** the firmware is flashed. The firmware reads the IMEI directly from the modem at boot and uses it to authenticate against the server. If the IMEI is not registered, authentication will fail and the device will not upload any data. See [Before You Flash](#before-you-flash) below.
- **ST-Link V2** USB programmer (clone or genuine)
- **STM32CubeProgrammer** — [download from ST](https://www.st.com/en/development-tools/stm32cubeprog.html)
- ST-Link USB driver installed (Windows: run the driver installer bundled with STM32CubeProgrammer; Linux/macOS: libusb — usually automatic)
- Pre-built firmware binary: `epc_monitor.bin` (supplied by Run Automations)

---

## Before You Flash

> ⚠️ **This step is mandatory. Do not skip it.**

The firmware authenticates with the Run Automations server on every boot using the IMEI of the SIM800L module soldered on the board. The server looks up the device by IMEI to issue a session token. **If the IMEI is not registered on the portal before the device is powered on, authentication will fail and the device will not function.**

### Steps

1. **Read the IMEI off the SIM800L module.** The IMEI is a 15-digit number printed on a sticker on the module. If the sticker is missing or unreadable, you can retrieve it by temporarily connecting a serial terminal (115200 baud) to PB10/PB11 after powering the board — the firmware logs it as `[IMEI] IMEI: <number>` during boot.

2. **Log in to the Run Automations portal** and navigate to **Devices → Add Device**.

3. **Create a new device** and enter:
   - **Device type:** EPC Monitor
   - **IMEI:** the 15-digit number from the SIM800L on this specific board
   - **Label / name:** a human-readable identifier for the deployment site

4. **Save the device.** The portal will assign a serial number to the IMEI. The firmware fetches this serial number on first boot and caches it in the board's EEPROM — subsequent boots use the cached value without a network call.

5. Only after the device is saved on the portal should you proceed to flash the firmware.

> **One IMEI — one device.** Each physical board has a unique SIM800L with a unique IMEI. Do not reuse the same IMEI entry for multiple boards.

---

## 3. Burning Firmware via ST-Link

### 3.1 Wiring the ST-Link

Connect the ST-Link V2 to the BluePill's **SWD header** (4-pin connector on the end of the board):

```
ST-Link V2          BluePill SWD Header
─────────────────   ───────────────────
SWDIO  (pin 2)  →   SWDIO
SWDCLK (pin 6)  →   SWDCLK
GND    (pin 4)  →   GND
3.3V   (pin 8)  →   3V3   ← power the board from the programmer
```

> **Important:** Power the BluePill from the ST-Link's 3.3 V pin during programming — do **not** connect a separate 5 V USB supply and the ST-Link 3.3 V at the same time.

Ensure the **BOOT0 jumper** is set to **0** (GND side — normal boot from flash). BOOT0 = 1 is only needed for UART DFU and must not be set here.

---

### 3.2 Flashing with STM32CubeProgrammer

1. Open **STM32CubeProgrammer**.
2. In the top-right connection panel set:
   - Interface: **ST-LINK**
   - Mode: **SWD**
   - Speed: **4000 kHz**
3. Click **Connect**. The device information panel should populate with the chip UID and flash size (64 KB or 128 KB).
4. Go to the **Erasing & Programming** tab (rocket icon, left sidebar).
5. Configure the programming options:
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

> **Connection fails?** Hold the **RESET** button on the BluePill, click **Connect** in CubeProgrammer, then release RESET. This forces a connect-under-reset and works on boards that are already running firmware that interferes with SWD.

---

## 4. First Boot Checklist

After flashing, open a serial monitor at **115200 baud** on the debug port (PB10/PB11 — Serial3) and verify the following sequence appears:

```
========================================
  Run Automations — EPC Monitor + Auth
  PZEM004T + SIM800L + GPS + STM32 BluePill
========================================

[INIT] Relay LOW
[INIT] PZEM serial started
[INIT] GPS serial started
[SETUP] No manual IMEI — reading from modem...
[IMEI] IMEI: <15-digit number>
[GSM] Configuring GPRS...
[GSM] Network registered!
[AUTH] ===== runAuthSequence() =====
[AUTH] Serial from EEPROM: ...   ← or "Fetching serial from server..."
[AUTH] streamId: <token>
[AUTH] Auth sequence complete
[INIT] System ready!
```

If `[AUTH] Auth failed at boot` appears, the device will still enter the main loop and retry authentication on the first upload cycle — this is non-fatal.

---

## 5. Firmware Process Flow

### 5.1 Boot & Initialisation

```
Power on
    │
    ├─ GPIO init: Relay → LOW, PWRKEY pin OUTPUT
    ├─ Serial2 (PZEM) @ 9600
    ├─ Serial3 (debug) @ 115200
    ├─ SoftwareSerial (GPS) @ 9600
    ├─ EEPROM read → check session-active flag (power-cut recovery)
    ├─ Resolve IMEI
    │     ├─ MANUAL_IMEI set? → use it
    │     └─ MANUAL_IMEI empty → AT+GSN → parse 15-digit IMEI from modem
    │              └─ fail? → NVIC reset
    ├─ SIM800L power-cycle (PWRKEY toggle → 30 s boot wait)
    ├─ configureGPRS() → network registration + bearer open
    ├─ runAuthSequence() → serial number + streamId (see §5.2)
    └─ Power-cut recovery (if flag was set) → sendState(CS=2) → clear flag
```

---

### 5.2 Device Authentication

Authentication is a two-step sequence that runs at boot and is transparently repeated whenever the session token nears expiry.

```
runAuthSequence()
    │
    ├─ ensureConnectivity()          ← bearer must be up first
    │
    ├─ fetchSerialNumber()
    │     ├─ EEPROM has serial? → use cached value (skips network call)
    │     └─ EEPROM empty →
    │           POST /device-auth/get-serial-number/{imei}
    │               └─ parse "serialNumber" from JSON response
    │                   └─ persist to EEPROM[0x10] for future boots
    │
    └─ authenticate()
          POST /device-auth/authenticate  {imei, serialNumber}
              └─ parse "streamId" from JSON response
                  └─ set streamExpiry = now + 55 min

Token refresh:
    Every call to sendState() checks streamIdExpired()
        millis() + AUTH_MARGIN (5 min) >= streamExpiry?
            YES → runAuthSequence() before uploading
            NO  → proceed with current streamId
```

The `streamId` is sent as an `X-Stream-Id` HTTP header on every subsequent data upload. A `401` or `403` response from the server immediately invalidates the local token, forcing re-auth on the next cycle.

On clean shutdown (hourly NVIC reset), `disconnectDevice()` is called first — it POSTs to the disconnect endpoint with the current `streamId` so the server can close the session gracefully.

---

### 5.3 Main Loop

The loop runs continuously with no `delay()` blocking the hot path. All timing is millis()-based.

```
loop() ─ runs every iteration (~ms)
    │
    ├─ Feed GPS parser (non-blocking char reads from SoftwareSerial)
    │     └─ valid fix? → latch gpsLat / gpsLon
    │
    ├─ Read PZEM current (I)
    │
    ├─ Cooking state machine (§5.4)
    │
    ├─ Heartbeat upload?
    │     now − lastUpload >= 60 s
    │         └─ currentCS != 2 → sendState(currentCS)
    │
    └─ Hourly watchdog?
          now − lastReset >= 3600 s
              └─ hardwareReset("hourly watchdog")
                    ├─ disconnectDevice()
                    └─ NVIC_SystemReset()
```

---

### 5.4 Cooking Session State Machine

The EPC tracks whether the pressure cooker is active using AC current as the signal. States are encoded as an integer (`CS`) included in every upload payload.

```
States:
  -1  IDLE        — no current, not cooking
   0  STARTED     — current first detected (single event upload)
   1  ONGOING     — current confirmed sustained for 30 s
   2  ENDED       — current gone for 5 min (session end upload)

Transitions:

  I > 0.5 A  ────────────────────────────────────────────────────────────
                                                                         │
  CS = -1  ──[current detected]──▶  CS = 0  ──[sustained 30 s]──▶  CS = 1
            sendState(0)                                sendState(1)    │
                                                                        │
  I = 0 A  ───────────────────────────────────────────────────────────  │
                                                                        │
  CS = 1  ──[gap >= 5 min]──▶  CS = 2  ──[sent]──▶  CS = -1           │
                 sendState(2)                                            │
                                                                        │
  CS = 0  ──[gap >= 5 min, no confirmation]──▶  CS = -1  (false start) │
                                                                        │
  POWER CUT during CS = 1 or CS = 0:                                   │
    EEPROM[0x00] = 1 (written at CS=0 upload)                          │
    Next boot detects flag → sendState(2) → EEPROM[0x00] = 0  ────────┘
```

The EEPROM session flag ensures a cooking session is always properly closed even if the device loses power mid-cook.

---

### 5.5 Data Upload

`sendState(csValue)` handles a single upload cycle:

```
sendState(csValue)
    │
    ├─ ensureConnectivity()
    ├─ streamIdExpired()? → runAuthSequence()
    ├─ readGPS() — 1 s non-blocking window, latches latest fix
    ├─ pzem.voltage() / .current() / .power()
    │     NaN or < 0 → clamp to 0
    ├─ Build JSON payload:
    │     {IMEI, BV, LC, Lat, Long, CC, Temp:0, MT:{WGT:0, CS}}
    ├─ httpPost(resource, payload, X-Stream-Id header)
    │     ├─ 200/201 → parse response JSON
    │     │     └─ switchState == "Active"? → relay HIGH : relay LOW
    │     ├─ 401/403 → invalidate streamId (re-auth next cycle)
    │     └─ other  → log warning, continue
    └─ EEPROM session flag update
          CS=0 → EEPROM[0x00] = 1
          CS=2 → EEPROM[0x00] = 0
```

The relay state is driven entirely by the server's `switchState` field — the device has no local relay logic of its own.

---

### 5.6 Connectivity Recovery

`ensureConnectivity()` implements a three-stage recovery ladder before giving up:

```
Stage 1 — Check bearer
    AT+SAPBR=2,1 → state == 1 (connected)? → OK ✓

Stage 2 — Soft recovery (no power cycle)
    Close bearer → wait for CREG → re-open bearer
    Success? → OK ✓

Stage 3 — Modem power-cycle
    PWRKEY LOW 2 s → HIGH → wait 20 s → AT check
    Modem unresponsive? → NVIC reset
    Wait CREG → re-open bearer
    Success? → OK ✓
    Still failing? → NVIC reset
```

`hardwareReset()` is always a clean operation: it calls `disconnectDevice()` first (best-effort POST to the disconnect endpoint) before pulling PWRKEY and issuing `NVIC_SystemReset()`.

---

### 5.7 Hourly Watchdog

Every 3600 s the device performs a clean NVIC reset. This is a reliability measure that:

- Clears any accumulated modem state or GPRS stall
- Refreshes the bearer and re-authenticates
- Resets `millis()` overflow concerns for long-running deployments

The disconnect endpoint is always called before the reset so the backend can log the session end cleanly.

---

## 6. EEPROM Layout

| Address | Size | Content |
|---|---|---|
| `0x00` | 1 byte | Session-active flag (`1` = power cut mid-session, `0` = idle) |
| `0x01`–`0x0F` | 15 bytes | Reserved |
| `0x10` | 20 bytes | Device serial number (null-terminated ASCII, max 19 chars) |

The serial number is fetched once from the server (keyed to the device IMEI) and cached at `0x10`. Subsequent boots skip the network call unless the EEPROM byte at `0x10` is blank or `0xFF`.

To force a fresh serial number fetch (e.g. after replacing the SIM or reprogramming a device), clear EEPROM bytes `0x10`–`0x23` to `0xFF`.

---

## 7. Configuration Reference

| Constant | Default | Description |
|---|---|---|
| `MANUAL_IMEI` | `""` | Hard-code an IMEI. Leave empty to read from modem via AT+GSN |
| `UPLOAD_INTERVAL` | 60 s | Heartbeat upload period |
| `RESET_INTERVAL` | 3600 s | Hourly NVIC watchdog reset period |
| `START_CONFIRM_MS` | 30 s | Current must stay above threshold this long before CS=0 → CS=1 |
| `SESSION_GAP_MS` | 300 s | Current must be zero this long before CS=1 → CS=2 (session end) |
| `AUTH_MARGIN_MS` | 300 s | Re-authenticate this many ms before token expiry |
| `COOKING_CURRENT_THRESHOLD` | 0.5 A | Minimum current to consider the cooker active |
| `RELAY_PIN` | PA4 | Relay output pin (HIGH = relay on) |
| `apn` | `"internet"` | GPRS APN for the SIM card |

---

## 8. Troubleshooting

**ST-Link not detected**
- Reinstall ST-Link drivers from STM32CubeProgrammer's driver folder.
- Try connecting under reset: hold RESET on the BluePill, click Connect in CubeProgrammer, then release RESET.
- Check 3.3 V supply — ST-Link clones sometimes deliver less than 3.3 V; use a separate supply if needed.

**Upload fails with "No target connected"**
- Verify SWDIO/SWDCLK are not swapped.
- BOOT0 jumper must be at position **0** (GND side).
- Some BluePill clones ship with a wrong crystal — if the board was previously running at the wrong speed, the SWD interface may be unresponsive. Hold RESET throughout the entire connect + program sequence.

**IMEI read fails at boot (`FATAL: Could not read IMEI`)**
- SIM800L may still be booting — the 30 s wait in `setup()` is usually sufficient but some modules are slower. Increase the `delay(30000)` to `delay(35000)` as a diagnostic step.
- Check that `PA8` (PWRKEY) is correctly wired — the modem won't power on otherwise.
- Confirm baud rate: the firmware assumes 9600 baud on `SerialAT` (Serial).

**PZEM reads NaN**
- Check wiring on Serial2 (PA2/PA3). PZEM TX → PA2 (Serial2 RX), PZEM RX → PA3 (Serial2 TX).
- PZEM004T v3.0 requires AC power on its measurement terminals before it will respond — it will return NaN with no load connected.

**No GPS fix (`[GPS] No fix yet`)**
- Normal in an indoor environment. The firmware stores `(0.0, 0.0)` and uploads the last known fix until a satellite fix is acquired.
- Typical cold-start TTFF outdoors is 30–90 s.

**Authentication keeps failing**
- Verify the SIM has active data (check APN value matches your operator).
- Confirm the device IMEI has been registered on the portal before deployment.
- Check `[BEARER]` logs — if bearer state stays `-1` or `3`, GPRS is not connecting (SIM issue or APN mismatch).

**Relay not switching**
- The relay follows the `switchState` field in the server response. If no response body is received the relay state is unchanged from its last known state.
- Confirm `RELAY_PIN` (PA4) is correctly wired and the relay coil voltage matches the supply.
