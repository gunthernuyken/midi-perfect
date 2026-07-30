# Changelog

Der Build-Stempel steht sichtbar im Header der Anwendung. Er ist kein Zierrat:
`file://`-Seiten hängen hartnäckig im Chrome-Cache, und ohne sichtbare Version
lässt sich nicht unterscheiden zwischen „der Fix wirkt nicht" und „der Fix ist
gar nicht geladen".

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
