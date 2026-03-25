# Plex Media Server auf dem HP EliteDesk Docker-Host

## Ziel

Dieser Dienst stellt eine zentrale Medienbibliothek und Wiedergabeplattform für lokale Clients über Plex bereit.

## Architektur

- Plex läuft als Docker-Container auf dem HP EliteDesk.
- Die Mediendateien liegen auf einer Synology-NAS.
- Der Medien-Share des NAS wird per NFS auf dem Host eingebunden.
- Der Plex-Container erhält diesen Mount als Bind-Mount.
- Ein Raspberry Pi 5 mit LibreELEC/Kodi nutzt PlexKodiConnect als Client.

## Pfade

### Host

- Plex-Konfiguration: `/opt/docker/plex/config`
- Plex-Transcode: `/opt/docker/plex/transcode`
- NFS-Mount des Medien-Shares: `/mnt/nas/media`

### Container

- Plex-Konfiguration: `/config`
- Plex-Transcode: `/transcode`
- Medien: `/media`

## Docker-Mount-Mapping

- `/opt/docker/plex/config` → `/config`
- `/mnt/nas/media` → `/media`
- `/opt/docker/plex/transcode` → `/transcode`

## Client-Anbindung

### LibreELEC / PKC

Aktuell funktionierendes Setup:

- manuelle Serververbindung
- Host: `192.168.88.106`
- Port: `32400`
- lokale HTTP-Verbindung funktioniert
- HTTPS per roher IP wird nicht verwendet, da die Zertifikatsprüfung scheitert

## Wichtige Hinweise

- Der Root des Synology-NFS-Shares muss für den Plex-Prozess traversierbar sein.
- Wenn Plex-Bibliotheken leer bleiben, immer als tatsächlicher Laufzeit-User prüfen, nicht nur als `root`.
- Falls LibreELEC/PKC nach einer Migration auf einen neuen Plex-Host Probleme macht, gezielt nach Altlasten in PKC/Kodi suchen.
