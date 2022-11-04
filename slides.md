---
marp: true
class: invert
paginate: true
---

<style>
img[alt~="center"] {
  display: block;
  margin: 0 auto;
}
</style>

![h:120 center](img/github-actions.png)

# Continuous Integration mit KiCad

### _Oder auch:_ Dumme Fehler und nervige Aufgaben vermeiden
### _Oder auch:_ Automatisierte Hilfe beim PCB-Review

---

![bg left:42%](img/raphael_01.jpg)

# Über mich

**Raphael Lehmann**

👩‍🎓 [_Elektrotechnik, Informationstechnik & technische Informatik_ @ RWTH Aachen](#)
🤖 [Roboterclub Aachen e.V.](https://www.roboterclub.rwth-aachen.de/)
💫 [TechAachen e.V.](https://techaachen.de/)

Kontakt
💌 raphael@rleh.de
🦜 [@rleh_de](https://twitter.com/rleh_de)
![invert h:25](img/Octicons-mark-github.svg) [@rleh](https://github.com/rleh/)

---

<!-- class: invert -->

![bg right:40%](img/TechTurbo_Julia.jpg)

# PCB Design Prozess

1. ✏️ Schaltplan
2. 👀 Review
3. ✍️ Layout
4. 👀 Review
5. 🩹 Anmerkungen aus Reviews umsetzen
6. ⚙️ Fertigungsdaten vorbereiten
7. 💥 Fehler in Fertigungsdaten oder Bauteile nicht lieferbar
8. 🔁 Nächste Iteration
9. 🔝 ...
10. 💾 Projekt archivieren

---

# Wer kennt es nicht?

- Unterschiedliche Titel, Revision, Datum auf verschiedenen Schaltplanseiten
- Uneindeutige Bezeichnungen
- DRC Errors
- Falsche oder keine KiCad-Version installiert
- Outdated PDFs von Schaltplan und Dokumentation
<!-- weitere ...-->

![bg left h:300](img/kicad-error-lib-absolute-path.png)

---

# Github Actions

![bg h:600 right](img/gh-modm-pr-ci-success.jpg)

- Skripte bei Events (Push, Pull Request, Merge, ...) ausführen
- Umgebung: Docker
  [`docker run ghcr.io/rleh/kicad6_and_pandoc:latest`](https://github.com/rleh/docker-kicad-pandoc)
- ...

---

![bg w:580 right:45%](img/kibot_workflow.png)

# KiBot 

https://github.com/INTI-CMNB/KiBot

🚧 Under development, funktioniert gut!

🚢 Enthalten im Docker image `ghcr.io/rleh/kicad6_and_pandoc`

🏳️‍🌈 Nutzt coole KiCad plugins
- [🔗](https://github.com/openscopeproject/InteractiveHtmlBom) Interactive HTML BOM
- [🔗](https://github.com/SchrodingersGat/KiBoM) KiBoM

---
<!-- _class: non-inverted -->

# 👩‍🍳🍳 `.github/workflows/kicad.yml`

```yaml
name: "Kicad Checks"

on:
  push:

jobs:
  kicad_checks:
    runs-on: ubuntu-latest
    container: ghcr.io/rleh/kicad6_and_pandoc:latest

# ...
```

---
<!-- _class: non-inverted -->

# 👩‍🍳🍳 Vorbereitungen

```yaml
- name: Identify changed KiCad projects (compared to main)
  run: |
    echo PROJECTS=$(git diff --diff-filter=ACMRT \
    --name-only origin/main... | grep "pcbs/" | grep -v "pcbs/lib" | \
    cut -d/ -f2 | uniq) >> $GITHUB_ENV

- name: Exclude projects without KiBot file
  shell: bash
  run: |
    FILTERED_PROJECTS=()
    for P in $PROJECTS; do
      if [[ -f "pcbs/$P/config.kibot.yaml" ]]; then
        # This project has a valid KiBot file and will be checked
        FILTERED_PROJECTS+=($P)
      fi
    done
    echo PROJECTS=$(echo ${FILTERED_PROJECTS[@]}) >> $GITHUB_ENV
    echo PROJECT_COUNT=$(echo ${#FILTERED_PROJECTS[@]}) >> $GITHUB_ENV
```

---
<!-- _class: non-inverted -->

# 🍜 Revisionsvergleich

```yaml
- name: Check revision numbers identical
  if: env.PROJECT_COUNT >= 1
  run: |
    for P in $PROJECTS
    do
      (cd pcbs/$P && ../../.github/res/kicad_revision_check.sh)
    done
```

---
<!-- _class: non-inverted -->

# 🧙 `kicad_revision_check.sh`

```bash
#!/bin/bash
return_code=0

echo "All revision strings have to be identical. Checking..."
n=$(grep -hR "^    (rev" | cut -c 11- | rev | cut -c 3- | rev | uniq | wc -l)
if [ $n -ne 1 ]; then
    echo "ERROR: Multiple different revision strings detected ($n)"
    ((return_code+=10))
fi

# ...

exit $return_code

```

---
<!-- _class: non-inverted -->

# KiBot

```yaml
- name: Print schematic
  if: env.PROJECT_COUNT >= 1
  run: |
    for P in $PROJECTS
    do
      echo $P
      (cd pcbs/$P && kibot -v --skip-pre all print_sch)
    done

- name: Gerber export
  if: env.PROJECT_COUNT >= 1
  run: |
    for P in $PROJECTS
    do
      (cd pcbs/$P && kibot -v --skip-pre all gerbers gerber_drills)
      (cd pcbs/$P/gerber && zip ../gerber.zip *)
    done
```
---

KiBot Export Möglichkeiten
- PDF von Schaltplan und Layout
- Gerber/Drill, Pick & place
- Verschiedene Varianten für BOM
- Interactive HTML zum Bestücken
- 3D Modell (STEP, o.ä.)
- Renderings 🖼
- ...

➜ [KiBot Dokumentation 🔗](https://github.com/INTI-CMNB/KiBot#the-outputs-section)

![bg h:330 left:45%](img/kibot_740x400_logo.png)

---

## Interactive HTML BOM plugin for KiCad [🔗](https://github.com/openscopeproject/InteractiveHtmlBom)

![h:520](img/ibom-plugin.png)

😘😘😘

---

# Bauteil-Matching 🧩

- Automatisierte Zuordnung von Bauteilen in Bauteildatenbank
  ![h:25](img/partsbox-logo.png) PartsBox oder ![h:25](img/partkeepr-logo.svg) PartKeepr
- Heuristik für Standard-Bauteile (Widerstände, Kondensatoren, LEDs, ...)
- Stringmatching: MPN im _Value_-Feld
- Report als PDF/HTML/... generieren
- Merge: Projekt/BOM in Bauteildatenbank automatisiert anlegen

---
<!-- _class: non-inverted -->

# Bauteil-Matching 🧩

🔥 KiCad Python-Scripting

```python
#!/usr/bin/env python3
import pcbnew

filename = "test.kicad_pcb"
board: pcbnew.BOARD = pcbnew.LoadBoard(filename)

components = board.GetFootprints()
rev = str(board.GetTitleBlock().GetRevision())

# ...
```
---

# Bauteildatenbank

![](img/partsbox-rca-leds.png)

Wichtig: Einheitliche Konvention für Bauteilbezeichnungen
Z.B.: `LED {color} {current rating} {package}`

---
<!-- _class: non-inverted -->

# Bauteil-Matching 🔥🌈

```python
for c im components:
  reference = str(c.GetReference())
  value = str(c.GetValue())
  footprint = str(c.GetFPID().GetLibItemName())
  package = re.search(r"(?P<package>0402|0603|0805|1206)", footprint)
  if package is not None:
    package = package.group("package")

  # ...

  if "LED" in c.footprint:
    color = re.search(r"(?P<color>red|green|blue|yellow|orange)", value.lower())
    if package and color:
      color = color.group("color")
      return partsbox_search_by_name(f"LED {color} 20mA {package}")
  if re.search(r'C\d+', reference):
    # ...
  # ...
  else:
    # fallthrough case: MPN specified in value field
    return partsbox_search_by_name(value)
    
# ...
```

---

# Vielen Dank für eure Aufmerksamkeit!

## Fragen?

---

# Backup

---

# KiCad Libraries
🥇 Qualitätsstandard übertrifft kommerzielle Bibliotheken um Größenordnungen

📈 Aktuell ≥ 900 offene Merge-Requests für Symbole und Footprints 📚

👎 Wechsel vom Github zu Gitlab am 01.10.2020
➜ ≥ 500 Pull-Requests faktisch beerdigt

### Review-Prozess
👥 Zu wenig Reviewer
⚙️ Viel manueller Aufwand
➜ Mehr Automatisierung

**Wie können _wir_ das langfristig verbessern? 🤔**

---

![](img/gh-kicad-symbols-unmerged-screenshot.png)

---

![](img/gh-kicad-footprints-unmerged-screenshot.png)

---

![](img/gl-kicad-symbols-unmerged-screenshot.png)

---

![](img/gl-kicad-footprints-unmerged-screenshot.png)

---

# 💡 Idee: KiCad Library Hackathon

🔧 Tooling verbessern

👶 KiCad Nutzer als Library-Maintainer gewinnen

📚 Backlog an offenen PRs mit Symbolen und Footprints abarbeiten

🌍 An mehreren Orten?

---

# Ende 🏁
