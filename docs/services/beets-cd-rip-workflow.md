# Beets / Whipper CD-Rip-Workflow

> Stand: 22. Juli 2026

## Zweck

Diese Dokumentation beschreibt den produktiven CD-Rip-Workflow für den Musikimport über **Whipper**, **Beets**, **Docker** und den **HP EliteDesk** als zentrale Import-Instanz.

Ziele:

- möglichst genaue und reproduzierbare CD-Rips
- klare Trennung zwischen Rippen, Prüfen, Freigeben und Importieren
- keine halbfertigen Rips im automatischen Beets-Eingang
- problematische CDs kontrolliert behandeln, statt Prozesse stundenlang hängen zu lassen
- unvollständige, aber bewusst akzeptierte Rips ohne Prüfschleife importieren
- eindeutige Ablage für erfolgreiche Importe, manuelle Prüfung und technische Fehler

Die Rollen sind bewusst getrennt:

- **MACiMessernix / Rip-Station** = CD lokal rippen, Log prüfen und fertigen Rip manuell freigeben
- **HP EliteDesk** = Docker-Host, Beets-Datenbank, Importlogik, Polling und Cleanup
- **NAS** = zentraler Eingang, Review-Bereich und produktive Musikbibliothek

---

## Architektur

### MACiMessernix / Rip-Station

- rippt Audio-CDs primär mit `whipper`
- schreibt während des Rippens ausschließlich auf die lokale Platte:

  ```text
  $HOME/rip-local
  ```

- prüft nach dem Rip das Whipper-Log, die erzeugten Dateien und AccurateRip
- verschiebt erst danach den fertigen Albumordner manuell nach:

  ```text
  /mnt/nas/musicincome
  ```

Das lokale Rip-Ziel verhindert, dass ein stockender NAS-Mount den eigentlichen Lesevorgang blockiert oder wie ein CD-/Laufwerksfehler aussieht.

Eine zusätzliche automatische Veröffentlichung lokaler Rips wurde bewusst **nicht** eingerichtet. Nach der ohnehin notwendigen manuellen Logprüfung ist das direkte Verschieben nach `musicincome` einfacher und transparenter.

### HP EliteDesk / Docker-Host

- sieht dieselbe NAS-Freigabe unter:

  ```text
  /mnt/nas/media
  ```

- führt den Import über den Container `music-beets` aus
- überwacht zwei getrennte Importquellen:
  - normale Importpakete aus `musicincome`
  - manuell genehmigte unvollständige Rips aus `musicreview-approvedincomplete`
- betreibt Polling und Cleanup per `systemd`

### NAS

Zentrale Ablage für:

- `musicincome` – Eingang für geprüfte, reguläre Importpakete
- `Musik` – produktive Musikbibliothek
- `music-review/_IMPORTED_DONE` – Restdateien erfolgreicher Importe
- `music-review/_MANUAL_REVIEW` – reguläre Importe mit nicht übernommenen Audiodateien
- `music-review/_FAILED_IMPORTS` – technische Fehler regulärer Importe
- `music-review/musicreview-approvedincomplete` – manuell akzeptierte, unvollständige Rips
- `music-review/_APPROVED_IMPORT_FAILED` – technische Fehler beim Import genehmigter Rips
- `musictemp` – derzeit **kein Rip-Übergabeordner mehr**, sondern noch Speicherort der aktiven Beets-Datenbank

---

## Wichtige Pfade

### MACiMessernix

| Zweck | Pfad |
|---|---|
| Lokaler Rip-Arbeitsbereich | `$HOME/rip-local` |
| NAS-Mount | `/mnt/nas` |
| Regulärer Beets-Eingang | `/mnt/nas/musicincome` |

### HP EliteDesk

| Zweck | Pfad |
|---|---|
| NAS-Mount | `/mnt/nas/media` |
| Regulärer Beets-Eingang | `/mnt/nas/media/musicincome` |
| Produktive Bibliothek | `/mnt/nas/media/Musik` |
| Aktueller Datenbank-Speicherort | `/mnt/nas/media/musictemp/beets.db` |
| Erfolgreich verarbeitete Restordner | `/mnt/nas/media/music-review/_IMPORTED_DONE` |
| Manuelle Nachbearbeitung | `/mnt/nas/media/music-review/_MANUAL_REVIEW` |
| Technische Fehler regulärer Importe | `/mnt/nas/media/music-review/_FAILED_IMPORTS` |
| Genehmigte unvollständige Rips | `/mnt/nas/media/music-review/musicreview-approvedincomplete` |
| Fehler genehmigter Importe | `/mnt/nas/media/music-review/_APPROVED_IMPORT_FAILED` |

### Beets-Container

| Zweck | Pfad |
|---|---|
| Regulärer Eingang | `/music/incoming` |
| Eingang für genehmigte Rips | `/music/approvedincoming` |
| Bibliothek | `/music/library` |
| Aktuelle Beets-Datenbank | `/music/temp/beets.db` |
| Hauptkonfiguration | `/config/config.yaml` |
| Sonderprofil für genehmigte Rips | `/config/approved-import.yaml` |

---

## Mount-Unterschiede zwischen Rip-Station und HP

Die NAS-Freigabe wird auf beiden Systemen unterschiedlich eingehängt:

```text
MACiMessernix: /mnt/nas
HP EliteDesk:  /mnt/nas/media
```

Daher gilt beispielsweise:

```text
MACiMessernix: /mnt/nas/musicincome
HP EliteDesk:  /mnt/nas/media/musicincome
```

Auf MACiMessernix nicht versehentlich `/mnt/nas/media/...` verwenden. Die Freigabe `media` ist dort bereits direkt unter `/mnt/nas` eingehängt.

Für manuelles Verschieben im Dateimanager wird der fest eingehängte CIFS-Pfad `/mnt/nas/musicincome` verwendet. Die zusätzlich sichtbare GVFS-Verbindung über `smb://...` wird für den dokumentierten Workflow nicht benötigt.

---

## Aktuell geprüfte Umgebung

### Rip-Werkzeuge

```text
whipper 0.10.0
cdparanoia III 10.2 / libcdio 2.1.0
cdrdao 1.2.4
flac 1.4.3
libcdio-paranoia2t64 installiert
```

In `~/.config/whipper/whipper.conf` ist das automatische Schließen der Laufwerkslade deaktiviert:

```ini
[whipper]
drive_auto_close = False
```

Der für das Verbatim-Laufwerk konfigurierte Read-Offset `6` wird beibehalten. Zahlreiche AccurateRip-Treffer bestätigen, dass die grundsätzliche Offset-Konfiguration funktioniert.

### Beets

```text
beets 2.8.0
Python 3.12.12
Plugins: duplicates, embedart, fromfilename, lyrics, musicbrainz, web
```

---

## Produktiver Standard-Workflow

### 1. Lokalen Arbeitsbereich anlegen

```bash
mkdir -p "$HOME/rip-local"
```

### 2. CD lokal rippen

```bash
whipper cd rip -p -k -r 2 \
  -O "$HOME/rip-local" \
  --track-template "%A/%d/%t - %n" \
  --disc-template "%A/%d/%A - %d"
```

Bedeutung der wichtigsten Optionen:

- `-p`: bei mehreren MusicBrainz-Treffern nachfragen
- `-k`: nach einem nicht lesbaren Track mit den folgenden Tracks fortfahren
- `-r 2`: höchstens zwei vollständige Rip-Versuche pro Track
- `-O`: Ausgabe zunächst auf die lokale Platte

`-r 0` darf nicht verwendet werden, da dies unbegrenzt viele Versuche bedeutet.

### 3. Rip-Ergebnis prüfen

Ein Rip wird nicht allein deshalb als erfolgreich betrachtet, weil der Prozess beendet wurde.

#### Sicher akzeptieren

- AccurateRip meldet `rip accurate` für alle Tracks.
- Oder: Für eine nicht in AccurateRip bekannte CD wurden mindestens zwei unabhängige Durchläufe mit demselben Laufwerk und identischen Prüfsummen erzeugt.

#### Manuell prüfen

- Track-CRCs stimmen intern überein, aber AccurateRip kennt die Ausgabe nicht.
- MusicBrainz-Zuordnung ist unsicher.
- Eine CD enthält Besonderheiten wie HTOA, Daten-Track oder ungewöhnliche Pregaps.

#### Nicht als vollständigen Archiv-Rip akzeptieren

- `checksums do not match`
- `track ... failed to rip`
- `cdparanoia couldn't read any frames`
- `file size 0 did not match expected size`
- `transport error`
- fehlende FLAC-Dateien
- AccurateRip meldet `rip NOT accurate`

Eine angezeigte `Rip quality` unter 100 % ist nicht automatisch ein Fehler. Wenn AccurateRip den Track bestätigt, ist der Rip trotz zusätzlicher Korrekturarbeit verifiziert.

### 4. Geprüften Albumordner manuell freigeben

Nach der Logprüfung wird der komplette Albumordner im Dateimanager ausgeschnitten und nach folgendem Pfad verschoben:

```text
/mnt/nas/musicincome
```

Alternativ im Terminal:

```bash
mv -- \
  "$HOME/rip-local/<ALBUMORDNER>" \
  "/mnt/nas/musicincome/"
```

`musicincome` ist ausschließlich für fertige Importpakete vorgesehen. Dort darf nicht mehr gerippt, kopiert oder nachträglich ergänzt werden.

Die lokale Kopie wird beim Verschieben entfernt. Wer zusätzliche Sicherheit möchte, kopiert zunächst mit `rsync`, prüft die Übertragung und löscht die lokale Quelle erst nach erfolgreichem Beets-Import.

### 5. Ruhezeit und automatischer Import

Das Polling-Skript wartet, bis innerhalb des Albumordners mindestens 15 Minuten lang keine Datei mehr verändert wurde. Anschließend startet es den nicht-interaktiven Beets-Import.

### 6. Ergebnis prüfen

Beets-Datenbank:

```bash
docker exec music-beets beet ls -p '<SUCHBEGRIFF>'
```

Dateisystem:

```bash
find /mnt/nas/media/Musik \
  -type f \
  -ipath '*<SUCHBEGRIFF>*' \
  -print
```

Bei einer Artist-Suche kann zusätzlich `albumartist` relevant sein. Eine Suche nur über den Dateinamen mit `-iname` findet den Künstler nicht zwingend, wenn sein Name nur im Verzeichnispfad steht.

---

## Qualitätsampel

| Status | Bedeutung | Aktion |
|---|---|---|
| Grün | Alle Tracks AccurateRip-bestätigt | Direkt nach `musicincome` freigeben |
| Gelb | Reproduzierbare CRCs, aber kein Datenbanktreffer | Log aufheben, manuell entscheiden, gegebenenfalls zweites Laufwerk testen |
| Rot | Unterschiedliche CRCs, fehlende Tracks, Null-Byte-Dateien oder Transportfehler | Nicht als vollständigen Archiv-Rip importieren; Problem-CD-Workflow starten |

Ein bewusst akzeptierter unvollständiger Rip wird nicht nach `musicincome`, sondern nach `musicreview-approvedincomplete` gelegt.

---

## Docker- und Beets-Konfiguration

### Relevante Volume-Mounts

```yaml
volumes:
  - /opt/docker/beets/config:/config
  - /mnt/nas/media/musicincome:/music/incoming:rw
  - /mnt/nas/media/Musik:/music/library
  - /mnt/nas/media/musictemp:/music/temp
  - /mnt/nas/media/music-review:/music/review
  - /mnt/nas/media/music-review/musicreview-approvedincomplete:/music/approvedincoming
```

Der zusätzliche Mount `/music/approvedincoming` ist notwendig, damit genehmigte unvollständige Rips direkt importiert werden können, ohne sie erneut durch `musicincome` zu schicken.

### Hauptkonfiguration

Auszug aus `/opt/docker/beets/config/config.yaml`:

```yaml
directory: /music/library
library: /music/temp/beets.db

paths:
  default: $albumartist/$album/$track - $title
  singleton: Non-Album/$artist - $title
  comp: Compilations/$album/$track - $title

import:
  move: yes
  copy: no
  autotag: yes
  duplicate_action: ask
  quiet: no
  timelimit: 300
```

Eine Option `destination:` wird nicht verwendet. Das Ziel der Bibliothek wird durch `directory: /music/library` festgelegt.

### Sonderprofil für genehmigte unvollständige Rips

Datei:

```text
/opt/docker/beets/config/approved-import.yaml
```

Inhalt:

```yaml
threaded: no

import:
  move: yes
  copy: no
  autotag: no
  duplicate_action: merge
  quiet: yes
  resume: no
```

Bedeutung:

- `autotag: no`: vorhandene Tags werden verwendet; die manuelle Entscheidung wird nicht erneut durch eine MusicBrainz-Vollständigkeitsprüfung aufgehoben
- `duplicate_action: merge`: vorhandene Tracks werden mit einem bereits vorhandenen Album zusammengeführt
- `threaded: no`: verhindert bei Beets 2.8.0 Probleme beim parallelen Zusammenführen
- `move: yes`: erfolgreich importierte Audiodateien werden nach `/music/library` verschoben

Konfiguration prüfen:

```bash
docker exec music-beets \
  beet --config /config/approved-import.yaml config \
  | grep -E 'threaded:|move:|autotag:|duplicate_action:'
```

Erwartet:

```text
threaded: no
move: yes
autotag: no
duplicate_action: merge
```

---

## Polling-Automatisierung

### Komponenten

| Komponente | Pfad |
|---|---|
| Polling-Skript | `/opt/docker/beets/poll-import.sh` |
| Service | `/etc/systemd/system/beets-poll.service` |
| Timer | `/etc/systemd/system/beets-poll.timer` |
| Log | `/opt/docker/beets/beets-poll.log` |

### Produktive Werte

- Polling-Intervall: **5 Minuten**
- Ruhezeit vor Import: **15 Minuten ohne Änderungen**
- Parallelität: durch `flock` auf einen aktiven Lauf begrenzt

Timer:

```ini
[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Unit=beets-poll.service
Persistent=true
```

Ein Import kann länger als fünf Minuten dauern. In diesem Fall kann der nächste fällige Lauf unmittelbar nach Abschluss starten. Durch `flock` laufen trotzdem keine zwei Importe gleichzeitig.

### Quelle 1: normaler Eingang

Host:

```text
/mnt/nas/media/musicincome
```

Container:

```text
/music/incoming
```

Importaufruf:

```bash
beet import -q -P --quiet-fallback skip <PFAD>
```

Ergebnislogik:

- Exit-Code ungleich 0 → `_FAILED_IMPORTS`
- Exit-Code 0 und keine Audiodateien mehr → Restordner nach `_IMPORTED_DONE`
- Exit-Code 0, aber Audiodateien bleiben zurück → `_MANUAL_REVIEW`

### Quelle 2: genehmigte unvollständige Rips

Host:

```text
/mnt/nas/media/music-review/musicreview-approvedincomplete
```

Container:

```text
/music/approvedincoming
```

Importaufruf:

```bash
beet --config /config/approved-import.yaml \
  import -q -P -A -m <PFAD>
```

Bedeutung:

- `-A`: Import ohne erneute Autotag-Zuordnung; vorhandene Tags werden verwendet
- `-m`: Audiodateien in die Bibliothek verschieben
- `merge`: fehlende beziehungsweise ergänzende Tracks mit einem vorhandenen Album zusammenführen

Ergebnislogik:

- Exit-Code ungleich 0 → `_APPROVED_IMPORT_FAILED`
- Exit-Code 0 und keine Audiodateien mehr → Restordner nach `_IMPORTED_DONE`
- Exit-Code 0, aber Audiodateien bleiben zurück → `_APPROVED_IMPORT_FAILED`

Wichtige Sicherheitsregel:

> Ein Ordner mit verbliebenen Audiodateien darf niemals nach `_IMPORTED_DONE` verschoben werden.

Der Exit-Code allein gilt nicht als Erfolgsnachweis. Das Skript kontrolliert nach dem Import zusätzlich, ob im Quellordner noch Audiodateien vorhanden sind.

### Lyrics-Fehler

Meldungen wie:

```text
lyrics: LRCLib: Request error: 400 Client Error
```

betreffen nur den Abruf von Liedtexten. Sie verhindern den Musikimport nicht, solange Beets den eigentlichen Import erfolgreich beendet.

---

## Manuelle Nachbearbeitung und Freigabe unvollständiger Rips

### Regulärer Problemfall

Nicht vollständig übernommene reguläre Importpakete landen unter:

```text
/mnt/nas/media/music-review/_MANUAL_REVIEW
```

Typische Gründe:

- Beets überspringt einzelne Tracks im Quiet-Modus.
- MusicBrainz-Zuordnung ist nicht sicher genug.
- Metadaten sind unklar.
- Ein oder mehrere Tracks fehlen.
- Ein Rip ist nur als unverifizierte Hörkopie vorhanden.

### Manuelle Entscheidung

Nach der Prüfung gibt es zwei Wege:

- erneut rippen → nach `musicreview-rerip` beziehungsweise zurück in den Rip-Workflow
- vorhandene unvollständige Fassung bewusst behalten → nach `musicreview-approvedincomplete`

Beispiel auf dem HP:

```bash
mv -- \
  "/mnt/nas/media/music-review/_MANUAL_REVIEW/<ALBUMORDNER>" \
  "/mnt/nas/media/music-review/musicreview-approvedincomplete/"
```

Der nächste Polling-Lauf verwendet automatisch das Approved-Sonderprofil. Der Ordner wird **nicht** wieder durch die normale Vollständigkeitslogik von `musicincome` geschickt. Dadurch entsteht kein Teufelskreis.

### Technischer Fehler beim Approved-Import

Ordner landen unter:

```text
/mnt/nas/media/music-review/_APPROVED_IMPORT_FAILED
```

Dort wird geprüft:

- Beets-Log und Exit-Code
- verbleibende Audiodateien
- Tags und Dateiformate
- vorhandene Duplikate beziehungsweise Albumzusammenführung
- Schreibrechte auf `/music/library`

---

## Cleanup-Automatisierung

### Komponenten

| Komponente | Pfad |
|---|---|
| Cleanup-Skript | `/opt/docker/beets/cleanup-imported-done.sh` |
| Service | `/etc/systemd/system/beets-cleanup.service` |
| Timer | `/etc/systemd/system/beets-cleanup.timer` |

### Regel

Alles in `_IMPORTED_DONE`, das älter als 30 Tage ist, wird gelöscht.

Dort dürfen deshalb ausschließlich Restdateien erfolgreicher Importe liegen. Wichtige Whipper-, AccurateRip- oder Diagnose-Logs müssen vor Ablauf der Frist an anderer Stelle archiviert werden, wenn sie langfristig erhalten bleiben sollen.

---

## Problem-CD-Workflow

Bibliotheks-CDs können durch Kratzer, Fingerabdrücke, Randbeschädigungen oder Schäden an der Reflexionsschicht schwer lesbar sein. Software kann fehlende oder bei jedem Durchlauf unterschiedliche Audiodaten nicht zuverlässig erraten.

### 1. CD prüfen und reinigen

- CD unter gutem Licht prüfen
- insbesondere den äußeren Bereich kontrollieren
- Staub und Fingerabdrücke entfernen
- mit einem weichen Tuch radial von innen nach außen wischen
- keine kreisförmigen Bewegungen entlang der Spur
- bei geliehenen CDs keine Schleif- oder Poliergeräte verwenden

### 2. Problemtrack direkt mit cd-paranoia testen

Beispiel für Track 17:

```bash
mkdir -p "$HOME/cdtest"

timeout --signal=INT --kill-after=20s 30m \
  cd-paranoia \
    --stderr-progress \
    --force-cdrom-device /dev/sr0 \
    17 "$HOME/cdtest/track17-run1.wav"
```

Zweiter Durchlauf mit demselben Laufwerk:

```bash
timeout --signal=INT --kill-after=20s 30m \
  cd-paranoia \
    --stderr-progress \
    --force-cdrom-device /dev/sr0 \
    17 "$HOME/cdtest/track17-run2.wav"
```

Vergleich:

```bash
sha256sum "$HOME"/cdtest/track17-run*.wav
cmp "$HOME/cdtest/track17-run1.wav" \
    "$HOME/cdtest/track17-run2.wav"
```

Zwei Durchläufe unterschiedlicher Laufwerke dürfen ohne Offset-Korrektur nicht direkt mit `cmp` verglichen werden. Für einen reproduzierbaren Test werden immer zwei Durchläufe desselben Laufwerks miteinander verglichen.

### 3. Niedrigere Lesegeschwindigkeit versuchen

```bash
timeout --signal=INT --kill-after=20s 30m \
  cd-paranoia \
    --stderr-progress \
    --force-cdrom-device /dev/sr0 \
    --force-read-speed 4 \
    17 "$HOME/cdtest/track17-speed4.wav"
```

Eine geringere Geschwindigkeit kann helfen, ist aber keine Garantie. Wenn das Laufwerk dieselben Bereiche ständig erneut liest, bringt ein stundenlanger Lauf meist keinen verifizierbaren Vorteil.

### 4. Zweites Laufwerk verwenden

Problematische CDs werden mit einem Laufwerk anderer Bauart und eines anderen Herstellers getestet.

Bevorzugt gesucht wird:

- vollformatiges internes **5,25-Zoll-SATA-Laufwerk**
- Schublade statt Slot-in
- andere Laufwerksfamilie als das vorhandene Slim-Blu-ray-Laufwerk
- bei externem Betrieb ein Gehäuse mit eigenem Netzteil

Interessante gebrauchte Modellreihen:

1. `HL-DT-ST / LG BH14NS48`
2. `ASUS BC-12B1ST`
3. `Lite-On iHAS224 B` – auf Hardware-Revision `B` achten
4. `TSSTcorp SH-216DB`

Diese Modelle sind Beispiele, keine Garantie für jede beschädigte CD. Unterschiedliche Laufwerksmechaniken sind wichtiger als der Kauf eines möglichst teuren Einzelgeräts.

### 5. Entscheidung nach dem Vergleich

- Zweites Laufwerk liest den Track reproduzierbar und AccurateRip bestätigt ihn: Rip übernehmen.
- Beide Laufwerke hängen oder erzeugen unterschiedliche Prüfsummen: CD als nicht zuverlässig lesbar einstufen.
- Nur eine Hörkopie möglich: getrennt vom verifizierten Archiv behandeln und klar kennzeichnen.

---

## Bedeutung beobachteter cd-paranoia-Meldungen

| Meldung | Einordnung |
|---|---|
| `[read]` | Laufwerk liest einen neuen Bereich |
| `[verify]` | neu gelesene Daten werden mit vorhandenen Daten verglichen |
| `[correction]` | widersprüchliche Daten werden durch erneutes Lesen zu korrigieren versucht |
| `[overlap]` | cd-paranoia sucht die korrekte Überlappung zwischen Leseblöcken |
| `[transport error]` | SCSI-/ATAPI-Leseanforderung wurde nicht regulär erfüllt |
| Prozessstatus `D` / `blk_execute_rq` | Prozess wartet im Kernel auf eine Geräteanforderung |

Viele Korrektur-, Overlap- und Verify-Meldungen rund um dieselben Positionen bedeuten, dass cd-paranoia nicht vollständig eingefroren ist, sondern wiederholt versucht, einen instabil lesbaren Bereich zu rekonstruieren.

---

## Diagnose bei einem Hänger

### Prozesse prüfen

```bash
ps -eo pid,ppid,stat,pcpu,etime,wchan:32,cmd \
  | grep -E '[w]hipper|[c]d-paranoia'
```

### Laufenden cd-paranoia-Prozess beobachten

```bash
pid=$(pgrep -n -x cd-paranoia)

sudo strace -tt -T -p "$pid" \
  -e trace=ioctl,read,write,poll,ppoll,select,pselect6
```

### Kernelmeldungen beobachten

```bash
sudo journalctl -kf
```

Nach einem Fehler:

```bash
journalctl -k -b \
  | grep -Ei 'sr0|cdrom|medium error|I/O error|sense|reset|usb|uas|ata'
```

### USB-Treiber prüfen

```bash
lsusb -t
```

Das vorhandene Verbatim-Laufwerk wurde beim Test über `usb-storage` und nicht über UAS betrieben. Eine UAS-Blacklist ist daher für dieses Gerät nicht der passende Ansatz.

AppArmor-Meldungen des Profils `snap.fing-agent.fingagent` betreffen den Fing Agent und nicht den CD-Rip-Prozess.

---

## Optionaler cd-paranoia-Timeout

Whipper 0.10.0 besitzt keinen eigenen Timeout für einen einzelnen Lesevorgang. Ein Wrapper kann verhindern, dass ein Track unbegrenzt lange gelesen wird.

Der Wrapper ist derzeit **noch nicht produktiv eingerichtet** und bleibt ein späterer optionaler Verbesserungspunkt.

Beispiel:

```bash
mkdir -p "$HOME/.local/libexec/whipper"

cat > "$HOME/.local/libexec/whipper/cd-paranoia" <<'WRAPPER'
#!/usr/bin/env bash

exec /usr/bin/timeout \
  --signal=INT \
  --kill-after=20s \
  30m \
  /usr/bin/cd-paranoia "$@"
WRAPPER

chmod +x "$HOME/.local/libexec/whipper/cd-paranoia"
```

Whipper mit Wrapper:

```bash
PATH="$HOME/.local/libexec/whipper:$PATH" \
  whipper cd rip -p -k -r 2 \
    -O "$HOME/rip-local" \
    --track-template "%A/%d/%t - %n" \
    --disc-template "%A/%d/%A - %d"
```

Der Wrapper verbessert nicht die Lesbarkeit. Er setzt nur eine Obergrenze pro cd-paranoia-Aufruf. Ein Prozess im Kernelstatus `D` kann erst beendet werden, wenn die aktuell laufende Geräteanforderung zum Kernel zurückkehrt.

---

## 🍎 Workflow für kopiergeschützte oder fehlerhafte CDs auf dem Mac

Manche älteren Audio-CDs besitzen absichtlich fehlerhafte oder nicht standardkonforme TOCs beziehungsweise zusätzliche Datensessions. Andere CDs sind schlicht stark zerkratzt. macOS, Apple Music und XLD sprechen Laufwerke teilweise anders an als Whipper unter Linux und können deshalb als zusätzlicher Rettungsweg nützlich sein.

Dieser Weg „entfernt“ keinen Kopierschutz und garantiert keinen korrekten Rip. Entscheidend bleibt, ob das Ergebnis reproduzierbar ist oder durch AccurateRip bestätigt wird.

### Option A: Apple Music

Apple Music eignet sich als unkomplizierter Fallback, wenn Whipper oder XLD die Disc nicht sauber verarbeiten. Nach Möglichkeit wird **Apple Lossless Encoder (ALAC)** verwendet:

```text
Musik → Einstellungen → Dateien → Importeinstellungen
Importieren mit: Apple Lossless Encoder
```

Ein ALAC-Rip wird als `.m4a` gespeichert, ist aber verlustfrei. AAC sollte nur verwendet werden, wenn ALAC nicht funktioniert oder lediglich eine Hörkopie benötigt wird.

Ein Apple-Music-Import besitzt normalerweise kein Whipper-/XLD-Log und keine eigene sichere Mehrfachlesung. Ohne zusätzliche Verifikation wird er als **unverifiziert** behandelt.

### Option B: XLD

Zuerst mit sicheren Einstellungen versuchen:

- Ausgabeformat: `FLAC`, `ALAC` oder `WAV`
- Rip-Modus: `XLD Secure Ripper`
- AccurateRip: aktiviert
- Logdatei: speichern
- Laufwerksoffset für das konkrete Mac-Laufwerk konfigurieren

Wenn XLD im Secure-Modus dauerhaft hängt, kann als letzter Rettungsversuch getestet werden:

- Rip-Modus: `Burst`
- Paranoia Mode: `None`

Burst/None reduziert die Fehlerkorrektur. Ohne AccurateRip-Bestätigung oder reproduzierbare unabhängige Durchläufe bleibt das Ergebnis eine gekennzeichnete Hörkopie.

### M4A richtig behandeln

`.m4a` ist nur der Container. Darin kann ALAC oder AAC stecken.

Codec prüfen:

```bash
ffprobe -v error \
  -select_streams a:0 \
  -show_entries stream=codec_name \
  -of default=noprint_wrappers=1:nokey=1 \
  "input.m4a"
```

ALAC → FLAC bleibt verlustfrei:

```bash
ffmpeg -i "input.m4a" \
  -map 0:a:0 \
  -map_metadata 0 \
  -c:a flac \
  "output.flac"
```

AAC → FLAC stellt verlorene Informationen nicht wieder her. AAC-M4A möglichst unverändert behalten und als verlustbehaftete Quelle kennzeichnen.

### Freigabe

Nach Prüfung des Mac-Rips wird der Albumordner ebenfalls manuell nach `musicincome` verschoben. Unvollständige, aber bewusst akzeptierte Rips gehen stattdessen nach `musicreview-approvedincomplete`.

---

## Multi-Disc-Sets

- Alle Discs zunächst vollständig lokal rippen.
- Jede Disc einzeln qualitativ prüfen.
- Erst nach Abschluss aller Discs das vollständige Set gemeinsam nach `musicincome` verschieben.

`musicincome` ist kein Arbeitsbereich für halbfertige Sets.

---

## Betriebsbefehle

### Timer prüfen

```bash
systemctl list-timers --all \
  | grep -E 'beets-poll|beets-cleanup'
```

### Polling-Status

```bash
systemctl status beets-poll.timer
systemctl status beets-poll.service
```

### Polling-Log

```bash
tail -n 100 /opt/docker/beets/beets-poll.log
journalctl -u beets-poll.service -n 50 --no-pager
```

### Cleanup-Log

```bash
journalctl -u beets-cleanup.service -n 50 --no-pager
```

### Beets-Version

```bash
docker exec music-beets beet --version
```

### Aktive Datenbank und Bibliothek

```bash
docker exec music-beets \
  beet config \
  | grep -E '^library:|^directory:'
```

### Datenbankdatei prüfen

```bash
docker exec music-beets \
  sh -c 'ls -lh /music/temp/beets.db* 2>/dev/null'
```

### Album in Beets suchen

```bash
docker exec music-beets beet ls -p '<SUCHBEGRIFF>'
```

### Album im Dateisystem suchen

```bash
find /mnt/nas/media/Musik \
  -type f \
  -ipath '*<SUCHBEGRIFF>*' \
  -print
```

### Rip-Werkzeuge prüfen

```bash
whipper -v
cd-paranoia --version
cdrdao
flac --version
dpkg -l 'libcdio-paranoia*'
```

---

## Bekannte Besonderheiten

### Whipper 0.10.0

- Whipper wartet auf den gestarteten cd-paranoia-Prozess und besitzt keinen eigenen Timeout pro Track.
- Nach fehlgeschlagenen Tracks kann Whipper beim Schreiben des Abschlusslogs mit einem `NoneType`-Fehler abbrechen.
- Ein solcher Logger-Absturz ist ein Folgefehler und beweist nicht, dass zuvor erfolgreich erzeugte Tracks beschädigt sind.
- Umgekehrt beweist ein normaler Programmabschluss nicht allein, dass alle Tracks vollständig und verifiziert sind.

Daher sind immer Trackdateien, Rip-Log, CRC-Ergebnisse und AccurateRip maßgeblich.

### Beets 2.8.0

- Genehmigte unvollständige Rips werden mit `threaded: no` importiert.
- Das verhindert Probleme beim parallelen Zusammenführen bereits vorhandener Alben.
- Ein späteres Beets-Upgrade sollte separat getestet werden und darf die aktuelle funktionierende Importlogik nicht ungeprüft ersetzen.

---

## Später noch erledigen

Diese Punkte sind bewusst aufgeschoben und sollen nicht vergessen werden.

### 1. Beets-Datenbank aus `musictemp` herauslösen

Aktuell:

```yaml
library: /music/temp/beets.db
```

Dadurch darf `musictemp` noch nicht gelöscht oder aus Docker entfernt werden.

Geplante Zielstruktur:

```text
Host:      /opt/docker/beets/data/beets.db
Container: /data/beets.db
```

Geplanter Compose-Mount:

```yaml
- /opt/docker/beets/data:/data
```

Geplante Konfiguration:

```yaml
library: /data/beets.db
```

Migration nur kontrolliert durchführen:

1. Polling-Timer stoppen.
2. Datenbank sichern.
3. Datenbank nach `/opt/docker/beets/data` kopieren.
4. Compose-Mount und `library:` anpassen.
5. Container neu erstellen.
6. Anzahl und Stichproben der Beets-Einträge prüfen.
7. Erst danach entscheiden, ob `musictemp` vollständig entfallen kann.

### 2. Gebrauchtes vollformatiges CD-/DVD-Laufwerk beschaffen

Gesucht wird ein 5,25-Zoll-SATA-Laufwerk anderer Bauart als das vorhandene Slim-Blu-ray-Laufwerk. Es dient als zusätzlicher Fehlerleser für beschädigte Bibliotheks-CDs.

Interessante Modelle:

- `HL-DT-ST / LG BH14NS48`
- `ASUS BC-12B1ST`
- `Lite-On iHAS224 B`
- `TSSTcorp SH-216DB`

### 3. Optionalen cd-paranoia-Timeout entscheiden

Der Wrapper ist dokumentiert, aber noch nicht produktiv. Er wird eingerichtet, wenn stundenlange Hänger trotz manuellem Eingreifen weiterhin häufig vorkommen.

### 4. Beets-Upgrade prüfen

Aktuell läuft Beets 2.8.0. Ein Upgrade sollte zunächst in einer Sicherungskopie beziehungsweise mit gesicherter Datenbank getestet werden. Danach insbesondere prüfen:

- normaler Quiet-Import
- Approved-Import mit `merge`
- Verschieben der Audiodateien
- Lyrics-Plugin
- Duplicate-Reports
- Weboberfläche

### 5. LRCLib-Fehler optional untersuchen

Einzelne Titel liefern `400 Bad Request`, beispielsweise bei einer erkannten Dauer von `0`. Das blockiert den Musikimport nicht. Eine Prüfung ist nur nötig, wenn fehlende Lyrics störend werden.

### 6. Automatische Veröffentlichung lokaler Rips bleibt bewusst verworfen

Ein Marker-/Timer-Workflow wäre technisch möglich, spart nach der manuellen Logprüfung aber kaum Arbeit. Der derzeitige manuelle Schritt bleibt deshalb:

```text
Rip prüfen → Albumordner nach /mnt/nas/musicincome verschieben
```

---

## Lessons Learned

- Direktes Rippen auf ein NAS erschwert die Fehlerdiagnose; lokale Arbeitsdateien sind robuster.
- Nach der manuellen Logprüfung ist das direkte Verschieben nach `musicincome` einfacher als ein zusätzlicher Veröffentlichungs-Timer.
- Unterschiedliche Mount-Pfade zwischen Rip-Station und HP sind eine zentrale Fehlerquelle.
- `musictemp` ist derzeit kein Übergabeordner mehr, darf wegen `beets.db` aber noch nicht gelöscht werden.
- Reguläre und genehmigte unvollständige Rips benötigen getrennte Importregeln.
- Ein Beets-Exit-Code 0 beweist allein nicht, dass alle Audiodateien übernommen wurden.
- Ordner mit verbliebenen Audiodateien dürfen niemals nach `_IMPORTED_DONE` verschoben werden.
- Polling und eine klare Ordnerlogik sind robuster als Event-Watching auf NAS-Mounts.
- Bibliotheks-CDs können in einzelnen Bereichen physisch nicht reproduzierbar lesbar sein.
- Ein zweites Laufwerk mit anderer Mechanik ist bei Problem-CDs oft wirksamer als weitere Softwareoptionen.
- AAC nach FLAC zu konvertieren macht die ursprüngliche verlustbehaftete Kompression nicht rückgängig.

---

## Merksatz

**Lokal rippen. Log prüfen. Manuell freigeben. HP importiert. Problemfälle bleiben getrennt.**

---

## Weiterführende Dokumentation

- [Whipper – Projekt und Dokumentation](https://github.com/whipper-team/whipper)
- [Whipper `cd rip` – Optionen](https://github.com/whipper-team/whipper/blob/develop/man/whipper-cd-rip.rst)
- [CDDA Paranoia – Benutzerhandbuch](https://www.xiph.org/paranoia/manual.html)
- [Beets – Dokumentation](https://beets.readthedocs.io/)
- [XLD – offizielle Website](https://tmkk.undo.jp/xld/index_e.html)
- [CUERipper – CUETools Wiki](https://cue.tools/wiki/CUERipper)
- [CUETools Database](https://cue.tools/wiki/CUETools_Database)
