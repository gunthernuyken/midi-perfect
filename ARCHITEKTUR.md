# Architektur

Eine einzelne HTML-Datei, ~157 KB, ~2450 Zeilen JavaScript in einem `<script>`-Block
am Ende des `<body>`. Kein Build, kein Modulsystem, keine Abhängigkeiten,
kein Framework. ES5-Syntax durchgehend (`var`, `function`) — das ist eine bewusste
Entscheidung: die Datei soll ohne Transpiler in jedem Chromium laufen und sich mit
einem Texteditor reparieren lassen.

## Warum eine Datei

Der Anwendungsfall ist „im Studio doppelklicken und loslegen". Ein Build-Schritt
oder ein lokaler Server wären genau die Reibung, die das Werkzeug unbenutzbar
macht. Der Preis ist eine große Datei — der wird mit einer strikten
Abschnittsgliederung bezahlt.

---

## Abschnitte

Der Code ist in nummerierte Blöcke unterteilt, jeder mit Kommentarbanner:

| # | Abschnitt | Inhalt |
|---|---|---|
| 0 | Konstanten / Utils | `PPQ`, `BAR`, `Q`, Notennamen, GM-Instrumentenliste, `DK` (Drum-Keymap), `clamp`, RNG |
| 1 | Lanes | `LANES`, `LANE_STYLES`, `DRUMPAT` (43 Patterns), `BANDS` (29 Presets) |
| 2 | Progression | Akkord-Parser, Takt-Modell, Quintenzirkel, Stufen, Reharmonisierung |
| 3 | Voicing / Scale Helper | Intervalle, Voicings, Stimmführung, Skalen |
| 4 | Generatoren | `genDrums`, `genBass`, `genChords`, `genArp`, `genMelody` |
| 5 | Mutation / Lock / Reroll | Seed-Verwaltung, Takt-Locks |
| 6 | Web MIDI | Port-Enumeration, `requestMIDIAccess`, Statusanzeige |
| 7 | Transport | Lookahead-Scheduler, `sendAt`, Kanalsperre, MIDI-Monitor |
| 7b | Cubase Sync | MIDI Clock (Ausgang), MMC, Count-In |
| 7c | MIDI-Clock-Slave | Clock-Eingang, Tempoerkennung, externe Transportsteuerung |
| 8 | SMF Export | Standard MIDI File Type 1 |
| 9 | UI-Aufbau | `buildLanes`, `buildChMon`, `buildPiano`, `buildPresets`, `buildBands` |
| 10 | Events | Alle Handler, Shortcuts, Kommandoleiste, Klapp-Panels |
| 11 | Init | Reihenfolge des Startens |

---

## Datenmodell

### Lane

```js
{
  id:'bass',            // Schlüssel für GEN, LANE_STYLES, DOM-IDs
  name:'BASS',
  color:'#ff6622',      // durchgängig: Border, Kanal-Monitor, Klaviatur, Topbar
  on:true,              // sendet oder nicht
  ch:0,                 // 0-basiert! Anzeige immer ch+1
  oct:2,                // Basisoktave (bei DRUMS ungenutzt)
  vel:94,               // 30..127, wirkt als Fader auch im laufenden Loop
  dens:70,              // Dichte in %
  style:'walking',      // Schlüssel in LANE_STYLES[id]
  lock:false,           // schützt vor Reroll und Mutation
  seed:1234,            // Determinismus
  prog:-1               // Program Change, -1 = keiner
}
```

Für ARP zusätzlich `rate`, `arpOct`, `gate`; für DRUMS `fill`.

**Kanäle sind intern 0-basiert.** Jede Anzeige rechnet `+1`. Diese Grenze sauber
zu halten ist der häufigste Fehlerpunkt beim Erweitern.

### Take

Ein *Take* ist die auskomponierte Akkordfolge über alle Lanes:

```js
{ lanes: { drums:[ev,…], bass:[ev,…], … }, totalTicks: n }
ev = { t, d, m, v }   // Tick, Dauer in Ticks, MIDI-Note, Velocity
```

`buildTake(includeAll)` erzeugt ihn. Mit `includeAll=true` werden auch
ausgeschaltete Lanes generiert — das kostet etwas Rechenzeit, macht aber das
Ein- und Ausschalten einer Lane im laufenden Loop verzögerungsfrei, weil nichts
neu gebaut werden muss.

### Clock-Slave-Zustand

```js
ext = { on, input, running, last, ivl[], mpt, bpm, ticks }
```

`ivl` sammelt die letzten 48 Clock-Abstände (= zwei Viertel), daraus ergeben sich
`mpt` (Millisekunden pro Tick) und `bpm`. Kurz genug für echte Tempoänderungen im
laufenden Takt, lang genug gegen Jitter.

### Scheduler-Zustand

```js
sched = { on, timer, ev[], idx, tickRef, msRef, loop, loopMax, loopTicks, active[], clkTick }
```

`ev` ist die flachgeklopfte, nach Tick sortierte Ereignisliste aller Lanes.
`idx` ist der Lesezeiger. `active` hält klingende Noten, damit STOP sie gezielt
beenden kann statt nur All-Notes-Off zu feuern.

---

## Sendepfad

**Alle** Kanal-Nachrichten laufen durch eine einzige Funktion:

```js
function sendAt(bytes, time) {
  if (!midiOutput) return;
  var st = bytes[0];
  if (st < 240) {                      // Kanal-Nachricht, kein System-Byte
    var ch = st & 15, ty = st & 240;
    if (!chAllowed(ch)) { monLine(ch, ty, bytes, true); return; }   // Kanalsperre
    bumpCh(ch); monLine(ch, ty, bytes, false);                      // Zähler + Monitor
  }
  midiOutput.send(bytes, time);
}
```

Das ist der Grund, warum Kanalsperre und Monitor beweiskräftig sind: es gibt
keinen zweiten Weg nach draußen. Clock und MMC laufen bewusst getrennt über
`sendSync()`, weil sie System-Nachrichten ohne Kanal sind und optional auf einen
anderen Port gehen können.

### Scheduler-Schleife

```
alle 20 ms:
  curTick  = tickRef + (now - msRef) / msPerTick
  horizon  = curTick + 220ms
  solange ev[idx].t < horizon:
     Lane aus? -> überspringen        ← Stummschalten wirkt sofort
     usedCh[ch] = 1                   ← erst hier, damit Panic keine fremden Kanäle trifft
     sendAt(NoteOn,  timestamp)
     sendAt(NoteOff, timestamp)
```

Der Lookahead von 220 ms ist der Kompromiss: groß genug, dass der Eventloop das
Timing nicht verhagelt, klein genug, dass STOP als sofort empfunden wird.

### Zwei Zeitquellen

`sched.tickRef` ist die Position, `sched.msRef` der Zeitstempel dazu. Wer die
beiden fortschreibt, hängt vom Modus ab:

| | Position kommt aus | `mpt` kommt aus |
|---|---|---|
| **Intern** (Default) | Wanduhr: `tickRef += (now-msRef)/mpt` in `schedTick` | BPM-Regler |
| **Clock-Slave** | eingehenden F8-Ticks: `tickRef += CLKDIV` in `extClock` | gemessenem Clock-Abstand |

Im Slave-Modus **schreibt `schedTick` die beiden Felder nicht mehr**, sondern
interpoliert nur für den Horizont:

```js
var curTick = sched.tickRef + (now - sched.msRef) / mpt;
if (!slave) { sched.tickRef = curTick; sched.msRef = now; }
```

Dadurch bleibt die gesamte Nachfolgelogik — Lookahead, Note-Offs, Loop-Wechsel,
Taktanzeige — unverändert. Nur die Frage „wo sind wir gerade" wird woanders
beantwortet. Das ist der Grund, warum der Slave-Modus mit rund 80 Zeilen
auskommt statt mit einem Umbau des Sequencers.

Der Loop-Wechsel muss beide Fälle kennen: im Slave-Modus wird `tickRef`
dekrementiert (`-= loopTicks`), statt aus `curTick` neu gesetzt zu werden —
sonst würde die Wanduhr die clock-getriebene Zeitbasis überschreiben.

**Eine Zeitbasis, nicht zwei.** Der MIDI-Eingangs-Handler stempelt bewusst mit
`performance.now()` und nicht mit `e.timeStamp` des Events. Chrome liefert für
MIDI-Events nicht zwingend dieselbe Epoche wie für `performance.now()`; mischt
man beide, wird `curTick` um Größenordnungen falsch und es geht keine Note mehr
raus — während Taktzähler und BPM-Anzeige weiterhin gesund aussehen, weil die
Position aus den Clock-Ticks kommt. `schedTick` hat zusätzlich eine Notbremse:
liegt `now - msRef` außerhalb von 0…4000 ms, wird resynchronisiert.

### Rückkopplungsschutz

Generator und DAW hängen typischerweise am selben IAC-Bus. Solange
`extSlaving()` wahr ist, geben `clockStart`, `clockStop`, `pumpClock`,
`cubaseStart` und `cubaseStopCmd` sofort zurück. Sonst würde der Generator die
Clock, der er folgt, selbst wieder aussenden.

---

## UI-Zustand

Drei getrennte `localStorage`-Schlüssel:

| Schlüssel | Inhalt |
|---|---|
| `midiperfect2.lanes.v1` | Lane-Zustand (on, ch, prog, style, oct, vel, dens, lock, fill) |
| `midiperfect2.panels.v1` | Welche Panels eingeklappt sind |

(Der Slave-Modus wird bewusst **nicht** persistiert — er hängt an einem konkreten
MIDI-Port, der beim nächsten Start ein anderer sein kann.)

Die Kommandoleiste **dupliziert keine Logik**. Ihre Buttons delegieren an die
echten Bedienelemente (`btnPlay.click()`), und ein `requestAnimationFrame`-Loop
spiegelt Zustand zurück (disabled, Taktzähler, Beat-Dots). Damit gibt es weiterhin
nur eine Quelle der Wahrheit.

Layout: `column-width` statt CSS-Grid. Grund: bei Grid erben alle Panels einer
Zeile die Höhe des höchsten, was bei stark unterschiedlichen Panelhöhen große
Löcher erzeugt. Multicolumn packt vertikal; `column-span:all` erlaubt trotzdem
volle Breite für die großen Panels.

---

## Erweitern

### Neue Lane

1. Eintrag in `LANES` mit freiem `ch` und eigener `color`
2. `LANE_STYLES.<id>` mit mindestens einem Style anlegen
3. Generatorfunktion schreiben und in `GEN` registrieren
4. `GM_DEFAULT.<id>` setzen

Das UI baut sich aus `LANES` auf — Lane-Panel, Kanal-Monitor, Topbar-Chips und
Export folgen automatisch. Nur die Tastatur-Shortcuts sind auf `1`–`9` begrenzt.

### Neuer Style

Ein Eintrag `[value, Label, Gruppe]` in `LANE_STYLES.<lane>` plus ein `case` im
zugehörigen Generator. Die Gruppe erzeugt automatisch ein `<optgroup>`.

### Neues Drum-Pattern

`DRUMPAT` nimmt Grid-Strings:

```js
rock8: { name:'Rock 8tel', g:'Rock / Pop', grid:16, rows:[
  [DK.kick,  'X-------X-------'],
  [DK.snare, '----X-------X---'],
  [DK.hhC,   'x-x-x-x-x-x-x-x-']
]}
```

`X` = Akzent, `x` = normal, `-` = Pause. Die Länge des Strings muss `grid`
entsprechen.

### Neues Band-Preset

Ein Eintrag in `BANDS`:

```js
key: ['Anzeigename',
      {drums:'styleId', bass:'…', chords:'…', arp:'…', melody:'…'},
      {on:{drums:1,bass:1,chords:1,arp:0,melody:1}, bpm:132, swing:62}]
```

Ob das Preset die Lane-Schaltung überschreiben darf, entscheidet der Chip
*Lane On/Off übernehmen* zur Laufzeit.

---

## Konventionen

- Klassen mit `Cls_`-Präfix gibt es hier nicht — das ist VB.NET-Konvention und
  auf ES5-Funktionen nicht übertragbar. Stattdessen: Funktionsnamen nach Verb
  (`buildX`, `genX`, `sendX`, `renderX`, `saveX`, `loadX`).
- Kommentare stehen dort, wo eine Entscheidung erklärt werden muss, nicht dort,
  wo der Code sich selbst erklärt.
- Kein `let`/`const`, keine Arrow Functions, keine Template Literals — die Datei
  soll überall ohne Transpiler laufen und in einem Rutsch lesbar bleiben.
- Jede Änderung am Sendepfad gehört in `sendAt`, nicht daneben.
