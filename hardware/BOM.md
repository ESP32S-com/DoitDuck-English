# Bill of Materials

Hardware required for one DoitDuck. See the respective schematics for the complete component lists, including resistors and capacitors, for the two custom boards.

## Complete robot

| Item | Model | Quantity | Notes |
| --- | --- | --- | --- |
| Controller | DOIT Cove-610 (Hi3516CV610) | 1 | |
| Serial bus servo | Feetech HD1910 | 15 | IDs in [README](../README.md#hardware) |
| IMU board | Custom; below | 1 | Mounted on torso, connected to servo bus; also carries battery connector and power switch |
| CV610 adapter board | Custom; below | 1 | Plugs into Cove-610 and converts UART1 to the half-duplex servo bus |
| Battery | 2S lithium, 7.4 V nominal, 8.4 V fully charged | 1 | Connects to IMU board; powers servo bus through switch |
| Servo cables | 3-pin bus-servo cables | Per frame documentation | Both custom boards use A2007WR-3P, 2.0 mm 3-pin sockets, matching the servos |
| Frame | 3D-printed parts, bearings, and screws | 1 set | See Frame below |

## Optional peripherals

Associated features are disabled when the peripherals are absent.

| Item | Model | Quantity | Purpose |
| --- | --- | --- | --- |
| Camera | GC4023, MIPI | 1 | Browser video and gestures |
| Microphone | Capsule connected to Cove-610 microphone input | 1 | Wake word, command words, Xiaozhi |
| Speaker | Small speaker connected to Cove-610 speaker output | 1 | Duck sounds and Xiaozhi |
| ToF sensor | VL53L5CX module | 1 | Distance map; connects to CV610 adapter board |

## Tools

| Item | Purpose |
| --- | --- |
| USB-to-servo-bus adapter, 1 Mbps | Set new servo IDs using `tools/servo_tool.py` |
| USB-to-UART adapter, 3.3 V logic | Flash the IMU board through its programming test points |

## IMU board

Schematic: [imu_board.pdf](imu_board.pdf), DoitDuck IMU v1.0.

| Reference | Part | Quantity | Notes |
| --- | --- | --- | --- |
| U1 | ESP32-C2 (ESP8684) | 1 | No external flash in schematic; select a variant with integrated flash |
| U2 | LSM6DSV16XTR | 1 | SPI IMU |
| U4 | SN74LVC2G125DCUR | 1 | Bus transceiver |
| U9 | BL1117-33CX | 1 | 8.4 V to 3.3 V |
| X2 | 26 MHz crystal | 1 | |
| D1 | PESD5V0S1BA | 1 | Bus ESD protection |
| CN1, CN3, CN4 | A2007WR-3P | 3 | Servo-bus sockets matching servo connectors |
| CN2 | ZX-XH2.54-2PWZ | 1 | XH2.54 2-pin battery connector |
| DC-44A | DC socket | 1 | Labeled as charging input in schematic; parallel with battery connector |
| U8 | SS12D10G6-WJ | 1 | Switch powers this board and servo bus (DXL_VDD) from battery |
| LED1 | LED | 1 | Test indicator on GPIO18 |
| LED2 | LED | 1 | 3.3 V power indicator |

No programming connector or buttons are fitted. Programming test points: VCC, GND, TX, RX, RESET, BOOT. R10 is a reserved bus-pull-up footprint.

## CV610 adapter board

Schematic: [cv610_board.pdf](cv610_board.pdf), DoitDuck CV610 v1.0.

| Reference | Part | Quantity | Notes |
| --- | --- | --- | --- |
| H3 | PM254-2-12-Z-8.5 | 1 | 2×12 female header for Cove-610 |
| U4 | SN74LVC2G125DCUR | 1 | Bus transceiver; 1OE# connects to DXL_TX_EN, active low for transmission |
| U1 | TPS54302DDCR | 1 | Steps DXL_VDD down to 3.3 V, connected to header's 3.3 V pin |
| L2 | 6.8 µH inductor | 1 | Buck converter inductor |
| CN4 | A2007WR-3P | 1 | Servo-bus connector matching servos |
| CN2 | LAIL-PHC2.0-4P | 1 | ToF connector: 3.3 V, GND, TOF_SDA, TOF_SCL |

R8 and R9, the TPS54302 enable-divider footprints, are not populated; the schematic specifies leaving them open.

## Frame

Use the official Microduck frame, such as the printed parts in [kissqy/Microduck-kissqy](https://github.com/kissqy/Microduck-kissqy). Refer to that project's `DUCK-SHOPPING-LIST.md` for print files and bearing/screw quantities.
