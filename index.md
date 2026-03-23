---
title: krach
subtitle: live coding audio
author:
    - Jonas Köhler
    - Anyere Bendrien
author-url:
    - https://argmin.xyz
    - https://github.com/Bendrien
date: 2026-03-23
lang: en
toc-title: Contents
version: v1.0
---

## What is krach?

A live coding audio system. Write synths in Python, sequence them with composable patterns, hear changes instantly. One process, zero latency.

```python
import krach.dsp as krs

@kr.dsp
def acid_bass() -> krs.Signal:
    freq = krs.control("freq", 55.0, 20.0, 800.0)
    gate = krs.control("gate", 0.0, 0.0, 1.0)
    cutoff = krs.control("cutoff", 800.0, 100.0, 4000.0)
    env = krs.adsr(0.005, 0.15, 0.3, 0.08, gate)
    return krs.lowpass(krs.saw(freq), cutoff) * env * 0.55

kr.voice("bass", acid_bass, gain=0.3)
kr.play("bass", kr.seq("A2", "D3", None, "E2").over(2))
kr.play("bass/cutoff", kr.mod_sine(200, 2000).over(4))
```

## How it works

**Python** defines synths and patterns. **Rust** runs the audio engine. **FAUST** compiles DSP to native code via LLVM JIT.

```
Python (define)          Rust (execute)              Audio
──────────────           ──────────────              ─────
kr.voice("bass", fn) →  FAUST JIT compile       →  CoreAudio
kr.play("bass", pat) →  pattern → curves         →  block-rate automation
kr.set("bass/cut", v) → SetControl               →  node parameter
```

Two symbols: `kr` (the mixer) and `krs` (DSP primitives). That's the entire API.

## Features

**Synth design** — write DSP functions in Python, they transpile to FAUST and JIT-compile to native audio. Hot reload: edit a function, hear the change.

```python
@kr.dsp
def kick() -> krs.Signal:
    gate = krs.control("gate", 0.0, 0.0, 1.0)
    env = krs.adsr(0.001, 0.25, 0.0, 0.05, gate)
    return krs.sine_osc(55.0 + env * 200.0) * env * 0.9
```

**Composable patterns** — TidalCycles-inspired algebra. Sequence, layer, stretch, swing.

```python
kr.play("kick", kr.hit() * 4)                          # 4 on the floor
kr.play("hat",  (kr.rest() + kr.hit()) * 4)             # offbeat
kr.play("bass", kr.seq("A2", "D3", None, "E2").over(2)) # bass line
kr.play("hat",  (kr.hit() * 8).swing(0.67))             # swung 8ths
```

**Pattern algebra** — everything returns a pattern, compose infinitely:

```python
a + b           # sequence
a | b           # layer (simultaneous)
p * 4           # repeat
p.over(2)       # stretch to 2 cycles
p.swing(0.67)   # swing feel
p.every(4, lambda p: p.reverse())   # transform every 4th cycle
p.spread(3, 8)  # euclidean rhythm
```

**Effect routing** — buses, sends, wires:

```python
kr.bus("verb", reverb_fn, gain=0.3)
kr.send("bass", "verb", level=0.4)
```

**Live performance** — mute, solo, fade, save, export:

```python
kr.fade("bass/gain", 0.0, bars=4)
kr.mute("drums")
kr.export("my_session.py")
```

## Architecture

Single-process Rust binary. Pattern IR compiles to block-rate automation curves (~172 updates/sec) — no per-event IPC during playback. Lock-free audio thread.

```
noise/
├── audio-engine/      Rust — graph runtime, crossfade, automation
├── audio-faust/       Rust — FAUST LLVM JIT, hot reload
├── pattern-engine/    Rust — pattern sequencer, rational time, curve compiler
├── krach-engine/      Rust — unified binary
├── faust-dsl/         Python — Python → FAUST transpiler
└── krach/             Python — live coding REPL
```

## Install

```bash
# Prerequisites: macOS, Rust, Python 3.13+, uv, FAUST + LLVM
brew install faust llvm
export LLVM_SYS_181_PREFIX=$(brew --prefix llvm)

git clone https://github.com/krachio/noise.git && cd noise
cargo build --release -p krach-engine
cd krach && uv sync && cd ..
./bin/krach
```

## Links

[Documentation](https://github.com/krachio/noise/tree/main/docs) · [GitHub](https://github.com/krachio/noise) · [MIT License](https://github.com/krachio/noise/blob/main/LICENSE)

## Credits

Design: [the-monospace-web](https://github.com/owickstrom/the-monospace-web)
