# Troubleshooting: Plex + LibreELEC + PlexKodiConnect

## Symptom: Plex-Bibliothek bleibt leer

### Host-Mount prüfen

```bash
findmnt -t nfs,nfs4,cifs,smb3
ls -lah /mnt/nas/media
````

### Docker-Bind-Mounts prüfen

```bash
docker inspect plex --format '{{json .Mounts}}'
```

### Tatsächliche Medien im Container prüfen

```bash
docker exec -it plex /bin/sh -c 'find /media/Filme -maxdepth 2 -type f | head -40'
```

### Zugriff als Plex-Laufzeit-User prüfen

```bash
docker exec -it plex /bin/sh -c 'su -s /bin/sh abc -c "cd /media && pwd && ls"'
```

### Plex-Logs prüfen

```bash
docker exec -it plex /bin/sh -c 'tail -n 200 "/config/Library/Application Support/Plex Media Server/Logs/Plex Media Server.log"'
```

Wenn dort `Permission denied` für `/media/...` auftaucht, den Root des NFS-Shares und dessen Berechtigungen auf dem NAS prüfen.

---

## Symptom: PKC-Wiedergabe schlägt fehl / NAS offline / alte Bibliotheksreste sichtbar

### Erreichbarkeit des aktuellen Plex-Hosts von LibreELEC aus prüfen

```bash
ping -c 3 192.168.88.106
curl -I http://192.168.88.106:32400/identity
```

### Kodi-Log hochladen

```bash
pastekodi
```

### PKC-/Kodi-Zustand hart zurücksetzen

```bash
systemctl stop kodi
rm -rf /storage/.kodi/userdata/addon_data/plugin.video.plexkodiconnect
cd /storage/.kodi/userdata/Database
rm -f MyVideos*.db
rm -f Textures*.db
rm -rf /storage/.kodi/userdata/Thumbnails/*
systemctl start kodi
```

### PKC manuell neu konfigurieren

In diesem Setup funktionierte am Ende:

* Host: `192.168.88.106`
* Port: `32400`
* lokales HTTP
* keine HTTPS-/IP-basierte SSL-Verifikation

---

## Symptom: HTTPS-Verbindung per manueller IP schlägt fehl

Wenn die manuelle Verbindung per IP mit SSL-Fehlern scheitert, zunächst lokal per HTTP testen.

Grund:

* Zertifikate validieren gegen rohe lokale IPs oft nicht sauber
* im vertrauenswürdigen lokalen Netz ist HTTP hier ausreichend pragmatisch
