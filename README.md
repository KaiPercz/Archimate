# Archimate — jArchi-Plugin & Archi-Sync

Build-/Transport-Repository für ein **selbst gebautes jArchi-Plugin** (für Archi 5.8.0) und die
zwei **jArchi-Skripte**, die einen extern erzeugten `archimate_model.json` in ein Archi-Modell
synchronisieren. Dieses Repo enthält **keine** internen Netz-/Kommunikationsdaten (siehe `.gitignore`).

Ausführliche Referenz: [`SETUP.md`](SETUP.md) (Bau + Installation), [`BUILD_PROMPT.md`](BUILD_PROMPT.md)
(headless-Build mit Claude Code), [`CLAUDE.md`](CLAUDE.md) (Repo-Leitfaden).

## Inhalt

| Pfad | Zweck |
|---|---|
| `dist/com.archimatetool.script_1.12.0.*.jar` | fertig gebautes jArchi-Plugin (OSGi-Bundle) |
| `dist/jArchi-1.12.0.sha256` | Prüfsumme des Artefakts |
| `scripts/check_jarchi.ajs` | Voraussetzungs-Check (jArchi/GraalVM/Modell lesbar) |
| `scripts/sync_archi.ajs` | Upsert des Modells in Archi (per `extid`, inkl. Prune) |
| `SETUP.md`, `BUILD_PROMPT.md`, `CLAUDE.md` | Doku / Build-Prompt / Repo-Leitfaden |

---

## Installation unter Linux (Kubuntu) — Plugin bauen

Ziel: das Plugin `com.archimatetool.script` aus den Quellen erzeugen. **Das aktuelle
`dist/`-Artefakt ist bereits gebaut** — die folgenden Schritte sind für einen Neu-/Nachbau.

**Voraussetzungen:** JDK 21, Archi 5.8.0 (Linux, als Target/Klassenpfad), Internet.

1. **JDK & Quellen** (Details: SETUP.md Teil A):
   ```bash
   sudo apt install -y openjdk-21-jdk git zip unzip
   export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
   git clone https://github.com/archimatetool/archi-scripting-plugin.git
   git -C archi-scripting-plugin checkout v1.12.0
   ```
2. **Bauen** — zwei Wege:
   - **Headless (empfohlen hier):** Prompt aus [`BUILD_PROMPT.md`](BUILD_PROMPT.md) an Claude Code
     geben → kompiliert gegen die Archi-`plugins/` + gebündelte GraalVM-Libs und legt das Bundle
     in `dist/` ab.
   - **Eclipse-GUI:** Schritte in [`SETUP.md`](SETUP.md) Teil C–D (Target Platform = Archi 5.8.0,
     Export als *Deployable plug-ins*).
3. **Verifizieren:**
   ```bash
   unzip -p dist/com.archimatetool.script_1.12.0.*.jar META-INF/MANIFEST.MF \
     | grep -E "Bundle-SymbolicName|Bundle-Version|Bundle-ClassPath"
   sha256sum -c dist/jArchi-1.12.0.sha256
   ```
4. **Übertragen:** `dist/` committen/pushen (GitHub-Auth: SETUP.md Teil E).

> Hinweis: Ein Ausführen der `scripts/*.ajs` unter Linux ist nur sinnvoll, wenn dort auch
> Archi läuft **und** ein `archimate_model.json` unter dem erwarteten Pfad liegt
> (`$HOME/Projekte/vmware_inventory/archimate_model.json`). Im Normalfall ist Linux nur die
> **Build-Maschine**; die Synchronisation läuft auf Windows.

---

## Installation unter Windows — Plugin einsetzen

Ziel: das gebaute Plugin in Archi 5.8.0 laden und den Sync nutzen.

1. **Artefakt holen & prüfen:**
   ```powershell
   git clone https://github.com/KaiPercz/Archimate.git   # oder: git pull
   Get-FileHash .\Archimate\dist\com.archimatetool.script_1.12.0.*.jar -Algorithm SHA256
   # Wert mit dist\jArchi-1.12.0.sha256 vergleichen
   ```
2. **Plugin installieren (dropins):** Archi schließen, das Bundle-Jar kopieren nach
   `C:\Users\kgpercz\AppData\Roaming\Archi\dropins\`, Archi neu starten.
   (Alternative über die GUI: SETUP.md Teil F, Weg 2.)
3. **Scripts-Ordner setzen:** in Archi **Preferences → Scripting** → Ordner auf das
   **`scripts/`-Verzeichnis dieses Klons** zeigen (dort liegen die `.ajs`).
4. **Prüfen & synchronisieren** (Scripts-Fenster):
   - `check_jarchi.ajs` — bestätigt jArchi, GraalVM und Lesbarkeit des Modells.
   - `sync_archi.ajs` — bei geöffnetem Zielmodell ausführen → Upsert aus `archimate_model.json`.

---

## Hinterlegte Pfade in den Skripten (wichtig)

Beide `.ajs` erkennen das Betriebssystem automatisch und leiten den Pfad zur Modelldatei ab:

```js
var isWindows = System.getProperty("os.name").toLowerCase().indexOf("win") >= 0;
var JSON_PATH = isWindows
    ? "C:\\Users\\kgpercz\\Projekte\\vmware_inventory\\archimate_model.json"
    : Paths.get(System.getProperty("user.home"), "Projekte", "vmware_inventory", "archimate_model.json").toString();
```

- **Windows:** `C:\Users\kgpercz\Projekte\vmware_inventory\archimate_model.json`
- **Linux:** `$HOME/Projekte/vmware_inventory/archimate_model.json`

Der Pfad ist damit **umgebungsspezifisch** (fester Benutzer/Projektname). Weicht Ihr Layout ab,
muss `JSON_PATH` oben in **beiden** Skripten angepasst werden. Die Modelldatei selbst
(`archimate_model.json`) wird **außerhalb** dieses Repos erzeugt (`archimate_export.py`) und ist
hier bewusst **nicht** versioniert (`.gitignore`).

---

## Implementierungsstand

| Baustein | Status |
|---|---|
| jArchi-Plugin (headless gegen Archi 5.8.0 gebaut) | ✅ gebaut, in `dist/` |
| `check_jarchi.ajs` / `sync_archi.ajs` (plattformabhängiger Pfad) | ✅ vorhanden |
| Windows-Installation (dropins) + Verifikation | ⏳ durch Anwender auszuführen |
| Datengrundlage → `archimate_model.json` (Export) | ✅ funktionsfähig (intern, außerhalb dieses Repos) |
| Sync in ein reales Archi-Modell (Erst-Lauf) | ⏳ ausstehend |
| Application-/Business-Ebene im Modell | ⏳ leer bis Zuordnung gepflegt (Checkmk) |
| Kommunikations-/Flow-Beziehungen | ❌ fehlt (benötigt Flow-Quelle, s. Roadmap) |

---

## Roadmap (mögliche Verbesserungen)

1. **Pfad-Parametrisierung:** `JSON_PATH` per Umgebungsvariable / Argument / Config statt
   festem Pfad → keine Skript-Edits mehr bei anderem Layout.
2. **View-Generierung:** jArchi-Skript, das nach dem Sync automatisch Teil-Views erzeugt
   (z. B. je Subnetz oder je Business Service) — statt einer unlesbaren Gesamtansicht.
3. **Application-/Business-Ebene füllen:** Checkmk-Anbindung (Labels/Inventory) liefert die
   faktenbasierte Zuordnung Host → Anwendung → Business Service.
4. **Kommunikations-/Flow-Daten:** Anbindung von FortiAnalyzer/NSX für echte Flow-Beziehungen
   (Node → Node) — erst damit wird aus der Landschaftskarte eine echte Kommunikationslandkarte.
5. **Artefakt-Distribution:** statt Commit ins Repo ein **GitHub Release** (oder Git LFS) für
   das Bundle-Jar, um die Historie schlank zu halten.
6. **Automatisierung:** `archimate_export.py` → Sync als ein wiederholbarer Schritt
   (z. B. Wrapper, der Export und jArchi-Aufruf koppelt), inkl. optionalem headless-Build in CI.
7. **Versions-Update-Pfad:** dokumentierter, koordinierter Wechsel auf Archi 5.9.0 auf beiden
   Maschinen (Lockstep) samt Neubau (SETUP.md 1.1).

---

## Sicherheitsgrenze

Interne Netz-/Kommunikationsdaten (`ips.txt`, `dns.json`, `hosts.json`, `*.csv`,
`archimate_model.json`, `*.accdb` …) gehören **niemals** in dieses externe Repo. Die `.gitignore`
blockiert sie; für Daten ausschließlich ein internes/privates Git verwenden.
