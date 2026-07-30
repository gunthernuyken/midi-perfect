# Cubase-Setup

Getestet mit **Cubase Pro 14 / macOS**, Verbindung über den **IAC-Treiber**.
Das Prinzip gilt für jede DAW, die MIDI-Kanäle auf Spuren verteilt.

---

## 1 · IAC-Treiber

Der IAC-Treiber ist ein virtueller MIDI-Port von macOS. Er ist standardmäßig
**offline**.

```
Programme → Dienstprogramme → Audio-MIDI-Setup
  Fenster → MIDI-Studio einblenden
  IAC-Treiber doppelklicken
  ☑ Gerät ist online
  Ports: mindestens "Bus 1"
```

Ohne diesen Schritt zeigt der Browser keinen Output an und loggt
*„Keine MIDI-Outputs – IAC-Treiber aktivieren!"*.

## 2 · MIDI-Anschlusseinstellungen in Cubase

```
Studio → Studio-Einstellungen → MIDI-Anschlusseinstellungen
```

`IAC-Treiber Bus 1` muss als Eingang **sichtbar** sein. Die Spalte
*In „All MIDI Inputs"* darf aktiv bleiben — sie ist bequem, aber sie ist auch
die Ursache der Falle in Abschnitt 4.

## 3 · Spuren anlegen

Eine MIDI-Spur pro Lane. Namen frei, aber die Kanalnummer im Namen zu führen
spart später Sucherei:

| Spur | Eingang | **Eingangskanal** | Ausgang | Ausgangskanal |
|---|---|---|---|---|
| CH_01 | All MIDI Inputs | **Kanal 1** | HALion Sonic | Kanal 1 |
| CH_02 | All MIDI Inputs | **Kanal 2** | HALion Sonic | Kanal 2 |
| CH_03 | All MIDI Inputs | **Kanal 3** | HALion Sonic | Kanal 3 |
| CH_04 | All MIDI Inputs | **Kanal 4** | HALion Sonic | Kanal 4 |
| CH_10 | All MIDI Inputs | **Kanal 10** | Groove Agent | **Kanal 1** ← siehe Abschnitt 5 |

Alle Felder liegen im Inspector unter *Routing*.

---

## 4 · Falle 1: Eingangskanal „Alle Eingänge"

**Symptom:** Im Generator ist nur eine Lane eingeschaltet, in Cubase zeigen
mehrere Spuren gleichzeitig MIDI-Aktivität, und man hört mehrere Instrumente.

**Ursache:** Der **Eingangskanal** einer MIDI-Spur steht per Default auf
*Alle Eingänge* (= Any). Eine solche Spur filtert nichts. Sie nimmt jede
eingehende Note — egal auf welchem Kanal — und schickt sie an ihren **eigenen**
Ausgangskanal weiter, sobald Monitor oder Aufnahmebereitschaft aktiv ist.

Bei fünf Spuren mit *Alle Eingänge* wird aus einer Drum-Note auf Kanal 10:

```
Generator  Ch10 ──┬─▶ CH_01 (Any) ──▶ HALion Sonic Ch1
                  ├─▶ CH_02 (Any) ──▶ HALion Sonic Ch2
                  ├─▶ CH_03 (Any) ──▶ HALion Sonic Ch3
                  ├─▶ CH_04 (Any) ──▶ HALion Sonic Ch4
                  └─▶ CH_10 (Any) ──▶ Groove Agent Ch10
```

Fünf Instrumente spielen das Schlagzeug. Der Generator hat dabei genau
**eine** Nachricht gesendet.

**Fix:** Pro Spur den Eingangskanal hart auf den eigenen Kanal setzen.
Inspector → *Routing* → zweites Feld → `Kanal N`.

> **Diagnose-Hinweis:** Bevor man in der DAW sucht, sollte man ausschließen,
> dass der Sender schuld ist. Dafür gibt es in MIDI PERFECT 2 die
> **Kanalsperre** und den **MIDI-Monitor**. Zeigt der Monitor nur `Ch10` und
> die DAW meldet trotzdem Aktivität auf Ch1 — dann ist es die DAW.

## 5 · Falle 2: Groove Agent hört nicht auf Kanal 10

**Symptom:** Die Drum-Spur empfängt sauber auf Kanal 10, aber Groove Agent
bleibt stumm. Kein Pad leuchtet.

**Ursache:** Die GM-Konvention „Schlagzeug liegt auf Kanal 10" gilt für
GM-Klangerzeuger. **Groove Agent verteilt seine vier Kit-Slots auf MIDI-Kanal
1 bis 4.** Kanal 10 empfängt kein Slot — die Noten laufen ins Leere.

**Fix:** Die Drum-Spur als Umsetzer benutzen. Eingang filtert Kanal 10,
Ausgang sendet auf Kanal 1:

```
Eingang        All MIDI Inputs
Eingangskanal  Kanal 10      ← nur die Drums aus dem Generator kommen rein
Ausgang        Groove Agent
Ausgangskanal  Kanal 1       ← Slot 1
```

Im Generator bleibt DRUMS auf Kanal 10. Nichts ändern.

Alternativ: DRUMS im Generator auf einen freien Kanal legen und die Spur
1:1 durchreichen. Der Umsetzer-Weg ist vorzuziehen, weil er die GM-Konvention
im Generator und im Export intakt lässt.

## 6 · Falle 3: Aufnahmebereitschaft

Alle Spuren gleichzeitig **aufnahmebereit** ist für Live-Monitoring genau
richtig — beim Aufnehmen schreibt dann aber jede armierte Spur mit.
Vor der Aufnahme nur die Spur scharf schalten, die man wirklich braucht.

Der Monitor-Schalter einer **Ordnerspur** gilt für alle enthaltenen Spuren.
Wenn plötzlich alles mithört, obwohl an den einzelnen Spuren nichts aktiv
aussieht: den Ordner prüfen.

---

## 7 · Transport-Sync

Hier lauert der größte Irrtum, deshalb zuerst die Tatsache:

> **Cubase kann sich nicht auf eingehende MIDI Clock synchronisieren.**
> Unter *Projekt-Synchronisationseinstellungen → Quellen* stehen als Timecode-Quelle
> ausschließlich **Interner Timecode**, **MIDI-Timecode**, **ASIO-Audio-Gerät** und
> **VST System Link**. MIDI Clock taucht dort nicht auf und hat es seit Cubase SX
> nie getan — mit der Umstellung auf die lineare Zeit-Engine hat Steinberg die
> Clock-Slave-Fähigkeit entfernt. Cubase ist MIDI-Clock-*Master*, nie Slave.

Daraus folgen zwei brauchbare Wege und ein Irrweg.

### 7a · Clock-Slave — Cubase gibt den Takt vor (empfohlen)

Die Richtung, die funktioniert. Cubase sendet Clock, der Generator folgt.

```
Cubase  ──MIDI Clock (F8/FA/FB/FC/F2)──▶  IAC-Treiber Bus 1  ──▶  MIDI PERFECT
MIDI PERFECT  ──Noten──▶  IAC-Treiber Bus 1  ──▶  Cubase
```

Ein IAC-Bus trägt beide Richtungen gleichzeitig. Ein zweiter Bus ist sauberer,
aber nicht erforderlich.

**In Cubase:**

```
Transport → Projekt-Synchronisationseinstellungen → Ziele
  MIDI-Clock-Ziele            ☑ IAC-Treiber Bus 1
  MIDI-Clock-Voreinstellungen ☑ MIDI-Clock folgt Projektposition
                              ☑ Immer Start-Befehl senden
                              ☑ MIDI-Clock-Befehle im Stop-Modus senden
```

Das letzte Häkchen ist bequem: Cubase sendet Clock auch im Stop-Zustand, der
Generator kennt das Tempo also schon vor dem ersten Play.

**Im Generator**, Panel *DAW Sync*: **SLAVE** einschalten, Clock-Eingang auf
denselben IAC-Bus stellen. Die Statuszeile muss dann lauten:

```
SLAVE ← IAC-Treiber Bus 1 · wartet auf Start · 111.0 BPM erkannt · 240 Clocks
```

Ab jetzt starten und stoppen Cubases Transport den Generator, Tempoänderungen
greifen im laufenden Takt, und der BPM-Regler folgt sichtbar mit.

Solange SLAVE aktiv ist, unterdrückt der Generator seinen eigenen Clock-Ausgang
und alle MMC-Sendungen — sonst entstünde auf demselben Bus eine Rückkopplung.

### 7b · MMC — Generator steuert Cubases Transport

Funktioniert, muss aber in Cubase erst aktiviert werden. Standardmäßig ist es aus.

```
Transport → Projekt-Synchronisationseinstellungen → Gerätesteuerung
  Machine-Control-Eingang
    ☑ MMC-Slave aktiv
       MIDI-Eingang        IAC-Treiber Bus 1
       MMC-Gerätekennung   127        ← muss zum Feld "MMC Device" im Generator passen
```

Der Generator sendet dann:

| Aktion | SysEx |
|---|---|
| Cubase Play | `F0 7F 7F 06 02 F7` |
| Cubase Record | `F0 7F 7F 06 06 F7` → `F0 7F 7F 06 03 F7` (Record Strobe + Deferred Play) |
| Cubase Stop | `F0 7F 7F 06 01 F7` |

MMC braucht die **SysEx-Freigabe des Browsers**. Die Statuszeile im Generator
muss *„MIDI-Zugriff erteilt (inkl. SysEx/MMC)"* zeigen. Steht dort *„ohne SysEx"*,
Seite neu laden und bei der Abfrage SysEx mit erlauben.

**Count-In** lässt Cubase vorlaufen, bevor die erste Note kommt — beim Aufnehmen
schneidet der Record-Start sonst die erste Zählzeit an.

### 7c · Der Irrweg: Clock-Ausgang Richtung Cubase

Der Schalter **CLOCK** und der **Clock-Burst** senden MIDI Clock plus Start/Stop
nach draußen. Für Hardware — Drumcomputer, Groovebox, Looper — ist das genau
richtig. Cubase ignoriert es vollständig. Wer hier sucht, sucht in die falsche
Richtung; siehe 7a.

## 8 · Checkliste bei Stille

Der Reihe nach, von der Quelle zum Ziel:

1. Zeigt die Statuszeile im Generator einen grünen Punkt und einen Port?
2. Zeigt der **MIDI-Monitor** ausgehende Nachrichten? Auf welchem Kanal?
3. Ist die **Kanalsperre** versehentlich an und blockt den Kanal?
4. Ist die Lane überhaupt eingeschaltet? (Routing-Zeile im Panel prüfen,
   ausgeschaltete Lanes stehen dort ausgegraut mit `(aus)`)
5. Zeigt der Spurmeter in Cubase Aktivität? Nein → Eingangskanal falsch.
6. Meter ja, aber stumm → Instrument hört auf einem anderen Kanal
   (Abschnitt 5) oder das Kit hat keine Pads auf den gesendeten Noten.
7. Die gesendeten Drum-Noten sind GM-Standard:
   36 Kick · 38 Snare · 42 HiHat closed · 46 HiHat open · 49 Crash · 51 Ride.

Im Clock-Slave-Modus zusätzlich:

8. Zeigt die Slave-Statuszeile Clocks und ein BPM? Nein → Cubase sendet keine
   Clock (Ziele-Tab prüfen) oder der falsche Eingang ist gewählt.
9. Steht dort *Clock abgerissen*, hat Cubase mittendrin aufgehört zu senden —
   meist weil *MIDI-Clock-Befehle im Stop-Modus senden* aus ist. Das ist
   unkritisch, der Generator läuft beim nächsten Start wieder mit.

---

## 9 · Projekt speichern

Spur-Routing, MMC-Slave und die MIDI-Clock-Ziele stecken **alle in der `.cpr`**.
Nach dem Einrichten speichern — sonst ist beim nächsten Öffnen alles wieder auf
*Alle Eingänge*, MMC aus und kein Clock-Ziel gesetzt.
