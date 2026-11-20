# Antworten – Phase 4: Den Workflow verstehen

## Teil A: Aufbau der Workflow-Datei

1. **Bei welchen Ereignissen (Events) startet der Workflow?**  
   - `push` (bei jedem Push auf jeden Branch)  
   - `pull_request` (bei jedem Pull Request auf den Branch `main`)  
   - `workflow_dispatch` (manuell per Knopfdruck im Actions-Tab)

2. **Auf welchem Runner läuft der Job? Wer stellt ihn bereit?**  
   `ubuntu-latest` – ein frischer Ubuntu-Rechner mit Docker und Docker Compose.  
   Bereitgestellt wird er von **GitHub** (GitHub-hosted Runner). In öffentlichen Repos ist er kostenlos.

3. **Wie heißt der Job und wie viele Steps hat er?**  
   Job-Name: `Compose-Stack testen (Port 8085)` (intern: `stack-test`)  
   Er hat **10 Steps**.

4. **Welcher Step nutzt eine fertige Action (`uses:`), welche führen eigene Befehle aus (`run:`)?**  
   - **Action (`uses:`):** nur Step 1 – `actions/checkout@v7`  
   - **Eigene Befehle (`run:`):** alle anderen Steps (2 bis 10)

5. **Welche Rechte hat der Workflow (`permissions`)? Warum so knapp?**  
   `contents: read` – der Workflow darf den Code nur lesen.  
   Das Prinzip der minimalen Rechte (Least Privilege) reduziert das Risiko, falls der Workflow jemals kompromittiert wird. Mehr braucht er nicht.

6. **Was bewirkt die Bedingung `if: ${{ !github.event.repository.is_template }}`?**  
   - In der **Vorlage** (Template-Repository): Der Job wird **übersprungen**, weil dort noch kein echter Stack liegt.  
   - In **deinem Repo**: Der Job läuft immer, weil dein Repo kein Template mehr ist.

---

## Teil B: Was wird geprüft?

1. **Welcher Step würde zuerst fehlschlagen, wenn …**

   | Situation | Erster fehlschlagender Step | Begründung |
   |-----------|-----------------------------|----------|
   | In der `compose.yml` steht ein Tabulator statt Leerzeichen | **Step 3** (YAML-Syntax prüfen) | `docker compose config` erkennt ungültiges YAML (Tabs sind in YAML verboten). |
   | Der Ordner `html/` wurde nie committet | **Step 7** (Inhalt prüfen) | Step 7 prüft explizit, ob `html/index.html` existiert. |
   | `ports: - "8085:8080"` eingetragen ist | **Step 4** (Port 8085 prüfen) | Der Host-Port muss **8085** sein. Hier ist der Host-Port 8085, der Container-Port aber 8080 – das ist falsch (richtig wäre z. B. `"8085:80"`). |
   | Die `compose.yml` liegt versehentlich in einem Unterordner | **Step 2** (Compose-Datei suchen) | Step 2 sucht nur im Root-Verzeichnis des Repos. |

2. **Warum reicht es in Step 7 nicht, nur zu prüfen, ob Port 8085 antwortet?**  
   Weil der Server auch eine **falsche Seite** ausliefern könnte (z. B. die Standard-Willkommensseite von Apache oder nginx).  
   Der `diff` stellt sicher, dass **genau deine** `html/index.html` ausgeliefert wird. Das wird besonders wichtig in **Phase 5**, wenn du auf Apache umstellst und das Volume korrekt einbinden musst.

3. **Wozu dient Step 10 (Aufräumen), obwohl der Runner danach gelöscht wird?**  
   - Sauberkeit und gute Praxis  
   - Verhindert, dass laufende Container oder Volumes im Log oder bei abgebrochenen Läufen Probleme machen  
   - Macht den Lauf deterministisch und nachvollziehbar  
   - `if: always()` stellt sicher, dass aufgeräumt wird, auch wenn vorher etwas schiefging

---

## Teil C: Einordnen

1. **Ist das Continuous Integration, Continuous Delivery oder Continuous Deployment?**  
   → **Continuous Integration (CI)**  
   Begründung: Der Workflow baut und testet automatisch bei jedem Push/PR. Es wird nichts ausgeliefert oder deployed. Das Deployment machst du später selbst manuell mit `git pull && docker compose up -d`.

2. **Zwei Vorteile gegenüber „mal kurz mit curl testen“:**  
   1. **Frühe und zuverlässige Fehlererkennung** – Fehler werden sofort nach dem Push sichtbar, nicht erst, wenn jemand manuell testet.  
   2. **Reproduzierbarkeit** – Jeder Lauf startet in einer frischen, identischen Umgebung. „Bei mir geht’s“ zählt nicht mehr. Außerdem ist die Prüfung versioniert und nachvollziehbar.

---
