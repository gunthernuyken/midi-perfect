# MIDI PERFECT 2

**[▶ Live-Demo](https://gunthernuyken.github.io/midi-perfect/)** · [English version](README.md) · MIT-Lizenz

Multi-Lane MIDI-Generator als **einzelne HTML-Datei**. Läuft ohne Server, ohne Build,
ohne Abhängigkeiten direkt im Browser und spielt über Web MIDI live in eine DAW —
entwickelt und getestet gegen **Cubase Pro 14** auf macOS über den IAC-Treiber.

```
Browser (Web MIDI)  ──IAC-Treiber Bus 1──▶  Cubase  ──▶  HALion Sonic / Groove Agent / EZdrummer
```

Fünf unabhängige Lanes (DRUMS, BASS, CHORDS, ARP, MELODY) erzeugen aus einer
Akkordfolge deterministisch Patterns, senden sie auf getrennten MIDI-Kanälen und
lassen sich live umschalten, würfeln und mutieren. Zusätzlich: MIDI Clock und MMC
für Transport-Sync, sowie SMF-Export (Type 1, eine Spur pro Lane).

> **Chrome oder Edge nötig.** Web MIDI ist in Safari und Firefox nicht implementiert.

Die Oberfläche ist **zweisprachig** — der Umschalter `DE · EN` sitzt in der Kommandoleiste, die Wahl wird gemerkt.

---

## Wozu das Ganze

Backing-Tracks zum Improvisieren sind entweder feste Aufnahmen, die man nach dem
dritten Durchlauf auswendig kann, oder Abo-Werkzeuge in fremder Cloud. Dieses hier
erzeugt das Material lokal, in jedem Durchlauf anders, in jeder Tonart, und spielt
deine eigenen Instrumente an, statt eigene Klänge mitzubringen.

Dass alles in einer Datei steckt, ist keine Spielerei: das Werkzeug funktioniert
damit auch in fünf Jahren noch, braucht keine Installation und passt auf einen
USB-Stick.

---

## Inhalt

| Datei | Zweck |
|---|---|
| `MIDI-PERFECT-2.html` | Die komplette Anwendung. Eine Datei, ~224 KB. |
| `CUBASE-SETUP.md` | Cubase-Einrichtung + die Fallstricke beim MIDI-Routing |
| `ARCHITEKTUR.md` | Aufbau des Codes, Datenmodell, Erweiterungspunkte |
| `CHANGELOG.md` | Build-Historie mit Begründung der Änderungen |

---

## Schnellstart

1. **IAC-Treiber aktivieren** (macOS): Audio-MIDI-Setup → Fenster → MIDI-Studio →
   IAC-Treiber doppelklicken → *Gerät ist online* aktivieren, Bus 1 anlegen.
2. `MIDI-PERFECT-2.html` in **Chrome oder Edge** öffnen. Firefox und Safari
   unterstützen die Web MIDI API nicht.
3. MIDI-Zugriff erlauben. Bei der Abfrage **SysEx mit freigeben** — ohne SysEx
   funktioniert MMC nicht (MIDI Clock aber schon).
4. Als MIDI Output `IAC-Treiber Bus 1` wählen.
5. In Cubase pro Kanal eine MIDI-Spur anlegen → siehe [CUBASE-SETUP.md](CUBASE-SETUP.md).
   **Das ist der Schritt, an dem es üblicherweise klemmt.** Lies ihn.
6. PLAY.

### Browser-Voraussetzungen

| | Web MIDI | SysEx / MMC |
|---|---|---|
| Chrome | ✅ | ✅ (nach Freigabe) |
| Edge | ✅ | ✅ |
| Safari | ❌ | — |
| Firefox | ❌ | — |

Beim Öffnen als `file://` merkt sich Chrome die MIDI-Freigabe nicht dauerhaft.
Nach einem harten Reload (`Cmd+Shift+R`) muss sie erneut erteilt werden.

---

## Bedienung

### Kommandoleiste (klebt oben, immer erreichbar)

```
▶ PLAY   ■ STOP   🎲   1.1   ● ● ● ●   │   DRUMS 10   BASS 1   CHORDS 2   ARP 3   MELODY 4
```

Die Lane-Chips zeigen Farbe, Zustand und Zielkanal.
**Klick** = Lane an/aus · **Shift+Klick** = Solo (nochmal = alle wieder an).

### Tastatur

| Taste | Wirkung |
|---|---|
| `Space` | Play / Stop |
| `1` … `5` | Lane an/aus (Reihenfolge wie in der Leiste) |
| `⇧1` … `⇧5` | Solo auf diese Lane |
| `R` | Reroll aller nicht gesperrten Lanes |
| `D` | Alle Styles würfeln |

Shortcuts greifen nicht, während der Fokus in einem Eingabefeld oder Auswahlfeld steht.

### Panels

Jede Panel-Überschrift ist ein Schalter — Klick klappt zu. Der Zustand überlebt den
Reload. `MIDI Connection` und `Cubase Sync` sind Einrichtung; nach dem ersten Mal
kann man beide dauerhaft zuklappen.

| Panel | Inhalt |
|---|---|
| MIDI Connection | Port, Kanal-Routing, GM-Sounds, Kanalsperre, MIDI-Monitor |
| Transport & Makros | Play/Stop, BPM, Swing, Humanize, Energy, Complexity, Loop, Mutation |
| Blues-Werkstatt | Formgenerator, Turnaround, Quick-Change, Transposition, Groove-Kopplung, Backbeat, Tempofelder, Chorus-Dynamik |
| Cubase Sync | MIDI Clock, MMC Play/Record/Stop, Count-In, Tempo-Handshake |
| Progression | Akkordfolge als Text oder Blöcke, Takt-Locks gegen Mutation |
| Harmonie-Engine | Quintenzirkel, Stufen, Vorschläge, Reharmonisierung, Generator |
| Lanes | Pro Lane: Style, Kanal, Sound, Oktave, Velocity, Density, Swing, Vel-Streuung |
| MIDI Export | SMF Type 1, eine Spur pro Lane |
| Keyboard | Live-Anzeige der klingenden Noten, eingefärbt nach Lane |

---

## Konzepte

### Lanes

Fünf feste Lanes. Jede hat einen eigenen MIDI-Kanal, einen eigenen Style-Katalog,
einen eigenen Zufalls-Seed und einen Lock. Die Standardbelegung:

| Lane | Kanal | Default-Style | GM-Default |
|---|---|---|---|
| DRUMS | 10 | Rock 8tel | Standard Kit |
| BASS | 1 | Walking Bass | E-Bass (33) |
| CHORDS | 2 | Swing Comping | Flügel (0) |
| ARP | 3 | Up/Down | Clean-Gitarre (27) |
| MELODY | 4 | Motif | Trompete (56) |

**DRUMS bleibt bei den Routing-Buttons immer auf Kanal 10.** Das ist Absicht —
GM-Drums liegen dort. Wer Groove Agent benutzt, muss das in Cubase umsetzen,
nicht hier; siehe [CUBASE-SETUP.md](CUBASE-SETUP.md).

Der Lane-Zustand (an/aus, Kanal, Style, Sound, Oktave, Velocity, Density, Lock)
wird in `localStorage` gespeichert und beim nächsten Öffnen wiederhergestellt.
Der Button *Lane-Zustand zurücksetzen* verwirft ihn und lädt die Werkseinstellung.

### Determinismus

Die Generatoren sind **seed-basiert und deterministisch**: gleicher Seed, gleiche
Akkordfolge, gleicher Takt → gleiches Pattern. Auch die Velocity-Streuung der
Groove-Engine wird aus dem Lane-Seed abgeleitet und ist damit reproduzierbar.
Nur `Humanize` (Timing- und Velocity-Jitter) ist echt zufällig und wird erst beim
Senden aufgeschlagen.

Daraus folgt: Ein Pattern, das gefällt, lässt sich mit `Lock` einfrieren, während
alles andere weiter mutiert.

### Groove-Engine

Zwischen Generator und Ausgabe liegen vier Bearbeitungsschritte, die in
`buildTake()` **in den Take gebacken** werden — Wiedergabe und SMF-Export
durchlaufen also dieselbe Kette und klingen identisch:

1. **Swing**, pro Lane auflösbar. `100 %` = volles Triolen-Feel (der Offbeat
   sitzt auf der dritten Triole). Blues-Shuffle liegt bei 62–68, Slow Blues 12/8
   bei 95–100, Funk bei 12–20.
   Patterns auf Triolenraster (`grid:12`) werden **nicht** zusätzlich
   verschoben — sie sind bereits triolisch notiert. Die Erkennung läuft über die
   Notenposition relativ zur Viertel, nicht über den Style-Namen.
2. **Velocity-Streuung**, pro Lane, deterministisch aus dem Seed.
3. **Backbeat-Akzent** auf 2 und 4. Der Akzent wird nicht nur addiert, sondern
   alles andere gleichzeitig leicht abgesenkt — sonst läuft die Snare in die
   127er-Sättigung und der Akzent verschwindet.
4. **Chorus-Dynamik**, siehe unten.

Die **Groove-Kopplung** (Blues-Werkstatt) zwingt DRUMS, BASS und CHORDS auf
denselben Swing-Wert. Sie ist ab Werk an; ohne sie lässt sich die
Rhythmusgruppe versehentlich auseinanderziehen.

### Chorus-Dynamik

Über 2–8 Durchläufe baut sich der Track auf und fällt wieder zurück. Das ist ein
**Arrangement, keine Lautstärkerampe** — jeder Durchlauf legt eine Schicht dazu:

| Stufe | Drums | Bass | Chords |
|---|---|---|---|
| 0 | Kick/Snare, Hi-Hat nur auf den Vierteln, kein Crash | auf 1 und 3 | ein Akkord pro Takt |
| 1 | Hi-Hat durchgehend | auf allen Vierteln | auf 1 und 3 |
| 2 | volle Band, Crash und offene Hi-Hat | Achtel | durchgehend |
| 3 | zusätzlich Ride und Glocke | Achtel | durchgehend |

Die Auswahl läuft über die **Position im Takt** und die Notennummer, nicht über
Style-Namen — der Bogen greift damit für alle 137 Patterns. Kick, Snare,
Rimshot und Clap sind geschützt und werden nie ausgedünnt, sonst verliert der
Take den Puls statt nur an Dichte. Die Velocity steigt zusätzlich mit. Zusätzlich geht pro Chorus ein
**Expression-CC** (Nummer frei wählbar) auf alle Nicht-Drum-Kanäle — bei einer
Hammond lässt sich dort eine Drawbar-CC eintragen, dann zieht die Orgel über den
Bogen selbstständig auf.

Der Bogen läuft auch im **Export** über die Wiederholungen mit.

### Blues-Werkstatt

Formen sind **stufenbasiert** hinterlegt (Halbtonabstand zur Tonika + Akkordtyp),
nicht als Akkordnamen. Damit ist jede Form in jeder Tonart verfügbar:
12-Bar Standard, 12-Bar Slow mit 9er-Voicings, Jazz-Blues, Minor-Blues, 8-Bar,
16-Bar — kombinierbar mit Quick-Change und fünf Turnaround-Varianten für
Takt 11–12.

Die Transposition schreibt die Akkordzeile neu und erhält dabei die Suffixe
(`m7b5`, `7b9`, `:2`) sowie die Tonart der Harmonie-Engine. Der Übungs-Zirkel
geht bewusst über G, C, F, Bb, Eb, A, D, E — also auch durch die Tonarten, die
man auf der Gitarre sonst umgeht.

### Infinity / Mutation

Bei aktivem `INFINITY` würfelt der Generator am Ende jedes Loop-Durchlaufs einen
Teil der Takte und Lane-Seeds neu — die Wahrscheinlichkeit steuert der
`Mutation`-Regler. Gesperrte Takte (Klick auf einen Takt-Block) und gesperrte
Lanes bleiben unangetastet. Das ist als Übe- und Ideenmaschine gedacht: eine
Akkordfolge, die sich endlos weiterentwickelt, ohne je identisch zu sein.

### Scheduling

Der Sequencer arbeitet mit **Lookahead-Scheduling**: alle 20 ms wandern die
nächsten ~220 ms Musik mit exakten Timestamps in die MIDI-Queue
(`MIDIOutput.send(bytes, timestamp)`). Das ergibt stabile Timings trotz
JavaScript-Eventloop und hält gleichzeitig STOP praktisch sofort wirksam, weil
nie mehr als ein Fünftel einer Sekunde vorausgeplant ist.

Tempoänderungen, Style-Wechsel und Akkordänderungen greifen im laufenden Takt:
Der Take wird neu gebaut und der Zeiger auf die aktuelle Position vorgespult.
Bereits eingereihte Note-Offs behalten ihre Zeit, es bleibt also nichts hängen.

MIDI Clock läuft im selben Tick-Raster (24 Clocks pro Viertel) aus derselben
Zeitrechnung — dadurch driftet Cubase nicht gegen den Generator.

### Kanalsperre und MIDI-Monitor

Zwei Diagnosewerkzeuge im MIDI-Connection-Panel:

- **Kanalsperre** blockt vor dem Senden jede Nachricht auf Kanälen, die keiner
  eingeschalteten Lane gehören — inklusive Program Change und All-Notes-Off.
  Kommt in der DAW trotzdem etwas an, liegt es beweisbar nicht an dieser Seite.
- **MIDI-Monitor** protokolliert jede ausgehende Nachricht mit Kanal und Typ:

```
SEND  Ch10  NoteOn  36 v110
SEND  Ch10  NoteOff 36 v0
BLOCK Ch 1  CC     123 v0
```

Beides sitzt an einer einzigen zentralen Sendefunktion (`sendAt`), es kann also
keine Nachricht daran vorbei.

---

## Export

`MIDI Export` schreibt ein **Standard MIDI File Type 1** mit einer Spur pro
aktiver Lane, inklusive Tempo, Taktart, Spurnamen und Program Change. Die Datei
lässt sich direkt in Cubase ziehen; jede Lane landet auf einer eigenen Spur.

Program Change wird nur geschrieben, wenn im Lane-Panel ein Sound gewählt ist.
Steht dort *aus*, behält das Instrument in der DAW seinen eigenen Klang.

---

## Bekannte Grenzen

- **Nur Chrome/Edge.** Web MIDI ist in Safari und Firefox nicht implementiert.
- **Kein MIDI-Input.** Die Seite sendet ausschließlich; sie hört nicht zu.
- **Fünf feste Lanes.** Die Lane-Liste ist ein Array im Code, keine UI zum
  Hinzufügen. Erweitern ist trivial (siehe ARCHITEKTUR.md), aber nicht anklickbar.
- **Kein Undo für Lane-Einstellungen.** Nur die Progression hat eine Undo-Historie.
- **`localStorage` ist pro Dateipfad.** Verschiebt man die HTML-Datei, ist der
  gespeicherte Zustand weg.

---

## Mitmachen

Issues und Pull Requests sind willkommen. Die ganze Anwendung ist eine Datei —
in jedem Editor zu öffnen, die Abschnitte sind nummeriert und kommentiert.
[ARCHITEKTUR.md](ARCHITEKTUR.md) beschreibt Datenmodell und Erweiterungspunkte.

---

## Kaffeekasse

Wenn dir das einen Nachmittag MIDI-Routing erspart hat, freue ich mich über
[einen Kaffee](https://paypal.me/guenuy). Völlig freiwillig — das Projekt
bleibt so oder so MIT-lizenziert und kostenlos.

---

## Lizenz

[MIT](LICENSE) © 2026 Günther Nuyken
