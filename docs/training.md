# Training Motion Models

DoitDuck runs Microduck-format motion policies. This guide explains how to train for your own robot using [pollen-robotics/microduck_rl](https://github.com/pollen-robotics/microduck_rl) and deploy the results.

## 1. Model requirements

| Item | Requirement |
| --- | --- |
| Network | MLP 61 → 512 → 256 → 128 → 14, ELU; mean/std normalization embedded in ONNX |
| Frequency | 50 Hz |
| Observation | Gyroscope, gravity direction, 14 joint positions and velocities, previous action, three command values, four head commands, six body-posture commands; see [Design §7.1](doitduck-design.md#71-network-and-observations) |
| Action | 14 joint angles relative to home posture, radians, scale 1 |
| Joint order | Five left-leg joints, four neck/head joints, five right-leg joints; servo mapping is `policy.action_ids` |
| Manifest | Microduck schema 2; `robot.model` must appear in `policy.compatible_models` |

Models exported by the referenced `microduck_rl` framework satisfy these requirements.

## 2. Environment

Requires an NVIDIA GPU, CUDA for MuJoCo Warp training, and [uv](https://docs.astral.sh/uv/).

```sh
git clone https://github.com/pollen-robotics/microduck_rl
cd microduck_rl && git rev-parse HEAD        # Record commit; use one version per model batch
uv sync                                     # Large initial CUDA download; UV_HTTP_TIMEOUT=600 may help
uv run train Mjlab-Velocity-Flat-MicroDuck --env.scene.num-envs 4096 --agent.logger tensorboard
```

- Use `--agent.logger tensorboard` instead of wandb; logs appear in `logs/rsl_rl/<task>/`.
- Reduce `--env.scene.num-envs` if GPU memory is insufficient. As a reference, 12 GB supports 2048 environments for walking tasks; recovery tasks use much less memory.
- Run long training jobs in `tmux` so an SSH disconnect does not stop them.

## 3. Measure the real robot

At this scale, servo behavior is a major source of simulation-to-hardware mismatch. Measure first:

| Parameter | Method | Used for |
| --- | --- | --- |
| Total and head mass | Weigh them | Simulation masses |
| Servo model | Identify on a [BAM](https://github.com/Rhoban/bam) pendulum rig with the actual firmware P gain and supply voltage | Actuator model |
| Command delay | Time from target write to first measured movement | Delay randomization |
| Backlash | With torque off, move joints both ways and measure encoder dead travel | `*-Backlash` randomization |
| Voltage | Fully charged and under load | Voltage randomization |
| IMU | Stationary noise and gravity direction when standing | Observation noise and mounting offset |

Acceptance: simulated and measured single-joint step-response curves should approximately overlap.

## 4. Training configuration

In your branch, modify only robot-specific parameters. Retain tasks, rewards, and observations to preserve 61 observations, 14 actions, and 50 Hz operation.

- Frame: for the official geometry, use the official `robot_*.xml` and change masses only. For custom geometry, export your own MJCF, for example using onshape-to-robot.
- Actuators: substitute your servo's BAM model and measured voltage, delay, and friction randomization ranges.
- Home posture: match `motors.home_deg`, or provide the training posture as `doitduck.home_deg` in the policy manifest.

Acceptance: `uv run list-envs` lists your task and training starts successfully.

## 5. Tasks

| Task | DoitDuck use | Notes |
| --- | --- | --- |
| `Velocity-*-Backlash` | Walking | No stand-up recovery |
| `StandUp-*-Backlash` | Recovery | |
| `VelStand-*-Backlash` | Walking and recovery | One model serves both roles |
| `GroundPick-*` | Ground picking | `phase` encoding, four-second period |
| `Roulade-Flat` | Somersault | Suppresses fall detection with `hold_recovery` |
| `BallKick*` | Kicking | |
| `SitStand-*` | Sit/stand | |

`VelStand` does not train from scratch in the referenced official configuration: it warm-starts from a walking checkpoint, distills a stand-up teacher, and uses the walking checkpoint as an anchor. For entirely self-trained weights, train `Velocity-*-Backlash` and `StandUp-*-Backlash` first, then point the warm-start, teacher, and anchor paths to your own checkpoints. Confirm that no third-party checkpoint is downloaded. The other listed tasks do not use warm-starting or distillation in the documented configuration.

Resume interrupted training with `--agent.load-checkpoint model_XXXX.pt --agent.resume True`.

## 6. Acceptance checks

### Simulation

- Run `uv run scripts/infer_policy.py --walking walk.onnx --standing ...` in CPU MuJoCo to check behavior in another simulator.
- Repeat with `--vin` and `--kp-fw` set to measured worst-case voltage and gain.
- Target: walk on level ground for 60 seconds without falling, recover from a push, stand within five seconds after falling, and track velocity commands.

### Hardware

1. Export ONNX and manifest, convert into `models/`, or import through the browser.
2. Progress from supported Inference only output inspection, to suspended torque-enabled checks for symmetry and oscillation, to standing, then slow walking from 0.1 m/s, followed by turning and sideways motion.
3. If hardware disagrees with simulation, revisit measurements and training configuration. Do not hide the mismatch with extra runtime smoothing or action scaling.

## 7. Deploy to DoitDuck

```sh
uv run scripts/export.py <task> --checkpoint-file logs/rsl_rl/<experiment>/<run>/model_XXXX.pt --onnx-file policy.onnx
python3 tools/import_policy.py policy.onnx manifest.json --out models/<name>.bin   # In the DoitDuck repository
```

Use [Microduck schema 2](https://github.com/pollen-robotics/microduck/blob/main/docs/policy-manifest.md) for `manifest.json`. `uv run publish` in `microduck_rl` generates one when publishing; an existing model manifest can also be adapted.

Add a `policy.policies` entry with filename, `name`, `kind`, `slot` or `duration_s`, `command`, and other applicable fields, following the existing configuration. Alternatively, import directly through Actions. The manifest's `robot.model` must be in `policy.compatible_models`.

## 8. Record provenance

Record in each manifest:

- Framework and commit, task, whether training started from scratch, and any warm-start/teacher checkpoints.
- Servo-model source and total robot mass.
- Checkpoint iteration and ONNX SHA-256.
- Hardware validation scope: motions, surfaces, and duration.

If third-party weights are used for teaching, warm-starting, or fine-tuning, the resulting model must comply with those weights' licenses.
