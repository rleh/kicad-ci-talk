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

# Γber mich

**Raphael Lehmann**

π©βπ [_Elektrotechnik, Informationstechnik & technische Informatik_ @ RWTH Aachen](#)
π€ [Roboterclub Aachen e.V.](https://www.roboterclub.rwth-aachen.de/)
π« [TechAachen e.V.](https://techaachen.de/)

Kontakt
π raphael@rleh.de
π¦ [@rleh_de](https://twitter.com/rleh_de)
![invert h:25](img/Octicons-mark-github.svg) [@rleh](https://github.com/rleh/)

---

<!-- class: invert -->

![bg right:40%](img/TechTurbo_Julia.jpg)

# PCB Design Prozess

1. βοΈ Schaltplan
2. π Review
3. βοΈ Layout
4. π Review
5. π©Ή Anmerkungen aus Reviews umsetzen
6. βοΈ Fertigungsdaten vorbereiten
7. π₯ Fehler in Fertigungsdaten oder Bauteile nicht lieferbar
8. π NΓ€chste Iteration
9. π ...
10. πΎ Projekt archivieren

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

- Skripte bei Events (Push, Pull Request, Merge, ...) ausfΓΌhren
- Umgebung: Docker
  [`docker run ghcr.io/rleh/kicad6_and_pandoc:latest`](https://github.com/rleh/docker-kicad-pandoc)
- ...

---

![bg w:580 right:45%](img/kibot_workflow.png)

# KiBot 

https://github.com/INTI-CMNB/KiBot

π§ Under development, funktioniert gut!

π’ Enthalten im Docker image `ghcr.io/rleh/kicad6_and_pandoc`

π³οΈβπ Nutzt coole KiCad plugins
- [π](https://github.com/openscopeproject/InteractiveHtmlBom) Interactive HTML BOM
- [π](https://github.com/SchrodingersGat/KiBoM) KiBoM

---
<!-- _class: non-inverted -->

# π©βπ³π³ `.github/workflows/kicad.yml`

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

# π©βπ³π³ Vorbereitungen

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

# π Revisionsvergleich

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

# π§ `kicad_revision_check.sh`

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

KiBot Export MΓΆglichkeiten
- PDF von Schaltplan und Layout
- Gerber/Drill, Pick & place
- Verschiedene Varianten fΓΌr BOM
- Interactive HTML zum BestΓΌcken
- 3D Modell (STEP, o.Γ€.)
- Renderings πΌ
- ...

β [KiBot Dokumentation π](https://github.com/INTI-CMNB/KiBot#the-outputs-section)

![bg h:330 left:45%](img/kibot_740x400_logo.png)

---

## Interactive HTML BOM plugin for KiCad [π](https://github.com/openscopeproject/InteractiveHtmlBom)

![h:520](img/ibom-plugin.png)

πππ

---

# Bauteil-Matching π§©

- Automatisierte Zuordnung von Bauteilen in Bauteildatenbank
  ![h:25](img/partsbox-logo.png) PartsBox oder ![h:25](img/partkeepr-logo.svg) PartKeepr
- Heuristik fΓΌr Standard-Bauteile (WiderstΓ€nde, Kondensatoren, LEDs, ...)
- Stringmatching: MPN im _Value_-Feld
- Report als PDF/HTML/... generieren
- Merge: Projekt/BOM in Bauteildatenbank automatisiert anlegen

---
<!-- _class: non-inverted -->

# Bauteil-Matching π§©

π₯ KiCad Python-Scripting

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

Wichtig: Einheitliche Konvention fΓΌr Bauteilbezeichnungen
Z.B.: `LED {color} {current rating} {package}`

---
<!-- _class: non-inverted -->

# Bauteil-Matching π₯π

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

# Vielen Dank fΓΌr eure Aufmerksamkeit!

## Fragen?

---

# Backup

---

# KiCad Libraries
π₯ QualitΓ€tsstandard ΓΌbertrifft kommerzielle Bibliotheken um GrΓΆΓenordnungen

π Aktuell β₯ 900 offene Merge-Requests fΓΌr Symbole und Footprints π

π Wechsel vom Github zu Gitlab am 01.10.2020
β β₯ 500 Pull-Requests faktisch beerdigt

### Review-Prozess
π₯ Zu wenig Reviewer
βοΈ Viel manueller Aufwand
β Mehr Automatisierung

**Wie kΓΆnnen _wir_ das langfristig verbessern? π€**

---

![](img/gh-kicad-symbols-unmerged-screenshot.png)

---

![](img/gh-kicad-footprints-unmerged-screenshot.png)

---

![](img/gl-kicad-symbols-unmerged-screenshot.png)

---

![](img/gl-kicad-footprints-unmerged-screenshot.png)

---

# π‘ Idee: KiCad Library Hackathon

π§ Tooling verbessern

πΆ KiCad Nutzer als Library-Maintainer gewinnen

π Backlog an offenen PRs mit Symbolen und Footprints abarbeiten

π An mehreren Orten?

---

# Ende π
