# FIK-10 - Stratospheric Balloon Experiment

This repository documents the hardware and software used for the FIK-10 flight on April 28, 2026. The flight was intended to test the usability of a PX4-based autopilot in combination with the [AIRDOS03](https://docs.thunderfly.cz/avionics/AIRDOS03/) cosmic ray detector.
It is primarily a reproducible integration guide, backed by a mission report showing that the equipment worked in a real stratospheric balloon flight.

![FIK-10 stratospheric balloon gondola](doc/img/gondola.jpg)

## Mission summary

The balloon was launched from the [Praha-Libus meteorological station](https://cs.wikipedia.org/wiki/Meteorologick%C3%A1_v%C4%9B%C5%BE_Libu%C5%A1). The AIRDOS03 detector performed very well during the flight. The plotted data show count rates in higher energy channels, including two crossings of the Regener-Pfotzer maximum: once during ascent and once during descent.

The full raw and processed data sets are large, on the order of gigabytes. They can be provided on request.

## Repository contents

This repository is a flight-specific integration snapshot. It combines documentation, configuration notes, firmware sources, and the ground application used during the FIK-10 mission.

  - `README.md` - flight-level documentation and configuration overview.
  - `doc/img/` - photos of the gondola and plotted flight data.
  - `fw/AIRDOS03/` - AIRDOS03 firmware, including the MAVLink TUNNEL variant used for PX4 integration.
  - `fw/SiK/` - SiK telemetry modem firmware and provisioning scripts for the one-way balloon link.
  - `sw/GAPP/` - GAPP ground application server and dashboard.
  - `sw/GAPP-cli/` - CLI uploader used in chase cars to send station position and telemetry to GAPP.

## System overview

FIK-10 used a PX4 autopilot as the central telemetry router. AIRDOS03 produced radiation measurement data as MAVLink TUNNEL packets, the autopilot forwarded telemetry through LoRa and SiK radio links, and chase cars uploaded received telemetry to the GAPP server for tracking and prediction.

The main onboard data paths were:

  - AIRDOS03 -> UNIPAYLOAD01 -> PX4 Telem2 -> uLog file
  - TFGPS01 -> PX4 primary GPS port
  - PX4 -> TFLoRA01 over SPI
  - PX4 Telem1 -> TFSIK01
  - LoRaWAN Ground Gateways and chase cars SiK receiver -> GAPP-cli -> GAPP server

## Technical description

![Balloon electronics](doc/img/gondola_internals.jpg)

### Hardware configuration

Based on the [TF-B1](https://docs.thunderfly.cz/instruments/TF-B1) PX4 subsystem. Final gondola mass: 534 g, measured after the flight.

  - [CUAV 5+](https://docs.px4.io/main/en/flight_controller/cuav_v5_plus) (Autopilot FMU)
  - [TFLoRA01](https://docs.thunderfly.cz/avionics/TFLORA01/) (connected to the autopilot SPI bus)
  - [TFSIK01](https://docs.thunderfly.cz/avionics/TFLORA01/) (connected to the autopilot Telem1 port)
  - [UNIPAYLOAD01](https://docs.thunderfly.cz/avionics/TFUNIPAYLOAD01/) with [AIRDOS03](https://docs.dos.ust.cz/airdos/AIRDOS03) (connected to Telem2 and linked to TFGPS01 via the TFPayload connector)
  - [TFGPS01](https://docs.thunderfly.cz/avionics/TFGPS01/) (connected to the autopilot primary GPS port)
  - Power from the two-cell MLAB [LION2CELL01](https://www.mlab.cz/module/LION2CELL01/) module through a [TPS63060V01](https://www.mlab.cz/module/TPS63060V01/) converter to 5 V.

### Software configuration

#### PX4

Firmware was modified so forwarding continues even when there is no heartbeat on the other side of the link.

```text
MAV_0_CONFIG = TELEM 1
MAV_0_FORWARD = Enable
MAV_0_RADIO_CTL = Enable
MAV_0_MODE = Minimal
MAV_0_RATE = 0
SER_TEL2_BAUD = 115200 8N1
SER_TEL1_BAUD = 9600 8N1
MAV_1_CONFIG = TELEM 2
MAV_1_MODE = Normal
MAV_1_FORWARD = Enable
MAV_1_RADIO_CTL = Disable
SDLOG_PROFILE = 1041
SDLOG_MODE = from boot until shutdown
```

TODO: Add the LoRa configuration after it has been verified.

#### SiK

Configured using the scripts included in the [repository scripts folder](https://github.com/ThunderFly-aerospace/SiK/tree/226d3f2c34546ddb9b1f5473599bf921491b5561/scripts).
The local copy also contains a [SiK provisioning README](fw/SiK/scripts/README.md) with the expected one-way modem setup workflow.

  - AIRSPEED 2
  - Balloon BAUD 9600
  - GCS BAUD 57600

#### AIRDOS03

The AIRDOS03 firmware variant used here is documented in [fw/AIRDOS03/fw/AIRDOS03_MAVLink/README.md](fw/AIRDOS03/fw/AIRDOS03_MAVLink/README.md).

  - The firmware sends a MAVLink TUNNEL message every 10 seconds, with data addressed to system 1, component 1 (the autopilot).
  - Every 60 seconds, the firmware broadcasts an ALIVE TUNNEL message, which is forwarded to the ground station for launch checks.

#### GAPP - Ground Application

Telemetry uploads from the LoRa and SiK modems were handled by GAPP running on the fik.crreat.eu server with 4 cars and 2 balloons (separate LoRa and SiK balloon instances, usable for different predictions).
[GAPP-cli](https://github.com/ODZ-UJF-AV-CR/GAPP-cli) was used in the cars. In 3 cases, it was connected to [QGroundControl](https://qgroundcontrol.com/), which was configured to forward MAVLink data to the UDP port received by GAPP-cli. In the 4th car, Roman modified the application operationally so that the CLI printed MAVLink packets to the terminal and the serial MAVLink stream was connected directly to GAPP-cli without using QGC.

Example `car-4.toml` configuration:

```toml
[uploader]
enabled = true
station_callsign = "fik-car-4"
server_url = "https://fik.crreat.eu"

[gpsd]
enabled = true
#host = "127.0.0.1"
#port = 2947
interval = 5

[mavlink]
callsign = "fik-sik"
enabled = true
connection_string = "/dev/ttyUSB0"
baud = 57600
print_packets = true
# connection_string = "udpin:0.0.0.0:14550"
source_system = 1
source_component = 1
```

## Reusing this setup

The repository can be used as a reference for similar balloon flights that need to carry a PX4-based avionics stack and forward an instrument data stream over MAVLink.

Recommended starting points:

  - Review the hardware list above and the gondola wiring photo.
  - Flash the AIRDOS03 MAVLink firmware from `fw/AIRDOS03/fw/AIRDOS03_MAVLink` to AIRDOS03 hardware.
  - Configure the SiK balloon and GCS modems using `fw/SiK/scripts/configure_balloon_modem.py` and `fw/SiK/scripts/configure_gcs_modem.py`.
  - Configure PX4 telemetry forwarding so Telem2 receives instrument data and Telem1 is used for the SiK downlink.
  - Run GAPP and GAPP-cli for live telemetry collection and chase-car position upload.

## Results

![FIK-10 stratospheric balloon measured data](doc/img/flight_data.png)

The figure shows AIRDOS03 count rates in higher energy channels during the stratospheric balloon flight. The two visible enhancements correspond to crossings of the Regener-Pfotzer maximum on the way up and on the way down.

The result confirms that the AIRDOS03 works as expected and  MAVLink telemetry path through the PX4-based gondola avionics was confirmed for this mission profile.


