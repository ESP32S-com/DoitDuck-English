# DoitDuck IMU Board

ESP32-C2 + LSM6DSV16XTR, mounted on the torso and connected to the servo bus as device ID 200. Protocol and registers are listed below.

All code is in [main/main.c](main/main.c).

| Task | Priority | Function |
| --- | --- | --- |
| `bus_task` | Highest | Parses frames; replies only to PING/READ/WRITE for ID 200 |
| `sample_task` | High | Reads FIFO every 2 ms: acceleration/angular velocity at 480 Hz, SFLP bias/gravity at 120 Hz; decodes sensor XYZ, subtracts bias, applies mounting-angle rotation, averages over 10 ms, and performs calibration |
| `app_main` | Low | NVS writes, UART0 logs, and keyboard commands |

## Wiring

The [schematic](../../hardware/imu_board.pdf) is authoritative.

| ESP32-C2 | Net | Function |
| --- | --- | --- |
| GPIO0 | DXL_TX_EN | SN74LVC2G125 1OE#: low to transmit, high to release bus |
| GPIO1 | DXL_TX | UART1 TX, 1 Mbps |
| GPIO2 | DXL_RX | UART1 RX; receive buffer always enabled |
| GPIO3 | IMU_INT1 | Reserved; current implementation polls FIFO |
| GPIO4 | IMU_MISO | SPI MISO |
| GPIO5 | IMU_MOSI | SPI MOSI |
| GPIO6 | IMU_SCK | SPI clock, 10 MHz, mode 3 |
| GPIO7 | IMU_CS | Active-low SPI chip select; R5 10 kΩ pull-up |
| GPIO18 | LED | Test indicator, currently unused |
| GPIO19 / GPIO20 | U0RXD / U0TXD | UART0 programming, logs, calibration; boot-log baud rate, 74880 with 26 MHz crystal |
| GPIO9 | BOOT | Download-mode boot, exposed at BOOT test point |

LSM6DSV16XTR supports SPI and I2C; this firmware uses the schematic's four-wire SPI. UART1 uses `UART_MODE_RS485_HALF_DUPLEX` for RTS control. That mode assumes active-high DE, whereas 1OE# is active low, so `UART_SIGNAL_RTS_INV` inverts RTS: GPIO0 goes low during transmission and high after the last stop bit. The physical interface remains single-wire TTL; the buffer replaces the previous direct open-drain connection.

The 2S battery (8.4 V fully charged) connects to CN2. Switch U8 powers the board and servo bus (DXL_VDD). BL1117-33 produces 3.3 V; ground is shared with the bus. There is no battery-voltage divider, so register `0x5C` always returns 0 (unavailable).

The board does not swap axes or invert individual axes: XYZ matches the sensor. The main controller handles installation-axis mapping. The existing protocol's overall negative sign for acceleration and gravity is retained, along with bias and mounting-tilt calibration. Tilt calibration rotates about sensor axes.

## Build and flash

Requires ESP-IDF with ESP32-C2 support. `sdkconfig.defaults` specifies the board's 26 MHz crystal and 74880-baud serial logging.

```sh
idf.py set-target esp32c2
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

There are no programming connectors or buttons. Connect a 3.3 V USB-to-UART adapter to TX, RX, and GND test points. To enter download mode, connect BOOT to GND and briefly short RESET to GND. After flashing, esptool's automatic reset does not start this board normally: no logs or bus replies appear. Disconnect BOOT, then reset again or power-cycle. ESP32-C2 has no USB Serial/JTAG console; logs and commands use UART0.

## Operation

One log line per second reports acceleration (mg), bias-corrected angular velocity (°/s), status, and battery voltage. Status bits: bit0 data valid; bit1 bias valid; bit2 mounting tilt calibrated; bit3 IMU fault.

At startup, keep the board still for two seconds for automatic bias measurement. Subsequently, SFLP bias estimates are checked each minute and saved to NVS when the change exceeds 0.05 °/s.

Use Maintenance → Calibration wizard on the controller, or serial-monitor keys:

| Key | Command | Requirement |
| --- | --- | --- |
| `1` | Measure bias | Stationary for two seconds in any orientation |
| `2` | Set current orientation as level | Level body, stationary for two seconds; fails if tilt exceeds 15° |
| `3` | Clear bias and mounting tilt | — |

Calibration is stored on the IMU board, so replacing the main controller does not require recalibration.

## Bus protocol

Feetech SCS-compatible frames, little-endian, 1 Mbps. The controller initiates requests; the board only responds.

```text
Request: FF FF ID LEN INST PARAM… CHK      LEN = parameter count + 2
Reply:   FF FF ID LEN ERR  DATA…  CHK      LEN = data count + 2
CHK = ~(ID + LEN + INST/ERR + sum(parameters/data)) & 0xFF
```

| Instruction | Code | Description |
| --- | --- | --- |
| PING | 0x01 | Empty response for online detection |
| READ | 0x02 | Parameters: start address, length (1–32) |
| WRITE | 0x03 | Accepts only `0x60` (1 byte) and `0x62` (4 bytes) |

Broadcast ID 0xFE, sync read 0x82, and sync write 0x83 target servos and are ignored. Invalid addresses/lengths return ERR = 0x08.

| Address | Bytes | Contents | Access |
| --- | --- | --- | --- |
| 0x00 | 2 | Model `0xD0 0x0D` | R |
| 0x02 | 1 | Firmware version | R |
| 0x05 | 1 | Fixed ID 200 | R |
| 0x40 | 6 | Acceleration x/y/z, int16, mg | R |
| 0x46 | 6 | Bias-corrected angular velocity x/y/z, int16, 0.1 °/s | R |
| 0x4C | 2 | Sample counter, uint16, increments per new sample | R |
| 0x4E | 1 | bit0 data valid, bit1 bias valid, bit2 mounting tilt calibrated, bit3 IMU fault | R |
| 0x50 | 6 | SFLP gravity vector, int16, 1/16384 g | R |
| 0x56 | 6 | Current bias, int16, 0.01 °/s | R |
| 0x5C | 2 | Battery voltage, uint16, mV; fixed 0 because board has no divider | R |
| 0x60 | 1 | Calibration: 1 bias, 2 level, 3 clear; reads as 0 | W |
| 0x61 | 1 | Result: 0 idle, 1 running, 2 success, 3 failed (not stationary) | R |
| 0x62 | 4 | Mounting roll/pitch, int16, 0.01°, persistent | R/W |

The controller reads 15 bytes starting at `0x40` every 10 ms and 16 bytes starting at `0x56` once per second, with an 8 ms response timeout. An unchanged sample counter or cleared status bit0 indicates invalid IMU data; 100 ms of invalid data causes torque release.
