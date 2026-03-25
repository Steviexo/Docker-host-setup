# Incident: Plex-Bibliothek leer und LibreELEC/PKC-Wiedergabefehler

## Kurzfassung

Plex lief als Docker-Container auf dem HP EliteDesk (Ubuntu Server), die Film- und Serienbibliotheken blieben jedoch leer, obwohl die Mediendateien auf dem NAS lagen und per NFS auf dem Host eingebunden waren. Nachdem das Bibliotheksproblem behoben war, scheiterte die Wiedergabe auf dem LibreELEC-Client mit PlexKodiConnect (PKC) weiterhin.

Der Vorfall bestand letztlich aus zwei getrennten Ursachen:

1. Der Wurzelordner des NFS-Shares auf der Synology war für den Plex-Prozess im Container nicht traversierbar.
2. Der LibreELEC-Client hing noch an Resten des alten Plex-/NAS-Setups, und die manuelle PKC-Verbindung funktionierte nur per HTTP, nicht per HTTPS auf Basis der lokalen IP.

---

## Umgebung

### Serverseite

- **Host:** HP EliteDesk 800 G5 Mini
- **Betriebssystem:** Ubuntu Server
- **Rolle:** Docker-Host
- **Plex-Container-Image:** `lscr.io/linuxserver/plex:latest`
- **Plex-Konfigurationspfad:** `/opt/docker/plex/config`
- **Transcode-Pfad:** `/opt/docker/plex/transcode`
- **Medien-Mount auf dem Host:** `/mnt/nas/media`
- **Medien-Mount im Container:** `/media`
- **IP des Plex-Hosts:** `192.168.88.106`

### Storage

- **NAS:** Synology
- **IP des NAS:** `192.168.88.32`
- **Exportierter Pfad:** `/volume2/media`
- **Eingebunden auf dem Host via:** NFSv3

### Clientseite

- **Client-Gerät:** Raspberry Pi 5
- **Client-Betriebssystem:** LibreELEC / Kodi 21.2
- **Plex-Anbindung:** PlexKodiConnect (PKC)
- **IP des LibreELEC-Clients:** `192.168.88.169`

---

## Symptome

### Plex-Server

- Der Plex-Container lief unauffällig.
- Die Medienverzeichnisse waren im Container sichtbar.
- Plex-Bibliotheken für Filme und Serien blieben trotz Scan leer.
- Selbst Testinhalte wie `Alien (1979)/Alien (1979).mkv` wurden nicht hinzugefügt.

### LibreELEC / PKC

- PKC zeigte Wiedergabefehler.
- Kodi meldete `NAS offline`.
- Alte Kollektionen aus einem früheren Setup waren weiterhin sichtbar.
- PKC fand den Plex-Server nicht automatisch.
- Eine manuelle HTTPS-Verbindung schlug mit SSL-Fehlern fehl.

---

## Analyse

### 1. NFS-Mount auf dem Host prüfen

Der NFS-Mount war vorhanden und der Host konnte die Mediendateien korrekt sehen.

Beispielhafte Prüfungen:

```bash
findmnt -t nfs,nfs4,cifs,smb3
mount | egrep 'nfs|cifs|smb'
df -h
ls -lah /mnt/nas/media
````

Bestätigt wurde:

* `/mnt/nas/media` war sauber von `192.168.88.32:/volume2/media` eingebunden
* Mediendateien waren auf dem Host sichtbar

### 2. Docker-Bind-Mounts prüfen

Die Container-Mounts von Plex waren korrekt:

```bash
docker inspect plex --format '{{json .Mounts}}'
```

Relevant war insbesondere:

* `/mnt/nas/media` → `/media`

### 3. Medien im Container prüfen

Im Container waren Verzeichnisse und Mediendateien sichtbar:

```bash
docker exec -it plex /bin/sh -c 'find /media/Filme -maxdepth 2 -type f | head -40'
docker exec -it plex /bin/sh -c 'find /media/Serien -maxdepth 3 -type f | head -40'
```

Damit war klar: Der grundsätzliche Datenpfad war vorhanden.

### 4. Tatsächlichen Plex-User prüfen

```bash
docker exec -it plex /bin/sh -c 'ps -ef | grep -i "[p]lex"'
docker exec -it plex /bin/sh -c 'echo "PUID=$PUID PGID=$PGID"; id abc 2>/dev/null || true'
```

Bestätigt wurde:

* Plex läuft als Benutzer `abc`
* `PUID=1026`
* `PGID=100`

### 5. Plex-Logs während des Bibliotheksscans prüfen

Beim Einlesen der Bibliothek protokollierte Plex unter anderem:

```text
ERROR - Couldn't check for the existence of file "/media/Filme": boost::filesystem::status: Permission denied
```

Das war der entscheidende Hinweis.

### 6. Verzeichnisrechte als root vs. Plex-Prozess vergleichen

Im Container:

```bash
docker exec -it plex /bin/sh -c 'ls -ld /media /media/Filme /media/Serien'
docker exec -it plex /bin/sh -c 'su -s /bin/sh abc -c "id && ls -ld /media /media/Filme /media/Serien"'
docker exec -it plex /bin/sh -c 'su -s /bin/sh abc -c "cd /media && pwd && ls"'
```

Auf dem Host:

```bash
ls -ld /mnt/nas/media /mnt/nas/media/Filme /mnt/nas/media/Serien
namei -l /mnt/nas/media/Filme
```

Wichtiger Befund:

```text
d--------- 1 root root ... /mnt/nas/media
```

Die Unterordner `Filme` und `Serien` waren offen, aber der Root des eingebundenen Shares war für den Plex-Prozess nicht traversierbar.

---

## Root Cause 1: Berechtigungsproblem auf dem NFS-Share-Root

Der Plex-Prozess im Container konnte den Wurzelordner des eingebundenen NFS-Shares nicht betreten.

Obwohl `/media/Filme` und `/media/Serien` beim Blick als `root` im Container zugänglich wirkten, konnte der eigentliche Plex-Prozess als Benutzer `abc` nicht einmal sauber in `/media` wechseln. Dadurch scheiterte jeder Bibliotheksscan.

### Fix

Auf der Synology wurde die NFS-Regel für das relevante Client-Subnetz geändert auf:

* **Squash:** `Map all users to admin`

Danach wurde der NFS-Mount auf dem Host neu eingebunden:

```bash
sudo umount /mnt/nas/media
sudo mount /mnt/nas/media
```

Prüfung:

```bash
docker exec -it plex /bin/sh -c 'su -s /bin/sh abc -c "cd /media && pwd && ls | head"'
```

Ergebnis:

* Benutzer `abc` konnte `/media` betreten
* Plex-Bibliotheksscans füllten die Datenbank wieder

---

## Root Cause 2: PKC hing noch am alten Setup

Nachdem das Serverproblem gelöst war, scheiterte die Wiedergabe auf LibreELEC/PKC weiterhin.

Auffälligkeiten auf Clientseite:

* alte Kollektionen aus dem früheren Setup waren weiterhin sichtbar
* PKC hing noch an Altlasten aus dem NAS-basierten Plex-Setup
* der neue Plex-Server wurde nicht automatisch gefunden
* die manuelle HTTPS-Verbindung per IP scheiterte an der SSL-Zertifikatsprüfung

### Tatsächlichen Netzwerkpfad zum neuen Plex-Host prüfen

Vom LibreELEC-Client aus:

```bash
ping -c 3 192.168.88.106
curl -I http://192.168.88.106:32400/identity
curl -k -I https://192.168.88.106:32400/identity
```

Bestätigt wurde:

* der Raspberry Pi konnte den aktuellen Plex-Host erreichen
* Plex auf `192.168.88.106:32400` antwortete korrekt

Damit waren Host-Firewall und grundlegender Netzwerkpfad als Hauptursache vom Tisch.

### Bereinigung auf PKC-/Kodi-Seite

Für einen sauberen Neustart wurde der PKC-/Kodi-Zustand hart zurückgesetzt:

```bash
mkdir -p /storage/backup-pkc-reset
cp -a /storage/.kodi/userdata/addon_data/plugin.video.plexkodiconnect /storage/backup-pkc-reset/ 2>/dev/null || true
cp -a /storage/.kodi/userdata/Database /storage/backup-pkc-reset/Database-$(date +%F-%H%M)

systemctl stop kodi
rm -rf /storage/.kodi/userdata/addon_data/plugin.video.plexkodiconnect
cd /storage/.kodi/userdata/Database
rm -f MyVideos*.db
rm -f /storage/.kodi/userdata/Database/Textures*.db
rm -rf /storage/.kodi/userdata/Thumbnails/*
systemctl start kodi
```

Dadurch wurde die PKC-Ersteinrichtung erzwungen.

### Final funktionierende Verbindung

PKC fand den Server weiterhin nicht automatisch, die manuelle Verbindung funktionierte jedoch mit:

* **Host:** `192.168.88.106`
* **Port:** `32400`
* **Protokoll:** HTTP
* **SSL-Verifikation:** deaktiviert / nicht verwendet

HTTPS per IP scheiterte, weil das Zertifikat nicht sauber zur rohen lokalen IP passte.

---

## Ergebnis / finaler Zustand

### Server

* Plex-Bibliotheksscans funktionieren
* Filmbibliothek füllt sich korrekt
* Serienbibliothek füllt sich korrekt
* Medien werden per NFS von der Synology eingebunden

### Client

* LibreELEC / PKC kann sich mit dem aktuellen Plex-Server verbinden
* Filme lassen sich erfolgreich abspielen
* veralteter PKC-/Kodi-Zustand wurde entfernt

---

## Besonders hilfreiche Befehle im Troubleshooting

### Plex-Host / Docker

```bash
findmnt -t nfs,nfs4,cifs,smb3
docker inspect plex --format '{{json .Mounts}}'
docker exec -it plex /bin/sh -c 'find /media/Filme -maxdepth 2 -type f | head -40'
docker exec -it plex /bin/sh -c 'ps -ef | grep -i "[p]lex"'
docker exec -it plex /bin/sh -c 'tail -n 200 "/config/Library/Application Support/Plex Media Server/Logs/Plex Media Server.log"'
docker exec -it plex /bin/sh -c 'su -s /bin/sh abc -c "cd /media && pwd && ls"'
```

### LibreELEC / Kodi / PKC

```bash
ping -c 3 192.168.88.106
curl -I http://192.168.88.106:32400/identity
curl -k -I https://192.168.88.106:32400/identity
pastekodi
```

---

## Lessons Learned

1. **Der Blick als root im Container ist trügerisch.**
   Nur weil `root` Dateien sehen kann, heißt das noch nicht, dass der eigentliche Plex-Prozess Zugriff hat.

2. **Der Root des Shares ist genauso wichtig wie die Unterordner.**
   Offene Rechte auf `Filme` und `Serien` bringen nichts, wenn `/media` selbst nicht traversierbar ist.

3. **Eine Servermigration hinterlässt leicht Client-Geister.**
   PKC/Kodi schleppte Altlasten aus dem früheren NAS-basierten Plex-Setup mit.

4. **HTTPS per lokaler IP ist fehleranfällig.**
   Im lokalen Netz funktionierte HTTP zuverlässig, während HTTPS per roher IP an der Zertifikatsprüfung scheiterte.

5. **Server- und Client-Troubleshooting trennen.**
   Erst den Bibliotheksscan auf Serverseite sauber herstellen, danach die Wiedergabe auf Clientseite angehen.

---

## Follow-up / mögliche Verbesserungen

* Film- und Serienbenennung weiter normalisieren, um das Matching von Metadaten zu verbessern.
* Periodische Plex-Scans aktiviert lassen, da Netzwerk-Mounts Änderungen nicht immer sofort triggern.
* Die Synology-NFS-Berechtigungen später sauberer modellieren und die grobe `map all users to admin`-Lösung durch ein engeres Rechtekonzept ersetzen.
* Langfristig prüfen, warum PKC den Server nicht automatisch findet, obwohl die manuelle Verbindung funktioniert.

---

## Status

* **Incident gelöst**
* **Wiedergabe auf LibreELEC bestätigt**
* **Plex-Bibliotheken werden korrekt eingelesen**
