# Smart Kitchen – IoT System (MQTT + SSDP)

A smart kitchen IoT system written in C. Sensors, actuators, a central controller and a user application run as separate processes that communicate through a **Mosquitto MQTT broker**, while devices are discovered and tracked using the **SSDP** protocol (NOTIFY alive/byebye, M-SEARCH).

## Components

| Process | File | Role |
|---|---|---|
| Humidity sensor | `ssdp_and_mqtt/senzor_vlage.c` | Simulates humidity (%RH) and publishes it every 5 s |
| Socket sensor | `ssdp_and_mqtt/senzor_uticnice.c` | Simulates current (A) and power (W), QoS 1 |
| Range hood | `ssdp_and_mqtt/aktuator_aspirator.c` | ON/OFF, speed 0–3, AUTO/MANUAL mode |
| Smart socket | `ssdp_and_mqtt/aktuator_uticnica.c` | ON/OFF |
| Light bulb | `ssdp_and_mqtt/aktuator_sijalica.c` | ON/OFF, state published with retain=true |
| Controller | `ssdp_and_mqtt/controller.c` + `pravila.c` | SSDP device registry, automation rules |
| Application | `app/aplikacija.c` | Console view of system state and manual commands |

Shared MQTT topics, SSDP parameters and thresholds are defined in [`ssdp_and_mqtt/pravila.h`](ssdp_and_mqtt/pravila.h).

## Automation logic

- **Range hood speed by humidity** (with ±3 %RH hysteresis):
  `<50` → 0, `50–70` → 1, `70–85` → 2, `>85` → 3
- **Socket protection**: current > 16 A → `OVERCURRENT`, power > 3500 W → `OVERPOWER`; the status is published to `kuhinja/uticnica/status` and the socket is switched off. Negative values are logged as errors.
- **Command priority**: 1) socket protection → 2) manual commands from the app → 3) automatic range hood control.
- The controller sends commands **only to devices that are active in the SSDP registry**; a device with no NOTIFY for more than 25 s is considered offline. SSDP messages without a valid `AUTH` token are discarded.
- Actuator status messages are published with `retain=true`, so a new client immediately receives the current state.

## MQTT topics

| Topic | Direction | Payload |
|---|---|---|
| `kuhinja/senzor/vlaga` | sensor → | float or `ERROR` |
| `kuhinja/uticnica/struja`, `.../snaga` | sensor → | float |
| `kuhinja/aspirator/ukljucenost` \| `brzina` \| `mod` | → range hood | `ON`/`OFF` \| `0–3` \| `AUTO`/`MANUAL` |
| `kuhinja/aspirator/status` | range hood → | JSON `{ukljucenost, brzina, mod}` |
| `kuhinja/uticnica/ukljucenost` | → socket | `ON`/`OFF` |
| `kuhinja/uticnica/status` | → | `OK`/`OVERCURRENT`/`OVERPOWER` |
| `kuhinja/svetlo/glavno/ukljucenost` | → light bulb | `ON`/`OFF` |
| `kuhinja/svetlo/glavno/status` | light bulb → | JSON state |
| `kuhinja/app/get_status`, `kuhinja/app/status` | app ↔ controller | request / full system state |
| `kuhinja/app/aspirator` \| `uticnica` \| `svetlo` | app → controller | manual commands |

## Running

Requirements (Linux): `gcc`, `make`, `libmosquitto-dev` and a Mosquitto broker running on `localhost:1883`.

```bash
sudo apt install build-essential libmosquitto-dev mosquitto
make            # builds everything into bin/
```

Run each component in a separate terminal:

```bash
make run-senzor-vlage
make run-senzor-uticnice
make run-aspirator
make run-uticnica
make run-sijalica
make run-controller   # runs from ssdp_and_mqtt/ because of config.json
make run-app
```

Or start everything at once (the app in the foreground, the rest in the background): `make run-all`. Other targets: `make clean`, `make help`.

## Structure

```
app/              user application
ssdp_and_mqtt/    sensors, actuators, controller, rules, config.json
spec/             project specification (v1–v3, PDF/DOCX)
Makefile
```

Detailed requirements (HLR/SWR/ARCH) are in [`spec/`](spec/).

## Authors

Katarina Lazarević, Sara Janković, Ivana Radić, Marija Sekulić
