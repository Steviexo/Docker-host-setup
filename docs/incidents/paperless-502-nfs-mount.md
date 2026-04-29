# Incident: Paperless-ngx 502 Bad Gateway durch fehlgeschlagenen NAS-NFS-Mount

**Datum:** 2026-04-29
**Status:** Gelöst
**Host:** HP EliteDesk / Ubuntu Server
**Betroffener Dienst:** Paperless-ngx
**Betroffene URL:** Paperless-ngx über Nginx Proxy Manager
**NAS:** Synology `192.168.88.32`
**Mount:** `/mnt/nas/docker` → `192.168.88.32:/volume1/docker`

---

## Kurzfassung

Paperless-ngx war im Browser nicht erreichbar und zeigte über Nginx Proxy Manager einen **502 Bad Gateway**. In Portainer war der Container `paperless` seit mehreren Tagen gestoppt:

```text
Status: stopped / exited
ExitCode: 128
```

Die unmittelbare Docker-Fehlermeldung lautete:

```text
error while creating mount source path '/mnt/nas/docker/paperless-ngx/data': mkdir /mnt/nas/docker/paperless-ngx: permission denied
```

Der eigentliche Fehler lag nicht in Paperless selbst und auch nicht in Nginx Proxy Manager, sondern im darunterliegenden NAS-Mount:

```text
/mnt/nas/docker
```

Dieser Mount war fehlgeschlagen. Dadurch war `/mnt/nas/docker` nicht mehr der NAS-Pfad, sondern nur noch ein lokaler Schattenordner auf dem HP.

---

## Auswirkungen

* Paperless-ngx war über den Browser nicht erreichbar.
* Nginx Proxy Manager zeigte **502 Bad Gateway**, weil der Upstream-Dienst nicht lief.
* Der Container `paperless` konnte nicht gestartet werden.
* Docker meldete einen Fehler beim Zugriff auf den Host-Bind-Mount.
* Andere Hilfscontainer wie `paperless-db`, `paperless-broker`, `paperless-gotenberg` und `paperless-tika` liefen teilweise weiter.

---

## Symptome

### Browser

```text
502 Bad Gateway
```

### Portainer / Docker

```text
paperless  Exited (128)
```

### Docker Inspect

```text
Error=error while creating mount source path '/mnt/nas/docker/paperless-ngx/data': mkdir /mnt/nas/docker/paperless-ngx: permission denied
```

### Mount-Status

```bash
systemctl status mnt-nas-docker.mount --no-pager -l
```

Ergebnis:

```text
mnt-nas-docker.mount - /mnt/nas/docker
Active: failed
What: 192.168.88.32:/volume1/docker
mount.nfs4: Protocol not supported for 192.168.88.32:/volume1/docker on /mnt/nas/docker
```

---

## Root Cause

In `/etc/fstab` war der NAS-Docker-Mount als **NFSv4.1** eingetragen:

```text
192.168.88.32:/volume1/docker  /mnt/nas/docker  nfs4  rw,vers=4.1,noatime,_netdev,x-systemd.automount,x-systemd.requires=network-online.target,nofail  0 0
```

Der Synology-Export `/volume1/docker` funktionierte in dieser Umgebung aber zuverlässig mit **NFSv3**, nicht mit `nfs4`/`vers=4.1`.

Dadurch konnte systemd nach einem Unmount/Reboot den Mount nicht wiederherstellen:

```text
mount.nfs4: Protocol not supported
```

Die Fehlerkette war:

```text
NFSv4.1-Mount schlägt fehl
→ /mnt/nas/docker ist nicht das NAS, sondern lokaler Schattenordner
→ Docker findet/erstellt Paperless-Bind-Mounts am falschen Ort
→ paperless startet nicht, ExitCode 128
→ NPM erreicht keinen Upstream
→ Browser zeigt 502 Bad Gateway
```

---

## Diagnose

### Mounts unter `/mnt/nas` prüfen

```bash
findmnt -R /mnt/nas
```

Auffällig war: `/mnt/nas/media` war korrekt gemountet, `/mnt/nas/docker` aber nicht.

### fstab prüfen

```bash
grep -nE 'nas|192\.168\.88\.32|paperless|docker|media|volume' /etc/fstab
```

Relevanter fehlerhafter Eintrag:

```text
192.168.88.32:/volume1/docker  /mnt/nas/docker  nfs4  rw,vers=4.1,noatime,_netdev,x-systemd.automount,x-systemd.requires=network-online.target,nofail  0 0
```

### systemd-Mounts prüfen

```bash
systemctl list-units --type=mount --type=automount | grep -Ei 'mnt|nas|media|docker|paperless'
```

Auffällig:

```text
mnt-nas-docker.mount loaded failed failed /mnt/nas/docker
```

### Mount-Fehler anzeigen

```bash
systemctl status mnt-nas-docker.mount --no-pager -l
journalctl -u mnt-nas-docker.mount --no-pager -n 80
```

Ergebnis:

```text
mount.nfs4: Protocol not supported
```

### NFSv3-Testmount

Vor Änderung der fstab wurde der NAS-Pfad testweise per NFSv3 auf einen temporären Mountpoint gemountet:

```bash
sudo mkdir -p /mnt/nas-docker-test
sudo mount -t nfs -o vers=3,rw,noatime 192.168.88.32:/volume1/docker /mnt/nas-docker-test
findmnt /mnt/nas-docker-test
ls -lah /mnt/nas-docker-test
ls -lah /mnt/nas-docker-test/paperless-ngx
```

Das funktionierte erfolgreich. Damit war klar: Der Export ist erreichbar, aber nicht mit der bisherigen NFSv4.1-Konfiguration.

---

## Sicherheitsmaßnahme vor der Reparatur

Da während des fehlgeschlagenen Mounts lokale Inhalte unter `/mnt/nas/docker` sichtbar waren, wurde der lokale Schattenordner zuerst gesichert.

```bash
TS=$(date +%Y%m%d-%H%M%S)
RECOVERY="/opt/recovery/local-mnt-nas-docker-shadow-$TS"
sudo mkdir -p "$RECOVERY"
sudo rsync -aHAX --numeric-ids /mnt/nas/docker/ "$RECOVERY/"
sudo du -sh "$RECOVERY"
```

Erstellte Sicherung:

```text
/opt/recovery/local-mnt-nas-docker-shadow-20260429-091252
```

Größe:

```text
394M
```

Diese Sicherung wurde bewusst nicht zurückkopiert und nicht gelöscht.

---

## Fix

### fstab sichern

```bash
sudo cp -a /etc/fstab "/etc/fstab.backup.$(date +%Y%m%d-%H%M%S)"
```

### fstab-Eintrag von NFSv4.1 auf NFSv3 ändern

Alter Eintrag:

```text
192.168.88.32:/volume1/docker  /mnt/nas/docker  nfs4  rw,vers=4.1,noatime,_netdev,x-systemd.automount,x-systemd.requires=network-online.target,nofail  0 0
```

Neuer Eintrag:

```text
192.168.88.32:/volume1/docker /mnt/nas/docker nfs rw,vers=3,noatime,_netdev,x-systemd.automount,x-systemd.requires=network-online.target,nofail 0 0
```

### systemd neu laden und Mount neu starten

```bash
sudo systemctl daemon-reload
sudo systemctl stop mnt-nas-docker.automount mnt-nas-docker.mount 2>/dev/null || true
sudo systemctl reset-failed mnt-nas-docker.mount mnt-nas-docker.automount
sudo mount -v /mnt/nas/docker
```

### Mount prüfen

```bash
findmnt /mnt/nas/docker
```

Erwartetes Ergebnis:

```text
/mnt/nas/docker  192.168.88.32:/volume1/docker  nfs  rw,noatime,vers=3,...
```

### NAS-Inhalt prüfen

```bash
ls -lah /mnt/nas/docker | head -50
```

Erwartete NAS-Struktur:

```text
docker_storage
eBesucher
paperless-ngx
portainer
portainer_data
vikunja
```

---

## Paperless wieder starten

Da der alte Container bereits existierte, wurde nicht sofort der gesamte Compose-Stack neu erzeugt, sondern der vorhandene Paperless-Container direkt gestartet:

```bash
docker start paperless
```

Statusprüfung:

```bash
docker inspect paperless --format 'Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}} ExitCode={{.State.ExitCode}} Error={{.State.Error}}'
```

Ergebnis:

```text
Status=running Health=healthy ExitCode=0 Error=
```

Lokaler HTTP-Test:

```bash
curl -I --max-time 10 http://127.0.0.1:8000
```

Ergebnis:

```text
HTTP/1.1 302 Found
Location: /login/?next=/
```

Das ist korrekt: Paperless antwortet lokal und leitet zur Login-Seite weiter.

---

## Abschlusszustand

### fstab

```text
192.168.88.32:/volume1/docker /mnt/nas/docker nfs rw,vers=3,noatime,_netdev,x-systemd.automount,x-systemd.requires=network-online.target,nofail 0 0
```

### Mount

```text
/mnt/nas/docker → 192.168.88.32:/volume1/docker, nfs vers=3
```

### Paperless

```text
Status=running
Health=healthy
ExitCode=0
Error=
```

### Browser

Paperless ist wieder erreichbar.

---

## Offene Punkte / Nacharbeiten

### 1. Recovery-Ordner vorerst behalten

Nicht löschen:

```text
/opt/recovery/local-mnt-nas-docker-shadow-20260429-091252
```

Empfehlung: Frühestens nach 14 Tagen löschen, wenn Paperless stabil läuft und keine Daten fehlen.

Vor dem Löschen nochmal prüfen:

```bash
sudo du -sh /opt/recovery/local-mnt-nas-docker-shadow-20260429-091252
sudo find /opt/recovery/local-mnt-nas-docker-shadow-20260429-091252 -maxdepth 2 -type d | sort
```

### 2. Docker-/Compose-Altlasten separat prüfen

Beim Startversuch per Compose gab es Namenskonflikte:

```text
Conflict. The container name "/paperless-db-backup" is already in use
Conflict. The container name "/paperless-tika" is already in use
```

Außerdem auffällig:

```text
paperless-redis     Created
paperless-broker    Up / healthy
vikunja-db          Exited (128)
```

Das ist nicht akut, sollte aber separat bereinigt werden. Dabei nicht blind Container löschen, sondern erst Compose-Dateien, Container-Namen, Netzwerke und Volumes prüfen.

Empfohlene spätere Diagnose:

```bash
cd /mnt/nas/docker/paperless-ngx

docker compose config

docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Labels}}' | grep -Ei 'paperless|redis|postgres|tika|gotenberg|db'

docker inspect paperless --format '{{json .Config.Labels}}'
docker inspect paperless-db --format '{{json .Config.Labels}}'
docker inspect paperless-broker --format '{{json .Config.Labels}}'
docker inspect paperless-tika --format '{{json .Config.Labels}}'
docker inspect paperless-gotenberg --format '{{json .Config.Labels}}'
```

Ziel der Nacharbeit:

* Compose-Projektzugehörigkeit klären
* doppelte/alte Container identifizieren
* `paperless-redis Created` einordnen
* Namenskonflikte auflösen
* sicherstellen, dass `docker compose up -d` künftig wieder sauber funktioniert

---

## Lessons Learned

* Ein `502 Bad Gateway` bei Reverse Proxy Setups bedeutet oft nicht, dass der Proxy kaputt ist. Häufig läuft der Upstream-Dienst nicht.
* Bei Docker-Bind-Mount-Fehlern zuerst den Host-Pfad prüfen, nicht direkt den Container neu bauen.
* Wenn ein NAS-Mount ausfällt, kann unter demselben Pfad ein lokaler Schattenordner sichtbar werden. Das ist gefährlich, weil Container dann scheinbar auf dem richtigen Pfad arbeiten, tatsächlich aber lokal schreiben.
* Vor dem Reparieren eines Mounts lokale Schattenordner sichern.
* Nach Änderungen an `/etc/fstab` immer testen:

```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt -R /mnt/nas
```

* Für diesen Synology-Mount ist aktuell NFSv3 die funktionierende Variante.

---

## Schneller Prüfblock für Wiederholungsfall

```bash
echo "### Mount prüfen"
findmnt /mnt/nas/docker

echo
 echo "### fstab prüfen"
grep -n '192.168.88.32:/volume1/docker' /etc/fstab

echo
 echo "### systemd Mountstatus"
systemctl status mnt-nas-docker.mount --no-pager -l

echo
 echo "### Paperless Status"
docker inspect paperless --format 'Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}} ExitCode={{.State.ExitCode}} Error={{.State.Error}}' 2>&1

echo
 echo "### Paperless lokal erreichbar?"
curl -I --max-time 10 http://127.0.0.1:8000 2>&1 || true
```

Erwartung bei gesundem Zustand:

```text
/mnt/nas/docker ist als nfs vers=3 gemountet
paperless läuft und ist healthy
curl liefert HTTP 302 zur Login-Seite
```
