# DoitDuck Quick Start

This guide covers flashing the DoitDuck image, network setup, calibration, and control. For building and development, see the [README](../README.md).

## Safety

- A 2S lithium battery powers the servos directly. The switch is on the IMU board. Servos may move suddenly when torque is enabled; keep fingers clear of joints.
- Support or suspend the robot when first enabling torque, calibrating its stance, or trying new motions. Use a flat, open surface.
- Click Emergency stop at the upper right, or press Space on a computer, to release servo torque immediately.
- Below 6.6 V the robot warns and refuses to enable torque. Charge the battery; avoid over-discharge.

## Requirements

| Item | Details |
| --- | --- |
| Assembled robot | 15 servos, IMU board, CV610 adapter board; IDs and wiring in the [README](../README.md#hardware) |
| 2S lithium battery | Fully charged, connected to the IMU board |
| Linux or Windows computer | For flashing |
| USB cable | Connects Cove-610 to the computer |
| Phone or desktop browser | Robot control |

The camera, microphone, speaker, and ToF sensor are optional. Corresponding features are disabled if the required peripherals are absent.

## Flash the image

1. Extract `doitduck-image-<version>.zip` to `doitduck-image-<version>/`.
2. Flash its `nand_burn_table.xml` with burntool-cli:

```sh
./burntool-cli -u -x doitduck-image-<version>/nand_burn_table.xml
```

Hold UPDATE on Cove-610, then power-cycle or press RST. Release UPDATE when the tool reports that Boot is running (the original tool message is `Boot 已运行`).

See [V610-Solution flashing instructions](https://github.com/doit-open-org/V610-Solution/blob/main/docs/burntool.md) for tool downloads, drivers, and serial flashing.

DoitDuck starts automatically after flashing. The image sets SDIO1 pins to `none` for the servo bus, so LCD and SD-card functions are unavailable. The source repository itself does not include the complete image archive described above.

## First startup

### 1. Connect to the network

**Option A: spoken setup, with a microphone**

1. Say **开始配网** (*kai shi pei wang*, "Start network setup"). The robot opens an unsecured `DoitDuck-xxxx` hotspot. The supplied model recognizes the Chinese phrase, not its English translation.
2. Connect your phone. If the setup page does not open automatically, visit `http://192.168.4.1/`.
3. Select your Wi-Fi, enter its password, and connect. The page shows progress; correct the password and retry if needed.
4. Note the robot's IP shown in the dialog. Open `http://<IP>/` for subsequent control.

The robot reconnects to this Wi-Fi at future startups.

**Option B: USB setup, without a microphone**

1. Connect Cove-610 to your computer over USB.
2. Set the newly detected USB network adapter's IP to `192.168.137.1`, mask `255.255.255.0`.
3. Open `http://192.168.137.2/`, then connect to your Wi-Fi under Settings → Wi-Fi.

### 2. Calibrate

Calibrate after assembly or servo replacement. Open Maintenance → Calibration wizard:

1. Servo check: confirm all 15 servos and the IMU are online.
2. IMU bias: keep the robot still for two seconds.
3. Zero positions: straighten both legs and face the head forward. With torque off, position the joints by hand and capture all offsets, or enable torque and adjust each joint with its slider.
4. Standing posture: place the robot upright, apply the suggested fine adjustments, then set the current orientation as level.

Calibration persists across restarts. Maintenance offers export, import, and undo.

### 3. Start walking

On Control, select Walk mode:

- Left joystick: forward, backward, and sideways movement. Right joystick: turning.
- Skill buttons: ground picking and somersaults.
- Keyboard: W/S forward/backward, A/D sideways, Q/E turn, Space emergency stop.

The documented firmware supports automatic recovery after a fall. Walking, recovery, and skills require their respective motion models to be installed.

## Control interface

Open `http://<IP>/` on a phone or computer. Use the top-left language button for English/Chinese. Light/dark appearance follows the system.

| Page | Contents |
| --- | --- |
| Control | Camera, modes, dual joysticks, skills, voice-command bar |
| Actions | Import, verify, delete policies |
| Status | Attitude, servo angles and online state, battery, IMU, ToF distance map |
| Maintenance | Calibration, servo scan/ID/gain tools, calibration export/import |
| Settings | Wi-Fi, smoothing, volume/sounds, Xiaozhi, command and gesture lists, version/licenses |

| Mode | Behavior |
| --- | --- |
| Torque off | Joints can be moved by hand |
| Inference only | Runs the policy without driving servos |
| Stand | Holds the default stance |
| Walk | Joystick or command-driven walking |

## Voice and Xiaozhi

Requires microphone, speaker, and network access.

### First activation

1. Open Settings → Xiaozhi to obtain the six-digit activation code.
2. Add a device in the [xiaozhi.me](https://xiaozhi.me) console and enter that code.

### Conversation

Say **你好小鸭** (*ni hao xiao ya*, "Hello, little duck") to wake the robot, then talk to Xiaozhi. The language model can command walking, turning, skills, head, and mouth movements; examples in the original guide include asking it to walk forward, perform a somersault, or look up.

### Commands

The supplied always-listening command is **开始配网** ("Start network setup"), which opens the provisioning hotspot without a wake phrase.

The browser command bar also offers stop, relax, stand, forward/backward/sideways walking, turns, spin, ground pick, somersault, head directions, and mouth open/close. See Settings for the complete list. Browser command labels do not imply separate spoken-recognition models for every command.

## Gestures

Requires a camera.

| Gesture | Response |
| --- | --- |
| Thumbs up | Nod |
| Open palm | Open mouth |
| Fist | Close mouth |

## Sounds

The robot makes sounds for wake-up, accepted/rejected commands, falls, low battery, and emergency stops. After two idle minutes while standing, it also calls every one or two minutes. Settings provides volume control, disabling idle calls, and uploaded replacement recordings in 16 kHz mono WAV format.

## Hotspot

A hotspot can run while the robot remains connected to Wi-Fi. Select Settings → Wi-Fi → hotspot setup, or say the Chinese setup phrase.

- With no activity, the hotspot closes after ten minutes when the robot is connected to Wi-Fi.
- The always-on option keeps it open and starts it at boot. Away from a router, connect to `DoitDuck-xxxx` and open `http://192.168.4.1/` to control the robot.

## Motion models

Walking, recovery, ground picking, and somersaults use motion policies. To import one:

1. On Actions, select Import and choose `policy.onnx` and a Microduck schema-2 `manifest.json`.
2. Imported policies start as unverified and can only be triggered from the browser. First release torque, use Inference only, and inspect joint-target curves for smooth behavior.
3. After checking, mark the policy verified to make it available to voice and Xiaozhi.
4. Release torque before replacing walking or recovery policies.

Check third-party model licenses before use.

## Log in to the board

For logs or manual maintenance:

```sh
ssh root@<IP>
```

The upstream guide specifies user `root` and default password `123456`. Change it with `passwd` after login; defaults may differ on a supplier-customized image.

The application is in `/root/doitduck/`. Offsets, IMU level, imported policies, Xiaozhi device ID, and other user data are in `/root/doitduck/data/`.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| Low-battery warning; torque cannot be enabled | Charge the battery; threshold is 6.6 V |
| Servo or IMU offline | Check cables, servo IDs, battery supply, and Maintenance → servo bus scan |
| No automatic hotspot page | Open `http://192.168.4.1/` |
| Robot IP unknown | Use USB at `http://192.168.137.2/`, or check the router's device list |
| No wake response | Check microphone; Xiaozhi also needs network access and activation |
| No camera view or gestures | Check camera connection |
| Robot leans when standing | Repeat standing-posture and level calibration |

## License

DoitDuck uses the [PolyForm Noncommercial License 1.0.0](../LICENSE.md); commercial use requires separate authorization. See [THIRD_PARTY.md](../THIRD_PARTY.md) for third-party licenses.
