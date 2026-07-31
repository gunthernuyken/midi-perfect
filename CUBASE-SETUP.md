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
| CH_10 | All MIDI Inputs | **Kanal 10** | Groove Agent / EZdrummer | **Kanal 1** ← siehe Abschnitt 5 |

Alle Felder liegen im Inspector unter *Routing*.

Der **Eingangskanal** filtert, was hereinkommt. Der **Ausgangskanal** bestimmt,
auf welchem Kanal es beim Instrument ankommt — die Spur ist damit ein
Kanal-Umsetzer. Genau das ist der Hebel in den Abschnitten 5 und 6.

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

## 6 · Falle 3: Plugins mit einem MIDI-Kanal pro Klangerzeuger

Groove Agent ist kein Sonderfall, sondern die Regel. Jedes Plugin, das mehrere
Klangerzeuger in einer Instanz hält, verteilt sie auf MIDI-Kanäle — und die
Werkseinstellung passt fast nie zur Lane-Belegung des Generators.

**Beispiel Cherry Audio Blue3** (Tonewheel-Orgel, drei Manuale). Werkseinstellung
unter *Zahnrad → Global → MIDI Channel*:

```
Upper:  1
Lower:  2
Pedals: 3
```

Die CHORDS-Lane sendet auf **Kanal 2**. Legt man die Spur 1:1 durch, spielt man
also das **Lower-Manual**. Man kann an den oberen Drawbars drehen so lange man
will — es ändert sich nichts, und das Instrument klingt dabei nicht falsch,
sondern nur unerklärlich anders als erwartet. Das ist die unangenehme Variante
eines Fehlers: es kommt Ton, er ist bloß aus der falschen Quelle.

**Fix:** Ausgangskanal der Spur auf `Kanal 1` stellen, Eingangskanal auf
`Kanal 2` lassen.

```
Eingang        All MIDI Inputs
Eingangskanal  Kanal 2       ← nur die CHORDS-Lane kommt rein
Ausgang        Blue3 Organ
Ausgangskanal  Kanal 1       ← Upper-Manual
```

Das Plugin bleibt damit auf Werkseinstellung, und Lower und Pedals bleiben frei.
Der umgekehrte Weg — im Plugin `Upper` auf 2 setzen — führt zur Kollision mit
`Lower`, das ebenfalls auf 2 steht: dann spielen **beide** Manuale.

**Als Regel:** Bei jedem mehrstimmigen Plugin einmal nachsehen, welche Kanäle
seine Slots, Manuale oder Parts belegen, und die Zuordnung über den
**Ausgangskanal der Spur** herstellen — nicht durch Umkonfigurieren des Plugins.
Die Spur ist projektlokal, die Plugin-Einstellung folgt oft dem Preset.

> Zusätzlich prüfen, ob das Plugin einen **Split** aktiv hat. Bei Blue3 begrenzt
> `UPPR/LWR` das Upper-Manual auf den oberen Tastaturbereich; Comping unterhalb
> des Splitpunkts landet dann wieder im Lower.

## 7 · Falle 4: Aufnahmebereitschaft und Monitor

Alle Spuren gleichzeitig **aufnahmebereit** ist für Live-Monitoring genau
richtig — beim Aufnehmen schreibt dann aber jede armierte Spur mit.
Vor der Aufnahme nur die Spur scharf schalten, die man wirklich braucht.

Der Monitor-Schalter einer **Ordnerspur** gilt für alle enthaltenen Spuren.
Wenn plötzlich alles mithört, obwohl an den einzelnen Spuren nichts aktiv
aussieht: den Ordner prüfen.

**Für den Dauerbetrieb ist Monitor die richtige Wahl, nicht Aufnahmebereitschaft.**
Aufnahmebereitschaft lässt zwar ebenfalls MIDI durch, hängt aber am
Aufnahme-Workflow — sobald man scharf schaltet, um etwas mitzuschneiden,
verschiebt sich, welche Lane gerade klingt. Monitor auf allen fünf Lane-Spuren
lassen und Aufnahmebereitschaft nur dann setzen, wenn wirklich aufgenommen wird.

---

## 8 · Transport-Sync

Zwei unabhängige Mechanismen, beide im Panel *Cubase Sync*:

### MIDI Clock (Tempo und Position)

Der Generator sendet 24 Clocks pro Viertel plus Start/Continue/Stop.
Cubase übernimmt damit Tempo und Position:

```
Transport → Projekt-Synchronisationseinstellungen
  Timecode-Quelle: MIDI Timecode / MIDI Clock
  Eingang: derselbe Port (IAC Bus 1)
  Sync in der Transportleiste aktivieren
```

### MMC (Transportbefehle)

Play, Record und Stop per SysEx. **Setzt voraus, dass der Browser SysEx
freigegeben hat** — die Statuszeile im Generator zeigt das an
(*„MIDI-Zugriff erteilt (inkl. SysEx/MMC)"*).

```
Projekt-Synchronisation → MMC Slave aktiv
MMC-Eingang: IAC
Device-ID identisch (Default 127 = alle)
```

### Count-In

Lässt Cubase vorlaufen, bevor die erste Note kommt — sinnvoll beim
Aufnehmen, damit der Record-Start nicht die erste Zählzeit abschneidet.

---

## 9 · Checkliste bei Stille

Der Reihe nach, von der Quelle zum Ziel:

1. Zeigt die Statuszeile im Generator einen grünen Punkt und einen Port?
2. Zeigt der **MIDI-Monitor** ausgehende Nachrichten? Auf welchem Kanal?
3. Ist die **Kanalsperre** versehentlich an und blockt den Kanal?
4. Ist die Lane überhaupt eingeschaltet? (Routing-Zeile im Panel prüfen,
   ausgeschaltete Lanes stehen dort ausgegraut mit `(aus)`)
5. Zeigt der Spurmeter in Cubase Aktivität? Nein → Eingangskanal falsch.
6. Meter ja, aber stumm → Instrument hört auf einem anderen Kanal
   (Abschnitt 5 oder 6) oder das Kit hat keine Pads auf den gesendeten Noten.
7. Die gesendeten Drum-Noten sind GM-Standard:
   36 Kick · 38 Snare · 42 HiHat closed · 46 HiHat open · 49 Crash · 51 Ride.

---

## 10 · Projekt speichern

Das gesamte Spur-Routing steckt in der `.cpr`. Nach dem Einrichten
speichern — sonst ist beim nächsten Öffnen alles wieder auf *Alle Eingänge*.
