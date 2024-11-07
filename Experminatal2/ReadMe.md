## Erweiterung des Skripts zur Echtzeitüberwachung, erweiterten Release-Notizen und konfigurierbaren Zeitzoneneinstellungen

Das folgende Python-Skript wurde weiter optimiert, um die folgenden zusätzlichen Funktionen zu integrieren:

1. **Echtzeitüberwachung der `repositories.txt`**: Neue GitHub-URLs können dynamisch hinzugefügt und automatisch verarbeitet werden, ohne das Skript neu starten zu müssen.
2. **Erweiterte Release-Notizen**: Die tatsächliche Commit-Historie seit dem letzten Release wird integriert, um detailliertere Release-Notizen zu generieren.
3. **Konfigurierbare Zeitzoneneinstellungen**: Die tägliche Prüfungszeit und die Zeitzone können flexibel über Kommandozeilenargumente festgelegt werden.

### **Hauptmerkmale der Erweiterungen**

- **Live-Monitoring**: Nutzung der `watchdog`-Bibliothek zur Überwachung der `repositories.txt` und automatischen Verarbeitung neuer URLs.
- **Detaillierte Release-Notizen**: Nutzung der GitHub-Compare-API, um Commits zwischen dem letzten Release und dem aktuellen Stand abzurufen und in die Release-Notizen aufzunehmen.
- **Flexible Zeitplanung**: Einführung von Kommandozeilenargumenten zur Festlegung der Prüfungszeit und der Zeitzone.

### **Voraussetzungen**

Stellen Sie sicher, dass die folgenden Python-Pakete installiert sind:

```bash
pip install requests schedule pytz watchdog
```

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
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# Standard-Konfiguration
DEFAULT_DOWNLOAD_DIR = "downloads"
DEFAULT_REPOSITORIES_FILE = "repositories.txt"
DEFAULT_CHECK_TIME = "10:00"
DEFAULT_TIMEZONE = "Europe/Zurich"

# Argument Parser für flexibles Download-Verzeichnis, Prüfungszeit und Zeitzone
parser = argparse.ArgumentParser(description='GitHub Releases und Master Branch Downloader für mehrere Repositories')
parser.add_argument('--download-dir', default=DEFAULT_DOWNLOAD_DIR, help='Verzeichnis zum Speichern der Downloads')
parser.add_argument('--repositories-file', default=DEFAULT_REPOSITORIES_FILE, help='Datei mit GitHub-Repository-URLs')
parser.add_argument('--check-time', default=DEFAULT_CHECK_TIME, help='Zeit für tägliche Prüfungen im Format HH:MM (24-Stunden)')
parser.add_argument('--timezone', default=DEFAULT_TIMEZONE, help='Zeitzone für die Prüfungszeit, z.B. Europe/Zurich')
args = parser.parse_args()

DOWNLOAD_DIR = args.download_dir
REPOSITORIES_FILE = args.repositories_file
CHECK_TIME = args.check_time
TIMEZONE = args.timezone

# Logging Einrichtung
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Globale Variable zur Speicherung der Repositories
repositories = {}

def load_repositories(file_path):
    """Lädt eine Liste von GitHub-Repository-URLs aus einer Datei."""
    if not os.path.exists(file_path):
        logging.error(f"Repositories-Datei nicht gefunden: {file_path}")
        return []
    with open(file_path, 'r', encoding='utf-8') as file:
        urls = [line.strip() for line in file if line.strip()]
    logging.info(f"{len(urls)} Repository(ies) geladen.")
    return urls

def parse_github_url(repo_url):
    """Parst die GitHub-URL und gibt den Besitzer und Repository-Namen zurück."""
    parsed_url = urlparse(repo_url)
    path_parts = parsed_url.path.strip("/").split("/")
    if len(path_parts) < 2:
        logging.error(f"Ungültige GitHub-URL: {repo_url}. Erwartetes Format: 'https://github.com/Owner/Repo'")
        return None, None
    owner, repo = path_parts[0], path_parts[1]
    return owner, repo

def setup_repository_dirs(owner, repo):
    """Erstellt die erforderlichen Verzeichnisse für ein Repository."""
    repo_dir = os.path.join(DOWNLOAD_DIR, repo)
    releases_dir = os.path.join(repo_dir, "releases")
    os.makedirs(releases_dir, exist_ok=True)
    return repo_dir, releases_dir

def get_github_api_url(owner, repo):
    """Konstruiert die GitHub-API-URL für ein Repository."""
    return f"https://api.github.com/repos/{owner}/{repo}"

def get_latest_release(api_url):
    """Holt die neueste Veröffentlichung von GitHub."""
    url = f"{api_url}/releases/latest"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

def get_latest_commit(api_url, branch='master'):
    """Holt den neuesten Commit eines angegebenen Branches."""
    url = f"{api_url}/commits/{branch}"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

def get_commit_history(api_url, base, head):
    """Holt die Commit-Historie zwischen zwei Commits."""
    url = f"{api_url}/compare/{base}...{head}"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

def download_file(url, dest):
    """Lädt eine Datei von einer gegebenen URL herunter."""
    try:
        with requests.get(url, stream=True) as response:
            response.raise_for_status()
            with open(dest, 'wb') as file:
                for chunk in response.iter_content(chunk_size=8192):
                    file.write(chunk)
        logging.info(f"Heruntergeladen: {dest}")
    except requests.exceptions.RequestException as e:
        logging.error(f"Fehler beim Herunterladen von {url}: {e}")

def download_release_assets(release, releases_dir):
    """Lädt alle Assets einer Veröffentlichung herunter."""
    for asset in release.get('assets', []):
        asset_url = asset['browser_download_url']
        asset_name = asset['name']
        dest_path = os.path.join(releases_dir, asset_name)
        logging.info(f"Herunterladen des Release-Assets: {asset_name}")
        download_file(asset_url, dest_path)

def generate_release_notes(commits, release_notes_path):
    """Generiert Release-Notizen basierend auf Commits."""
    with open(release_notes_path, 'a', encoding='utf-8') as file:
        file.write("\nNeue Commits seit dem letzten Release:\n")
        file.write("======================================\n\n")
        for commit in commits:
            message = commit['commit']['message'].split('\n')[0]
            author = commit['commit']['author']['name']
            date = commit['commit']['author']['date']
            file.write(f"- {message} (von {author} am {date})\n")
    logging.info(f"Erweiterte Release-Notizen generiert: {release_notes_path}")

def check_and_download_release(api_url, repo_dir, releases_dir, version_file, release_notes_file):
    """Überprüft auf neue Releases und lädt diese herunter."""
    try:
        latest_release = get_latest_release(api_url)
        latest_version = latest_release['tag_name']
        latest_commit_hash = latest_release['target_commitish']

        # Prüfen, ob diese Version bereits heruntergeladen wurde
        if os.path.exists(version_file):
            with open(version_file, 'r', encoding='utf-8') as file:
                current_version, current_hash = file.read().strip().split(',')
            if current_version == latest_version and current_hash == latest_commit_hash:
                logging.info(f"Neueste Veröffentlichung ({latest_version}) bereits heruntergeladen.")
                return

            # Commit-Historie seit dem letzten Release abrufen
            commit_history = get_commit_history(api_url, current_hash, latest_commit_hash)
            commits = commit_history.get('commits', [])
            if commits:
                generate_release_notes(commits, release_notes_file)
        else:
            commits = []

        # Neue Veröffentlichung gefunden
        logging.info(f"Neue Veröffentlichung gefunden: {latest_version}. Download beginnt...")
        download_release_assets(latest_release, releases_dir)

        # Release-Notizen generieren
        generate_release_notes(commits, release_notes_file)

        # Versionsinfo aktualisieren
        with open(version_file, 'w', encoding='utf-8') as file:
            file.write(f"{latest_version},{latest_commit_hash}")
        logging.info(f"Versionsinfo aktualisiert: {latest_version}")
    except requests.exceptions.HTTPError as e:
        logging.error(f"HTTP-Fehler beim Abrufen der neuesten Veröffentlichung: {e}")
    except Exception as e:
        logging.error(f"Unbekannter Fehler bei der Überprüfung der Veröffentlichung: {e}")

def check_and_download_master(api_url, repo_dir, master_zip_path, version_file_master, branch='master'):
    """Überprüft auf neue Commits im Hauptbranch und lädt den aktuellen Stand herunter."""
    try:
        latest_commit = get_latest_commit(api_url, branch)
        latest_commit_hash = latest_commit['sha']

        # Prüfen, ob dieser Commit bereits heruntergeladen wurde
        if os.path.exists(version_file_master):
            with open(version_file_master, 'r', encoding='utf-8') as file:
                _, current_hash = file.read().strip().split(',')
            if current_hash == latest_commit_hash:
                logging.info("Hauptbranch ist auf dem neuesten Stand.")
                return

        # Neuer Commit gefunden
        logging.info("Neuer Commit im Hauptbranch gefunden. Download beginnt...")
        owner, repo = parse_github_url(api_url)[0], parse_github_url(api_url)[1]
        master_zip_url = f"https://github.com/{owner}/{repo}/archive/refs/heads/{branch}.zip"
        download_file(master_zip_url, master_zip_path)

        # Versionsinfo für den Hauptbranch aktualisieren
        with open(version_file_master, 'w', encoding='utf-8') as file:
            file.write(f"{branch},{latest_commit_hash}")
        logging.info(f"Hauptbranch-Version aktualisiert: {latest_commit_hash}")
    except requests.exceptions.HTTPError as e:
        logging.error(f"HTTP-Fehler beim Abrufen des neuesten Commits: {e}")
    except Exception as e:
        logging.error(f"Unbekannter Fehler bei der Überprüfung des Hauptbranches: {e}")

def initial_download(owner, repo, api_url, repo_dir, releases_dir):
    """Führt die initialen Downloads von Master und Releases durch."""
    version_file_release = os.path.join(repo_dir, "version_info_release.txt")
    version_file_master = os.path.join(repo_dir, "version_info_master.txt")
    release_notes_file = os.path.join(repo_dir, "release_notes.txt")
    master_zip_path = os.path.join(repo_dir, f"{repo}_master.zip")

    logging.info(f"Initialer Download für Repository: {repo}")
    check_and_download_master(api_url, repo_dir, master_zip_path, version_file_master)
    check_and_download_release(api_url, repo_dir, releases_dir, version_file_release, release_notes_file)

def daily_check(owner, repo, api_url, repo_dir, releases_dir):
    """Führt tägliche Überprüfungen auf Updates durch."""
    version_file_release = os.path.join(repo_dir, "version_info_release.txt")
    version_file_master = os.path.join(repo_dir, "version_info_master.txt")
    release_notes_file = os.path.join(repo_dir, "release_notes.txt")
    master_zip_path = os.path.join(repo_dir, f"{repo}_master.zip")

    logging.info(f"Tägliche Überprüfung gestartet für Repository: {repo}")
    check_and_download_master(api_url, repo_dir, master_zip_path, version_file_master)
    check_and_download_release(api_url, repo_dir, releases_dir, version_file_release, release_notes_file)
    logging.info(f"Tägliche Überprüfung abgeschlossen für Repository: {repo}")

def process_repository(repo_url):
    """Verarbeitet ein einzelnes GitHub-Repository."""
    owner, repo = parse_github_url(repo_url)
    if not owner or not repo:
        return

    if repo in repositories:
        logging.info(f"Repository bereits verarbeitet: {repo}")
        return

    repo_dir, releases_dir = setup_repository_dirs(owner, repo)
    api_url = get_github_api_url(owner, repo)

    version_file_release = os.path.join(repo_dir, "version_info_release.txt")
    version_file_master = os.path.join(repo_dir, "version_info_master.txt")
    release_notes_file = os.path.join(repo_dir, "release_notes.txt")
    master_zip_path = os.path.join(repo_dir, f"{repo}_master.zip")

    # Initialer Download, falls Versionsdateien nicht vorhanden sind
    if not os.path.exists(version_file_master) or not os.path.exists(version_file_release):
        initial_download(owner, repo, api_url, repo_dir, releases_dir)

    # Planung täglicher Prüfungen
    schedule.every().day.at(CHECK_TIME).do(daily_check, owner, repo, api_url, repo_dir, releases_dir)
    logging.info(f"Geplante tägliche Prüfungen um {CHECK_TIME} Uhr für Repository: {repo}")

    # Speichern der verarbeiteten Repository
    repositories[repo] = {
        'owner': owner,
        'repo': repo,
        'api_url': api_url,
        'repo_dir': repo_dir,
        'releases_dir': releases_dir
    }

class RepositoriesFileEventHandler(FileSystemEventHandler):
    """Handler für Dateiänderungen an repositories.txt."""

    def __init__(self, file_path):
        super().__init__()
        self.file_path = os.path.abspath(file_path)

    def on_modified(self, event):
        if os.path.abspath(event.src_path) == self.file_path:
            logging.info(f"Änderung erkannt an: {self.file_path}. Laden der Repositories...")
            new_urls = load_repositories(self.file_path)
            for url in new_urls:
                process_repository(url)

def main():
    """Hauptfunktion zur Ausführung des Skripts."""
    # Initiales Laden der Repositories
    initial_urls = load_repositories(REPOSITORIES_FILE)
    for repo_url in initial_urls:
        process_repository(repo_url)

    # Einrichtung der Echtzeitüberwachung der repositories.txt
    event_handler = RepositoriesFileEventHandler(REPOSITORIES_FILE)
    observer = Observer()
    observer.schedule(event_handler, path=os.path.dirname(os.path.abspath(REPOSITORIES_FILE)) or '.', recursive=False)
    observer.start()
    logging.info(f"Echtzeitüberwachung der Datei: {REPOSITORIES_FILE} gestartet.")

    # Zeitzone festlegen
    try:
        timezone = pytz.timezone(TIMEZONE)
    except pytz.UnknownTimeZoneError:
        logging.error(f"Unbekannte Zeitzone: {TIMEZONE}. Verwenden von Europe/Zurich als Standard.")
        timezone = pytz.timezone("Europe/Zurich")

    # Formatierung der Prüfungszeit entsprechend der Zeitzone
    def get_local_check_time():
        now = datetime.now(timezone)
        check_datetime = timezone.localize(datetime.strptime(CHECK_TIME, "%H:%M"))
        return check_datetime.strftime("%H:%M")

    # Aktualisieren der Prüfungszeit im Schedule entsprechend der Zeitzone
    schedule_time = get_local_check_time()
    for job in schedule.jobs:
        job.at = schedule_time

    logging.info(f"Update-Prüfer läuft. Tägliche Prüfungen um {CHECK_TIME} Uhr in Zeitzone {TIMEZONE}.")

    # Endlosschleife zur Ausführung geplanter Aufgaben und Beobachtung von Änderungen
    try:
        while True:
            schedule.run_pending()
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
        logging.info("Update-Prüfer gestoppt vom Benutzer.")
    observer.join()

if __name__ == "__main__":
    main()
```

### **Erläuterungen zu den Erweiterungen**

#### **1. Echtzeitüberwachung der `repositories.txt`**

- **Watchdog-Bibliothek**: Das Skript verwendet die `watchdog`-Bibliothek, um Änderungen an der `repositories.txt`-Datei in Echtzeit zu erkennen.
- **Event-Handler**: Die Klasse `RepositoriesFileEventHandler` überwacht die Datei und lädt neue URLs automatisch, sobald Änderungen erkannt werden.
- **Dynamische Verarbeitung**: Neue Repository-URLs, die zur `repositories.txt` hinzugefügt werden, werden sofort verarbeitet und in das entsprechende Ordnersystem integriert.

#### **2. Erweiterte Release-Notizen**

- **Commit-Historie**: Die Funktion `get_commit_history` nutzt die GitHub-Compare-API, um die Commits zwischen dem letzten und dem neuesten Release abzurufen.
- **Detaillierte Notizen**: Diese Commits werden in den `release_notes.txt`-Dateien der jeweiligen Repositories dokumentiert, um eine umfassende Übersicht der Änderungen bereitzustellen.

#### **3. Konfigurierbare Zeitzoneneinstellungen**

- **Kommandozeilenargumente**: Über die Argumente `--check-time` und `--timezone` können Benutzer die tägliche Prüfungszeit und die entsprechende Zeitzone festlegen.
- **Zeitzonenvalidierung**: Das Skript überprüft die eingegebene Zeitzone und fällt bei ungültigen Eingaben auf eine Standardzeitzone zurück.
- **Anpassbare Prüfungszeit**: Die Prüfungszeit wird entsprechend der angegebenen Zeitzone formatiert und im Zeitplan aktualisiert.

### **Struktur des Download-Verzeichnisses**

Die Struktur des Download-Verzeichnisses wird für jedes Repository individuell erstellt. Beispielsweise bei der Verarbeitung von `https://github.com/GreemDev/Ryujinx` sieht die Struktur wie folgt aus:

```
downloads/
└── Ryujinx/
    ├── Ryujinx_master.zip
    ├── releases/
    │   ├── asset1.zip
    │   └── asset2.zip
    ├── version_info_release.txt
    ├── version_info_master.txt
    └── release_notes.txt
```

### **Anwendung des Skripts**

1. **Erstellen der Repositories-Datei**

   Erstellen Sie eine Datei namens `repositories.txt` im selben Verzeichnis wie das Skript und fügen Sie die GitHub-URLs der zu überwachenden Repositories hinzu, eine pro Zeile.

   **Beispiel:**
   ```
   https://github.com/GreemDev/Ryujinx
   https://github.com/Owner/AnotherRepo
   ```

2. **Ausführen des Skripts**

   Führen Sie das Skript mit den gewünschten Parametern aus:

   ```bash
   python github_downloader.py --download-dir /pfad/zum/verzeichnis --repositories-file /pfad/zur/repositories.txt --check-time 09:30 --timezone Europe/Berlin
   ```

   **Optionale Parameter:**
   - `--download-dir`: Verzeichnis zum Speichern der Downloads (Standard: `downloads`)
   - `--repositories-file`: Datei mit GitHub-Repository-URLs (Standard: `repositories.txt`)
   - `--check-time`: Zeit für tägliche Prüfungen im Format `HH:MM` (Standard: `10:00`)
   - `--timezone`: Zeitzone für die Prüfungszeit, z.B. `Europe/Zurich` (Standard: `Europe/Zurich`)

3. **Automatische Verarbeitung**

   - **Live-Hinzufügen**: Fügen Sie neue Repository-URLs zur `repositories.txt` hinzu. Das Skript erkennt die Änderungen automatisch und beginnt mit der Verarbeitung der neuen Repositories.
   - **Tägliche Prüfungen**: Das Skript führt täglich zur festgelegten Zeit Prüfungen durch und lädt bei Bedarf neue Releases oder Master-Commits herunter. Die entsprechenden Informationen werden in den jeweiligen Repository-Ordnern gespeichert.

### **Zusätzliche Verbesserungsvorschläge**

- **Erweiterte Fehlerbehandlung**: Implementieren Sie spezifischere Fehlerbehandlungen, z.B. für API-Rate-Limiting von GitHub.
- **Benachrichtigungen**: Integrieren Sie eine Benachrichtigungsfunktion (z.B. E-Mail oder Slack), um über erfolgreiche Downloads oder Fehler informiert zu werden.
- **Konfigurationsdatei**: Nutzen Sie eine Konfigurationsdatei (z.B. YAML oder JSON) für erweiterte Konfigurationseinstellungen anstelle von ausschließlich Kommandozeilenargumenten.
- **Parallelverarbeitung**: Optimieren Sie das Skript durch parallele Verarbeitung mehrerer Repositories gleichzeitig, um die Effizienz zu steigern.

### **Fazit**

Dieses erweiterte Skript bietet eine umfassende und flexible Lösung zur automatischen Verwaltung und Aktualisierung mehrerer GitHub-Repositories. Durch die Integration von Echtzeitüberwachung, detaillierten Release-Notizen und konfigurierbaren Zeitzoneneinstellungen ist es sowohl skalierbar als auch benutzerfreundlich. Die strukturierten Ordneraufteilungen und die robuste Fehlerbehandlung gewährleisten eine zuverlässige und wartbare Anwendung, die sich ideal für die Verwaltung einer großen Anzahl von Projekten eignet.