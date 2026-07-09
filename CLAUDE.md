# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

This repository is the **external build/transport repo** for a self-built jArchi plugin used with Archi 5.8.0. It does not contain the Archi model or source data; it contains:

- Build instructions and the reproducible setup (`SETUP.md`, `BUILD_PROMPT.md`).
- The locally cloned jArchi sources and build tooling under `jarchi-build/`.
- Two jArchi scripts (`*.ajs`) that run inside Archi on Windows to validate the plugin and sync an exported `archimate_model.json` into an Archi model.

## What is (and is not) in this repo

- `sync_archi.ajs` — upserts elements/relationships from `archimate_model.json` into the currently open Archi model. Matches by `extid`, creates new entries, updates existing ones, and prunes data-owned objects no longer present in the export.
- `check_jarchi.ajs` — prerequisite check: verifies jArchi is running, GraalVM Java interop is available, and `archimate_model.json` is readable and valid.
- `SETUP.md` — full manual setup: install Eclipse RCP, import `archi-scripting-plugin`, set Archi 5.8.0 as target platform, export plugin, transport via GitHub, install on Windows, verify.
- `BUILD_PROMPT.md` — headless Kubuntu build prompt for the `com.archimatetool.script` plugin.
- `jarchi-build/archi-scripting-plugin` — git submodule-like clone of `github.com/archimatetool/archi-scripting-plugin` at tag `v1.12.0`.
- `jarchi-build/eclipse` — Eclipse IDE for RCP/RAP Developers (Linux x86_64 tarball).
- `jarchi-build/out/plugins/com.archimatetool.script_1.12.0.*.jar` — the generated plugin jar.

There is **no application code, package manager, test runner, linter, or CI pipeline** at the repository root. The only "tests" are the two `.ajs` scripts executed inside Archi.

## Common commands

Because this repo is documentation and build artifacts, the usual commands are file/script operations rather than `npm`/`maven`/`pytest` workflows.

### Build the jArchi plugin (manual/Eclipse GUI)

Follow `SETUP.md`. The short version is:

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
~/jarchi-build/eclipse/eclipse -vm "$JAVA_HOME/bin/java"
```

In Eclipse:

1. Import existing projects from `jarchi-build/archi-scripting-plugin/`.
2. Set the Target Platform to the installed Archi 5.8.0 directory.
3. Export `com.archimatetool.script` via **Plug-in Development → Deployable plug-ins and fragments** to `jarchi-build/out`.

### Build the jArchi plugin (headless)

Use `BUILD_PROMPT.md` as the task prompt. It describes compiling against the Archi 5.8.0 `plugins/` directory and the bundled GraalVM libs in `com.archimatetool.script/lib/`, then assembling the OSGi bundle jar into `dist/` with a SHA256 checksum.

### Verify the built plugin

List bundle contents and inspect the manifest:

```bash
BUNDLE=jarchi-build/out/plugins/com.archimatetool.script_1.12.0.*.jar
unzip -l "$BUNDLE" | grep -E "MANIFEST\.MF|plugin\.xml|lib/|\.class|\.jar"
unzip -p "$BUNDLE" META-INF/MANIFEST.MF | grep -E "Bundle-SymbolicName|Bundle-Version|Bundle-ClassPath"
```

### Run the scripts

`check_jarchi.ajs` and `sync_archi.ajs` are executed from Archi's **Scripts** view, not from the shell. Requirements:

- Archi 5.8.0 with the self-built jArchi plugin loaded.
- The script folder configured in Archi preferences points to the directory containing these `.ajs` files.
- A target Archi model is open (for `sync_archi.ajs`).

### Prepare a distributable plugin archive

```bash
cd jarchi-build/out
zip -r ../jArchi-1.12.0.archiplugin plugins features 2>/dev/null || zip -r ../jArchi-1.12.0.archiplugin plugins
```

### Transport the artifact via this repo

```bash
mkdir -p dist
cp jarchi-build/out/plugins/com.archimatetool.script_1.12.0.*.jar dist/
( cd dist && sha256sum *.jar > jArchi-1.12.0.sha256 )
```

Then commit/push only `dist/` as described in `SETUP.md` section 6. The `.gitignore` already blocks internal data files.

## High-level architecture

### Repository roles

The repository has three distinct roles:

1. **Build reference** (`SETUP.md`, `BUILD_PROMPT.md`) — documents how to build jArchi from source on Kubuntu and install it on Windows.
2. **Artifact transport** (`dist/`) — holds the generated OSGi bundle and checksum. This is the only binary output that should be committed to this external repo.
3. **Runtime scripts** (`*.ajs`) — small JavaScript files executed by the GraalVM engine inside Archi. They bridge Archi's Java model API with the external JSON export.

### Data flow

```
External data source (.accdb / inventory DB)
       │
       ▼
  archimate_model.json   (produced outside this repo, NOT committed here)
       │
       ▼
  sync_archi.ajs         (runs inside Archi, reads JSON, mutates model)
       │
       ▼
  Archi model            (open in Archi on Windows)
```

`check_jarchi.ajs` validates the bridge before mutation.

### Build flow

```
jarchi-build/archi-scripting-plugin  (sources, tag v1.12.0)
       │
       ▼
Eclipse PDE / headless javac          (compile against Archi 5.8.0 + lib/*.jar)
       │
       ▼
jarchi-build/out/plugins/com.archimatetool.script_1.12.0.*.jar
       │
       ▼
dist/                                 (committed and pushed to GitHub)
       │
       ▼
Windows Archi 5.8.0 dropins           (installed plugin)
```

## Important constraints

### Version lockstep

Archi on the build machine and on Windows must be the same version. Current target: **Archi 5.8.0** ↔ **jArchi v1.12.0** ↔ **JDK 21** ↔ **GraalVM JS libs 24.1.2**. Do not change these versions without explicit agreement.

### Security boundary: no internal data in this repo

The `.gitignore` blocks internal network/communication data:

```
ips.txt, dns.json, hosts.json, *.csv, archimate_model.json,
checkmk_*.json, ip_enrichment.csv, *.accdb, *.mdb
```

Never commit these to this external repo. Only the plugin artifact in `dist/`, the `.ajs` scripts, and the documentation should be versioned here. If internal data must be versioned, use an internal/private Git repository, not `github.com/KaiPercz/Archimate`.

### Where to find authoritative facts

- Source origin and versions: `SETUP.md` section 1 and 10.
- Headless build steps: `BUILD_PROMPT.md`.
- GitHub authentication/transport: `SETUP.md` section 6.
- Windows installation and verification: `SETUP.md` sections 7 and 8.
