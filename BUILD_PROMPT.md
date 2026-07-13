# Claude-Code-Prompt: jArchi headless auf Kubuntu bauen

Kopiere den folgenden Abschnitt als Aufgabe in **Claude Code auf der Kubuntu-Maschine**.
Er baut das jArchi-Plugin ohne Eclipse-GUI und legt das installierbare Artefakt in `dist/` ab.

> Lies zuerst `SETUP.md` in diesem Verzeichnis — es ist die maßgebliche Referenz für
> Versionen, Sicherheitsleitplanken und den Windows-Installations-/Transportweg.

---

## PROMPT (ab hier kopieren)

Du arbeitest auf einer Kubuntu-Maschine mit Internetzugang und **ohne Eclipse-GUI**
(rein headless). Ziel: das **jArchi-Plugin** aus den Quellen bauen und als installierbares
OSGi-Bundle bereitstellen. Der genaue Kontext steht in `SETUP.md` in diesem Repo — lies es zuerst.

### Ziel / Erfolgskriterien
- Ein Bundle-Jar `com.archimatetool.script_1.12.0.*.jar`, das enthält:
  - die kompilierten Klassen (gemäß `Bundle-ClassPath` ggf. als eingebettetes Jar),
  - den Ordner `lib/` mit den gebündelten GraalVM-Libs (24.1.2),
  - `META-INF/MANIFEST.MF`, `plugin.xml` und die Ressourcen laut `build.properties`.
- Artefakt liegt unter `dist/` inkl. `.sha256`-Prüfsumme.
- **Keine** internen Daten anfassen/committen — nur das Plugin-Artefakt (siehe Leitplanken).

### Feste Fakten (nicht neu erraten)
- Ziel-Archi-Version: **5.8.0** (muss zur Windows-Installation passen — Lockstep-Regel).
- jArchi-Quellen: Tag **v1.12.0** aus `github.com/archimatetool/archi-scripting-plugin`.
- Build-Werkzeug im Repo: **keins** (kein `pom.xml`, kein `.target`) → reines PDE-Projekt.
  Deshalb **headless bauen**, NICHT über die (hier nicht verfügbare) Eclipse-GUI.
- Für JS-Sync genügt das eine Plugin **`com.archimatetool.script`**; die Sub-Projekte
  `.groovy/.jruby/.nashorn/.premium` sind nicht nötig.
- Die JS-Engine (GraalVM 24.1.2) liegt bereits gebündelt in
  `com.archimatetool.script/lib/` — nichts extern aufzulösen.

### Umgebung vorbereiten
```bash
sudo apt update && sudo apt install -y openjdk-21-jdk unzip zip git
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
mkdir -p ~/jarchi-build && cd ~/jarchi-build

# Quellen versionsgepinnt
git clone https://github.com/archimatetool/archi-scripting-plugin.git
git -C archi-scripting-plugin checkout v1.12.0

# Archi 5.8.0 Linux NUR als Kompilier-Klassenpfad (liefert com.archimatetool.editor u.a.).
# Download-Link der Version 5.8.0 (Linux x86_64) von archimatetool.com verwenden, dann:
#   tar xzf Archi-*linux*.tgz   ->  Ordner mit plugins/*.jar
```

### Bauen — primärer Weg: direkte Kompilierung + Bundle-Assemblierung
1. Lies `com.archimatetool.script/META-INF/MANIFEST.MF` und `com.archimatetool.script/build.properties`
   als **maßgebliche** Vorgabe. Bestimme daraus:
   - `Bundle-ClassPath` (ob die Klassen als eingebettetes Jar, z. B. `com.archimatetool.script.jar`,
     erwartet werden — dann Klassen in dieses Nested-Jar packen),
   - `bin.includes` (welche Ordner/Dateien ins Bundle gehören: `META-INF/`, `plugin.xml`,
     `plugin.properties`, `img/`, `templates/`, `js/`, `lib/`, `help/`, …),
   - `Bundle-SymbolicName` und `Bundle-Version` (für den Jar-Namen).
2. Kompiliere gegen Archi + die gebündelten Libs — **und kopiere danach die Nicht-`.java`-
   Ressourcen aus `src/` mit** (ohne diesen Schritt fehlen die NLS-`messages.properties`;
   jArchi wirft dann „NLS missing message: …" und bricht mit einer Exception ab):
   ```bash
   ARCHI=~/jarchi-build/<entpacktes-archi-5.8.0>
   SRC=~/jarchi-build/archi-scripting-plugin/com.archimatetool.script
   mkdir -p ~/jarchi-build/work/classes
   # a) kompilieren
   javac --release 21 \
     -cp "$ARCHI/plugins/*:$SRC/lib/*" \
     -d ~/jarchi-build/work/classes \
     $(find "$SRC/src" -name '*.java')
   # b) *** WICHTIG *** alle Ressourcen (Nicht-.java) aus src/ ins Kompilat kopieren,
   #    v.a. **/messages.properties (NLS). cp --parents erhaelt die Paketstruktur.
   ( cd "$SRC/src" && find . -type f ! -name '*.java' -exec cp --parents {} ~/jarchi-build/work/classes/ \; )
   ```
3. Assembliere das Bundle **exakt gemäß Schritt 1**:
   ```bash
   # Nested-Code-Jar (Name laut Bundle-ClassPath, z.B. com.archimatetool.script.jar) —
   # enthaelt jetzt Klassen UND Ressourcen (.properties):
   ( cd ~/jarchi-build/work/classes && jar cf ../com.archimatetool.script.jar . )
   # PFLICHT-Kontrolle: die NLS-Ressourcen muessen enthalten sein (sonst Build ungueltig):
   jar tf ~/jarchi-build/work/com.archimatetool.script.jar | grep -c messages.properties   # > 0 !
   ```
   Danach das äußere Bundle-Jar bauen: `com.archimatetool.script.jar` + `lib/` + `META-INF/` +
   `plugin.xml` + `plugin.properties` + `img/`/`templates/`/`js/`/`help/` (laut `bin.includes`).
   Der `MANIFEST.MF` aus den Quellen wird unverändert übernommen.

> Falls die manuelle Assemblierung am OSGi-Klassenpfad scheitert: **Fallback** = ein minimales
> **Tycho**-Setup erzeugen (Parent-`pom.xml` + `.target`, das auf `$ARCHI/plugins` zeigt) und
> `mvn -q clean package` laufen lassen. Beide Wege sind zulässig — entscheidend ist ein
> ladefähiges Bundle. Dokumentiere kurz, welcher Weg genutzt wurde.

### Verifikation (vor dem Ablegen)
```bash
BUNDLE=<pfad-zum-gebauten-jar>
unzip -l "$BUNDLE" | grep -E "MANIFEST.MF|plugin.xml|lib/|\.class|\.jar"
unzip -p "$BUNDLE" META-INF/MANIFEST.MF | grep -E "Bundle-SymbolicName|Bundle-Version|Bundle-ClassPath"
# NLS-Ressourcen im inneren Code-Jar (MUSS Treffer liefern, sonst 'NLS missing message'):
unzip -p "$BUNDLE" com.archimatetool.script.jar | jar tf /dev/stdin 2>/dev/null | grep messages.properties \
  || python3 -c "import zipfile,io,sys; z=zipfile.ZipFile('$BUNDLE'); n=zipfile.ZipFile(io.BytesIO(z.read('com.archimatetool.script.jar'))).namelist(); print([x for x in n if x.endswith('messages.properties')])"
```
Erwartet: `Bundle-SymbolicName: com.archimatetool.script`, Version `1.12.0.*`, `lib/`-Jars,
die Klassen **und** mehrere `.../messages.properties` im Code-Jar. Die **endgültige** Ladeprüfung erfolgt auf Windows mit `check_jarchi.ajs`
(siehe SETUP.md Teil G) — weise am Ende darauf hin.

### Artefakt ablegen & übertragen
```bash
mkdir -p dist
cp "$BUNDLE" dist/
( cd dist && sha256sum *.jar > jArchi-1.12.0.sha256 )
```
Danach gemäß **SETUP.md Teil E** committen und pushen (Abschnitt „Einmalige GitHub-Authentifizierung"
beachten). Ein bare Bundle-Jar reicht für die dropins-Installation auf Windows.

### Leitplanken (verbindlich)
- **Nur** das Plugin-Artefakt nach `dist/` legen und committen. **Niemals** interne Netz-/
  Kommunikationsdaten anfassen oder pushen (`ips.txt`, `dns.json`, `hosts.json`, `*.csv`,
  `archimate_model.json`, `*.accdb`). Die `.gitignore` blockiert diese — nicht umgehen.
- Version **v1.12.0 / Archi 5.8.0** einhalten. Kein Wechsel auf andere Versionen ohne Rückfrage.
- Bei blockierenden Unklarheiten (fehlender Archi-Download, Kompilierfehler) stoppen und
  präzise berichten, statt zu raten.
