# Changelog

Der Build-Stempel steht sichtbar im Header der Anwendung. Er ist kein Zierrat:
`file://`-Seiten hängen hartnäckig im Chrome-Cache, und ohne sichtbare Version
lässt sich nicht unterscheiden zwischen „der Fix wirkt nicht" und „der Fix ist
gar nicht geladen".

---

## BUILD 2026-07-31-K — Blues-Werkstatt: Groove-Engine und Form

**Problem:** Die Seite erzeugte korrekte Patterns, aber keine Musik, zu der sich
üben lässt. Vier konkrete Mängel:

1. **Swing war global.** Ein einziger Regler für alle fünf Lanes. Wer Drums
   shufflen und den Bass gerade lassen wollte, konnte das — nur war das genau
   die Einstellung, die man *nie* will. Umgekehrt gab es keine Möglichkeit,
   eine einzelne Lane bewusst herauszunehmen.
2. **Statische Velocity.** Die Generatoren setzten feste Anschlagswerte mit
   kleinen Offsets. Die einzige Streuung kam aus `Humanize` und wurde erst
   beim Senden aufgeschlagen — der SMF-Export klang also anders als die
   Wiedergabe.
3. **Takt 12 klang wie Takt 4.** Ohne Turnaround verschwimmt die Form. Nach
   zehn Minuten hört man nur noch die Schleife, nicht mehr die Harmonik.
4. **Kein Weg zur Transposition.** Progressionen mussten von Hand neu getippt
   werden. Praktisch heißt das: man übt in A und E und in nichts sonst.

**Geändert**

- **Swing pro Lane** (`L.swing`, `-1` = folgt global). Darüber liegt die
  **Groove-Kopplung**: solange sie aktiv ist, ziehen DRUMS, BASS und CHORDS
  zwingend den globalen Wert, ihre Auswahlfelder sind sichtbar gesperrt.
  Auseinanderlaufende Rhythmusgruppe ist kein Feature, das man versehentlich
  einschalten können sollte.
- `applySwing` greift jetzt zusätzlich auf die 16tel-Offbeats und kürzt Noten,
  die sonst in den verschobenen Offbeat hineinlaufen würden.
  **Triolenraster bleibt unangetastet:** Patterns mit `grid:12`
  (Blues Shuffle, Jazz Ride, Purdie) sind bereits triolisch notiert; ein
  zweiter Shuffle darüber ergäbe Unsinn. Die Prüfung läuft über die Position
  relativ zur Viertel (`t % Q`), nicht über den Style-Namen.
- **Velocity-Streuung pro Lane** (`L.vspread`), **deterministisch aus dem
  Lane-Seed** statt aus `Math.random()`. Damit klingt derselbe Seed
  reproduzierbar gleich und der Export liefert exakt das Gehörte.
- **Backbeat-Akzent auf 2 und 4.** Nicht als reine Addition: der Akzent
  bekommt +60 % des Reglerwerts, alles andere −40 %. Reines Draufrechnen
  lief bei Drum-Velocity 100 plus Pattern-Akzent in die 127er-Sättigung —
  der Backbeat verschwand genau dann, wenn man ihn am deutlichsten wollte.
- **Blues-Werkstatt** als neues Panel:
  - **Formgenerator** stufenbasiert (Halbtonabstand + Akkordtyp), damit jede
    Form in jeder Tonart verfügbar ist: 12-Bar Standard, 12-Bar Slow mit
    9er-Voicings, Jazz-Blues, Minor-Blues, 8-Bar, 16-Bar.
  - **Turnaround** für Takt 11–12: V7, VI7→V7, ii7→V7, #IV°7→V7, bVI7→V7.
  - **Quick-Change** (Takt 2 wird zur IV-Stufe).
  - **Turnaround-Fill** erzwingt einen Fill im letzten Takt der Form.
  - **Transposition** ±Halbton, Zieltonart direkt, und ein Übungs-Zirkel
    G → C → F → Bb → Eb → A → D → E. Dafür merkt sich `parseToken` jetzt das
    rohe Suffix (`c.suf`), sonst ginge beim Umschreiben `m7b5` oder `7b9`
    verloren. Tonartabhängige Vorzeichenwahl über eine Flat-Key-Menge.
  - **Tempofelder** Slow Blues 12/8 (60–75, Swing 98), Shuffle (95–130,
    Swing 65), Rock-Blues gerade (110–140, Swing 0), Funk-Blues (100–115,
    Swing 16). Sie setzen Styles, Swing, Backbeat und Tempo; jeder weitere
    Klick geht 5 BPM weiter durch den Bereich. Sie schalten außerdem MELODY
    und ARP ab — die Melodie spielt der Mensch, und programmierte
    Rhythmusgitarre verrät sich ohnehin sofort.
  - **Chorus-Dynamik:** Bogen über 2–8 Durchläufe. Velocity steigt, die
    Hi-Hat wandert ab einem einstellbaren Chorus aufs Ride, im Spitzen-Chorus
    liegt die Glocke **nur auf den Vierteln** — eine durchgehende Ride-Bell
    auf allen Achteln ist kein Drummer. Pro Chorus geht ein Expression-CC
    (Nummer frei wählbar) auf alle Nicht-Drum-Kanäle.

**Architektur:** Alle vier Bearbeitungsschritte (Swing, Streuung, Backbeat,
Chorus) hängen in `buildTake()` hinter dem Generator und **werden in den Take
gebacken**. Wiedergabe und SMF-Export durchlaufen damit dieselbe Kette. Der
Chorus-Bogen läuft im Export über die Wiederholungen mit (`gStageOverride`).

**Getestet** headless über Playwright gegen die Datei: Formgenerator in allen
Tonarten, Transposition mit Suffix-Erhalt, Swing-Verschiebung und Nicht-
Verschiebung auf Triolenraster, Kopplungslogik, Determinismus der
Velocity-Streuung, Backbeat ohne Sättigung, alle Chorus-Stufen, Tempofelder,
Turnaround-Fill, SMF-Export. Keine Konsolenfehler.

---

## BUILD 2026-07-30-D — Kommandoleiste, volle Breite, Klapp-Panels

**Problem:** Play/Stop lagen im zweiten Panel und scrollten weg. Nach jeder
Änderung an einer Lane musste man ganz nach oben scrollen. Gleichzeitig waren
die Panels auf `max-width:1180px` begrenzt — auf einem 2560-px-Display blieben
über 50 % der Breite ungenutzt, während die Seite ~4000 px hoch war.

**Geändert**

- **Sticky Kommandoleiste** (49 px) mit PLAY / STOP / Reroll, Taktzähler,
  Beat-Dots und fünf Lane-Schnellschaltern. Chips zeigen Farbe, Zustand und
  Zielkanal. Klick = an/aus, Shift+Klick = Solo.
  Die Leiste dupliziert keine Transportlogik, sondern delegiert an die
  bestehenden Buttons und spiegelt deren Zustand per `requestAnimationFrame`.
- **Tastatur erweitert:** `1`–`5` Lane an/aus, `⇧1`–`⇧5` Solo.
  Umsetzung über `e.code` statt `e.key`, weil `Shift+1` als `!` ankommt.
- **Volle Breite** über CSS-Multicolumn (`column-width:560px`) statt festem
  `max-width`. Grid wäre die naheliegende Wahl gewesen, erzeugt aber
  300-px-Löcher, weil alle Panels einer Zeile die Höhe des höchsten erben.
  `column-span:all` gibt den großen Panels trotzdem volle Breite.
- Lanes innerhalb ihres Panels als Grid — bei 2560 px stehen vier nebeneinander.
- **Klapp-Panels:** Klick auf die Panel-Überschrift, Zustand in `localStorage`.
- Textkontraste auf WCAG-AA-Niveau angehoben: Panel-Untertitel, Erklärtexte und
  Kanal-Monitor-Ziffern lagen bei 1.79:1 bis 2.94:1, jetzt 3.95:1 bis 7.5:1.
- `transform:scale(1.03)` beim Hovern entfernt — ein Ziel, das unter dem Cursor
  wegwandert, kostet Klickgenauigkeit.

**Ergebnis:** Seitenhöhe bei 2560 px von ~4000 px auf 2169 px.

---

## BUILD 2026-07-30-C — Kanalsperre und MIDI-Monitor

**Problem:** In Cubase kam MIDI auf Kanälen an, die keiner eingeschalteten Lane
gehörten. Unklar, ob der Generator oder die DAW schuld war.

**Geändert**

- **Build-Stempel** im Header, damit „hart neu geladen?" beantwortbar wird.
- **Zentrale Sendeschleuse:** `sendAt` ist der einzige Weg für Kanal-Nachrichten
  nach draußen. Dort hängen jetzt Kanalsperre und Monitor.
- **Kanalsperre** blockt jede Nachricht auf Kanälen ohne eingeschaltete Lane —
  inklusive Program Change und All-Notes-Off.
- **MIDI-Monitor** protokolliert jede ausgehende Nachricht mit Kanal und Typ,
  blockierte in Rot.
- Kanal-Monitor zählt jetzt alle gesendeten Nachrichten, nicht nur Noten.
  Vorher zählte `flashCh`, was CC und Program Change unterschlug.

**Befund aus der Messung:** Mit DRUMS solo sendete die Seite ausschließlich auf
Ch10 — 41 Nachrichten, kein einziges Byte woanders. Damit war der Generator
entlastet und der Fehler in Cubase lokalisiert. Siehe
[CUBASE-SETUP.md](CUBASE-SETUP.md), Abschnitte 4 und 5.

---

## BUILD 2026-07-30-B — Solo, Zustandspersistenz, Preset-Schutz

**Problem:** „Nur Drums eingeschaltet" ließ sich nicht reproduzieren. Nach jedem
Reload standen wieder BASS, CHORDS und MELODY auf ON — exakt die Kanäle 1, 2
und 4, die in der DAW auftauchten.

**Ursachen**

1. `LANES` ist ein Literal im Code. Jeder Reload setzte auf Werkseinstellung
   zurück. Der Button *HTML speichern* half nicht — der serialisiert nur das DOM,
   nicht die JavaScript-Objekte.
2. `applyBand()` setzte `L.on` aus dem Preset hart. Ein Klick auf ein
   Band-Preset machte jede Solo-Einstellung lautlos zunichte.

**Geändert**

- **SOLO-Chip pro Lane.** Nochmal klicken hebt auf.
- **Lane-Zustand persistiert** in `localStorage`: on, ch, prog, style, oct, vel,
  dens, lock, fill. Beim Start steht im Log, was wiederhergestellt wurde.
  Button *Lane-Zustand zurücksetzen* verwirft ihn.
- **Chip „Lane On/Off übernehmen"** — aus, und Band-Presets setzen nur noch
  Styles.

---

## BUILD 2026-07-30-A — Routing-Fixes

**Geändert**

- `usedCh` wurde in `loadLoopEvents()` für **alle** Lanes gesetzt, auch für
  ausgeschaltete. Bei STOP feuerte `panic()` daraufhin CC123 und CC120 auf
  Kanälen, die gar nicht spielten — in der DAW sichtbar als Datenverkehr auf Ch1
  und Ch2. Jetzt wird `usedCh` erst im Scheduler gesetzt, nach der `L.on`-Prüfung.
- `setPrograms(true)` sendete Program Change auf alle Lanes unabhängig vom
  Ein-/Aus-Zustand. Jetzt nur noch für aktive Lanes; der Wert wird weiterhin für
  alle gesetzt und landet im Export.
- **Routing-Anzeige** unter den Kanal-Buttons: zeigt permanent das Ist-Mapping,
  graut ausgeschaltete Lanes aus, warnt bei doppelt belegten Kanälen.
- Die beiden Routing-Buttons sind jetzt eine Toggle-Gruppe mit sichtbarem
  Zustand. Vorher blieb nach dem Klick kein Hinweis, was passiert war.
- „auf Ch 1-4 verteilen" verteilt jetzt **ab dem gewählten Startkanal**
  fortlaufend über alle 16 Kanäle und überspringt dabei Ch10.
  Label entsprechend geändert.
