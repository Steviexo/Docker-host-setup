# Plex/Plexamp: Server extern wieder erreichbar, lokaler WLAN-Zugriff durch VPN-Lockdown eingeschränkt

## Kontext

Nach Änderungen im Heimnetz trat ein Fehlerbild auf, bei dem **Plexamp auf dem Handy weder lokal im WLAN noch extern sauber auf die Bibliothek zugreifen konnte**. Parallel funktionierten andere selbst gehostete Dienste wie **Home Assistant** und **Grocy** weiterhin lokal und extern.

Das Problem wirkte zunächst wie ein möglicher Effekt des Heimnetzumbaus, stellte sich bei der Analyse aber als **Kombination aus einem Plex-spezifischen Veröffentlichungsproblem und einer lokalen Einschränkung durch die Handy-Sicherheitskonfiguration** heraus.

## Betroffene Umgebung

* **Docker-Host:** HP EliteDesk 800 G5 Mini
* **Host-IP:** `192.168.88.106`
* **Plex-Container:** LinuxServer.io Image
* **Plex-Port:** `32400/tcp`
* **Handy:** GrapheneOS
* **VPN-App:** Proton VPN
* **Relevante Handy-Einstellungen:**

  * Always-on VPN: aktiv
  * Block connections without VPN: aktiv
  * LAN-Verbindungen in Proton: aktiv

## Fehlerbild

### Beobachtungen

* **Plexamp** zeigte auf dem Handy sinngemäß: *„Wir konnten keine Daten von deinem Server erhalten.“*
* Die **externe Plex-URL** war im Browser erreichbar.
* `http://HOST-IP:32400/web` war auf dem Docker-Host lokal erreichbar.
* Im Handy-Browser war `http://HOST-IP:32400/web` im WLAN zunächst **nicht** erreichbar.
* In Plex Web wurde **Fernzugriff als verfügbar** angezeigt.
* Die normale **Plex-App** auf dem Handy konnte sich zwar anmelden, zeigte aber zunächst **keine verfügbaren Bibliotheken**.

## Analyse

### 1. Plex selbst lief lokal sauber

Die grundlegenden Serverprüfungen waren unauffällig:

* Plex-Container lief
* Port `32400` war auf dem Host gebunden
* lokale Plex-API lieferte alle Bibliotheken korrekt aus

Per lokaler Abfrage über `127.0.0.1:32400` wurden die Bibliotheken korrekt geliefert:

* Filme
* Serien
* Musik
* Andere Videos

Zusätzlich war der Server korrekt mit dem Plex-Konto verknüpft:

* `PlexOnlineToken` vorhanden
* `PlexOnlineUsername` korrekt
* `PlexOnlineMail` korrekt
* `MachineIdentifier` und `ProcessedMachineIdentifier` vorhanden

Damit war klar:

* **Die Bibliotheken waren nicht weg**
* **Der Server war nicht unclaimed oder vom Konto getrennt**

### 2. Plex veröffentlichte zunächst nur eine unbrauchbare Docker-interne Verbindung

Die entscheidende Analyse der über plex.tv publizierten Ressourcen zeigte:

* Der Plex-Server wurde cloudseitig grundsätzlich gesehen.
* Als nutzbare Verbindung war jedoch zunächst nur eine **Docker-interne Adresse** veröffentlicht:

```text
https://172-17-0-3....plex.direct:32400
```

Diese Adresse verweist auf die **Bridge-IP des Containers** und ist für normale Clients außerhalb des Containers nicht sinnvoll nutzbar.

Folge:

* Die Apps konnten den Server grundsätzlich sehen,
* aber keine brauchbare Verbindung zu den Bibliotheken aufbauen.

### 3. Zusätzlich blockierte das Handy den lokalen Direktzugriff im WLAN

Ein weiterer Test zeigte:

* Mit aktivem **Block connections without VPN** war `http://HOST-IP:32400/web` im WLAN auf dem Handy **nicht** erreichbar.
* Nach Deaktivierung von **Block connections without VPN** war dieselbe URL im WLAN **sofort erreichbar**.

Damit war klar:

* Der **lokale Direktzugriff** vom Handy wurde durch die Kombination aus **GrapheneOS-Lockdown + Proton VPN** blockiert.
* Das erklärte die lokalen WLAN-Probleme unabhängig vom externen Plex-Verhalten.

## Ursache

Das Gesamtproblem bestand aus **zwei getrennten Ursachen**:

### Ursache A: Falsche Server-Veröffentlichung durch Plex

Plex veröffentlichte gegenüber plex.tv zunächst nur die interne Docker-Verbindung statt einer für Clients brauchbaren Adresse.

### Ursache B: Lokaler WLAN-Zugriff vom Handy wurde durch VPN-Lockdown blockiert

Die Kombination aus:

* Always-on VPN
* Block connections without VPN
* aktivem Proton VPN

blockierte lokale Direktverbindungen zu Selfhosted-Diensten im Heimnetz.

## Lösung

### 1. Externe Server-URL in Plex hinterlegt

In Plex Web unter

**Einstellungen → Server → Netzwerk → Eigene URL für den Zugriff auf diesen Server**

wurde die externe Plex-URL hinterlegt, sinngemäß:

```text
https://plex.example.tld
```

Anschließend wurde der Plex-Container neu gestartet.

### 2. Ergebnis nach dem Fix

Danach veröffentlichte Plex in den Ressourcen zusätzlich eine brauchbare externe Verbindung:

```text
https://plex.example.tld
```

Zusätzlich war weiterhin die interne Docker-Verbindung sichtbar, aber nun stand den Clients endlich auch eine funktionierende externe Route zur Verfügung.

### 3. Funktionstest

Nach dem Eintrag der externen URL funktionierte im **Mobilfunk** wieder:

* Plexamp mit Zugriff auf die Musikbibliothek
* Plex-App mit Anzeige der Bibliotheken

Damit war bestätigt:

* **Die Serverseite war repariert**
* **Die externe Client-Anbindung funktionierte wieder**

## Offener Restpunkt

### Lokaler WLAN-Zugriff bleibt von der Handy-Konfiguration abhängig

Solange auf dem Handy aktiv ist:

* Always-on VPN
* Block connections without VPN

kann der lokale Direktzugriff auf Plex und andere Selfhosted-Dienste im Heimnetz weiterhin eingeschränkt sein.

Das ist **kein verbleibender Plex-Server-Fehler**, sondern eine direkte Folge der Sicherheitskonfiguration auf dem Handy.

## Entscheidung für den Betrieb

### Kurzfristig

Bis der Heimnetzumbau abgeschlossen ist, wird der **praktische Mittelweg** verwendet:

* Proton VPN aktiv lassen
* LAN-Verbindungen in Proton aktiviert lassen
* `Block connections without VPN` deaktivieren

Ziel:

* Internetverkehr weiterhin möglichst über VPN führen
* lokale Selfhosted-Dienste im Heimnetz aber nicht unnötig blockieren

### Mittelfristig

Als sauberere Zielarchitektur ist ein **eigenes Heim-VPN** vorgesehen, um künftig besser zwischen:

* abgesichertem externem Zugriff
* und stabiler lokaler Nutzung selbst gehosteter Dienste

zu trennen.

## Lessons Learned

* **Fernzugriff grün in Plex** bedeutet nicht automatisch, dass Apps den Server auch sinnvoll erreichen.
* Bei Plex in Docker sollte geprüft werden, **welche Verbindungen der Server tatsächlich gegenüber plex.tv veröffentlicht**.
* Wenn Plex-Clients Bibliotheken nicht sehen, obwohl Plex Web lokal funktioniert, lohnt sich der Blick auf die **Ressourcen-/Connection-Ausgabe**.
* Bei GrapheneOS/Android kann **Block connections without VPN** lokale Selfhosted-Dienste im Heimnetz ebenfalls blockieren.
* In so einem Fehlerbild können **Serverproblem und Client-Sicherheitskonfiguration gleichzeitig** beteiligt sein.

## Kurzfassung

* **Symptom:** Plexamp und Plex-App sahen keine Bibliotheken
* **Serverstatus:** Plex lief, Bibliotheken lokal vorhanden, Konto-Verknüpfung korrekt
* **Hauptproblem:** Plex veröffentlichte zunächst nur die interne Docker-Adresse
* **Zusatzproblem:** Lokaler WLAN-Zugriff vom Handy wurde durch VPN-Lockdown blockiert
* **Fix:** Externe URL in Plex unter „Eigene URL für den Zugriff auf diesen Server“ eintragen und Container neu starten
* **Ergebnis:** Externer Zugriff per Mobilfunk wieder funktionsfähig; lokaler WLAN-Zugriff hängt weiterhin von der Handy-Konfiguration ab
