# Antworten zu Phase 4 – Den Workflow verstehen

## Teil A: Aufbau der Workflow-Datei

1. **Events:** `push` (auf jeden Branch), `pull_request` (nur mit Ziel
   `main`) und `workflow_dispatch` (manuell per Knopf im Actions-Tab).
2. **Runner:** `ubuntu-latest`, eine frische virtuelle Ubuntu-Maschine
   (GitHub-hosted Runner). GitHub startet sie für jeden Lauf neu und
   verwirft sie danach. Docker und Compose sind vorinstalliert.
3. **Job:** `stack-test` (Anzeigename "Compose-Stack testen (Port 8085)")
   mit **10 Steps**.
4. Nur **Step 1** nutzt eine fertige Action (`actions/checkout@v7`, lädt
   den Code auf den Runner). Alle anderen führen Shell-Befehle mit `run:`
   aus.
5. **Rechte:** `contents: read`. Der Workflow darf den Code nur lesen. Das
   ist das Prinzip der minimalen Rechte: Wird ein Schritt kompromittiert,
   kann er wenig Schaden anrichten.
6. **`if: ${{ !github.event.repository.is_template }}`:** In der Vorlage
   gibt es noch keinen Stack, der Job wird dort übersprungen, damit die
   Vorlage nicht dauerhaft rot ist. In meinem Repo läuft er immer.

## Teil B: Was wird geprüft?

7. Welcher Step schlägt zuerst fehl, wenn ...
   1. ... in der `compose.yml` ein Tabulator statt Leerzeichen steht?
      Step 3 (YAML-Syntax prüfen).
   2. ... der Ordner `html/` nie committet wurde?
      Step 7 (Inhalt prüfen), denn `html/index.html` fehlt.
   3. ... `ports: - "8085:8080"` eingetragen ist?
      Step 6 (Erreichbarkeit): Docker leitet auf Container-Port 8080
      weiter, der Webserver lauscht aber auf 80.
   4. ... die `compose.yml` in einem Unterordner liegt?
      Step 2 (Compose-Datei suchen).
8. Ein Webserver antwortet auch dann auf Port 8085, wenn er die falsche
   Seite ausliefert, z. B. seine eingebaute Standardseite, weil das Volume
   am falschen Ort hängt (so beim ersten Apache-Versuch in Phase 5). Ein
   guter Test prüft das Ergebnis, das der Kunde erwartet, nicht nur, ob
   "irgendwas läuft".
9. Step 10 sorgt für ein sauberes Ende (Container und Netzwerke werden
   entfernt). Auf selbst gehosteten Runnern, die wiederverwendet werden,
   würden sonst Container liegen bleiben und Ports blockieren.

## Teil C: Einordnen

10. **Continuous Integration.** Jede Änderung wird automatisch gebaut und
    getestet. Continuous Delivery (fertiges Artefakt bereitstellen) und
    Continuous Deployment (automatisch auf den Server ausrollen)
    passieren hier nicht. Den Server aktualisiere ich selbst mit
    `git pull` und `docker compose up -d`.
11. Der Test läuft bei jeder Änderung und wird nicht vergessen. Er läuft
    immer gleich in einer sauberen Umgebung ("bei mir geht's" zählt
    nicht). Fehler fallen vor dem Betrieb auf, und das Ergebnis ist für
    alle sichtbar und dokumentiert (Haken am Commit, Abzeichen).

## Teil D: Die DoD-Abzeichen

12. Ein Abzeichen zeigt den letzten Lauf auf einem Branch. DoD 3 soll
    zeigen, ob es `dev` mit einer eigenen Änderung gibt. Die Prüfung muss
    also dort laufen, wo `dev` gepusht wird, und das Abzeichen muss die
    Läufe auf `dev` zeigen (`?branch=dev`). Apache und Pull Request
    betreffen den Betriebsstand, also `main`.
13. Es braucht den zuletzt gemergten Pull Request von `dev` nach `main`
    mit Merge-Zeitpunkt (`merged_at`) und letztem Commit (`head.sha`),
    dazu das Ergebnis des Checks "Compose-Stack testen (Port 8085)" für
    genau diesen Commit mit Zeitpunkt (`completed_at`). DoD 5 ist nur
    grün, wenn der Check erfolgreich war und vor dem Merge fertig wurde.

---

## Expertenfrage: Was prüft der Workflow, und warum vergleicht Step 7?

- Step 1: Code des Repos auf den Runner holen.
- Step 2: Liegt eine Compose-Datei im Hauptordner?
- Step 3: Ist sie gültiges Compose-YAML (`docker compose config`)?
- Step 4: Veröffentlicht ein Service den Host-Port 8085?
- Step 5: Startet dieser Service auf dem Runner?
- Step 6: Antwortet auf Port 8085 überhaupt ein Webserver?
- Step 7: Liefert er mit HTTP 200 genau meine `html/index.html` aus?
- Step 8: Steckbrief in die Zusammenfassung schreiben (immer).
- Step 9: Container-Logs anzeigen (nur bei Fehler).
- Step 10: Aufräumen (immer).

**Warum Step 7 die Inhalte vergleicht:** Nur den Port zu testen, würde
einen Dienst als "gut" durchwinken, der zwar läuft, aber das Falsche tut.
Beim Wechsel auf Apache zeigt sich das direkt: Wird das Volume noch nach
`/usr/share/nginx/html` gemountet, antwortet Apache auf 8085 trotzdem,
liefert aber seine Standardseite "It works!". Step 6 wäre grün, die
Statusseite im Betrieb aber weg. Erst der Vergleich mit der Datei im Repo
erkennt, dass der Dienst nicht das liefert, was der Kunde erwartet.
