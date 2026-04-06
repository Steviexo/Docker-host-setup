# Plex Media Server auf dem HP EliteDesk Docker-Host

## Einordnung

Dieser Dienst stellt eine zentrale Medienbibliothek im Homelab bereit und dient als Backend für lokale Clients wie LibreELEC/Kodi mit PlexKodiConnect.

Die Mediendateien liegen nicht lokal auf dem Host, sondern auf einer Synology-NAS und werden per NFS eingebunden.

---

## Ziel

Der Plex Media Server auf dem HP EliteDesk übernimmt folgende Aufgaben:

- zentrale Verwaltung der Medienbibliotheken
- Bereitstellung von Filmen, Serien und weiteren Medien für lokale Clients
- Trennung von Laufzeitumgebung (Docker-Host) und Storage (NAS)
- Nutzung eines einheitlichen Medien-Backends für verschiedene Wiedergabegeräte

---

## Architektur

- Plex läuft als Docker-Container auf dem HP EliteDesk.
- Die Mediendateien liegen auf einer Synology-NAS.
- Der Medien-Share des NAS wird per NFS auf dem Host eingebunden.
- Der Plex-Container erhält diesen Mount als Bind-Mount.
- Ein Raspberry Pi 5 mit LibreELEC/Kodi nutzt PlexKodiConnect als Client.

---

## Pfade

### Host

- Plex-Konfiguration: `/opt/docker/plex/config`
- Plex-Transcode: `/opt/docker/plex/transcode`
- NFS-Mount des Medien-Shares: `/mnt/nas/media`

### Container

- Plex-Konfiguration: `/config`
- Plex-Transcode: `/transcode`
- Medien: `/media`

---

## Docker-Mount-Mapping

- `/opt/docker/plex/config` → `/config`
- `/mnt/nas/media` → `/media`
- `/opt/docker/plex/transcode` → `/transcode`

---

## Storage-Anbindung

Die Mediendateien liegen auf einer Synology-NAS und werden per NFSv3 auf dem Host eingebunden.

### Relevante Punkte

- NAS-Share: `/volume2/media`
- Host-Mount: `/mnt/nas/media`
- Container-Mount: `/media`

Wichtig ist, dass nicht nur die Medien-Unterordner, sondern auch der Root des eingebundenen Shares für den tatsächlichen Plex-Prozess traversierbar ist.

---

## Client-Anbindung

### LibreELEC / PKC

Aktuell funktionierendes Setup:

- manuelle Serververbindung
- Host: `192.168.88.106`
- Port: `32400`
- lokale HTTP-Verbindung funktioniert
- HTTPS per roher IP wird nicht verwendet, da die Zertifikatsprüfung scheitert

---

## Betriebshinweise

- Wenn Plex-Bibliotheken leer bleiben, immer als tatsächlicher Laufzeit-User prüfen, nicht nur als `root`.
- Bei NFS-Problemen zuerst den Root des Shares und die effektiven Zugriffsrechte prüfen.
- Nach Migrationen oder Umbauten können auf Clientseite Altlasten in PKC/Kodi verbleiben.
- Für netzwerkbasierten Storage sind periodische Bibliotheksscans sinnvoll, da Änderungen nicht immer sofort erkannt werden.

---

## Verwandte Dokumentation

- [`docs/incidents/plex-bibliothek-leer-und-libreelec-pkc-wiedergabefehler.md`](docs/incidents/plex-bibliothek-leer-und-libreelec-pkc-wiedergabefehler.md)
- [`docs/troubleshooting/plex-libreelec-pkc.md`](docs/troubleshooting/plex-libreelec-pkc.md)
