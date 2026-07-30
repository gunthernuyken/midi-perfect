# Changelog

Der Build-Stempel steht sichtbar im Header der Anwendung. Er ist kein Zierrat:
`file://`-Seiten hängen hartnäckig im Chrome-Cache, und ohne sichtbare Version
lässt sich nicht unterscheiden zwischen „der Fix wirkt nicht" und „der Fix ist
gar nicht geladen".

---

## BUILD 2026-07-30-I — DAW-Sync-Einstellungen bleiben erhalten

Das ganze *DAW Sync*-Panel überlebt jetzt den Reload: Clock-Ausgang, SLAVE,
Clock-Eingang, Sync-Ausgang, Transportbefehl, *Cubase bei PLAY*, MMC-Device,
Count-In und *Bei STOP*.

**MIDI-Ports werden über den Namen gespeichert, nicht über die ID.** Die ID
vergibt der Browser pro Sitzung neu; der Name bleibt. Ist der gespeicherte Port
beim nächsten Start nicht da, bleibt SLAVE aus und es steht eine Warnung im Log —
lieber ehrlich stumm als heimlich am falschen Port hängen.

Werkseinstellung von *Cubase bei PLAY* auf **Play** geändert (vorher
*nichts senden*); das ist der Arbeitsmodus, nicht die Ausnahme.

---

## BUILD 2026-07-30-H — Tempomessung und MMC im Slave-Modus

Zwei Fehler, die erst im echten Betrieb mit Cubase auffielen.

**1 · Tempo wurde etwa halb so hoch gemessen wie es war.**
Bei 111 BPM Projekttempo zeigte die Slave-Statuszeile 61 BPM.

Ursache: Chrome stellt MIDI-Realtime-Bytes gebündelt zu — mehrere `F8` im
selben Eventloop-Durchlauf mit nahezu 0 ms Abstand, danach eine große Lücke.
Die Tempomessung mittelte über Einzelabstände und musste die Nullabstände als
Ausreißer wegfiltern; übrig blieben nur die Lücken, also ein zu großer
Mittelwert und damit ein zu niedriges Tempo.

Gefixt: Tempo kommt jetzt aus der **Gesamtspanne der letzten 96 Clocks geteilt
durch deren Anzahl**. Das ist unabhängig davon, ob der Browser einzeln oder
gebündelt zustellt. Verifiziert mit einem absichtlich bündelnden Testmaster
(4 Clocks am Stück): vorher rund die Hälfte, jetzt 110 statt 111 BPM.

Auf die Wiedergabeposition hatte der Fehler keinen Einfluss — die zählt
Clock-Ticks und misst keine Zeit. Betroffen waren die Anzeige, der mitlaufende
BPM-Regler und das Tempo im SMF-Export.

**2 · Der große PLAY-Button startete Cubase nur beim ersten Mal.**

Ursache: Build F unterdrückte im Slave-Modus pauschal *alle* MMC-Sendungen —
gedacht als Rückkopplungsschutz, aber zu grob. MMC ist ein Befehlskanal, keine
Clock. Genau die Kombination ist der sinnvolle Arbeitsablauf: PLAY im Generator
startet Cubase per MMC, Cubase schickt daraufhin sein `FA`, beide laufen
synchron los.

Gefixt: MMC ist im Slave-Modus wieder aktiv. Der Rückkopplungsschutz sitzt jetzt
präzise da, wo er hingehört — ein `ext.busy`-Flag während der Verarbeitung eines
eingehenden `FA`/`FC`. Ein von Cubase kommendes Start oder Stop geht dadurch
nicht als MMC an Cubase zurück. Der Clock-Ausgang bleibt im Slave-Modus
weiterhin komplett gesperrt.

Zusätzlich: Drückt man PLAY, während die DAW steht, zeigt die Statuszeile jetzt
**„bereit – wartet auf DAW-Start"** statt scheinbar zu hängen.

**Verifiziert** über drei Play/Stop-Zyklen gegen einen Fake-Cubase, der auf MMC
reagiert: jedes Mal startet und stoppt der Transport auf beiden Seiten, Noten
fließen in jeder Runde.

---

## BUILD 2026-07-30-G — Clock-Slave: Zeitbasis-Fix

**Problem:** Im Slave-Modus lief die Synchronisation sichtbar korrekt — Clocks
wurden gezählt, das Tempo erkannt, der BPM-Regler folgte Cubase — aber es kam
**keine einzige Note** in der DAW an. Nur während einer Tempoänderung war kurz
etwas zu hören.

**Ursache:** Zwei verschiedene Zeit-Epochen im selben Rechenweg.
`extMessage` übernahm `e.timeStamp` des eingehenden MIDI-Events als
`sched.msRef`, `schedTick` rechnete danach mit `performance.now()`. Wenn Chrome
für MIDI-Events eine andere Epoche liefert als für `performance.now()`, wird

```js
curTick = sched.tickRef + (now - sched.msRef) / mpt
```

um Größenordnungen falsch. Der Lookahead-Horizont landet weit in der
Vergangenheit, die Bedingung `ev[idx].t < horizon` trifft nie zu — und weil
`tickRef` weiterhin sauber aus den Clock-Ticks kommt, sehen Taktzähler,
BPM-Anzeige und Clock-Zähler völlig gesund aus. Die Anzeige lügt nicht, sie misst
nur etwas anderes als der Scheduler.

Das kurze Aufflackern bei Tempoänderungen kam von `extSpp`, das `msRef` auf
`performance.now()` zurücksetzte — bis der nächste F8-Tick es wieder verstellte.

**Geändert**

- `extMessage` nimmt ausschließlich `performance.now()`. Die Handler-Latenz liegt
  unter einer Millisekunde und ist irrelevant, weil das Tempo über 48 Clocks
  gemittelt wird.
- **Notbremse in `schedTick`:** liegt `now - msRef` außerhalb von 0…4000 ms,
  wird die Zeitbasis resynchronisiert statt weitergerechnet, mit Log-Eintrag.
- **Doppelter Loop-Wechsel entfernt.** `extClock` und `schedTick` hätten beide
  `loopTicks` abgezogen — der Zeiger wäre einen kompletten Durchlauf zu weit
  zurückgesprungen. Der Loop-Wechsel liegt jetzt allein bei `schedTick`.

**Regressionstest** mit absichtlich verschobener Event-Epoche (+1,78·10¹²):
vorher 0 Noten, nachher durchgehende Wiedergabe über mehrere Loop-Grenzen
(48 → 98 → 150 Noten), Loop-Zähler und Taktanzeige laufen mit, null Rückläufer
auf den Bus.

---

## BUILD 2026-07-30-F — MIDI-Clock-Slave

**Problem:** Tempo-Sync mit Cubase funktionierte in keiner Konfiguration.

**Ursache:** Cubase kann sich nicht auf eingehende MIDI Clock synchronisieren.
Als Timecode-Quelle akzeptiert es nur Interner Timecode, MIDI-Timecode,
ASIO-Positioning und VST System Link. Der „Tempo-Handshake" konnte prinzipiell
nie wirken. Der Hilfetext im Panel („Timecode-Quelle MIDI Timecode/MIDI Clock")
war schlicht falsch.

**Geändert — Rollen umgedreht.** Die DAW hält die Zeit, der Generator folgt:

- **Clock-Slave-Modus** (Abschnitt 7c). Wertet F8 (Clock), FA (Start),
  FB (Continue), FC (Stop) und F2 (Songposition) eines wählbaren MIDI-Eingangs
  aus. Cubases Transport startet und stoppt damit den Generator, Tempoänderungen
  greifen im laufenden Takt, der BPM-Regler folgt sichtbar.
- **Zwei Zeitquellen.** Im Slave-Modus schreibt `schedTick` `tickRef`/`msRef`
  nicht mehr fort, sondern interpoliert nur für den Lookahead-Horizont; die
  Position kommt ausschließlich aus den Clock-Ticks. Die gesamte
  Nachfolgelogik bleibt unverändert — rund 80 Zeilen statt Sequencer-Umbau.
- **Rückkopplungsschutz.** Solange Slave aktiv ist, geben `clockStart`,
  `clockStop`, `pumpClock`, `cubaseStart` und `cubaseStopCmd` sofort zurück.
  Generator und DAW hängen am selben IAC-Bus; sonst sendet der Generator die
  Clock, der er folgt, selbst wieder aus.
- Tempoerkennung über das gleitende Mittel der letzten 48 Clock-Abstände,
  mit Ausreißerfilter (nur 1,5–400 ms) und Abriss-Erkennung nach 900 ms.
- MIDI-Eingänge werden jetzt mit enumeriert (`populateInputs`).

**Verifiziert** mit simuliertem Clock-Master: externer Start erkannt,
Tempowechsel 120→160 nachgezogen, externer Stop greift, null Clock-Bytes und
null SysEx gehen zurück nach draußen.

---

## BUILD 2026-07-30-E — Sync-Panel korrigiert

- Panel heißt jetzt **DAW Sync** statt Cubase Sync.
- Der falsche Setup-Hinweis ist ersetzt durch eine Warnung, dass Cubase MIDI
  Clock nicht als Sync-Quelle annimmt.
- „Tempo-Handshake" → **Clock-Burst**, gekennzeichnet als Hardware-Feature
  (Drumcomputer, Groovebox, Looper).
- Der MMC-Setup-Pfad ist auf die tatsächliche Dialogstruktur korrigiert:
  Transport → Projekt-Synchronisationseinstellungen → **Gerätesteuerung** →
  MMC-Slave aktiv. Dass dieser Haken standardmäßig **aus** ist, war der Grund,
  warum Play/Record/Stop nichts bewirkten — der Generator hatte immer korrekt
  gesendet (`F0 7F 7F 06 02 F7` usw., headless verifiziert).

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
