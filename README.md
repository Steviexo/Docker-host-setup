# 📖 Docker-host-setup

Dokumentation meines HP EliteDesk 800 G5 Mini als Docker-Host im Homelab.

## 📝 Ziel dieses Repositories

✅ **Dokumentation der Host-Struktur, der darauf betriebenen Container und Services sowie relevanter Incidents, Troubleshooting-Pfade und Lessons Learned aus dem laufenden Betrieb**\
✅ **Strukturierte Anleitungen für zukünftige Anpassungen & Erweiterungen**
✅ **Hilfestellung für andere, die ähnliche Setups umsetzen möchten**

Der Fokus liegt auf nachvollziehbarer technischer Dokumentation: nicht nur der Soll-Zustand, sondern auch Fehlerbilder, Analysewege und belastbare Lösungen sollen festgehalten werden.

Dieses Repository dokumentiert:

- den Aufbau und Betrieb des Docker-Hosts
- darauf laufende Container und Services
- Anbindungen an NAS, lokale Clients und weitere Homelab-Komponenten
- technische Entscheidungen rund um Storage, Netz, Laufzeitumgebung und Service-Betrieb
- Incidents, Troubleshooting und Lessons Learned

---

## Systemrolle im Homelab

Der HP EliteDesk übernimmt die Rolle eines zentralen Docker-Hosts im Homelab.
Auf ihm laufen ausgewählte Dienste, die entweder direkt lokal genutzt oder in weitere Teile der Infrastruktur eingebunden werden.

Je nach Service bestehen Anbindungen an weitere Systeme, z. B.:

- Synology-NAS als Storage
- lokale Clients im Heimnetz
- Reverse Proxy / Webzugriffe für einzelne Dienste
- weitere Homelab-Services und Infrastruktur-Bausteine

---

## 📂 Struktur des Repositories

```text
Docker-host-setup/
└── docs/
    ├── services/
    ├── incidents/
    └── troubleshooting/
````

### Bereiche

* `docs/services/` – Beschreibung einzelner Dienste, ihrer Architektur, Pfade und Betriebslogik
* `docs/incidents/` – konkrete Vorfälle, Fehleranalysen, Ursachen und Fixes
* `docs/troubleshooting/` – wiederverwendbare Prüf- und Debug-Schritte

---

## Dokumentierte Services

### Plex Media Server

Der HP EliteDesk dient unter anderem als Host für den lokalen Plex Media Server.
Die Mediendateien liegen auf der Synology-NAS und werden per NFS auf dem Host eingebunden. Plex läuft in einem Container und stellt die Bibliotheken für lokale Clients wie LibreELEC/Kodi mit PlexKodiConnect bereit.

**Relevante Dokumentation**

* [`docs/services/plex-media-server.md`](docs/services/plex-media-server.md) – Beschreibung des Dienstes, der Architektur und der aktuell funktionierenden Anbindung
* [`docs/incidents/plex-bibliothek-leer-und-libreelec-pkc-wiedergabefehler.md`](docs/incidents/plex-bibliothek-leer-und-libreelec-pkc-wiedergabefehler.md) – Incident-Doku zum Fehlerbild „Bibliothek leer / PKC-Wiedergabefehler“
* [`docs/troubleshooting/plex-libreelec-pkc.md`](docs/troubleshooting/plex-libreelec-pkc.md) – wiederverwendbare Prüf- und Debug-Schritte für Plex, LibreELEC und PKC

**Kurzüberblick zum Setup**

* **Plex-Host:** HP EliteDesk (`192.168.88.106`)
* **Storage:** Synology-NAS per NFS
* **Host-Mount:** `/mnt/nas/media`
* **Container-Mount:** `/media`
* **Client:** Raspberry Pi 5 mit LibreELEC/Kodi + PlexKodiConnect

**Wichtige Hinweise**

* Der Root des NFS-Shares muss für den tatsächlichen Plex-Prozess traversierbar sein.
* Der Blick als `root` im Container reicht für die Fehlersuche nicht aus, wenn Plex selbst als anderer Benutzer läuft.
* Nach einer Migration des Plex-Servers können auf Clientseite Altlasten in PKC/Kodi zurückbleiben.
* Die aktuell funktionierende PKC-Anbindung erfolgt manuell über `192.168.88.106:32400` per lokalem HTTP.

---

## Incidents und Troubleshooting

Ein wesentlicher Teil dieses Repositories ist die nachvollziehbare Dokumentation realer Fehlerbilder.
Der Anspruch ist nicht nur, funktionierende Setups zu beschreiben, sondern auch Störungen, Analysewege und belastbare Lösungen festzuhalten.

### Beispielhafte Themen

* Dienste laufen, liefern aber funktional keine Ergebnisse
* Storage-/Mount-Probleme zwischen Host, NAS und Container
* Berechtigungsprobleme zwischen Host, Container und Service-Prozess
* Client-Probleme nach Migrationen oder Umbauten
* reproduzierbare Prüfpfade zur Eingrenzung von Fehlerursachen

---

## Arbeitsweise in diesem Repository

Die Dokumentation orientiert sich an folgenden Grundsätzen:

* technische Nachvollziehbarkeit vor Selbstdarstellung
* klare Trennung von Soll-Zustand, Incident und Troubleshooting
* konkrete Prüfungen und Befehle statt rein abstrakter Beschreibungen
* Lessons Learned nicht weglassen, sondern bewusst dokumentieren

---

## Einordnung

Dieses Repository ergänzt weitere Homelab-Repositories, z. B. für Netzwerk, NAS oder andere Infrastruktur-Bausteine.
Während andere Repositories stärker auf Netzstruktur oder einzelne Plattformen fokussieren, liegt der Schwerpunkt hier auf dem Docker-Host als Laufzeitumgebung für produktiv genutzte Services im Homelab.

---

## Status

Dieses Repository wird fortlaufend erweitert.
Bestehende Dienste, neue Services, Umbauten und relevante Incidents werden nach und nach dokumentiert.
