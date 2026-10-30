# EmirSense-1 telemetry and command protocol

Version 1.0 · applies to the flight ESP32, the ground-station ESP32 and the mission control dashboard.

## Link overview

```
CubeSat ESP32 ──LoRa 433 MHz──▶ Ground ESP32 + SX1278 ──USB serial 115200──▶ Dashboard (Web Serial)
              ◀──────── commands (in the 300 ms listen window) ────────────────
```

- Every line is plain ASCII, comma-separated, ending in `\n`.
- The ground ESP32 forwards telemetry lines to USB unchanged, appending `,rssi,snr` from the LoRa radio.
- Lines that start with anything other than `ES1,` or `ACK,` are shown in the dashboard event log as ground-station messages, which is useful for debug prints.

## Telemetry packet

```
ES1,seq,time,mode,tempC,hum,press_hPa,gas_kOhm,busV,mA,mW,dhtT,dhtH,roll,pitch,gyroZ,wheelPWM,ldr,sd[,rssi,snr]
```

| # | Field | Unit | Source | Notes |
|---|---|---|---|---|
| 1 | `seq` | count | ESP32 | Increments every packet, including packets that are not sent. Gaps show lost packets. |
| 2 | `time` | s | ESP32 | Mission clock. Runs 10× faster in demo mode. `time % 5400` is the orbit phase. |
| 3 | `mode` | text | ESP32 | `SUN`, `PAY`, `GND`, `ECL` or `SAFE` |
| 4 | `tempC` | °C | BME680 | Holds the last reading outside Payload mode |
| 5 | `hum` | %RH | BME680 | |
| 6 | `press_hPa` | hPa | BME680 | |
| 7 | `gas_kOhm` | kΩ | BME680 | Gas resistance |
| 8 | `busV` | V | INA219 | Battery voltage |
| 9 | `mA` | mA | INA219 | Load current |
| 10 | `mW` | mW | INA219 | Load power |
| 11 | `dhtT` | °C | DHT11 | Whole degrees |
| 12 | `dhtH` | %RH | DHT11 | Whole percent |
| 13 | `roll` | ° | MPU6050 | |
| 14 | `pitch` | ° | MPU6050 | |
| 15 | `gyroZ` | °/s | MPU6050 | Spin rate about the wheel axis |
| 16 | `wheelPWM` | % | TB6612FNG | Signed, −100 to 100 |
| 17 | `ldr` | 0–4095 | KY-018 | Raw 12-bit ADC. Below about 400 means eclipse. |
| 18 | `sd` | 0/1 | MicroSD | 1 = last write succeeded |
| 19 | `rssi` | dBm | Ground SX1278 | Added by the ground station |
| 20 | `snr` | dB | Ground SX1278 | Added by the ground station |

Example:

```
ES1,412,2380,PAY,28.71,41.3,1008.42,121.6,4.012,331.8,1331,31,36,2.4,-1.1,0.42,4,3290,1,-66,8.7
```

Leave a field empty if a sensor fails (`...,28.71,,1008.42,...`). The dashboard shows `—` for it.

## Orbit modes

| Mode | Phase (s) | Duration | On |
|---|---|---|---|
| `SUN` | 0–1800 | 30 min | Reaction wheel, driver, MPU6050, ESP32, solar + CN3065, INA219, MT3608, LDR, DHT11 |
| `PAY` | 1800–3300 | 25 min | As SUN, plus BME680 and MicroSD |
| `GND` | 3300–3900 | 10 min | MPU6050, ESP32, SX1278 LoRa, solar + CN3065, INA219, MT3608, LDR, DHT11, MicroSD |
| `ECL` | 3900–5400 | 25 min | MPU6050, ESP32, INA219, MT3608, LDR, DHT11 |
| `SAFE` | any | until `MODE_AUTO` | As ECL. Forced by command or by low battery. |

In flight-realistic mode the radio only transmits during `GND`. In demo mode the link stays on so the bench demo shows continuous data.

## Commands

The dashboard sends:

```
CMD,id,action
```

After each telemetry packet the CubeSat listens for 300 ms. The ground station sends any queued command in that window. The CubeSat replies:

```
ACK,id,OK
ACK,id,REJECTED
```

If no ACK arrives within 3 s, the dashboard resends once, then marks the command **No ACK**.

| Action | Effect | Rejected when |
|---|---|---|
| `MODE_SAFE` | Force safe mode | never |
| `MODE_AUTO` | Return to the orbit timer | battery below 3.5 V |
| `WHEEL_ON` | Enable the reaction wheel | battery below 3.5 V |
| `WHEEL_OFF` | Disable the reaction wheel | never |
| `PAYLOAD_NOW` | Take one BME680 reading now | never |
| `DEMO_ON` | Run the orbit clock 10× faster | never |
| `DEMO_OFF` | Run the orbit clock in real time | never |
| `PING` | Link test | never |

Outside demo mode, commands are only heard during the Ground Station pass.

## On-board safety rules

- Below 3.5 V the CubeSat enters `SAFE` by itself and refuses `WHEEL_ON` and `MODE_AUTO`.
- Keep the reaction wheel PWM under about 40% on the bench. The 130 motor spins up to 17,000 RPM and a printed flywheel can crack if unbalanced.
