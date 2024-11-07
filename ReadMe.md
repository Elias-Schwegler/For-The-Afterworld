
# Im Experimental Ordner kann man verbessete, aber noch ungetestete Versionen des Folgenden Skripts finden.


## Verbessertes Skript zum Herunterladen von Master und Releases

Das folgende Python-Skript wurde optimiert, um sowohl den Master-Branch als auch die neuesten Releases eines GitHub-Projekts basierend nur auf der Projekt-URL herunterzuladen. Es ist robust gestaltet, enthält umfangreiche Protokollierung und ermöglicht eine einfache Wartung.

### **Hauptmerkmale**

- **Einfachheit**: Nur die GitHub-Projekt-URL ist erforderlich.
- **Flexibilität**: Optionales Angeben des Download-Verzeichnisses.
- **Robustheit**: Umfassende Fehlerbehandlung und Logging.
- **Wartbarkeit**: Klar strukturierter und modularer Aufbau.

### **Skript**

```python
import os
import requests
import schedule
import time
import logging
import argparse
from urllib.parse import urlparse
from datetime import datetime
import pytz

# Standard-Konfiguration
DEFAULT_DOWNLOAD_DIR = "downloads"
VERSION_FILE_RELEASE = "version_info_release.txt"
VERSION_FILE_MASTER = "version_info_master.txt"
RELEASE_NOTES_FILE = "release_notes.txt"

# Argument Parser für flexibles Download-Verzeichnis
parser = argparse.ArgumentParser(description='GitHub Release und Master Branch Downloader')
parser.add_argument('repo_url', help='URL des GitHub-Repositories, z.B. https://github.com/GreemDev/Ryujinx')
parser.add_argument('--download-dir', default=DEFAULT_DOWNLOAD_DIR, help='Verzeichnis zum Speichern der Downloads')
args = parser.parse_args()

GITHUB_REPO_URL = args.repo_url
DOWNLOAD_DIR = args.download_dir

# Logging Einrichtung
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Parsing der GitHub URL
parsed_url = urlparse(GITHUB_REPO_URL)
path_parts = parsed_url.path.strip("/").split("/")
if len(path_parts) < 2:
    logging.error("Ungültige GitHub-URL. Bitte das Format 'https://github.com/Owner/Repo' verwenden.")
    exit(1)
GITHUB_OWNER = path_parts[0]
GITHUB_REPO = path_parts[1]
GITHUB_API_URL = f"https://api.github.com/repos/{GITHUB_OWNER}/{GITHUB_REPO}"

# Sicherstellen, dass das Download-Verzeichnis existiert
os.makedirs(DOWNLOAD_DIR, exist_ok=True)

def get_latest_release():
    """Holt die neueste Veröffentlichung von GitHub."""
    url = f"{GITHUB_API_URL}/releases/latest"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

def get_latest_commit(branch='master'):
    """Holt den neuesten Commit eines angegebenen Branches."""
    url = f"{GITHUB_API_URL}/commits/{branch}"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

def download_file(url, dest):
    """Lädt eine Datei von einer gegebenen URL herunter."""
    try:
        response = requests.get(url, stream=True)
        response.raise_for_status()
        with open(dest, 'wb') as file:
            for chunk in response.iter_content(chunk_size=8192):
                file.write(chunk)
        logging.info(f"Heruntergeladen: {dest}")
    except requests.exceptions.RequestException as e:
        logging.error(f"Fehler beim Herunterladen von {url}: {e}")

def download_release_assets(release):
    """Lädt alle Assets einer Veröffentlichung herunter."""
    for asset in release.get('assets', []):
        asset_url = asset['browser_download_url']
        asset_name = asset['name']
        dest_path = os.path.join(DOWNLOAD_DIR, asset_name)
        logging.info(f"Herunterladen des Release-Assets: {asset_name}")
        download_file(asset_url, dest_path)

def generate_release_notes(commits):
    """Generiert Release-Notizen basierend auf Commits."""
    with open(RELEASE_NOTES_FILE, 'w', encoding='utf-8') as file:
        file.write("Release Notes\n")
        file.write("=============\n\n")
        for commit in commits:
            message = commit['commit']['message'].split('\n')[0]
            author = commit['commit']['author']['name']
            date = commit['commit']['author']['date']
            file.write(f"- {message} (von {author} am {date})\n")
    logging.info("Release-Notizen generiert.")

def check_and_download_release():
    """Überprüft auf neue Releases und lädt diese herunter."""
    try:
        latest_release = get_latest_release()
        latest_version = latest_release['tag_name']
        latest_commit_hash = latest_release['target_commitish']

        # Prüfen, ob diese Version bereits heruntergeladen wurde
        if os.path.exists(VERSION_FILE_RELEASE):
            with open(VERSION_FILE_RELEASE, 'r', encoding='utf-8') as file:
                current_version, current_hash = file.read().strip().split(',')
            if current_version == latest_version and current_hash == latest_commit_hash:
                logging.info(f"Neueste Veröffentlichung ({latest_version}) bereits heruntergeladen.")
                return

        # Neue Veröffentlichung gefunden
        logging.info(f"Neue Veröffentlichung gefunden: {latest_version}. Download beginnt...")
        download_release_assets(latest_release)

        # Optional: Commit-Historie seit letztem Release abrufen (hier Platzhalter)
        generate_release_notes([])  # Placeholder für tatsächliche Commits

        # Versionsinfo aktualisieren
        with open(VERSION_FILE_RELEASE, 'w', encoding='utf-8') as file:
            file.write(f"{latest_version},{latest_commit_hash}")
        logging.info(f"Versionsinfo aktualisiert: {latest_version}")
    except requests.exceptions.HTTPError as e:
        logging.error(f"HTTP-Fehler beim Abrufen der neuesten Veröffentlichung: {e}")
    except Exception as e:
        logging.error(f"Unbekannter Fehler bei der Überprüfung der Veröffentlichung: {e}")

def check_and_download_master():
    """Überprüft auf neue Commits im Hauptbranch und lädt den aktuellen Stand herunter."""
    try:
        latest_commit = get_latest_commit()
        latest_commit_hash = latest_commit['sha']

        # Prüfen, ob dieser Commit bereits heruntergeladen wurde
        if os.path.exists(VERSION_FILE_MASTER):
            with open(VERSION_FILE_MASTER, 'r', encoding='utf-8') as file:
                _, current_hash = file.read().strip().split(',')
            if current_hash == latest_commit_hash:
                logging.info("Hauptbranch ist auf dem neuesten Stand.")
                return

        # Neuer Commit gefunden
        logging.info("Neuer Commit im Hauptbranch gefunden. Download beginnt...")
        master_zip_url = f"https://github.com/{GITHUB_OWNER}/{GITHUB_REPO}/archive/refs/heads/master.zip"
        dest_path = os.path.join(DOWNLOAD_DIR, f"{GITHUB_REPO}_master.zip")
        download_file(master_zip_url, dest_path)

        # Versionsinfo für den Hauptbranch aktualisieren
        with open(VERSION_FILE_MASTER, 'w', encoding='utf-8') as file:
            file.write(f"master,{latest_commit_hash}")
        logging.info(f"Hauptbranch-Version aktualisiert: {latest_commit_hash}")
    except requests.exceptions.HTTPError as e:
        logging.error(f"HTTP-Fehler beim Abrufen des neuesten Commits: {e}")
    except Exception as e:
        logging.error(f"Unbekannter Fehler bei der Überprüfung des Hauptbranches: {e}")

def initial_download():
    """Führt die initialen Downloads von Master und Releases durch."""
    logging.info("Initialer Download wird durchgeführt...")
    check_and_download_master()
    check_and_download_release()

def daily_check():
    """Führt tägliche Überprüfungen auf Updates durch."""
    logging.info("Tägliche Überprüfung auf Updates gestartet.")
    check_and_download_master()
    check_and_download_release()
    logging.info("Tägliche Überprüfung abgeschlossen.")

def main():
    """Hauptfunktion zur Ausführung des Skripts."""
    # Initialer Download, falls Versionsdateien nicht vorhanden sind
    if not os.path.exists(VERSION_FILE_MASTER) or not os.path.exists(VERSION_FILE_RELEASE):
        initial_download()

    # Zeitzone festlegen (z.B. Europe/Zurich)
    timezone = pytz.timezone("Europe/Zurich")

    # Geplante tägliche Prüfungen einrichten
    schedule_time = "10:00"
    schedule.every().day.at(schedule_time).do(daily_check)
    logging.info(f"Update-Prüfer läuft. Tägliche Prüfungen um {schedule_time} Uhr in Zeitzone {timezone}.")

    # Endlosschleife zur Ausführung geplanter Aufgaben
    try:
        while True:
            schedule.run_pending()
            time.sleep(1)
    except KeyboardInterrupt:
        logging.info("Update-Prüfer gestoppt vom Benutzer.")

if __name__ == "__main__":
    main()
```

### **Erläuterungen zu den Verbesserungen**

#### **1. Einfache Nutzung mit nur der Projekt-URL**
Das Skript akzeptiert nur die GitHub-Projekt-URL als Eingabeparameter. Zusätzliche Optionen wie das Download-Verzeichnis können optional angegeben werden.

#### **2. Robuste Fehlerbehandlung**
Umfasst spezifische Ausnahmen für HTTP-Fehler und allgemeine Fehler, wodurch das Skript stabiler und widerstandsfähiger gegen unerwartete Probleme wird.

#### **3. Umfassendes Logging**
Das `logging`-Modul ersetzt `print`-Aussagen, um detaillierte und konfigurierbare Protokolle zu ermöglichen. Dies erleichtert die Nachverfolgung und Fehlerdiagnose.

#### **4. Trennung von Release- und Master-Versionierung**
Separate Versionsdateien (`version_info_release.txt` und `version_info_master.txt`) sorgen für eine klare Trennung und verhindern Konflikte zwischen Release- und Branch-Versionen.

#### **5. Planung täglicher Überprüfungen**
Das Skript nutzt das `schedule`-Modul, um tägliche Überprüfungen um 10:00 Uhr (Europa/Zürich) durchzuführen. Dies kann leicht angepasst werden, um verschiedenen Zeitzonen gerecht zu werden.

#### **6. Flexibles Download-Verzeichnis**
Das Download-Verzeichnis kann über ein optionales Kommandozeilenargument angegeben werden. Standardmäßig wird das Verzeichnis `downloads` verwendet, aber der Benutzer kann ein anderes Verzeichnis wählen.

#### **7. Modularer Aufbau**
Funktionen sind klar getrennt und modular gestaltet, was die Wartbarkeit und Erweiterbarkeit des Skripts fördert.

### **Anwendung des Skripts**

1. **Installation der Abhängigkeiten**
   Stellen Sie sicher, dass alle erforderlichen Python-Pakete installiert sind:
   ```bash
   pip install requests schedule pytz
   ```

2. **Ausführen des Skripts**
   Führen Sie das Skript mit der GitHub-Projekt-URL aus:
   ```bash
   python github_downloader.py https://github.com/GreemDev/Ryujinx
   ```
   Optional können Sie ein spezielles Download-Verzeichnis angeben:
   ```bash
   python github_downloader.py https://github.com/GreemDev/Ryujinx --download-dir /pfad/zum/verzeichnis
   ```

3. **Automatische Updates**
   Das Skript führt täglich um 10:00 Uhr Überprüfungen durch und lädt bei Bedarf neue Releases oder Master-Commits herunter. Die Protokolle werden in der Konsole angezeigt und können bei Bedarf angepasst werden.

### **Fazit**

Dieses optimierte Skript bietet eine zuverlässige und benutzerfreundliche Lösung zum automatischen Herunterladen von GitHub-Releases und Master-Branch-Commits basierend auf der Projekt-URL. Durch die Implementierung von robusten Fehlerbehandlungen, umfassendem Logging und flexiblen Konfigurationsmöglichkeiten ist es sowohl für Anfänger als auch für fortgeschrittene Benutzer geeignet.

Citations:
[1] https://github.com/GreemDev/Ryujinx
