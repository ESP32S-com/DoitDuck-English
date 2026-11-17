# Third-Party Components

| Component | Location | License | Purpose |
| --- | --- | --- | --- |
| cJSON | `firmware/main_board/components/cjson/` | MIT; see its LICENSE | JSON |
| Mbed TLS 3.6.4 | `firmware/main_board/components/mbedtls/` | Apache-2.0 or GPL-2.0-or-later; see its LICENSE | Xiaozhi HTTPS and WebSocket |
| ST VL53L5CX Ultra Lite Driver 1.3.9 | `firmware/main_board/components/vl53l5cx/`; `vl53l5cx_api.c/.h` and `vl53l5cx_buffers.h` are unmodified; `platform.c/.h` is this project's Linux I2C adapter | BSD-3-Clause; see LICENSE.md; source: [STMicroelectronics/stm32-vl53l5cx](https://github.com/STMicroelectronics/stm32-vl53l5cx) | ToF driver, including sensor firmware |
| jmuxer 2.0.5 | `web/vendor/jmuxer.min.js` | MIT; see `web/vendor/jmuxer.LICENSE.txt` | Browser H.264 playback |
| Microduck `sounds` synthesizer | `tools/gen_sounds.py`; Rust-to-Python port, changes documented in its header | Apache-2.0; see `tools/gen_sounds.LICENSE`; source: [pollen-robotics/microduck](https://github.com/pollen-robotics/microduck/tree/main/sounds) | Synthesizes the duck sounds in `sounds/` |

The following are not included and must be supplied separately at build or runtime:

| Component | Notes |
| --- | --- |
| V610-App-SDK (hifw, media_engine, mipk) | Required to build for Cove-610; see SDK licensing |
| Motion models | Place in `models/` or import in the browser; see each model's license |

`models/nihaoxiaoya.mipk` and `models/kaishipeiwang.mipk` are speech models trained by this project and use the repository's license.

The manifest format follows Microduck's `policy-manifest.md` (schema 2); this repository reads and writes that format. Files in `sounds/` are synthesized by `tools/gen_sounds.py`, without using recordings. The synthesis algorithm was ported from Microduck's Apache-2.0 `sounds` project, as listed above.
