# Samba-Freigabe vom HP EliteDesk zum Ubuntu-iMac

## Ziel

Vom Hauptarbeitsrechner (iMac mit Ubuntu Desktop) auf ausgewählte Dateien des HP EliteDesk (Ubuntu Server / Docker-Host) zugreifen, um Konfigurationen und Daten einfacher zu durchsuchen und bei Bedarf herunterzuladen.

## Ergebnis

Die Samba-Freigabe funktioniert dauerhaft und reboot-fest.

Freigegeben werden über einen zentralen Share auf dem HP:

* `docker` → Inhalt von `/opt/docker`
* `paperless` → Inhalt von `/opt/paperless`

Der Zugriff vom Ubuntu-iMac erfolgt per SMB auf den Share `HPUbuntuShare`.

---

## Ausgangslage

Auf dem HP wurde zunächst ein Samba-Share unter folgendem Pfad eingerichtet:

* `/srv/samba/share`

Danach wurden symbolische Links in diesen Share gelegt, um auf die eigentlichen Verzeichnisse unter `/opt/...` zu zeigen.

Beispielhaft:

* Symlink von `/srv/samba/share/docker` nach `/opt/docker`
* Symlink von `/srv/samba/share/paperless` nach `/opt/paperless`

### Symptom

Die SMB-Verbindung vom Ubuntu-iMac ließ sich zwar aufbauen, der Share erschien aber leer oder zeigte keine nutzbaren Inhalte.

Auch ein lokaler Test auf dem HP mit `smbclient` zeigte zunächst nur:

* `.`
* `..`

also effektiv einen leeren Share.

---

## Ursache

Die Ursache war **nicht** der Ubuntu-iMac, **nicht** GNOME/GVFS und am Ende auch **nicht** das SMB-Passwort.

Die eigentliche Ursache war die Konstruktion mit **Symlinks innerhalb des Shares**, die auf Pfade **außerhalb** des Share-Verzeichnisses zeigten.

Diese Variante ist mit Samba unnötig fehleranfällig und führt leicht zu verwirrendem Verhalten:

* Share wirkt leer
* Samba liefert nicht zuverlässig den Zielinhalt aus
* zusätzliche Optionen wie `wide links`, `follow symlinks` oder ähnliche Workarounds machen die Konfiguration eher fragiler als besser

---

## Saubere Lösung

Statt Symlinks wurden **Bind-Mounts** verwendet.

Dadurch liegen die Zielverzeichnisse für Samba effektiv direkt innerhalb des Share-Pfads.

### Verwendete Bind-Mounts

* `/opt/docker` → `/srv/samba/share/docker`
* `/opt/paperless` → `/srv/samba/share/paperless`

### Vorteile

* Samba sieht echte Verzeichnisse innerhalb des Shares
* deutlich robuster als Symlinks
* keine Sonderkonfiguration für externe Links nötig
* nach Neustarts sauber reproduzierbar

---

## Umsetzung auf dem HP EliteDesk

### 1. Samba-Benutzer

Es wurde ein eigener Samba-Benutzer angelegt:

* Benutzer: `sambastevie`

### 2. Share-Verzeichnis

Zentrales Share-Verzeichnis:

```bash
sudo mkdir -p /srv/samba/share
sudo chown sambastevie:sambastevie /srv/samba/share
```

### 3. Bind-Mounts setzen

```bash
sudo mkdir -p /srv/samba/share/docker
sudo mkdir -p /srv/samba/share/paperless

sudo mount --bind /opt/docker /srv/samba/share/docker
sudo mount --bind /opt/paperless /srv/samba/share/paperless
```

### 4. Dauerhaft per `/etc/fstab`

Einträge in `/etc/fstab`:

```fstab
/opt/docker     /srv/samba/share/docker     none    bind    0 0
/opt/paperless  /srv/samba/share/paperless  none    bind    0 0
```

Test:

```bash
sudo mount -a
mount | grep '/srv/samba/share'
```

---

## Finale Samba-Konfiguration

### Relevanter Share in `/etc/samba/smb.conf`

```ini
[HPUbuntuShare]
   comment = Shared files from HP
   path = /srv/samba/share
   browseable = yes
   read only = no
   guest ok = no
   valid users = sambastevie
   create mask = 0664
   directory mask = 0775
```

### Zusätzliche sinnvolle globale Härtung

Im globalen Bereich:

```ini
load printers = no
usershare allow guests = no
usershare max shares = 0
```

### Bewusst nicht verwendet

Folgende Optionen wurden am Ende **nicht** benötigt und sollten in diesem Setup nicht als Lösungsweg verwendet werden:

```ini
wide links = yes
follow symlinks = yes
unix extensions = no
allow insecure wide links = yes
```

---

## Funktionstests

### Samba-Konfiguration prüfen

```bash
sudo testparm
```

### Samba-Dienst neu starten

```bash
sudo systemctl restart smbd
```

### Lokaler Funktionstest auf dem HP

```bash
sudo smbclient //localhost/HPUbuntuShare -U sambastevie -c 'ls'
```

Erwartetes Ergebnis:

* `docker`
* `paperless`

### Reboot-Test

Nach einem Neustart des HP wurde geprüft:

```bash
mount | grep '/srv/samba/share'
sudo smbclient //localhost/HPUbuntuShare -U sambastevie -c 'ls'
```

Ergebnis:

* Bind-Mounts weiterhin vorhanden
* `docker` und `paperless` weiterhin sichtbar
* Zugriff vom Ubuntu-iMac weiterhin funktionsfähig

---

## Zugriff vom Ubuntu-iMac

Im Dateimanager kann der Server per SMB angebunden werden, z. B. über:

```text
smb://<IP-des-HP-EliteDesk>/HPUbuntuShare
```

Nach erfolgreicher Anmeldung mit dem Samba-Benutzer `sambastevie` sind die Verzeichnisse `docker` und `paperless` sichtbar.

---

## Wichtige Betriebsnotizen

### 1. Wenn ein Ordner sichtbar, aber nicht schreibbar ist

Dann ist das in der Regel **kein Samba-Problem**, sondern ein normales Linux-Rechteproblem im Zielverzeichnis unter `/opt/...`.

Dann gezielt prüfen:

```bash
ls -ld <zielpfad>
ls -l <zielpfad>
```

### 2. Warum keine Symlink-Lösung?

Die Symlink-Variante kann mit Samba zwar theoretisch zum Laufen gebracht werden, ist hier aber unnötig kompliziert und fehleranfällig. Bind-Mounts sind für diesen Anwendungsfall deutlich robuster.

### 3. Warum eigener Samba-Benutzer?

Ein separater Samba-Benutzer hält den Zugriff klar getrennt und macht das Setup nachvollziehbarer als eine Vermischung mit beliebigen anderen Benutzerkonten.

---

## Kurzfazit

Die stabile Lösung bestand nicht in zusätzlichen Samba-Sonderoptionen, sondern in einer strukturell sauberen Freigabe:

* zentraler Share unter `/srv/samba/share`
* Bind-Mounts auf die tatsächlich benötigten Verzeichnisse
* einfache, klare `smb.conf`
* erfolgreicher Funktionstest lokal und vom Ubuntu-iMac
* erfolgreicher Persistenztest nach Reboot

Damit ist die SMB-Freigabe vom HP EliteDesk auf den Ubuntu-iMac zuverlässig einsatzbereit.

---

## Nützliche Befehle für spätere Diagnose

```bash
sudo testparm
sudo systemctl status smbd
sudo smbclient //localhost/HPUbuntuShare -U sambastevie -c 'ls'
mount | grep '/srv/samba/share'
ls -la /srv/samba/share
id sambastevie
```
