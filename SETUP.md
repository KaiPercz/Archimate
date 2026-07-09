# SETUP.md — jArchi aus Quellen bauen (Kubuntu, entkoppelt) und auf Windows installieren

Vollständige, reproduzierbare Anleitung, um das **jArchi-Plugin** aus dem Quellcode auf
einer **entkoppelten (air-gapped) Kubuntu-Maschine** zu bauen und das Ergebnis auf der
**Windows-Arbeitsmaschine** (Archi 5.8.0) zu installieren.

> Kontext: jArchi wird als fertiges Plugin nur gegen Projektunterstützung verteilt.
> Da hier der Selbstbau erforderlich ist (Beschaffung nicht möglich / Quell-Audit erwünscht),
> beschreibt dieses Dokument den kompletten Weg. Das Plugin wird für die
> `sync_archi.ajs`/`check_jarchi.ajs`-Skripte dieses Projekts benötigt.

---

## 0. Annahmen & zu bestätigende Punkte

Diese Anleitung basiert auf folgenden Annahmen. Wo sie nicht zutreffen, sind die
betroffenen Schritte anzupassen (siehe jeweiligen Hinweis):

| # | Annahme |
|---|---|
| 1 | Kubuntu-Buildmaschine hat **Internetzugang** → Artefakte direkt per Download/apt/git |
| 2 | Transport des **Plugin-Artefakts** via **Git** (`github.com/KaiPercz/Archimate.git`) — **keine** internen Daten (siehe Teil E) |
| 3 | Kubuntu-Architektur **x86_64 (amd64)** |
| 4 | Build gegen die **bereits installierte Archi 5.8.0** (Linux) als Target Platform |
| 5 | Windows-Installation primär über **dropins**, alternativ `.archiplugin` |
| 6 | Versions-Pin siehe Versionsmatrix (Abschnitt 9) |
| 7 | Einmaliger **GUI-Export** in Eclipse (kein headless-Build) |

---

## 1. Versionsmatrix (Reproduzierbarkeit)

| Komponente | Version | Bemerkung |
|---|---|---|
| Archi (Windows-Ziel **und** Linux-Target) | **5.8.0** | müssen exakt übereinstimmen |
| jArchi Quellen | **Tag `v1.12.0`** | passt zu Archi ≥ 5.8.0 |
| JDK | **21** (Temurin/OpenJDK) | Archi/jArchi zielen auf 21 |
| Eclipse IDE | **for RCP and RAP Developers** (aktuelles Release) | enthält PDE |
| GraalVM JS Libs | **24.1.2** | bereits im Repo unter `com.archimatetool.script/lib/` gebündelt — kein Extra-Download |

> **Wichtig (Lockstep-Regel):** Die Archi-Version auf der Build-Maschine (Target Platform)
> muss mit der Ziel-Windows-Installation identisch sein (5.8.0). Ein Build gegen eine andere
> API-Version kann ein Plugin erzeugen, das auf 5.8.0 nicht lädt.

### 1.1 Optional: Update auf Archi 5.9.0

Ein Update ist **kein isolierter Schritt** auf der Build-Maschine. Wegen der Lockstep-Regel
gilt: entweder **beide** Maschinen bleiben auf 5.8.0, **oder beide** wechseln gemeinsam auf 5.9.0.

Empfehlung: Für die Erstinbetriebnahme **auf 5.8.0 bleiben** (eine Variable zur Zeit).
Wenn später auf 5.9.0 aktualisiert werden soll:

1. Windows **und** Kubuntu auf Archi 5.9.0 aktualisieren.
2. Passenden jArchi-Tag prüfen (Repo → Tags; ggf. neuer als `v1.12.0`), Quellen auschecken.
3. Target Platform auf die 5.9.0-Installation umstellen, neu bauen (Teil C–D).
4. Neu installieren (Teil F) und `check_jarchi.ajs` erneut ausführen (Teil G).

---

## 2. Teil A — Artefakte beschaffen (Kubuntu mit Internet)

Da die Kubuntu-Maschine Internet hat, werden die Artefakte direkt geholt. **Alle für Linux x86_64.**

```bash
mkdir -p ~/jarchi-build && cd ~/jarchi-build

# 1) JDK 21 (aus den Paketquellen)
sudo apt update && sudo apt install -y openjdk-21-jdk

# 2) GTK-Laufzeit für Eclipse/SWT (auf Kubuntu meist vorhanden)
sudo apt install -y libgtk-3-0

# 3) jArchi-Quellen, versionsgepinnt auf v1.12.0
git clone https://github.com/archimatetool/archi-scripting-plugin.git
git -C archi-scripting-plugin checkout v1.12.0
```

**Eclipse IDE for RCP and RAP Developers** (die apt-Version ist oft veraltet → Tarball nutzen):
von eclipse.org/downloads/packages das Paket *„Eclipse for RCP and RAP Developers"*
(Linux x86_64) laden und entpacken:
```bash
tar xzf ~/Downloads/eclipse-rcp-*-linux-gtk-x86_64.tar.gz -C ~/jarchi-build
```

> **Archi als Target Platform:** wird **nicht** heruntergeladen — die auf dieser Maschine
> bereits installierte **Archi 5.8.0** wird verwendet (siehe Teil C).
> **GraalVM-JS-Libs** werden ebenfalls nicht benötigt — sie liegen bereits im Repo unter
> `com.archimatetool.script/lib/` (Version 24.1.2).

---

## 3. Teil B — Kubuntu-Buildumgebung einrichten

JDK-Pfad nach der apt-Installation (typisch):
```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
"$JAVA_HOME/bin/java" -version    # muss 21 melden
```

Archi-Installationsverzeichnis der vorhandenen 5.8.0 ermitteln (für die Target Platform):
```bash
which archi 2>/dev/null; readlink -f "$(which archi)" 2>/dev/null
# sonst typische Orte prüfen:
ls -d /opt/Archi ~/Archi /usr/share/archi /usr/lib/archi 2>/dev/null
```
Merken Sie sich den Ordner, der `plugins/` und `Archi` (Startdatei) enthält — er wird in Teil C gebraucht.

Eclipse mit dem JDK 21 starten (verhindert Nutzung eines falschen System-Java):
```bash
~/jarchi-build/eclipse/eclipse -vm "$JAVA_HOME/bin/java"
```
Alternativ dauerhaft in `~/jarchi-build/eclipse/eclipse.ini` vor `-vmargs` eintragen:
```
-vm
/usr/lib/jvm/java-21-openjdk-amd64/bin/java
```

In Eclipse: **Window → Preferences → Java → Installed JREs** → JDK 21 hinzufügen und als
Standard markieren. **Compiler Compliance Level** = 21.

---

## 4. Teil C — Projekte importieren & Target Platform setzen

1. **Import:** File → Import → General → *Existing Projects into Workspace* →
   Root-Verzeichnis `~/jarchi-build/archi-scripting-plugin/` wählen → alle Projekte importieren
   (`com.archimatetool.script`, `.commandline`, `.feature`, `.groovy`, `.jruby`,
   `.nashorn`, `.premium`, `.tests`).
2. **Target Platform:** Preferences → **Plug-in Development → Target Platform → Add…**
   → *Nothing: Start with an empty target definition* → **Add → Directory** →
   auf das **vorhandene Archi-5.8.0-Verzeichnis** (aus Teil B, z. B. `/opt/Archi`) zeigen
   (Eclipse scannt dessen `plugins`-Ordner).
   Fertigstellen, das neue Target **anhaken (aktiv setzen)**, *Apply and Close*.
   → Damit lösen sich `com.archimatetool.editor` 5.8.0, `com.archimatetool.commandline`
   und alle Eclipse-RCP-Bundles direkt aus der Archi-Installation auf.
3. **Kompilieren:** Eclipse baut automatisch. Im **Problems**-View dürfen **keine
   Compile-Fehler** stehen. (Warnungen sind unkritisch.)

> Für unsere JS-Skripte genügt das Plugin `com.archimatetool.script` (enthält die
> GraalVM-JS-Engine aus `lib/`). Die Projekte `.groovy`/`.jruby`/`.nashorn`/`.premium`
> sind für andere Sprachen bzw. Zusatzfunktionen und für dieses Projekt nicht nötig.

---

## 5. Teil D — Bauen / Exportieren

**Empfohlen (JS reicht): einzelnes Plugin exportieren**

1. Projekt `com.archimatetool.script` selektieren.
2. File → Export → **Plug-in Development → Deployable plug-ins and fragments**.
3. Ziel: *Directory* = `~/jarchi-build/out` → **Finish**.
4. Ergebnis: `~/jarchi-build/out/plugins/com.archimatetool.script_1.12.0.*.jar`.

**Alternativ (komplettes Feature):** Projekt `com.archimatetool.script.feature`
selektieren → Export → **Deployable features** → `~/jarchi-build/out`.
Ergebnis: `out/features/…` + `out/plugins/…` (mehr Jars, inkl. weiterer Engines).

**Optional als `.archiplugin` verpacken** (für Installation über die Archi-GUI):
```bash
cd ~/jarchi-build/out
zip -r ../jArchi-1.12.0.archiplugin plugins features 2>/dev/null || zip -r ../jArchi-1.12.0.archiplugin plugins
```
> Eine `.archiplugin` ist ein ZIP mit `plugins/` (und ggf. `features/`) auf oberster Ebene.
> Wenn der GUI-Import diese Struktur einmal nicht akzeptiert, den **dropins-Weg** (Abschnitt 7)
> nutzen — der ist deterministisch.

---

## 6. Teil E — Transport zur Windows-Maschine (über Git-Repository)

Austausch-Repo: `https://github.com/KaiPercz/Archimate.git`

> **⚠️ SICHERHEIT — nur das Plugin, keine Daten:** Über dieses **externe** Repo darf
> **ausschließlich das jArchi-Plugin** (generisch, quelloffen) transportiert werden.
> **NIEMALS** interne Netzwerkdaten dorthin pushen: `ips.txt`, `dns.json`, `hosts.json`,
> die CSVs, `archimate_model.json`, `*.accdb`. Sie enthalten Ihre interne
> Kommunikationslandkarte (interne IPs, Hostnamen, offene Ports) → Offenlegungs-/Governance-Risiko.
> Die `.gitignore` im Projekt blockiert diese Dateien; falls Daten je versioniert werden
> müssen, ausschließlich ein **internes/privates** Git verwenden.

### Einmalige GitHub-Authentifizierung

GitHub akzeptiert **keine Passwörter** mehr für Git-Operationen (nur Token/SSH). Steht in der
Remote-URL eine E-Mail als Benutzername (`https://name%40mail@github.com/...`), wird zudem der
Credential-Manager umgangen und ein Passwort-Prompt erzwungen → Login schlägt fehl.

**Windows (getestet, funktioniert)** — im geklonten Verzeichnis:
```powershell
# 1) Eingebetteten Benutzernamen aus der URL entfernen
git remote set-url origin https://github.com/KaiPercz/Archimate.git
# 2) Git Credential Manager als Helper sicherstellen (GCM ist bei Git for Windows dabei)
git config --global credential.helper manager
# 3) Push -> GCM öffnet ein Browserfenster zum GitHub-Login (als Konto KaiPercz anmelden)
git push
```
Nach dem Browser-Login wird das Token im **Windows-Anmeldeinformationsspeicher** hinterlegt;
weitere Pushes/Pulls laufen ohne Nachfrage.

- **Fallback ohne Browser:** Personal Access Token (Settings → Developer settings →
  Personal access tokens, *fine-grained*, Repo `Archimate`, *Contents: Read and write*).
  Beim Push als Benutzername `KaiPercz`, als Passwort den **PAT** (nicht das Konto-Passwort).
- **Alte, falsche Anmeldung im Cache?** In der Windows-Anmeldeinformationsverwaltung den
  Eintrag `git:https://github.com` entfernen, dann erneut pushen.
- **Kubuntu (Build-Maschine):** analog PAT verwenden oder SSH-Key hinterlegen und die
  SSH-URL nutzen (`git@github.com:KaiPercz/Archimate.git`).
- **Commit-Adresse:** bei einem persönlichen Repo ggf. repo-lokal die private Adresse setzen,
  damit keine dienstliche E-Mail in die öffentliche Historie gelangt:
  `git config user.email "kaipercz@gmx.de"`.

**Auf Kubuntu (Build-Maschine)** — Artefakt einchecken:
```bash
cd ~/jarchi-build
git clone https://github.com/KaiPercz/Archimate.git repo   # einmalig
mkdir -p repo/dist
cd out && zip -r ../repo/dist/jArchi-1.12.0.zip plugins features 2>/dev/null || zip -r ../repo/dist/jArchi-1.12.0.zip plugins
cd ../repo
sha256sum dist/jArchi-1.12.0.zip > dist/jArchi-1.12.0.zip.sha256
git add dist/ && git commit -m "jArchi 1.12.0 build gegen Archi 5.8.0" && git push
```

**Auf Windows (Ziel-Maschine)** — Artefakt holen und Integrität prüfen:
```powershell
git clone https://github.com/KaiPercz/Archimate.git   # oder im vorhandenen Klon: git pull
Get-FileHash .\Archimate\dist\jArchi-1.12.0.zip -Algorithm SHA256
# Wert muss mit dem Inhalt von dist\jArchi-1.12.0.zip.sha256 uebereinstimmen
```
Danach das ZIP entpacken → weiter mit Teil F.

> **Größenhinweis:** Mit den gebündelten GraalVM-Libs kann das Artefakt mehrere 10 MB groß
> sein. GitHub warnt ab 50 MB, hartes Limit 100 MB/Datei. Bei häufigen Builds oder Überschreiten
> besser **GitHub Releases** (Datei an ein Tag anhängen) oder **Git LFS** nutzen statt in den
> Baum zu committen. Für gelegentliche Builds ist ein normaler Commit in `dist/` in Ordnung.
> Die Integrität ist ohnehin durch Git (Content-Hashing) plus die `.sha256`-Datei abgesichert.

---

## 7. Teil F — Installation auf Windows (Archi 5.8.0)

### Weg 1 — dropins (empfohlen, Pfad bekannt)
1. Empfangenes `jArchi-1.12.0.zip` auf Windows entpacken (ergibt `plugins\`, ggf. `features\`).
   Prüfsumme wie in Teil E vergleichen.
2. Archi schließen.
3. Den Inhalt (Ordner `plugins\` und ggf. `features\`) kopieren nach:
   `C:\Users\kgpercz\AppData\Roaming\Archi\dropins\`
   Dieser Pfad ist der p2-Reconciler-Ordner aus `Archi.ini`
   (`...reconciler.dropins.directory=%user.home%/AppData/Roaming/Archi/dropins`).
4. Archi starten — der Reconciler übernimmt das Plugin.

### Weg 2 — als `.archiplugin`
1. Archi: **Help → Manage Plug-ins… → Install New…**
2. `jArchi-1.12.0.archiplugin` wählen → Archi neu starten.

---

## 8. Teil G — Verifikation & Nutzung

1. Nach dem Neustart erscheint das **Scripts**-Fenster
   (Window → Show View → Other → Scripts / „Scripting").
2. **Scripts folder** setzen: Preferences → **Scripting** → Ordner auf
   `C:\Users\kgpercz\Projekte\vmware_inventory\plugin-build`
   (hier liegen `check_jarchi.ajs` und `sync_archi.ajs`).
3. **Voraussetzungs-Check:** `check_jarchi.ajs` ausführen. Es bestätigt:
   jArchi läuft, GraalVM/Java-Interop aktiv, `archimate_model.json` lesbar.
   → **Damit ist verifiziert, dass der Linux-Build auf Windows trägt.**
4. Modell öffnen/anlegen, dann `sync_archi.ajs` ausführen.
5. Wiederkehrender Ablauf:
   ```
   python archimate_export.py "<Ihre .accdb>"   # DB -> archimate_model.json
   # in Archi: Zielmodell öffnen -> sync_archi.ajs
   ```

---

## 9. Troubleshooting

| Symptom | Ursache / Lösung |
|---|---|
| Compile-Fehler `com.archimatetool.editor cannot be resolved` | Target Platform nicht aktiv oder falsche Archi-Version. Abschnitt 4.2 prüfen; Archi-Linux muss 5.8.0 sein. |
| Eclipse startet nicht / SWT-Fehler | `libgtk-3-0` fehlt. Installieren (Abschnitt 2). |
| Eclipse nutzt falsches Java | `-vm` in `eclipse.ini` auf JDK 21 setzen (Abschnitt 3). |
| Scripts-Fenster fehlt nach Installation | Plugin nicht geladen: dropins-Pfad prüfen, Archi neu starten; ggf. `-clean` einmalig: `Archi.exe -clean`. |
| `check_jarchi.ajs`: „Java-Interop FEHLT" | Falsche/zu alte jArchi-Quelle. Tag `v1.12.0` verwenden. |
| Plugin lädt nicht (Version) | Build-Archi ≠ Ziel-Archi. Beide auf 5.8.0 bringen und neu bauen. |

---

## 10. Herkunft & Prüfbarkeit (facts, not assumptions)

- Quellen: `github.com/archimatetool/archi-scripting-plugin`, Tag `v1.12.0`.
- JS-Engine (GraalVM 24.1.2) ist Teil des Repos (`com.archimatetool.script/lib/`) —
  im Selbstbau vollständig einsehbar, keine externe Blackbox.
- Das Artefakt ist reines Java und damit plattformunabhängig; der Windows-Ladevorgang
  wird durch `check_jarchi.ajs` (Abschnitt 8.3) bestätigt, bevor produktiv gearbeitet wird.
