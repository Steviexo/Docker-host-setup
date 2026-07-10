# Systemupdate (07.07.2026) – Root-Cause-Analyse &amp; Recovery

---

## 📌 TL;DR

Ein **Systemupdate auf dem Docker-Host** (Ubuntu Server) führte am **07.07.2026** zum Ausfall mehrerer Container-Dienste. Ursache waren:

- **AppArmor-Profil-Konflikte** (Home Assistant + Bluetooth).
- **Berechtigungsprobleme** bei Volumes (Mosquitto, Grafana Plugins).
- **Manuell gestoppte Container** (PostgreSQL, InfluxDB) durch das Update.

**Betroffene Dienste:**  
Home Assistant, zigbee2mqtt (Mosquitto), NetBox (PostgreSQL), InfluxDB, Grafana.

**Lösungszeit:** \~2 Stunden (schrittweise Fehlersuche nach der Methode: Beobachtung → Hypothese → Test → Ergebnis).

---

## 🔍 Kontext

### Systemumgebung


| Komponente             | Details                                                                                  |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| **Host**               | HP EliteDesk 800 G5 Mini, Ubuntu Server (Docker-Host)                                    |
| **Docker-Version**     | `24.x` (exakt: siehe `docker --version`)                                                 |
| **Betroffenes Update** | Systemupdate (Kernel + Pakete) am **07.07.2026**, \~12:00 UTC.                           |
| **Docker-Compose**     | Services: Home Assistant, Mosquitto, PostgreSQL, InfluxDB, Grafana, NetBox, zigbee2mqtt. |


### Auslöser

- **Automatisches Systemupdate** (`unattended-upgrades`) führte zu:
  - **Neustart des Hosts** (oder Docker-Daemon).
  - **Zurücksetzen von AppArmor-Profilen** (blockierte Bluetooth-Zugriff in Home Assistant).
  - **Berechtigungsänderungen** bei Docker-Volumes (Mosquitto-Logs, Grafana-Plugins).

---

## ⚠️ Symptome

### 1. Home Assistant

- **Status:** `Exited (137)` (durch SIGKILL getötet).
- **Fehler:**
  ```
  bluetooth.bluez: Failed to connect to dbus: org.freedesktop.DBus.Error.AccessDenied
  ```
- **Ursache:** AppArmor blockierte den Zugriff auf `/dev/rfkill` und DBus.

### 2. zigbee2mqtt

- **Status:** `Exited (1)` (Verbindungsfehler zu MQTT).
- **Fehler:**
  ```
  Error: Connection refused: Not connected (ECONNREFUSED)
  ```
- **Ursache:** Mosquitto-Container war gestoppt (Berechtigungsproblem bei `/mosquitto/log/`).

### 3. NetBox

- **Status:** `netbox-netbox-worker-1` im `Restarting`-Loop, `netbox-postgres-1` mit `Exited (0)`.
- **Fehler:** Keine expliziten Fehler in den Logs, aber PostgreSQL nicht erreichbar.
- **Ursache:** PostgreSQL-Container wurde durch das Update gestoppt und startete nicht automatisch neu.

### 4. InfluxDB

- **Status:** `Exited (2)` seit 2 Tagen.
- **Fehler:** Keine Fehler in den Logs, aber Container stoppte nach \~13 Minuten Laufzeit.
- **Ursache:** Manueller Stop (vermutlich durch `docker stop` während des Updates).

### 5. Grafana

- **Status:** `Created` (nie gestartet).
- **Fehler:**
  ```
  Failed to install plugin elasticsearch/zipkin: unlinkat /usr/share/grafana/data/plugins-bundled/...: permission denied
  ```
- **Ursache:** Plugin-Verzeichnisse gehörten `root:root` oder UID 1000, Grafana läuft als UID 472.

---

## 🔬 Root-Cause-Analyse

### 1. AppArmor (Home Assistant + Bluetooth)

- **Beobachtung:** Home Assistant konnte nicht auf Bluetooth-Geräte zugreifen.
- **Hypothese:** AppArmor-Profil blockiert den Zugriff auf `/dev/rfkill` und DBus.
- **Test:**
  ```bash
  docker inspect homeassistant | grep -i apparmor
  ```
  → Ausgabe: \\\`"AppArmorProfile": "docker-default"\\\` (Standard-Profil nach Update).
- **Ergebnis:** Das Update setzte AppArmor-Profile zurück, was den Zugriff auf Host-Geräte blockierte.
- **Lösung:** `apparmor=unconfined` für den Home Assistant-Container in `docker-compose.yml`.

### 2. Mosquitto (zigbee2mqtt)

- **Beobachtung:** Mosquitto-Container startet nicht (`Exited (0)`).
- **Hypothese:** Fehlendes Log-Verzeichnis oder falsche Berechtigungen.
- **Test:**
  ```bash
  ls -la /mosquitto/log/
  ```
  → Ausgabe: \\\`No such file or directory\\\`.
- **Ergebnis:** Das Verzeichnis `/mosquitto/log/` existierte nicht, Mosquitto konnte keine Log-Datei anlegen.
- **Lösung:**
  ```bash
  sudo mkdir -p /mosquitto/log
  sudo chown -R root:root /mosquitto
  docker start mosquitto
  ```

### 3. PostgreSQL (NetBox)

- **Beobachtung:** PostgreSQL-Container war gestoppt (`Exited (0)`).
- **Hypothese:** Manueller Stop während des Updates.
- **Test:**
  ```bash
  docker inspect netbox-postgres-1 | grep -i restart
  ```
  → Ausgabe: \\\`"RestartPolicy": {"Name": "unless-stopped"}\\\`.
- **Ergebnis:** Der Container wurde manuell gestoppt und startete nicht automatisch neu (trotz `unless-stopped`).
- **Lösung:** `docker start netbox-postgres-1`.

### 4. InfluxDB

- **Beobachtung:** InfluxDB-Container stoppte nach 13 Minuten Laufzeit (`Exited (2)`).
- **Hypothese:** Manueller Stop oder externes Signal.
- **Test:**
  ```bash
  docker inspect influxdb | grep -i oom
  ```
  → Ausgabe: \\\`"OOMKilled": false\\\`.
- **Ergebnis:** Kein Speichermangel, daher manueller Stop während des Updates.
- **Lösung:** `docker start influxdb`.

### 5. Grafana (Plugin-Berechtigungen)

- **Beobachtung:** Plugin-Updates scheiterten mit `permission denied`.
- **Hypothese:** Falsche Berechtigungen für Plugin-Verzeichnisse.
- **Test:**
  ```bash
  docker exec -it grafana ls -la /usr/share/grafana/data/plugins-bundled/
  ```
  → Ausgabe: Verzeichnisse gehörten \\\`root:root\\\` oder UID 1000, Grafana läuft als UID 472.
- **Ergebnis:** Grafana hatte keine Schreibrechte auf die Plugin-Dateien.
- **Lösung:** Plugin-Updates deaktiviert (Option 1), da die Dateien im Docker-Image `read-only` sind.

---

## ✅ Lösungen

### 1. Home Assistant (AppArmor + Bluetooth)

**Problem:** AppArmor blockierte den Zugriff auf `/dev/rfkill` und DBus.

**Lösung:**  
In der `docker-compose.yml` für Home Assistant:

```yaml
services:
  homeassistant:
    # ...
    security_opt:
      - apparmor:unconfined
```

**Risiko:** AppArmor schützt nicht mehr vor schädlichen Aktionen des Containers.  
**Alternative:** Ein benutzerdefiniertes AppArmor-Profil erstellen (langfristig sauberer).

---

### 2. Mosquitto (Fehlendes Log-Verzeichnis)

**Problem:** `/mosquitto/log/` existierte nicht.

**Lösung:**

```bash
# Verzeichnis erstellen
sudo mkdir -p /mosquitto/log

# Berechtigungen für root (Mosquitto läuft als root im Container)
sudo chown -R root:root /mosquitto

# Container neu starten
docker start mosquitto
```

---

### 3. PostgreSQL (NetBox)

**Problem:** Container wurde gestoppt und startete nicht neu.

**Lösung:**

```bash
# Container manuell starten
docker start netbox-postgres-1
```

**Hinweis:** Die `RestartPolicy: unless-stopped` sollte den Container automatisch neu starten. Falls nicht, prüfe:

- Docker-Daemon-Logs: `journalctl -u docker --no-pager | grep -i error`.
- Manuell gestoppte Container: `docker inspect netbox-postgres-1 | grep -i status`.

---

### 4. InfluxDB

**Problem:** Container wurde gestoppt.

**Lösung:**

```bash
# Container neu starten
docker start influxdb
```

---

### 5. Grafana (Plugin-Berechtigungen)

**Problem:** Plugin-Updates scheiterten wegen `permission denied`.

**Lösung (Option 1 – Plugin-Updates deaktivieren):**

1. `grafana.ini` bearbeiten (Host-Pfad: `/opt/docker/grafana/config/grafana.ini`):
  ```ini
   [plugin_installation]
   disable_plugin_installation = true
  ```
2. Container neu starten:
  ```bash
   docker restart grafana
  ```

**Alternative (Option 2 – Neues Plugin-Volume):**

1. In `docker-compose.yml` ein neues Volume für Plugins hinzufügen:
  ```yaml
   grafana:
     # ...
     volumes:
       - /opt/docker/grafana/plugins:/var/lib/grafana/plugins
     user: "472:472"
  ```
2. Berechtigungen setzen:
  ```bash
   sudo mkdir -p /opt/docker/grafana/plugins
   sudo chown -R 472:472 /opt/docker/grafana/plugins
  ```
3. Container neu starten:
  ```bash
   docker-compose up -d grafana
  ```

---

## 📚 Lessons Learned

### 1. Systemupdates &amp; Docker

- **Problem:** Systemupdates können Docker-Container, AppArmor-Profile und Berechtigungen beeinflussen.
- **Lösung:**
  - **Vor dem Update:** Alle Container stoppen (`docker-compose down`).
  - **Nach dem Update:** Container manuell prüfen (`docker ps -a`).
  - **AppArmor:** Bei Hardware-Zugriff (Bluetooth, USB) immer `apparmor:unconfined` oder benutzerdefinierte Profile verwenden.

### 2. Berechtigungen in Containern

- **Problem:** Docker-Volumes behalten ihre Berechtigungen auch nach Updates.
- **Lösung:**
  - **Immer explizit Berechtigungen setzen** (z. B. `chown -R <UID>:<GID> /pfad`).
  - **UID/GID des Containers prüfen:** `docker exec -it <container> id`.

### 3. Restart-Policies

- **Problem:** `unless-stopped` startet Container nicht neu, wenn sie manuell gestoppt wurden.
- **Lösung:**
  - Für kritische Dienste (z. B. Datenbanken) **`restart: always`** verwenden.
  - Manuell gestoppte Container nach Updates **immer prüfen**.

### 4. Plugin-Management in Grafana

- **Problem:** Plugin-Updates scheitern, wenn Berechtigungen nicht passen.
- **Lösung:**
  - **Option 1:** Plugin-Updates deaktivieren (stabiler).
  - **Option 2:** Eigenes Volume für Plugins mit korrekten Berechtigungen.

### 5. Fehlersuche-Methode

- **Beobachtung → Hypothese → Test → Ergebnis → neue Hypothese** (erfolgreich angewendet).
- **Ein Problem nach dem anderen lösen** (kein Shotgun-Troubleshooting).

---

## 🔗 Verwandte Dateien

- [Docker-Compose-Konfiguration](https://github.com/Steviexo/Docker-host-setup/blob/main/docker-compose.yml)
- [Netzwerk-Topologie](https://github.com/Steviexo/network-setup)

---

## 📅 Zeitplan


| Datum      | Aktion                                                                   |
| ---------- | ------------------------------------------------------------------------ |
| 05.07.2026 | Systemupdate → Ausfall mehrerer Container.                               |
| 06.07.2026 | Fehlersuche (AppArmor, Berechtigungen, Container-Status).                |
| 07.07.2026 | Lösungen umgesetzt (AppArmor, Mosquitto, PostgreSQL, InfluxDB, Grafana). |
| 10.07.2026 | Dokumentation erstellt.                                                  |


---

## ✏️ Offene Punkte

- [ ] **AppArmor:** Langfristig ein benutzerdefiniertes Profil für Home Assistant erstellen.
- [ ] **Backup-Strategie:** Regelmäßige Backups der Docker-Volumes (z. B. mit `docker-volume-backup`).
- [ ] **Monitoring:** Healthchecks für alle Container einrichten (z. B. mit `docker healthcheck`).

---

## 📝 Changelog


| Version | Datum      | Autor       | Änderungen                                            |
| ------- | ---------- | ----------- | ----------------------------------------------------- |
| 1.0     | 10.07.2026 | Stevie | Initiales Dokument nach Systemupdate-Fehler erstellt. |
