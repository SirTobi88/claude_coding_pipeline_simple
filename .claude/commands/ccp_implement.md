---
description: Einen Task umsetzen lassen – Worktree, Modell nach Tier, Prüfung, Übergabe ans Review.
argument-hint: "<task-id>"
---

# /ccp_implement

Task: `$ARGUMENTS` (lokal die Nummer aus `tasks/`, GitHub die Issue-Nummer). Lies
`.claude/ccp/pipeline.json` und `.claude/ccp/TASK.md`. `<wt>` = `.worktrees/<id>`.

## 1. Task laden

- **local:** `tasks/<id>-*.md`.
- **github:** `gh issue view <id> --json number,title,body,labels,comments`; zusätzlich der
  offene PR zum Branch `task/<id>`, falls es einen gibt:
  `gh pr list --head task/<id> --json number,url,comments`.

Erlaubt sind die Status `todo`, `fix` und `doing` (unterbrochener Lauf). Sonst melden, Ende.
Ein offener Eintrag in `blocked_by` → melden, Ende. Status auf `doing` setzen
(GitHub: altes `status:*`-Label entfernen, `status:doing` setzen).

## 2. Worktree

- `<wt>` existiert → weiterverwenden (Nachbesserung oder Fortsetzung).
- Sonst Startpunkt `<start>`: lokal `<default_branch>`; GitHub erst
  `git fetch origin <default_branch>`, dann `origin/<default_branch>`.
  - Branch `task/<id>` gibt es noch nicht: `git worktree add <wt> -b task/<id> <start>`
  - gibt es schon: `git worktree add <wt> task/<id>`

## 3. Runner und Modell

`runner` = Task-Feld, sonst `runner` aus der Konfiguration; `auto` → `auto_runner[tier]`.
Dann `tiers[tier][runner]`. Ist das `null` (z. B. `local`, noch nicht eingerichtet): Hinweis
ausgeben und `claude` nehmen.

## 4. Auftrag

Der Auftrag besteht aus: `Worktree: <absoluter Pfad von <wt>>`, `Testbefehl: <test_command>`,
dem vollständigen Task-Text, und bei Status `fix` den Befunden des letzten Versuchs (lokal der
letzte `## Review`-Abschnitt; GitHub der letzte Kommentar mit `<!-- ccp:fix -->` an Issue oder PR)
sowie einer vorhandenen `## Antwort` des Planners.

## 5. Ausführen

- **claude:** Agent `implementer` mit `model` und `effort` aus Schritt 3 und dem Auftrag als Prompt.
- **codex:**
  - Neuestes Modell der Familie:
    `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-<family>"' | cut -d'"' -f4 | sort -V | tail -1`
    Leer → Hinweis, Runner `claude` nehmen.
  - Prompt bauen: Auftrag nach `.worktrees/<id>.task.md` schreiben, dann
    ```
    awk 'n>=2; /^---$/{n++}' .claude/agents/implementer.md .worktrees/<id>.task.md > .worktrees/<id>.prompt.md
    ```
    (Implementer-Text ohne Frontmatter, darunter der Auftrag.)
  - Starten (Timeout 10 Minuten; reicht das nicht, im Hintergrund starten und auf die Meldung warten):
    ```
    codex exec -C .worktrees/<id> -s workspace-write -m <slug> -c model_reasoning_effort='"<effort>"' -o .worktrees/<id>.result.md - < .worktrees/<id>.prompt.md > .worktrees/<id>.log 2>&1
    ```
  - Rückgabe = `.worktrees/<id>.result.md`.

## 6. Ergebnis prüfen – nie ungeprüft übernehmen

- `ERGEBNIS: ESKALATION` → Schritt 8.
- `git -C <wt> status --porcelain`: keine Änderung → Fehlversuch „keine Änderung“.
- **Allowlist:** jeder geänderte oder neue Pfad muss unter **Dateien** passen → sonst
  Fehlversuch mit den Pfaden.
- **Tests:** `cd <wt> && <test_command>` → rot: Fehlversuch mit den letzten ~40 Zeilen der Ausgabe.

**Fehlversuch** = Befund festhalten (lokal neuer Abschnitt `## Review <n>` im Task; GitHub
Kommentar an PR, sonst Issue, beginnend mit `<!-- ccp:fix -->`), dann Status `fix` und die
Regel „hochstufen statt schleifen“ aus `TASK.md` anwenden. Ende.

## 7. Übergeben

- `git -C <wt> add -A`, dann commit mit der `COMMIT:`-Zeile der Rückgabe (sonst
  `feat: <title>`) und der Zeile `Task <id> · <runner> <modell> · tier <n>` im Body.
- **local:** Status `review`.
- **github:** `git -C <wt> push -u origin task/<id>`. Gibt es noch keinen PR:
  `gh pr create --base <default_branch> --head task/<id> --title "<title>" --body "<Zusammenfassung der Rückgabe>\n\nCloses #<id>"`.
  Label `status:review`.
- Melden: Task, Runner/Modell, Tests grün, nächster Schritt `/ccp_review <id>`.

## 8. Eskalation

- Noch nicht eskaliert (lokal `escalated: false`; GitHub kein `<!-- ccp:answer -->`):
  Rückfrage festhalten (lokal `## Rückfrage`, GitHub Issue-Kommentar), dann Agent `planner`
  mit `Modus answer. mode=<mode>. Task: <text>. Rückfrage: <frage>`.
  - `ANTWORT:` → Status `todo`; der Worktree bleibt, der nächste Lauf setzt dort fort.
  - `FRAGE:` → Status `blocked`, Frage im Task bzw. als Issue-Kommentar.
- Schon einmal eskaliert → Status `blocked` mit der Frage.

Melde in jedem Fall in einer Zeile, wo der Task jetzt steht.
