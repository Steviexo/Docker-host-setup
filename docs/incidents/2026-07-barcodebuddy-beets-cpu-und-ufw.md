# Incident: BarcodeBuddy-Verbindungsfehler, Beets-Dauerlast und blockiertes Webinterface

## Einordnung

Dieser Incident betrifft mehrere Beobachtungen auf dem HP EliteDesk 800 G5 Mini als Docker-Host:

- BarcodeBuddy konnte zeitweise keine Verbindung zur Grocy-API herstellen.
- Bei neuen Barcodes blieben die Schaltflächen **Add** und **Consume** trotz Produktauswahl deaktiviert.
- Bei der Fehlersuche fiel eine dauerhafte CPU-Last von ungefähr 100 % beim Container `music-beets` auf.
- Nach Aktivierung des Beets-Web-Plug-ins war das Webinterface lokal erreichbar, wurde aus dem Heimnetz jedoch durch UFW blockiert.

Die Probleme traten zeitlich nahe beieinander auf. Ein direkter technischer Zusammenhang zwischen der Beets-Dauerlast und den BarcodeBuddy-/Grocy-Verbindungsfehlern konnte jedoch nicht nachgewiesen werden.

---

## Kurzfassung

BarcodeBuddy protokollierte wiederholt:

```text
Could not connect to Grocy server:
Could not lookup Grocy barcodes
Please check your URL and API key in the settings menu!
```

Nach einem Neustart des BarcodeBuddy-Containers funktionierte die Grocy-Produktzuordnung wieder. Sobald in der Produktauswahl ein Grocy-Produkt gewählt wurde, wurden **Add** und **Consume** wieder aktiv.

Bei der anschließenden Prüfung der Systemlast zeigte `docker stats`, dass der Container `music-beets` dauerhaft ungefähr einen vollständigen CPU-Kern belegte. Die Logs enthielten fortlaufend:

```text
error: unknown command 'web'
```

Das LinuxServer-Beets-Image startet den Beets-Webdienst. In der Beets-Konfiguration war das dafür benötigte Plug-in `web` jedoch nicht aktiviert. Der Dienst beendete sich deshalb sofort und wurde durch den s6-Service-Manager wieder gestartet. Dieser schnelle Neustart-Loop verursachte die Dauerlast.

Nach Ergänzung des Plug-ins startete Beets korrekt und die CPU-Last sank von ungefähr 100 % auf etwa 0,02 %.

Das Webinterface war anschließend auf dem Host erreichbar, aus dem LAN aber nicht. Ursache war eine fehlende UFW-Regel für TCP-Port 8337. Nach Freigabe des Ports für das lokale Subnetz war das Interface im Browser erreichbar.

---

## Umgebung

### Docker-Host

- **Host:** HP EliteDesk 800 G5 Mini
- **Betriebssystem:** Ubuntu Server
- **Rolle:** zentraler Docker-Host im Homelab
- **Host-IP:** `192.168.88.106`
- **Lokales Subnetz:** `192.168.88.0/24`
- **Host-Firewall:** UFW aktiv

### Betroffene Container

- **BarcodeBuddy**
- **Grocy**
- **Beets-Container:** `music-beets`
- **Beets-Image:** LinuxServer.io
- **Beets-Version:** `2.8.0`
- **LinuxServer.io-Version:** `2.8.0-ls321`
- **Netzwerkmodus von Beets:** `host`
- **Beets-Web-Port:** `8337/tcp`

### Ursprünglich aktivierte Beets-Plug-ins

```yaml
plugins: musicbrainz lyrics embedart fromfilename duplicates
```

Das Plug-in `web` fehlte.

---

## Symptome

### BarcodeBuddy

Bei Barcodes, die BarcodeBuddy noch nicht kannte:

- Die Grocy-Produktauswahl war sichtbar.
- Ein passendes Grocy-Produkt konnte gewählt werden.
- Die Schaltflächen **Add** und **Consume** blieben jedoch ausgegraut.
- In den Wochen zuvor waren wiederholt Grocy-Verbindungsfehler aufgetreten.

Relevante Logmeldung:

```text
Could not connect to Grocy server:
Could not lookup Grocy barcodes
Please check your URL and API key in the settings menu!
```

### Docker-Host

Der Lüfter des EliteDesk drehte zeitweise hörbar hoch.

`docker stats` zeigte für Beets:

```text
CONTAINER      CPU %
music-beets    ~100 %
```

Beispiel:

```text
music-beets    101.94%
```

Dies entsprach ungefähr der vollständigen Auslastung eines logischen CPU-Kerns.

### Beets

In den Logs erschien wiederholt:

```text
error: unknown command 'web'
```

Der eigentliche `beet`-Prozess war mit `docker top` nicht zuverlässig sichtbar, weil er jeweils nur sehr kurz lief und sofort wieder beendet wurde.

### Beets-Webinterface

Nach Aktivierung des Plug-ins startete der Webdienst erfolgreich:

```text
* Serving Flask app 'beetsplug.web'
* Debug mode: off
* Running on all addresses (0.0.0.0)
* Running on http://127.0.0.1:8337
* Running on http://192.168.88.106:8337
Connection to localhost (127.0.0.1) 8337 port [tcp/*] succeeded!
[ls.io-init] done.
```

Trotzdem endete der Zugriff von einem anderen Gerät im LAN zunächst mit:

```text
ERR_CONNECTION_TIMED_OUT
```

---

## Analyse

### 1. BarcodeBuddy neu starten

Da die Fehlermeldung auf eine unterbrochene oder festhängende Verbindung zur Grocy-API hindeutete, wurde zunächst nur BarcodeBuddy neu gestartet:

```bash
docker restart barcodebuddy
```

Danach wurde die Weboberfläche vollständig neu geladen.

#### Ergebnis

- Grocy-Produkte konnten wieder gewählt werden.
- **Add** und **Consume** wurden nach Auswahl eines Produkts wieder blau und anklickbar.
- URL und API-Key mussten nicht geändert werden.

Damit war bestätigt, dass die gespeicherte Grundkonfiguration weiterhin funktionierte. Die genaue Ursache der vorausgegangenen wiederkehrenden Grocy-Verbindungsfehler blieb jedoch offen.

---

### 2. Containerlast prüfen

Zur Untersuchung des hochdrehenden Lüfters wurde die aktuelle Containerlast geprüft:

```bash
docker stats
```

Auffällig war insbesondere:

```text
music-beets    ~100 % CPU
```

Für eine einmalige Momentaufnahme:

```bash
docker stats --no-stream
```

---

### 3. Prozesse im Beets-Container prüfen

Der Prozessbaum wurde einschließlich Laufzeit, CPU-Last und vollständigem Befehl angezeigt:

```bash
docker top music-beets -eo pid,ppid,etime,pcpu,pmem,args
```

Sichtbar waren hauptsächlich:

- `s6-svscan`
- `s6-supervise`
- `crond`
- weitere s6-Hilfsprozesse

Ein dauerhaft laufender `beet`-Prozess war nicht zu sehen.

#### Erklärung

Der fehlerhafte Prozess lief jeweils nur kurz:

1. s6 startete den Beets-Dienst.
2. Der Container führte `beet web` aus.
3. Beets beendete sich sofort mit `unknown command 'web'`.
4. s6 startete den Dienst erneut.
5. Die Schleife wiederholte sich sehr schnell.

`docker top` konnte den kurzlebigen Prozess leicht verfehlen. Die aufsummierte CPU-Last blieb in `docker stats` dennoch sichtbar.

---

### 4. Container-Startbefehl prüfen

```bash
docker inspect --format 'Path: {{.Path}} Args: {{json .Args}}' music-beets
```

Ausgabe:

```text
Path: /init Args: []
```

Das LinuxServer-Image verwendet `/init` und den integrierten s6-Service-Manager. Der eigentliche Dienststart erfolgt innerhalb der Image-Servicekonfiguration und ist deshalb nicht direkt in `.Path` oder `.Args` als `beet web` sichtbar.

---

### 5. Logs auswerten

```bash
docker logs --since 2h --tail 300 music-beets
```

Entscheidender Befund:

```text
error: unknown command 'web'
```

Damit war klar, dass der Container zwar den Webdienst starten wollte, Beets den Unterbefehl `web` aber nicht kannte.

---

## Root Cause 1: Fehlendes Beets-Web-Plug-in

Der Beets-Befehl `beet web` steht nur zur Verfügung, wenn das Plug-in `web` aktiviert ist.

Die bestehende Konfiguration enthielt:

```yaml
plugins: musicbrainz lyrics embedart fromfilename duplicates
```

Da `web` fehlte, scheiterte jeder Startversuch des Dienstes.

Der s6-Service-Manager interpretierte das Prozessende als Dienstabbruch und startete ihn erneut. Dadurch entstand ein permanenter Neustart-Loop mit ungefähr 100 % CPU-Last auf einem logischen Kern.

### Fix

Die Plug-in-Zeile wurde ergänzt:

```yaml
plugins: musicbrainz lyrics embedart fromfilename duplicates web
```

Anschließend wurde der Container neu gestartet:

```bash
docker restart music-beets
```

### Prüfung

```bash
docker exec music-beets beet version
```

Ergebnis:

```text
beets version 2.8.0
Python version 3.12.12
plugins: duplicates, embedart, fromfilename, lyrics, musicbrainz, web
```

Die CPU-Last wurde erneut geprüft:

```bash
docker stats music-beets
```

Ergebnis:

```text
CPU ~0.02 %
```

Der Neustart-Loop war damit beendet.

---

## Beets-Webinterface für Netzwerkzugriff konfigurieren

Ohne abweichende Konfiguration bindet das Beets-Web-Plug-in standardmäßig nur an `127.0.0.1`.

Für den Zugriff über das Heimnetz wurde folgender Block ergänzt:

```yaml
web:
  host: 0.0.0.0
  port: 8337
  readonly: true
```

Danach:

```bash
docker restart music-beets
```

Die Logs bestätigten anschließend:

```text
Running on all addresses (0.0.0.0)
Running on http://127.0.0.1:8337
Running on http://192.168.88.106:8337
```

`readonly: true` verhindert schreibende API-Operationen über das Web-Plug-in und ist für eine reine Bibliotheksansicht die sinnvollere Voreinstellung.

---

## Docker-Netzwerkmodus einordnen

Der Netzwerkmodus wurde geprüft:

```bash
docker inspect -f '{{.HostConfig.NetworkMode}}' music-beets
```

Ergebnis:

```text
host
```

Bei `network_mode: host` teilt der Container den Netzwerk-Namespace des Docker-Hosts.

Daraus folgen zwei wichtige Punkte:

1. Ein separater `ports:`-Block beziehungsweise `-p 8337:8337` ist nicht erforderlich.
2. `docker port music-beets` liefert erwartungsgemäß keine Ausgabe.

Die fehlende Ausgabe von:

```bash
docker port music-beets
```

war daher kein Fehler und kein Hinweis auf eine fehlende Docker-Portfreigabe.

### Bewertung des Host-Modus

Der Host-Modus funktioniert für Beets, ist aber für einen einzelnen Webport nicht zwingend notwendig. Ein benutzerdefiniertes Bridge-Netz mit gezielter Portfreigabe wäre stärker isoliert:

```yaml
ports:
  - "8337:8337"
```

Für den aktuellen Incident bestand jedoch kein akuter Grund, den funktionierenden Netzwerkmodus zu ändern.

---

## Lokale Erreichbarkeit prüfen

Auf dem Docker-Host wurden beide Adressen getestet:

```bash
curl -v http://127.0.0.1:8337
curl -v http://192.168.88.106:8337
```

Beide Aufrufe lieferten:

```text
HTTP/1.1 200 OK
```

Zusätzlich wurde geprüft, ob der Dienst auf allen IPv4-Interfaces lauscht:

```bash
sudo ss -ltnp | grep ':8337'
```

Ergebnis sinngemäß:

```text
LISTEN 0 ... 0.0.0.0:8337
```

Damit waren folgende Punkte bestätigt:

- Das Beets-Web-Plug-in lief.
- Der Webdienst lauschte auf dem Host.
- Port 8337 war lokal erreichbar.
- Docker beziehungsweise der Host-Netzwerkmodus war nicht die Fehlerursache.

Da der Zugriff von anderen LAN-Geräten dennoch in einen Timeout lief, blieb als nächster Prüfpunkt die Host-Firewall.

---

## Root Cause 2: UFW blockierte TCP-Port 8337

UFW war aktiv, enthielt aber noch keine Regel für das Beets-Webinterface.

Statusprüfung:

```bash
sudo ufw status
```

Da lokale `curl`-Aufrufe erfolgreich waren, der Zugriff von einem anderen Gerät jedoch mit einem Timeout endete, blockierte UFW den eingehenden LAN-Verkehr.

### Fix

Port 8337 wurde ausschließlich für das lokale Subnetz freigegeben:

```bash
sudo ufw allow from 192.168.88.0/24 to any port 8337 proto tcp
```

Prüfung:

```bash
sudo ufw status numbered
```

### Ergebnis

Das Webinterface war anschließend im Browser erreichbar:

```text
http://192.168.88.106:8337
```

Die Freigabe ist auf das lokale Subnetz begrenzt. Port 8337 wurde nicht pauschal für beliebige Quellnetze geöffnet.

---

## Status von BarcodeBuddy

Der BarcodeBuddy-Teil des Incidents wurde durch einen Containerneustart behoben:

```bash
docker restart barcodebuddy
```

Danach funktionierte die Zuordnung neuer Barcodes zu Grocy-Produkten wieder.

### Bestätigt

- Produktliste in BarcodeBuddy verfügbar
- Grocy-Produkt auswählbar
- **Add** und **Consume** werden nach der Auswahl aktiviert
- vorhandene Grocy-URL und vorhandener API-Key weiterhin gültig

### Nicht abschließend geklärt

Die wiederkehrenden Meldungen zur fehlgeschlagenen Grocy-Verbindung wurden nicht weiter bis zur eigentlichen Netzwerk- oder API-Ursache zurückverfolgt.

Die hohe CPU-Last durch Beets könnte den Host zusätzlich belastet haben. Sie ist aber kein ausreichender Beleg dafür, dass sie die Grocy-Verbindungsabbrüche verursacht hat.

Bei erneutem Auftreten sollten BarcodeBuddy- und Grocy-Logs zeitgleich mit Netzwerk- und Ressourcenwerten erfasst werden.

---

## Ergebnis / finaler Zustand

### BarcodeBuddy

- BarcodeBuddy kann Grocy-Produkte wieder abrufen.
- Neue Barcodes können wieder Produkten zugeordnet werden.
- **Add** und **Consume** funktionieren wieder.
- Kein API-Key-Wechsel erforderlich.

### Beets

- Das Plug-in `web` ist aktiviert.
- `beet web` startet erfolgreich.
- Kein s6-Neustart-Loop mehr.
- CPU-Last im Leerlauf ungefähr `0,02 %`.
- Geladene Plug-ins wurden mit `beet version` bestätigt.
- Das Webinterface läuft schreibgeschützt auf Port 8337.

### Netzwerk und Firewall

- Beets läuft weiterhin im Host-Netzwerkmodus.
- Der Dienst lauscht auf `0.0.0.0:8337`.
- Lokale HTTP-Aufrufe liefern `200 OK`.
- UFW erlaubt TCP-Port 8337 ausschließlich aus `192.168.88.0/24`.
- Das Webinterface ist im Heimnetz unter `http://192.168.88.106:8337` erreichbar.

---

## Besonders hilfreiche Befehle

### Containerlast

```bash
docker stats
docker stats --no-stream
docker stats music-beets
```

### Prozesse und Laufzeit

```bash
docker top music-beets -eo pid,ppid,etime,pcpu,pmem,args
```

### Container-Start und Netzwerkmodus

```bash
docker inspect --format 'Path: {{.Path}} Args: {{json .Args}}' music-beets
docker inspect -f '{{.HostConfig.NetworkMode}}' music-beets
docker port music-beets
```

Hinweis: Bei `network_mode: host` ist eine leere Ausgabe von `docker port` normal.

### Logs

```bash
docker logs --since 2h --tail 300 music-beets
docker logs -f --tail 50 music-beets
```

### Beets-Konfiguration und Plug-ins

```bash
docker exec music-beets beet version
docker exec music-beets beet config -p
```

### Netzwerkdienst prüfen

```bash
curl -v http://127.0.0.1:8337
curl -v http://192.168.88.106:8337
sudo ss -ltnp | grep ':8337'
```

### UFW prüfen und freigeben

```bash
sudo ufw status
sudo ufw status numbered
sudo ufw allow from 192.168.88.0/24 to any port 8337 proto tcp
```

### BarcodeBuddy neu starten

```bash
docker restart barcodebuddy
```

---

## Diagnosemuster für ähnliche Fälle

### Hohe Container-CPU, aber kein Prozess in `docker top`

Mögliche Erklärung:

- Der fehlerhafte Prozess ist sehr kurzlebig.
- Ein Supervisor startet ihn nach jedem Abbruch sofort neu.
- `docker top` zeigt nur eine Momentaufnahme.
- `docker stats` erfasst trotzdem die fortlaufende CPU-Nutzung.

Prüfpfad:

```bash
docker stats CONTAINER
docker logs --tail 300 CONTAINER
docker top CONTAINER -eo pid,ppid,etime,pcpu,pmem,args
```

### Dienst lokal erreichbar, im LAN aber Timeout

Typischer Prüfpfad:

```bash
curl -v http://127.0.0.1:PORT
curl -v http://HOST-IP:PORT
sudo ss -ltnp | grep ':PORT'
sudo ufw status numbered
```

Interpretation:

- **Lokaler Zugriff schlägt fehl:** Dienst oder Bind-Adresse prüfen.
- **Nur `127.0.0.1` funktioniert:** Dienst bindet nicht an das LAN-Interface.
- **Beide lokalen Aufrufe funktionieren, LAN-Zugriff aber nicht:** Host-Firewall oder Netzsegmentierung prüfen.
- **HTTP-Antwort lokal und UFW aktiv:** gezielte UFW-Regel für das benötigte Quellnetz ergänzen.

---

## Lessons Learned

- Eine dauerhaft hohe CPU-Last kann durch sehr schnelle Prozessneustarts entstehen, ohne dass der problematische Prozess in einer einzelnen Prozessaufnahme sichtbar ist.
- Containerlogs sind bei Supervisor-basierten Images oft aussagekräftiger als eine einzelne `docker top`-Momentaufnahme.
- Ein einzelnes fehlendes Beets-Plug-in kann den vom Image erwarteten Startbefehl vollständig unbrauchbar machen.
- Nach Konfigurationsänderungen sollte nicht nur geprüft werden, ob der Container läuft, sondern auch, ob der erwartete Dienst dauerhaft läuft.
- `docker port` ist im Host-Netzwerkmodus kein geeigneter Verfügbarkeitstest.
- `HTTP 200` auf Host-IP plus Timeout aus dem LAN ist ein starkes Indiz für eine Firewallregel.
- Neue lokal erreichbare Webdienste benötigen auf einem Host mit aktiver UFW fast immer eine bewusst gesetzte Freigabe.
- Firewallregeln sollten möglichst auf das notwendige lokale Subnetz beschränkt werden.
- Zeitgleich beobachtete Probleme sind nicht automatisch kausal miteinander verbunden.
- Ein Containerneustart kann einen Dienst wiederherstellen, ersetzt aber keine Ursachenanalyse, wenn der Fehler wiederkehrt.

---

## Follow-up

- BarcodeBuddy- und Grocy-Logs bei einem erneuten Verbindungsfehler zeitgleich sichern.
- Prüfen, ob BarcodeBuddy und Grocy über stabile interne Docker-DNS-Namen kommunizieren.
- Optional bewerten, ob Beets langfristig von `network_mode: host` auf ein eigenes Bridge-Netz mit expliziter Portfreigabe umgestellt werden soll.
- UFW-Regeln bei neuen Webdiensten direkt als Teil der jeweiligen Service-Dokumentation festhalten.
- Optional Temperatur- und CPU-Verlauf des EliteDesk beobachten, um zukünftige Lastspitzen schneller einem Container zuzuordnen.

---

## Referenzen

- [LinuxServer.io: Beets Docker Image](https://docs.linuxserver.io/images/docker-beets/)
- [Beets-Dokumentation: Web Plugin](https://beets.readthedocs.io/en/stable/plugins/web.html)
- [Docker-Dokumentation: Host network driver](https://docs.docker.com/engine/network/drivers/host/)
- [Docker-Dokumentation: Port publishing and mapping](https://docs.docker.com/engine/network/port-publishing/)
- [Ubuntu Manpages: UFW framework](https://manpages.ubuntu.com/)

---

## Status

- **BarcodeBuddy-Funktion wiederhergestellt**
- **Beets-Neustart-Loop behoben**
- **CPU-Dauerlast beseitigt**
- **Beets-Webinterface im LAN erreichbar**
- **UFW-Regel auf lokales Subnetz begrenzt**
- **Incident gelöst**
