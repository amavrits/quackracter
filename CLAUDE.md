# Microduck Personality Project

## What this is

A personality layer on top of the Pollen Robotics Microduck robot.
Instead of task execution, the duck has a persistent character that reacts
to its environment — through movement, audio, and memory — driven by an LLM.

Forked from: `pollen-robotics/microduck`

## Project goal

Make the Microduck feel alive. Not an assistant. A character.

The LLM is a character engine, not a task planner. It reads sensor state,
updates internal mood, and decides how the personality reacts — unprompted,
continuously, with memory of past interactions.

## Architecture

```
Sensor input (camera, IMU, ToF, microphone)
  → personality state update (mood, energy, memory)
  → LLM character engine (Claude)
  → {movement: policy_name, audio: text_or_quack}
  → robot.play(movement) + onboard speaker
```

## Repository structure

```
microduck-personality/
├── CLAUDE.md
├── personality/
│   ├── system_prompt.py      # character definition and prompt builder
│   ├── state.py              # mood, energy, memory, time tracking
│   └── reactor.py            # LLM call → structured action output
├── sensors/
│   ├── camera.py             # camera feed, presence detection
│   └── imu.py                # motion, stillness, fall detection
├── audio/
│   ├── voice.py              # TTS via onboard speaker
│   └── listen.py             # microphone input, wake word, VAD
├── robot/
│   └── client.py             # wrapper around microduck SDK policy calls
└── main.py                   # main event loop
```

## Robot interface

Policies available on the robot (via `robotctl` / SDK):

| Policy | Description |
|---|---|
| `walk` | velocity-tracking gait |
| `sitstand` | sit down and stand up |
| `kick` | one-shot kick |
| `grab` | beak dip to floor and up |
| `recover` | get back up from fall |
| `roller` | roller skating locomotion |

Call pattern:
```python
robot.play("walk", velocity=0.3)
robot.stop()
```

## LLM interface

Model: Claude (Anthropic API)

The reactor sends a structured prompt describing:
- Current personality state (mood, energy, boredom level)
- Recent sensor events (motion detected, stillness duration, time of day)
- Recent interaction history (last N events)

Expected structured output (JSON):
```json
{
  "movement": "kick",
  "audio": "That startled me!",
  "mood_delta": {"energy": +0.1, "curiosity": -0.05},
  "memory_note": "owner walked past at 14:32"
}
```

## Personality state

Internal state tracked continuously:

```python
state = {
    "mood": "neutral",          # neutral | curious | bored | excited | grumpy
    "energy": 0.8,              # 0.0 - 1.0, decays over time
    "boredom": 0.2,             # 0.0 - 1.0, increases with stillness
    "curiosity": 0.5,           # 0.0 - 1.0
    "last_interaction": dt,     # datetime of last owner interaction
    "memory": []                # list of recent notable events
}
```

## Reaction triggers

The reactor fires when:
- Boredom exceeds threshold (duck initiates behaviour unprompted)
- Motion detected by camera or IMU (someone nearby)
- ToF sensor detects close object
- Time-based events (morning, long idle period)
- Owner interaction (sound, presence)

## Sensor notes

- **Camera**: presence detection, rough motion tracking
- **IMU**: orientation, stillness detection, fall detection
- **ToF (8×8)**: low-res depth, obstacle/proximity detection
- **Microphones**: onboard — voice input, ambient sound detection
- **Speaker**: onboard — per-robot generated voice, fully self-contained audio

The robot has a full onboard audio stack. No companion device needed —
the duck can listen and speak natively. Voice input → LLM → voice output
+ movement is a fully self-contained loop.

## Development phases

1. **Sim first** — build and test personality layer against the browser
   simulator at `huggingface.co/spaces/pollen-robotics/microduck-simulator`
   before hardware ships (December 2026)
2. **Personality definition** — write the character brief and system prompt
3. **State machine** — mood/energy/boredom update logic
4. **Reactor** — LLM call with structured output
5. **Sensor integration** — camera presence, IMU stillness
6. **Audio** — onboard TTS + microphone input, wake word detection
7. **Hardware deploy** — test on real robot when it arrives

## Key design decisions

- LLM decides WHAT to express, trained policy decides HOW to move
- Personality persists across sessions (serialised state)
- Reactions should feel organic, not triggered — timing variation matters
- The duck should do nothing most of the time (idle is a personality choice)

## Related projects

- `quackd` (rokbenko/quackd) — LLM task execution for Microduck, no personality layer
- `pollen-robotics/microduck_rl` — official RL training stack (MuJoCo Warp + PPO)
- `jax-agents` (amavrits/jax-agents) — JAX RL library, potential training backend

## Stack

- Python 3.11+
- Anthropic SDK (Claude)
- Microduck SDK
- OpenCV (camera)
- Onboard speaker/mic via robot audio API
- Optional: ElevenLabs for richer voice quality over WiFi
- MuJoCo / microduck simulator for dev
