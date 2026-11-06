# Lager – Inventar mit Bluetooth-Scanner

**Stand: Meilensteine 1–6 fertig.** Scan-Engine, Scanner-Anbindung (Inateck BCST-70), Live-Anzeige mit Tönen, Verwaltung, Etiketten, Gebäudeplan, Datensicherung und Installation auf dem Raspberry Pi.

## Inbetriebnahme auf dem Raspberry Pi

### 0. Pi vorbereiten (einmalig)

Mit dem **Raspberry Pi Imager** „Raspberry Pi OS (64-bit)“ auf die SD-Karte schreiben. Die Version mit Desktop wählen, wenn am Pi ein Bildschirm die Live-Anzeige zeigen soll. In den Einstellungen des Imagers:
Rechnername `lager`, Benutzer `pi` mit Passwort, WLAN (falls kein Netzwerkkabel) und **SSH aktivieren**. Karte einlegen, Pi starten. Der Pi braucht für die Installation Internet.

### 1. Draufspielen – ein Befehl auf dem Windows-PC

```powershell
cd "$env:USERPROFILE\OneDrive - Volkswagen AG\Documents\Vs Code\lager"
powershell -ExecutionPolicy Bypass -File .\deploy\push.ps1
```

Heißt der Pi anders oder ist er nur per IP erreichbar: `... push.ps1 -PiHost 192.168.1.50 -PiUser pi`.
Das Skript packt das Projekt, kopiert es per `scp` auf den Pi und startet dort `deploy/install.sh`. Es fragt ein- bis zweimal nach dem Pi-Passwort.
Am Ende stehen in der Ausgabe die Adresse und das **Admin-Passwort**, danach öffnet sich der Browser mit der Einrichtung.
Updates: denselben Befehl noch einmal ausführen. Daten und `.env` bleiben erhalten.

Ohne Windows-PC: Projektordner per USB-Stick auf den Pi kopieren und dort `sudo bash deploy/install.sh` ausführen.

### 2. Fertig einrichten – im Browser

**Admin → Einrichtung** führt in 10 Schritten durch alles Weitere. Die Haken setzen sich automatisch:
Zugang, Scanner koppeln (`sudo lager-scanner-koppeln`), Scanner testen, Standard-Lager, Räume, Artikel, Etiketten, Gebäudeplan, Probelauf, Sicherung.

**Versteckter Admin-Zugang:** fünfmal schnell auf das Logo „Lager“ oben links tippen oder `/admin` aufrufen. Normale Nutzer sehen nur *Live*, *Bestand* und *Plan*.

### Was die Installation einrichtet

| Was | Wo |
|---|---|
| Programm (nur lesbar für den Dienst) | `/opt/lager`, Konfiguration `/opt/lager/.env` |
| Daten: Datenbank, Pläne, Sicherungen | `/var/lib/lager` |
| Webserver als Dienst `lager` (Benutzer `lager`, Gruppen `input` + `bluetooth`) | `http://lager.local:8000` |
| Sicherung jede Nacht 02:30, die letzten 14 bleiben | Timer `lager-backup.timer` |
| Scanner koppeln | `sudo lager-scanner-koppeln` |
| Vollbild-Browser mit `/live` nach der Anmeldung am Pi-Desktop | `~/.config/autostart/lager-kiosk.desktop` |

```bash
sudo systemctl status lager      # läuft der Dienst?
sudo journalctl -u lager -f      # Live-Log
sudo systemctl restart lager     # nach Änderungen an der .env
```

Wiederherstellen einer Sicherung:

```bash
sudo systemctl stop lager
sudo rm -f /var/lib/lager/lager.db-wal /var/lib/lager/lager.db-shm
sudo -u lager unzip -o /var/lib/lager/backups/lager-JJJJMMTT-HHMMSS.zip -d /var/lib/lager
sudo systemctl start lager
```

## Scanner: Inateck BCST-70

- **1D-Laserscanner.** Er liest Strichcodes, **keine QR-Codes**. Die Etiketten tragen deshalb einen **Code 128** und zusätzlich einen kleinen QR-Code für spätere 2D-Scanner.
- Koppeln: am Scanner im Handbuch *Enter Setup* → *Bluetooth Pairing* scannen, dann auf dem Pi `sudo lager-scanner-koppeln`. Danach verbindet der Pi ihn automatisch. Per USB-Kabel geht es auch ohne Koppeln.
- Der Scanner ist für Linux eine Tastatur. Die App liest ihn direkt über `/dev/input` und belegt ihn exklusiv. Gescannte Codes landen also nie im Browser.
- Werkseinstellungen passen: US-Tastatur, Enter am Ende. Wurde die Tastatursprache auf Deutsch gestellt: `SCANNER_KEYMAP=de`.
- Nach 3 Minuten ohne Benutzung schläft er ein. Ein Druck auf den Abzug weckt ihn, die App verbindet sich von selbst neu („Scanner verbunden“ mit Ton).

## Bedienung: das Session-Modell

1. **Raum scannen** (z. B. `R-IT-TECHNIK`). Damit beginnt eine Session für diesen Raum.
2. **Beliebig viele Artikel scannen.** Jeder Artikel (`E-…` Einzelstück, `T-…` Artikeltyp) wird sofort in den aktiven Raum gebucht und gespeichert. Der Raum bleibt aktiv, ein erneuter Raum-Scan ist nicht nötig.
3. **Anderen Raum scannen.** Das Ziel wechselt sofort.
4. **60 Sekunden ohne Scan:** Die Session endet automatisch („Kein Raum ausgewählt“). Ein Artikel-Scan danach ist ein Fehler und bucht nichts.
5. **Falsch gebucht?** `R-LAGER` scannen und denselben Artikel noch einmal scannen. Das ist eine neue, normale Rückbuchung ins Lager. Ein Undo gibt es nicht. Beide Buchungen bleiben im Verlauf.

Es gibt keine Modi, keine Bestätigung, keinen Fertig- und keinen Undo-Befehl.

| Code | Bedeutung |
|---|---|
| `R-<KÜRZEL>` | Raum/Ort, z. B. `R-LAGER`, `R-IT-TECHNIK` |
| `T-<KÜRZEL>` | Artikeltyp mit Menge, z. B. `T-HDMI-2M` – 1 Stück pro Scan |
| `E-<NUMMER>` | Einzelstück, z. B. `E-0042` |

### Regeln der Scan-Engine

| Scan | Ergebnis | Session danach |
|---|---|---|
| bekannter Raum | Session startet bzw. Ziel wechselt | aktiv, 60 s laufen neu |
| unbekannter Raum | Fehler | beendet |
| Artikel ohne Session | Fehler, nichts gebucht | keine |
| unbekannter E-/T-Code | Fehler, nichts gebucht, Code für „Neu anlegen“ | bleibt, 60 s neu |
| ausgemustertes Stück | Fehler, nichts gebucht | bleibt, 60 s neu |
| Stück ist schon im Ziel | Warnung „Schon in …“, nichts gebucht | bleibt, 60 s neu |
| Stück liegt woanders | gebucht | bleibt, 60 s neu |
| Typ in einen Raum | Standard-Lager −1, Raum +1 (Lager war 0: bleibt 0, Warnung) | bleibt, 60 s neu |
| Typ ins Lager | Raum mit Bestand, der den Typ zuletzt erhalten hat, −1, Lager +1 (kein solcher Raum: Lager +1, Warnung) | bleibt, 60 s neu |
| unbekanntes Präfix / ungültiger Code | Fehler | beendet (Sicherheit) |
| gleicher Code innerhalb 500 ms | ignoriert: kein Log, kein Ton, Timer unverändert | unverändert |
| Scanner getrennt / Neustart | bei Trennung einmal Ton „Scanner getrennt“ | beendet |

### Töne (im Browser der Live-Anzeige)

| Ereignis | Ton |
|---|---|
| Session gestartet | zwei steigende Töne |
| Raum gewechselt | drei steigende Töne |
| Buchung gespeichert | kurzer heller Piep |
| Warnung | zwei mittlere Pieptöne |
| Fehler | tiefer, langer Brummton |
| Session abgelaufen | drei fallende Töne |
| Scanner verbunden / getrennt | Dreiklang aufwärts / zwei tiefe Töne abwärts |

Der Kiosk-Browser auf dem Pi darf sofort Töne abspielen. In anderen Browsern einmal auf „Ton aktivieren“ klicken. 🔊 oben rechts schaltet den Ton pro Browser aus.

## Entwicklung unter Windows

Voraussetzung: **Python 3.11 oder neuer** (prüfen mit `python --version`).

```powershell
cd "$env:USERPROFILE\OneDrive - Volkswagen AG\Documents\Vs Code\lager"
python -m venv "$env:LOCALAPPDATA\venvs\lager"          # einmalig – bewusst außerhalb von OneDrive
& "$env:LOCALAPPDATA\venvs\lager\Scripts\Activate.ps1"  # in jedem neuen Terminal
pip install -r requirements.txt                         # einmalig bzw. nach Änderungen
python -m pytest                                        # alle Tests
python -m app                                           # Webserver: http://localhost:8000/live
```

Falls PowerShell das Aktivieren blockiert, statt `python` direkt `& "$env:LOCALAPPDATA\venvs\lager\Scripts\python.exe"` aufrufen.
Unter Windows gibt es keinen Scanner. Admin-Passwort ohne `.env` ist `admin`, unter *Admin → Einrichtung → Scanner testen* lassen sich Scans eintippen.
Demo-Daten: *Admin → System → Demo-Daten laden* oder `python -m scripts.seed_demo`. Zum Neuanfang *Admin → System → Alle Daten löschen*.
Die Datenbank liegt standardmäßig in `data\lager.db` im Projektordner, unter Windows also in OneDrive, dessen Synchronisierung SQLite-Dateien sperren kann. Besser: `.env.example` nach `.env` kopieren und dort `DB_PATH=C:\Users\<Benutzer>\AppData\Local\lager-daten\lager.db` eintragen.

## Konfiguration

`.env` im Projektordner (Vorlage mit Erklärungen: `.env.example`; auf dem Pi `/opt/lager/.env`, UTF-8, Kommentare nur in eigenen Zeilen) oder Umgebungsvariablen. Umgebungsvariablen haben Vorrang.

| Variable | Standard | Bedeutung |
|---|---|---|
| `DB_PATH` | `data/lager.db` | SQLite-Datei (relativ zum Projektordner oder absolut) |
| `SESSION_TIMEOUT_S` | `60` | Session endet nach so vielen Sekunden ohne Scan |
| `DEBOUNCE_MS` | `500` | gleicher Code innerhalb dieser Zeit gilt als Doppel-Scan |
| `HOST`, `PORT` | `0.0.0.0`, `8000` | Adresse der Weboberfläche |
| `ADMIN_PASSWORD` | leer = `admin` | Passwort für den Admin-Bereich (die Installation erzeugt eins) |
| `SECRET_KEY` | leer = zufällig pro Start | signiert das Login-Cookie |
| `SCANNER` | `auto` | `auto`: Scanner unter Linux lesen, `off`: nur Test-Scans |
| `SCANNER_NAME` | `BCST\|Inateck\|Barcode\|Scanner` | regulärer Ausdruck auf den Gerätenamen |
| `SCANNER_DEVICE` | leer | fester Gerätepfad, z. B. `/dev/input/event5` |
| `SCANNER_KEYMAP` | `us` | Tastatursprache des Scanners: `us` oder `de` |
| `BACKUP_DIR`, `BACKUP_KEEP` | `backups` neben der DB, `14` | Ablage und Anzahl der Sicherungen |

## Aufbau

| Datei | Aufgabe |
|---|---|
| `app/scan_engine.py` | Zustandsmaschine – reine Logik ohne I/O. Die Zeit kommt als Parameter `now`, der Datenzugriff über die Schnittstelle `ScanStore`. Liefert strukturierte Ergebnisse inklusive Feedback-Ereignis. |
| `app/db.py` | SQLite (WAL, Fremdschlüssel), Schema-Version, `SqlScanStore`. `read_sessions` zum Lesen (blockiert keinen Scan), `write_sessions` für alles, was Daten ändert (`BEGIN IMMEDIATE`). |
| `app/models.py` | Tabellen (SQLAlchemy 2.0) mit CHECK-Constraints |
| `app/scanner.py` | Scanner über Linux-evdev: Tastencodes → Text, Gerät finden, exklusiv belegen, Wiederverbindung |
| `app/live.py` | Scans durch die Engine schicken, Session-Timeout, Scanner-Status, Verteilung an alle Browser per Server-Sent Events |
| `app/inventory.py` | Abfragen für die Seiten und geprüfte Änderungen aus der Verwaltung |
| `app/main.py` | FastAPI-App, öffentliche Seiten (`/live`, `/bestand`, `/plan`), `/api/events`, Start/Stopp |
| `app/admin.py` | Admin-Bereich: Einrichtung, Räume, Artikel, Einzelstücke, Etiketten, Pläne, Protokoll, System |
| `app/labels.py` | Etiketten-PDF (Code 128 + QR) |
| `app/backup.py` | Sicherung als ZIP (konsistente SQLite-Kopie + Pläne) |
| `app/auth.py` | Admin-Login mit signiertem Cookie |
| `app/templates/`, `app/static/` | Oberfläche (Jinja2, CSS, JavaScript ohne Framework); Töne erzeugt `app.js` mit Web Audio |
| `deploy/` | `push.ps1` (Windows → Pi), `install.sh`, systemd-Dienste, Kiosk, Koppel-Skript |
| `scripts/seed_demo.py`, `scripts/backup.py` | Demo-Daten, Sicherung per Kommandozeile |

Ablauf pro Scan: Lock nehmen, Timeout prüfen, dann eine DB-Transaktion (Buchung und `scan_log`). Den neuen Session-Zustand übernimmt die Engine erst nach dem Commit; erst danach geht das Ereignis an die Browser.
Die Session (aktiver Raum und letzter Scan) liegt bewusst nur im Speicher und ist nach einem Neustart leer.

## Annahmen

- Hardware: Raspberry Pi mit Raspberry Pi OS Bookworm oder neuer (Python ≥ 3.11), Scanner **Inateck BCST-70** (Bluetooth 3.0 HID, 1D-Laser, Werkseinstellung US-Tastatur mit Enter am Ende). Ton kommt aus dem Browser der Live-Anzeige, auf dem Pi also über HDMI-Monitor oder Klinke. Einen eigenen Lautsprecher-Treiber im Pi gibt es nicht.
- Weil der BCST-70 keine QR-Codes liest, tragen alle Etiketten einen Code 128. Der Hinweistext der Engine („Bitte zuerst den QR-Code des Ziel-Raumes scannen“) ist unverändert geblieben, weil die Engine-Texte zur abgenommenen Geschäftslogik gehören.
- Codes sind höchstens 24 Zeichen lang. So bleiben die Striche auf einem 70-mm-Etikett mindestens 0,2 mm breit. Raum-, Typ- und Einzelstück-Codes lassen sich nach dem Anlegen nicht mehr ändern (sie kleben schon). E-Codes werden fortlaufend vergeben: höchste Nummer + 1, mindestens vierstellig.
- Der Scanner wird am Gerätenamen erkannt; genommen wird das erste passende Eingabegerät mit Buchstaben- und Enter-Taste. Eine angefangene Eingabe, die länger als 1 s auf Enter wartet, wird verworfen. Unbekannte Tasten werden zu „?“, damit ein verstümmelter Scan ein ungültiger Code wird und kein zufällig gültiger.
- Admin-Bereich: ein gemeinsames Passwort aus der `.env`, Login gilt 12 Stunden. *Bestand* und *Plan* sind für alle sichtbar, aber nur lesend.
- Löschen geht nur ohne Inhalt und ohne Buchungen im Verlauf. Einzelstücke mit Verlauf werden stattdessen „ausgemustert“. Neu angelegte Einzelstücke erzeugen keine Buchung (wie der Seed). Ein Ortswechsel im Formular ist eine Buchung mit Quelle `web`.
- Bestandskorrektur: Man trägt die gezählte Menge ein, gebucht wird die Differenz als `correction`. Der Anfangsbestand eines neuen Typs ist eine Korrektur im Standard-Lager.
- Test-Scans im Admin-Bereich laufen durch dieselbe Engine und buchen wirklich. Sie erscheinen mit Quelle `scanner` im Verlauf, weil die Engine die Quelle nicht kennt.
- Gebäudepläne: PNG, JPG, WebP oder GIF bis 20 MB, je Raum eine rechteckige Zone in Prozent des Bildes, ein Raum liegt auf höchstens einem Plan.
- „Alle Daten löschen“ erstellt vorher eine Sicherung und behält nur Timeout und Debounce in `settings`.
- Zeitstempel werden in UTC gespeichert und in der Ortszeit des Pi angezeigt. Die Installation stellt die Zeitzone auf Europe/Berlin, wenn noch die Vorgabe des Images (Europe/London bzw. UTC) aktiv ist. Der Pi hat keine Echtzeituhr und braucht für die richtige Zeit Netzwerk (NTP); *System* zeigt an, ob die Zeit synchronisiert ist.
- Kiosk: Er startet nur, wenn ein Desktop mit Chromium vorhanden ist, für den Benutzer, der die Installation mit `sudo` gestartet hat. `/live?kiosk=1` holt öffentliche Seiten nach 2 Minuten ohne Bedienung zurück auf die Live-Anzeige.
- Maßgeblich für Timeout und Debounce sind `SESSION_TIMEOUT_S` und `DEBOUNCE_MS` aus der Konfiguration. Die Tabelle `settings` zeigt nach dem Start die aktiven Werte an. `default_storage_location_id` steht nur in `settings` und wird unter *Räume* gesetzt.
- Code-Syntax: Präfix, Bindestrich, dann Großbuchstaben oder Ziffern, auch mit weiteren Bindestrichen (z. B. `R-IT-TECHNIK`). Bekannte Präfixe sind nur `R`, `T` und `E`. Die Engine prüft E-Codes nicht auf Ziffern: `E-ABC` gilt als unbekanntes Einzelstück, die Session bleibt bestehen.
- Leere Eingaben (nur Enter oder Leerzeichen) werden wie ein Doppel-Scan ignoriert. `scan_log.raw` speichert die Eingabe unverändert. Gesucht, verglichen und gebucht wird mit dem normalisierten Code (getrimmt, Großbuchstaben).
- Debounce vergleicht den normalisierten Code mit dem zuletzt verarbeiteten Scan. Bei genau 500 ms Abstand wird der Scan wieder verarbeitet.
- Timeout-Grenze: Genau 60 s nach dem letzten Scan gilt die Session als abgelaufen. Ist `now` bis zu 5 s älter als der letzte Scan, ist das kein Uhrsprung, sondern nur ein Zeitversatz zwischen Timer und Scan. Springt die Systemuhr weiter zurück, endet die Session, damit der Zustand sicher bleibt.
- `location_changed` gibt es nur beim Wechsel während einer aktiven Session. Erneut derselbe Raum oder ein Raum nach Session-Ende ergibt `session_started`.
- „Schon in <Ort>“ ist eine Warnung ohne Buchung und bekommt den Warnton. Defekte Stücke werden normal gebucht, nur ausgemusterte werden abgelehnt.
- Rücklagerung eines Typs: Als „erhalten“ zählt jede Bewegung mit Menge > 0 in einen Nicht-Lager-Ort (Scanner-Buchung oder positive Korrektur). Die zuletzt gebuchte Bewegung zählt zuerst; maßgeblich ist die Buchungsreihenfolge, nicht die Systemuhr. Das Stück kommt damit aus dem Raum, der den Typ zuletzt erhalten hat – das ist nicht zwingend der Raum der Fehlbuchung.
- Es kann mehrere Lager geben (`is_storage`). Jeder Typ-Scan in ein Lager ist eine Rücklagerung; ausgelagert wird immer aus dem Standard-Lager. Das Standard-Lager muss als Lager markiert sein. Fehlt es oder ist es nicht markiert, ist eine Typ-Buchung in einen Raum ein Fehler; die Session bleibt bestehen.
- Korrekturbuchungen haben `kind=correction`, kein `from_location_id` und als `qty` die Differenz mit Vorzeichen. Der Seed bucht den Anfangsbestand als Korrektur.
- Bei einem internen Verarbeitungsfehler (z. B. in der Datenbank) wird nichts gebucht, und die Session endet sicherheitshalber. Das Gleiche gilt, wenn der Ziel-Raum während der Session gelöscht wurde.
- Bei einem Scanner-Verlust meldet die Engine das Session-Ende ohne eigenen Ton; den einmaligen Ton „Scanner getrennt“ spielt die Live-Schicht.
- Die Datenbank prüft Präfixe und Wertebereiche, die Formulare prüfen Pflichtangaben (Name, Verantwortlicher eines Raumes) und die vollständige Code-Syntax.
- SQLite läuft mit `synchronous=FULL`, damit eine bestätigte Buchung auch einen Stromausfall übersteht.
- Die Zonen-Koordinaten (`zone_x` usw.) dürfen leer sein, solange für einen Raum keine Zone eingezeichnet ist.
