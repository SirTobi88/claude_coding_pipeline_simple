---
description: Pipeline im aktuellen Repo einrichten – Modus local oder github.
argument-hint: "local | github"
---

# /ccp_setup

Modus: `$ARGUMENTS` (leer → fragen). Dieser Command läuft **mit dir**, nicht im Loop: frag
nach, wo etwas unklar ist, und bestätige vor jedem Push und jeder Änderung an GitHub-Einstellungen.

## Immer

1. Git-Repo mit mindestens einem Commit? Sonst anbieten: `git init` + erster Commit.
2. `.claude/ccp/pipeline.json` anpassen:
   - `mode` = gewählter Modus.
   - `default_branch`: lokal `git symbolic-ref --short HEAD`; GitHub
     `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`.
   - `test_command`: aus dem Projekt ableiten (`run_tests.sh`, `package.json` → `npm test`,
     `pyproject.toml`/`pytest.ini` → `python3 -m pytest`, `Package.swift` → `swift test`,
     `Cargo.toml` → `cargo test`, `go.mod` → `go test ./...`, `Makefile` mit `test`) und
     **einmal ausführen**. Exit-Code muss das Ergebnis sein. Nichts gefunden → fragen.
   - Den Testbefehl in `.claude/settings.json` unter `permissions.allow` als `Bash(<test_command>)`
     eintragen, damit `/loop /ccp_next` dafür nicht nachfragt.
3. `.gitignore` um `.worktrees/` ergänzen.
4. `codex --version` – fehlt Codex, `runner` auf `claude` setzen und das melden.
5. Prüfen, dass `CLAUDE.md` existiert; sonst die Vorlage aus der Pipeline anbieten.

## Modus local

- `tasks/` und `tasks/ideas.md` (Überschrift `# Ideen`) anlegen.
- `.gitignore` um `tasks/` ergänzen – Tasks sind lokaler Zustand. (Willst du sie versioniert,
  lass die Zeile weg; dann committe Task-Dateien selbst.)

## Modus github

1. `gh auth status` und `gh repo view --json nameWithOwner,visibility,isPrivate`.
2. Labels anlegen (`gh label create <name> --color <farbe> --force`):
   `idea`, `status:todo`, `status:doing`, `status:review`, `status:fix`, `status:blocked`,
   `tier:1` … `tier:5`, `runner:claude`, `runner:codex`.
3. `ci` in der Konfiguration:
   - `true`: `.github/workflows/ci.yml` muss existieren; Toolchain-Schritt für das Projekt
     ausfüllen (die Vorlage hat eine EDIT-Stelle) und `branches: [main]` an `<default_branch>`
     anpassen. Committen; **pushen machst du selbst** (`git push origin <default_branch>`) –
     die Pipeline darf nicht auf den Standardbranch pushen.
   - `false`: kein Workflow; die Pipeline testet nur lokal.
4. **Branch Protection (optional, nur Pflicht-Check, keine Approval-Pflicht).** Nach Bestätigung:
   ```
   gh api -X PUT repos/<owner>/<repo>/branches/<default_branch>/protection --input .worktrees/protection.json
   ```
   mit `.worktrees/protection.json`:
   ```json
   {"required_status_checks": {"strict": false, "contexts": ["ci"]},
    "enforce_admins": false, "required_pull_request_reviews": null, "restrictions": null}
   ```
   Antwortet GitHub mit 403 („Upgrade to GitHub Pro …“) – privates Repo mit GitHub Free –,
   melde das und mach weiter: die Pipeline prüft Tests und Checks vor jedem Merge selbst.

## Abschluss

Zusammenfassung: Modus, Testbefehl (Ergebnis des Probelaufs), Runner, Branch Protection
ja/nein, und der nächste Schritt: `/ccp_plan "<idee>"`, dann `/ccp_next` oder `/loop /ccp_next`.
