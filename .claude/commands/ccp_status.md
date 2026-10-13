---
description: Überblick über alle Tasks und was dich braucht.
---

# /ccp_status

Lies `.claude/ccp/pipeline.json` (`mode`). Ändere nichts.

- **local:** Frontmatter aller `tasks/*.md`, offene Zeilen in `tasks/ideas.md`, `git worktree list`.
- **github:** `gh issue list --state open --json number,title,labels --limit 200`,
  `gh pr list --json number,headRefName,title,statusCheckRollup`.

Ausgabe:
1. Tabelle `ID | Titel | Status | Tier | Runner | Versuche` – ohne `done`, sortiert nach Status
   (blocked, review, fix, doing, todo).
2. **Braucht dich:** jeder `blocked`-Task und jede blockierte Idee mit ihrer Frage – und wie
   man antwortet: lokal Antwort unter `## Antwort` in die Datei schreiben und `status: todo`
   setzen; GitHub Kommentar schreiben und Label `status:blocked` durch `status:todo` ersetzen.
3. Offene Ideen (Anzahl), verwaiste Worktrees in `.worktrees/` ohne passenden offenen Task.
4. Nächster Schritt, den `/ccp_next` tun würde.
