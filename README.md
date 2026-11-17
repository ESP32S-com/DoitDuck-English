[# DoitDuck](https://esp32s.com/product/biped-robot-duck-esp32-p4-hi3516cv610/)

[DoitDuck](https://esp32s.com/product/biped-robot-duck-esp32-p4-hi3516cv610/) is a recreation of the Feetech HD1910 servo version of the [Microduck](https://github.com/pollen-robotics/microduck) biped robot duck. It retains that version's mechanical structure, servos, and motion-model format, replaces the controller with DOIT's Cove-610 (Hi3516CV610), and adds browser control, voice conversation, and vision features.

- 15 bus servos and a bus-connected IMU; 100 Hz feedback and 50 Hz reinforcement-learning policy inference.
- Walking, automatic recovery after a fall, and skills such as ground picking and somersaults.
- Phone and desktop browser control: dual joysticks, skill buttons, camera view, status monitoring, calibration wizard, and Wi-Fi settings, in English and Chinese.
- Say the Chinese wake phrase **你好小鸭** (*ni hao xiao ya*, "Hello, little duck") to talk to Xiaozhi; the language model can command robot actions.
- Camera gestures, optional ToF ranging, and synthesized duck sounds.

> English documentation edition. Original project: [doit-open-org/DoitDuck](https://github.com/doit-open-org/DoitDuck). Documentation translation does not change firmware capabilities or convert the supplied Chinese speech-recognition models to English. Original authorship and licenses are retained.

| I want to... | Start here |
| --- | --- |
| Flash an image, connect Wi-Fi, and start using the robot | [Quick start](docs/quickstart.md) |
| Obtain parts, make the boards, and assemble the robot | [Bill of materials](hardware/BOM.md) and Hardware below |
| Build, deploy, or modify the software | Build and Deployment below; [software design](docs/doitduck-design.md) |
| Train custom motions | [Training motion models](docs/training.md) |

## Relationship to Microduck

| Area | Retained from Microduck | DoitDuck changes |
| --- | --- | --- |
| Mechanics | HD1910-version printed frame, e.g. [Microduck-kissqy](https://github.com/kissqy/Microduck-kissqy), and the arrangement of 15 Feetech HD1910 servos | None |
| Motion models | MLP 61 → 512 → 256 → 128 → 14; ONNX plus schema-2 `manifest.json`; compatible Microduck models can be imported | Converted to float32 weights for controller inference |
| Sounds | Synthesis algorithm from Microduck's `sounds` | Ported from Rust to Python at 16 kHz; idle sounds and uploaded recordings added |
| Controller and electronics | — | Cove-610, custom ESP32-C2 + LSM6DSV16X IMU/power board, CV610 adapter board |
| Interaction | — | Browser control, calibration wizard, hotspot setup, wake word and Xiaozhi, gestures, ToF |

Thanks to the Microduck authors for sharing their designs and models.

## Hardware

See [hardware/BOM.md](hardware/BOM.md).

| Part | Description |
| --- | --- |
| Controller | Cove-610 (Hi3516CV610); UART1 servo bus; U-Boot setting `sdio1_pins=none` |
| CV610 adapter board | Plugs into Cove-610; half-duplex 1 Mbps servo bus, transmit enable on GPIO1_0; servo-bus supply stepped down to 3.3 V; ToF connector. [Schematic](hardware/cv610_board.pdf) |
| IMU board | ESP32-C2 + LSM6DSV16X, bus ID 200; battery connector, power switch, and three servo connectors. [Firmware](firmware/imu_board), [schematic](hardware/imu_board.pdf) |
| Servos | 15; IDs below |
| Optional peripherals | Camera for browser video and gestures; microphone and speaker for wake word, Xiaozhi, and sounds; VL53L5CX ToF connected to the adapter board, I2C1: SDA GPIO1_5, SCL GPIO1_4 |

| ID | Joint |
| --- | --- |
| 10–14 | Right leg: hip yaw, hip roll, hip pitch, knee, ankle |
| 20–24 | Left leg, same order |
| 30–33 | Neck pitch, head pitch, head yaw, head roll |
| 34 | Mouth |

New servos can be configured in batches using the Windows utility [tools/servo_tool.py](tools/servo_tool.py) and a USB-to-servo-bus adapter at 1 Mbps. Build an executable with `pyinstaller -F -w tools/servo_tool.py`.

- Change IDs in a batch: connect servos sharing one ID to the same bus, then enter the old and new IDs.
- Move to zero: move to 2048 and hold torque while installing the horn; release torque afterward.
- Set current position as zero: assign the current position the value 2048.
- Position control: move the slider for the ID in the zero-position section, over 0–4095; release torque afterward.

Bus scanning gives an exact count only when IDs are unique. Replies collide when IDs are duplicated, so the utility reports only "at least N". After changing a shared ID, it can confirm only that the old ID no longer replies and the new ID does.

## Robot configuration

`robots/microduck.json` defines servos, home posture, gains, policies, voice commands, and sounds for the official Microduck frame, such as Microduck-kissqy printed parts with Feetech HD1910 servos. Motion files are `microduck_walk.bin`, `microduck_ground_pick.bin`, and `microduck_roulade.bin`.

## Build

### PC, without a board

Test the browser UI and commands, without servos or audio/video:

```sh
cmake -S firmware/main_board/pc -B build/pc && cmake --build build/pc
cd build/pc && ./doitduck          # Open http://localhost:9527
```

### Cove-610

Requires V610-App-SDK. From the V610-Apps root:

```sh
source <V610-App-SDK>/export.sh
sh tools/build.sh projects/robot/doitduck/firmware/main_board
```

Output in `firmware/main_board/out/` contains `doitduck`, `robots/`, `web/`, `sounds/`, and an empty `models/` directory. Feature switches and board parameters are in [Kconfig](firmware/main_board/Kconfig), with defaults in [sdkconfig.defaults](firmware/main_board/sdkconfig.defaults).

## Deployment

```sh
ssh root@<IP> '/etc/init.d/S40app stop'
tar cf - -C firmware/main_board/out . | ssh root@<IP> 'mkdir -p /root/doitduck && tar xf - -C /root/doitduck'
scp -r models root@<IP>:/root/doitduck/            # Models: see below
```

- Autostart: set `APP_PATH` in `/etc/init.d/S40app` to `/root/doitduck/doitduck`.
- Wi-Fi at boot: copy [tools/board/S52wifi](tools/board/S52wifi) to `/etc/init.d/` and run `chmod +x` on it.
- Hotspot provisioning requires AP support in `wpa_supplicant`: Buildroot option `BR2_PACKAGE_WPA_SUPPLICANT_AP_SUPPORT`.
- Zero offsets, IMU level calibration, imported policies, Xiaozhi device ID, and other user data are stored in `/root/doitduck/data/`; application redeployment does not overwrite them.

To build a flashable image containing DoitDuck, first build V610-Buildroot, then run `sh tools/pack_image.sh <V610-Buildroot-path>` from this directory. Flash files appear in `build/image/flash/`, with autostart and Wi-Fi setup installed.

## First use

For a manual deployment, install motion models first (see below). The ready-to-flash image described by the upstream guide includes them. See [Quick start](docs/quickstart.md) for networking, calibration, and walking. This source archive is not a ready-to-flash firmware image.

## Models

The supplied voice and gesture models in `models/` run only on Hi3516CV610. **Motion models are not included in the repository**; add them to `models/` yourself. Copy the whole directory to `/root/doitduck/models/`. A missing model disables its associated feature without stopping the others.

| File | Purpose | Source |
| --- | --- | --- |
| Motion files listed in Robot configuration | Walking, recovery, skills | [Train your own](docs/training.md), or convert compatible Microduck models |
| `nihaoxiaoya.mipk` | Chinese wake phrase 你好小鸭 ("Hello, little duck") | Included, project-trained MicroWakeWord model |
| `kaishipeiwang.mipk` | Chinese command 开始配网 ("Start network setup") | Included, project-trained MicroWakeWord model |
| `mediapipe_hand.mipk` | Gestures | Included MediaPipe hand model, Apache-2.0 |

A motion model consists of ONNX (MLP 61 → 512 → 256 → 128 → 14) and a Microduck schema-2 `manifest.json`. Convert to runtime `.bin` weights using the Python standard library:

```sh
python3 tools/import_policy.py policy.onnx manifest.json --out models/microduck_walk.bin
python3 tools/import_policy.py policy.onnx manifest.json --host <IP>
```

Alternatively, import through the browser's Actions page. Imported policies start as unverified and can only be triggered from the browser. After verification there, voice, Xiaozhi, and gamepad sources may use them. Release torque before replacing walking or recovery policies. The manifest's `robot.model` must be in `policy.compatible_models`.

Check the license before using third-party weights. For example, Microduck-kissqy's self-trained models have a personal noncommercial license.

## Browser interface

Open `http://<IP>/` on a phone or computer. English/Chinese switching is at the top left; light/dark appearance follows the system.

| Page | Contents |
| --- | --- |
| Control | Camera, modes (torque off, inference only, stand, walk), dual joysticks, skills, voice-command bar |
| Actions | Import, verify, delete policies |
| Status | Attitude, servo positions and online status, IMU, ToF map, loop rates |
| Maintenance | Calibration wizard, servo scan/ID/gain tools, calibration export/import/undo |
| Settings | Wi-Fi, smoothing, volume and sounds, Xiaozhi, command/gesture lists, about and licenses |

## Voice and Xiaozhi

Say **你好小鸭** (*ni hao xiao ya*) to wake Xiaozhi. MCP tools let the language model command walking, turning, skills, head, and mouth movements. For first activation, find the six-digit code under Settings → Xiaozhi, add a device in the [xiaozhi.me](https://xiaozhi.me) console, and enter the code.

### Command words

Entries in `voice.commands` that have a `model` execute without a wake phrase. Each has a MicroWakeWord model at `models/<model>.mipk`; `voice.wake_model` selects the wake model. Commands without a `model` are used only in the browser command bar. To add spoken commands, train a mipk model, place it in `models/`, and add an entry:

```json
{ "words": "开始配网", "model": "kaishipeiwang", "call": "wifi.ap", "params": { "on": true } }
```

The Chinese text above is the actual trained command ("Start network setup"), not an English voice command.

### Hotspot setup

Say **开始配网** (*kai shi pei wang*) or select Settings → Wi-Fi → hotspot setup. The open hotspot `DoitDuck-xxxx` does not interrupt the existing Wi-Fi connection.

1. Connect your phone to the hotspot. If the setup page does not open automatically, visit `http://192.168.4.1/`.
2. Select a network, enter its password, and connect; status updates appear on the page.
3. A dialog shows the new IP. Confirm to close the hotspot. An incorrect password leaves the hotspot open for another attempt.

When the robot is online, an unused hotspot closes after ten minutes. Enable the always-on option to keep it open and start it at boot, allowing direct phone control without a router.

## Gestures, ToF, and sounds

- Thumbs up: nod; open palm: open mouth; fist: close mouth. Mappings are in `gesture` in the robot configuration.
- VL53L5CX is detected at startup; Status shows an 8×8 distance map. If absent, startup logs report it once and disable ToF.
- `tools/gen_sounds.py` synthesizes `sounds/`; `--seed <number>` changes the voice. Event mappings and idle-call timing are in `sounds` in the robot configuration. The UI supports uploaded 16 kHz mono WAV recordings, volume adjustment, and disabling idle calls.

## Configuration reference

### Kconfig

| Option | Default | Description |
| --- | --- | --- |
| `DOITDUCK_WEB` / `VOICE` / `GESTURE` / `WIFI` / `TOF` | On | Feature switches |
| `DOITDUCK_XIAOZHI` | On | Requires mipk wake-word recognition and networking |
| `DOITDUCK_GAMEPAD` | Off | Input-reading implementation not yet available |
| `DOITDUCK_ROBOT` | `microduck` | Robot configuration |
| `DOITDUCK_HTTP_PORT` | 80 | Web port |
| `DOITDUCK_BUS_DEVICE` | `/dev/ttyAMA1` | Servo-bus UART |
| `DOITDUCK_BUS_PINMUX` | — | Startup pinmux writes, space-separated `register=value` entries |
| `DOITDUCK_BUS_TX_ENABLE_BASE` / `PIN` | GPIO1_0 | Transmit enable; `PIN=-1` for automatic-direction transceivers |
| `DOITDUCK_TOF_I2C` | `/dev/i2c-1` | ToF I2C device |
| `DOITDUCK_MIC_CHANNEL` / `MIC_GAIN_DB` / `SPEAKER_GAIN_DB` | 0 / 50 / -10 | Microphone channel, microphone gain, speaker gain |

### Robot configuration (`robots/*.json`)

| Field | Description |
| --- | --- |
| `motors.ids` / `names` / `home_deg` | IDs, labels, home posture in degrees |
| `motors.kp` / `kd` / `mouth_kp` / `kp_by_id` | Gains; per-servo P override, e.g. `{ "30": 12 }` |
| `imu.axis_map` / `axis_sign` | Mapping from IMU board axes to body axes |
| `policy.policies` | Files, names, kinds, durations; `doitduck` may define labels, smoothing `alpha`, `mouth_deg` / `mouth_until`, `hold_recovery`, and policy-specific `home_deg` |
| `policy.smoothing` / `smoothing_head` | Default leg/head smoothing, adjustable in UI |
| `policy.fall` / `stand` | Fall and stable-standing detection |
| `limits` | Maximum speeds by control source |
| `voice` / `gesture` / `gamepad` / `head_presets` | Commands, gestures, buttons, head presets |
| `sounds` | Event and idle sounds |
| `xiaozhi` | Enable flag and OTA URL |

## IMU firmware

```sh
cd firmware/imu_board
idf.py set-target esp32c2 && idf.py build && idf.py -p <serial-port> flash monitor
```

See [IMU board documentation](firmware/imu_board/README.md) for wiring, crystal frequency, calibration, and the bus protocol.

## Directory layout

```text
firmware/main_board/   Main controller
├── main/core/         Bus, arbitration, motion, policy inference
├── main/app/          Startup, API, configuration, policies, calibration, events, sounds
├── main/features/     Independently selectable web/gamepad/voice/xiaozhi/gesture/tof/wifi
├── components/        Platform adapters and third-party libraries
└── pc/                PC build
firmware/imu_board/    ESP-IDF IMU firmware
robots/               Robot configurations
web/                  Browser UI
sounds/               Duck sounds
tools/                Import, sound synthesis, image packaging
hardware/             Schematics and BOM
docs/                 Documentation
```

See [Software design](docs/doitduck-design.md), [Training](docs/training.md), and [Third-party components](THIRD_PARTY.md).

## License

This project uses the [PolyForm Noncommercial License 1.0.0](LICENSE.md). **Commercial use is prohibited under this license.** Modification and sharing are permitted subject to the license; include `LICENSE.md` when sharing. Commercial use requires separate authorization.

Third-party components follow their own licenses; see [THIRD_PARTY.md](THIRD_PARTY.md).
