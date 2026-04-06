# Troubleshooting: Beets / Whipper Import-Pipeline

## Zweck

Diese Datei sammelt wiederverwendbare Prüfpfade für den CD-Rip- und Import-Workflow mit **Whipper**, **Beets**, **Docker** und dem **HP EliteDesk** als zentraler Import-Instanz.

Sie ist bewusst als operative Fehleranalyse gedacht:

* sieht der HP den Rip?
* sieht der Container `/music/incoming`?
* landet ein Fall in `_MANUAL_REVIEW`?
* läuft `beets-poll.timer`?
* gibt es erkennbare Duplikate?

---

## Grundprinzip

### Rollen

* **Ubuntu-Rechner** = Rip-Station
* **HP EliteDesk** = Quelle der Wahrheit
* **NAS** = zentrale Ablage
* **Beets-Container** = Import-Instanz

### Kritischer Mount-Unterschied

Die Freigabe ist auf beiden Systemen unterschiedlich eingehängt:

* Ubuntu-Rechner:

  * `/mnt/nas`
* HP EliteDesk:

  * `/mnt/nas/media`

Daraus folgt:

* Ubuntu-Rechner schreibt nach `/mnt/nas/musictemp`
* HP sieht denselben Inhalt unter `/mnt/nas/media/musictemp`

Wenn hier versehentlich der falsche Pfad genutzt wird, landet der Rip schnell im falschen Unterordner.

---

## 1. Sieht der HP den Rip?

### Typisches Problem

* Auf dem Rip-System scheint der Albumordner da zu sein
* Auf dem HP ist er aber nicht sichtbar
* Beets kann deshalb nichts importieren

### Prüfen auf dem HP

```bash
find /mnt/nas/media/musictemp -maxdepth 3 -type d | sort
find /mnt/nas/media/musictemp -maxdepth 3 -type f | sort
```

### Prüfen auf dem Ubuntu-Rechner

```bash
find /mnt/nas/musictemp -maxdepth 3 -type d | sort
find /mnt/nas/musictemp -maxdepth 3 -type f | sort
```

### Erwartung

* Der Albumordner muss auf beiden Systemen sichtbar sein
* Wenn nicht: zuerst die Pfad-/Mount-Frage klären, nicht Beets debuggen

### Probe-Datei-Test

Auf dem Ubuntu-Rechner:

```bash
TS=$(date +%Y%m%d_%H%M%S)
echo "probe-$TS" > "/mnt/nas/musictemp/IMAC_PROBE_$TS.txt"
sync
ls -lah /mnt/nas/musictemp | grep IMAC_PROBE
```

Auf dem HP:

```bash
ls -lah /mnt/nas/media/musictemp | grep IMAC_PROBE
find /mnt/nas/media/musictemp -maxdepth 1 -type f | grep IMAC_PROBE
```

Wenn der HP die Datei nicht sieht, stimmt der Pfad auf dem Rip-System nicht.

---

## 2. Sieht der Container `/music/incoming`?

### Typisches Problem

* Der HP sieht den Ordner unter `musicincome`
* Im Container kommt aber nichts an
* `beet import` liefert dann "no such file or directory"

### Auf dem HP prüfen

```bash
find /mnt/nas/media/musicincome -maxdepth 3 -type d | sort
find /mnt/nas/media/musicincome -maxdepth 3 -type f | sort
```

### Im Container prüfen

```bash
docker exec music-beets find /music/incoming -maxdepth 4 -type d | sort
docker exec music-beets find /music/incoming -maxdepth 4 -type f | sort
```

### Docker-Mounts prüfen

```bash
docker inspect music-beets --format '{{json .Mounts}}'
```

### Erwartung

* Was unter `/mnt/nas/media/musicincome/...` liegt, muss im Container unter `/music/incoming/...` sichtbar sein
* Wenn nicht: Docker-Mount prüfen

---

## 3. Läuft der Polling-Import?

### Relevante Komponenten

* Skript:

  * `/opt/docker/beets/poll-import.sh`
* Service:

  * `beets-poll.service`
* Timer:

  * `beets-poll.timer`
* Logdatei:

  * `/opt/docker/beets/beets-poll.log`

### Timer prüfen

```bash
systemctl list-timers --all | grep beets-poll
systemctl status beets-poll.timer --no-pager -l
```

### Service prüfen

```bash
systemctl status beets-poll.service --no-pager -l
journalctl -u beets-poll.service -n 50 --no-pager
```

### Log prüfen

```bash
tail -n 100 /opt/docker/beets/beets-poll.log
```

### Typische erwartete Logzeilen

#### Nichts zu tun

```text
INFO: Keine Unterordner in musicincome gefunden.
```

#### Ordner noch zu frisch

```text
INFO: Überspringe noch aktiven Ordner: ...
```

#### Automatischer Import

```text
INFO: Starte Auto-Import für: ...
INFO: beet import beendet für: ...
```

#### Erfolgreicher Abschluss

```text
INFO: Restordner nach erfolgreichem Import verschoben nach: ...
```

#### Problemfall

```text
WARNUNG: Ordner benötigt manuelle Nachbearbeitung. Verschoben nach: ...
```

#### Technischer Fehler

```text
FEHLER: beet import fehlgeschlagen für ...
```

---

## 4. Warum landet ein Fall in `_MANUAL_REVIEW`?

### Bedeutung

`_MANUAL_REVIEW` ist **kein technischer Crash-Ordner**, sondern ein Sammelort für Fälle, die Beets im nicht-interaktiven Modus nicht sauber übernehmen wollte.

### Typische Ursachen

* unsicheres oder schlechtes Match
* Quiet-Fallback = `skip`
* Audio-Dateien bleiben nach dem Import im Quellordner liegen

### Prüfen

```bash
find /mnt/nas/media/_MANUAL_REVIEW -maxdepth 4 | sort
```

### Manuelle Nachbearbeitung

Passenden Ordner im Container prüfen:

```bash
docker exec music-beets find /music/incoming -maxdepth 5 | sort
```

Dann gezielt interaktiv importieren, falls der Fall bewusst geprüft werden soll.

Beispiel:

```bash
docker exec -it music-beets beet import "/music/incoming/<PFAD_ZUM_FALL>"
```

### Bewertung

`_MANUAL_REVIEW` ist gewollt. Lieber dort sauber parken als blind falsch importieren.

---

## 5. Warum landet ein Fall in `_FAILED_IMPORTS`?

### Bedeutung

Hier landen technische Fehlerfälle.

### Typische Ursachen

* Container nicht erreichbar
* Pfadproblem
* echter `beet import`-Exit-Code ungleich 0

### Prüfen

```bash
find /mnt/nas/media/_FAILED_IMPORTS -maxdepth 4 | sort
journalctl -u beets-poll.service -n 100 --no-pager
```

### Nächster Schritt

Immer zuerst die konkrete Logzeile aus `beets-poll.log` bzw. dem Journal lesen. Nicht raten.

---

## 6. Warum liegt ein Restordner in `_IMPORTED_DONE`?

### Bedeutung

Der Import war erfolgreich, die Audio-Dateien wurden in die Bibliothek verschoben, aber Begleitdateien sind übrig geblieben.

Typisch:

* `.cue`
* `.log`
* `.m3u`
* `.toc`

### Prüfen

```bash
find /mnt/nas/media/_IMPORTED_DONE -maxdepth 4 | sort
```

### Wichtig

`_IMPORTED_DONE` ist **kein zweites Musikarchiv**. Die eigentlichen Audiodateien liegen nach erfolgreichem Import in `/mnt/nas/media/Musik`.

---

## 7. Läuft das Cleanup für `_IMPORTED_DONE`?

### Komponenten

* Skript:

  * `/opt/docker/beets/cleanup-imported-done.sh`
* Service:

  * `beets-cleanup.service`
* Timer:

  * `beets-cleanup.timer`

### Prüfen

```bash
systemctl list-timers --all | grep beets-cleanup
systemctl status beets-cleanup.timer --no-pager -l
journalctl -u beets-cleanup.service -n 50 --no-pager
```

### Zweck

Alles in `_IMPORTED_DONE`, das älter als 30 Tage ist, wird automatisch entfernt.

---

## 8. Duplicate-Reports

### Grundsatz

* Beim Import bleibt `duplicate_action: skip` aktiv
* Das `duplicates`-Plugin ist aktiv
* Duplikate werden aktuell **nicht automatisch gelöscht**

### Reports

#### Exakte Duplicate-Alben prüfen

```bash
docker exec -it music-beets beet duplicates -a -F -p
```

#### Exakte Duplicate-Tracks prüfen

```bash
docker exec -it music-beets beet duplicates -F -p
```

#### Kandidaten mit gleichem Artist + Titel prüfen

```bash
docker exec -it music-beets beet duplicates -k title -k artist -F -p
```

### Bewertung

* Keine Ausgabe bedeutet in der Regel: aktuell keine erkannten Duplikate
* Sampler + Künstleralbum mit identischem Song sind nicht automatisch ein Fehler

---

## 9. Direkte Beets-Prüfungen

### Importierte Titel anzeigen

```bash
docker exec -it music-beets beet ls | tail -n 50
```

### Bestimmtes Album suchen

```bash
docker exec -it music-beets beet ls "<ALBUMNAME>"
```

### Bibliothek auf dem NAS prüfen

```bash
find /mnt/nas/media/Musik -maxdepth 4 -type f | tail -n 50
```

---

## 10. Häufige Fehlerbilder

### A. Auf dem Rip-System sichtbar, auf dem HP nicht

**Ursache:** falscher Pfad auf dem Ubuntu-Rechner, meist versehentlich `/mnt/nas/media/...` statt `/mnt/nas/...`

### B. Auf dem HP sichtbar, im Container nicht

**Ursache:** Docker-Mountproblem oder falscher Zielordner in `musicincome`

### C. `No files imported`

**Ursache:** falscher Container-Pfad oder im Ordner liegen nur noch `.cue/.log/.m3u/.toc`, aber keine Audiodateien mehr

### D. `Skipping.` im Auto-Import

**Ursache:** Beets hat den Fall im Quiet-Modus nicht automatisch übernommen → `_MANUAL_REVIEW`

### E. Polling reagiert nicht sofort

**Ursache:** Produktive Sicherheitswerte:

* Polling alle 5 Minuten
* Import erst nach 15 Minuten ohne Änderungen

---

## 11. Betriebs-Merksatz

**Ubuntu-Rechner rippt. HP verifiziert. HP verschiebt. HP importiert.**
