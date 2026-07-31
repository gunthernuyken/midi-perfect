# MIDI PERFECT 2

**[▶ Live demo](https://gunthernuyken.github.io/midi-perfect/)** · [Deutsche Fassung](README.de.md) · MIT licensed

A multi-lane MIDI generator in **one HTML file**. No server, no build step, no
dependencies. It runs straight from the filesystem, generates patterns from a
chord progression and plays them live into a DAW over Web MIDI — developed and
tested against **Cubase Pro 14** on macOS through the IAC driver.

```
Browser (Web MIDI)  ──IAC Bus 1──▶  Cubase  ──▶  HALion Sonic / Groove Agent / EZdrummer
```

Five independent lanes (DRUMS, BASS, CHORDS, ARP, MELODY) turn a chord
progression into deterministic patterns, send them on separate MIDI channels,
and can be switched, rerolled and mutated while the sequencer runs. Plus MIDI
Clock and MMC for transport sync, and SMF export (Type 1, one track per lane).

> **Chrome or Edge required.** Web MIDI is not implemented in Safari or Firefox.

The interface is **bilingual** — the `DE · EN` switch sits in the command bar and the choice is remembered.

---

## Why it exists

Backing tracks for practising improvisation are either fixed recordings that you
memorise after the third pass, or subscription tools that live in someone else's
cloud. This one generates the material locally, differently every loop, in any
key, and drives your own instruments instead of shipping its own sounds.

The whole thing being a single file is not a gimmick. It means the tool still
works in five years, needs nothing installed, and can be copied to a USB stick.

---

## Contents

| File | Purpose |
|---|---|
| `MIDI-PERFECT-2.html` | The complete application. One file, ~224 KB. |
| `CUBASE-SETUP.md` | Cubase setup — and the MIDI routing traps that cost people afternoons |
| `ARCHITEKTUR.md` | Code structure, data model, extension points *(German)* |
| `CHANGELOG.md` | Build history with the reasoning behind each change *(German)* |

---

## Quick start

1. **Enable the IAC driver** (macOS): Audio MIDI Setup → Window → Show MIDI
   Studio → double-click IAC Driver → tick *Device is online*, create Bus 1.
   On Windows use [loopMIDI](https://www.tobias-erichsen.de/software/loopmidi.html)
   instead.
2. Open the [live demo](https://gunthernuyken.github.io/midi-perfect/) or the
   downloaded `MIDI-PERFECT-2.html` in **Chrome or Edge**.
3. Allow MIDI access. **Include SysEx** when asked — without it MMC will not
   work (MIDI Clock still will).
4. Select `IAC Driver Bus 1` as MIDI output.
5. Create one MIDI track per channel in your DAW → see
   [CUBASE-SETUP.md](CUBASE-SETUP.md). **This is the step where it usually goes
   wrong.** Read it.
6. Press PLAY.

When opened as `file://`, Chrome does not remember the MIDI permission
permanently — after a hard reload you have to grant it again. Served over
HTTPS it sticks.

---

## Controls

### Command bar (sticky, always reachable)

```
▶ PLAY   ■ STOP   🎲   1.1   ● ● ● ●   │   DRUMS 10   BASS 1   CHORDS 2   ARP 3   MELODY 4
```

Lane chips show colour, state and target channel.
**Click** = lane on/off · **Shift+click** = solo (click again to restore).

| Key | Action |
|---|---|
| `Space` | Play / Stop |
| `1` … `5` | Lane on/off |
| `⇧1` … `⇧5` | Solo that lane |
| `R` | Reroll every unlocked lane |
| `D` | Randomise all styles |

Shortcuts are suppressed while focus sits in an input or select.

### Panels

Every panel heading is a toggle; the collapsed state survives a reload.

| Panel | Contents |
|---|---|
| MIDI Connection | Port, channel routing, GM sounds, channel lock, MIDI monitor |
| Transport & Macros | Play/Stop, BPM, swing, humanize, energy, complexity, loop, mutation |
| Blues workshop | Form generator, turnaround, quick change, transposition, groove link, backbeat, tempo fields, chorus dynamics |
| Cubase Sync | MIDI Clock, MMC play/record/stop, count-in, tempo handshake |
| Progression | Chord sequence as text or blocks, per-bar locks against mutation |
| Harmony engine | Circle of fifths, degrees, suggestions, reharmonisation, generator |
| Lanes | Per lane: style, channel, sound, octave, velocity, density, swing, velocity spread |
| MIDI Export | SMF Type 1, one track per lane |
| Keyboard | Live display of sounding notes, coloured by lane |

---

## Concepts

### Lanes

Five fixed lanes. Each has its own MIDI channel, style catalogue, random seed
and lock.

| Lane | Channel | Default style | GM default |
|---|---|---|---|
| DRUMS | 10 | Rock 8ths | Standard Kit |
| BASS | 1 | Walking bass | Electric Bass (33) |
| CHORDS | 2 | Swing comping | Grand Piano (0) |
| ARP | 3 | Up/Down | Clean Guitar (27) |
| MELODY | 4 | Motif | Trumpet (56) |

**DRUMS stays on channel 10** in the routing buttons. That is deliberate — GM
drums live there. Groove Agent and EZdrummer listen elsewhere, and that gets
translated in the DAW, not here; see [CUBASE-SETUP.md](CUBASE-SETUP.md).

Lane state persists in `localStorage` and is restored on the next visit.

### Determinism

The generators are **seed-based and deterministic**: same seed, same chord
progression, same bar → same pattern. The velocity spread of the groove engine
derives from the lane seed too, so it is reproducible. Only `Humanize` (timing
and velocity jitter) is truly random, and it is applied at send time.

Consequence: a pattern you like can be frozen with `Lock` while everything else
keeps mutating.

### Groove engine

Four processing stages sit between generator and output, inside `buildTake()`,
and are **baked into the take** — so playback and SMF export run through the
same chain and sound identical:

1. **Swing**, resolvable per lane. `100 %` is a full triplet feel (the offbeat
   lands on the third triplet). Blues shuffle sits at 62–68, slow blues 12/8 at
   95–100, funk at 12–20.
   Patterns already written on a triplet grid (`grid:12`) are **not** shifted
   again — the detection uses note position relative to the beat, not the style
   name. Getting this wrong produces a double shuffle.
2. **Velocity spread**, per lane, deterministic from the seed.
3. **Backbeat accent** on 2 and 4. The accent is not merely added — everything
   else is lowered slightly at the same time. Pure addition drives the snare
   into the 127 ceiling, and the accent vanishes exactly when you want it most.
4. **Chorus dynamics**, see below.

The **groove link** forces DRUMS, BASS and CHORDS onto the same swing value.
It is on by default; without it the rhythm section can drift apart by accident.

### Chorus dynamics

Across 2–8 loop passes the track builds and falls back. This is an
**arrangement**, not a volume ramp — each pass adds a layer:

| Stage | Drums | Bass | Chords |
|---|---|---|---|
| 0 | kick/snare, hi-hat on quarters only, no crash | on 1 and 3 | one chord per bar |
| 1 | hi-hat throughout | on all quarters | on 1 and 3 |
| 2 | full kit, crash and open hi-hat | eighths | throughout |
| 3 | plus ride and bell | eighths | throughout |

Selection works on **note position within the bar** and on note number, not on
style names, so the arc applies to all 137 patterns. Kick, snare, rimshot and
clap are protected and never thinned — otherwise the take loses its pulse
rather than just its density. Velocity rises across the arc as well, and the
bell lands on the quarter notes only at the peak. An **expression CC** (freely assignable
number) is sent per pass to all non-drum channels — point it at a drawbar CC and
a Hammond emulation opens up over the arc by itself.

The arc also runs across repetitions in the **export**.

### Blues workshop

Forms are stored **by degree** (semitone distance from the tonic plus chord
type), not as chord names, so every form is available in every key: 12-bar
standard, 12-bar slow with 9th voicings, jazz blues, minor blues, 8-bar, 16-bar —
combinable with quick change and five turnaround variants for bars 11–12.

Transposition rewrites the chord line while preserving suffixes (`m7b5`, `7b9`,
`:2`) and pulls the harmony engine's key along. The practice circle deliberately
runs G → C → F → Bb → Eb → A → D → E, so it also covers the keys guitarists
usually avoid.

### Infinity / mutation

With `INFINITY` on, the generator rerolls a portion of the bars and lane seeds
at the end of every loop; the `Mutation` slider sets the probability. Locked
bars and locked lanes stay untouched. The point is a progression that keeps
evolving without ever repeating itself exactly.

### Scheduling

The sequencer uses **lookahead scheduling**: every 20 ms the next ~220 ms of
music goes into the MIDI queue with exact timestamps
(`MIDIOutput.send(bytes, timestamp)`). That gives stable timing despite the
JavaScript event loop, while keeping STOP effectively instant, because never
more than a fifth of a second is planned ahead.

Tempo changes, style switches and chord edits take effect mid-bar: the take is
rebuilt and the pointer fast-forwarded to the current position. Note-offs
already queued keep their time, so nothing hangs.

MIDI Clock runs on the same tick grid (24 clocks per quarter) from the same time
base, so the DAW does not drift against the generator.

### Channel lock and MIDI monitor

Two diagnostic tools, both hanging off the single central send function
(`sendAt`) — no message can slip past them:

- **Channel lock** blocks every message on channels that belong to no active
  lane, including program change and all-notes-off. If the DAW still shows
  traffic, it is provably not this page.
- **MIDI monitor** logs every outgoing message with channel and type:

```
SEND  Ch10  NoteOn  36 v110
SEND  Ch10  NoteOff 36 v0
BLOCK Ch 1  CC     123 v0
```

---

## Export

`MIDI Export` writes a **Standard MIDI File Type 1** with one track per active
lane, including tempo, time signature, track names and program change. Drag it
straight into the DAW; each lane lands on its own track.

Program change is only written when a sound is selected in the lane panel.

---

## Known limitations

- **Chrome/Edge only.** Web MIDI is not implemented in Safari or Firefox.
- **No MIDI input.** The page only sends; it does not listen.
- **Five fixed lanes.** The lane list is an array in the code, with no UI to add
  more. Extending it is trivial, just not clickable.
- **No undo for lane settings.** Only the progression has an undo history.
- **`localStorage` is per origin/path.** Move the HTML file and the stored state
  is gone.

---

## Contributing

Issues and pull requests are welcome. The whole application is one file — open
it in any editor, the sections are numbered and commented. `ARCHITEKTUR.md`
describes the data model and the extension points (in German; ask in an issue
if you need a specific part translated).

---

## Coffee fund

If this saved you an afternoon of MIDI routing, feel free to
[buy me a coffee](https://paypal.me/guenuy). Entirely optional — the
project stays MIT licensed and free either way.

---

## License

[MIT](LICENSE) © 2026 Günther Nuyken
