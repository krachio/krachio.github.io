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

A live coding audio system. Write synths in Python, wire them as a graph, sequence with composable patterns, hear changes instantly.

```python
import krach.dsp as krs

@kr.dsp
def acid_bass() -> krs.Signal:
    freq = krs.control("freq", 55.0, 20.0, 800.0)
    gate = krs.control("gate", 0.0, 0.0, 1.0)
    cutoff = krs.control("cutoff", 800.0, 100.0, 4000.0)
    env = krs.adsr(0.005, 0.15, 0.3, 0.08, gate)
    return krs.lowpass(krs.saw(freq), cutoff) * env * 0.55

bass = kr.node("bass", acid_bass, gain=0.3)
verb = kr.node("verb", reverb_fn, gain=0.3)
bass >> verb                                       # route
bass @ kr.seq("A2", "D3", None, "E2").over(2)      # play
bass["cutoff"] = 1200                               # control
```

## How it works

**Python** defines synths and patterns. **Rust** runs the engine. **FAUST** JIT-compiles DSP to native audio.

```
Python (define)          Rust (execute)              Audio
──────────────           ──────────────              ─────
kr.node("bass", fn)  →  FAUST JIT compile       →  CoreAudio
bass @ pattern       →  pattern → curves         →  block-rate automation
bass["cutoff"] = v   →  SetControl               →  node parameter
bass >> verb         →  graph rebuild + crossfade →  seamless routing
```

Two symbols: `kr` (the audio graph) and `krs` (DSP primitives). That's the entire API.

## Operator DSL

Everything is a node. Routing uses `>>`. Patterns use `@`. Controls use `[]`.

```python
bass = kr.node("bass", bass_fn, gain=0.3)
verb = kr.node("verb", reverb_fn, gain=0.3)

bass >> verb                           # route signal
bass >> (verb, 0.4)                    # route with send level
bass @ kr.seq("A2", "D3").over(2)      # play pattern
bass @ "A2 D3 ~ E2"                   # mini-notation shorthand
bass @ None                            # hush
bass["cutoff"] = 1200                  # set control

with kr.transition(bars=8):           # all changes fade over 8 bars
    bass["gain"] = 0.8
    kr.tempo = 140
```

## Synth design

Write DSP functions in Python. They transpile to FAUST and JIT-compile via LLVM. Hot reload on save. Auto-smoothing on all non-gate parameters.

```python
@kr.dsp
def kick() -> krs.Signal:
    gate = krs.control("gate", 0.0, 0.0, 1.0)
    env = krs.adsr(0.001, 0.25, 0.0, 0.05, gate)
    return krs.sine_osc(55.0 + env * 200.0) * env * 0.9
```

## Pattern algebra

TidalCycles-inspired composable patterns. Every operation returns a pattern.

```python
a + b           # sequence
a | b           # layer (simultaneous)
p * 4           # repeat
p.over(2)       # stretch to 2 cycles
p.swing(0.67)   # swing feel
p.mask("1 1 0 1")           # suppress events
p.sometimes(0.3, reverse)   # probabilistic transform
p.every(4, reverse)          # transform every 4th cycle
p.spread(3, 8)               # euclidean rhythm

kr.struct(rhythm, melody)    # impose rhythm onto melody
kr.cat(a, b, c)             # play each for 1 cycle, loop
kr.p("x . x . x . . x")    # mini-notation
```

## Architecture

Single-process Rust binary. Patterns compile to block-rate automation curves (~172 updates/sec). Lock-free audio thread. FAUST hot-reload.

```
noise/
├── audio-engine/      Rust — graph runtime, crossfade, automation
├── audio-faust/       Rust — FAUST LLVM JIT, hot reload
├── pattern-engine/    Rust — pattern sequencer, rational time, curve compiler
├── krach-engine/      Rust — unified binary (one process, one socket)
├── faust-dsl/         Python — Python → FAUST transpiler
└── krach/             Python — live coding REPL, operator DSL
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

[Documentation](https://krach.io/noise/docs/) · [GitHub](https://github.com/krachio/noise) · [Try it](https://krach.io/noise/try/) · [MIT License](https://github.com/krachio/noise/blob/main/LICENSE)

## Credits

Design: [the-monospace-web](https://github.com/owickstrom/the-monospace-web)
