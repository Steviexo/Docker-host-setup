# Beets / Whipper CD-Rip-Workflow

## Zweck

Diese Dokumentation beschreibt den produktiven CD-Rip-Workflow für den Musikimport über **Whipper**, **Beets**, **Docker** und den **HP EliteDesk** als zentrale Import-Instanz.

Ziel ist ein sauberer, reproduzierbarer und möglichst weit automatisierter Ablauf mit klaren Zuständigkeiten:

* **Ubuntu-Rechner** = Rip-Station
* **HP EliteDesk** = Quelle der Wahrheit für Sichtprüfung, Import und Automatisierung
* **NAS** = zentrale Ablage

---

## Architektur

### Rollen

#### Ubuntu-Rechner / Rip-Station

* Rippt CDs mit `whipper`
* schreibt fertige Rips nach:

  * `/mnt/nas/musictemp`

#### HP EliteDesk / Docker-Host

* sieht dieselbe Freigabe unter:

  * `/mnt/nas/media`
* prüft, ob der Rip wirklich auf dem NAS angekommen ist
* verschiebt fertige Import-Pakete nach:

  * `/mnt/nas/media/musicincome`
* führt den Import über den Beets-Container aus
* betreibt die Polling- und Cleanup-Automatisierung per `systemd`

#### NAS

* zentrale Datenablage für:

  * `musictemp`
  * `musicincome`
  * `Musik`
  * `_IMPORTED_DONE`
  * `_MANUAL_REVIEW`
  * `_FAILED_IMPORTS`

---

## Wichtige Pfade

### Auf dem Ubuntu-Rechner

* Staging / Rip-Ziel:

  * `/mnt/nas/musictemp`

### Auf dem HP EliteDesk

* NAS-Mount:

  * `/mnt/nas/media`
* Eingang für fertige Import-Pakete:

  * `/mnt/nas/media/musicincome`
* Bibliothek:

  * `/mnt/nas/media/Musik`
* Erfolgreich verarbeitete Restordner:

  * `/mnt/nas/media/_IMPORTED_DONE`
* Manuelle Nachbearbeitung:

  * `/mnt/nas/media/_MANUAL_REVIEW`
* Technische Fehler:

  * `/mnt/nas/media/_FAILED_IMPORTS`

### Im Beets-Container

* Eingang:

  * `/music/incoming`
* Bibliothek:

  * `/music/library`
* temporäre Beets-Datenbank:

  * `/music/temp/beets.db`

---

## Mount-Unterschiede zwischen Ubuntu-Rechner und HP

Dieser Punkt ist kritisch.

Die NAS-Freigabe wird auf den beiden Systemen **unterschiedlich eingehängt**:

* **Ubuntu-Rechner:** `/mnt/nas`
* **HP EliteDesk:** `/mnt/nas/media`

Das bedeutet:

* Was auf dem Ubuntu-Rechner unter `/mnt/nas/musictemp` liegt,
* erscheint auf dem HP unter `/mnt/nas/media/musictemp`.

Wichtig:

Nicht versehentlich auf dem Ubuntu-Rechner Pfade wie `/mnt/nas/media/...` verwenden. Das würde in einen falschen Unterpfad innerhalb der Freigabe schreiben.

---

## Produktiver Workflow

### 1. CD auf dem Ubuntu-Rechner rippen

Empfohlener Aufruf:

```bash
whipper cd rip -p -k \
  -O /mnt/nas/musictemp \
  --track-template "%A/%d/%t - %n" \
  --disc-template "%A/%d/%A - %d"
```

### 2. Auf dem HP prüfen, ob der Rip sichtbar ist

Beispiel:

```bash
find /mnt/nas/media/musictemp -maxdepth 3 -type d | sort
find /mnt/nas/media/musictemp -maxdepth 3 -type f | sort
```

Erst wenn der HP den Rip unter `/mnt/nas/media/musictemp/...` sieht, gilt er als wirklich auf dem NAS angekommen.

### 3. Fertigen Albumordner auf dem HP nach `musicincome` verschieben

Beispiel:

```bash
mv "/mnt/nas/media/musictemp/<ALBUMORDNER>" "/mnt/nas/media/musicincome/"
```

### 4. Polling übernimmt den Rest automatisch

Der Polling-Mechanismus auf dem HP erkennt fertige Import-Pakete in `musicincome` und führt den Import automatisch aus.

### 5. Ergebnis prüfen

Beispiel:

```bash
docker exec -it music-beets beet ls | tail -n 30
find /mnt/nas/media/Musik -maxdepth 4 -type f | tail -n 30
```

---

## Beets-Konzept

### Import-Verhalten

* Import läuft bewusst **nicht-interaktiv** im Polling-Workflow
* problematische Fälle werden **nicht blind übernommen**
* stattdessen klare Trennung zwischen:

  * erfolgreich verarbeitet
  * manuelle Nachbearbeitung
  * technischer Fehler

### Wichtige Beets-Entscheidungen

* `duplicate_action: skip`
* `musicbrainz` aktiv
* `duplicates`-Plugin aktiv
* vereinfachte Lyrics-Konfiguration ohne unnötige Zusatzquellen

### Bibliothekspfad

Die Pfadlogik ist so eingestellt, dass die Library sauber strukturiert wird.

Beispiel für Compilations:

* `Compilations/<Albumname>/...`

Dadurch entstehen in der Musikbibliothek nicht unnötig viele `Various Artists`-Ordner.

---

## Duplicate-Strategie

### Grundsatz

* Offensichtliche Doppelimporte sollen nach Möglichkeit schon beim Import abgefangen werden.
* Gleichzeitig sollen Sampler und Originalalben **nicht blind** als Fehler behandelt werden, nur weil ein Song mehrfach vorkommt.

### Aktuelle Strategie

* `duplicate_action: skip` bleibt aktiv
* das `duplicates`-Plugin bleibt aktiv
* Duplikate werden vorerst **nicht automatisch gelöscht**
* bei Bedarf werden Reports manuell ausgeführt

### Report-Befehle

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

* Keine Ausgabe bedeutet in der Regel: aktuell keine erkannten Duplikate.
* Sampler + Künstleralbum mit demselben Song sind nicht automatisch ein Fehler.
* Erst wenn später echte Doppelimporte sichtbar werden, sollte die Strategie verschärft werden.

---

## Polling-Automatisierung

### Ziel

Der frühere `inotify`-Ansatz wurde verworfen. Stattdessen läuft eine Polling-Lösung per `systemd`.

### Komponenten

#### Polling-Skript

* `/opt/docker/beets/poll-import.sh`

#### Service

* `/etc/systemd/system/beets-poll.service`

#### Timer

* `/etc/systemd/system/beets-poll.timer`

### Produktive Werte

* Polling-Intervall: **5 Minuten**
* Ruhezeit vor Import: **15 Minuten ohne Änderungen**

### Verhalten

#### Erfolgreicher Import

* Audiodateien werden nach `/mnt/nas/media/Musik` verschoben
* Restordner mit `.cue`, `.log`, `.m3u`, `.toc` werden nach:

  * `/mnt/nas/media/_IMPORTED_DONE`
    verschoben

#### Problemfall ohne technischen Fehler

* Fall wird nach:

  * `/mnt/nas/media/_MANUAL_REVIEW`
    verschoben

#### Technischer Fehler

* Fall wird nach:

  * `/mnt/nas/media/_FAILED_IMPORTS`
    verschoben

---

## Cleanup-Automatisierung

### Ziel

`_IMPORTED_DONE` soll nicht dauerhaft mit alten Restordnern volllaufen.

### Komponenten

#### Cleanup-Skript

* `/opt/docker/beets/cleanup-imported-done.sh`

#### Service

* `/etc/systemd/system/beets-cleanup.service`

#### Timer

* `/etc/systemd/system/beets-cleanup.timer`

### Regel

* alles in `_IMPORTED_DONE`, das älter als **30 Tage** ist, wird gelöscht

---

## Multi-Disc-Hinweis

Für größere Multi-Disc-Sets gilt:

* immer erst komplett nach `musictemp` rippen
* erst wenn **alle Discs** fertig sind, das gesamte Paket nach `musicincome` verschieben

`musicincome` ist der **Eingang für fertige Import-Pakete**, nicht der Arbeitsbereich für halbfertige Rips.

---

## Manuelle Nachbearbeitung

Wenn ein Fall nicht automatisch übernommen wird, landet er in:

* `/mnt/nas/media/_MANUAL_REVIEW`

Dort kann später bewusst manuell geprüft werden.

Typische Gründe:

* Beets überspringt den Fall im Quiet-Modus
* Zuordnung ist nicht sauber genug
* Metadatenlage ist unklar

---

## Betriebsbefehle

### Timer prüfen

```bash
systemctl list-timers --all | grep -E 'beets-poll|beets-cleanup'
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

---

## Lessons Learned

* Unterschiedliche Mount-Pfade zwischen Ubuntu-Rechner und HP waren die zentrale Fehlerquelle.
* Der HP muss die **Quelle der Wahrheit** bleiben.
* `inotify` auf NAS/NFS war für diesen Workflow die falsche Wahl.
* Polling + klare Ordnerlogik ist robuster.
* Lieber problematische Fälle sauber nach `_MANUAL_REVIEW` schieben als blind falsch importieren.

---

## Merksatz

**Ubuntu-Rechner rippt. HP verifiziert. HP verschiebt. HP importiert.**
