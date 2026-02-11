# ESP32-Hcalory-HBH1S-bluetooth
ESP32 control by MQTT for Hcalory HBH1S Diesel Heater via bluetooth

With this code you can control Hcalory HBH1S diesel heater by sending MQTT commands and read data with MQTT with ESP32 S3

This is a hobby project done for my personal need to remote control Hcalory HBH1S heater with MQTT. 
Code is part of bigger project to remote control off-grid cabin gear. With this I can control heater over wifi. My MQTT broker is in the cabin and using Tailscale so I can remote control heater over the Internet before arriving to cabin.

Code is widely generated using Google Gemini AI and it is also used to help analyze Bluetooth traffic. Code is generated using Finnish language so some parts are in Finnish (you can still quite easily follow it, or use AI to translate)

Bluetooth traffic is reverse engineered from android app. Bluetooth log is exported from tablet and I used Wireshark to analyze log with help of Google Gemini.

Tested with ESP32 S3 mini and working.

Full Bluetooth protocol is not analyzed just parts to send commands and read needed data:
- Heating
- Fan
- Fan level
- OFF
- Status Query
  receiving:
  - Main status
  - Voltage
  - Ambient temp
  - Alu temp
  - Blow level

This is tested just with one heater and there might be heater specific parts in commands and this might not work with any other heater. If this is the case you can do the same I did and analyze the phone/tabler bluetooth log and use AI to assist to analyze code and modify the yaml code. 
I have no personal interest to develop this further.

BEFORE COMPILE: Change WIFI settings and MQTT Broker to your configuration in yaml file

# TEST UI
index.html is light UI to use and test heater. Change your MQTT broker address so you can use the heater. This is also half finnish.
You will also need MQTT broker to be configured on your machine before UI and ESP will communicate.
Personally I have hoster this UI and MQTT broker on Raspberry PI Zero 2WH machine with Tailscale to have control over Internet to heater. UI host I have been using is nginx.

Hear is AI generated documentation of yaml:

# TECHNICAL SPECIFICATION: HCALORY DIESEL HEATER BLE PROTOCOL

This document describes the Bluetooth Low Energy (BLE) communication protocol 
between the ESP32 (ESPHome) and the HCalory Diesel Heater.

-------------------------------------------------------------------------------
1. HANDSHAKE SEQUENCE (CONNECTION INITIALIZATION)
-------------------------------------------------------------------------------
Executed immediately after BLE connection to authorize the controller.

Phase          | HEX Command (GATT 0x0003 / 0x0006)                             | Description                   | Delay
---------------|----------------------------------------------------------------|-------------------------------|-------
Notifications  | 01 00 (GATT 0x0006)                                             | Activate heater responses     | 0 ms
Login          | 00 02 00 01 00 01 00 0A 0C 00 00 05 01 00 00 00 00 12          | Authentication                | 1000 ms
Security Lock  | 02 10 00 1D 00 19 00 04 00 52 03 00 00 02 00 01 00 01 00 0E... | Unlock control interface      | 500 ms
Activation     | 02 10 00 16 00 12 00 04 00 52 03 00 00 02 00 01 00 01 00 07... | Confirm authorization         | 300 ms

*Full Security Lock: 02 10 00 1D 00 19 00 04 00 52 03 00 00 02 00 01 00 01 00 0E 04 00 00 09 00 00 00 00 00 00 00 00 00 0D
*Full Activation:    02 10 00 16 00 12 00 04 00 52 03 00 00 02 00 01 00 01 00 07 0C 00 00 02 01 00 0F

-------------------------------------------------------------------------------
2. POLLING (STATUS UPDATES)
-------------------------------------------------------------------------------
Sent every 5 seconds. Byte 26 (seq) is a counter, Byte 29 (crc) is 0x56 + seq.

Type         | Full HEX Command (30 bytes)
-------------|-----------------------------------------------------------------
Status Query | 02 10 00 19 00 15 00 04 00 52 03 00 00 02 00 01 00 01 00 0A 0A 00 00 05 0D 35 15 05 00 6B

-------------------------------------------------------------------------------
3. CONTROL COMMANDS
-------------------------------------------------------------------------------
Sent from ESP32 to Heater via MQTT.

Function     | MQTT Topic                        | Full HEX Command (GATT 0x0003)                                   | Length
-------------|-----------------------------------|------------------------------------------------------------------|--------
POWER ON     | .../lammitin_virta_on/command      | 02 10 00 1D 00 19 00 04 00 52 03 00 00 02 00 01 00 01 00 0E...02 0F | 34 B
POWER OFF    | .../lammitin_virta_off/command     | 02 10 00 1D 00 19 00 04 00 52 03 00 00 02 00 01 00 01 00 0E...01 0E | 34 B
FAN MODE     | .../lammitin_puhallus/command      | 02 10 00 1D 00 19 00 04 00 52 03 00 00 02 00 01 00 01 00 0E...08 15 | 34 B
SET POWER    | .../lammitin_tehoasetus/command    | 02 10 00 15 00 11 00 04 00 52 03 00 00 02 00 01 00 01 00 06...      | 26 B

*Power Command detail: ... 00 06 07 00 00 01 [LEVEL] [CRC] (CRC = LEVEL + 0x08)

-------------------------------------------------------------------------------
4. STATUS MESSAGE INTERPRETATION (RESPONSE)
-------------------------------------------------------------------------------
Decoded from the heater's notification response.

Byte (Idx) | Data Info      | Interpretation / Formula         | Example (HEX) | Result
-----------|----------------|----------------------------------|---------------|--------
21         | Operation Mode | 00=OFF, 02=Heating, 03=Fan only  | 02            | Heating
22         | Fan Speed      | Direct value (1-10)              | 05            | 5 lvl
25         | Voltage        | Value / 10                       | 7D (125)      | 12.5 V
28         | Aluminum Temp  | Value / 10 (Heat Exchanger)      | 01 F4 (500)   | 50.0 C
31         | Ambient Temp   | Value / 10 (Intake Air)          | 00 C8 (200)   | 20.0 C
-------------------------------------------------------------------------------
