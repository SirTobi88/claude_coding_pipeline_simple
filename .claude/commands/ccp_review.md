---
description: Einen Task reviewen lassen und bei OK mergen.
argument-hint: "<task-id>"
---

# /ccp_review

Task: `$ARGUMENTS`. Lies `.claude/ccp/pipeline.json` und `.claude/ccp/TASK.md`.
`<wt>` = `.worktrees/<id>`.

## 1. Vorbereiten

- Task laden wie in `/ccp_implement` § 1. Status muss `review` sein, sonst melden, Ende.
- `<wt>` fehlt → `git worktree add <wt> task/<id>` (GitHub vorher `git fetch origin task/<id>`
  und, falls der lokale Branch fehlt, `git worktree add <wt> -b task/<id> origin/task/<id>`).
- Basis `<base>`: lokal `<default_branch>`; GitHub `git fetch origin <default_branch>`, dann
  `origin/<default_branch>`.

## 2. Reviewer starten

Agent `reviewer` mit `model`/`effort` aus `reviewer[tier]`, sonst `reviewer.default`. Prompt:
`Worktree: <absoluter Pfad>`, `Basis: <base>`, `Testbefehl: <test_command>`, der vollständige
Task-Text.

## 3. Urteil umsetzen

### OK → mergen

**local**
1. `git -C <wt> rebase <default_branch>` – Konflikt → `git -C <wt> rebase --abort`, als
   Fehlversuch „Konflikt mit <default_branch> in <dateien>“ behandeln (unten), Ende.
2. `cd <wt> && <test_command>` – rot → Fehlversuch mit der Ausgabe, Ende.
3. Im Hauptverzeichnis: aktueller Branch muss `<default_branch>` sein. `git merge --squash task/<id>`
   – verweigert Git das wegen lokaler Änderungen: Status `blocked` mit „Hauptverzeichnis hat
   ungesicherte Änderungen in <dateien>“, Ende.
4. `git commit` mit dem Titel des letzten Task-Commits und `Task <id> · review <modell>: OK` im Body.
5. Aufräumen: `git worktree remove <wt>`, `git branch -D task/<id>`,
   `.worktrees/<id>.task.md`, `.prompt.md`, `.result.md`, `.log` löschen. Status `done`.

**github**
1. Urteil als PR-Kommentar (beginnt mit `<!-- ccp:review -->`).
2. Konflikt prüfen: `gh pr view <pr> --json mergeable` → `CONFLICTING`:
   `git -C <wt> merge origin/<default_branch>`; lassen sich die Konflikte nicht eindeutig
   lösen → `git -C <wt> merge --abort`, Fehlversuch, Ende. Sonst Tests im Worktree,
   `git -C <wt> push`.
3. `cd <wt> && <test_command>` – rot → Fehlversuch, Ende.
4. Checks: `gh pr checks <pr> --watch` (Timeout 15 Minuten). Meldet `gh` „no checks
   reported“ und `ci` ist `false` → weiter. Ein roter Check → Fehlversuch mit Name und
   Log-Auszug (`gh run view <run-id> --log-failed | tail -40`), Ende.
5. `gh pr merge <pr> --squash --delete-branch`. Kein `--auto`: die Pipeline prüft selbst und
   verlässt sich nicht auf Branch Protection.
6. Aufräumen wie lokal (Worktree, lokaler Branch, Hilfsdateien), `status:*`-Labels vom Issue
   entfernen. Das Issue schließt GitHub über `Closes #<id>`.

### CHANGES → Fehlversuch

Befunde festhalten (lokal `## Review <n>` im Task; GitHub PR-Kommentar, beginnt mit
`<!-- ccp:fix -->`), Status `fix`, Regel „hochstufen statt schleifen“ aus `TASK.md`.

### ESCALATE → blocked

Frage festhalten (lokal `## Rückfrage`; GitHub Issue-Kommentar), Status `blocked`.

Melde in einer Zeile: Urteil, Modell, wo der Task jetzt steht.
