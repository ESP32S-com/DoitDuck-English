# DoitDuck Software Design

Structure and mechanisms of the main-controller software. See the [IMU documentation](../firmware/imu_board/README.md) for its bus protocol and the [README](../README.md) for usage.

## 1. Design choices

PC and board builds share application code. The PC build does not require the board SDK and can test the UI, commands, arbitration, MCP, and policy import without hardware.

Web, gamepad, voice, Xiaozhi, gesture, ToF, and Wi-Fi are independent optional features. Unsupported capabilities are omitted at build time. Speech recognition, networking, codecs, storage, and media use adapters selected through Kconfig.

Each motion consists of weights and a manifest supplied by the user; motion weights are not included in the repository. Servos, home posture, gains, policies, commands, and sounds are defined in `robots/<name>.json`.

## 2. Layers

```text
Browser / Gamepad / Microphone / Camera / Xiaozhi server
                         |
features/  web gamepad voice xiaozhi gesture tof wifi
                         |
app/       Startup, API, configuration, policies, calibration, events, sounds
                         |
core/      Bus, arbitration, motion, policy inference
                         |
components/ Interfaces and platform implementations (V610 or PC stubs)
```

Dependencies point downward: `features → app → core`, `features → components`, and `core → components/{osal,bus,storage}`. Features do not include each other's headers; they interact through commands and events.

## 3. Directory layout

Application code is in `main/`. Each adapter or third-party library is a component; `REQUIRES` declares dependencies. Unused components are not compiled.

```text
firmware/main_board/
├── CMakeLists.txt  Kconfig  sdkconfig.defaults
├── main/
│   ├── main.c           Calls only app_run()
│   ├── core/            duck.h bus.c motion.c arbiter.c policy.c
│   ├── app/             app.c api.c config.c policies.c calib.c sound.c feature.h event.h
│   └── features/        web gamepad voice xiaozhi gesture tof wifi
├── components/
│   ├── osal/            osal.h linux/
│   ├── bus/             linux_uart.c none.c
│   ├── storage/         posix.c
│   ├── media/           audio.h camera.h hand.h v610_mpp.c *_none.c
│   ├── asr/             mipk_wake.c none.c
│   ├── net/ opus/ httpd/ pad/
│   └── cjson/ mbedtls/ vl53l5cx/
└── pc/                  PC CMakeLists; shared main and components
```

Core code does not use malloc, file, or thread APIs. Board settings (UART, pinmux, direction GPIO, microphone/speaker gains, web port) are selected at compile time under Kconfig's `DoitDuck board` menu. The program takes no command-line arguments.

## 4. Platform adapters

### 4.1 Interfaces

| Interface | Main functions | Users |
| --- | --- | --- |
| `osal.h` | `os_now`, `os_sleep_until`, `os_thread` (optional CPU affinity), `os_lock`, `os_device_id` | All |
| `bus.h` | `bus_open`, `bus_write`, `bus_read`, `bus_flush_input` | core |
| `storage.h` | `store_blob` (read-only files), `store_kv_get/set` (small settings), `store_put/get/remove/list` (user files) | core, app |
| `audio.h` | `audio_read`, `audio_play`, `audio_stream`, `audio_flush` | voice, xiaozhi, sounds |
| `camera.h` | `camera_h264_read`, `camera_key_frame`, `camera_jpeg`, `camera_lines` | web, gesture, xiaozhi |
| `hand.h` | `hand_read`, 21 landmarks | gesture |
| `asr.h` | `asr_open`, `asr_caps`, `asr_feed`, `asr_touch`; text input/output | voice |
| `net.h` | `net_https_post`, `net_ws_connect/send_text/send_binary/close` | xiaozhi |
| `opus_codec.h` | `opus_codec_encode`, `opus_codec_decode` | xiaozhi |
| `httpd.h` | `httpd_start(port, routes)`, `httpd_ws_send/recv/close` | web |
| `pad.h` | `pad_read`, joystick/buttons | gamepad |

V610 audio, camera, and gesture share an MPP context in `components/media/v610_mpp.c`; their headers and `none.c` implementations remain separate.

### 4.2 Implementation selection

Each interface has a Kconfig `choice`, with defaults in `sdkconfig.defaults`; only the selected implementation compiles. Unsupported capabilities use `none`, returning -1 so dependent features do not start.

| Interface | V610 | PC |
| --- | --- | --- |
| osal / storage / httpd | linux / posix / posix | Same |
| bus | linux_uart, including direction GPIO | none |
| audio / camera / hand | v610_mpp | none |
| asr | mipk_wake, one MicroWakeWord model per wake/command phrase | none |
| net / opus | mbedtls / libopus | mbedtls / none |
| pad | none | none |

## 5. Features

```c
struct feature {
    const char *name;                       /* robots/*.json section and robot.status field */
    int  (*configure)(const cJSON *section); /* Nonzero: disabled or missing capability */
    void (*start)(void);                    /* Optional: start own thread */
    void (*status)(cJSON *r);               /* Optional: own robot.status field */
    void (*on_event)(int ev, const char *text); /* Optional */
    void (*on_audio)(const int16_t *pcm, int n); /* Optional microphone PCM */
    const struct api_method *methods;       /* Optional commands */
    int nmethods;
};
```

- `app.c` defines `FEATURES[]` conditionally through Kconfig. Startup calls `configure`, then `start`.
- App reads microphone audio centrally. It suppresses delivery during duck sounds and for 0.15 seconds afterward to avoid self-triggering.
- `robot.info` lists compiled, successfully started features; the UI hides unavailable controls.

| Feature | Dependencies | Function |
| --- | --- | --- |
| web | httpd; optional camera | UI, WebSocket commands/status, H.264 video |
| gamepad | pad | Maps buttons to commands; input-reading backend not implemented |
| voice | audio, asr | Wake phrase; always-available commands with a `model`; browser command bar uses configured command table |
| xiaozhi | audio, net, opus; wake from voice | Server conversation and MCP motion control |
| gesture | camera, hand | Maps gestures to commands |
| tof | Linux I2C | VL53L5CX 8×8 ranging, thread pinned to CPU1 |
| wifi | wpa_supplicant | Scan/connect and hotspot setup |

`wifi.ap` uses wpa_supplicant's global control interface (`-g /var/run/wpa_global`, enabled by `tools/board/S52wifi`) to create `ap0` alongside `wlan0`: 2.4 GHz channel 6, no password, `192.168.4.1`. The existing wlan0 connection stays active. This requires Buildroot `BR2_PACKAGE_WPA_SUPPLICANT_AP_SUPPORT` and concurrent STA+AP support, available on AIC8800.

- The hotspot thread provides DHCP/DNS. DNS maps every hostname to `192.168.4.1`; HTTP requests arriving through that address with another Host are redirected with 302 to Settings, providing a captive portal.
- While the hotspot is active, `wifi.connect` first clears wlan0's IP. The UI waits for `COMPLETED` and a valid IP, shows a dialog, then sends `wifi.ap {on: false}` after confirmation.
- Without confirmation, the hotspot closes after ten inactive minutes if wlan0 is online. `{always: true}` is stored in `data/wifi_ap.txt` and enables the hotspot at boot.

Events are broadcast with `event_post(ev, text)`:

| Event | Source | Use |
| --- | --- | --- |
| `EV_WAKE` | voice | Wake sound; interrupt Xiaozhi speech |
| `EV_WAKE_IDLE` | voice | Start conversation after wake |
| `EV_POLICIES_CHANGED` | Policy library | Refresh skill buttons and MCP descriptions |
| `EV_COMMAND_OK` / `EV_COMMAND_REJECTED` | voice, gesture | Acceptance/rejection sounds |
| `EV_LISTEN_TIMEOUT` | voice, xiaozhi | End listening |
| `EV_SKILL_DONE` / `EV_FALL` / `EV_LOW_BATTERY` / `EV_ESTOP` | app, from robot state | Corresponding sounds |

## 6. Control core

- Bus thread: real-time priority 80, CPU0 with servo-UART interrupts; reads 15 servos and the IMU every 10 ms, filters IMU, and writes targets when torque is enabled.
- Motion thread: real-time priority 70; every 20 ms builds observations, runs policy, smooths outputs, and converts to servo targets.
- Modes: torque off, inference only, stand, walk, calibration, and servo setup.
- During walking, tilt beyond `policy.fall.tilt_deg` for `seconds` indicates a fall. The robot first holds its home posture for `hold_s`, then selects recovery. Tilt below `policy.stand.tilt_deg` for `seconds` returns to walking. Skills can disable detection with `hold_recovery`, e.g. somersaults.
- Startup and torque enable write temporary servo P/D registers: global `kp`/`kd`, `mouth_kp`, and per-ID `kp_by_id`.
- One source controls motion at a time: gamepad > web > voice > Xiaozhi > gesture. Higher priority can preempt. Gamepad/web maintain ownership with repeated commands; voice/Xiaozhi moves carry an expiry. Stop and emergency stop work from every source.
- Protection: release torque for nonnumeric policy output, no new motion target for 250 ms, or no IMU data for 100 ms. Reject torque enable below `low_battery_v`.

## 7. Motion policies

### 7.1 Network and observations

Supported network: MLP 61 → 512 → 256 → 128 → 14, ELU, 50 Hz, using the Microduck format. The index ranges below are half-open.

| Indices | Contents |
| --- | --- |
| 0..3 / 3..6 | Gyroscope and gravity direction in body coordinates |
| 6..20 / 20..34 / 34..48 | 14 joint offsets from home, velocities, previous actions; mouth excluded |
| 48..51 | Command slots interpreted by `command.encoding` |
| 51..55 | Neck pitch, head pitch, head yaw, head roll commands |
| 55..61 | Body-posture commands, always zero |

Outputs are joint angles relative to home in radians, scale 1. They are smoothed, then home positions are added. `policy.action_ids` determines servo mapping and left/right leg order.

### 7.2 Manifest

Each policy consists of weights plus a Microduck schema-2 manifest.

| `kind` | Termination | Examples |
| --- | --- | --- |
| `perpetual` | Runs until switched | Walk (`slot: walk`), recovery (`slot: stand`) |
| `episodic` | Ends after `duration_s` | Somersault, ground pick |
| `scripted` | Timed, interruptible | Sit/stand |

| `command.encoding` | Command slots | Examples |
| --- | --- | --- |
| `constant` | `command.idle`, all zero by default | Somersault |
| `phase` | `[cos 2πφ, sin 2πφ, 0]`, φ goes from 0 to 1 over `period_s`, ends at `end_phase` | Ground pick |
| `posture_flag` | vx slot: 1 sitting, 0 standing | Sit/stand with `ramp_s` |

DoitDuck extensions live under `doitduck`:

| Field | Meaning |
| --- | --- |
| `label` | Display name, `{"zh": ..., "en": ...}` |
| `alpha` | Smoothing coefficient, weight of previous value; if omitted, disable smoothing when `action_filter.enabled` is false, otherwise use UI smoothing |
| `mouth_deg` / `mouth_until` | Open mouth at start, close at specified progress |
| `hold_recovery` | Suppress fall detection while running |
| `home_deg` | Training home posture, degrees per servo, when different from robot configuration |

Loading validates `model_api == 1`, `obs_len == 61`, `action_len == 14`, `action_scale == 1`, file size, `robot.model` membership in `compatible_models`, and supported kind/encoding combinations. Failures are rejected with log/UI error codes.

### 7.3 Policy library

| Source | Location | Behavior |
| --- | --- | --- |
| Built-in configuration | `policy.policies`, weights in `models/` | Missing files skipped |
| Imported | `data/policies/<name>.bin` / `.json` | Same name overrides built-in; walk/recovery override by slot; deleting restores built-in |

Maximum 16 policies. Both walking and recovery must be available; otherwise only torque off, stand, and preview are possible.

The Actions page accepts `policy.onnx` and `manifest.json`. Browser code in `web/js/onnx.js` parses a graph of Sub, Div, (Gemm, Elu)×3, Gemm, extracts mean/std and weights into float32, validates the manifest, and calls `policy.import`. The board validates again, saves, loads, and emits `EV_POLICIES_CHANGED`. `tools/import_policy.py` uses the same interface and can also convert without uploading.

- Imported policies start unverified and are browser-only until `policy.verify` enables voice, Xiaozhi, and gamepad use.
- Replacing walk/recovery requires confirmation. Running policies cannot be replaced or deleted.
- Skill buttons, MCP `do_action` choices, gamepad `skill:<name>`, and voice skills are generated or checked against the library. References to absent policies are skipped.

## 8. Unified commands

WebSocket `/ws`, voice, Xiaozhi, gestures, and gamepad share the methods in `app/api.c` and each feature's `methods` table.

| Method | Owner | Web source only |
| --- | --- | --- |
| `robot.info / status / set_mode / move / stop / estop / skill / head / mouth / sound` | app | No |
| `robot.filter`, `servo.*`, `config.get` | app | Yes |
| `calib.*` | app/calib | Yes |
| `policy.list / import / remove / verify` | app/policies | Except list |
| `sound.list / upload / remove / settings` | app/sound | Except list |
| `voice.command / status / enable / wake` | voice | Yes |
| `xiaozhi.status / mcp` | xiaozhi | Yes |
| `gesture.control` | gesture | Yes |
| `wifi.status / scan / connect` | wifi | Yes |
| `wifi.ap` | wifi | No |
| `pad.status` | gamepad | Yes |

Errors return `{"code": "...", "message": "..."}`. `code` is a stable English identifier localized by the UI; `message` is an English explanation for scripts/logs.

Xiaozhi MCP tools: `self.duck.get_status / stand / rest / walk / turn / stop / do_action / look / mouth / quack`, plus `self.camera.take_photo`.

## 9. Browser UI

- Native JavaScript ES modules and CSS, with no build step; refresh after editing.
- `robot.info` determines available controls.
- Bottom navigation on phones, side navigation on wide screens, system light/dark theme, minimum 44 px click targets.
- JSON-RPC commands and 100 ms status pushes use `/ws`; camera video uses `/video`.

| Page | Contents |
| --- | --- |
| Control | Camera/hand skeleton, modes, skills, joysticks, mouth, voice bar with state, last phrase, Xiaozhi captions, wake, interrupt |
| Actions | Policy library, import, verify, delete |
| Status | Attitude indicator, servo table, IMU, ToF, loop rates |
| Maintenance | Calibration, servo tools, calibration export/import/undo |
| Settings | Wi-Fi, smoothing, sounds, Xiaozhi, command/gesture lists, about/licenses |

The fixed top bar shows connection, mode, language, and emergency stop. Missing zero or level calibration produces a banner linking to the wizard.

Translations are in `web/i18n/zh.json` and `en.json`. HTML uses `data-i18n="key"`; JS uses `t(key, params)`. Language defaults to the browser and persists in `localStorage` after switching. Configuration labels use `{"zh": ..., "en": ...}`. `tools/i18n_check.py` checks matching keys.

```text
web/
├── index.html
├── css/        base.css (variables/themes), layout.css, components.css
├── js/
│   ├── main.js        Startup, routing, feature visibility
│   ├── rpc.js         WebSocket, reconnect, error codes
│   ├── i18n.js        Localization
│   ├── store.js       Latest state and subscriptions
│   ├── onnx.js        ONNX parsing/conversion
│   ├── components/    joystick attitude diagram chart toast
│   └── pages/         control actions status maintain settings
├── i18n/       zh.json en.json
└── vendor/     jmuxer.min.js, H.264 playback
```

## 10. Calibration

Maintenance → Calibration wizard has four independently repeatable steps:

| Step | Procedure | Check |
| --- | --- | --- |
| 1. Servo check | Ping 15 servos and IMU individually; optionally scan entire bus | Missing/extra IDs |
| 2. IMU bias | Hold still for two seconds | Board reports success/failure |
| 3. Zero positions | Torque off: pose by hand and capture all/individual joints. Torque on: hold current position gently, adjust each slider to zero, then capture | — |
| 4. Standing posture | Measure torso roll/pitch and joint tracking error, suggest corrections using `calibration.roll_fix`, allow manual adjustments, then set level | Roll within 1° |

Configuration example (illustrative JSON with comments):

```jsonc
"calibration": {
  "pose_deg": [0, 0, ...], // Joint angles in calibration pose or fixture
  "roll_fix": []          // [[servo index, correction degrees per degree of roll], ...]
}
```

`data/calibration.json` stores offsets, IMU level, time, robot name, and step completion. Previous data is saved as `calibration.prev.json`, allowing one undo. Export/import is supported. Data is flushed before file replacement to protect saved calibration against power loss.

| Method | Function |
| --- | --- |
| `calib.enter` / `exit` | Release all torque on entry; `{hold: true}` holds current positions gently |
| `calib.targets` / `finish` | Torque-on target adjustment and per-joint capture |
| `calib.capture` | Torque-off capture of all joints or one `{index}` |
| `calib.imu` | Bias/level |
| `calib.suggest` / `nudge` | Suggest/apply corrections |
| `calib.mark` / `export` / `import` / `undo` | Step flags and calibration data |

## 11. Sounds

| Name | Default event |
| --- | --- |
| `quack` | Wake |
| `quack_double` | Command success |
| `quack_happy` | Skill complete |
| `quack_question` | Listening timeout |
| `quack_angry` | Rejected command or emergency stop |
| `quack_scared` | Fall |
| `quack_sad` | Low battery |
| `quack_sleepy` | Long inactivity |
| `tick` | UI button feedback |

- `tools/gen_sounds.py` synthesizes 16 kHz mono WAV using the Apache-2.0 Microduck algorithm. Seed controls voice; default 146.
- Uploaded recordings in `data/sounds/` override built-in files with the same name.
- The robot's `sounds` section maps events and configures randomized idle calls while standing.
- During duck sounds and Xiaozhi speech, mouth movement follows loudness in 20 ms intervals, only with torque enabled and no skill controlling the mouth.
- One speaker stream at a time: Xiaozhi speech > event sounds > idle/UI-requested sounds.
- `app/sound.c` depends only on `audio.h`. Without a speaker it is silently skipped, and `robot.sound` returns `no_audio`.

## 12. Storage

| Data | Examples | Location |
| --- | --- | --- |
| Application files | `doitduck`, `robots/`, `web/`, `sounds/` | Application directory |
| Models | User-supplied motion weights; supplied wake/gesture models | `models/` |
| Imported policies | `<name>.bin` / `.json` | `data/policies/` |
| User recordings | `<name>.wav` | `data/sounds/` |
| Calibration | `calibration.json` and previous copy | `data/` |
| Small settings | Xiaozhi device ID, volume, etc. | `data/*.json`, `data/*.txt` |

Application updates replace program files without touching `data/`.

## 13. Platforms

V610: run `sh tools/build.sh projects/robot/doitduck/firmware/main_board` in the V610-Apps root. Output: `firmware/main_board/out/`.

PC: `cmake -S firmware/main_board/pc -B build/pc && cmake --build build/pc`, then run `./doitduck` from the build directory. Without bus/audio/camera hardware, this tests the UI, commands, arbitration, MCP, and policy import. `tools/ci.sh` builds it and checks translations.
