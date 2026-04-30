# Grampus

## ⚠️ IF YOU NEED ONE PIECE OF INFO, IT'S THIS ⚠️

# **Press `Ctrl+H` (keyboard) or `R1+Select` (gamepad) to open the in-app HELP POPUP.**

**It lists every keyboard shortcut, gamepad combo, and mode binding.**

---

A terminal application that pairs an **[Orca](https://github.com/hundredrabbits/Orca-c)-style grid sequencer** with **built-in synth engines**. You write procedural music by placing single-character glyphs on a 2D grid; those glyphs drive polyphonic voices and effects rendered in real time through your audio output.

Grampus runs on macOS, Linux, Windows, and small-form-factor Linux handhelds. It works in terminals as small as 78×30.

---

## Table of Contents

- [Core idea](#core-idea)
- [Signal flow](#signal-flow)
- [Sequencer](#sequencer)
  - [Grid model](#grid-model)
  - [Base-36 values](#base-36-values)
  - [Bang and mark system](#bang-and-mark-system)
  - [Tick rates and timing](#tick-rates-and-timing)
  - [Operator reference](#operator-reference)
- [Synth engines](#synth-engines)
  - [Drifter (EngType 0)](#drifter-engtype-0)
  - [Warper (EngType 1)](#warper-engtype-1)
  - [BrokenFM (EngType 2)](#brokenfm-engtype-2)
  - [String / Karplus (EngType 3)](#string--karplus-engtype-3)
  - [ROMpler (EngType 4)](#rompler-engtype-4)
  - [LoFiTron (EngType 5)](#lofitron-engtype-5)
  - [WaveSurf (EngType 6)](#wavesurf-engtype-6)
  - [SynDrums (EngType 7)](#syndrums-engtype-7)
  - [BoneSaw (EngType 8)](#bonesaw-engtype-8)
- [Shared voice chain](#shared-voice-chain)
- [Drive models](#drive-models)
- [LFO system](#lfo-system)
- [Scenes](#scenes)
- [Master effects bus](#master-effects-bus)
- [Parameter system](#parameter-system)
- [Features](#features)
- [CLI](#cli)
- [Credits](#credits)

---

## Core idea

Two strictly separated pillars:

1. **Sequencer** - a grid-based esoteric programming language for procedural sequencing, drawing its operator set from [Orca-c](https://github.com/hundredrabbits/Orca-c) via [bOrca](https://github.com/boorch/bOrca) (a fork with custom and modified operators). Emits notes and parameter messages. Knows nothing about sound.
2. **Engines** - eight polyphonic synth tracks with filter, envelope, drive, and effect sends. Receive messages. Know nothing about the sequencer.

The sequencer talks to the engines one way only - **sequencer → engines**. Either side can be swapped without touching the other. The sequencer can also send MIDI externally (planned).

---

## Signal flow

```
Grid (sequencer)
       │
       │  notes / parameter changes
       ▼
8 synth tracks (T1..T8)
  each track: voice → envelope → filter → drive → per-track compressor
                                                     ↓
                                         pan + volume → sends
                                                     ↓
                              ┌──────────────────────┴───────────────────────┐
                              ▼                                              ▼
                       Master bus dry                          Delay / Reverse delay / Reverb
                              └──────────────────────┬───────────────────────┘
                                                     ▼
                                  Tape noise + Vinyl dust mixed in
                                                     ▼
                                 Master gain → FAT compressor → safety limiter
                                                     ▼
                                              Stereo output
```

Every track is editable on its own page (T1..T8). The master page hosts the global delay, reverb, tape/vinyl loops, master gain, and the FAT bus compressor. The sequencer drives notes and can also write parameter changes live via the `!` operator.

---

## Sequencer

### Grid model

The sequencer holds a large 2D grid of single-character cells (248×128 by default; the viewport scrolls). Each cell holds either:
- **An operator** (letters `a`-`z`/`A`-`Z` and symbols `:`, `%`, `!`, `?`, `=`, `;`, `#`, `$`, `&`, `*`)
- **A data glyph** (`0`-`9`, `a`-`z` used as base-36 values, or `.` = empty)

Uppercase vs. lowercase for letter operators:
- **Uppercase**: runs every tick (always active).
- **Lowercase**: only runs when it has a neighbouring bang (`*`). This is how you gate them.

### Base-36 values

All operator ports read and write a single base-36 digit:

| Glyph | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | a | b | c | d | e | f | g | h | i |
|------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Value | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 |

| Glyph | j | k | l | m | n | o | p | q | r | s | t | u | v | w | x | y | z |
|------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Value | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 | 32 | 33 | 34 | 35 |

Case is preserved in the grid for readability (uppercase often signals an operator, lowercase signals data), but for *value* reads case is ignored. 36 distinct values per port is rich enough for notes, channels, CC values, rates, etc.

### Bang and mark system

`*` is the **bang** glyph. It is transient: the cell is reset to empty the tick after it fires. Lowercase operators inspect their four neighbours and only run when one of them holds `*`. This is how you build rhythms: a bang moves, hits an operator, fires it, disappears.

A few symbol operators run every tick unconditionally regardless of banging: `#`, `$`, `&`, `?`. The note / event operators and `!` (`;`, `:`, `=`, `%`, `!`) need a neighbouring bang to fire, the same way lowercase letter ops do. The only truly always-on symbol op is `?`; if you want to rate-limit it, gate it with another operator (e.g., a `C` clock + conditional).

### Tick rates and timing

- **BPM** drives the base tempo.
- **TickRate** multiplies the division: `Half` = 2 ticks/beat, `Normal` = 4 ticks/beat, `Double` = 8 ticks/beat.
- **Shuffle** (25–75%, default 50%) adds swing by displacing every odd-numbered tick (the 2nd, 4th, 6th, … tick after `Reset`). 50% = straight, classic forward swing lives around 60–67%, values below 50% pull the offbeat earlier (anticipated / reverse swing). The pair-average BPM is preserved, so external sync stays on grid; only the within-pair phase moves. Reset always restarts cleanly from tick 0 (an even tick, never displaced).

### Operator reference

The operator set is derived from [Orca-c](https://github.com/hundredrabbits/Orca-c) and its fork [bOrca](https://github.com/boorch/bOrca). Grampus takes these as a starting point and adapts a few of them for its built-in synths: `!` is rebuilt as a **SetParam** operator addressing the internal synth parameters (not MIDI CC), `&` uses a musical cycle-length rate model, and the `$`/`=` chord table contains grampus's own selection and ordering of chord types.

**Conventions** used in the port tables below:

- The row of cells shown *is* the literal grid layout around the operator. Cells labelled `I` (or named) to the left/right/above of the operator are **inputs**; cells labelled `O` are **outputs**.
- Uppercase glyph = operator runs every tick. Lowercase glyph = operator only runs when it has a neighbouring bang (`*`) touching it from N / E / S / W.
- The note / event operators and `!` (`;`, `:`, `=`, `%`, `!`) are dispatched every tick but **require a neighbouring bang** to actually fire (each checks internally and exits if no bang is present).
- `?` **fires every tick unconditionally** regardless of banging. Gate it externally (typically with an upstream `D`) if you don't want it firing constantly.
- `$`, `#`, `&` **run every tick unconditionally** (no lowercase/uppercase distinction).
- A grey glyph like `.` in a port table means "input cell; value read here". Capital glyphs in port tables (`O`, `N`, `V`…) are the single-letter names used in the port-name row directly above them.

---

#### Arithmetic & comparison

##### `A` - Add

Outputs `(a + b) mod 36`. Case of the output follows the case of the right input.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
|  a   |   `A`    |   b   |
|      |    `O`   |       |

##### `B` - Subtract

Outputs `|b − a|`. Case follows the right input.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
|  a   |   `B`    |   b   |
|      |    `O`   |       |

##### `M` - Multiply

Outputs `(a × b) mod 36`. Case follows the right input.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
|  a   |   `M`    |   b   |
|      |    `O`   |       |

##### `L` - Lesser

Outputs the smaller of the two inputs. Returns `.` (empty) if either input is empty.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
|  a   |   `L`    |   b   |
|      |    `O`   |       |

---

#### Timing

##### `C` - Clock

Rolling counter. Outputs `(tick ÷ rate) mod modulo`. Defaults: rate = 1, modulo = 8.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
| rate |   `C`    |  mod  |
|      |    `O`   |       |

##### `D` - Delay

Emits a bang (`*`) on the cell below every `rate × modulo` ticks. Otherwise the cell below is cleared. Defaults: rate = 1, modulo = 8.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
| rate |   `D`    |  mod  |
|      |    `*`   |       |

##### `U` - Uclid

Euclidean rhythm - distributes `steps` bangs as evenly as possible across `max` ticks. Defaults: steps = 1, max = 8.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
|steps |   `U`    |  max  |
|      |    `*`   |       |

---

#### Flow control

##### `F` - If

Emits a bang south if left == right, otherwise clears the output cell.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
|  a   |   `F`    |   b   |
|      |    `*`   |       |

##### `H` - Halt

Locks the cell directly south so it won't execute this tick. Has no numeric output - it just acts as a gate on its own column.

| Operator |
|:--------:|
|   `H`    |
|  locked  |

##### `I` - Increment

Reads the cell below, adds `rate` (default 1), wraps at `max` (default 36), writes it back. A self-incrementing counter that runs in place.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
| rate |   `I`    |  max  |
|      | counter  |       |

##### `Z` - Lerp

Moves the cell below toward `target` by `rate` each tick, stops when reached. Smoothing for parameter sweeps.

| Left | Operator | Right  |
|:----:|:--------:|:------:|
| rate |   `Z`    | target |
|      |  value   |        |

---

#### Directional movement

Each of these has no input ports - it simply tries to move itself one cell in its direction every tick. If the destination is empty, it moves there; if the destination is a wall or another glyph, it collides and turns into a bang (`*`) in place.

##### `E` - East

Moves one cell right each tick.

##### `W` - West

Moves one cell left each tick.

##### `N` - North

Moves one cell up each tick.

##### `S` - South

Moves one cell down each tick.

---

#### Jumps

##### `J` - Jump

Reads the cell directly **north**, writes it to the first non-`J` cell directly south. Chains of `J`s act as a single vertical relay.

| north input |
|:-----------:|
|    value    |
|    `J`      |
|   (J…J…J)   |
|  written    |

##### `Y` - Yump

Reads the cell directly **west**, writes it to the first non-`Y` cell directly east. Horizontal relay.

| West input | Operator | (chain of Ys…) | Output |
|:----------:|:--------:|:--------------:|:------:|
|   value    |   `Y`    |     `Y Y Y`    | value  |

---

#### Memory

The sequencer has 36 global variable slots indexed by base-36 glyph name.

##### `V` - Variable

Two modes, selected by which port is filled:

- **Write**: if the left port (name) is non-empty, store `right` into the named slot.
- **Read**: if the left port is empty and the right port (name) is non-empty, output that slot's value south.

| Left (name) | Operator | Right (value or name) |
|:-----------:|:--------:|:---------------------:|
|    name     |   `V`    |    value / name       |
|             |  output  |                       |

##### `K` - Konkat

Reads `len` variable names starting at the cell one east of `K`, outputs each one's value directly below it. Fast way to fetch a whole cluster of variables.

| Left | Operator | +1 | +2 | … | +len |
|:----:|:--------:|:--:|:--:|:-:|:----:|
| len  |   `K`    | n₁ | n₂ | … |  nₖ  |
|      |          | v₁ | v₂ | … |  vₖ  |

---

#### Array-style addressing

##### `T` - Track

Indexed read from a horizontal row of `len` cells placed to the east of the operator. `key mod len` selects which cell is read; the result comes out south.

| −2  | −1  | Operator | +1 | +2 | … | +len |
|:---:|:---:|:--------:|:--:|:--:|:-:|:----:|
| key | len |   `T`    | d₁ | d₂ | … |  dₙ  |
|     |     |  output  |    |    |   |      |

##### `P` - Push

Indexed write. The value on the east is written into one of `len` cells directly south, chosen by `key mod len`.

| −2  | −1  | Operator | +1 (val) |
|:---:|:---:|:--------:|:--------:|
| key | len |   `P`    |  value   |

Row below: `s₁ s₂ … s_len`, with `value` written into slot `key mod len`.

##### `O` - Offset read

Reads the cell at relative offset `(ox + 1, oy)` from the operator and outputs it south.

| −2 | −1 | Operator |
|:--:|:--:|:--------:|
| ox | oy |   `O`    |
|    |    |  output  |

##### `X` - Teleport

Writes the value on the east to the cell at relative offset `(ox, oy + 1)`. Can reach anywhere on the grid.

| −2 | −1 | Operator | +1 (val) |
|:--:|:--:|:--------:|:--------:|
| ox | oy |   `X`    |  value   |

##### `G` - Generator

Batch write. Reads `len` values from cells `+1 … +len` east of the operator, writes them starting at relative `(ox, oy + 1)`.

| −3 | −2 | −1  | Operator | +1 | +2 | … | +len |
|:--:|:--:|:---:|:--------:|:--:|:--:|:-:|:----:|
| ox | oy | len |   `G`    | v₁ | v₂ | … |  vₙ  |

##### `Q` - Query

Batch read. Reads `len` cells starting at relative `(ox + 1, oy)`, outputs them in a horizontal row south of the operator (aligned so the last output sits directly under the operator).

| −3 | −2 | −1  | Operator |
|:--:|:--:|:---:|:--------:|
| ox | oy | len |   `Q`    |

Row below the operator: `v₁ v₂ … v_len`.

---

#### Randomness

##### `R` - Random

Pure random between `min` and `max`, hashed from position + tick. Runs every tick.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
| min  |   `R`    |  max  |
|      |  output  |       |

##### `r` - Shuffle Random

Fisher-Yates shuffled permutation of `[min, max]`. Guarantees no consecutive duplicates; reseeds when the sequence is exhausted. Lowercase only - **requires a bang**.

| Left | Operator | Right |
|:----:|:--------:|:-----:|
| min  |   `r`    |  max  |
|      |  output  |       |

---

#### Musical / pitch

##### `$` - Scale / Chord

Unified scale + chord lookup. Runs every tick. Outputs the **note** directly south of the operator and the **octave** one cell to the south-west.

| Operator | Octave | Root | Scale/Chord | Degree |
|:--------:|:------:|:----:|:-----------:|:------:|
|   `$`    |   O    |   R  |      S      |    D   |
|**octave**|**note**|      |             |        |

- **O** - octave (0-9). If empty, no octave is output.
- **R** - root note (`C`, `c`=C♯/D♭, `D`, `d`=D♯/E♭, `E`, `F`, `f`=F♯/G♭, `G`, `g`=G♯/A♭, `A`, `a`=A♯/B♭, `B`).
- **S** - scale or chord:
  - `0`-`9` - one of the 10 scales: Major, Minor, Lydian, Dorian, Mixolydian, Hirajoshi, Pentatonic, Tetratonic, Fifths, Iwato.
  - `a`-`z` - one of the 26 chord types (see chord table below).
  - Uppercase `A`-`Z` for chords = **first inversion** (root is bumped up an octave).
- **D** - scale/chord degree (0-indexed). Degrees beyond the last note wrap and add an octave.

##### `%` - Arpeggiator

Degree-based arpeggiator - outputs degree *numbers* (not MIDI notes), designed to feed into `$`. State is per-cell: the step counter advances each bang. **Requires a bang.**

| Operator | Range | Pattern |
|:--------:|:-----:|:-------:|
|   `%`    |   R   |    P    |
|  degree  |       |         |

- **R** - range in octaves (1-4). Output degrees cycle over `7 × R` positions.
- **P** - pattern 0-13:
  - `0` Up · `1` Down · `2` Up-Down (no endpoint repeat) · `3` Down-Up (no endpoint repeat)
  - `4` Up-Down+ (endpoints repeat) · `5` Down-Up+ · `6` Converge (outside-in) · `7` Diverge (inside-out)
  - `8` Pinky Up · `9` Thumb Up · `a` Up-Down Alt · `b` Down-Up Alt · `c` Random · `d` Bounce.

Changing pattern or range resets the counter.

##### `&` - Bouncer (LFO)

Tempo-synced LFO. Runs every tick. Resets its phase on a neighbouring bang or when `rate` / `shape` changes.

| Operator | Start | End | Rate | Shape |
|:--------:|:-----:|:---:|:----:|:-----:|
|   `&`    |   A   |  B  |   R  |   S   |
|  output  |       |     |      |       |

- **A / B** - output range. If `A > B` they swap; the LFO always interpolates between `min(A,B)` and `max(A,B)`.
- **R** - rate, 0-35. Indexes a 36-entry tempo-synced cycle table shared with the LFO Rate slots; higher values = faster cycles. See the [Rate table](#rate-table-shared-by-bouncer-and-lfos) for full speeds. The Bouncer can't fire faster than once per sequencer tick, so the sub-tick fast values (`x`/`y`/`z` = `1`/`0.125`/`0.0625` ticks) all collapse to a 1-tick cycle on the Bouncer (LFOs use them at the audio-rate speeds the table actually advertises).
- **S** - shape 0-7: `0` Tri · `1` InvTri · `2` Sine · `3` InvSine · `4` Square · `5` InvSquare · `6` Saw · `7` InvSaw.

## CHORD TABLE

Used by `=` for all 36 indices (`0`-`9` plus `a`-`z` / `A`-`Z`) and by `$` for indices 10-35 only (the chord glyphs). For `$`, indices `0`-`9` mean **scales** instead. See the Scale Table below.

Indices 10-35: lowercase glyph = root position; **UPPERCASE = first inversion** (the original root is bumped up an octave so the 3rd sits in the bass and the root sits on top: classical 1st inversion). Both `$` and `=` apply the inversion identically.

Indices 0-9 are `=`-only **octave-thickened** voicings: triads fattened with octaves above/below the root, then progressively reduced down to pure octave stacks. Pattern: Up / Down / Both → fifth → power → octave pair → octave triple. Negative intervals shift below the root; if the root is too low to fit, the note octaves up to stay in MIDI range. These don't have an inversion form (no uppercase digit glyphs).

For `=`, an empty chord port = `0 12 24` (Octaves stack: root + 1 oct + 2 oct).

| Index | Glyph | Chord / Voicing | Root pos (lowercase) | First inv (UPPERCASE) |
|---|---|---|---|---|
| 0 | `0` | Major + OctUp (`=` only) | 0 4 7 12 | - |
| 1 | `1` | Minor + OctUp (`=` only) | 0 3 7 12 | - |
| 2 | `2` | Major + OctDown (`=` only) | -12 0 4 7 | - |
| 3 | `3` | Minor + OctDown (`=` only) | -12 0 3 7 | - |
| 4 | `4` | Major Spread (`=` only) | 0 7 12 16 | - |
| 5 | `5` | Minor Spread (`=` only) | 0 7 12 15 | - |
| 6 | `6` | Fifth (`=` only) | 0 7 | - |
| 7 | `7` | Power, Fifth + OctUp (`=` only) | 0 7 12 | - |
| 8 | `8` | OctPair (`=` only) | 0 12 | - |
| 9 | `9` | OctTriple (`=` only) | -12 0 12 | - |
| 10 | `a` / `A` | Major | 0 4 7 | 4 7 12 |
| 11 | `b` / `B` | Minor | 0 3 7 | 3 7 12 |
| 12 | `c` / `C` | Sus4 | 0 5 7 | 5 7 12 |
| 13 | `d` / `D` | Sus2 | 0 2 7 | 2 7 12 |
| 14 | `e` / `E` | Major7 | 0 4 7 11 | 4 7 11 12 |
| 15 | `f` / `F` | Minor7 | 0 3 7 10 | 3 7 10 12 |
| 16 | `g` / `G` | Dom7 | 0 4 7 10 | 4 7 10 12 |
| 17 | `h` / `H` | MinorMaj7 | 0 3 7 11 | 3 7 11 12 |
| 18 | `i` / `I` | Minor6 | 0 3 7 9 | 3 7 9 12 |
| 19 | `j` / `J` | Major6 | 0 4 7 9 | 4 7 9 12 |
| 20 | `k` / `K` | Major9 | 0 4 7 11 14 | 4 7 11 14 12 |
| 21 | `l` / `L` | Minor9 | 0 3 7 10 14 | 3 7 10 14 12 |
| 22 | `m` / `M` | Major add9 | 0 4 7 14 | 4 7 14 12 |
| 23 | `n` / `N` | Minor add9 | 0 3 7 14 | 3 7 14 12 |
| 24 | `o` / `O` | Dim | 0 3 6 | 3 6 12 |
| 25 | `p` / `P` | Half Dim7 | 0 3 6 10 | 3 6 10 12 |
| 26 | `q` / `Q` | Dim7 | 0 3 6 9 | 3 6 9 12 |
| 27 | `r` / `R` | Aug | 0 4 8 | 4 8 12 |
| 28 | `s` / `S` | Aug7 | 0 4 8 10 | 4 8 10 12 |
| 29 | `t` / `T` | Dom9 | 0 4 7 10 14 | 4 7 10 14 12 |
| 30 | `u` / `U` | Dom7♭9 | 0 4 7 10 13 | 4 7 10 13 12 |
| 31 | `v` / `V` | Dom7♯9 | 0 4 7 10 15 | 4 7 10 15 12 |
| 32 | `w` / `W` | Major 6/9 | 0 4 7 9 14 | 4 7 9 14 12 |
| 33 | `x` / `X` | Minor 6/9 | 0 3 7 9 14 | 3 7 9 14 12 |
| 34 | `y` / `Y` | Minor11 | 0 3 7 10 17 | 3 7 10 17 12 |
| 35 | `z` / `Z` | Minor7♭5 | 0 3 6 10 | 3 6 10 12 |

For `=` the inverted chord is voiced ascending (notes never overlap), so the result is always a clean stacked chord. For `$` walking degrees 0..N produces those intervals as a sequential melody: degree 0 plays the new bass, and the wrapped-back original root plays an octave higher than root position would.

## SCALE TABLE

Complete `$` operator reference: every glyph the Scale/Chord port accepts. Indices `0`-`9` are scales (unique to `$`); indices 10-35 are chords shared verbatim with `=` (repeated here for convenience so you don't have to scroll to the Chord Table).

For `$`, an empty Scale/Chord port = Chromatic (all 12 semitones).

For chord glyphs (10-35): lowercase = root position, **UPPERCASE = first inversion** (3rd in bass, original root bumped up an octave). Walking degrees with `$` produces the intervals as a sequential melody.

| Index | Glyph | Scale / Chord | Root pos (lowercase) | First inv (UPPERCASE) |
|---|---|---|---|---|
| (empty) | `.` | Chromatic | 0 1 2 3 4 5 6 7 8 9 10 11 | - |
| 0 | `0` | Scale: Major | 0 2 4 5 7 9 11 | - |
| 1 | `1` | Scale: Minor | 0 2 3 5 7 8 10 | - |
| 2 | `2` | Scale: Lydian | 0 2 4 6 7 9 11 | - |
| 3 | `3` | Scale: Dorian | 0 2 3 5 7 9 10 | - |
| 4 | `4` | Scale: Mixolydian | 0 2 4 5 7 9 10 | - |
| 5 | `5` | Scale: Hirajoshi | 0 2 3 7 8 | - |
| 6 | `6` | Scale: Pentatonic | 0 2 4 7 9 | - |
| 7 | `7` | Scale: Tetratonic | 0 3 5 7 | - |
| 8 | `8` | Scale: Fifths | 0 7 | - |
| 9 | `9` | Scale: Iwato | 0 1 5 6 10 | - |
| 10 | `a` / `A` | Chord: Major | 0 4 7 | 4 7 12 |
| 11 | `b` / `B` | Chord: Minor | 0 3 7 | 3 7 12 |
| 12 | `c` / `C` | Chord: Sus4 | 0 5 7 | 5 7 12 |
| 13 | `d` / `D` | Chord: Sus2 | 0 2 7 | 2 7 12 |
| 14 | `e` / `E` | Chord: Major7 | 0 4 7 11 | 4 7 11 12 |
| 15 | `f` / `F` | Chord: Minor7 | 0 3 7 10 | 3 7 10 12 |
| 16 | `g` / `G` | Chord: Dom7 | 0 4 7 10 | 4 7 10 12 |
| 17 | `h` / `H` | Chord: MinorMaj7 | 0 3 7 11 | 3 7 11 12 |
| 18 | `i` / `I` | Chord: Minor6 | 0 3 7 9 | 3 7 9 12 |
| 19 | `j` / `J` | Chord: Major6 | 0 4 7 9 | 4 7 9 12 |
| 20 | `k` / `K` | Chord: Major9 | 0 4 7 11 14 | 4 7 11 14 12 |
| 21 | `l` / `L` | Chord: Minor9 | 0 3 7 10 14 | 3 7 10 14 12 |
| 22 | `m` / `M` | Chord: Major add9 | 0 4 7 14 | 4 7 14 12 |
| 23 | `n` / `N` | Chord: Minor add9 | 0 3 7 14 | 3 7 14 12 |
| 24 | `o` / `O` | Chord: Dim | 0 3 6 | 3 6 12 |
| 25 | `p` / `P` | Chord: Half Dim7 | 0 3 6 10 | 3 6 10 12 |
| 26 | `q` / `Q` | Chord: Dim7 | 0 3 6 9 | 3 6 9 12 |
| 27 | `r` / `R` | Chord: Aug | 0 4 8 | 4 8 12 |
| 28 | `s` / `S` | Chord: Aug7 | 0 4 8 10 | 4 8 10 12 |
| 29 | `t` / `T` | Chord: Dom9 | 0 4 7 10 14 | 4 7 10 14 12 |
| 30 | `u` / `U` | Chord: Dom7♭9 | 0 4 7 10 13 | 4 7 10 13 12 |
| 31 | `v` / `V` | Chord: Dom7♯9 | 0 4 7 10 15 | 4 7 10 15 12 |
| 32 | `w` / `W` | Chord: Major 6/9 | 0 4 7 9 14 | 4 7 9 14 12 |
| 33 | `x` / `X` | Chord: Minor 6/9 | 0 3 7 9 14 | 3 7 9 14 12 |
| 34 | `y` / `Y` | Chord: Minor11 | 0 3 7 10 17 | 3 7 10 17 12 |
| 35 | `z` / `Z` | Chord: Minor7♭5 | 0 3 6 10 | 3 6 10 12 |

---

#### Synth / MIDI output

These operators send notes and parameter changes from the grid to the engines. Channel `0` addresses the master effects bus (for `!`), while channels `1`-`8` address synth tracks T1-T8. Note operators (`;`, `:`, `=`) are silent on channel `0` since the master bus has no voices.

##### `;` - Note (poly)

Sends `NoteOn` to a synth track. Polyphonic - allocates a free voice in the track's voice pool. (Note: Grampus uses `;` for poly and `:` for mono, the opposite of ORCA, where `:` is the only note operator and `;` is the arpeggiator.)

| Operator | Channel | Octave | Note | Velocity | Length |
|:--------:|:-------:|:------:|:----:|:--------:|:------:|
|   `;`    |    C    |    O   |   N  |     V    |    L   |

- **C** - channel. `1`-`8` = synth tracks T1-T8. `0` (master) is silent - note ops can't address the master bus.
- **O** - octave (0-9). Required; if empty, no note is sent.
- **N** - note glyph. Letters `C c D d E F f G g A a B` (lowercase = sharp; letters past G wrap to higher octaves: `H` = A+12, `I` = B+12, etc.). Digits `0`-`9` map to chromatic semitones 0-9 (`0` = C, `1` = C#, `2` = D, … `9` = A) so arithmetic operators (`A`, `M`, `I`, `Z`) can drive note ports directly. A# and B above the digit range still need `a` / `B`. Empty / unknown glyphs play the channel's C.
- **V** - velocity 0-35 (scaled to MIDI 0-127). `0` = no event. Empty = full velocity (127).
- **L** - note length in ticks.

##### `:` - Note (mono)

Same ports as `;`, but monophonic: any existing note on the track is stolen immediately.

| Operator | Channel | Octave | Note | Velocity | Length |
|:--------:|:-------:|:------:|:----:|:--------:|:------:|
|   `:`    |    C    |    O   |   N  |     V    |    L   |

##### `=` - Midichord

Plays an entire chord in one tick on a synth channel. Uses the chord table above (a-z root position, A-Z first inversion) plus the octave-thickened voicings at indices 0-9.

| Operator | Channel | Octave | Root | Chord | Velocity | Length |
|:--------:|:-------:|:------:|:----:|:-----:|:--------:|:------:|
|   `=`    |    C    |    O   |   R  |   T   |     V    |    L   |

Notes are voiced ascending (each chord tone is raised by octaves if needed to stay above the previous one).

##### `^` - Strum

Plays the same chord notes as `=` but staggered in time and walked in arpeggiator-pattern order. One bang fires one full pass of the pattern, like a guitarist sweeping across the strings. Same first six ports as `=`, plus a Pattern slot (identical to `%`'s pattern table) and a Strum Speed slot.

| Operator | Channel | Octave | Root | Chord | Velocity | Length | Pattern | Speed |
|:--------:|:-------:|:------:|:----:|:-----:|:--------:|:------:|:-------:|:-----:|
|   `^`    |    C    |    O   |   R  |   T   |     V    |    L   |    P    |   S   |

- **C / O / R / T / V / L** identical to `=`. Empty chord port (`T`) falls back to octave doubling. Uppercase `T` (any chord letter) gives first inversion.
- **P** pattern, 0-13. Same table as `%` (Up / Down / Up-Down / Down-Up / Up-Down+ / Down-Up+ / Converge / Diverge / Pinky Up / Thumb Up / Up-Down Alt / Down-Up Alt / Random / Bounce). Each pattern's natural cycle plays exactly ONCE per bang. A 4-note chord with Up-Down+ produces 8 events (`1,2,3,4,4,3,2,1`); the same chord with Up-Down (no endpoint repeat) produces 6 events (`1,2,3,4,3,2`).
- **S** strum speed (per-event spacing in ticks):
  - `0` or `.` bypasses the pattern entirely. All notes fire simultaneously, identical to `=`.
  - `1` to `9` are sub-tick fractions: `1/64`, `1/48`, `1/32`, `1/24`, `1/16`, `1/12`, `1/8`, `1/6`, `1/4` of one tick. Useful for humanisation (a 1/64-tick spacing on a 5-note chord is a barely-perceptible attack flam).
  - `a` to `c` are short multiples: `1/2`, `1`, `1.5` ticks.
  - `d` to `z` are linear in ticks: `2`, `3`, `4`, `5`, ..., `24` ticks per event. The lazy end (`z` = 24 ticks per event on a 5-note chord at default tempo) lasts about a second and a half.

Random pattern (`P=c`) shuffles the chord's notes into a non-repeating permutation each bang, so every note plays exactly once but in a different order each time.

When the pattern revisits a note that's still ringing (most patterns past Up / Down hit the same notes more than once per cycle), the operator sends a clean note-off before the re-trigger, so you don't get the double-strike artefact of two voices stacking on the same pitch.

Re-banging the same `^` cell while a slow strum is still in flight cancels the queued events and starts the new strum on top. Already-ringing notes from the previous strum keep ringing until their natural Length expires, so chord changes overlap their decay tails like a real guitar.

##### `!` - SetParam

Addresses the built-in synth engines' 36-slot parameter system directly. Differs from Orca-c / bOrca, where `!` is a MIDI CC operator.

| Operator | Channel | Param | Value | Rate |
|:--------:|:-------:|:-----:|:-----:|:----:|
|   `!`    |    C    |   P   |   V   |   R  |

- **C** - channel. `0` = master effects bus. `1`-`8` = synth tracks T1-T8.
- **P** - parameter id (0-35 in base-36). See the [Parameter system](#parameter-system) section for the map.
- **V** - new value (0-35, base-36). A literal `!` glyph here means "reset this param to its default".
- **R** - interpolation rate. `0` or `.` = instant. `1`-`z` = lerp in the raw-value domain over N ticks; great for smooth sweeps that don't match any single-tick resolution.

##### `?` - If Not (inequality test)

Mirror of `F` (If). Outputs a `*` bang one cell south when the two inputs differ; otherwise the cell stays empty (`.`). Combine with `F` for if/else: `F a K` and `? a K` placed side by side fire mutually-exclusively (one always bangs, never both).

| Operator | A (west) | B (east) |
|:--------:|:--------:|:--------:|
|    `?`   |     A    |     B    |

The output cell `(0, 1)` south is stunned, so the `*` doesn't get re-evaluated as a bang operator the same tick.

---

#### Meta

##### `#` - Comment

Locks every cell to its right up to the next `#` on the same row (or end of row). Used for in-grid notes and to safely park data without it being interpreted as operators.

| Operator | … locked cells … | Closing |
|:--------:|:----------------:|:-------:|
|   `#`    |  this is a note  |   `#`   |

##### `*` - Bang

Transient activation signal. Auto-clears one tick after it is placed or produced.

---

**Lowercase `d`, `u`, `f`** use a special "bang excluding south" rule: they ignore a bang arriving from the south, so their own south-facing bang output can't feed back into themselves and cause self-sustaining loops.

---

## Synth engines

Each of the eight tracks runs one synth engine at a time. Switching the engine on a track (`EngType`, param 0) swaps the voice and restores the last preset you had on that engine for that track, so you can A/B engines without losing your tweaks. Polyphony is **8 voices per track** for every engine type; the same envelope, filter, drive, and routing follow regardless of which engine you pick.

Nine engines ship today:

### Drifter (EngType 0)

Dual oscillator virtual-analog. Two PolyBLEP-anti-aliased oscillators (A and B) each pass through a phase-distortion stage, get crossfaded together, optionally summed with a sub-oscillator, and panned for stereo width. Slow Brownian pitch drift gives the static-DCO sound a living, analog feel. The shape control is a continuous morph through **triangle → saw → square** with **saw at the centre**, which is why Shape A and Shape B render as bipolar sliders centred at `h`.

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | ShapeA | bipolar | Continuous morph for Osc A. **`0` = pure triangle**, **`h` = pure saw (centre)**, **`y` = pure square**. Anything in between linearly interpolates between the two adjacent shapes (e.g. `8` is roughly halfway between tri and saw). |
| 2 | SkewA | bipolar | Phase-distortion knee for Osc A. **`h` = no skew (symmetric waveform)**. Below `h` skews the cycle so the first half is shorter than the second; above `h` does the opposite. Adds odd-order harmonics - for square waves it acts like PWM, for tri/saw it asymmetrises the timbre. |
| 3 | ShapeB | bipolar | Same shape morph as ShapeA, applied independently to Osc B. Useful for layering two different waveforms (e.g. ShapeA at saw `h`, ShapeB at square `y`). |
| 4 | SkewB | bipolar | Skew on Osc B. Independent from SkewA. |
| 5 | BPitch | unipolar | Osc B pitch offset above Osc A, **1 semitone per glyph (0 to +35 semitones)**. `0` = unison, `7` = +7 semi (perfect 5th), `c` (12) = +1 octave, `o` (24) = +2 octaves, `z` = +35 (~3 octaves). For unison detune, leave it at `0` and use Drift; for chord-stab intervals, set explicit semitone values. |
| 6 | Xfade | bipolar | A/B level balance. **`0` = Osc A only**, **`h` = equal mix (centre)**, **`y` = Osc B only**. |
| 7 | Drift | unipolar | Brownian pitch instability - both oscillators wander slightly off-pitch over ~200-350 ms via a Gaussian random walk with a one-pole smoother. **`0` = locked pitch**, **`z` ≈ ±25 cents max wander**. Each oscillator drifts independently, so light Drift makes a single voice sound like two slightly-out-of-tune oscillators (the natural source of "analog" warmth). |
| 8 | SubOsc | bipolar | Sub oscillator level + octave selector. **`h` = off**. Below `h`: sub is **two octaves below** Osc A, magnitude controls its level. Above `h`: sub is **one octave below**, magnitude controls level. Triangle waveform internally - sits clean under the main oscillators. |
| 9 | Stereo | bipolar | Stereo spread by panning A and B in opposite directions via equal-power pan. **`h` = both centred (mono)**. Magnitude controls how hard they push to L/R. Combine with detune (BPitch + Drift) and/or per-osc shape differences for instant stereo width without an external chorus. |

**Common combos:**
- *Classic detuned saw lead*: ShapeA `h` (saw), ShapeB `h`, BPitch `0` (unison), Xfade `h`, Drift `8`. For a fifth-up oscillator instead, set BPitch `7`.
- *Hollow square pad*: ShapeA `t` (mostly square), ShapeB same, mild Skew differences (SkewA `m`, SkewB `g`), Stereo `r`, Drift `e`.
- *Sub-bass*: ShapeA `b` (mostly triangle), ShapeB same, Xfade `h`, SubOsc `4` (two-octave-down at moderate level).
- *"Living" pad*: heavy Drift (`m`-`r`), differentiated Skew between A and B, max Stereo, SubOsc `t` (one-octave-down).

Character: warm, classic analog.

### Warper (EngType 1)

Dual wavetable oscillator inspired by [Vital](https://vital.audio/). Each oscillator picks from **6 wavetables**, then runs through an independently-selectable **warp mode** that bends the phase before the table is read. Unlike Drifter's symmetric tri ↔ saw ↔ square model, Warper's shape control sweeps **linearly through an ordered list of 6 distinct wavetables** - there's no natural "centre" shape to anchor a bipolar slider on, so Shape is unipolar (`0` low → `z` high).

**Shape value → wavetable mapping** (smoothly morphed between adjacent entries):

| Glyph | Wavetable |
|-------|-----------|
| `0`   | Sine - pure fundamental |
| `7`   | SatSin - saturated sine, warm even-order harmonics |
| `e`   | Tri - triangle |
| `l`   | Square - full square |
| `s`   | Pulse - 25% duty-cycle pulse |
| `z`   | Saw - sawtooth |

Anything in between (e.g. `4` or `j`) crossfades between the two adjacent tables. So `4` is roughly halfway between Sine and SatSin; `j` is roughly halfway between Tri and Square. The morph is continuous, so you can park anywhere on the spectrum.

**Warp modes** (WrpMdA and WrpMdB independently pick one of these, discrete; the slider shows the mode name). The Warp slider is **bipolar**, `h` (centre) is identity (raw wavetable, no warp). Five modes warp the *phase* before the wavetable read (shape-aware), two warp the *amplitude* after the read (shape-agnostic):

| Glyph | Mode | Domain | What it does | Slider behaviour |
|-------|------|--------|--------------|------------------|
| `0` | **Bend** | phase | Phase exponent (`power = 8^amount`). Stretches one half of the cycle and compresses the other. | **bipolar**, below `h` compresses the end (concave), above `h` compresses the start (convex). |
| `1` | **Sync** | phase | Hard-sync slave at variable rate. Negative side interpolates 0.25× to 1× (glyph `8` lands exactly on 0.5×); positive side snaps to integer mults so each glyph above `h` is +1× (glyph `h+1` = 2×, `h+15` = 16×, `h+16` and beyond clamp to 16×). | **bipolar with positive-side snap**. |
| `2` | **Formant** | phase + amp | Same Sync ratio as above plus a half-sine amplitude window for vocal "ah/oh" character. The window narrows with warp magnitude so high settings sound like a formant burst. | **bipolar**, identical snap to Sync. |
| `3` | **Squeeze** | phase | Piecewise-linear pivot that shifts the cycle's midpoint. Both halves stay linear, only the slope rates diverge. | **bipolar**, below `h` shifts midpoint left (compress start), above `h` shifts right (compress end). |
| `4` | **Fold** | amp | Wavefolder. Drives the output (up to 16× at extremes) and reflects anything past ±1. Positive side: symmetric triangle fold (Buchla-style buzz, all-odd harmonics). Negative side: DC-biased asymmetric fold (warm, vocal/cuica character with even harmonics on top of the odd ones). | **bipolar with two flavors**, sign picks character, magnitude controls drive. |
| `5` | **Quantize** | amp | Bitcrush. Snaps the output to multiples of a step that grows with magnitude. Low: subtle stair-step. Mid: 5-level reduction. Extreme: near silence punctuated by impulses at the wavetable's peak excursions. | **magnitude**, sign ignored (both halves of the slider give the same effect). |
| `6` | **Pulse** | phase | Split-cycle warp. Compresses the wavetable's first half into the left edge of the cycle and the second half into the right edge, holding the midpoint in between. Shape-aware: on sine produces "hump + flat 0 + hump", on saw produces "ramp + flat 0 + ramp", on square produces "+1 + 0 + -1". | **bipolar**, positive holds at the wavetable midpoint, negative holds at the cycle boundary (start/end of wavetable). |

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | ShapA | unipolar | Osc A wavetable selector (see mapping above). |
| 2 | WarpA | bipolar | Warp depth/direction for Osc A. **`h` = no warp**, deflection in either direction engages the warp. The mode (above) decides whether sign matters. |
| 3 | WrpMdA | discrete (0-6) | Warp mode for Osc A. |
| 4 | ShapB | unipolar | Osc B wavetable selector. Independent from ShapA, so you can layer e.g. Sine + Saw. |
| 5 | WarpB | bipolar | Warp depth/direction for Osc B. Same semantics as WarpA. |
| 6 | WrpMdB | discrete (0-6) | Warp mode for Osc B. Independent from WrpMdA, so you can have Bend on A and Pulse on B simultaneously. |
| 7 | BPitch | unipolar | Osc B pitch offset above Osc A, **1 semitone per glyph (0 to +35 semitones)**. `0` = unison, `7` = +7 semitones (perfect 5th), `c` (12) = +1 octave, `o` (24) = +2 octaves, `z` = +35 (~3 octaves). Only goes up. |
| 8 | Xfade | bipolar | A/B crossfade. **`0` = Osc A only**, **`h` = equal mix**, **`y` = Osc B only**. |
| 9 | Detune | unipolar | Unison detune + stereo spread combined. `0` = mono / no detune; magnitude raises both detune amount (up to ±25 cents) and the stereo width of the detuned copies. A second detuned voice of each oscillator pans opposite the original. |

**Common combos:**
- *Buchla-buzz lead*: ShapA `l` (square), WarpA `z`, WrpMdA `4` (Fold positive), Xfade `h`, Detune `4`. Aggressive metallic timbre, all-odd harmonics.
- *Vocal cuica pad*: ShapA `e` (tri), WarpA `4` (negative side), WrpMdA `4` (Fold negative), Detune `g`. Warmer asymmetric fold with even harmonics, hollow vocal character.
- *Vocal formant pad*: ShapA `z` (saw), WarpA `t`, WrpMdA `2` (Formant), ShapB `e` (tri), WrpMdB `0` (Bend), Detune `g`.
- *Bitcrushed sine*: ShapA `0` (sine), WarpA `r`, WrpMdA `5` (Quantize). Mid-amount = 5-level stair-step reduction; push to `z` for near-silence with impulse spikes at the sine peaks.
- *Pulse-width on sine*: ShapA `0` (sine), WrpMdA `6` (Pulse), sweep WarpA from `j` (positive, narrowing humps with growing flat zero) toward `y` (almost-silent flat with brief ±1 flicks). Negative side flips the hold to the boundary.
- *Octave-up sync lead*: ShapA `z` (saw), WarpA `i` (h+1 = exactly 2×), WrpMdA `1` (Sync), Xfade `8`.
- *Sub-octave sync*: ShapA `z` (saw), WarpA `8` (= exactly 0.5× mult), WrpMdA `1` (Sync). Slave runs half-speed, octave-down sync flavour.
- *Bend pulled both ways*: WrpMdA `0` (Bend), try WarpA `0` (full concave) and WarpA `z` (full convex) on a saw to hear the bipolar character.

> Filter tip: warp modes (especially Sync, Fold, Formant, Pulse) generate a lot of high-frequency content. If FltCut is at the default `p` (~8 kHz) you'll hear maybe a third of what they actually produce, try sweeping FltCut up to `y` or `z` while auditioning warp settings.

Character: modern digital wavetable. Can go from vintage-sounding (square + fold) to bitcrushed (sine + quantize) to vocal (saw + formant or tri + Fold negative) to PWM-leaning (any shape + Pulse) without leaving one engine.

### BrokenFM (EngType 2)

A 6-operator FM synth that ships with **21 presets** covering pads, leads, basses, EPs, brass, strings, percussion, FX, and vocal-style tones. Instead of asking you to design operator routings from scratch (which is the brutal part of FM), the user surface is a **preset selector + 4 morphing macros**. `h` on every macro = the preset is preserved exactly; deflect a macro to morph the preset in a meaningful direction. Each preset has its own internal sensitivity tuning so subtle presets stay subtle even when you push macros hard.

**Shipped presets**, indexed by Flavor `0..14` (the slider shows the name):

| Glyph | Name | Glyph | Name | Glyph | Name |
|-------|------|-------|------|-------|------|
| `0` | HARPY (Karplus-like pluck) | `7` | PIANIC (FM piano) | `e` | CEREMONY (orchestral hit) |
| `1` | ZUITAR (FM guitar) | `8` | PLUCKY (sharp pluck) | `f` | KICKFLIP (percussive) |
| `2` | MYSTPAD (atmospheric pad) | `9` | DESIRE (warm pad) | `g` | PLATES (metallic) |
| `3` | FELTPAD (soft pad) | `a` | DEEPER (sub bass) | `h` | EXPLODE (FX) |
| `4` | ZOEY (vocal-ish lead) | `b` | JOURNEY (evolving pad) | `i` | TOPHAT (hi-hat-ish) |
| `5` | SAWSPOR (saw lead) | `c` | JOKER (FX) | `j` | GHOSTS (atmospheric) |
| `6` | BELLKEY (bell EP) | `d` | GALACTIC (sweeping) | `k` | SPACEWAR (FX/lead) |

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | Flavor | discrete | Preset selector - pick one of the 21 presets above. |
| 2 | Length | bipolar | Symmetric exponential scaler over **every per-operator envelope** (attack, decay, release together). **`0` = ×0.2** (sharp / percussive - fast attacks, fast release), **`h` = preset's natural lengths**, **`y` = ×5** (slow / swelly - perfect for turning a pluck preset into a pad). |
| 3 | Heat | bipolar | Modulator intensity multiplier - controls how much modulator signal hits each carrier. **`0` = ×0.20** (cool / clean / nearly pure carrier - useful for soft EP tones), **`h` = preset**, **`y` = ×5** (hot / chaotic FM - sidebands explode into noise). Exponential, no dead zone around `h`. |
| 4 | Color | bipolar | Shifts every modulator's ratio relative to its carrier. **Anchors** at `h ± 0/4/8/12/16/17` land on musically valid ratios (Digitone-style quantised table: 0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3, 4, 5, 6, 7, 8, 9, 11, 13, 16…). In-between values get ±35 / ±70 cents detune offsets - slightly off-musical but still useful for inharmonic textures. **`h` = preset's stock ratios**, push toward `0` or `z` for higher-numbered ratios in either direction. |
| 5 | Feedback | bipolar | Self-feedback added to the highest-level modulator. **`h` = preset's natural feedback** (0 for most, 0.5 for PIANIC). **`y` = positive feedback** (growl / noise / chaos), **`0` = negative feedback** (inverted harmonics - phaser-like, hollow, unique to FM). Useful for adding grit or dialing in formant-like resonances. |

**Common combos:**
- *Slow swelling pad from a pluck*: Flavor `0` (HARPY), Length `y` (×5 envelopes), Heat `h`, Color `h`, Feedback `h`.
- *Aggressive FM growl*: any flavor, Heat `y` + Feedback `y`, Color `0` or `z` for extra bite.
- *Phaser-like sweep*: any flavor, Color `0` (negative feedback inverts harmonics), Length `m`-`o` for slower attacks.
- *Detuned/inharmonic FX*: Flavor `c` (JOKER) or `k` (SPACEWAR), Color anywhere away from anchor points (e.g. `5` or `m`) for slight detune flavour.

Character: anything from clean DX-style EPs and basses to noisy growls, plucked strings, bell tones, vocal/formant textures, and inharmonic FX. Every preset is musical at default macros; extremes give very different but still controlled sounds.

### String / Karplus (EngType 3)

Karplus-Strong physical-modelling pluck. Two detuned strings excited by a hammer/noise burst, fed back through a 4-stage cascaded allpass network that generates the pitch-dependent inharmonicity of real strings. Two sympathetic strings (tuned to the octave and fifth) ring along quietly for body. Damping scales with pitch so basses ring longer than trebles, matching real-instrument behaviour without manual tuning per note.

The engine internally runs two parallel feedback polarities - one for "string" (positive feedback, harmonic-rich pluck) and one for "bell" (negative feedback, metallic inharmonic ringing) - and blends between them via the Str→Bell crossfade.

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | Bright | unipolar | Feedback-loop lowpass cutoff. **`0` = very dark / muffled**, **`z` = very bright / cutting**. Velocity adds a small brightness bump (~15% of velocity) so harder hits sound brighter, like real strings. |
| 2 | Decay | unipolar | Feedback amount = how long the string rings before the loop's natural losses silence it. **`0` ≈ 0.99 feedback (~0.5s ring)**, **`z` ≈ 0.999 feedback (multi-second ring)**. The exponential approach to 1.0 means high values get dramatically longer. |
| 3 | StrBel | unipolar | **Continuous crossfade** between string (positive-polarity feedback) and bell (negative-polarity feedback) timbres. **`0` = full string** (regular Karplus pluck, harmonic), **`h` = 50/50 blend**, **`y` = full bell** (inverted feedback, sounds metallic / inharmonic). Smooth all the way through - nothing binary about it despite the name. |
| 4 | Detune | unipolar | Cents spread between the two primary strings. **`0` = no detune (mono)**, **`z` ≈ ±25 cents spread** (lush chorus-like detune). At any non-zero value, the two strings also spread to opposite sides of the stereo field automatically. |
| 5 | Hammer | unipolar | Length of the excitation burst at note-on, scaled exponentially. **`0` = very short / percussive (sharp pluck)**, **`z` = long / soft (slow bow / mallet)**. Short hammers emphasise transient harmonics; long hammers smooth them out. |
| 6 | Inharm | unipolar | 4-stage allpass coefficient that simulates the slight pitch shift higher partials acquire on stiff strings (piano-like). **`0` = ideal string (perfectly harmonic partials)**, **`z` = heavy inharmonicity** (bell / detuned-tine character). Scales with pitch automatically - high notes get more inharmonic. |
| 7 | Stiff | unipolar | How strongly pitch affects damping. **`0` = even decay across the keyboard**, **`z` = bass rings far longer than treble** (real-piano style). Combine with Decay to get long bass thumps that don't make treble notes ring forever. |
| 8 | Color | unipolar | Excitation mix between half-sine and white noise. **`0` = pure half-sine** (clean tonal pluck - piano hammer), **`z` = pure noise** (breathy, harpsichord / bow attack). |
| 9 | Stereo | bipolar | Stereo distribution of the four internal strings (2 primary + 2 sympathetic). **`h` = balanced centre**. Magnitude pushes the strings apart in the field. Combine with Detune for wide chorus sounds. |

**Common combos:**
- *Acoustic guitar pluck*: Bright `r`, Decay `t`, StrBel `0` (string), Detune `4`, Hammer `7`, Inharm `4`, Stiff `c`, Color `5`, Stereo `n`.
- *Tubular bell / metallic*: Bright `n`, Decay `w`, StrBel `r` (mostly bell), Detune `0`, Hammer `2`, Inharm `s`, Stiff `0`, Color `0`, Stereo `e`.
- *Soft piano-ish*: Bright `o`, Decay `s` (long bass tail), StrBel `0`, Detune `2`, Hammer `b`, Inharm `7` (light inharm), Stiff `m` (real-piano damping curve), Color `4`.

Character: plucked, struck, bell-like. Very responsive to velocity (boosts brightness) and register (Stiff modulates decay by pitch).

### ROMpler (EngType 4)

Sample-playback engine. Drum kits and pitched instruments share the same 6-parameter interface; the shipped kits cover both. Pitch zones are auto-constructed from the samples in each kit, so any note you play picks the closest sample and pitches it from there. Linear interpolation between adjacent samples keeps pitch shifts clean across the playable range; the `8bit` toggle is the place to add lo-fi character on demand.

> **Drum kits and per-octave sample switching.** For the drum-focused kits (`909`, `808`, `REALDRUM`, `AMEN`), each kit slot occupies a full octave. `C` plays the original (unpitched) sample; every other note up to (but not including) the next `C` is the same sample pitched. So `C..B` in octave 3 is one sample (pitched up across the octave), and `C` in octave 4 picks the next slot's sample. This makes it easy to lay drums out chromatically, pick the kick at one `C`, snare at the next `C`, hats at the next, etc.

**Shipped kits (17)**, indexed by ROM 0..16:

| ROM | Name | Type |
|-----|------|------|
| 0 | 909 | Drum machine (40 kHz MPC-style bake) |
| 1 | 808 | Drum machine (40 kHz MPC-style bake) |
| 2 | REALDRUM | Acoustic drum kit |
| 3 | AMEN | Classic break, sliced (40 kHz MPC-style bake) |
| 4 | PIANO | Pitched, multi-sampled |
| 5 | PIASOFT | Soft-pressed piano, multi-sampled (24 kHz) |
| 6 | RHODES | Pitched, multi-sampled |
| 7 | HAMMOND | Pitched, looped |
| 8 | STRINGS | Pitched, looped |
| 9 | BRASS | Pitched, looped |
| 10 | GUITAR | Pitched, multi-sampled |
| 11 | SINE | Chromatic 7-octave waveform bank, looped |
| 12 | TRIANGLE | Chromatic 7-octave waveform bank, looped |
| 13 | SQUARE | Chromatic 7-octave waveform bank, looped |
| 14 | SAW | Chromatic 7-octave waveform bank, looped |
| 15 | NOISYSAW | Chromatic 7-octave waveform bank, looped (noisy variant) |
| 16 | ZEN | Sustained pad, looped (24 kHz) |

| # | Name | Role |
|---|------|------|
| 1 | ROM | Kit selector. |
| 2 | Pitch | Coarse tune ±17 semitones (1 base-36 step = 1 semitone). |
| 3 | Tune | Fine tune ±50 cents. |
| 4 | Reverse | 0 = forward, 1 = reversed playback. |
| 5 | VelCut | Velocity → filter cutoff modulator. Bipolar. `h` = no effect. Stacks with FltCut, FEnvAm, and any LFOs that target FltCut. |
| 6 | 8bit | 0 = clean, 1 = quantise output to 8-bit (256 levels). Live bitcrush, applied AFTER pitch shifting so it crunches the audible signal, not the source data. |

Note-off behaviour depends on the kit:
- **Drum / one-shot kits**: note-off is ignored, the sample plays through. If you want the tail cut, use the Amp envelope (`AmpHld` / `AmpDcy`).
- **Pitched / looped kits**: note-off triggers the standard Amp envelope release.

**Live pitch / tune modulation**: `Pitch` (slot 2) and `Tune` (slot 3) recompute the playback rate every time they change during a sustained note, so they respond to LFO mod, `!` operator writes, and `!`-with-interpolation lerps in real time. This lets you build pitch envelopes (one-shot LFO on Tune or Pitch) and smooth pitch glides (`!` on Pitch with interpolation > 0). Static patches (no modulation) behave identically to before, the per-note rate is still latched at note-on; only the live-update path is new.

Character: classic sampler. Drum kits hit hard; pitched instruments stretch through pitch zones with that "tape-slowed" character at extremes.

---

### LoFiTron (EngType 5)

A 1:1 clone of ROMpler, reading from a separate **TAPE** pool instead of the ROM pool. Same parameter interface, same pitch / tune / reverse / VelCut / 8bit behaviour, slot 1 is just labelled `TAPE` (not `ROM`) and the discrete selector picks from a separate sample pool.

The intent is to keep two independent sample banks that you can mix and match per track without fighting for ROM slots: ROMpler tends to hold drums + clean melodic content, LoFiTron tends to hold lo-fi sustained / atmospheric material with a darker, downsampled character.

**Shipped TAPEs (7)**, indexed by TAPE 0..6:

| TAPE | Name | Type |
|------|------|------|
| 0 | XTRINGS | Wide-range string layer mix, looped |
| 1 | 3VIOLINS | Three-violin layer, looped |
| 2 | ORGAN | Sustained organ, looped |
| 3 | VIBES | Vibraphone, one-shot natural decay |
| 4 | CHOIRMIX | Choir layer mix, looped |
| 5 | BRASS | Soft brass layer, looped |
| 6 | FLUTE | Flute, looped |

The TAPE selector and ROM selector are independent, adding a TAPE never bumps a ROM index, and vice versa.

---

### WaveSurf (EngType 6)

A 2D wavetable morph engine. Picks two wavetables from the available pool, scans through frames in each, and crossfades between them to produce an evolving terrain. Spiritual descendant of Korg Wavestation / Sequential Prophet VS vector synthesis, where each "corner" is a fully scannable wavetable rather than a static waveform.

| ID | Slot | Notes |
|----|------|-------|
| 1 | WaveX | Picks one of the available wavetables |
| 2 | WaveY | Picks one of the available wavetables |
| 3 | ScanX | Frame position inside WaveX (`0` = first frame, `z` = last frame) |
| 4 | ScanY | Frame position inside WaveY, doubles as the X↔Y morph weight |
| 5 | PitchX | Per-axis pitch offset for WaveX (bipolar ±17 semitones, `h` = unison) |
| 6 | PitchY | Per-axis pitch offset for WaveY (bipolar ±17 semitones, `h` = unison) |

**Asymmetric bilinear morph**: at `ScanY = 0` you hear pure `WaveX` at frame `ScanX`; at `ScanY = z` you hear pure `WaveY` at its last frame. Sweeping `ScanY` simultaneously scans through `WaveY`'s frames AND morphs from X to Y. ScanX is independent, it only selects which frame of WaveX is in the mix.

**Smooth modulation**: all scan/wave params smooth at audio rate (~50 ms time constant) so base-36 modulation (e.g. `!` writes from the sequencer every tick) glides between values rather than zippering.

**Wavetable swaps** (changing WaveX or WaveY) crossfade over ~10 ms instead of cutting. Mid-note picker changes are click-free.

**Per-axis pitch with dual-resolution semantics**:
- **Manual / `!` writes** (integer base-36 input) → exact integer semitones. `PitchX = 18` is `+1` semi, `PitchX = 5` is `-12` semi (an octave down), etc. Use for octaves, fifths, sub-foundations.
- **LFO modulation** → continuous fractional semitones. A slow low-depth LFO on `PitchX` (e.g. rate `8`, dep `i`) gives a wandering ±0.05 semi drift, classic chorus-style detune without using the chorus effect.

At unison (default, both pitches = `h`) the two oscillators run on the same phase and the morph is a pure timbral blend. At non-unison they're two independently-pitched oscillators crossfaded by ScanY (octaves, fifths, detune, layered intervals), same model as Vital's Osc A/B transpose.

---

### SynDrums (EngType 7)

A drum-synthesis voice with five built-in models (kick, snare, clap, toms, cymbals), each with their own algorithm and parameter mapping. On top of the synthesised body sits a transient-sample player (one of 29 baked transient hits in the pool) that crossfades with the synth via the BAL slider. Per-hit micro-detune (±0.5%) and velocity-scaled decay are applied automatically so consecutive hits don't sound mechanically identical.

| ID | Slot | Notes |
|----|------|-------|
| 1 | MODEL | Picks the drum model: BD / SD / CLAP / TOMS / CYMBALS |
| 2 | TRANSNT | Picks one of the baked transient samples (kick clicks, snare hits, hi-hat sticks, etc.) |
| 3 | BAL | Bipolar synth↔transient mix (`a` = body-favoured, `h` = equal mix, `z` = pure transient) |
| 4 | SHAPE | Per-model character (see table below) |
| 5 | DECAY | Per-model decay / morph (also gently scaled by note velocity for realism) |
| 6 | TONE | Per-model timbre / brightness |
| 7 | VIBE | Per-model: OUT/AUX algorithm crossfade for kick/snare/cymbals; burst spacing for clap |
| 8 | COLOR | Per-model post-FX (saturation / rattle / shimmer reverb, see table below) |

**SHAPE / TONE / DECAY / VIBE per model:**

| Model | SHAPE | TONE | DECAY | VIBE |
|-------|-------|------|-------|------|
| **BD** | Attack sharpness + amount of overdrive | Body brightness | Body decay time | Crossfade between two parallel kick algorithms (analog 808-style at one extreme, FM kick at the other) |
| **SD** | Balance between the harmonic body and the noisy snare-wires component | Balance between the drum's shell modes | Body decay time | Crossfade between two parallel snare algorithms (resonant shell+bandpass-noise at one extreme, FM-sine pair + HP-noise at the other) |
| **CLAP** | Centre frequency of the resonant peak in the noise (~800 Hz at `0`, ~13 kHz at `z`); also pitch-tracks the played note (one octave per octave of input) | Final low-pass cutoff (~3 kHz at `0`, ~12 kHz at `z`) | Length of the long tail after the burst sequence (~50 ms at `0`, ~320 ms at `z`) | Spacing between the four noise bursts that make up the clap (~5 ms tight at `0`, ~26 ms wide at `z`) |
| **TOMS** | Same as BD (attack sharpness + drive); pitched up an octave-ish from kick territory | Same as BD (body brightness) | Same as BD (body decay) | Same as BD (analog ↔ FM crossfade) |
| **CYMBALS** | Balance of the metallic and filtered-noise components | High-pass filter cutoff (lower = darker, higher = thinner) | Decay length | Crossfade between two parallel hi-hat algorithms (six squares + dirty VCA at one extreme, ring-modulated square pairs + clean VCA at the other) |

**COLOR per model (distinct post-FX, all at zero by default):**

| Model | COLOR effect |
|-------|-----------|
| **BD / SD / CLAP** | Soft-clip saturation. Adds harmonic warmth without extra loudness. Overdrives the body more aggressively as you turn it up. |
| **TOMS** | Subtle short HP-filtered noise burst that decays alongside the body, adding a "rattle" character. The decay length grows quickly only at the top of the slot, so low/mid values give a sharp snap and only the highest values open into a longer rattle. |
| **CYMBALS** | Built-in shimmer reverb (per voice). COLOR controls both the reverb feedback (decay length) and the wet mix, so a single knob takes you from dry hi-hat to long lush cymbal sustain. Each voice has its own reverb so different cymbal hits don't smear into one tank. |

**Notes**

- The TRANSNT picker stores its choice by stable name, so reordering or renaming the transient pool in future updates won't break older patches.
- The MODEL choice is also stored by stable name (BD / SD / CLAP / TOMS / CYMBALS), so you can safely save a patch on one model and the slot will resolve correctly even if model order changes.
- Loud hits decay slightly longer than soft hits (linear ±30% velocity scaling on DECAY); small acoustic-realism cue across all models.
- Each new note adds a tiny ±0.5% pitch jitter to both the synth body (where applicable) and the transient sample, so machine-gun repeats don't sound mechanically identical.

---

### BoneSaw (EngType 8)

A **Roland JP-8000-inspired SuperSaw** engine. Seven detuned saws (one centre + six sides at fixed Roland-derived ratios) summed together, a sub-oscillator for low-end weight, and a per-saw hard-sync section that goes well past anything the original instrument could produce. The detune curve is the actual JP-8000 polynomial measured by Adam Szabo, applied as a Hz offset rather than a pitch ratio - which is why the same SWARM setting feels wider on low notes than on high notes. That uneven-across-the-keyboard character is the signature of a real SuperSaw: huge fat bass, focused upper register.

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | SHIFT | bipolar | Volume balance between the three negative-detune sides and the three positive-detune sides. **`h` = balanced (equal both groups)**. Below `h` favours the negative-detune sides (slightly flatter ensemble); above `h` favours the positive-detune sides (slightly sharper ensemble). The centre saw is unaffected. Subtle on its own; mainly useful with STEREO for asymmetric stereo placement of the ensemble. |
| 2 | SWARM | unipolar | Detune amount across all six side oscillators. **`0` = unison (all 7 saws at the same pitch, super-thick mono saw)**, **`h` = classic JP-8000 SuperSaw spread (~8 cents at A4, ~55 cents at C2 - the sound of every '90s trance lead)**, **`z` = max width (~20 cents at A4, ~133 cents at C2 - wider than the original instrument permits)**. The detune amount in cents grows automatically as you go down the keyboard, so bass notes feel huge while treble notes stay focused. |
| 3 | STEREO | bipolar | Pans the side saws across L/R based on the sign of their detune coefficient. **`h` = mono centre (all sides on top of each other)**. Magnitude pushes the negatives to one channel and positives to the other; polarity (above/below `h`) decides which group goes which way. Centre saw stays mono. Combine with SWARM > 0 for instant wide stereo SuperSaw pads. |
| 4 | SUBOSC | bipolar | Sub-oscillator level + octave selector. **`h` = off (no sub)**. Below `h`: sub is **two octaves below** the played note, magnitude controls level. Above `h`: sub is **one octave below**, magnitude controls level. The sub is intentionally NOT detuned and NOT synced - it stays clean and centred to keep the low end rock-solid even when the supersaw above it is wide and chaotic. |
| 5 | SHAPE | unipolar | Saw → square morph applied to all 8 oscillators (7 saws + sub). **`0` = pure saw (the classic SuperSaw timbre)**, **`z` = pure square (more hollow, PWM-pad-ish character)**. Anywhere between linearly crossfades. Subtle deflections feel like the saw has a slight hollowness; full deflection is genuinely a different timbre - try it on pads. |
| 6 | SYNC | unipolar | Per-saw hard sync. Each of the 7 saws gets a hidden slave oscillator running 1× to 4× its own frequency; the slave's phase resets every time its master saw cycles. **`0` = no sync (identical to a vanilla SuperSaw)**, **`z` = max sync ratio (4×, aggressive resonant sweep with a metallic "talking" character)**. Stays tonal at every setting - perceived pitch is locked to each saw's own frequency. This knob is what differentiates BoneSaw from a stock SuperSaw clone: a full 0-z SYNC sweep with FltCut wide open is the BoneSaw signature move. |

**Common combos:**
- *Classic JP-8000 trance lead*: SWARM `h`, STEREO `q`, SUBOSC `h`, SHAPE `0`, SYNC `0`. Long Amp release, FltCut wide open. The "Children of '95" sound.
- *Ultra-wide SuperSaw pad*: SWARM `s`, STEREO `z`, SUBOSC `m` (one-octave sub at moderate level), SHAPE `5` (slight squareness), SYNC `0`. Slow Amp attack (`AmpAtk` `j`), long release.
- *Resonant sync lead*: SWARM `e` (tight unison-ish), STEREO `h`, SUBOSC `h`, SHAPE `0`, SYNC `r`. Sweep FltCut while playing for the classic sync filter sweep.
- *Sub bass + thick saw*: SWARM `0` (true unison - just one fat saw), SUBOSC `c` (two-octave sub at strong level), SHAPE `0`, SYNC `0`. BoneSaw functions as a single-osc-with-sub bass synth at SWARM=0.
- *Aggressive hardstyle stab*: SWARM `s`, STEREO `q`, SUBOSC `q`, SHAPE `0`, SYNC `n`. Snappy Amp envelope (`AmpAtk` `0`, `AmpHld` `4`, `AmpDcy` `0`), heavy Drive on top.
- *PVox + BoneSaw signature*: BoneSaw at any combo above + `FltTyp` `9` (PVox BP), `FltRes` mid-to-high. The Polivoks BP filter on top of a synced supersaw is genuinely unhinged.

> **Sweet spot tip.** SWARM `h` is calibrated to feel like a stock JP-8000 SuperSaw at typical detune-knob position - the kind of sound that defined trance and progressive house. Below `h` the ensemble tightens up toward unison; above `h` it goes wider than the original synth ever permitted, into territory that feels almost like a chord. The whole knob is musical, but `h` is "home base".

Character: classic '90s SuperSaw at SWARM `h`, focused mono saw at SWARM `0`, otherworldly wide chord-like ensemble above SWARM `h`. SYNC adds a metallic resonant sweep that the original JP-8000 couldn't do on its own. Sub-oscillator is rock-solid clean regardless of what the saws above it are doing.

---

## Shared voice chain

Every voice, regardless of type, passes through the same chain:

- **AHD envelope** (params 10-12: AmpAtk / AmpHld / AmpDcy)
  - Hand-picked time curve, dense in the snappy region: see the [Envelope time table](#envelope-time-table) below.
  - **`AmpHld = 0` → INF (gate mode)**: level stays at peak until note-off arrives, then Decay runs. Default. Use for sustained notes, pads, leads.
  - **`AmpHld = 1..z` → AHD (auto-decay)**: Atk → Hld (dwell) → Dcy fires automatically regardless of note length. Use for drums, plucks, blips, the envelope shape defines the sound duration.
- **True-stereo MultiFilter** (params 13-15: FltTyp / FltCut / FltRes) - independent L/R filter instances:
  - 0 Off (passthrough)
  - 1 **Moog LP** (4-pole ladder, 24 dB/oct) - the classic warm transistor-ladder character.
  - 2 **SVF HP** (2-pole, 12 dB/oct)
  - 3 **SVF BP** (2-pole, 12 dB/oct)
  - 4 **SVF Notch** (cascaded 4-pole, 24 dB/oct)
  - 5 **LP>HP** (serial 1-pole LP → 1-pole HP - band-pass-ish)
  - 6 **DJ Filter** (bipolar: cutoff center = bypass, below = LP, above = HP)
  - 7 **PVox LP** - Polivoks-inspired modelling filter, low-pass tap.
  - 8 **PVox HP** - same Polivoks topology, high-pass tap.
  - 9 **PVox BP** - same Polivoks topology, band-pass tap (the most overtly resonant of the three).
  - Cutoff: 20 Hz - ~20 kHz mapped across 0-z. Parameter smoothing (~5 ms one-pole) eliminates zipper noise.

  > **About the PVox family.** Inspired by the Soviet Formanta Polivoks - a synth famous for the snarling vocal "bite" of its filter. The defining feature is an **asymmetric diode-pair feedback path** that replaces the linear resonance loop of a normal SVF. At low FltRes the filter is gentle and clean; as you push FltRes up, the feedback path saturates harder and harder, FltRes acts as both Q **and** drive at the same time. The asymmetric clipping shape leaks even-order harmonics into the resonance, which is what produces the singing, almost-vocal quality. Past about FltRes `r` the filter starts to overtly distort - this is the "bite". Internally 2x oversampled around the nonlinear core to keep aliasing out of the saturation; loudness is auto-compensated post-filter so cranking FltRes gets gnarly without getting unmanageably loud. Try it on saw leads, plucks, and anything with rich high-frequency content - it transforms ordinary patches into something with attitude.
- **Filter envelope** (params 16-19: FEnvAm / FEnvAt / FEnvHl / FEnvDc) - bipolar amount, modulates filter cutoff in normalized space. Only triggers when amount ≠ 0. Atk/Hld/Dcy share the same time table as the amp envelope (see [Envelope time table](#envelope-time-table) below). Special case at FEnvAt=0: the filter envelope **truly snaps** to peak with no anti-click ramp (filter cutoff has no click risk, unlike amplitude), giving you instant Roland-style filter stabs.
- **Per-track tilt EQ** (param 28: `S>Tone`) - bipolar one-pole tilt inserted right after the per-track compressor. `h` (centre) is bypass; values below `h` roll off highs (LP), values above roll off lows (HP). Same shape as the master Delay/Reverb tone controls. Useful for gentle per-track mix shaping (e.g. dipping the subs of a snare or trimming hiss off cymbals) without a separate FX module.
- **Per-track stereo chorus** (param 27: `R>Chorus`) - bipolar single-knob chorus inserted post-drive. `h` (centre) is bypass. **Negative side** ramps a "warm/Juno-style" chorus with **2.5/3.75/5/6.25 ms** base delays - tight, classic. **Positive side** ramps a "spacey" chorus with **10/15/20/25 ms** base delays - wider, more dispersed. Magnitude scales depth (up to ~4.8 ms LFO swing), rate (0.3-1.0 Hz), feedback (0-50%), and wet mix together, so one slider gives you a cohesive musical sound at any setting. Feedback ramps slightly faster than depth/rate so the resonant character builds in earlier - matches the mental model that pushing the slider further means more obvious chorus, not just louder dry-vs-wet. Internally: 4 LFO-modulated taps 90° apart, ~110 Hz HPF on the wet path keeps bass mono and out of the feedback loop. Costs ~2% of one CPU core per active track at 48 kHz; bypassed tracks are essentially free.
- **Drive** (params 29 / 30: DrvMdl / Drive amount) - 11 modes, see the [Drive models](#drive-models) section below for the full breakdown.
- **Sends** (params 31-33): DlySnd, RDlSnd, RevSnd - per-track wet amounts to the master buses.
- **Pan** (param 34) - bipolar, `cos`/`sin` equal-power.
- **Volume** (param 35) - dry output level.

### Envelope time table

Both the amp envelope (`AmpAtk` / `AmpHld` / `AmpDcy`) and the filter envelope (`FEnvAt` / `FEnvHl` / `FEnvDc`) read time values from the same hand-picked table below. Dense in the snappy region (5 of the first 12 glyphs cover 0-50 ms for fine control over percussive transients), still granular through the body region (175 ms to 6 s), reaching `z = 32 s` at the top.

| Glyph | Time     | Glyph | Time     | Glyph | Time     | Glyph | Time     |
|-------|----------|-------|----------|-------|----------|-------|----------|
| `0`   | 0 (special, see below) | `9` | 125 ms | `i` | 600 ms | `r` | 5.0 s |
| `1`   | **2.5 ms** | `a` | 150 ms | `j` | 750 ms | `s` | 6.0 s |
| `2`   | 5 ms      | `b` | 175 ms | `k` | 1.0 s  | `t` | 7.5 s |
| `3`   | 10 ms     | `c` | 200 ms | `l` | 1.25 s | `u` | 10.0 s |
| `4`   | 20 ms     | `d` | 250 ms | `m` | 1.5 s  | `v` | 12.5 s |
| `5`   | 30 ms     | `e` | 300 ms | `n` | 2.0 s  | `w` | 16.0 s |
| `6`   | 50 ms     | `f` | 350 ms | `o` | 2.5 s  | `x` | 20.0 s |
| `7`   | 75 ms     | `g` | 400 ms | `p` | 3.0 s  | `y` | 24.0 s |
| `8`   | 100 ms    | `h` | 500 ms | `q` | 4.0 s  | `z` | 32.0 s |

**Value `0` special cases** (different meaning per slot):

- **AmpAtk (10) / AmpDcy (12)**: `0` means "as fast as possible without clicking". Internally clamped to a 1 ms anti-click floor (a literal zero-sample ramp on amplitude pops, so we never let those go below 1 ms).
- **AmpHld (11)**: `0` is the **INF / gate-mode marker**: level holds at peak until note-off arrives. Default. Any value `1..z` is a finite hold dwell that auto-decays after the time elapses.
- **FEnvAt (17)**: `0` is a **true instant snap** (no anti-click floor; filter cutoff has no click risk). Roland-style hard-snap filter envelopes.
- **FEnvHl (18)**: same INF behavior as AmpHld. `0` waits at peak for note-off.
- **FEnvDc (19)**: 1 ms anti-click floor at `0`, same idea as AmpDcy.

**Practical recipes**:

- *Snappy percussive amp env*: `AmpAtk = 0`, `AmpHld = 4` (20 ms dwell), `AmpDcy = 6` (50 ms tail). Quick stab, no sustain needed.
- *Pad with slow filter sweep*: `AmpHld = 0` (gate mode), `AmpDcy = h` (500 ms release), `FEnvAm` open positive, `FEnvAt = j` (750 ms slow open), `FEnvHl = 0` (filter sustains while note held), `FEnvDc = h` (500 ms close on release).
- *Hard filter stab*: `FEnvAt = 0` (true snap), `FEnvHl = 4` (20 ms hold), `FEnvDc = 6` (50 ms close), `FEnvAm` open positive. Instant filter chirp on every note.

---

## Drive models

Eleven distortion algorithms selected by `DrvMdl` (param 29) and intensity-controlled by `Drive` (param 30). Two important things to know upfront:

**Pre-filter vs post-filter placement.** Where the drive lives in the signal chain changes the result substantially:

- **Pre-filter (per-voice, before the filter)**: `Clip`, `Sin`, `Fold`, `Wrap`. The waveshaper runs on each voice's raw oscillator output. The filter then shapes the harmonics the distortion produced. Classic subtractive synth architecture: gnarly oscillator, mellow filter sweep can take the edge off, resonance interacts with the new harmonics.
- **Post-filter (per-track, after the filter)**: `Post`, `PostAd`, `Alg1/2/3`, `Cream`, `Doom`. The drive runs on the full mixed track signal after the filter. The filter's removed harmonics aren't there to be distorted; the distortion adds new harmonics on top of whatever survived. Better for amp-style colouring of an already-shaped sound.

**Drive amount has two phases.** From `0` to `8` the wet/dry mix ramps from clean to 100% wet. From `8` to `z` the mix stays at 100% wet but the input pre-gain keeps climbing (~1× to ~12× for most shapes, ~19× for the Post variants), so the timbre keeps deepening without further level change. Means `8` is the "fully wet but mild" point; push past it for proper saturation.

---

### 0: Clip (pre-filter)

Hard digital clip at ±1.0. Bright, edgy, harmonically dense, classic "fuzz pedal into LP filter" texture when paired with the Moog. Below `8` you hear it as a dirt overlay; past `8` the squarewave-like character takes over.

- **Best for**: aggressive bass, hard leads, drum bus glue, anything you want shaped by the filter afterward
- **Math**: `clamp(x · g_in, -1, 1)` (pure hard-knee clip)

### 1: Post (post-filter)

Same hard clip as `Clip`, but placed after the filter. The filter's already softened the high end so what gets clipped is the surviving fundamental + low harmonics. Sounds more like a transparent limiter at low drive, more like a square-wave shaper at high drive.

- **Best for**: post-filter level taming, deliberate compression-like behaviour, "studio limiter pushed too hard" character
- **Math**: same as Clip, but with a hotter pre-gain (1 → ~19× across the drive range, vs ~12× for Clip)

### 2: PostAd (post-filter)

Adaptive `tanh` soft-clip post-filter. Smoother than `Post` because tanh has a soft knee rather than a hard corner, and the higher pre-gain (~19×) means even moderate drive settings push well into the saturation region.

- **Best for**: warm bus saturation, cleaner-sounding heavy drive, rounding off a too-bright synth
- **Math**: `tanh(x · g_in)` (smooth saturator, infinite-soft knee)

### 3: Sin (pre-filter)

Sine waveshaper: `sin(x · π/2)`. Below ±1 it's nearly identity (very mild colour); past ±1 it folds back into itself. Adds odd harmonics smoothly, then odd+even as it folds. Often described as "bell-like" at high drive.

- **Best for**: enriching thin oscillators with smooth harmonics, lead synths that need bite without harshness, evolving textures as drive rises
- **Math**: `sin(x · π/2 · g_in)`. Wraps over the sine peak when |x · g_in| > 1.

### 4: Fold (pre-filter)

Triangle wavefolder, Buchla-style: signal reflects off ±1 boundaries instead of clipping or wrapping. Extremely characterful: creates upper harmonics that move *with* the input level rather than at fixed multiples of the fundamental.

- **Best for**: West Coast / Buchla-style FM-ish timbres, animated textures, anything where you want the harmonic content to breathe with dynamics
- **Math**: triangular reflection that bounces off ±1; output stays bounded but the waveform shape morphs continuously with drive

### 5: Wrap (pre-filter)

Sawtooth wrap: when the signal crosses ±1, it jumps to the opposite extreme. Brutal discontinuities that produce massive aliasing-like high content (intentionally; these are real harmonics, not aliasing). Sounds digital and chaotic.

- **Best for**: harsh industrial textures, aggressive sound design, glitch-style noise
- **Math**: `((x · g_in + 1) mod 2) - 1` (sharp wraparound at boundaries)

### 6: Alg1 (post-filter)

Single sine fold: `sin(x · π)`. Folds the signal once into a sinusoidal envelope. Adds odd harmonics with a notably smooth, almost vocal character at moderate drive.

- **Best for**: organ-like richness, Rhodes-ish overtones, smooth warming
- **Math**: `sin(x · π · g_in)` (single full-cycle sine fold)

### 7: Alg2 (post-filter)

Double sine fold: `sin(x · 2π)`. Twice the fold rate, so you get twice the harmonic density before clipping kicks in. Brighter and busier than `Alg1`, with a metallic edge starting to appear.

- **Best for**: metallic leads, bell-like accents, FM-style timbres without an FM operator
- **Math**: `sin(x · 2π · g_in)` (double-frequency sine fold)

### 8: Alg3 (post-filter)

Triple sine fold: `sin(x · 3π)`. The most extreme of the Alg variants. Inharmonic, gritty, glassy. Often unmusical at full drive but a powerful shaper at low-to-mid amounts.

- **Best for**: aggressive sound design, ring-mod-ish flavours, granular-sounding overlays
- **Math**: `sin(x · 3π · g_in)` (triple-frequency sine fold)

### 9: Cream (post-filter, stateful)

Marshall-style tube saturation. Three cascaded `tanh` stages with DC bias (for asymmetric clipping, hence even harmonics), inter-stage coupling-cap HPFs, and a 6-band cabinet simulation (HPF + 4 EQ bells + LPF that opens with drive). 50Hz drive smoothing prevents zipper noise on automation. A sub-50Hz clean-bass bypass is mixed back in so the bottom doesn't disappear under saturation. Genuine guitar amp character.

- **Best for**: rock-style leads, warm crunchy bass, anything that wants "tube warmth" rather than digital edge
- **Math**: 3 × `tanh(x · stage_gain + bias) → DC blocker → coupling HPF → ...` then the cab sim. Stateful: has memory across samples, so transients respond differently than steady tones.

### 10: Doom (post-filter, stateful)

Rat-style fuzz pedal. Three cascaded asymmetric hard/soft fuzz stages (blend of hard clipping and tanh, with adjustable asymmetry per stage), looser coupling caps than Cream (60-80Hz vs 120-180Hz so more bass survives), and a darker, more scooped cabinet simulation (mid scoop at 700Hz, brighter shelf at 5500Hz). Significantly more aggressive gain staging than Cream, saturates much harder for the same drive setting. Like Cream, includes the sub-50Hz clean bypass.

- **Best for**: doom / sludge bass, fuzz leads, brutal saturated drums, anything that wants to sound completely destroyed but musically
- **Math**: 3 × asymmetric fuzz stage with blended hard-clip + tanh → coupling caps → scooped cab sim. Stateful.

---

## LFO system

Every track carries **three monophonic LFOs** (`L1`, `L2`, `L3`) that can modulate any combination of the track's 36 base-36 params: oscillator slots, envelopes, filter, drive, sends, pan, volume, even other LFOs' depth or rate. Edits live on the LFO panel, but a few of their controls are surfaced as regular grid params so the sequencer (`!` operator, other LFOs) can drive them directly.

LFOs run at **480 Hz** internally, decoupled from the sequencer tick rate for smooth modulation regardless of BPM.

### Track-level slot layout

Each LFO claims a pair of grid params:

| Param | LFO | Purpose |
|-------|-----|---------|
| `K` (20) | L1 | Dep (target depth) |
| `L` (21) | L1 | Rate |
| `M` (22) | L2 | Dep |
| `N` (23) | L2 | Rate |
| `O` (24) | L3 | Dep |
| `P` (25) | L3 | Rate |

**Dep slots (`K` / `M` / `O`)** are bipolar centred on `h`. They behave specially: the visible base value stays pinned at `h`, but writes to it (manual or `!`) update **all three of the LFO's stored per-target depths uniformly**. So if your LFO is routed to filter cutoff, drive amount, and sends, `!K17` is unison neutral, `!Kz` boosts every target's depth uniformly, and `!K0` inverts every target's depth. Lets a single grid write swing the entire LFO output without editing each target depth individually. Other LFOs can also modulate `K/M/O` for "depth modulation" (LFO modulating LFO).

**Rate slots (`L` / `N` / `P`)** are unipolar `0..z`. The base-36 value indexes a tempo-synced cycle table (same one the Bouncer `&` uses): `0` is 4096 ticks per cycle (very slow drift), `h` is 32 ticks (≈ 4 s/cycle at BPM 120 Normal: the default and the comfortable "musical pad" speed), `w` is 4 ticks (= quarter note, the last "musical" stop), and `x`/`y`/`z` are sub-tick fast values for sound design (`x` = 8 Hz, `y` = 64 Hz, `z` = 128 Hz at BPM 120 Normal). Cycle length is in ticks, so the LFO always tempo-syncs as you change BPM or TickRate. Full speeds at every glyph live in the [Rate table](#rate-table-shared-by-bouncer-and-lfos) below.

### Per-LFO parameters

These live on the LFO panel (not on the grid), one set per LFO:

| Param | Range | What it does |
|-------|-------|--------------|
| **Prm 0/1/2** | `.` or `0..z` | Three target params per LFO. `.` = off; otherwise a base-36 param id (0..35). The same LFO can hit up to 3 different track params at once. |
| **Wav** | `0..8` | Waveform shape (see table below). |
| **Smo** | `0..z` | Smoothing. `0` = none (pure waveform). For continuous shapes (Sin / Tri / Saw / Sqr) it's a one-pole lowpass on the output: 10 ms time constant at `1`, ramping to ~1 s at `z`. For S&H shapes (SH-R / SH-S) it's a two-phase step interpolation: low values give the classic instant-step character, high values smoothly ease between held samples (turns S&H into something more like a glide-filtered random walk). |
| **Skw** | `0..y`, bipolar `h` | Phase skew. `h` = symmetric. Below `h` compresses the first half of the cycle (sharper attack, longer release on Tri/Saw); above `h` does the opposite. On SQR it acts like PWM. |
| **HOf** | `0..y`, bipolar `h` | Horizontal phase offset. `h` = no offset; magnitude shifts the cycle start by ±50%. Useful for de-syncing two LFOs running at the same rate. |
| **VOf** | `0..y`, bipolar `h` | Vertical (DC) offset added to the waveform before polarity. `h` = no offset; magnitude shifts the bipolar output up or down (clamped to ±1). Lets you bias a bipolar LFO toward positive-only or negative-only excursions without switching to UNI polarity. |
| **Pol** | `BI` / `UNI` | Bipolar (default, output `[-1, +1]`) or unipolar (output `[0, +1]`). UNI is useful when you want a target to only swing one way from its base value. |
| **Run** | `FREE` / `TRIG` / `ONCE` | Phase reset behaviour, see below. |

### Waveforms

Nine shapes, selected by the `Wav` panel param:

| Idx | Name | Character |
|-----|------|-----------|
| 0 | **SIN** | Smooth sine. Default modulator for everything. |
| 1 | **TRI** | Symmetric triangle. Sharper transitions than sine, no high harmonics like a sawtooth. Good for filter sweeps that need to feel deliberate. |
| 2 | **SAW+** | Rising sawtooth (ramps `-1` → `+1` then snaps back). Classic "envelope-like" sweep: slow attack, instant return. |
| 3 | **SAW-** | Falling sawtooth (ramps `+1` → `-1` then snaps back). Mirror of SAW+: instant attack, slow decay. Useful as a poor-mans envelope on a target without using the actual envelope. |
| 4 | **SQR** | Square wave, hard `+1`/`-1` toggle. Trance-gate / step-modulation flavour. Use Skw to PWM. |
| 5 | **SH-R** | **Sample & Hold, Random**. Each cycle picks a fresh pseudo-random value. **Non-deterministic**: the sequence is different on every play and different across each LFO instance. Use when you want true unpredictability. |
| 6 | **SH-S** | **Sample & Hold, Seeded**. Each cycle picks a deterministic value derived from the project seed, the track, the LFO instance, and the cycle count. **Reproducible**: replays the same sequence between runs of the same project, but each track / LFO instance gets its own pattern, and changing the project seed reshuffles all of them at once. Use when you want a "wonky but predictable" pattern that survives save/load. |
| 7 | **CONST** | Constant `+1`. Always at full positive output. Combine with VOf and depth scaling to lock a param at any chosen offset. Useful for "always-on detune" or static sub-modulation that interacts with another LFO via `K/M/O` depth modulation. |
| 8 | **NOISE** | Continuous pseudo-random output. Same generator as SH-R but without the hold step, so it changes every ~2 ms instead of once per cycle. Pure noise modulation. |

### Run modes

Controls how the LFO responds to NoteOn events on its track:

- **FREE**: phase ignores notes entirely. The LFO runs as a continuous live stream, useful for ambient drift / tempo-synced patterns that shouldn't restart with every note.
- **TRIG**: phase resets to `0` on every NoteOn. Good for "pluck-like" envelope substitutes (combine with SAW- and a one-shot-ish rate).
- **ONCE**: phase resets to `0` on NoteOn, then **freezes at the end of the first cycle**. The LFO runs through one full pass and stops. Effectively a one-shot envelope generator. SH-S's seeded sequence also restarts from cycle 0 on TRIG / ONCE so retriggers replay the same pattern.

`Ctrl+R` (sequencer reset) jumps FREE and TRIG LFOs back to phase 0 and clears their scopes; ONCE LFOs are left untouched so a long one-shot in flight isn't interrupted.

### Rate table (shared by Bouncer and LFOs)

The Bouncer (`&`) Rate port and the per-track LFO Rate slots (`L` / `N` / `P`) both look up cycle length in this table. The cycle is always expressed in **ticks**, so changing BPM or TickRate scales every entry by the same factor, keeping things tempo-synced.

**How to read the period column**: at BPM 120 Normal (1 tick = 125 ms / 8 ticks per second). Other tempos: multiply periods by `120/BPM`. TickRate `Half` doubles all periods, `Double` halves them.

| Glyph | Ticks    | LFO period @120 Normal | LFO frequency | Musical equivalent       |
|-------|----------|------------------------|---------------|--------------------------|
| `0`   | 4096     | 512 s                  | 0.00195 Hz    | 256 bars                 |
| `1`   | 3072     | 384 s                  | 0.00260 Hz    | 192 bars                 |
| `2`   | 2048     | 256 s                  | 0.00391 Hz    | 128 bars                 |
| `3`   | 1536     | 192 s                  | 0.00521 Hz    | 96 bars                  |
| `4`   | 1024     | 128 s                  | 0.00781 Hz    | 64 bars                  |
| `5`   | 768      | 96 s                   | 0.01042 Hz    | 48 bars                  |
| `6`   | 512      | 64 s                   | 0.01563 Hz    | 32 bars                  |
| `7`   | 384      | 48 s                   | 0.02083 Hz    | 24 bars                  |
| `8`   | 256      | 32 s                   | 0.03125 Hz    | 16 bars                  |
| `9`   | 192      | 24 s                   | 0.04167 Hz    | 12 bars                  |
| `a`   | 160      | 20 s                   | 0.05000 Hz    | 10 bars                  |
| `b`   | 128      | 16 s                   | 0.06250 Hz    | 8 bars                   |
| `c`   | 96       | 12 s                   | 0.08333 Hz    | 6 bars                   |
| `d`   | 80       | 10 s                   | 0.10000 Hz    | 5 bars                   |
| `e`   | 64       | 8 s                    | 0.12500 Hz    | 4 bars                   |
| `f`   | 48       | 6 s                    | 0.16667 Hz    | 3 bars                   |
| `g`   | 40       | 5 s                    | 0.20000 Hz    | 2.5 bars                 |
| `h`   | 32       | 4 s                    | 0.25000 Hz    | 2 bars *(default)*       |
| `i`   | 24       | 3 s                    | 0.33333 Hz    | 1.5 bars (dotted whole)  |
| `j`   | 20       | 2.5 s                  | 0.40000 Hz    | 1.25 bars                |
| `k`   | 16       | 2 s                    | 0.50000 Hz    | whole note (1 bar)       |
| `l`   | 15       | 1.875 s                | 0.53333 Hz    | quintuplet whole-ish     |
| `m`   | 14       | 1.75 s                 | 0.57143 Hz    |                          |
| `n`   | 13       | 1.625 s                | 0.61538 Hz    |                          |
| `o`   | 12       | 1.5 s                  | 0.66667 Hz    | dotted half              |
| `p`   | 11       | 1.375 s                | 0.72727 Hz    |                          |
| `q`   | 10       | 1.25 s                 | 0.80000 Hz    |                          |
| `r`   | 9        | 1.125 s                | 0.88889 Hz    |                          |
| `s`   | 8        | 1.0 s                  | 1.00000 Hz    | half note                |
| `t`   | 7        | 0.875 s                | 1.14286 Hz    |                          |
| `u`   | 6        | 0.75 s                 | 1.33333 Hz    | dotted quarter           |
| `v`   | 5        | 0.625 s                | 1.60000 Hz    |                          |
| `w`   | **4**    | **0.5 s**              | **2.0 Hz**    | **quarter note** (last musical stop) |
| `x`   | **1**    | **125 ms**             | **8 Hz**      | **fast tempo-locked** (vibrato/tremolo extreme) |
| `y`   | **0.125**| **15.6 ms**            | **64 Hz**     | **audio-rate** (FM-ish timbral modulation; LFO only) |
| `z`   | **0.0625**| **7.8 ms**            | **128 Hz**    | **sound-design ceiling** (near LFO Nyquist; LFO only) |

**Two flavours of behaviour at the fast end**:

- **`0` to `w`**: musically meaningful subdivisions: bars, beats, dotted/quintuplet variants. Both Bouncer and LFO interpret these identically. `h` is the default sweet spot for slow-evolving modulation.
- **`x`**: the last whole-tick value (1 tick / 8 Hz at default). Tempo-locked. Both Bouncer and LFO use this directly.
- **`y` / `z`**: sub-tick fractions for LFO sound design. The LFO runs at 480 Hz internally, so these advertise 64 Hz / 128 Hz audio-rate modulation that's perfectly usable on most params. The **Bouncer** can only advance once per sequencer tick, so `y` and `z` collapse to the same 1-tick cycle as `x` on the Bouncer (no crash, just no extra speed).

**Practical ceilings** (when modulating with `y` / `z`):

- Most params (volume, pan, drive amount, sends, voice oscillator slots) have no extra DSP smoothing, so they receive the full LFO output up to ~120 Hz. At `z` (128 Hz) you'll hear genuine audio-rate stepping/buzz that's perfect for sound design.
- Filter cutoff (`FltCut`) and resonance (`FltRes`) have a built-in ~5 ms anti-zipper smoother, so they receive ~−1 dB at 100 Hz and ~−3 dB at 200 Hz. `y` and `z` still produce audible movement on the filter, just with progressively reduced depth. Sometimes that's exactly the spongey-resonance sound you want.

---

## Scenes

A grampus project holds **8 scene slots**. Each scene is a complete snapshot of the grid, all 8 tracks' params + LFO config, master params, BPM, TickRate, and Shuffle. The scene system itself (which scene is active, which one is queued, the quantize value) lives outside any single scene so it stays consistent as you move around.

### Mental model

> Think of a scene as the entire current state of your project. You have 8 of them. When you switch scenes, the music stays running but everything around it can change.

Fresh projects boot with all 8 slots empty (no glyphs in the grid, factory tracks, 120 BPM, Normal tick rate). Edit anything while a scene is active and the changes accumulate on that slot. Switch to a different scene and your edits stay parked on the original slot, ready to come back the next time you swap to it.

There's no explicit "save scene" gesture. The scene you're currently playing IS slot N at all times.

### Visual states on the SCENES panel

The master page has a SCENES panel at the bottom: 8 scene cells, plus a Quantize cell on the right. The cells use three independent visual signals so multiple states can stack:

- **Brackets `[ ]`** mark the active scene (whose audio is playing right now).
- **Inverted video** marks the focused cell (your cursor, when you've navigated into the panel from the master page rows above).
- **Icon to the left of the number** shows that scene's play state:
  - `▶` solid play: active scene during playback.
  - `▷` hollow play: armed, waiting for the next quantize boundary to swap in.
  - `-` dash: idle.

| Cell appearance | What it means |
|-----------------|---------------|
| ` -1 `          | Scene 1 is idle, not active. |
| `[-1]`          | Scene 1 is active, sequencer is stopped. |
| `[▶1]`          | Scene 1 is active and playing. |
| ` ▷1 `          | Scene 1 is armed (queued to swap in). |

Cells whose scene has a fully empty grid render in an extra-dim shade so populated scenes pop visually. Reset a scene and its cell drops to that dim state immediately. Paste content into an empty scene and the cell brightens.

A small **scene indicator** also lives at the top-right corner of the screen, visible from every page. It alternates between the active scene number and the armed scene number while a swap is pending, so you always know what's coming next without having to be on the master page.

### Quantize

One global setting. The same Quantize value applies to every scene transition.

`Off / 8 / 16 / 24 / 32 / 48 / 64 / 96 / 128 / 256 / 512 / 1024 / 2048`

Default = **32** ticks (2 bars at TickRate=Normal in 4/4). The default leans long enough that you can change your mind: if you arm a scene a beat early it still lands on the next bar boundary instead of immediately.

Quantize is measured in ticks, not bars, so it follows the project's tempo and tick rate. Faster BPM = the same number of ticks elapses more quickly = scene swaps land sooner.

### Where scene controls live

| Surface | What it does |
|---------|--------------|
| Top-right corner indicator | Always visible. Shows the active scene number; alternates with the armed scene number when a swap is pending. |
| Master page SCENES panel | The interactive surface. 8 scene cells + Quantize cell. Navigate into it from the rows above with arrow keys. |

### Switching scenes

| Action | Keyboard | Behavior |
|--------|----------|----------|
| Arm scene N | **Alt+1** through **Alt+8** | Queue scene N. Lands at the next quantize boundary if playing; lands instantly if stopped or if Quantize is `Off`. |
| Cancel a pending arm | **Alt+N** where N is the currently active scene | Re-arming the active scene clears any pending swap. |

When you arm a scene during playback, its cell shows `▷N` (hollow play) until the boundary fires; the moment it actually swaps, the icon flips to `▶N` and the brackets move with it.

Changing Quantize while a swap is pending doesn't cause an instant swap. The pending arm just waits for the next genuine boundary under the new Quantize value.

### Working with the SCENES panel

From any cell on the master page, press Down repeatedly to walk down through the parameter rows. Past the last row, you enter the SCENES panel and land on Scene 1.

Inside the panel:

- **Left / Right** walk between the 8 scene cells and the Quantize cell.
- **Up** exits the panel back to the row above (Master Out).
- **Down** does nothing; you're already at the bottom.

On a focused **scene cell**:

- **ENTER** arms that scene.
- **DELETE** (or Backspace) opens a "Reset scene N to defaults?" confirmation. Confirming wipes that scene back to factory state.

On the focused **Quantize cell**:

- **ENTER** does nothing.
- **Edit + Left / Right** cycles the Quantize value by 1 step (down / up the table).
- **Edit + Up / Down** cycles by 4 steps.

"Edit" is the same modifier used everywhere else for parameter adjustments: **Shift+arrow** on a keyboard, the **Edit** button + arrow on the iPad on-screen keybar, and **South+D-pad** on the TrimUI Brick gamepad.

### What carries across a scene swap

The audio engine deliberately keeps these alive when scenes change so the swap feels musical rather than glitchy:

- Notes that are still ringing keep ringing through their natural release.
- Envelopes mid-stage continue from where they were.
- LFO phase counters keep advancing. The new scene's LFO settings (waveform, rate, depth) take effect immediately, but phase doesn't reset.
- The sequencer tick number is continuous across the whole session, which is what makes tick-quantized swaps line up to the beat.

What does reset on swap (because it was tied to the old scene's grid):

- LFO modulation routing cache.
- In-flight `Z`-lerp interpolations.
- Per-parameter "controlled by sequencer" flags.

### Undo / redo across scenes

Undo is one global timeline, capped at 100 entries. Each entry is a full snapshot of all 8 scenes captured right after a change.

The crucial behavior: **undo never changes which scene you're looking at**. If you edit scene 0, swap to scene 5, edit scene 5, then press Cmd+Z three times, the undo walks back through the changes in reverse order, but your view stays on whatever scene you were on when you pressed undo. A toast appears each time announcing what was rolled back and which scene it touched: `UNDO  Grid Edit · Scene 5`, `UNDO  Master · Scene 1`, etc.

Resetting a scene is undoable. The pre-reset state is captured as a snapshot, so Cmd+Z brings the whole scene back. Re-doing the reset re-runs it programmatically without re-asking for confirmation.

Arming a scene is NOT undoable. It's a transport action like Play / Stop, not an edit.

The undo timeline gets wiped whenever you load a project or start a new one. Mixing two projects' edits in one undo stack would be confusing.

For held-key sweeps (knob twists, BPM nudges, anything that fires the same parameter rapidly), grampus collapses consecutive same-target edits within a 400 ms window into one undo entry. Sweeping a knob from 0 to 100 returns to 0 in a single Cmd+Z, not 100 micro-steps.

### Copy, cut, and paste between scenes

There are three separate clipboards in flight at any time, each scoped to a different unit of work:

- **Sequencer clipboard** holds a region of grid cells.
- **Track / panel clipboard** holds a track's parameter snapshot or just one panel of it.
- **Scene clipboard** holds an entire scene (grid + tracks + master + tempo).

The first two work the same way they always have: copy on one scene, swap to another scene, paste, you're done.

The scene clipboard is new and triggers from the master page only, when a scene cell is focused on the SCENES panel:

| Shortcut | Effect |
|----------|--------|
| Copy (Cmd+C / Ctrl+C / your gamepad's Copy combo) | Copies the focused scene's full state into the scene clipboard. Toast: `Scene N copied`. |
| Cut  (Cmd+X / Ctrl+X) | Same as copy, but also resets the source scene to defaults. True "move" workflow: cut from scene 3, paste to scene 6, scene 3 ends up empty and scene 6 has the data. Toast: `Scene N cut`. |
| Paste (Cmd+V / Ctrl+V) | Drops the scene clipboard onto the focused scene cell. Toast: `Pasted into scene N`. If the clipboard was never set, you get a `Clipboard empty` toast instead. |

If you copy from a project and try to paste into a project of a different grid size, you'll see `Grid size mismatch. Try saving your old project with the new version before pasting`. This happens when you copy from a project that was saved before the grid size changed in a newer build; saving and reloading the source project is enough to bring it up to the current size.

### Project files

Project files (`.grampus`) carry all 8 scenes. The active scene index, the Quantize value, and the per-scene tempo settings (BPM, TickRate, Shuffle) all save and restore.

Older project files (saved by previous versions of grampus) load cleanly: their content lands in scene 0, and scenes 1-7 come up empty. From there you can edit them, save again, and the file becomes a current-format project carrying all 8 scenes.

If an old project was saved at a smaller grid size, loading it auto-grows the grid to the current default (256 wide × 128 tall) by padding with empty cells. Existing content stays where it was; nothing gets clipped.

### iPad keybar

The on-screen keybar's row 2 has 4 medium cells: **T−**, **T+**, **S−**, **S+**.

- **T−** / **T+** cycle through the 9 instrument pages (Master + 8 tracks), wrapping at the ends.
- **S−** / **S+** arm the previous / next scene.

This row replaces the old per-track T1-T8 row. If you used to tap individual track cells, you'll now tap T+ a few times to walk through tracks. The trade-off frees up the row for scene controls, which previously had no touch surface at all.

### TrimUI Brick gamepad

The Brick doesn't have analog sticks; in their place are two dedicated buttons along the bottom edge of the screen that emit the standard `L3` / `R3` events. The **left   button (L3)** arms the previous scene; the **right   button (R3)** arms the next scene. Both wrap around the 8 slots.

L1 and R1 still cycle through pages on solo press, so page navigation is unaffected.

---

## Master effects bus

Every track's dry signal sums into the master bus, and each track also feeds three send buses (delay, reverse delay, reverb) at user-controlled levels.

- **Delay** - stereo ping-pong, tempo-synced with 16 musical divisions from 1/32 up to 4 bars. Feedback, bipolar tone (LP ↔ HP), LFO time modulation, and an optional granular overlay (see below). Master params: DlyTime, DlyFdbk, DlyTone, DlyMod, DlyGrn.
- **Reverse delay** - true stereo reverse delay with crossfaded boundaries. Same 16 divisions, its own tone and mod. Also has a granular overlay.
- **Reverb** - 32-voice modulated FDN reverb with **shimmer feedback** (octave-up pitch shifter blended back into the reverb tail). Master params: RvbSize, RvbDcay, RvbDiff, RvbMod, RvbTone, RvbShim. Plus a `Dly→Rvb` send that feeds the delay outputs into the reverb tank for cascaded textures.
- **Granular texture** (used by both delays as an overlay, controlled by `DlyGrn`) - bipolar centred on `h` (= 17 = bypass, no grains). Magnitude scales density (up to 32 simultaneous grains spawning every ~120 ms with random offsets), grain size (smaller as the knob moves further from centre), and wet mix together. Two distinct shimmer flavours either side of centre:
  - **Above `h` (`i..z`)**: **octave shimmer**. ~50% of grains pitched +1 octave (2× rate), the rest at base pitch. Bright, lifting, classic shimmer-reverb-style upward halo.
  - **Below `h` (`0..g`)**: **power-chord shimmer**. Grains randomly pitched at base, +fifth (1.5× rate), or +octave (2× rate). Gives a chord-stack flavour because the fifth + octave + root sound together inside the grain cloud, like a ghosted power chord trailing the dry signal.
- **Tape Noise + Vinyl Dust** - two always-on stereo loops mixed into the master sum. Each has its own gain (master params `X TapeNois` / `Y VinylDst`); `0` = silent. The loops keep playing even when their gain is `0`, so automating `!X` or `!Y` from the grid never restarts them mid-loop.
- **FAT compressor** - one-knob bus compressor (Compress param). The single amount control simultaneously drives threshold, ratio, attack, release, and makeup gain along musical curves so `o` (`~70%`) gives instant glue and `z` pumps hard. Default is `o` for ready-to-go cohesion.
- **Safety limiter** - fixed brick-wall limiter at the very end of the chain. Catches stray peaks before output. No user controls.

Each track also runs its own light per-track compressor before the master bus to tame polyphonic stacking, so loud chords don't slam the FAT compressor into pumping unintentionally.

The Master page lays out as three stacked panels - Delay, Reverb, and Mst Gain / Tape / Vinyl / Compress - each with parameters on the left and a live visualization on the right. The visualizations: stereo tap echoes (delay), particle-wash diffusion (reverb), and a scrolling input-level + gain-reduction meter (master).

---

## Parameter system

The sequencer and the synth share one language: **36 base-36 parameter slots per track**, 0 through Z.

**Track params**

| ID | Name | Notes |
|----|------|-------|
| 0 | EngType | 0 = Drifter, 1 = Warper, 2 = BrokenFM, 3 = String, 4 = ROMpler, 5 = LoFiTron, 6 = WaveSurf |
| 1-9 | Oscillator-specific | Names change per engine (see above). BrokenFM only uses 1-5; rest are skipped. |
| 10-12 | AmpAtk / AmpHld / AmpDcy | Amplitude envelope |
| 13-15 | FltTyp / FltCut / FltRes | Filter |
| 16-19 | FEnvAm / FEnvAt / FEnvHl / FEnvDc | Filter envelope |
| 20-25 | L1Dep / L1Rat / L2Dep / L2Rat / L3Dep / L3Rat | Per-track LFO depths (bipolar) and rates. `!K<v>` / `!M<v>` / `!O<v>` write all three stored depths uniformly. |
| 26-28 | reserved | - |
| 29 | DrvMdl | Drive model |
| 30-33 | Drive / DlySnd / RDlSnd / RevSnd | Drive amount + sends |
| 34 | Pan | Bipolar |
| 35 | Volume | |

**Master params** (channel 0 from `!`)

| ID | Name |
|----|------|
| 0 | MstGain |
| 1-6 | DlyTime / DlyFdbk / DlyTone / DlyMod / DlyGrn / Dly→Rvb |
| 10-15 | RvbSize / RvbDcay / RvbDiff / RvbMod / RvbTone / RvbShim |
| 33 | TapeNois (always-on tape hiss loop, 0..0.125) |
| 34 | VinylDst (always-on vinyl crackle loop, 0..0.125) |
| 35 | Compress (FAT compressor amount) |

Every parameter set via `!` in the grid is marked as **sequencer-controlled** (shown with `~` next to the value on the corresponding instrument page). When playback stops, those revert to their user-set base values.

---

## Features

- **Per-track pages** - Tab / Shift+Tab cycle through the Sequencer screen, eight track pages (T1-T8), and the Master page, in that order. Each track page is a 2×3 panel grid (OSC / AMP+FILTER / FX+MIX on top, three LFO panels on the bottom) with a one-row preset strip above the panels (`TRACK: name`, `[ SAVE ]`, `[ LOAD ]`); the Master page hosts the global delay, reverb, tape/vinyl, and compressor. Every instrument page shows a tab strip across the top row (`SEQ T1 T2 … T8 MASTER`) with the active page highlighted, so you always know where you are.
- **LFOs** - three per track (KL / MN / OP). Each LFO routes to up to three target params on the same track, with three independently-edited depths. Nine waveforms (Sine, Tri, Saw±, Square, Sample&Hold Random, Sample&Hold Seed, Const, Noise), phase skew, two smoothing modes, bipolar/unipolar output, horizontal and vertical offsets, and FREE/TRIG/ONCE run modes. Each panel shows a braille scope of the current output. Rates reuse the Bouncer (`&`) rate table: `h` (17) ≈ 4 s/cycle at 120 BPM Normal, `z` = 1 tick/cycle.
- **Param adjust shortcuts** - on instrument pages, `Shift+Left/Right` adjusts the focused param by ±1, `Shift+Up/Down` by ±4. Bipolar params (Pan, Xfade, FEnvAm, etc.) clamp at the bipolar maximum so the centre point stays exact.
- **Parameter interpolation** - the `!` operator's fourth port is an interpolation rate. `0` = instant, `1`-`z` = smooth lerp over N ticks. Perfect for filter sweeps and envelope rides driven from the grid.
- **Per-engine presets** - switching `EngType` saves all params for the old engine and restores the last-used params for the new one, so you can A/B engines without losing your tweaks.
- **Per-track spectrum analyser** - eight side-by-side frequency bands per track rendered above the grid on the Sequencer screen.
- **Undo/redo** - grid edits and parameter changes use separate undo stacks (100-deep each).
- **Copy/paste** - Ctrl+C/X/V works on the grid (region selection). Track pages have a separate clipboard for params with **panel-scoped** behaviour:
  - **Ctrl+C** (or `L1/R1 + South` on gamepad) → copy **all** params of the current track (engine type, OSC, AMP/FILTER, FX/MIX, all three LFOs).
  - **Ctrl+X** (or `L1/R1 + West` on gamepad) → copy **only the panel** containing the focused param. Each track page has six panels (OSC, AMP/FILTER, FX/MIX, LFO KL, LFO MN, LFO OP); whichever one you're focused inside is what gets copied. Useful for sharing just an LFO routing or just an FX/MIX setup between tracks without overwriting anything else.
  - **Ctrl+V** (or `L1/R1 + East` on gamepad) → paste. Whole-track clipboards overwrite all 36 slots (and switch the destination's engine if needed). Panel-scoped clipboards only touch the matching panel on the destination, leaving every other panel untouched.

  **Where a panel paste lands** depends on the clipboard's panel and your current focus on the destination track. Non-LFO panels only exist once per track so they always go to the matching panel. LFO panels can be redirected by where you're focused, so you can copy an LFO setup and drop it into a different LFO slot on any track:

  | Clipboard scope | Focused panel on destination | Pastes into |
  |---|---|---|
  | OSC | anywhere | OSC |
  | AMP/FILTER | anywhere | AMP/FILTER |
  | FX/MIX | anywhere | FX/MIX |
  | LFO KL | LFO KL | LFO KL (same) |
  | LFO KL | LFO MN | **LFO MN** (redirected) |
  | LFO KL | LFO OP | **LFO OP** (redirected) |
  | LFO KL | OSC / AMP / FX | LFO KL (fallback to source) |

  Toast confirmation shows the destination panel (e.g. "Copied LFO KL from T2", "Pasted LFO OP to T5"). The Master page has no copy/paste.
- **Themes** - 14 built-in: Default, Catppuccin, Draculish, Gruvbox, Nord, Solarized, Tokyo Night, Amber, Crimson, GB, GB Inv, Mono, 1991, Goodnight. Selection persists across sessions.
- **Gamepad** - first-class controller support, designed around grid-editing ergonomics. D-pad navigation, shoulder combos for cut/copy/paste/undo, on-screen keyboard for glyph entry on the Sequencer screen. On instrument pages, hold South + D-pad for the same param-adjust deltas as Shift+Arrow on desktop.
- **On-screen keyboard** - available on the Sequencer screen for handheld / no-keyboard use. Instrument pages use Shift+Arrow / South+D-pad and (on desktop) direct alphanumeric entry.
- **Save / load** - `.grampus` project files store the grid, all track params, master params, BPM, seed, tick rate, and shuffle. Plain human-readable text. Older project files load cleanly across versions.
- **Track presets** - each track page has a one-row strip across the top: `TRACK: <name>` on the left, `[ SAVE ]` in the middle, `[ LOAD ]` on the right. Press Enter (or South on the gamepad) to activate the focused slot:
  - `TRACK:` opens a small text dialog so you can rename the track. The name is just a label, no file is touched.
  - `[ SAVE ]` writes the current track's full state (engine, OSC, AMP/FILTER, FX/MIX, all three LFOs) to a `.grampustrack` file inside `data_dir/presets/`. If a preset with the same name already exists, you get an overwrite prompt.
  - `[ LOAD ]` opens a file browser sandboxed to `data_dir/presets/`. Pick a preset and the whole track is replaced with the preset's state. One Ctrl+Z rolls the load back atomically.

  **Presets are independent of projects.** A track's full state always lives in the project file itself, so loading a preset is a one-shot copy: the params get stamped onto the track and from that moment on they belong to the project, not to the preset file. There's no link kept between the two. Overwriting a preset later does **not** retroactively change any project that previously loaded it. Likewise, deleting a preset doesn't break any project. The project still plays the same because every value is stored locally.

  **Factory presets** ship inside `data_dir/presets/_factory_presets/`. They're refreshed from the bundled set every launch (so app updates can ship new ones cleanly). Your own saves live in `data_dir/presets/` itself, never inside the `_factory_presets` subfolder. Even when you edit a factory preset and save it under the same name, the new file lands in the user folder, leaving the factory copy untouched.

  Filenames are limited to 32 characters (excluding the `.grampustrack` extension); empty names are silently ignored. Older save files with longer names still load fine. The limit only applies to new saves.
- **WAV recording** - `F10` (or R1+Start on the gamepad) toggles recording of the master output to `data_dir/recordings/<project>_<YYYYMMDD_HHMMSS>.wav`. 16-bit PCM stereo with TPDF dithering, headers patched after every chunk so the file is always valid even mid-take.
- **Toggle comment region** - Algorave style live mute (typical trick for Tidal Cycles/Strudel). Make a selection, press `/` (or L1+R1 on the gamepad), and grampus wraps each row of the rect with `#` (ORCA's comment operator) - silencing everything between. Press again on a wrapped selection to clear the `#`s. Refuses to overwrite real glyphs at the edges.
- **Envelopes**: the amp envelope and filter envelope each have three stages: Attack, Hold, Decay. Both follow the same rule: the **Hold value** determines the entire shape, independently of note length.

  - **`Hld = 0` → INF (gate mode)**: Attack → **Sustain** (level held at peak) → Decay on note-off. The note's operator length controls when note-off fires and Decay begins. Default for both envelopes. Use for pads, leads, sustained filter sweeps.
  - **`Hld = 1..z` → AHD (auto-decay)**: Attack → **Hold** (dwell for the specified time) → Decay fires automatically, regardless of note length. The envelope completes its full shape even while the voice is still sounding. Use for drums, plucks, or to put a rhythmic filter sweep on a long sustained pad.

  These two modes apply **independently** to the amp and filter envelopes, so you can mix and match freely. Example: `AmpHld = 0` (INF, voice holds for the full note) + `FEnvHld = h` (500 ms auto-AHD filter sweep that opens and closes mid-note).

  **Trigger notes (operator length = 0)** always use auto-AHD on both envelopes regardless of the Hld value, since no note-off ever arrives for them.

  This applies to both the **amp envelope** (`AmpAtk / AmpHld / AmpDcy`, params A/B/C) and the **filter envelope** (`FEnvAt / FEnvHl / FEnvDc`, params H/I/J).

  - **Time per value**: every Atk/Hld/Dcy slot reads its base-36 glyph through this table. For **Hld only, value `0` is the special INF marker** (not a time); for Atk and Dcy, `0` means instant (~1 ms internally):

    | Glyph | Atk / Dcy time | Hld | | Glyph | Atk / Dcy time | Hld |
    |:-----:|----------------|-----|---|:-----:|----------------|-----|
    | `0`   | instant (~1 ms) | **INF** | | `i`   | 600 ms | 600 ms |
    | `1`   | 2.5 ms  | 2.5 ms  | | `j`   | 750 ms | 750 ms |
    | `2`   | 5 ms    | 5 ms    | | `k`   | 1 s    | 1 s    |
    | `3`   | 10 ms   | 10 ms   | | `l`   | 1.25 s | 1.25 s |
    | `4`   | 20 ms   | 20 ms   | | `m`   | 1.5 s  | 1.5 s  |
    | `5`   | 30 ms   | 30 ms   | | `n`   | 2 s    | 2 s    |
    | `6`   | 50 ms   | 50 ms   | | `o`   | 2.5 s  | 2.5 s  |
    | `7`   | 75 ms   | 75 ms   | | `p`–`z` | 3–32 s | 3–32 s |
    | `8`   | 100 ms  | 100 ms  | | | | |
    | `a`–`h` | 150–500 ms | 150–500 ms | | | | |

---

## CLI

```
grampus [file.grampus] [flags]

Flags:
  --no-gamepad           Disable gamepad polling.
  --no-keyboard          Hide the Keyboard help tab (useful for handheld / controller-only setups).
  --data-dir <path>      Where to read/write saves and settings (defaults to the binary's directory).
  --buffer-size <N>      Force the audio buffer size in frames. Falls back to a smaller size if rejected by the device.
  --sample-rate <N>      Target audio sample rate (default 48000). Will be clamped to whatever the device supports.
  --low-cpu              Reduce DSP cost on weaker hardware. Halves the granular grain budget and lowers shimmer quality.
  --diag                 Print an audio underrun warning to stderr when a callback misses its budget.
```

The positional argument is an optional `.grampus` file to load at startup.

---

## Credits

Sequencer behaviour is modelled on [Orca-c](https://github.com/hundredrabbits/Orca-c) with additional custom operators inherited from [bOrca](https://github.com/boorch/bOrca).

---

## Running Grampus on non-TrimUI-Brick devices (unsupported)

> **This is not officially supported.** Grampus's TrimUI build is developed and tested only on the TrimUI Brick. If you've installed the PAK on a different device — Smart Pro, an RG-series handheld, or anything else — buttons may behave wrong or do nothing because the gamepad mapping shipped in `launch.sh` is hand-tuned for the Brick's exact controller. The notes below are a courtesy guide for users who want to try Grampus on those devices anyway. **No support is offered for issues on non-Brick hardware.**

The PAK ships a helper script, `apply_gamepad_mapping.sh`, that auto-detects whatever SDL sees for your gamepad and rewrites `launch.sh` with the right mapping. Brick users never need to touch it.

There are two ways to run it.

### Method 1 — File Manager (no SSH required)

1. On your device, open any file manager that can launch shell scripts (e.g. **Files.pak** on NextUI, or whatever your firmware's equivalent is).
2. Navigate into the **Grampus.pak** folder. Path varies per device, but on TrimUI variants installed via PakStore it's typically `/mnt/SDCARD/Tools/tg5040/Grampus.pak/`.
3. Find the file **`apply_gamepad_mapping.sh`** and execute it (usually a tap or "Run" action).
4. You almost certainly won't see any visible feedback — most file managers run shell scripts silently. That's fine; the script is doing its work.
5. Re-launch Grampus. If buttons feel correct now, you're done.

**Recovery / Undo.** The first time you run the script it creates a backup file called `launch.sh.bak` in the same folder, containing the original Brick-tuned mapping. (Subsequent runs do NOT overwrite this backup, so you always have the truly-original.) To revert: delete `launch.sh`, rename `launch.sh.bak` to `launch.sh`, re-launch Grampus.

### Method 2 — SSH (recommended if you have it set up)

You'll see actual output telling you what changed.

1. SSH into your device. (If your firmware doesn't have SSH out of the box, NextUI offers a `NativeSSH.pak` you can install from PakStore.)
2. `cd` into the Grampus.pak folder. On most TrimUI variants installed via PakStore:
   ```sh
   cd /mnt/SDCARD/Tools/tg5040/Grampus.pak/
   ```
   Adjust the path if your device puts PAKs elsewhere.
3. Run the apply script:
   ```sh
   ./apply_gamepad_mapping.sh
   ```
4. You'll see output similar to:
   ```
   Probing connected joystick...
   Saved original launch.sh → launch.sh.bak

   OLD mapping (will be commented out):
     030000005e0400008e02000014010000,TrimUI Player1,...

   NEW mapping (auto-detected):
     0300abcd1234.....,Your Device Name,...

   Applied. Re-launch Grampus from the Tools menu for the change to take effect.
   To revert: cp launch.sh.bak launch.sh
   ```
5. Re-launch Grampus from the device menu.

**Recovery via SSH.** A single command restores the original mapping:
```sh
cp launch.sh.bak launch.sh
```

### Troubleshooting

- **"Mappings are identical — launch.sh already up to date."** SDL on your device happens to report exactly the same mapping that was already in launch.sh. Nothing to do; the original mapping was correct.
- **"ERROR: Couldn't read a mapping from gamepad_config -l output."** Your device's joystick wasn't detected by SDL. Try unplugging/replugging any external controller (or rebooting the device) and re-run.
- **Buttons still wrong after applying.** Your device might genuinely need a custom interactive walkthrough. Run `./probe_gamepad.sh -c 0` (SSH only) for the full prompt-by-prompt mapper, then manually paste the resulting string into `launch.sh`'s `SDL_GAMECONTROLLERCONFIG=` line.
- **Want to verify what SDL actually sees?** Run `./probe_gamepad.sh -l` (SSH) — this shows the raw joystick + GameController info side by side, including the auto-detected mapping string.
