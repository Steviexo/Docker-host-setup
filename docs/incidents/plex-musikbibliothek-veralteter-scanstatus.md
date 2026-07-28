Plex-Musikalben nur in der Ordneransicht sichtbar und nicht abspielbar

Kurzfassung

Zwei neu importierte Ace-of-Base-Alben waren in Plex nur über die Ordneransicht sichtbar. Sie erschienen weder unter Alben noch unter Interpreten. Beim Start eines Titels meldete Plex:

Wiedergabefehler: Es gab einen Fehler beim Laden der Medien.

Die Dateien waren vorhanden, plausibel groß und für den Plex-Benutzer lesbar. Plex hatte jedoch nur noch veraltete Verzeichniseinträge gespeichert und keine gültigen Trackobjekte für die FLAC-Dateien angelegt.

Die Lösung bestand darin, die betroffenen Albumordner vorübergehend aus dem Bibliothekspfad zu verschieben, die Musikbibliothek zu scannen, nach Prüfung des NAS-Mounts den Papierkorb der Musikbibliothek zu leeren, die Alben zurückzuverschieben und erneut zu scannen.

Danach erschienen beide Alben wieder korrekt unter Interpret, Album und Songs und ließen sich abspielen.

Umgebung

Plex Media Server im LinuxServer.io-Container

Containername: plex

Host-Pfad der Medienfreigabe:

/mnt/nas/media

Mount im Plex-Container:

/media

Musikbibliothek im Container:

/media/Musik

Musikbibliothek auf dem Host:

/mnt/nas/media/Musik

Betroffene Alben:

/media/Musik/Ace of Base/Greatest Hits
/media/Musik/Ace of Base/Greatest Hits, Classic Remixes & Music Videos

Symptome

Die Alben waren nur sichtbar, wenn in Plex die Musikmediathek nach Ordnern angezeigt wurde.

Unter Alben und Interpreten fehlten sie.

Beim Abspielen erschien lediglich:

Wiedergabefehler: Es gab einen Fehler beim Laden der Medien.

Ein zur gleichen Zeit importiertes Album eines anderen Interpreten funktionierte normal.

Beim Öffnen des Ordners erschienen im Plex-Log Meldungen wie:

Unknown metadata type: folder

Beim eigentlichen Abspielversuch entstanden keine neuen Decoder-, Transcoder- oder Dateizugriffsfehler.

Das deutete darauf hin, dass Plex gar keinen echten Track-Wiedergabevorgang startete.

Diagnose

1. Richtigen Containerpfad ermitteln

Der Host-Pfad /mnt/nas/media wird im Container als /media eingehängt. Der tatsächliche Musikpfad lautet daher:

/media/Musik

Zum Auffinden der betroffenen Ordner:

docker exec plex find /media -maxdepth 6 -type d \
  \( -iname '*Ace of Base*' \
     -o -iname '*Greatest Hits*' \) \
  -print

2. Lesbarkeit und Dateigrößen prüfen

Die Dateien wurden als Plex-Benutzer abc geprüft:

BAD='/media/Musik/Ace of Base/Greatest Hits'
GOOD='/media/Musik/Phil Collins/…Hits'

docker exec \
  -u abc \
  -e BAD="$BAD" \
  -e GOOD="$GOOD" \
  plex bash -lc '
for d in "$BAD" "$GOOD"; do
    echo
    echo "=================================================="
    echo "$d"
    echo "=================================================="

    if [ ! -d "$d" ]; then
        echo "ORDNER NICHT GEFUNDEN"
        continue
    fi

    echo "-- Ordnerrechte --"
    stat -c "%A  %a  %U:%G  %n" "$d"

    echo
    echo "-- Dateien, Rechte und Größen --"
    find "$d" -maxdepth 1 -type f \
        -printf "%m  %u:%g  %s Byte  %p\n" |
        sort

    echo
    echo "-- Lesetest als Plex-Benutzer abc --"
    find "$d" -maxdepth 1 -type f -print0 |
    while IFS= read -r -d "" f; do
        if head -c 1 "$f" >/dev/null 2>&1; then
            printf "LESBAR        %s\n" "$f"
        else
            printf "NICHT LESBAR  %s\n" "$f"
        fi
    done
done
'

Ergebnis:

alle FLAC-Dateien waren lesbar,

keine Datei hatte 0 Byte,

die Größen waren plausibel,

Berechtigungen waren nicht die Ursache.

Hinweis: Die vorhandenen Rechte 777 sind unnötig weitreichend, verursachten den Fehler jedoch nicht.

3. Plex-Logs beim Öffnen und Abspielen beobachten

LOG='/opt/docker/plex/config/Library/Application Support/Plex Media Server/Logs'

sudo tail -n 0 -F \
  "$LOG/Plex Media Server.log" \
  "$LOG/Plex Transcoder.log" \
  "$LOG/Plex Media Scanner.log" \
  2>/dev/null |
grep --line-buffered -iE \
'ace of base|greatest hits|the sign|error|warn|failed|unable|invalid|flac|decoder|codec|transcod|not found|no such file|permission'

Beim Öffnen des Ordners erschienen nur:

Unknown metadata type: folder

Beim Abspielversuch entstand kein neuer Transcoder- oder Decoder-Eintrag. Damit war klar, dass Plex keinen normalen Track-Wiedergabevorgang startete.

4. Musikscan beobachten

LOG='/opt/docker/plex/config/Library/Application Support/Plex Media Server/Logs'

sudo tail -n 0 -F \
  "$LOG/Plex Media Scanner.log" \
  "$LOG/Plex Media Server.log" \
  2>/dev/null |
grep --line-buffered -iE \
'ace of base|greatest hits|scanner|scanning|ignore|ignored|skip|skipped|flac|invalid|metadata|error|warn'

Anschließend in Plex:

Musik → Drei-Punkte-Menü → Mediathek-Dateien scannen

Relevante Scanner-Ausgabe:

Scanner: Processing directory /media/Musik/Ace of Base/Greatest Hits
Skipping over directory 'Ace of Base/Greatest Hits', as nothing has changed; removing 16 media items from map.

Für das zweite Album erschien entsprechend:

Skipping over directory 'Ace of Base/Greatest Hits, Classic Remixes & Music Videos', as nothing has changed; removing 15 media items from map.

Der Scanner kannte die Verzeichnisse und hielt sie für unverändert. Ein normaler Scan erzwang deshalb keinen vollständigen Neuimport.

5. Trackobjekte über die Plex-API prüfen

Der entscheidende Test fragte Plex nach Trackobjekten, deren Dateipfad mit dem betroffenen Albumordner beginnt.

PREF="/opt/docker/plex/config/Library/Application Support/Plex Media Server/Preferences.xml"

PLEX_TOKEN=$(sudo sed -n \
  's/.*PlexOnlineToken="\([^"]*\)".*/\1/p' \
  "$PREF")

PLEX_TOKEN="$PLEX_TOKEN" python3 <<'PY'
import os
import urllib.parse
import urllib.request
import xml.etree.ElementTree as ET

token = os.environ["PLEX_TOKEN"]
needle = "/media/Musik/Ace of Base/Greatest Hits/"

params = {
    "type": "10",
    "includeMedia": "1",
    "X-Plex-Container-Start": "0",
    "X-Plex-Container-Size": "10000",
    "X-Plex-Token": token,
}

url = (
    "http://127.0.0.1:32400/library/sections/14/all?"
    + urllib.parse.urlencode(params)
)

with urllib.request.urlopen(url, timeout=30) as response:
    root = ET.parse(response).getroot()

matches = []

for track in root.findall(".//Track"):
    paths = [
        part.get("file", "")
        for part in track.findall(".//Part")
    ]

    if not any(path.startswith(needle) for path in paths):
        continue

    matches.append(
        (
            track.get("ratingKey", "(leer)"),
            track.get("grandparentTitle", "(leer)"),
            track.get("parentTitle", "(leer)"),
            track.get("title", "(leer)"),
            next(
                (path for path in paths if path.startswith(needle)),
                "(kein Pfad)",
            ),
        )
    )

for rating_key, artist, album, title, path in matches:
    print(f"ID:        {rating_key}")
    print(f"Interpret: {artist}")
    print(f"Album:     {album}")
    print(f"Titel:     {title}")
    print(f"Datei:     {path}")
    print("-" * 70)

print(f"Treffer insgesamt: {len(matches)}")
PY

unset PLEX_TOKEN

Ergebnis:

Treffer insgesamt: 0

Damit war bestätigt:

Plex kannte den Albumordner,

Plex hatte aber keine gültigen Trackobjekte für die Dateien,

der Ordnerstatus war veraltet oder unvollständig.

Ursache

Plex hatte für die beiden Albumordner noch gespeicherte Scanner- beziehungsweise Verzeichniseinträge. Gleichzeitig existierten keine gültigen Trackobjekte mehr für die enthaltenen FLAC-Dateien.

Dadurch:

erschienen die Alben nur in der Ordneransicht,

fehlten sie unter Alben und Interpreten,

konnte Plex keinen Track zum Abspielen auflösen,

übersprang ein normaler Scan die Ordner weiterhin als unverändert.

Der ursprüngliche Import fand zeitlich nahe an einer Plex-Metadatenstörung statt. Das ist ein plausibler Auslöser, konnte jedoch nicht abschließend als alleinige Ursache bewiesen werden.

Lösung

1. Betroffene Alben außerhalb der Plex-Musikbibliothek parken

Auf dem Docker-Host:

sudo mkdir -p "/mnt/nas/media/_plex-reimport/Ace of Base"

sudo mv \
  "/mnt/nas/media/Musik/Ace of Base/Greatest Hits" \
  "/mnt/nas/media/_plex-reimport/Ace of Base/"

sudo mv \
  "/mnt/nas/media/Musik/Ace of Base/Greatest Hits, Classic Remixes & Music Videos" \
  "/mnt/nas/media/_plex-reimport/Ace of Base/"

Prüfen:

ls -la "/mnt/nas/media/_plex-reimport/Ace of Base/"

Die Alben liegen damit weiterhin sicher auf dem NAS, aber außerhalb des Plex-Bibliothekspfads /mnt/nas/media/Musik.

2. Musikbibliothek scannen

In Plex:

Musik → Drei-Punkte-Menü → Mediathek-Dateien scannen

Danach prüfen, dass die beiden Ace-of-Base-Ordner nicht mehr in der Ordneransicht der Musikmediathek erscheinen.

3. NAS-Mount vor dem Leeren des Papierkorbs prüfen

Der tatsächliche Mountpoint ist:

/mnt/nas/media

Nicht /mnt/nas.

Sicherheitsprüfung:

mountpoint -q "/mnt/nas/media" \
  && test -d "/mnt/nas/media/Musik" \
  && echo "NAS und Musikpfad sind verfügbar" \
  || echo "STOPP: NAS oder Musikpfad nicht verfügbar"

Nur bei folgender Ausgabe fortfahren:

NAS und Musikpfad sind verfügbar

Diese Prüfung ist wichtig, weil Plex beim Leeren des Papierkorbs alle momentan nicht erreichbaren Medienobjekte der betroffenen Bibliothek entfernen kann.

4. Papierkorb der Musikmediathek leeren

In Plex ausschließlich bei der Musikmediathek:

Musik → Drei-Punkte-Menü → Papierkorb leeren

Dadurch werden die veralteten Bibliotheks- und Trackzustände entfernt.

5. Alben zurückverschieben

sudo mv \
  "/mnt/nas/media/_plex-reimport/Ace of Base/Greatest Hits" \
  "/mnt/nas/media/Musik/Ace of Base/"

sudo mv \
  "/mnt/nas/media/_plex-reimport/Ace of Base/Greatest Hits, Classic Remixes & Music Videos" \
  "/mnt/nas/media/Musik/Ace of Base/"

Dateianzahl beispielhaft prüfen:

find "/mnt/nas/media/Musik/Ace of Base/Greatest Hits" \
  -maxdepth 1 -type f -name '*.flac' |
wc -l

Erwartet:

16

6. Musikbibliothek erneut scannen

In Plex:

Musik → Drei-Punkte-Menü → Mediathek-Dateien scannen

Da die alten Einträge zuvor entfernt wurden, behandelt Plex die zurückverschobenen Dateien jetzt als neuen Import und legt neue Trackobjekte an.

Verifikation

Nach dem erneuten Import funktionierte alles wieder:

beide Ace-of-Base-Alben erschienen unter Alben,

der Interpret erschien unter Interpreten,

die Titel waren unter Songs sichtbar,

die Dateien ließen sich abspielen.

Optional kann der API-Test erneut ausgeführt werden. Statt 0 sollten nun die erwarteten Trackobjekte gefunden werden.

Wichtige Erkenntnisse

Ordneransicht ist kein Beweis für einen erfolgreichen Musikimport

Die Plex-Ordneransicht kann einen Dateisystemordner anzeigen, obwohl keine gültigen Album- oder Trackobjekte in der Bibliothek existieren.

Ein erfolgreicher Lesetest schließt nur Dateirechte aus

Lesbare Dateien mit plausiblen Größen beweisen nicht, dass Plex sie korrekt katalogisiert hat.

Ein normaler Scan repariert veraltete Ordnerzustände nicht immer

Wenn Plex einen Ordner bereits als unverändert gespeichert hat, kann er ihn trotz fehlender Trackobjekte weiterhin überspringen.

Papierkorb nur bei sicher verfügbarem NAS leeren

Vor dem Leeren immer den tatsächlichen Mountpoint und den Bibliothekspfad prüfen:

mountpoint -q "/mnt/nas/media" \
  && test -d "/mnt/nas/media/Musik"

Keine komplette Bibliothek neu aufbauen

Der gezielte Reimport der betroffenen Albumordner war ausreichend. Die übrige Plex-Musikbibliothek blieb unangetastet.

Kompakter Wiederherstellungsablauf

1. Betroffene Albumordner aus /mnt/nas/media/Musik herausverschieben
2. Plex-Musikbibliothek scannen
3. Prüfen, dass die Ordner aus Plex verschwunden sind
4. NAS-Mount und Musikpfad prüfen
5. Papierkorb der Musikmediathek leeren
6. Albumordner zurückverschieben
7. Musikbibliothek erneut scannen
8. Albumansicht, Trackliste und Wiedergabe prüfen

Status

Gelöst

Die beiden betroffenen Ace-of-Base-Alben wurden erfolgreich neu importiert und sind wieder vollständig sichtbar und abspielbar.
