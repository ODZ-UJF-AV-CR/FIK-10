# FIK-10

Repozitář odkazující na použitý SW v rámci letu Fik-10, 28.4.2026

## HW - PX4 subsystém
  - CUAV 5+ (kostka)
  - TFLoRA (připojena do SPI)
  - TF-SiK (připojeno do Telem1)
  - UNIPAYLOAD01 s AIRDOS03 (připojeno do Telem2, a propojeno s GPS)
  - TFGPS (připojeno do GPS)
  - Napájení z 2 článkového MLAB modulu (?)

## SW konfigurace
#### Sik
  - skripty přiloženými v repozitáři ve složce skripts
  - AIRSPEED 2
  - Balon BAUD 9600
  - GCS BAUD 57600

#### AIRDOS03
  - FW vysílí každých 10s mavlink TUNNEL zprávu, s daty adresovanou na system 1 component 1 (autopilot)
  - FW každých 60 s vysílá ALIVE TUNELOVOU zprávu jako broadcast, ta se přeposílá na zem (kontrola při startu)

#### PX4
  - úprava kodu, aby přeposílání běželo ikdyž na druhé straně linky není heatbeat
  - `SER_TEL1_BAUD = 9600 8N1`
- `MAV_0_CONFIG = TELEM 1`
- `MAV_0_FORWARD = Enable`
- `MAV_0_RADIO_CTL = Enable`
- `MAV_0_MODE = Minimal`
- `MAV_0_RATE = 0`
- `SER_TEL2_BAUD = 115200 8N1`
- `MAV_1_CONFIG = TELEM 2`
- `MAV_1_MODE = Normal`
- `MAV_1_FORWARD = Enable`
- `MAV_1_RADIO_CTL = Disable`
- `SDLOG_PROFILE = 1041`
- `SDLOG_MODE = from boot until shutdown`

Chybí konfigurace LoRA...


