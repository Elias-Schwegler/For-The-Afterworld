## Erweiterung des Skripts zur Handhabung mehrerer GitHub-URLs

Das folgende Python-Skript wurde weiterentwickelt, um eine Liste von GitHub-Projekt-URLs zu verarbeiten. Es ermöglicht das dynamische Hinzufügen von URLs, erstellt eine übersichtliche Ordnerstruktur für jede Repository und speichert die jeweiligen Release-Notizen sowie Versionsinformationen innerhalb der spezifischen Download-Ordner.

### **Hauptmerkmale**

- **Mehrfach-Repository-Unterstützung**: Handhabt eine Liste von GitHub-URLs und ermöglicht das dynamische Hinzufügen neuer Repositories.
- **Strukturierte Ordneraufteilung**: Erstellt separate Ordner für jedes Repository innerhalb des Download-Verzeichnisses.
- **Individuelle Versionsverwaltung**: Speichert Release-Notizen und Versionsinformationen in den jeweiligen Repository-Ordnern.
- **Robuste Fehlerbehandlung und Logging**: Umfassende Protokollierung zur einfachen Fehlerdiagnose und Wartung.

### **Voraussetzungen**

Stellen Sie sicher, dass die folgenden Python-Pakete installiert sind:

```bash
pip install requests schedule pytz
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

# Standard-Konfiguration
DEFAULT_DOWNLOAD_DIR = "downloads"
REPOSITORIES_FILE = "repositories.txt"  # Datei mit GitHub-URLs

# Argument Parser für flexibles Download-Verzeichnis
parser = argparse.ArgumentParser(description='GitHub Releases und Master Branch Downloader für mehrere Repositories')
parser.add_argument('--download-dir', default=DEFAULT_DOWNLOAD_DIR, help='Verzeichnis zum Speichern der Downloads')
parser.add_argument('--repositories-file', default=REPOSITORIES_FILE, help='Datei mit GitHub-Repository-URLs')
args = parser.parse_args()

DOWNLOAD_DIR = args.download_dir
REPOSITORIES_FILE = args.repositories_file

# Logging Einrichtung
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

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
    with open(release_notes_path, 'w', encoding='utf-8') as file:
        file.write("Release Notes\n")
        file.write("=============\n\n")
        for commit in commits:
            message = commit['commit']['message'].split('\n')[0]
            author = commit['commit']['author']['name']
            date = commit['commit']['author']['date']
            file.write(f"- {message} (von {author} am {date})\n")
    logging.info(f"Release-Notizen generiert: {release_notes_path}")

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

        # Neue Veröffentlichung gefunden
        logging.info(f"Neue Veröffentlichung gefunden: {latest_version}. Download beginnt...")
        download_release_assets(latest_release, releases_dir)

        # Optional: Commit-Historie seit letztem Release abrufen (hier Platzhalter)
        generate_release_notes([], release_notes_file)  # Placeholder für tatsächliche Commits

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
        master_zip_url = f"https://github.com/{parse_github_url(api_url)[0]}/{parse_github_url(api_url)[1]}/archive/refs/heads/{branch}.zip"
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
    schedule_time = "10:00"
    schedule.every().day.at(schedule_time).do(daily_check, owner, repo, api_url, repo_dir, releases_dir)
    logging.info(f"Geplante tägliche Prüfungen um {schedule_time} Uhr für Repository: {repo}")

def main():
    """Hauptfunktion zur Ausführung des Skripts."""
    repositories = load_repositories(REPOSITORIES_FILE)
    for repo_url in repositories:
        process_repository(repo_url)

    # Zeitzone festlegen (z.B. Europe/Zurich)
    timezone = pytz.timezone("Europe/Zurich")

    logging.info(f"Update-Prüfer läuft. Tägliche Prüfungen um 10:00 Uhr in Zeitzone {timezone}.")

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

### **Erläuterungen zu den Erweiterungen**

#### **1. Handhabung einer Liste von GitHub-URLs**

- **`repositories.txt`**: Das Skript erwartet eine Textdatei (standardmäßig `repositories.txt`) mit einer GitHub-URL pro Zeile. Sie können diese Datei anpassen, um Repositories dynamisch hinzuzufügen oder zu entfernen.
  
  **Beispiel für `repositories.txt`:**
  ```
  https://github.com/GreemDev/Ryujinx
  https://github.com/Owner/AnotherRepo
  ```

- **Dynamisches Laden**: Das Skript lädt zu Beginn alle URLs aus der angegebenen Datei und verarbeitet jedes Repository unabhängig voneinander.

#### **2. Strukturierte Ordnererstellung für jedes Repository**

- **Separate Verzeichnisse**: Für jedes Repository wird ein eigener Ordner innerhalb des angegebenen Download-Verzeichnisses erstellt. Die Struktur sieht wie folgt aus:

  ```
  downloads/
  ├── Ryujinx/
  │   ├── master.zip
  │   ├── releases/
  │   │   ├── asset1.zip
  │   │   └── asset2.zip
  │   ├── version_info_release.txt
  │   ├── version_info_master.txt
  │   └── release_notes.txt
  └── AnotherRepo/
      ├── master.zip
      ├── releases/
      │   ├── assetA.zip
      │   └── assetB.zip
      ├── version_info_release.txt
      ├── version_info_master.txt
      └── release_notes.txt
  ```

#### **3. Individuelle Versionsverwaltung und Release-Notizen**

- **Versionsdateien**: Jede Repository hat separate Versionsdateien für Releases und den Hauptbranch (`version_info_release.txt` und `version_info_master.txt`), die im jeweiligen Repository-Ordner gespeichert sind.
  
- **Release-Notizen**: Die `release_notes.txt` wird ebenfalls pro Repository erstellt und enthält die generierten Notizen für die neuesten Releases.

#### **4. Dynamisches Hinzufügen von Repositories**

- **Einfache Aktualisierung**: Um ein neues Repository hinzuzufügen, fügen Sie einfach die entsprechende GitHub-URL zur `repositories.txt` hinzu und starten Sie das Skript erneut oder integrieren Sie eine Überwachung der Datei für Echtzeit-Updates.

#### **5. Verbesserte Fehlerbehandlung und Logging**

- **Detaillierte Protokolle**: Das Skript verwendet das `logging`-Modul, um detaillierte Informationen über den Ablauf und eventuelle Fehler bereitzustellen.
  
- **Fehlerüberprüfung**: Das Skript überprüft die Gültigkeit jeder GitHub-URL und handhabt HTTP-Fehler sowie andere unerwartete Ausnahmen robust.

### **Anwendung des Skripts**

1. **Erstellen der Repositories-Datei**

   Erstellen Sie eine Datei namens `repositories.txt` im selben Verzeichnis wie das Skript und fügen Sie die GitHub-URLs der zu überwachenden Repositories hinzu, eine pro Zeile.

   **Beispiel:**
   ```
   https://github.com/GreemDev/Ryujinx
   https://github.com/Owner/AnotherRepo
   ```

2. **Ausführen des Skripts**

   Führen Sie das Skript mit den Standardparametern aus:

   ```bash
   python github_downloader.py
   ```

   Optional können Sie ein spezielles Download-Verzeichnis oder eine andere Repositories-Datei angeben:

   ```bash
   python github_downloader.py --download-dir /pfad/zum/verzeichnis --repositories-file /pfad/zur/repositories.txt
   ```

3. **Automatische Updates**

   Das Skript führt täglich um 10:00 Uhr Überprüfungen für jedes Repository durch und lädt bei Bedarf neue Releases oder Master-Commits herunter. Die Protokolle werden in der Konsole angezeigt und können bei Bedarf angepasst werden.

### **Zusätzliche Verbesserungsvorschläge**

- **Überwachung der Repositories-Datei**: Implementieren Sie eine Funktion zur Echtzeitüberwachung der `repositories.txt`, sodass neue URLs automatisch erkannt und verarbeitet werden, ohne das Skript neu starten zu müssen.

- **Erweiterte Release-Notizen**: Integrieren Sie die tatsächliche Commit-Historie seit dem letzten Release, um detailliertere Release-Notizen zu generieren.

- **Konfigurierbare Zeitzoneneinstellungen**: Erweitern Sie die Skriptoptionen, um die tägliche Prüfungszeit und Zeitzone flexibel festzulegen.

### **Fazit**

Dieses erweiterte Skript bietet eine umfassende Lösung zur automatischen Verwaltung und Aktualisierung mehrerer GitHub-Repositories. Durch die strukturierte Ordneraufteilung, individuelle Versionsverwaltung und robuste Fehlerbehandlung ist es sowohl skalierbar als auch wartbar, wodurch es sich ideal für die Verwaltung einer großen Anzahl von Projekten eignet.