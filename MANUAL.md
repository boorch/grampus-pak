# Grampus

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
- [Shared voice chain](#shared-voice-chain)
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
  - `0`-`9` - one of the 10 scales: Major, Minor, Dorian, Lydian, Mixolydian, Pentatonic, Hirajoshi, Iwato, Tetratonic, Fifths.
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
- **R** - rate, 0-35. Indexes a **16-entry musical cycle table**; higher values = faster cycles (from thousands of ticks per cycle down to 1 tick per cycle).
- **S** - shape 0-7: `0` Tri · `1` InvTri · `2` Sine · `3` InvSine · `4` Square · `5` InvSquare · `6` Saw · `7` InvSaw.

##### Chord table

Used by both `$` (indices 10-35) and `=` (indices a-z / A-Z):

| Index | Glyph | Chord | Intervals |
|-------|-------|-------|-----------|
| 10 | `a` | Major | 0 4 7 |
| 11 | `b` | Minor | 0 3 7 |
| 12 | `c` | Sus4 | 0 5 7 |
| 13 | `d` | Sus2 | 0 2 7 |
| 14 | `e` | Major7 | 0 4 7 11 |
| 15 | `f` | Minor7 | 0 3 7 10 |
| 16 | `g` | Dom7 | 0 4 7 10 |
| 17 | `h` | MinorMaj7 | 0 3 7 11 |
| 18 | `i` | Minor6 | 0 3 7 9 |
| 19 | `j` | Major6 | 0 4 7 9 |
| 20 | `k` | Major9 | 0 4 7 11 14 |
| 21 | `l` | Minor9 | 0 3 7 10 14 |
| 22 | `m` | Major add9 | 0 4 7 14 |
| 23 | `n` | Minor add9 | 0 3 7 14 |
| 24 | `o` | Dim | 0 3 6 |
| 25 | `p` | Half Dim7 | 0 3 6 10 |
| 26 | `q` | Dim7 | 0 3 6 9 |
| 27 | `r` | Aug | 0 4 8 |
| 28 | `s` | Aug7 | 0 4 8 10 |
| 29 | `t` | Dom9 | 0 4 7 10 14 |
| 30 | `u` | Dom7♭9 | 0 4 7 10 13 |
| 31 | `v` | Dom7♯9 | 0 4 7 10 15 |
| 32 | `w` | Major 6/9 | 0 4 7 9 14 |
| 33 | `x` | Minor 6/9 | 0 3 7 9 14 |
| 34 | `y` | Minor11 | 0 3 7 10 17 |
| 35 | `z` | Minor7♭5 | 0 3 6 10 |

For `=`, indices 0-9 are **enriched** chord variants with octave doublings (e.g., `0` = Major + octave = C-E-G-C, `4` = Major7 + 3rd-octave = C-E-G-B-E, `6` = Dom7 + 5th-octave = C-E-G-B♭-G).

---

#### Synth / MIDI output

These operators send notes and parameter changes from the grid to the engines. Channel `0` addresses the master effects bus (for `!`), while channels `1`-`8` address synth tracks T1-T8. Note operators (`;`, `:`, `=`) are silent on channel `0` since the master bus has no voices.

##### `;` - Note (poly)

Sends `NoteOn` to a synth track. Polyphonic - allocates a free voice in the track's voice pool. (Note: Grampus uses `;` for poly and `:` for mono — the opposite of ORCA, where `:` is the only note operator and `;` is the arpeggiator.)

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

Plays an entire chord in one tick on a synth channel. Uses the chord table above, with enriched variants at indices 0-9 and first inversions at A-Z.

| Operator | Channel | Octave | Root | Chord | Velocity | Length |
|:--------:|:-------:|:------:|:----:|:-----:|:--------:|:------:|
|   `=`    |    C    |    O   |   R  |   T   |     V    |    L   |

Notes are voiced ascending (each chord tone is raised by octaves if needed to stay above the previous one).

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

Five engines ship today:

### Drifter (EngType 0)

Dual oscillator virtual-analog. Two PolyBLEP-anti-aliased oscillators (A and B) each pass through a phase-distortion stage, get crossfaded together, optionally summed with a sub-oscillator, and panned for stereo width. Slow Brownian pitch drift gives the static-DCO sound a living, analog feel. The shape control is a continuous morph through **triangle → saw → square** with **saw at the centre**, which is why Shape A and Shape B render as bipolar sliders centred at `h`.

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | ShapeA | bipolar | Continuous morph for Osc A. **`0` = pure triangle**, **`h` = pure saw (centre)**, **`y` = pure square**. Anything in between linearly interpolates between the two adjacent shapes (e.g. `8` is roughly halfway between tri and saw). |
| 2 | SkewA | bipolar | Phase-distortion knee for Osc A. **`h` = no skew (symmetric waveform)**. Below `h` skews the cycle so the first half is shorter than the second; above `h` does the opposite. Adds odd-order harmonics - for square waves it acts like PWM, for tri/saw it asymmetrises the timbre. |
| 3 | ShapeB | bipolar | Same shape morph as ShapeA, applied independently to Osc B. Useful for layering two different waveforms (e.g. ShapeA at saw `h`, ShapeB at square `y`). |
| 4 | SkewB | bipolar | Skew on Osc B. Independent from SkewA. |
| 5 | BPitch | unipolar | Osc B pitch offset above Osc A, scaled linearly across **0 to +5 semitones** (`0` = unison, `z` = +5 semi). For unison detune, leave it at `0` and use Drift; for chord-stab intervals (octaves, fifths, etc.), set explicit values. |
| 6 | Xfade | bipolar | A/B level balance. **`0` = Osc A only**, **`h` = equal mix (centre)**, **`y` = Osc B only**. |
| 7 | Drift | unipolar | Brownian pitch instability - both oscillators wander slightly off-pitch over ~200-350 ms via a Gaussian random walk with a one-pole smoother. **`0` = locked pitch**, **`z` ≈ ±25 cents max wander**. Each oscillator drifts independently, so light Drift makes a single voice sound like two slightly-out-of-tune oscillators (the natural source of "analog" warmth). |
| 8 | SubOsc | bipolar | Sub oscillator level + octave selector. **`h` = off**. Below `h`: sub is **two octaves below** Osc A, magnitude controls its level. Above `h`: sub is **one octave below**, magnitude controls level. Triangle waveform internally - sits clean under the main oscillators. |
| 9 | Stereo | bipolar | Stereo spread by panning A and B in opposite directions via equal-power pan. **`h` = both centred (mono)**. Magnitude controls how hard they push to L/R. Combine with detune (BPitch + Drift) and/or per-osc shape differences for instant stereo width without an external chorus. |

**Common combos:**
- *Classic detuned saw lead*: ShapeA `h` (saw), ShapeB `h`, BPitch `3` (~+0.4 semi), Xfade `h`, Drift `8`.
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

**Warp modes** (WrpMdA and WrpMdB independently pick one of these - they're discrete, the slider shows the mode name). The Warp slider is **bipolar** - `h` (centre) is identity (raw wavetable, no warp). Each mode interprets the bipolar value as it makes sense:

| Glyph | Mode | What it does | Slider behaviour |
|-------|------|--------------|------------------|
| `0` | **Bend** | Phase exponent (`power = 8^amount`). Stretches one half of the cycle and compresses the other. | **bipolar** - below `h` compresses the end (concave), above `h` compresses the start (convex). |
| `1` | **Sync** | Hard-sync simulation (1× to 32× phase multiplier). Adds inharmonic partials and chirpy aliasing reminiscent of analog sync sweeps. | **magnitude** - sign ignored. Distance from `h` controls intensity. |
| `2` | **Formant** | Sync + half-sine amplitude window - vocal "ah/oh" character. The window narrows with warp magnitude so high settings sound like a formant burst. | **magnitude** - sign ignored. |
| `3` | **Squeeze** | Asymmetric phase stretch - moves the pivot away from centre. | **bipolar** - below `h` squeezes the start (pivot → 0.1), above `h` squeezes the end (pivot → 0.9). |
| `4` | **Fold** | Phase modulation by a sine. At low magnitude this gently bends the phase; at high magnitude the trajectory reverses direction twice per cycle, producing classic wavefolder-style harmonic richness. | **magnitude** - sign ignored. |
| `5` | **Quantize** | Phase quantisation / staircase (128 steps near `h`, down to 2 steps at the extremes). Produces a bitcrush / square-wave reduction effect. | **magnitude** - sign ignored. |

| # | Name | Polarity | Role |
|---|------|----------|------|
| 1 | ShapA | unipolar | Osc A wavetable selector (see mapping above). |
| 2 | WarpA | bipolar | Warp depth/direction for Osc A. **`h` = no warp**, deflection in either direction engages the warp. The mode (above) decides whether sign matters. |
| 3 | WrpMdA | discrete (0-5) | Warp mode for Osc A. |
| 4 | ShapB | unipolar | Osc B wavetable selector. Independent from ShapA, so you can layer e.g. Sine + Saw. |
| 5 | WarpB | bipolar | Warp depth/direction for Osc B. Same semantics as WarpA. |
| 6 | WrpMdB | discrete (0-5) | Warp mode for Osc B. Independent from WrpMdA - you can have Bend on A and Fold on B simultaneously. |
| 7 | BPitch | unipolar | Osc B pitch offset above Osc A, **0 to +5 semitones** (same scale as Drifter). `0` = unison. |
| 8 | Xfade | bipolar | A/B crossfade. **`0` = Osc A only**, **`h` = equal mix**, **`y` = Osc B only**. |
| 9 | Detune | unipolar | Unison detune + stereo spread combined. `0` = mono / no detune; magnitude raises both detune amount (up to ±25 cents) and the stereo width of the detuned copies. A second detuned voice of each oscillator pans opposite the original. |

**Common combos:**
- *Vintage square + fold lead*: ShapA `l` (square), WarpA `z`, WrpMdA `4` (Fold), Xfade `h`, Detune `4`.
- *Vocal formant pad*: ShapA `z` (saw), WarpA `t`, WrpMdA `2` (Formant), ShapB `e` (tri), WrpMdB `0` (Bend), Detune `g`.
- *Bell-like quantised sine*: ShapA `0` (sine), WarpA `z`, WrpMdA `5` (Quantize) - produces metallic, bitcrushed inharmonic tones.
- *Classic sync lead*: ShapA `z` (saw), WarpA `r` or `7`, WrpMdA `1` (Sync), Xfade `8`.
- *Bend pulled both ways*: WrpMdA `0` (Bend) - try WarpA `0` (full concave) and WarpA `z` (full convex) on a saw to hear the bipolar character.

> Filter tip: warp modes (especially Sync, Fold, Formant) generate a lot of high-frequency content. If FltCut is at the default `p` (~8 kHz) you'll hear maybe a third of what they actually produce - try sweeping FltCut up to `y` or `z` while auditioning warp settings.

Character: modern digital wavetable. Can go from vintage-sounding (square + fold) to bell-like (sine + quantize) to vocal (saw + formant) without leaving one engine.

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

Sample-playback engine. Drum kits and pitched instruments share the same 5-parameter interface; the shipped kits cover both. Pitch zones are auto-constructed from the samples in each kit, so any note you play picks the closest sample and pitches it from there. No interpolation - nearest-neighbour reads, tracker-style, with character (audible aliasing on extreme upward shifts).

> **Drum kits and per-octave sample switching.** For the drum-focused kits (`909`, `808`, `REALDRUM`, `AMEN`), each kit slot occupies a full octave. `C` plays the original (unpitched) sample; every other note up to (but not including) the next `C` is the same sample pitched. So `C..B` in octave 3 is one sample (pitched up across the octave), and `C` in octave 4 picks the next slot's sample. This makes it easy to lay drums out chromatically — pick the kick at one `C`, snare at the next `C`, hats at the next, etc.

**Shipped kits (15)**, indexed by ROM 0..14:

| ROM | Name | Type |
|-----|------|------|
| 0 | 909 | Drum machine |
| 1 | 808 | Drum machine |
| 2 | REALDRUM | Acoustic drum kit |
| 3 | AMEN | Classic break (sliced) |
| 4 | PIANO | Pitched, multi-sampled |
| 5 | RHODES | Pitched, multi-sampled |
| 6 | HAMMOND | Pitched, looped |
| 7 | STRINGS | Pitched, looped |
| 8 | BRASS | Pitched, looped |
| 9 | GUITAR | Pitched, multi-sampled |
| 10 | SINE | Chromatic 7-octave waveform bank, looped |
| 11 | TRIANGLE | Chromatic 7-octave waveform bank, looped |
| 12 | SQUARE | Chromatic 7-octave waveform bank, looped |
| 13 | SAW | Chromatic 7-octave waveform bank, looped |
| 14 | NOISYSAW | Chromatic 7-octave waveform bank, looped (noisy variant) |

| # | Name | Role |
|---|------|------|
| 1 | ROM | Kit selector. |
| 2 | Pitch | Coarse tune ±17 semitones (1 base-36 step = 1 semitone). |
| 3 | Tune | Fine tune ±50 cents. |
| 4 | Reverse | 0 = forward, 1 = reversed playback. |
| 5 | VelCut | Velocity → filter cutoff modulator. Bipolar. `h` = no effect. Stacks with FltCut, FEnvAm, and any LFOs that target FltCut. |

Note-off behaviour depends on the kit:
- **Drum / one-shot kits**: note-off is ignored, the sample plays through. If you want the tail cut, use the Amp envelope (`AmpHld` / `AmpDcy`).
- **Pitched / looped kits**: note-off triggers the standard Amp envelope release.

**Live pitch / tune modulation**: `Pitch` (slot 2) and `Tune` (slot 3) recompute the playback rate every time they change during a sustained note, so they respond to LFO mod, `!` operator writes, and `!`-with-interpolation lerps in real time. This lets you build pitch envelopes (one-shot LFO on Tune or Pitch) and smooth pitch glides (`!` on Pitch with interpolation > 0). Static patches (no modulation) behave identically to before - the per-note rate is still latched at note-on; only the live-update path is new.

Character: classic sampler. Drum kits hit hard; pitched instruments stretch through pitch zones with that "tape-slowed" character at extremes.

---

## Shared voice chain

Every voice, regardless of type, passes through the same chain:

- **AHD envelope** (params 10-12: AmpAtk / AmpHld / AmpDcy)
  - Custom time curve: hand-picked 0-12 (10 ms → 750 ms), exponential 13-35 (1 s → 24 s).
  - Hold = 0 gives **gate-based sustain**: level stays at peak until the sequencer sends NoteOff.
- **True-stereo MultiFilter** (params 13-15: FltTyp / FltCut / FltRes) - independent L/R filter instances:
  - 0 Off (passthrough)
  - 1 **Moog LP** (4-pole ladder, 24 dB/oct)
  - 2 **SVF HP** (2-pole)
  - 3 **SVF BP** (2-pole)
  - 4 **SVF Notch** (cascaded 4-pole)
  - 5 **LP>HP** (serial 1-pole LP → 1-pole HP - band-pass-ish)
  - 6 **DJ Filter** (bipolar: cutoff center = bypass, below = LP, above = HP)
  - Cutoff: 20 Hz - ~20 kHz mapped across 0-z. Parameter smoothing (~5 ms one-pole) eliminates zipper noise.
- **Filter envelope** (params 16-19: FEnvAm / FEnvAt / FEnvHl / FEnvDc) - bipolar amount, modulates filter cutoff in normalized space. Only triggers when amount ≠ 0.
- **Per-track tilt EQ** (param 28: `S>Tone`) - bipolar one-pole tilt inserted right after the per-track compressor. `h` (centre) is bypass; values below `h` roll off highs (LP), values above roll off lows (HP). Same shape as the master Delay/Reverb tone controls. Useful for gentle per-track mix shaping (e.g. dipping the subs of a snare or trimming hiss off cymbals) without a separate FX module.
- **Per-track stereo chorus** (param 27: `R>Chorus`) - bipolar single-knob chorus inserted post-drive. `h` (centre) is bypass. **Negative side** ramps a "warm/Juno-style" chorus with **2.5/3.75/5/6.25 ms** base delays - tight, classic. **Positive side** ramps a "spacey" chorus with **10/15/20/25 ms** base delays - wider, more dispersed. Magnitude scales depth (up to ~4.8 ms LFO swing), rate (0.3-1.0 Hz), feedback (0-50%), and wet mix together, so one slider gives you a cohesive musical sound at any setting. Feedback ramps slightly faster than depth/rate so the resonant character builds in earlier - matches the mental model that pushing the slider further means more obvious chorus, not just louder dry-vs-wet. Internally: 4 LFO-modulated taps 90° apart, ~110 Hz HPF on the wet path keeps bass mono and out of the feedback loop. Costs ~2% of one CPU core per active track at 48 kHz; bypassed tracks are essentially free.
- **Drive** (params 29 / 30: DrvMdl / Drive amount) - 11 modes:
  - 0 Clip, 1 Post, 2 PostAd (adaptive RMS-tracking soft clip), 3 Sin, 4 Fold, 5 Wrap, 6-8 Alg1/2/3 (sine-fold variants), 9 Cream (Marshall-style 3-stage tube), 10 Doom (Rat-style fuzz). Cream and Doom include cabinet simulation.
- **Sends** (params 31-33): DlySnd, RDlSnd, RevSnd - per-track wet amounts to the master buses.
- **Pan** (param 34) - bipolar, `cos`/`sin` equal-power.
- **Volume** (param 35) - dry output level.

---

## Master effects bus

Every track's dry signal sums into the master bus, and each track also feeds three send buses (delay, reverse delay, reverb) at user-controlled levels.

- **Delay** - stereo ping-pong, tempo-synced with 16 musical divisions from 1/32 up to 4 bars. Feedback, bipolar tone (LP ↔ HP), LFO time modulation, and an optional granular overlay (see below). Master params: DlyTime, DlyFdbk, DlyTone, DlyMod, DlyGrn.
- **Reverse delay** - true stereo reverse delay with crossfaded boundaries. Same 16 divisions, its own tone and mod. Also has a granular overlay.
- **Reverb** - 32-voice modulated FDN reverb with **shimmer feedback** (octave-up pitch shifter blended back into the reverb tail). Master params: RvbSize, RvbDcay, RvbDiff, RvbMod, RvbTone, RvbShim. Plus a `Dly→Rvb` send that feeds the delay outputs into the reverb tank for cascaded textures.
- **Granular texture** (used by both delays as an overlay) - up to 32 simultaneous grains spawning every ~120 ms with random offsets, octave-up shimmer when the param is positive, power-chord shimmer when negative.
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
| 0 | EngType | 0 = Drifter, 1 = Warper, 2 = BrokenFM, 3 = String, 4 = ROMpler |
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

- **Per-track pages** - Tab / Shift+Tab cycle through the Sequencer screen, eight track pages (T1-T8), and the Master page, in that order. Each track page is a 2×3 panel grid (OSC / AMP+FILTER / FX+MIX on top, three LFO panels on the bottom); the Master page hosts the global delay, reverb, tape/vinyl, and compressor. Every instrument page shows a tab strip across the top row (`SEQ T1 T2 … T8 MASTER`) with the active page highlighted, so you always know where you are.
- **LFOs** - three per track (KL / MN / OP). Each LFO routes to up to three target params on the same track, with three independently-edited depths. Nine waveforms (Sine, Tri, Saw±, Square, Sample&Hold Random, Sample&Hold Seed, Const, Noise), phase skew, two smoothing modes, bipolar/unipolar output, horizontal and vertical offsets, and FREE/TRIG/ONCE run modes. Each panel shows a braille scope of the current output. Rates reuse the Bouncer (`&`) rate table: `h` (17) ≈ 4 s/cycle at 120 BPM Normal, `z` = 1 tick/cycle.
- **Param adjust shortcuts** - on instrument pages, `Shift+Left/Right` adjusts the focused param by ±1, `Shift+Up/Down` by ±4. Bipolar params (Pan, Xfade, FEnvAm, etc.) clamp at the bipolar maximum so the centre point stays exact.
- **Parameter interpolation** - the `!` operator's fourth port is an interpolation rate. `0` = instant, `1`-`z` = smooth lerp over N ticks. Perfect for filter sweeps and envelope rides driven from the grid.
- **Per-engine presets** - switching `EngType` saves all params for the old engine and restores the last-used params for the new one, so you can A/B engines without losing your tweaks.
- **Per-track spectrum analyser** - eight side-by-side frequency bands per track rendered above the grid on the Sequencer screen.
- **Undo/redo** - grid edits and parameter changes use separate undo stacks (100-deep each).
- **Copy/paste** - Ctrl+C/X/V works on the grid (region selection). On a track page it operates on **all 36 params of the current track** (separate clipboard from the grid). Cut acts as copy for convenience. The Master page has no copy/paste.
- **Themes** - 14 built-in: Default, Catppuccin, Draculish, Gruvbox, Nord, Solarized, Tokyo Night, Amber, Crimson, GB, GB Inv, Mono, 1991, Goodnight. Selection persists across sessions.
- **Gamepad** - first-class controller support, designed around grid-editing ergonomics. D-pad navigation, shoulder combos for cut/copy/paste/undo, on-screen keyboard for glyph entry on the Sequencer screen. On instrument pages, hold South + D-pad for the same param-adjust deltas as Shift+Arrow on desktop.
- **On-screen keyboard** - available on the Sequencer screen for handheld / no-keyboard use. Instrument pages use Shift+Arrow / South+D-pad and (on desktop) direct alphanumeric entry.
- **Save / load** - `.grampus` file format containing grid, all track params, master params, BPM, seed, tick rate, and shuffle. Plain human-readable text. Backward compatible — files from older versions without a `SHUFFLE` field load cleanly with shuffle defaulted to 50% (straight).
- **WAV recording** - `F10` (or R1+Start on the gamepad) toggles recording of the master output to `data_dir/recordings/<project>_<YYYYMMDD_HHMMSS>.wav`. 16-bit PCM stereo with TPDF dithering, headers patched after every chunk so the file is always valid even mid-take.
- **Toggle comment region** - Algorave style live mute (typical trick for Tidal Cycles/Strudel). Make a selection, press `/` (or L1+R1 on the gamepad), and grampus wraps each row of the rect with `#` (ORCA's comment operator) - silencing everything between. Press again on a wrapped selection to clear the `#`s. Refuses to overwrite real glyphs at the edges.
- **Envelopes** - the amp and filter envelopes choose their shape based on the note's duration, so a single set of params covers both percussion and pads.
  - **`len = 0` (trigger)** → one-shot **AHD**: Attack → Hold (peak dwell) → Decay, all automatic. `Hold` is the dwell time at peak before Decay starts. Use this for drum hits, plucks, blips - anything where the envelope itself defines the sound's length.
  - **`len > 0` (sustained)** → **ASR**: Attack → Sustain (held at peak) → Decay (on note-off). `Hold` is ignored. The note's length controls how long the sound holds before tailing off. Use this for pads, leads, anything where the operator's duration port is the natural body length.
  - This applies to both the **amp envelope** (`Amp Atk/Hld/Dcy`, params A/B/C) and the **filter envelope** (`FEnv Atk/Hld/Dcy`, params H/I/J). Same param slots, behavior just switches based on how the operator triggered the note.
  - **Time per value** - every Atk/Hld/Dcy slot reads its base-36 glyph through this table:

    | Glyph | Time | | Glyph | Time | | Glyph | Time |
    |:-----:|------|---|:-----:|------|---|:-----:|------|
    | `0`   | instant¹ | | `c`   | 200 ms | | `o`   | 2.5 s |
    | `1`   | 2.5 ms   | | `d`   | 250 ms | | `p`   | 3 s |
    | `2`   | 5 ms     | | `e`   | 300 ms | | `q`   | 4 s |
    | `3`   | 10 ms    | | `f`   | 350 ms | | `r`   | 5 s |
    | `4`   | 20 ms    | | `g`   | 400 ms | | `s`   | 6 s |
    | `5`   | 30 ms    | | `h`   | 500 ms | | `t`   | 7.5 s |
    | `6`   | 50 ms    | | `i`   | 600 ms | | `u`   | 10 s |
    | `7`   | 75 ms    | | `j`   | 750 ms | | `v`   | 12.5 s |
    | `8`   | 100 ms   | | `k`   | 1 s    | | `w`   | 16 s |
    | `9`   | 125 ms   | | `l`   | 1.25 s | | `x`   | 20 s |
    | `a`   | 150 ms   | | `m`   | 1.5 s  | | `y`   | 24 s |
    | `b`   | 175 ms   | | `n`   | 2 s    | | `z`   | 32 s |

    ¹ Value `0` is clamped internally to ~1 ms - for Atk / Dcy that's an instant transition; for Hld in oneshot mode it skips the dwell stage.

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
