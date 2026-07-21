# Beets / Whipper CD-Rip-Workflow

## Zweck

Diese Dokumentation beschreibt den produktiven CD-Rip-Workflow für den Musikimport über **Whipper**, **Beets**, **Docker** und den **HP EliteDesk** als zentrale Import-Instanz.

Ziele:

- möglichst genaue und reproduzierbare CD-Rips
- klare Trennung zwischen Rippen, Übertragen, Prüfen und Importieren
- keine halbfertigen Rips im automatischen Beets-Eingang
- problematische CDs kontrolliert behandeln, statt Prozesse stundenlang hängen zu lassen
- eindeutige Ablage für erfolgreiche Importe, manuelle Prüfung und technische Fehler

Die Rollen bleiben klar getrennt:

- **Ubuntu-Rechner** = Rip-Station und lokaler Arbeitsbereich
- **HP EliteDesk** = Quelle der Wahrheit für NAS-Sichtprüfung, Import und Automatisierung
- **NAS** = zentrale Übergabe- und Musikablage

---

## Architektur

### Ubuntu-Rechner / Rip-Station

- rippt Audio-CDs primär mit `whipper`
- schreibt während des Rippens zunächst auf die lokale Platte:

  ```text
  $HOME/rip-local
  ```

- überträgt erst abgeschlossene Rips nach:

  ```text
  /mnt/nas/musictemp
  ```

Das lokale Rip-Ziel verhindert, dass ein stockender NAS-Mount den eigentlichen Lesevorgang blockiert oder wie ein CD-/Laufwerksfehler aussieht.

### HP EliteDesk / Docker-Host

- sieht dieselbe NAS-Freigabe unter:

  ```text
  /mnt/nas/media
  ```

- prüft, ob der vollständige Rip auf dem NAS angekommen ist
- verschiebt fertige Import-Pakete nach:

  ```text
  /mnt/nas/media/musicincome
  ```

- führt den Import über den Beets-Container aus
- betreibt Polling und Cleanup per `systemd`

### NAS

Zentrale Ablage für:

- `musictemp` – fertige, aber noch nicht freigegebene Rips
- `musicincome` – Eingang für vollständige Import-Pakete
- `Musik` – produktive Musikbibliothek
- `music-review/_IMPORTED_DONE` – erfolgreich verarbeitete Restdateien
- `music-review/_MANUAL_REVIEW` – inhaltlich oder qualitativ unklare Fälle
- `music-review/_FAILED_IMPORTS` – technische Importfehler

---

## Wichtige Pfade

### Ubuntu-Rechner

| Zweck | Pfad |
|---|---|
| Lokaler Rip-Arbeitsbereich | `$HOME/rip-local` |
| Übergabe auf dem NAS | `/mnt/nas/musictemp` |

### HP EliteDesk

| Zweck | Pfad |
|---|---|
| NAS-Mount | `/mnt/nas/media` |
| Sicht auf die Rip-Übergabe | `/mnt/nas/media/musictemp` |
| Eingang für fertige Import-Pakete | `/mnt/nas/media/musicincome` |
| Produktive Bibliothek | `/mnt/nas/media/Musik` |
| Erfolgreich verarbeitete Restordner | `/mnt/nas/media/music-review/_IMPORTED_DONE` |
| Manuelle Nachbearbeitung | `/mnt/nas/media/music-review/_MANUAL_REVIEW` |
| Technische Fehler | `/mnt/nas/media/music-review/_FAILED_IMPORTS` |

### Beets-Container

| Zweck | Pfad |
|---|---|
| Eingang | `/music/incoming` |
| Bibliothek | `/music/library` |
| Beets-Datenbank | `/music/temp/beets.db` |

---

## Mount-Unterschiede zwischen Ubuntu-Rechner und HP

Die NAS-Freigabe wird auf den beiden Systemen unterschiedlich eingehängt:

- **Ubuntu-Rechner:** `/mnt/nas`
- **HP EliteDesk:** `/mnt/nas/media`

Daher gilt:

```text
Ubuntu: /mnt/nas/musictemp
HP:     /mnt/nas/media/musictemp
```

Nicht auf dem Ubuntu-Rechner versehentlich `/mnt/nas/media/...` verwenden. Das würde in einen falschen Unterpfad innerhalb der Freigabe schreiben.

---

## Aktuell geprüfte Rip-Umgebung

Stand der bisherigen Diagnose:

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

Eine angezeigte `Rip quality` unter 100 % ist dagegen nicht automatisch ein Fehler. Wenn AccurateRip den Track bestätigt, ist der Rip trotz zusätzlicher Korrekturarbeit verifiziert.

### 4. Fertigen Rip auf das NAS kopieren

Beispiel für einen Albumordner:

```bash
rsync -a --partial --info=progress2 \
  "$HOME/rip-local/<ALBUMORDNER>/" \
  "/mnt/nas/musictemp/<ALBUMORDNER>/"
```

Anschließend optional bytegenau gegenprüfen:

```bash
rsync -a --checksum --dry-run \
  "$HOME/rip-local/<ALBUMORDNER>/" \
  "/mnt/nas/musictemp/<ALBUMORDNER>/"
```

Keine Ausgabe bedeutet, dass `rsync` keine Unterschiede gefunden hat.

Die lokale Kopie erst löschen, wenn der Rip auf dem HP sichtbar ist und der Import erfolgreich abgeschlossen wurde.

### 5. Auf dem HP prüfen, ob der Rip sichtbar ist

```bash
find /mnt/nas/media/musictemp -maxdepth 3 -type d | sort
find /mnt/nas/media/musictemp -maxdepth 3 -type f | sort
```

Erst wenn der HP den vollständigen Ordner unter `/mnt/nas/media/musictemp/...` sieht, gilt der Rip als auf dem NAS angekommen.

### 6. Albumordner nach `musicincome` verschieben

```bash
mv "/mnt/nas/media/musictemp/<ALBUMORDNER>" \
   "/mnt/nas/media/musicincome/"
```

`musicincome` ist nur für fertige Import-Pakete vorgesehen. Dort darf nicht mehr gerippt, kopiert oder nachträglich ergänzt werden.

### 7. Polling übernimmt den Import

Der Polling-Mechanismus erkennt ruhende, vollständige Import-Pakete und startet den nicht-interaktiven Beets-Import.

### 8. Ergebnis prüfen

```bash
docker exec -it music-beets beet ls | tail -n 30
find /mnt/nas/media/Musik -maxdepth 4 -type f | tail -n 30
```

---

## Qualitätsampel

| Status | Bedeutung | Aktion |
|---|---|---|
| Grün | Alle Tracks AccurateRip-bestätigt | Nach `musictemp`, anschließend Import |
| Gelb | Reproduzierbare CRCs, aber kein Datenbanktreffer | Log aufheben, bei Bedarf zweites Laufwerk testen |
| Rot | Unterschiedliche CRCs, fehlende Tracks, Null-Byte-Dateien oder Transportfehler | Nicht importieren; Problem-CD-Workflow starten |

Bei einem roten Ergebnis dürfen vorhandene, scheinbar abspielbare Dateien nicht automatisch in die produktive Bibliothek übernommen werden.

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

## Was die beobachteten cd-paranoia-Meldungen bedeuten

| Meldung | Einordnung |
|---|---|
| `[read]` | Laufwerk liest einen neuen Bereich |
| `[verify]` | neu gelesene Daten werden mit vorhandenen Daten verglichen |
| `[correction]` | widersprüchliche Daten werden durch erneutes Lesen zu korrigieren versucht |
| `[overlap]` | cd-paranoia sucht die korrekte Überlappung zwischen Leseblöcken |
| `[transport error]` | SCSI-/ATAPI-Leseanforderung wurde nicht regulär erfüllt |
| Prozessstatus `D` / `blk_execute_rq` | Prozess wartet im Kernel auf eine Geräteanforderung |

Viele Korrektur-, Overlap- und Verify-Meldungen rund um dieselben Positionen bedeuten, dass cd-paranoia nicht eingefroren ist, sondern wiederholt versucht, einen instabil lesbaren Bereich zu rekonstruieren.

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

## Optional: Timeout nur für cd-paranoia

Whipper besitzt in Version 0.10.0 keinen eigenen Timeout für einen einzelnen Lesevorgang. Ein Wrapper kann verhindern, dass ein Track unbegrenzt lange gelesen wird.

### Wrapper anlegen

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

### Whipper mit Wrapper starten

```bash
PATH="$HOME/.local/libexec/whipper:$PATH" \
  whipper cd rip -p -k -r 2 \
    -O "$HOME/rip-local" \
    --track-template "%A/%d/%t - %n" \
    --disc-template "%A/%d/%A - %d"
```

Der Wrapper verbessert nicht die Lesbarkeit. Er setzt nur eine Obergrenze pro cd-paranoia-Aufruf. Ein Prozess im Kernelstatus `D` kann erst beendet werden, wenn die aktuell laufende Geräteanforderung zum Kernel zurückkehrt.

---

## Alternative Rip-Software

### Linux: Cyanrip

Cyanrip kann als alternativer Ablauf getestet werden. Es verwendet jedoch ebenfalls die libcdio-/cd-paranoia-Basis. Deshalb ist bei physisch schwer lesbaren Bereichen keine grundsätzlich andere Laufwerksleistung zu erwarten.

### macOS: XLD

Für archivfähige Rips:

- Ausgabeformat `FLAC` oder `WAV`
- `XLD Secure Ripper` verwenden
- AccurateRip aktivieren
- Logdatei aufheben

Nicht als Standard für problematische Archiv-Rips verwenden:

- Burst-Modus
- Paranoia/Verifikation deaktiviert

Diese schnellen Modi können eine Datei erzeugen, reduzieren aber die Sicherheit, dass die Audiodaten korrekt sind.

### Windows: CUERipper / CUETools

CUERipper ist für schwierige CDs eine sinnvolle zusätzliche Strategie:

- AccurateRip-Unterstützung
- CTDB-Unterstützung
- vollständige Images mit CUE und Log möglich
- kleine beschädigte Bereiche können bei passendem Datenbankeintrag eventuell über CTDB rekonstruiert werden

Ein großflächig oder auf mehreren Laufwerken nicht reproduzierbar lesbarer Track ist jedoch auch damit nicht garantiert reparierbar.

### AAC/M4A nur als Hörkopie

Ein Rip als AAC/M4A kann als letzte Möglichkeit für eine reine Hörkopie dienen, ist aber kein verlustfreier Archiv-Rip.

Eine Konvertierung von AAC/M4A nach FLAC:

```bash
ffmpeg -i input.m4a -c:a flac output.flac
```

macht die Datei technisch zu FLAC, stellt aber die bei der AAC-Kompression verlorenen Informationen nicht wieder her. Die Qualität bleibt die einer verlustbehafteten Quelle. Das Original-M4A sollte deshalb erhalten und die Herkunft klar gekennzeichnet werden.

---

## Beets-Konzept

### Import-Verhalten

- Import läuft bewusst nicht-interaktiv im Polling-Workflow.
- Problematische Fälle werden nicht blind übernommen.
- Fälle werden getrennt in:
  - erfolgreich verarbeitet
  - manuelle Nachbearbeitung
  - technischer Fehler

### Wichtige Beets-Entscheidungen

- `duplicate_action: skip`
- MusicBrainz aktiv
- `duplicates`-Plugin aktiv
- vereinfachte Lyrics-Konfiguration ohne unnötige Zusatzquellen

### Bibliothekspfad

Beispiel für Compilations:

```text
Compilations/<Albumname>/...
```

Dadurch entstehen nicht unnötig viele `Various Artists`-Ordner.

---

## Duplicate-Strategie

### Grundsatz

- Offensichtliche Doppelimporte möglichst beim Import abfangen.
- Sampler und Originalalben nicht allein deshalb als Fehler behandeln, weil ein Titel mehrfach vorkommt.
- Duplikate nicht automatisch löschen.

### Report-Befehle

Exakte Duplicate-Alben:

```bash
docker exec -it music-beets beet duplicates -a -F -p
```

Exakte Duplicate-Tracks:

```bash
docker exec -it music-beets beet duplicates -F -p
```

Gleicher Artist und Titel:

```bash
docker exec -it music-beets \
  beet duplicates -k title -k artist -F -p
```

Keine Ausgabe bedeutet in der Regel, dass aktuell keine passenden Duplikate erkannt wurden.

---

## Polling-Automatisierung

### Komponenten

| Komponente | Pfad |
|---|---|
| Polling-Skript | `/opt/docker/beets/poll-import.sh` |
| Service | `/etc/systemd/system/beets-poll.service` |
| Timer | `/etc/systemd/system/beets-poll.timer` |

### Produktive Werte

- Polling-Intervall: **5 Minuten**
- Ruhezeit vor Import: **15 Minuten ohne Änderungen**

### Verhalten

#### Erfolgreicher Import

- Audiodateien werden nach `/mnt/nas/media/Musik` verschoben.
- Restordner mit `.cue`, `.log`, `.m3u` oder `.toc` werden nach folgendem Pfad verschoben:

  ```text
  /mnt/nas/media/music-review/_IMPORTED_DONE
  ```

#### Problemfall ohne technischen Fehler

```text
/mnt/nas/media/music-review/_MANUAL_REVIEW
```

#### Technischer Fehler

```text
/mnt/nas/media/music-review/_FAILED_IMPORTS
```

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

Vor dem automatischen Löschen muss sichergestellt sein, dass wichtige Whipper-, AccurateRip- oder Diagnose-Logs an anderer Stelle aufbewahrt werden, falls diese langfristig benötigt werden.

---

## Multi-Disc-Sets

- Alle Discs zunächst vollständig lokal rippen.
- Jede Disc einzeln qualitativ prüfen.
- Das vollständige Set gemeinsam nach `musictemp` übertragen.
- Erst nach Abschluss aller Discs das gesamte Paket nach `musicincome` verschieben.

`musicincome` ist der Eingang für fertige Import-Pakete und kein Arbeitsbereich für halbfertige Sets.

---

## Manuelle Nachbearbeitung

Fälle landen unter:

```text
/mnt/nas/media/music-review/_MANUAL_REVIEW
```

Typische Gründe:

- Beets überspringt den Fall im Quiet-Modus.
- Zuordnung ist nicht sicher genug.
- Metadatenlage ist unklar.
- AccurateRip kennt die Ausgabe nicht.
- Ein Track ist nur als nicht verifizierte Hörkopie vorhanden.
- Multi-Disc-Zuordnung ist unvollständig.

---

## Betriebsbefehle

### Timer prüfen

```bash
systemctl list-timers --all \
  | grep -E 'beets-poll|beets-cleanup'
```

### Polling-Log prüfen

```bash
tail -n 100 /opt/docker/beets/beets-poll.log
journalctl -u beets-poll.service -n 50 --no-pager
```

### Cleanup-Log prüfen

```bash
journalctl -u beets-cleanup.service -n 50 --no-pager
```

### Beets-Datenbank prüfen

```bash
docker exec -it music-beets beet ls | tail -n 50
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

## Bekannte Besonderheiten von Whipper 0.10.0

- Whipper wartet auf den gestarteten cd-paranoia-Prozess und besitzt keinen eigenen Timeout pro Track.
- Nach fehlgeschlagenen Tracks kann Whipper beim Schreiben des Abschlusslogs mit einem `NoneType`-Fehler abbrechen.
- Ein solcher Logger-Absturz ist ein Folgefehler und beweist nicht, dass zuvor erfolgreich erzeugte Tracks beschädigt sind.
- Umgekehrt beweist ein normaler Programmabschluss nicht allein, dass alle Tracks vollständig und verifiziert sind.

Daher sind immer die Trackdateien, das Rip-Log, die CRC-Ergebnisse und AccurateRip maßgeblich.

---

## Lessons Learned

- Rippen direkt auf ein NAS erschwert die Fehlerdiagnose; lokale Arbeitsdateien sind robuster.
- Unterschiedliche Mount-Pfade zwischen Ubuntu-Rechner und HP waren eine zentrale Fehlerquelle.
- Der HP bleibt die Quelle der Wahrheit für die NAS-Sicht und den Importstatus.
- `inotify` auf NAS/NFS war für diesen Workflow die falsche Wahl.
- Polling und eine klare Ordnerlogik sind robuster.
- Bibliotheks-CDs können in einzelnen Bereichen physisch nicht reproduzierbar lesbar sein.
- Ein zweites Laufwerk mit anderer Mechanik ist bei Problem-CDs oft wirksamer als weitere Softwareoptionen.
- Niedrigere Geschwindigkeit kann helfen, ist aber kein Ersatz für eine reproduzierbare Prüfsumme.
- AAC nach FLAC zu konvertieren macht die ursprüngliche verlustbehaftete Kompression nicht rückgängig.
- Lieber einen Fall nach `_MANUAL_REVIEW` verschieben, als einen unbestätigten Rip blind zu importieren.

---

## Merksatz

**Ubuntu rippt lokal. Ubuntu überträgt. HP verifiziert. HP verschiebt. HP importiert.**

---

## Weiterführende Dokumentation

- [Whipper – Projekt und Dokumentation](https://github.com/whipper-team/whipper)
- [Whipper `cd rip` – Optionen](https://github.com/whipper-team/whipper/blob/develop/man/whipper-cd-rip.rst)
- [CDDA Paranoia – Benutzerhandbuch](https://www.xiph.org/paranoia/manual.html)
- [XLD – offizielle Website](https://tmkk.undo.jp/xld/index_e.html)
- [CUERipper – CUETools Wiki](https://cue.tools/wiki/CUERipper)
- [CUETools Database](https://cue.tools/wiki/CUETools_Database)
